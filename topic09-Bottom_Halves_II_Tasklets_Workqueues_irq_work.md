# Topic 09 — Bottom Halves II: Tasklets, Workqueues, irq_work

> **Standing requirements check (R1/R2/R3):** R1 — covers tasklets (struct + semantics + why discouraged), workqueues *as an interrupt consumer* (incl. the BH workqueue that replaces tasklets), `irq_work` (hardirq/NMI-safe callbacks), `irq_poll` (NAPI-style), and a complete decision matrix across *all* deferral mechanisms in the series. R2 — second bottom-half topic, on top of softirqs (Topic 08: tasklets ride `TASKLET_SOFTIRQ`/`HI_SOFTIRQ`, the BH workqueue runs in softirq context) and after threaded IRQs (Topic 07), so the matrix can compare every option the reader has now seen. R3 — structs/APIs from `include/linux/interrupt.h`, `include/linux/workqueue.h`, `include/linux/irq_work*.h`, `include/linux/irq_poll.h` in `v7.1-rc6`. Re-read R1/R2/R3 in the plan.
>
> **Builds on:** Topic 07 (threaded IRQ as a deferral option), Topic 08 (softirqs, `local_bh_*`, the vector list).
>
> **Position in the plan:** completes the "where does the work go?" picture. Topic 10 then formalizes the context/locking rules that govern all of these.
>
> **Source:** `kernel/softirq.c` (tasklets), `kernel/workqueue.c` (`WQ_BH`/`system_bh_wq`), `kernel/irq_work.c`, `lib/irq_poll.c`; headers as above.

---

## 1. Position: Four Ways to Defer (plus threaded IRQ)

A top half (Topic 06) can hand off the rest of the work to:

```
                       context        may sleep?   built on
  threaded IRQ  ────►  process (kthread, SCHED_FIFO)   YES     Topic 07
  workqueue     ────►  process (worker pool)           YES     kernel/workqueue.c
  BH workqueue  ────►  softirq                          NO     WQ_BH (softirq) — tasklet replacement
  tasklet       ────►  softirq (TASKLET/HI_SOFTIRQ)      NO     Topic 08
  irq_work      ────►  hardirq (self-IPI)               NO     kernel/irq_work.c (callable from NMI)
  irq_poll      ────►  softirq (IRQ_POLL_SOFTIRQ)        NO     NAPI-style polling
```

This topic covers the four non-threaded ones; the decision matrix in §6 unifies all.

---

## 2. Tasklets (`include/linux/interrupt.h`, `kernel/softirq.c`)

A **tasklet** is a deferred callback that runs in **softirq context** on the `TASKLET_SOFTIRQ` (normal) or `HI_SOFTIRQ` (high-priority) vector, with one guarantee softirqs themselves don't give: **a tasklet never runs concurrently with itself**, even across CPUs.

```c
struct tasklet_struct {
    struct tasklet_struct *next;       /* per-CPU pending list link */
    unsigned long         state;       /* TASKLET_STATE_SCHED / TASKLET_STATE_RUN */
    atomic_t              count;       /* != 0 ⇒ disabled */
    bool                  use_callback;
    union {
        void (*func)(unsigned long data);       /* old style */
        void (*callback)(struct tasklet_struct *t);  /* new style (preferred) */
    };
    unsigned long         data;
};
```

API:
```c
DECLARE_TASKLET(name, callback);                 /* static, enabled */
tasklet_setup(t, callback);                       /* dynamic init (new callback style) */
tasklet_schedule(t);    tasklet_hi_schedule(t);   /* queue on TASKLET_/HI_SOFTIRQ */
tasklet_disable(t);     tasklet_enable(t);        /* counted disable; _disable waits for a running one */
tasklet_kill(t);                                  /* wait until it can't run again; for teardown */
```

How it runs: `tasklet_schedule` links `t` onto the per-CPU tasklet list and `raise_softirq(TASKLET_SOFTIRQ)`. When the softirq runs (Topic 08), `tasklet_action` walks the list; `tasklet_trylock` enforces the no-concurrent-self rule (`TASKLET_STATE_RUN`); `count == 0` is the enabled check.

### Why tasklets are discouraged for new code
- They run in softirq context (no sleeping) yet have **worse latency** than a plain softirq handler (extra list/lock dance) and **less flexibility** than a workqueue.
- The serialization guarantee is often misunderstood and rarely actually needed.
- The kernel is **migrating tasklet users to the BH workqueue** (§3) or to threaded IRQs. New code should pick one of those. (Tasklets are not removed — much existing code still uses them — but treat them as legacy.)

---

## 3. Workqueues as an Interrupt Consumer (`kernel/workqueue.c`)

A **workqueue** runs a `work_struct` callback in **process context** on a worker thread, so the callback **may sleep** (mutex, blocking allocation, slow bus). From an interrupt handler you defer like this:

```c
INIT_WORK(&my_work, my_work_fn);
schedule_work(&my_work);            /* or queue_work(wq, &my_work) */
```

The deep workqueue internals (pools, `max_active`, `WQ_UNBOUND`, concurrency management) are their own subsystem and **out of scope here** — the point for *interrupts* is: a workqueue is the answer when the deferred work must sleep but doesn't justify a dedicated threaded-IRQ kthread, or is occasional follow-up.

### The BH workqueue — the modern tasklet replacement
This tree has **`WQ_BH`** workqueues (`include/linux/workqueue.h`): a workqueue whose work runs in **softirq (bottom-half) context**, giving the work-API ergonomics with tasklet-like timing:

```c
WQ_BH = 1 << 0,    /* execute in bottom half (softirq) context */
extern struct workqueue_struct *system_bh_wq;
extern struct workqueue_struct *system_bh_highpri_wq;
```

So `queue_work(system_bh_wq, &work)` is the recommended replacement for `tasklet_schedule` — same softirq context and no-sleep rule, but the richer, better-understood workqueue interface (flush/cancel/named pools) instead of the bespoke tasklet machinery. New "softirq-context deferral" should use this rather than a tasklet.

---

## 4. `irq_work` — callbacks from hardirq/NMI context (`kernel/irq_work.c`)

The other mechanisms can't be *queued* from the most restricted contexts. **`irq_work`** can: it lets you schedule a callback to run in **hardirq context**, and crucially you can queue it **from NMI or hardirq context** safely (it uses a lockless per-CPU list and a self-IPI).

```c
struct irq_work { struct __call_single_node node; void (*func)(struct irq_work *); struct rcuwait irqwait; };
DEFINE_IRQ_WORK(name, func);
bool irq_work_queue(struct irq_work *work);            /* run on this CPU, soon (self-IPI) */
bool irq_work_queue_on(struct irq_work *work, int cpu);/* run on a specific CPU (IPI) */
```

Flags (`IRQ_WORK_INIT_*`):
- **default** — run in hardirq context via a self-IPI as soon as possible.
- **`IRQ_WORK_LAZY`** — run on the next timer tick (batch, lower overhead) instead of an immediate IPI.
- **`IRQ_WORK_HARD_IRQ`** — force true hardirq context even on PREEMPT_RT (where normal irq_work may be threaded).

Who uses it and why: **perf** (NMI handler needs to wake up the ring-buffer consumer — can't do that in NMI, so it queues irq_work), **printk** (defer console work out of NMI/hardirq), **RCU**, **scheduler**. It is the *only* safe way to escalate from NMI to "ordinary hardirq" work. Topic 12 (NMI) leans on it.

---

## 5. `irq_poll` — NAPI-style polled completion (`lib/irq_poll.c`)

For very high interrupt rates, taking an interrupt per event costs more than it saves (Topic 00 §1). **`irq_poll`** is a generic NAPI-style mechanism (block/storage analog of network NAPI): on the first interrupt, disable the device's IRQ and **poll** completions in the `IRQ_POLL_SOFTIRQ` softirq with a budget; re-enable the IRQ when the queue drains.

```c
struct irq_poll { ...; irq_poll_fn *poll; ...; };
void irq_poll_init(struct irq_poll *, int weight, irq_poll_fn *);
void irq_poll_sched(struct irq_poll *);      /* from the top half: switch to polling */
void irq_poll_complete(struct irq_poll *);   /* poll drained: re-arm the IRQ */
void irq_poll_enable/disable(struct irq_poll *);
```

Used by some block/storage drivers (the network stack has its own NAPI). The lesson: at scale, *interrupt mitigation* (coalescing + polling) is part of the interrupt subsystem, not a workaround.

---

## 6. The Decision Matrix (R1: pick correctly)

| Mechanism | Context | May sleep? | Latency | Concurrency | Queue-from-NMI? | Use when |
|---|---|---|---|---|---|---|
| **top half** | hardirq | no | lowest | per-CPU serial | n/a | tiny, urgent, no-sleep work |
| **irq_work** | hardirq (self-IPI) | no | low | per-CPU | **yes** | escalate NMI→hardirq; perf/printk |
| **softirq** (own vector) | softirq | no | low | per-vector across CPUs | no | hot, high-rate subsystem work (net/block/timer) — static only |
| **tasklet** *(legacy)* | softirq | no | low-med | never self-concurrent | no | existing code; prefer BH-wq for new |
| **BH workqueue** | softirq | no | low-med | per-work | no | **new** softirq-context deferral (tasklet replacement) |
| **threaded IRQ** | process (SCHED_FIFO kthread) | **yes** | med | per-action | no | sleepable work intrinsic to the line; RT |
| **workqueue** | process (worker) | **yes** | med-high | per-work | no | occasional sleepable follow-up |
| **irq_poll / NAPI** | softirq | no | amortized | per-poller | no | very high event rate; mitigate IRQ storms |

Quick rules:
- **Must sleep?** → threaded IRQ or workqueue.
- **Queued from NMI/hardirq?** → irq_work (only option).
- **New softirq-context deferral?** → BH workqueue, not a tasklet.
- **High event rate?** → NAPI/irq_poll.
- **Hot, static, fixed subsystem?** → a real softirq vector (you won't add new ones — those are reserved).

---

## 7. Observability & Pitfalls

### Observability
- Tasklets/BH-wq run on softirq vectors → `/proc/softirqs` (`TASKLET`, `HI`) and `irq:softirq_*` tracepoints (Topic 08).
- Tasklet tracepoints: `irq:tasklet_entry`/`irq:tasklet_exit` (if enabled).
- Workqueues: `workqueue:workqueue_*` tracepoints; `/sys/devices/virtual/workqueue/`.
- irq_work has no dedicated counters; trace via the queuing subsystem (perf/printk).

### Pitfalls
- **Sleeping in a tasklet / BH-workqueue / irq_work** — all run in atomic (softirq/hardirq) context; sleeping is a bug. Need to sleep → threaded IRQ or normal workqueue.
- **Using a tasklet in new code** — prefer the BH workqueue; tasklets are legacy.
- **`schedule_work` for latency-critical paths** — worker scheduling latency is unbounded under load; use a threaded IRQ if you need priority.
- **Calling `tasklet_kill`/`flush_work` from atomic context** — they can block; teardown belongs in process context.
- **irq_work `func` that does too much** — it runs in hardirq context; keep it minimal (its whole purpose is a *small* escalation out of NMI).
- **Re-arming irq_poll incorrectly** — forgetting `irq_poll_complete` leaves the device IRQ disabled (silent stall).

### Further reading
- `Documentation/core-api/workqueue.rst` (incl. BH workqueues), `Documentation/networking/napi.rst`.
- LWN: "The BH workqueue", "Tasklets considered harmful", "irq_work".
- Source: `kernel/softirq.c` (`tasklet_action_common`), `kernel/irq_work.c`, `lib/irq_poll.c`.

### Mental model
> After the top half, the work goes somewhere: **irq_work** if you're escalating out of NMI; a **softirq** if you're a hot static subsystem; a **BH workqueue** for new softirq-context deferral (tasklets are the legacy form); a **threaded IRQ** or **workqueue** if it must sleep; **NAPI/irq_poll** if the rate is so high you should poll. Topic 10 next gives the locking rules that keep all of these from racing each other.

---

**Next: Topic 10 — Contexts, Masking, Atomicity & `preempt_count`** (the concurrency rules that make every mechanism in Topics 05–09 correct).
