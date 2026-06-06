# Topic 14 — Time Accounting, Latency, and PREEMPT_RT

> **Standing requirements check (R1/R2/R3):** R1 — how IRQ/softirq time is measured and exported, the concrete sources of interrupt latency and the tools that find them (including this tree's `irqhandler.duration_warn_us=`), interrupt-storm handling, and the *complete* PREEMPT_RT model that Topics 07/08/10 introduced piecewise. R2 — placed after placement (Topic 13): latency is the temporal consequence of everything prior, so it can finally be discussed with all mechanisms known. R3 — `CONFIG_IRQ_TIME_ACCOUNTING`/`irqtime_account_irq` in `kernel/sched/cputime.c`, the duration check in `kernel/irq/handle.c`, verified in `v7.1-rc6`. Re-read R1/R2/R3 in the plan.
>
> **Builds on:** Topic 01 (entry/exit, IRQ stacks), Topic 07 (threaded IRQs, RT threading), Topic 08 (softirqs, ksoftirqd, RT softirqs), Topic 10 (IRQs-off regions, contexts), Topic 13 (affinity/isolation).
>
> **Position in the plan:** the performance/RT topic. Topic 16 (debugging) reuses these tools for diagnosis; this topic explains *what* they measure and *why* latency happens.
>
> **Source:** `kernel/sched/cputime.c` (`irqtime_account_irq`), `kernel/irq/handle.c` (duration check), `kernel/softirq.c` (ksoftirqd), `kernel/trace/` (irqsoff tracer), `Documentation/...` RT.

---

## 1. IRQ Time Accounting (`kernel/sched/cputime.c`)

By default the tick samples whatever context it lands in, which mis-attributes interrupt time. With **`CONFIG_IRQ_TIME_ACCOUNTING`**, the kernel measures hardirq/softirq time precisely:

```c
void irqtime_account_irq(struct task_struct *curr, unsigned int offset)  /* called on each context enter/exit */
```
- It reads a fine clock (`sched_clock`) on every hardirq/softirq boundary and attributes the delta to the **hardirq** or **softirq** bucket, subtracting it from the interrupted task's runtime.
- Exposed in **`/proc/stat`** as the per-CPU `irq` and `softirq` columns, and aggregated in the `intr`/`softirq` totals; `top`/`mpstat` show `%irq` and `%soft`.
- The generic entry layer (Topic 01) drives the enter/exit hooks (`vtime_account_hardirq`/`vtime_account_softirq`), so the accounting is consistent across arches.

Without it, heavy interrupt load shows up as inflated "system" or even "user" time on the unlucky task; with it, you can see a core spending, say, 30% in softirq (a NET_RX flood) directly.

---

## 2. Where Interrupt Latency Comes From

"Interrupt latency" has two meanings — both matter:

1. **Time to *handle* an interrupt** (device asserts → handler runs). Inflated by:
   - **IRQs-off regions** (Topic 10): any `local_irq_disable`/`spin_lock_irqsave` region delays *all* interrupts on that CPU until it ends. The single biggest cause.
   - **Long top halves** (Topic 06): a handler that loops keeps IRQs off and blocks every other IRQ behind it.
   - **Higher-priority interrupts / softirq floods** monopolizing the CPU.
   - **Cross-CPU contention** on a `spin_lock_irqsave` the handler needs.
2. **Time for the *deferred* work to run** (handler → softirq/thread completes). Inflated by:
   - **Softirq budget exhaustion → ksoftirqd** scheduling delay (Topic 8 §5).
   - **Threaded-IRQ wake-to-run latency** (SCHED_FIFO but still scheduled; Topic 7 §4).
   - Preemption-disabled regions delaying the scheduler.

### This tree's built-in handler-duration warning
`kernel/irq/handle.c` has an opt-in check that flags overlong top halves:
```c
__setup("irqhandler.duration_warn_us=", irqhandler_duration_check_setup);   /* boot param */
/* in __handle_irq_event_percpu: time each handler, warn if it exceeds the threshold */
```
Boot with `irqhandler.duration_warn_us=200` and any handler taking >200 µs prints a warning with the offending `%pS`. A zero-config way to catch a hog handler in production-ish kernels.

---

## 3. The Latency Tools

| Tool | Measures | Use |
|---|---|---|
| **`irqsoff` tracer** (`/sys/kernel/tracing/`) | longest contiguous IRQs-off region + the code path | find who keeps IRQs off too long |
| **`preemptoff` / `preemptirqsoff`** | longest preempt-off / combined region | RT preemption latency |
| **`irqhandler.duration_warn_us=`** | per-handler top-half time | catch a slow `request_irq` handler |
| **`/proc/interrupts` + `/proc/softirqs` deltas** | interrupt/softirq *rate* | spot the busiest line/vector |
| **`/proc/stat` `irq`/`softirq`** | CPU time in interrupt context | quantify load |
| **`perf sched` / `cyclictest`** | scheduling/wakeup latency | RT validation |
| **`ftrace` function_graph on the entry path** | exactly where time goes | drill into a hot handler |

`cyclictest` (rt-tests) is the canonical end-to-end "how late can a wakeup be" measurement for RT tuning. The `irqsoff` tracer is the canonical "what disabled IRQs for 800 µs" finder.

---

## 4. Interrupt Storms

A **storm** is an IRQ firing far faster than it can be usefully serviced, starving the CPU:

- **Screaming level IRQ** (stuck-asserted line, often a wrong trigger type, Topic 02/05): the genirq **spurious detector** disables it after ~100k unhandled events with `irq N: nobody cared` (Topic 05 §5) — the safety net.
- **High-rate legitimate IRQ** (a NIC under a packet flood): not spurious, but each interrupt costs more than it returns. The fix is **interrupt mitigation**: hardware coalescing (the device batches), and **polling** — NAPI for networking, `irq_poll` for block (Topic 09 §5) — switch from per-event interrupts to budgeted polling under load. `ksoftirqd` at 100% is the symptom.
- **IPI storms** (Topic 12): excessive TLB shootdowns / `smp_call_function` show as high "Function call interrupts"; usually an mm or scalability issue, not the IRQ layer.

---

## 5. PREEMPT_RT — the Full Picture

The RT changes were introduced piecewise (threaded IRQs in Topic 07, task-context softirqs in Topic 08, lock changes in Topic 10); here they are as one model. RT's goal: make the kernel *preemptible almost everywhere* so a high-priority task's wakeup latency is bounded.

- **IRQ handlers are forced-threaded** (Topic 07 §5): nearly every `request_irq` handler runs in a SCHED_FIFO `irq/N-name` thread, preemptible, able to take RT mutexes. Survivors in true hardirq: `IRQF_NO_THREAD` (the timer), per-CPU IRQs (Topic 12), and the few lines marked no-thread.
- **Softirqs run in task context** (Topic 08 §6): no atomic hardirq-tail processing; `local_bh_disable` becomes a per-CPU sleeping lock; softirq handlers can be preempted by higher-priority RT tasks.
- **Spinlocks become sleeping locks** (Topic 10 §7): `spin_lock` → an RT mutex with priority inheritance; only `raw_spinlock_t` stays a true spin (hence `desc->lock` is raw). Priority inheritance prevents unbounded priority inversion.
- **`irq_work`** can be threaded too, except `IRQ_WORK_HARD_IRQ` (Topic 9 §4) which forces real hardirq context.
- **Threaded-IRQ priorities** become a tuning surface: you set the `irq/N` thread's RT priority relative to your application threads so the *right* interrupts preempt the *right* work.

Net effect: on RT, the longest non-preemptible region shrinks from "a long IRQ handler + softirq flood" to "a few `raw_spinlock`/hardirq-only paths", which is what lets `cyclictest` show double-digit-microsecond worst cases.

---

## 6. Tuning (the knobs)

- **`threadirqs`** (Topic 07) — force-thread IRQs even on non-RT, to move handler work into schedulable, priority-tunable threads.
- **CPU isolation** — `isolcpus=`, `nohz_full=`, `rcu_nocbs=` keep timers/RCU/scheduler ticks off chosen cores; combine with **IRQ affinity** (Topic 13: pin IRQs *away* from isolated cores, disable `irqbalance`) so latency-critical threads run uninterrupted.
- **Affinity placement** — put a device's IRQ on the same NUMA node/CPU as the thread that consumes its data (locality); spread multiqueue IRQs (managed affinity, Topic 13).
- **Interrupt coalescing** — `ethtool -C` for NICs, device-specific for storage; trade latency for fewer interrupts under throughput load.
- **Threaded-IRQ RT priority** — `chrt` on the `irq/N-name` thread (RT kernels).
- **`irqhandler.duration_warn_us=`** — leave a generous threshold on to catch regressions.
- **Softirq tuning** — mostly indirect (NAPI budgets, `net.core.netdev_budget`); watch `ksoftirqd`.

---

## 7. Observability & Pitfalls

### Observability (recap, for this topic's lens)
- `/proc/interrupts`, `/proc/softirqs` rate deltas; `/proc/stat` `irq`/`softirq`; `mpstat -P ALL 1` `%irq`/`%soft`.
- `irqsoff`/`preemptirqsoff` tracers; `cyclictest`; the `irqhandler.duration_warn_us=` warnings.
- `irq/N-name` threads in `top` (RT/`threadirqs`); `ksoftirqd/N` CPU%.

### Pitfalls
- **Blaming "system CPU"** without `CONFIG_IRQ_TIME_ACCOUNTING` — turn it on to separate hardirq/softirq from real system time.
- **Long `spin_lock_irqsave` regions** — the stealth latency source; `irqsoff` finds them. Shorten or restructure.
- **Disabling `irqbalance` but not pinning** — IRQs pile on CPU0; isolation without affinity placement is half a fix.
- **Assuming `threadirqs`/RT removes all latency** — `raw_spinlock`/hardirq-only paths remain; measure with `cyclictest`.
- **Coalescing too aggressively** — fewer interrupts but higher per-event latency; wrong for latency-bound workloads.
- **Ignoring ksoftirqd** — sustained ksoftirqd CPU is a softirq-overload signal demanding NAPI/affinity work, not more handler optimization.

### Further reading
- `Documentation/timers/no_hz.rst`, `Documentation/admin-guide/kernel-per-CPU-kthreads.rst`, the PREEMPT_RT docs.
- LWN: the "Realtime" series, "Softirqs and realtime", "Per-CPU kthread isolation".
- Tools: `rt-tests` (`cyclictest`), `tuned` profiles (`latency-performance`, `realtime`).

### Mental model
> Interrupt latency = how long IRQs/preemption stay off (top-half + `spin_lock_irqsave` regions) plus how long deferred work waits to be scheduled (ksoftirqd / threaded-IRQ wakeup). `CONFIG_IRQ_TIME_ACCOUNTING` measures the time; `irqsoff`/`cyclictest`/`irqhandler.duration_warn_us=` localize the cause. PREEMPT_RT slashes the non-preemptible window by threading handlers and softirqs and making spinlocks sleep — leaving only `raw_spinlock`/hardirq paths atomic.

---

**Next: Topic 15 — Suspend/Resume, Wakeup, and IRQ Power Management** (what happens to interrupts across system sleep, and how a device wakes the machine).
