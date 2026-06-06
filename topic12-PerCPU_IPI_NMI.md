# Topic 12 — Per-CPU Interrupts, IPIs, and NMIs

> **Standing requirements check (R1/R2/R3):** R1 — all three non-wire delivery modes: per-CPU interrupts (`request_percpu_irq`, `handle_percpu_devid_irq`), IPIs (the genirq IPI layer, `smp_call_function`, x86 vectors / arm64 SGIs), and NMIs (both the genirq `request_nmi` path and x86's architectural `register_nmi_handler`, with the hard constraints). R2 — after MSI (Topic 11) as the second "advanced delivery" topic; relies on per-CPU handling (Topic 05 `handle_percpu_devid_irq`), the chip `ipi_send_*` ops (Topic 03), and the context rules (Topic 10, esp. the NMI row). R3 — verified against `kernel/irq/ipi.c`, `kernel/irq/manage.c`, `arch/x86/include/asm/irq_vectors.h`, `arch/x86/include/asm/nmi.h` in `v7.1-rc6`. Re-read R1/R2/R3 in the plan.
>
> **Builds on:** Topic 03 (`irq_chip.ipi_send_single/mask`, `irq_nmi_setup`), Topic 05 (`handle_percpu_devid_irq`, `handle_fasteoi_nmi`), Topic 06 (`request_percpu_irq`, `request_nmi`), Topic 10 (the NMI/atomic rules).
>
> **Source:** `kernel/irq/ipi.c`, `kernel/irq/ipi-mux.c`, `kernel/irq/manage.c` (percpu + nmi request), `kernel/smp.c` (`smp_call_function`), `arch/x86/include/asm/{irq_vectors.h,nmi.h}`, `arch/x86/kernel/nmi.c`, `drivers/irqchip/irq-gic-v3.c` (SGIs, pseudo-NMI).

---

## 1. Position: Three Deliveries That Don't Fit "device line → one CPU"

| Mode | Source | Target | Genirq handling |
|---|---|---|---|
| **per-CPU IRQ** | a line *private to each CPU* (arch timer, PMU) | the local CPU only | `handle_percpu_devid_irq`, per-CPU `dev_id` |
| **IPI** | *another CPU* | one/many CPUs | `ipi_send_*`, SGIs (arm64) / ICR (x86) |
| **NMI** | special line / perf / watchdog | a CPU, **unmaskable** | NMI entry, `request_nmi` / `register_nmi_handler` |

All three reuse the machinery of Topics 01–10 but bend one assumption each: per-CPU lines aren't global, IPIs originate from software on a peer CPU, and NMIs can't be masked by `local_irq_disable`.

---

## 2. Per-CPU Interrupts

Some interrupt lines exist *once per CPU* and only ever fire on the CPU they belong to: the **architected timer** (the tick), the **PMU** (perf counters), and GIC **PPIs** generally. Treating them as global would be wrong (affinity is meaningless; the `dev_id` and enable state are per-CPU).

API (`kernel/irq/manage.c`):
```c
int request_percpu_irq(unsigned int irq, irq_handler_t handler,
                       const char *devname, void __percpu *percpu_dev_id);
void enable_percpu_irq(unsigned int irq, unsigned int type);   /* must run ON each CPU */
void disable_percpu_irq(unsigned int irq);
void free_percpu_irq(unsigned int irq, void __percpu *percpu_dev_id);
```

Key differences from an ordinary `request_irq` (Topic 06):
- **`dev_id` is `void __percpu *`** — each CPU sees its own cookie. The handler gets the *local* CPU's instance.
- **Enable is per-CPU.** One `request_percpu_irq` registers the action, but `enable_percpu_irq` must be called *on every CPU* that should receive it (typically from a CPU-hotplug startup callback, Topic 13/17).
- **Flow handler is `handle_percpu_devid_irq`** (Topic 05 §2.4): no `desc->lock` (there's no cross-CPU contention on a per-CPU line), minimal ack/eoi.
- **`IRQF_PERCPU`** semantics; these lines are exempt from forced threading (Topic 07) and from affinity/balancing (Topic 13).

There is also `request_percpu_nmi` for a per-CPU NMI (e.g. the PMU NMI on arm64 pseudo-NMI), paired with `prepare_percpu_nmi`/`enable_percpu_nmi`.

---

## 3. IPIs (Inter-Processor Interrupts)

An **IPI** is a CPU telling another CPU "do something now". They underpin TLB shootdowns, rescheduling, function calls on remote CPUs, and timer broadcast.

### 3.1 The genirq IPI layer (`kernel/irq/ipi.c`)
Controllers that expose IPIs as IRQs (notably GIC SGIs) register them, and the core provides:
```c
int irq_reserve_ipi(struct irq_domain *domain, const struct cpumask *dest);  /* allocate an IPI virq range */
int __ipi_send_single(struct irq_desc *desc, unsigned int cpu);
int __ipi_send_mask(struct irq_desc *desc, const struct cpumask *dest);
/* delivered via the chip: */ chip->ipi_send_single(data, cpu) / chip->ipi_send_mask(data, mask);
```
So sending an IPI is "call the chip's `ipi_send_*`", which on arm64 writes a GIC **SGI** register targeting the destination CPUs; the receiving CPU takes it like any IRQ and runs the registered handler. `ipi-mux.c` multiplexes several logical IPIs onto one hardware IPI line for controllers with few SGIs.

### 3.2 The high-level users (`kernel/smp.c`)
Most kernel code doesn't call the IPI layer directly; it uses:
```c
smp_call_function_single(cpu, func, info, wait);   /* run func() on one remote CPU */
smp_call_function_many(mask, func, info, wait);    /* on a set of CPUs */
on_each_cpu(func, info, wait);                      /* everywhere */
smp_send_reschedule(cpu);                           /* poke the scheduler on a CPU */
```
These build a `call_single_data` work item and fire a **call-function IPI**; the remote CPU's IPI handler runs the queued `func`. `irq_work_queue_on` (Topic 09) also uses an IPI to run a callback on a target CPU.

### 3.3 The hardware vectors
- **x86** uses fixed high vectors (`arch/x86/include/asm/irq_vectors.h`): `RESCHEDULE_VECTOR (0xfd)`, `CALL_FUNCTION_VECTOR (0xfc)`, `CALL_FUNCTION_SINGLE_VECTOR (0xfb)`, `REBOOT_VECTOR (0xf8)`, plus TLB and others. A CPU sends one by writing its Local APIC **ICR** (Topic 02 §2.2). These appear as `DEFINE_IDTENTRY_SYSVEC` entries (Topic 01 §3.2), *not* as `vector_irq[]`/`irq_desc` device IRQs — that's why they "have their own entry points" (the comment in `common_interrupt`).
- **arm64** uses GIC **SGIs** (0–15), sent via the GIC and delivered through the normal `gic_handle_irq` path → an IPI `irq_desc`.

This is why `/proc/interrupts` shows IPIs differently on the two arches (x86: a separate block of named system IPIs at the bottom; arm64: SGI rows).

---

## 4. NMIs (Non-Maskable Interrupts)

An **NMI** cannot be blocked by `local_irq_disable()` — it fires even inside IRQs-off critical sections. That makes it the tool for watching a hung or interrupts-disabled CPU, but it imposes brutal constraints.

### 4.1 Where NMIs are used
- **Hardlockup / watchdog detector** — an NMI periodically checks the CPU is still making progress (catches a CPU wedged with IRQs off).
- **perf** — sampling counters often deliver via NMI so they can profile IRQs-off code; the NMI handler queues `irq_work` (Topic 09 §4) to do anything non-NMI-safe.
- **`nmi_panic` / KGDB / serial-console magic** — debugging a dead system.
- **MCE-adjacent / firmware** NMIs.

### 4.2 Two registration paths (don't confuse them)
- **Architectural NMI on x86** (`arch/x86/include/asm/nmi.h`): the x86 NMI is *not* a normal IRQ line, so it has its own registry:
  ```c
  register_nmi_handler(type, handler, flags, name);   /* type: NMI_LOCAL, NMI_UNKNOWN, NMI_SERR, NMI_IO_CHECK */
  unregister_nmi_handler(type, name);
  ```
  Handlers are chained per type; `arch/x86/kernel/nmi.c` dispatches. Returns `NMI_HANDLED`/`NMI_DONE`.
- **Genirq `request_nmi`** (`kernel/irq/manage.c`, Topic 06): for controllers that can deliver a *regular IRQ line as an NMI* — chiefly **arm64 pseudo-NMI** (GIC priorities, `CONFIG_ARM64_PSEUDO_NMI`, Topic 01/02). The chip implements `irq_nmi_setup`/`irq_nmi_teardown` (Topic 03), and the flow handler is `handle_fasteoi_nmi` (Topic 05 §2.4). `request_percpu_nmi` is the per-CPU form.

### 4.3 The constraints (Topic 10's NMI row, in force)
Inside an NMI handler:
- **No sleeping, no normal locks.** A normal `spin_lock` might be held by the very code the NMI interrupted on this CPU → instant deadlock. Use only **lockless** algorithms or `raw_spin_trylock`.
- **NMI-safe primitives only.** NMI-safe RCU (`rcu_nmi_enter` via `irqentry_nmi_enter`, Topic 01 §5/Topic 10 §5); `this_cpu_*`; `cmpxchg`. `printk` is made NMI-safe via deferral; most else is not.
- **Can't be masked, can nest carefully.** `in_nmi()` is true; the NMI entry (`irqentry_nmi_enter`) avoids the RCU/lockdep ops that could themselves deadlock.
- **Escalate, don't do work.** The pattern (perf, printk) is: do the minimal NMI-safe capture, then `irq_work_queue()` to finish in ordinary hardirq context (Topic 09 §4). This is *the* reason `irq_work` exists.

---

## 5. Design Rationale

- **Why per-CPU IRQs at all?** A timer/PMU line is physically per-CPU; modeling it globally would force pointless locking and meaningless affinity. The per-CPU `dev_id` + lockless `handle_percpu_devid_irq` matches the hardware.
- **Why a genirq IPI layer instead of arch-only code?** So generic code (`smp_call_function`, `irq_work`) and non-x86 controllers share one model; the chip's `ipi_send_*` is the only arch-specific piece.
- **Why NMIs despite the pain?** They are the *only* way to interrupt a CPU that has disabled IRQs — indispensable for lockup detection and profiling IRQs-off code. The constraints are the price.
- **Why `irq_work` is the NMI companion** — it's the sole safe escalation from "can't touch anything" (NMI) to "ordinary hardirq" where real work is allowed.

---

## 6. Observability & Pitfalls

### Observability
- **`/proc/interrupts`** bottom block: `Rescheduling interrupts`, `Function call interrupts`, `TLB shootdowns`, `Local timer interrupts`, plus `Non-maskable interrupts` and per-CPU counts. On arm64, IPIs are the SGI rows.
- **`/proc/interrupts` `NMI` line** and `nmi` in `/proc/stat`; perf's NMI overhead shows here.
- Tracepoints: `ipi:ipi_raise`, `ipi:ipi_entry`/`ipi_exit` (where enabled).
- Watchdog: `/proc/sys/kernel/{nmi_watchdog,hardlockup_panic}`; dmesg "NMI watchdog" messages.

### Pitfalls
- **Calling `enable_percpu_irq` on only one CPU** — the line then fires only there; you must enable on each CPU (hotplug callback).
- **Using a normal lock in an NMI handler** — deadlock against the interrupted code. Lockless or `trylock` only; escalate via `irq_work`.
- **Heavy work in an IPI handler** — IPIs run in hardirq context on the target; a slow `smp_call_function` callback stalls the remote CPU. Keep it tiny; `wait=1` blocks the sender too.
- **Confusing x86 `register_nmi_handler` with genirq `request_nmi`** — different subsystems; x86 architectural NMI is not an `irq_desc`.
- **Assuming NMIs respect `local_irq_disable`** — they don't; data shared with an NMI handler needs NMI-safe access, not just IRQ masking.
- **IPI storms** — excessive TLB shootdowns or `smp_call_function` (e.g. frequent global flushes) show as high "Function call interrupts"; a real scalability problem.

### Further reading
- `Documentation/core-api/genericirq.rst` (IPI/percpu sections); `Documentation/admin-guide/lockup-watchdogs.rst`.
- LWN: "IPI virtualization", "NMI handlers and the kernel", "irq_work".
- Source: `kernel/smp.c` (`smp_call_function*`), `kernel/irq/ipi.c`, `arch/x86/kernel/nmi.c`, `drivers/irqchip/irq-gic-v3.c` (`gic_send_sgi`, pseudo-NMI).

### Mental model
> Per-CPU IRQs are lines private to each core (timer/PMU) handled lockless with a per-CPU `dev_id`. IPIs are software-sent CPU-to-CPU interrupts — `smp_call_function`/`smp_send_reschedule`/`irq_work` on top of the chip's `ipi_send_*` (GIC SGIs / x86 ICR vectors). NMIs are unmaskable and run under near-total restriction; the safe pattern is "capture minimally, then `irq_work` to finish in ordinary hardirq".

---

**Next: Topic 13 — Affinity, Balancing, CPU Hotplug & the Matrix Allocator** (where interrupts land, who decides, and what happens when CPUs come and go).
