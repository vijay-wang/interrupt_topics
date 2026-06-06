# Topic 07 — Threaded IRQs & Forced Threading

> **Standing requirements check (R1/R2/R3):** R1 — the full threaded-IRQ model: the primary/thread split, `IRQF_ONESHOT` semantics, the per-action kthread and its scheduling, forced threading, and the PREEMPT_RT picture. R2 — directly after the driver API (Topic 06), because a threaded handler is just `request_threaded_irq` with a `thread_fn`; and after flow handlers (Topic 05), because the line-masking around the thread is a flow-handler behaviour. R3 — quoted from `kernel/irq/manage.c` (`irq_thread`, `irq_thread_fn`, `irq_forced_thread_fn`, `force_irqthreads_key`) and `kernel/irq/internals.h` (`IRQTF_*`) in `v7.1-rc6`. Re-read R1/R2/R3 in the plan.
>
> **Builds on:** Topic 03 (`irqaction.thread_fn`/`thread`/`thread_mask`, `desc->threads_oneshot`), Topic 05 (the `IRQS_ONESHOT` mask and deferred EOI; `IRQ_WAKE_THREAD`), Topic 06 (`request_threaded_irq`, `IRQF_ONESHOT`).
>
> **Position in the plan:** the bridge from "tiny hardirq handler" to "do real work in a schedulable context". Topics 08–09 cover the *other* deferral mechanisms; Topic 10 covers the locking rules that make all of them safe; Topic 14 covers the RT angle in full.
>
> **Source:** `kernel/irq/manage.c` (`irq_thread`, `irq_thread_fn`, `irq_forced_thread_fn`, `irq_finalize_oneshot`, `setup_forced_irqthreads`), `kernel/irq/internals.h`.

---

## 1. Why Thread an Interrupt

The top half (Topic 06) is straitjacketed: hardirq context, IRQs off, can't sleep, must be short. But many devices need handlers that:
- **sleep** (talk to a slow I2C/SPI/regmap bus, take a `mutex`, wait on a completion),
- **run long** (process a batch), or
- **need bounded latency for other IRQs** — a long hardirq handler blocks *every* interrupt on that CPU.

A **threaded IRQ** moves the heavy part into a dedicated kernel thread that runs in **process context** (can sleep, is preemptible, has a scheduler priority). The hardirq part shrinks to "acknowledge and wake the thread". Under PREEMPT_RT this becomes the default for almost everything (§5).

---

## 2. The Split: Primary Handler + `thread_fn`

```c
request_threaded_irq(irq, primary_handler, thread_fn, IRQF_ONESHOT, name, dev);
```

Two functions, two contexts:

- **`primary_handler(irq, dev_id)`** — runs in **hardirq** context from the flow handler's action walk (Topic 05 §3). It does the urgent, non-sleeping minimum (e.g. read/ack a status register), then returns **`IRQ_WAKE_THREAD`** to request the thread, or `IRQ_HANDLED`/`IRQ_NONE` to finish without it.
- **`thread_fn(irq, dev_id)`** — runs in the **IRQ kthread** (process context). Here you may sleep, take mutexes, do `GFP_KERNEL` allocation, talk to slow buses.

If you pass `handler == NULL`, the core installs a **default primary handler** (`irq_default_primary_handler`) that simply returns `IRQ_WAKE_THREAD` — i.e. "always run the thread". This is the common shape for pure-threaded drivers, and it is exactly why `IRQF_ONESHOT` is then required (next section).

---

## 3. `IRQF_ONESHOT` — keep the line masked until the thread finishes

The race `IRQF_ONESHOT` solves: with a level-triggered line and a NULL primary handler, the device keeps the line asserted until the thread services it. Without masking, the line would re-interrupt immediately and forever before the thread ever runs.

`IRQF_ONESHOT` makes the flow handler (Topic 05 `handle_fasteoi_irq`/`handle_level_irq`) **mask the line when the hardirq fires and leave it masked** through the whole thread, unmasking only when the thread completes:

```c
static irqreturn_t irq_thread_fn(struct irq_desc *desc, struct irqaction *action)
{
    irqreturn_t ret = action->thread_fn(action->irq, action->dev_id);   /* YOUR thread_fn */
    if (ret == IRQ_HANDLED) atomic_inc(&desc->threads_handled);
    irq_finalize_oneshot(desc, action);   /* clears this action's bit in desc->threads_oneshot;
                                             when the last bit clears, UNMASK + EOI the line */
    return ret;
}
```

`desc->threads_oneshot` is a bitmask of in-flight oneshot threads (one bit per action via `action->thread_mask`); the line is unmasked only when *all* oneshot threads on a shared line have finished. **Rule:** a `thread_fn` with no primary handler *must* set `IRQF_ONESHOT` — the core enforces it. (`request_irq` even folds in `IRQF_COND_ONESHOT` so a shared sharer can demand it.)

`IRQCHIP_EOI_THREADED` (Topic 03/05) is the chip-side cooperation: the EOI itself is deferred to thread completion for controllers that need it.

---

## 4. The IRQ Thread (`irq_thread`)

Each threaded `irqaction` gets its **own kthread** (`action->thread`), created in `__setup_irq`. Its loop:

```c
static int irq_thread(void *data)
{
    struct irqaction *action = data;
    struct irq_desc *desc = irq_to_desc(action->irq);

    irq_thread_set_ready(desc, action);          /* IRQTF_READY */
    sched_set_fifo(current);                      /* SCHED_FIFO! (secondary → sched_set_fifo_secondary) */

    handler_fn = (forced) ? irq_forced_thread_fn : irq_thread_fn;
    /* register irq_thread_dtor as task work for clean teardown */

    while (!irq_wait_for_interrupt(desc, action)) {   /* sleep until IRQTF_RUNTHREAD set */
        irqreturn_t r = handler_fn(desc, action);     /* → your thread_fn (via irq_thread_fn) */
        if (r == IRQ_WAKE_THREAD) irq_wake_secondary(desc, action);
        wake_threads_waitq(desc);                     /* wake synchronize_irq() waiters */
    }
    return 0;
}
```

Important properties:
- **Scheduling:** the thread runs at **SCHED_FIFO** (real-time) by default (`sched_set_fifo`) so a threaded handler isn't starved by normal tasks. Its priority and affinity matter for latency tuning (Topic 14).
- **Wakeup path:** the primary handler's `IRQ_WAKE_THREAD` → `__irq_wake_thread` sets `IRQTF_RUNTHREAD` and wakes the thread (`kernel/irq/handle.c`). `irq_wake_thread(irq, dev_id)` lets a driver wake it manually.
- **Affinity:** `irq_thread_check_affinity` keeps the thread on the IRQ's CPUs (`IRQTF_AFFINITY`).
- **Teardown:** `free_irq` calls `synchronize_hardirq` then `kthread_stop`; `irq_thread_dtor` (task work) cleans up `IRQTF_RUNTHREAD`/oneshot bits so the line isn't left masked.
- **State bits (`IRQTF_*`, `internals.h`):** `RUNTHREAD` (please run), `WARNED` (already warned about WAKE_THREAD-without-thread_fn), `AFFINITY` (adjust affinity), `FORCED_THREAD` (this action was force-threaded), `READY`.

The **`secondary`** action (`action->secondary`) exists for forced threading: when a non-threaded driver's handler is force-threaded but *itself* returns `IRQ_WAKE_THREAD`, the secondary carries the would-be thread.

---

## 5. Forced Threading & PREEMPT_RT

### 5.1 `threadirqs` / `force_irqthreads_key`
Booting with **`threadirqs`** flips a static key (`setup_forced_irqthreads`, `early_param("threadirqs", …)`); thereafter the core force-threads eligible IRQs even if the driver only gave a top-half handler:

```c
DEFINE_STATIC_KEY_FALSE(force_irqthreads_key);          /* enabled by "threadirqs" */
/* in __setup_irq: thread it unless exempt */
if (new->flags & (IRQF_NO_THREAD | IRQF_PERCPU | IRQF_ONESHOT)) /* not forced */ ;
```

Exempt from forced threading: `IRQF_NO_THREAD` (e.g. timers — `IRQF_TIMER` includes `NO_THREAD`), `IRQF_PERCPU` (per-CPU lines, Topic 12), and already-oneshot threaded ones.

For a force-threaded action the handler runs via **`irq_forced_thread_fn`**, which re-creates the hardirq's implicit guarantees that the driver's handler assumed:

```c
static irqreturn_t irq_forced_thread_fn(struct irq_desc *desc, struct irqaction *action)
{
    local_bh_disable();                              /* a hardirq handler ran with BH disabled */
    if (!IS_ENABLED(CONFIG_PREEMPT_RT)) local_irq_disable();  /* and with IRQs off (non-RT) */
    ret = irq_thread_fn(desc, action);
    if (!IS_ENABLED(CONFIG_PREEMPT_RT)) local_irq_enable();
    local_bh_enable();
    return ret;
}
```

So a driver written for hardirq context still sees "BH disabled, IRQs off" semantics even when force-threaded — that's the compatibility trick.

### 5.2 PREEMPT_RT
On PREEMPT_RT, forced threading is essentially always on: nearly all IRQ handlers run in threads so they are **preemptible** and can take RT mutexes (`sleeping spinlocks`). Note the `irq_forced_thread_fn` above does **not** disable IRQs on RT (it only disables BH) — because on RT you *want* the handler preemptible. The few survivors that stay in true hardirq context: `IRQF_NO_THREAD` lines, per-CPU interrupts, and the timer. Softirqs also move to task context on RT (Topic 08 §6, Topic 14).

---

## 6. Choosing: top-half / threaded / workqueue

| Need | Use |
|---|---|
| Tiny, no sleep, lowest latency | **top half only** (`request_irq`) |
| Must sleep / slow bus / longish, want RT priority & per-IRQ thread | **threaded IRQ** (`request_threaded_irq` + `IRQF_ONESHOT`) |
| Must sleep, work is occasional / batchable, no per-IRQ thread needed | **workqueue** from the top half (Topic 09) |
| Hard/softirq/NMI context callback | **irq_work** (Topic 09) |
| High event rate, want polling | **NAPI / irq_poll** (Topic 09) |

Threaded IRQ vs workqueue: a threaded IRQ gives you a *dedicated, RT-priority, IRQ-affined* thread and automatic line-masking (`ONESHOT`); a workqueue shares worker pools and has no masking semantics. Prefer threaded IRQ when the work is intrinsic to servicing the line; prefer a workqueue for occasional follow-up that can tolerate scheduling latency.

---

## 7. Observability & Pitfalls

### Observability
- The IRQ thread shows up as a kernel task named like **`irq/<N>-<name>`** in `ps`/`top` — watch its CPU% and priority.
- `/proc/interrupts` still counts the *hardirq*; the thread's work is invisible there — use the `irq:irq_handler_entry` tracepoint plus scheduler tracing for thread time.
- `synchronize_irq` waits on `desc->wait_for_threads`; `threads_active`/`threads_handled` track in-flight/handled.

### Pitfalls
- **`thread_fn` without `IRQF_ONESHOT` on a level line** — re-interrupt storm; the core warns/enforces, but custom flows can still get it wrong.
- **Doing the device-ack in `thread_fn` instead of the primary handler** — on a level line the IRQ keeps firing until acked; ack in the primary (or rely on ONESHOT masking).
- **Assuming the thread runs immediately** — it's SCHED_FIFO but still scheduled; under load there is wake-to-run latency. Latency-critical work that truly can't wait belongs in the top half.
- **Sharing data between primary and thread without care** — they run in different contexts on possibly different CPUs; use proper locking (Topic 10), not "IRQs are off so I'm safe".
- **Force-threading a handler that busy-waits** — a force-threaded handler is preemptible; long busy-waits there hurt more than help.
- **Returning `IRQ_WAKE_THREAD` with no `thread_fn`** — warned once (`IRQTF_WARNED`) and dropped.

### Further reading
- `Documentation/core-api/genericirq.rst` (threaded handler section); `Documentation/PREEMPT_RT...`.
- LWN: "Threaded interrupt handlers" and the RT softirq/threading articles.
- Source: `kernel/irq/manage.c` `irq_thread`/`irq_finalize_oneshot`; `kernel/irq/handle.c` `__irq_wake_thread`.

### Mental model
> A threaded IRQ splits the handler in two: a hardirq primary that acks and says `IRQ_WAKE_THREAD`, and a SCHED_FIFO kthread (`irq/N-name`) that does the sleepable work in process context. `IRQF_ONESHOT` keeps the line masked from the hardirq until the thread finishes. `threadirqs`/PREEMPT_RT force this model onto almost every IRQ so handlers become preemptible.

---

**Next: Topic 08 — Bottom Halves I: Softirqs** (the lowest-level deferral mechanism, the substrate tasklets and timers ride on, and the other place "deferred work" runs on the hardirq tail).
