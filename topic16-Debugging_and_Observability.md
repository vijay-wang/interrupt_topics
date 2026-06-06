# Topic 16 — Debugging & Observability (practitioner topic)

> **Standing requirements check (R1/R2/R3):** R1 — a complete reference of the interrupt observability surface (procfs/sysfs/debugfs/tracepoints/latency tracers) plus a symptom→diagnosis playbook covering every failure mode named across Topics 01–15. R2 — placed second-to-last: it reuses *every* prior topic's "Observability" section, so it can only be complete now; it precedes the boot capstone. R3 — surfaces verified in this tree: `kernel/irq/debugfs.c` (`/sys/kernel/debug/irq/{irqs,domains}`), `include/trace/events/irq.h`, `kernel/irq/proc.c`, `include/linux/irq_sim.h`, the `irqhandler.duration_warn_us=` param. Re-read R1/R2/R3 in the plan.
>
> **Builds on:** all of Topics 01–15 — this is where their observability notes are consolidated and turned into diagnosis.
>
> **Position in the plan:** the practitioner reference. Topic 17 (init) is the final capstone.
>
> **Source:** `kernel/irq/proc.c`, `kernel/irq/debugfs.c`, `include/trace/events/irq.h`, `kernel/irq/spurious.c`, `kernel/irq/irq_sim.c`, `kernel/trace/` (irqsoff).

---

## 1. The procfs/sysfs Surface

### `/proc/interrupts` — the first thing to read
Columns: **virq** | per-CPU `kstat_irqs` counts | **chip name** | **hwirq** | **flow type** | **action name(s)**.
```
            CPU0       CPU1
  24:    1048576          0   PCI-MSI 524288-edge      nvme0q1
  30:        512        498   GICv3 30 Level           arch_timer
 LOC:     900123     901240   Local timer interrupts
 RES:       1203       1190   Rescheduling interrupts        ← IPIs (Topic 12)
```
Read it as a snapshot of the whole model (Topics 03–05): which chip, which hwirq, which flow handler, which driver, and the per-CPU distribution (Topic 13). The named block at the bottom is system IPIs/NMIs (Topic 12).

### Others
- **`/proc/softirqs`** — per-CPU counts per softirq vector (Topic 08); rising `NET_RX` + `ksoftirqd` = receive overload.
- **`/proc/stat`** — `intr`/`softirq` totals; per-CPU `irq`/`softirq` time with `CONFIG_IRQ_TIME_ACCOUNTING` (Topic 14).
- **`/proc/irq/N/`** — `smp_affinity[_list]`, `effective_affinity[_list]` (Topic 13), `node`, `spurious` (unhandled/last-unhandled, Topic 05), `<chip>` name.
- **`/proc/irq/default_smp_affinity`** — default mask for new IRQs.
- **`/sys/kernel/irq/N/`** — `chip_name`, `hwirq`, `name`, `type`, `actions`, `per_cpu_count`, `wakeup` (the `irq_desc` exposed, Topic 03).

---

## 2. The debugfs Surface (`CONFIG_GENERIC_IRQ_DEBUGFS`)

The richest view, created by `kernel/irq/debugfs.c` under **`/sys/kernel/debug/irq/`**:

- **`irqs/N`** — for one virq: the installed `handle_irq` (flow handler), `IRQD_*`/`IRQS_*` **state flags** (masked? disabled? affinity-managed? wakeup-armed?), the **chip**, **hwirq**, and the full **domain hierarchy** (each parent level's hwirq/chip — Topic 04/11). The single best place to inspect an IRQ live.
- **`domains/`** — one node per `irq_domain` plus `default`: name, size, mapped count, flags, and the **parent chain** (Topic 04 §6). Where you confirm an MSI mapping or spot vector-domain exhaustion (Topic 11/13).

You can also **write** to some debugfs irq nodes to inject/retrigger for testing (gated; see the file).

---

## 3. Tracepoints (`include/trace/events/irq.h`)

| Tracepoint | Fires | Fields |
|---|---|---|
| `irq:irq_handler_entry` | before each action handler (Topic 05 §3) | irq, name |
| `irq:irq_handler_exit` | after, with result | irq, ret (handled?) |
| `irq:softirq_raise` | a softirq is raised (Topic 08) | vec |
| `irq:softirq_entry` / `softirq_exit` | a softirq handler runs | vec |
| `irq:tasklet_entry` / `tasklet_exit` | a tasklet runs (Topic 09) | |
| `ipi:ipi_raise` / `ipi_entry` / `ipi_exit` | IPIs (Topic 12) | target/reason |

bpftrace recipes:
```sh
# per-IRQ rate
bpftrace -e 'tracepoint:irq:irq_handler_entry { @[args->irq, str(args->name)] = count(); }'
# per-handler duration (entry→exit)
bpftrace -e 'tracepoint:irq:irq_handler_entry { @s[tid]=nsecs }
             tracepoint:irq:irq_handler_exit /@s[tid]/ { @us[args->irq]=hist((nsecs-@s[tid])/1000); delete(@s[tid]); }'
# softirq time by vector
bpftrace -e 'tracepoint:irq:softirq_entry { @s[cpu]=nsecs }
             tracepoint:irq:softirq_exit /@s[cpu]/ { @[args->vec]=sum(nsecs-@s[cpu]); delete(@s[cpu]); }'
```

---

## 4. Latency Tracers & the Duration Check

- **`irqsoff` / `preemptirqsoff`** (`/sys/kernel/tracing/current_tracer`) — capture the longest IRQs-off / preempt-off region and the code path that caused it (Topic 14 §3). The tool for "what kept IRQs off for 800 µs".
- **`irqhandler.duration_warn_us=<N>`** boot param (this tree, Topic 14 §2) — warns with `%pS` when a top half exceeds N µs. Zero-effort hog detection.
- **`function_graph`** tracer filtered on `common_interrupt`/`gic_handle_irq`/a specific handler — see exactly where time goes.
- **`cyclictest`** (rt-tests) — end-to-end wakeup latency for RT validation.

---

## 5. Symptom → Diagnosis Playbook (R1: cover every failure mode)

| Symptom | Likely cause | How to confirm / fix |
|---|---|---|
| **`irq N: nobody cared (try ... irqpoll)`** then IRQ disabled | screaming line: wrong trigger type, or a shared handler returning wrong value (Topic 02/05) | check DT/ACPI trigger; audit shared handlers' `IRQ_NONE` logic; `/proc/irq/N/spurious` |
| **An IRQ shows count 0 in `/proc/interrupts`** | never firing: wrong hwirq mapping, wrong affinity, device not generating, masked | `/sys/kernel/debug/irq/irqs/N` (hwirq/domain right? masked?); verify `effective_affinity`; check the device |
| **IRQ lands only on CPU0 despite broad `smp_affinity`** | single-target `effective_affinity` (normal for many chips), or `irqbalance` off + no pinning | read `effective_affinity_list`; pin or enable managed affinity (Topic 13) |
| **Can't write `/proc/irq/N/smp_affinity` (`-EIO`)** | managed IRQ (multiqueue) — kernel-owned (Topic 13 §2) | expected; change the device's queue count instead |
| **`ksoftirqd/N` at ~100%** | softirq overload, usually NET_RX flood (Topic 08/14) | `/proc/softirqs` deltas; enable NAPI/affinity tuning, `ethtool -C` coalescing |
| **High IRQs-off latency / RT misses** | long handler or `spin_lock_irqsave` region (Topic 10/14) | `irqsoff` tracer; `irqhandler.duration_warn_us=`; move work to threaded IRQ/softirq |
| **x86 `vector allocation failed`** | per-CPU vector exhaustion (Topic 02/11/13) | `/sys/kernel/debug/irq/domains/VECTOR`; reduce MSI-X vectors; use managed affinity |
| **Device dead only after a suspend/resume cycle** | controller state not restored, or lost wakeup (Topic 15) | check chip `irq_suspend/resume`; `enable_irq_wake`; `/sys/kernel/debug/wakeup_sources` |
| **System won't wake from sleep on a device** | missing `enable_irq_wake` or `/sys/.../power/wakeup` disabled (Topic 15) | enable both; confirm `IRQD_WAKEUP_ARMED` in debugfs |
| **`scheduling while atomic` / sleeping-in-atomic BUG** | sleeping in hardirq/softirq (Topic 10) | `CONFIG_DEBUG_ATOMIC_SLEEP` stack; move to threaded/workqueue |
| **lockdep `inconsistent {HARDIRQ-ON-W}`** | lock taken in IRQ ctx without `_irqsave` (Topic 10) | use `spin_lock_irqsave`/`_bh` per the two stacks lockdep prints |
| **High "Function call interrupts" / IPIs** | TLB-shootdown/`smp_call_function` storm (Topic 12/14) | trace `ipi:*`; usually an mm/scalability issue |
| **Threaded handler seems slow** | wake-to-run latency under load (Topic 07) | check `irq/N-name` thread priority/CPU; consider top-half work |

---

## 6. Testing Without Hardware: `irq_sim` (`kernel/irq/irq_sim.c`)

For writing IRQ-consumer tests, the **interrupt simulator** creates a software `irq_domain` whose IRQs you can fire programmatically:
```c
struct irq_domain *d = devm_irq_domain_create_sim(dev, fwnode, num_irqs);
int virq = irq_create_mapping(d, hwirq);   /* request_irq on it like normal */
irq_sim_fire(d, hwirq);                     /* trigger the handler from test code */
```
Used by GPIO and other subsystems' selftests. `kernel/irq/irq_test.c` exercises the core. This lets you validate handler/threading/affinity logic in CI without a device.

---

## 7. Common Pitfalls (in debugging itself)

- **Reading `/proc/interrupts` once** — it's cumulative; take *deltas* over a second to see rate.
- **Trusting `smp_affinity` for "where it runs"** — use `effective_affinity` (Topic 13).
- **Forgetting counts are per *handled* event** — coalescing/threading/sharing change what a count means.
- **Not enabling `CONFIG_GENERIC_IRQ_DEBUGFS`** — then the richest view (`/sys/kernel/debug/irq/`) is missing; build debug kernels with it.
- **Confusing spurious (unhandled) with high-rate (real)** — `/proc/irq/N/spurious` distinguishes; the fixes differ (trigger/sharing vs mitigation/NAPI).
- **Profiling without `CONFIG_IRQ_TIME_ACCOUNTING`** — interrupt time hides in "system"; turn it on.

---

## 8. Further Reading & Source Pointers

### Documentation
- `Documentation/core-api/genericirq.rst` (debugging section), `Documentation/admin-guide/lockup-watchdogs.rst`.
- `Documentation/trace/ftrace.rst` (irqsoff), `Documentation/admin-guide/perf/`.

### Source to read
- `kernel/irq/debugfs.c` — what `/sys/kernel/debug/irq/` exposes (and the state-flag decoding).
- `kernel/irq/spurious.c` — the "nobody cared" logic.
- `kernel/irq/proc.c` — how `/proc/interrupts` and `/proc/irq/*` are built.

### Mental model
> `/proc/interrupts` tells you *what fires and where*; `/sys/kernel/debug/irq/` tells you *the full state and hierarchy* of one IRQ; tracepoints tell you *when and how long*; `irqsoff`/`duration_warn_us` tell you *what hurts latency*. Match the symptom to the surface: zero count → debugfs mapping; "nobody cared" → trigger/sharing; ksoftirqd hot → softirq/NAPI; latency → irqsoff; can't wake → wakeup_sources.

---

**Next: Topic 17 — Initialization Walk-through** (the capstone: every prior topic re-threaded onto the boot timeline, from the first IDT/vector install to the first device interrupt).
