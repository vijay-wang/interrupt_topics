# Topic 08 — Bottom Halves I: Softirqs

> **Standing requirements check (R1/R2/R3):** R1 — the whole softirq mechanism: the fixed vector list, `open_softirq`/`raise_softirq`, the `__do_softirq`/`handle_softirqs` restart loop and its budget, where softirqs run (hardirq tail, `local_bh_enable`, `ksoftirqd`), the `preempt_count` bookkeeping, and the PREEMPT_RT change. R2 — first of the bottom-half topics, placed after the handler topics (06/07) because softirqs are what a top half *defers to*, and before tasklets (Topic 09) which are literally built on a softirq vector. R3 — quoted from `kernel/softirq.c` in `v7.1-rc6`. Re-read R1/R2/R3 in the plan.
>
> **Builds on:** Topic 00 (softirq context, `preempt_count`), Topic 01 (`irq_exit`/`irq_exit_rcu` — the hardirq tail), Topic 05/07 (the top half decides what to defer).
>
> **Position in the plan:** the lowest-level deferred-work engine. Topic 09 (tasklets/BH-workqueue/irq_work) sits on top; Topic 10 formalizes the context/locking rules; Topic 14 covers softirq-driven latency and the RT model.
>
> **Source:** `kernel/softirq.c` (`handle_softirqs`, `__do_softirq`, `invoke_softirq`, `raise_softirq`, `open_softirq`, `ksoftirqd`), `include/linux/interrupt.h` (the vector enum, `local_bh_*`).

---

## 1. The Model & Position

A **softirq** is a fixed, statically-defined deferred-work slot that runs in **softirq (bottom-half) context**: interrupts *enabled*, but still atomic — **no sleeping**, no blocking allocation. It is the seam right after the hardirq: the top half does the urgent ack and `raise_softirq(NR)`; on the way out of the interrupt the softirq runs the bulk work with IRQs back on, so it doesn't extend the IRQs-off window.

Why fixed and few? Softirqs are the *highest-performance, lowest-level* deferral — used by the hottest subsystems (networking, block, timers, RCU). They are statically numbered (no dynamic registration), run lockless per-CPU, and are carefully bounded so they can't starve the system. Everything more flexible (tasklets, work) is built on top.

```
top half (hardirq) ── raise_softirq(NET_RX_SOFTIRQ) ──► sets a per-CPU pending bit
   │
   └─ irq_exit_rcu() ── invoke_softirq() ──► __do_softirq() runs pending vectors (IRQs on, atomic)
                                              └─ if too long → wake ksoftirqd to finish
```

---

## 2. The Vector List (`include/linux/interrupt.h`)

A small fixed enum, **priority-ordered** (lower number runs first within one `__do_softirq` pass):

```c
enum {
    HI_SOFTIRQ = 0,    /* high-priority tasklets */
    TIMER_SOFTIRQ,     /* timer wheel callbacks */
    NET_TX_SOFTIRQ,    /* network transmit */
    NET_RX_SOFTIRQ,    /* network receive (NAPI) — usually the hottest */
    BLOCK_SOFTIRQ,     /* block I/O completion */
    IRQ_POLL_SOFTIRQ,  /* irq_poll (Topic 09) */
    TASKLET_SOFTIRQ,   /* normal tasklets (Topic 09) */
    SCHED_SOFTIRQ,     /* scheduler load-balancing */
    HRTIMER_SOFTIRQ,   /* high-resolution timers */
    RCU_SOFTIRQ,       /* RCU callbacks — kept last on purpose */
    NR_SOFTIRQS
};
```

Note the consumers: **TIMER/HRTIMER** (the tick and hrtimers), **NET_RX/NET_TX** (the network stack's softirq engine — NAPI polls in NET_RX), **BLOCK** (I/O completion), **RCU** (grace-period callbacks), **TASKLET/HI** (the substrate for tasklets, Topic 09). The order matters: timers before networking before RCU. `/proc/softirqs` has exactly these columns.

Registration is static, once at init:
```c
void open_softirq(int nr, void (*action)(void));   /* e.g. open_softirq(NET_RX_SOFTIRQ, net_rx_action) */
```
The handler takes **no arguments** in this tree (`h->action()`); per-CPU state lives elsewhere (e.g. `softnet_data`).

---

## 3. Raising and Running

### Raise
```c
void raise_softirq(unsigned int nr);          /* sets the per-CPU pending bit (IRQ-safe wrapper) */
void raise_softirq_irqoff(unsigned int nr);   /* same, caller already has IRQs off */
```
Raising just sets a bit in the per-CPU `local_softirq_pending()` mask and, if not already in interrupt/softirq context, wakes `ksoftirqd`. It does **not** run anything itself.

### Run — the hardirq tail
On interrupt exit, `irq_exit_rcu()` → `invoke_softirq()` → `__do_softirq()` → **`handle_softirqs()`** when there is pending work and we're not nested:

```c
static void handle_softirqs(bool ksirqd)
{
    unsigned long end = jiffies + MAX_SOFTIRQ_TIME;   /* 2 ms budget */
    int max_restart = MAX_SOFTIRQ_RESTART;            /* 10 passes */
    __u32 pending = local_softirq_pending();

    softirq_handle_begin();              /* add SOFTIRQ_OFFSET to preempt_count → in_serving_softirq() */
    account_softirq_enter(current);
restart:
    set_softirq_pending(0);              /* clear the mask; new raises accumulate for next pass */
    local_irq_enable();                  /* SOFTIRQS RUN WITH IRQs ON */
    while ((softirq_bit = ffs(pending))) {
        h = &softirq_vec[vec_nr];
        kstat_incr_softirqs_this_cpu(vec_nr);
        trace_softirq_entry(vec_nr);
        h->action();                     /* the registered handler, e.g. net_rx_action */
        trace_softirq_exit(vec_nr);
        /* WARN if the handler leaked preempt_count */
        pending >>= softirq_bit;
    }
    local_irq_disable();
    pending = local_softirq_pending();   /* did handlers raise more? */
    if (pending) {
        if (time_before(jiffies, end) && !need_resched() && --max_restart)
            goto restart;                /* keep going, within budget */
        wakeup_softirqd();               /* budget exhausted → punt the rest to ksoftirqd */
    }
    account_softirq_exit(current);
    softirq_handle_end();                /* remove SOFTIRQ_OFFSET */
}
```

The key design points are all visible here:
- **IRQs are enabled** while a softirq handler runs (only the pending-mask manipulation is done IRQs-off). So a softirq does *not* extend the hardirq's IRQs-off latency.
- **Bounded work:** at most `MAX_SOFTIRQ_RESTART` (10) re-passes or `MAX_SOFTIRQ_TIME` (2 ms), and it bails early on `need_resched()`. Beyond that the remaining work is handed to **`ksoftirqd`** so softirqs can't monopolize a CPU and starve user space.
- **Per-CPU & serialized:** softirqs of the same vector can run concurrently on *different* CPUs, but a CPU runs its softirqs single-threaded; the handler must do its own per-CPU/cross-CPU locking.
- **`PF_MEMALLOC` is masked** while borrowing the current task's context, so a network softirq doesn't accidentally dip into emergency reserves.

---

## 4. `local_bh_disable` / `local_bh_enable` & `preempt_count`

Code that shares data with a softirq protects it by disabling bottom halves:

```c
local_bh_disable();   /* __local_bh_disable_ip: adds SOFTIRQ_DISABLE_OFFSET (= 2*SOFTIRQ_OFFSET) */
   ... critical section vs softirqs ...
local_bh_enable();    /* on the dec-to-zero, runs any softirqs that became pending */
```

The softirq sub-count in `preempt_count` does double duty (see the `SOFTIRQ_OFFSET` comment in `kernel/softirq.c`):
- **`SOFTIRQ_OFFSET`** is added while a softirq is *being served* → `in_serving_softirq()` is true.
- **`SOFTIRQ_DISABLE_OFFSET` (= 2×)** is added by `local_bh_disable()` → `in_softirq()` (i.e. `softirq_count()`) is true, meaning "BH disabled or serving".

That's why `in_softirq()` and `in_serving_softirq()` differ (Topic 00 §4, Topic 10): the former means "softirqs can't run here", the latter means "a softirq handler is actually executing". Crucially, **`local_bh_enable()` is a softirq-run point** — finishing a BH-disabled region flushes pending softirqs.

---

## 5. ksoftirqd

One per-CPU thread (`run_ksoftirqd`, registered via `smpboot` as `ksoftirqd/<cpu>`):

```c
static int ksoftirqd_should_run(unsigned int cpu) { return local_softirq_pending(); }
static void run_ksoftirqd(unsigned int cpu) { ... handle_softirqs(true); ... }
```

It exists to drain softirq work that the hardirq tail couldn't finish within budget (§3). Under sustained load (e.g. a packet flood), softirq processing migrates from "inline on the IRQ tail" to ksoftirqd, which is a **schedulable** entity — so the scheduler can balance softirq CPU against user tasks instead of letting NET_RX starve everything. Seeing `ksoftirqd` near 100% is the classic "softirq overload" signal (Topic 14/16).

---

## 6. PREEMPT_RT

On PREEMPT_RT softirqs do **not** run on the hardirq tail in atomic context. Instead:
- `local_bh_disable()` becomes a **per-CPU lock** (a sleeping lock), so a "BH-disabled" region is preemptible and can be migrated.
- Softirq handlers run in **task context** (typically ksoftirqd or the raising thread), making them preemptible and allowing them to take RT mutexes.
- This is the softirq half of the RT story whose IRQ half was Topic 07 (threaded handlers). Topic 14 ties them together.

The non-RT invariant "softirqs are atomic, can't sleep" still holds for the configs most people run; just know RT relaxes it by moving them to threads.

---

## 7. Observability & Pitfalls

### Observability
- **`/proc/softirqs`** — per-CPU counts per vector. Rising `NET_RX` with high `ksoftirqd` CPU = receive overload.
- **`/proc/stat`** `softirq` line — total softirq count.
- Tracepoints **`irq:softirq_entry` / `irq:softirq_exit` / `irq:softirq_raise`** (with the vector). `bpftrace -e 'tracepoint:irq:softirq_entry { @[args->vec] = count(); }'`.
- `ksoftirqd/<cpu>` CPU% in `top`.

### Pitfalls
- **Sleeping in a softirq** — forbidden (atomic context); `might_sleep()`/lockdep will scream. Use a workqueue (Topic 09) if you must sleep.
- **Forgetting softirqs run with IRQs on** — data shared between a hardirq handler and a softirq needs `spin_lock_irqsave` (Topic 10), not just `spin_lock_bh`.
- **`spin_lock_bh` vs `spin_lock_irqsave`** — `_bh` protects against softirqs on the *same* CPU; if a *hardirq* also touches the data you need `_irqsave`. Mixing them up is a classic deadlock/race.
- **Long softirq handlers** — they push work to `ksoftirqd` and add latency; NAPI/`irq_poll` (Topic 09) exist to bound NET_RX. Don't add unbounded loops to a softirq.
- **Raising a softirq from the wrong context** — `raise_softirq` is IRQ-safe; `raise_softirq_irqoff` assumes IRQs already off.

### Further reading
- `Documentation/core-api/local_ops.rst`, the networking NAPI docs (`Documentation/networking/napi.rst`) for the busiest softirq.
- LWN: "Software interrupts and realtime", "Reducing softirq latency".
- Source: read `handle_softirqs` once top-to-bottom; then `kernel/softirq.c` `__local_bh_disable_ip`/`__local_bh_enable_ip`.

### Mental model
> A softirq is a fixed, per-CPU, atomic deferred slot. The top half sets a pending bit (`raise_softirq`); the hardirq tail (or `local_bh_enable`, or `ksoftirqd`) runs the handler with IRQs *on* but no sleeping, bounded to ~2 ms / 10 passes before punting to `ksoftirqd`. It is the high-performance substrate under networking, timers, block, RCU — and under tasklets, which Topic 09 builds next.

---

**Next: Topic 09 — Bottom Halves II: Tasklets, Workqueues, irq_work** (the higher-level deferral built on softirqs, plus the full "which mechanism do I pick" matrix).
