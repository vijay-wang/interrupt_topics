# Linux Interrupt Subsystem — Study Plan (v1, English, Comprehensive)

> Target kernel: **this source tree, `v7.1-rc6`** (`VERSION=7 PATCHLEVEL=1`), x86_64 and arm64 as the two reference architectures. Every claim in this series is to be verified against the code actually in this checkout (`kernel/irq/`, `kernel/softirq.c`, `arch/*/kernel/`, `drivers/irqchip/`, `include/linux/irq*.h`, `include/linux/interrupt.h`).

---

## STANDING REQUIREMENTS (re-read before writing or studying every topic)

These three rules are the contract for this whole series. **Every topic document must restate them in its header and be checked against them.**

1. **R1 — Knowledge must be comprehensive.** A topic must cover its subject end-to-end: every relevant data structure (with the purpose of each field that matters), every key code path, the *why* behind the design, the arch-specific realizations (at least x86_64 **and** arm64), the user/observability surface, and the common failure modes. No hand-waving over a mechanism the reader will actually meet in the source.

2. **R2 — The plan must be easy to learn; ordering must be reasonable.** Strict **bottom-up** progression: hardware → CPU entry → kernel abstraction → flow → driver API → deferred work → advanced delivery → system management → debugging → boot-time capstone. Never depend on a concept that a later topic introduces; when a forward reference is unavoidable, name it explicitly and give a one-line intuition. Each topic builds only on earlier ones.

3. **R3 — Use the kernel source freely.** All structures, function names, flags, and call chains are to be read out of *this* tree, not from memory. Branches/tags/grep are fair game. When a fact matters, the document cites the file (`path:symbol`) so the reader can confirm it. If something is arch- or config-dependent, say so.

> **How to use these:** at the top of each topic doc there is a "Standing requirements check" line confirming the topic was written against R1/R2/R3. If you (future author or reader) find a topic that violates one, that is a bug to fix.

---

## Ordering principles (the concrete shape of R2)

1. **Bottom-up**: what the *hardware* does before what the *kernel* does; the *mechanism* before its *consumers*; the *common core* before *specialized deliveries* (MSI, IPI, NMI).
2. **Core data model early**: master `irq_desc` / `irq_data` / `irq_chip` / `irqaction` / `irq_domain` before any flow or API topic, because every later topic refers back to them.
3. **Deferred work right after the top-half API**: you cannot reason about what a handler may do until you know where the *rest* of the work goes (softirq / tasklet / threaded IRQ / workqueue / irq_work).
4. **Concurrency & contexts after bottom halves**: `preempt_count`, `local_irq_disable`, and the hardirq/softirq/NMI context rules only make sense once softirqs exist.
5. **System-management and debugging last**, then a boot-time **initialization walk-through** that re-threads every prior topic onto the timeline — the capstone.
6. **Uniform 8-step format** (Appendix B) so the reader always knows where to look.

**Total: 18 topics (00 → 17) + appendices.**

---

## Topic 00 — Preliminaries & Whole-System Overview

After this you can draw the whole path "device asserts a wire → a C function runs → deferred work completes" and place every later topic on it.

- **0.1 What an interrupt is (and is not)**
  - Asynchronous external interrupt vs synchronous exception/trap/fault vs syscall vs `int3`/software interrupt
  - The taxonomy in Linux terms: IRQ, exception, NMI, IPI, MSI
  - Why interrupts exist: polling vs interrupt-driven I/O
- **0.2 The end-to-end path (the spine of the series)**
  - device → interrupt controller → CPU pin/vector → arch low-level entry → generic entry (`irqentry_enter`) → `generic_handle_irq` → flow handler → `irqaction` chain (top half) → bottom half (softirq/tasklet/threaded/workqueue) → return-from-interrupt
- **0.3 Two number namespaces you must never confuse**
  - **hwirq** — the controller-local hardware number
  - **Linux IRQ number / virq** — the kernel-global token, allocated by the IRQ domain layer
  - Where the translation lives (forward ref: Topic 04)
- **0.4 Execution contexts**
  - hardirq (interrupt) context, softirq/bottom-half context, process/task context, NMI context
  - What is and isn't allowed in each (sleeping, allocation, locks) — the short version; deep dive in Topic 10
  - `preempt_count` as the one word that encodes "where am I" (intro only)
- **0.5 Architecture realizations at a glance**
  - x86_64: IDT + vectors (0–255), Local/IO-APIC, `idtentry`
  - arm64: exception vector table, exception levels, GIC
  - Why Linux hides both behind `irq_chip` + `irq_domain`
- **0.6 Source-tree geography**
  - `kernel/irq/` (genirq core), `kernel/softirq.c`, `kernel/irq_work.c`
  - `arch/x86/kernel/{irq.c,apic/,idt.c}`, `arch/arm64/kernel/{irq.c,entry.S}`
  - `drivers/irqchip/`, `include/linux/irq*.h`, `include/linux/interrupt.h`
- **0.7 Glossary & the map** to keep returning to

---

## Topic 01 — CPU-Level Interrupt Mechanics (the architecture entry)

What the silicon does, and the lowest layer of kernel code that catches it. This is the floor everything stands on.

- **1.1 What the CPU does on an interrupt**
  - Finish/abort the current instruction, mask further interrupts, save minimal state, vector to a handler
  - Privilege transition (user→kernel), stack switch
- **1.2 x86_64 specifics**
  - The **IDT**: 256 gates; interrupt gates vs trap gates; DPL
  - Vectors: exception vectors (0–31), the device-vector range, special vectors (spurious, error, IPIs)
  - `IST` (Interrupt Stack Table) for NMI/#DF/#MC; per-CPU IRQ stacks
  - The `idtentry` macro family (`arch/x86/include/asm/idtentry.h`) and `DEFINE_IDTENTRY*`
  - Error code, `CR2` for page faults (contrast: that's an exception, not an IRQ)
- **1.3 arm64 specifics**
  - The exception **vector table** (`arch/arm64/kernel/entry.S`, `vectors`), 16 entries (EL/SP/sync-vs-irq)
  - Exception levels EL0/EL1; `ESR_EL1`, `FAR_EL1`
  - The IRQ/FIQ distinction
- **1.4 The generic entry/exit layer**
  - `include/linux/irq-entry-common.h`, `irqentry_enter` / `irqentry_exit` / `irqentry_state_t`
  - Context tracking (user/kernel/idle), RCU-on-IRQ, lockdep IRQ state, `need_resched` on exit
  - `irqentry_nmi_enter/exit`
- **1.5 IRQ stacks and stack switching**
  - Why a dedicated IRQ stack; `run_on_irqstack_cond` (x86), arm64 IRQ stack
- **1.6 Return from interrupt**
  - Resched check, signal delivery on return-to-user, preemption on return-to-kernel
- **1.7 Observability:** `/proc/interrupts` per-vector view; arch counters

---

## Topic 02 — Interrupt Controllers (the routing hardware)

The chips between the device and the CPU. Linux models all of them as `irq_chip`s, but you must know the real hardware to read the drivers.

- **2.1 What an interrupt controller does**
  - Aggregate many device lines, prioritize, mask, deliver to a CPU, acknowledge/EOI
  - Trigger types: level vs edge, polarity
- **2.2 x86 controllers**
  - Legacy **8259A PIC** (cascaded pair, 15 lines) — mostly historical, still emulated
  - **Local APIC** (per-CPU): LVT, timer, IPIs, EOI register; **x2APIC**
  - **I/O APIC**: redirection table, routing external lines to vectors
  - The vector space as a scarce resource (forward ref: matrix allocator, Topic 13)
- **2.3 arm/arm64 controllers**
  - **GIC** family: GICv2 (distributor + CPU interface), GICv3/v4 (redistributors, system registers, **ITS** for MSI, LPIs), SGIs/PPIs/SPIs
  - SoC-specific secondary controllers; chaining
- **2.4 Cascaded / chained controllers**
  - One IRQ line feeding another controller; `irq_set_chained_handler`
  - Contrast chained vs `irqaction`-based demux
- **2.5 The driver glue**
  - `drivers/irqchip/`, `IRQCHIP_DECLARE`, `irqchip_init`, ACPI vs DT probing
- **2.6 Observability:** controller identity in `/proc/interrupts`, `/sys/kernel/irq/`

---

## Topic 03 — The genirq Core Data Model

The four structures that the entire generic IRQ layer is built from. Internalize these before any flow or API topic.

- **3.1 `struct irq_desc`** (`include/linux/irqdesc.h`)
  - `irq_data`, `handle_irq` (the flow handler), `action` (the handler list), `status_use_accessors`, `depth`, `irq_count`, `irqs_unhandled`, `lock`, `kstat_irqs` (per-CPU counts), `affinity_hint`, `threads_active`, `wait_for_threads`, name
- **3.2 `struct irq_data`** (`include/linux/irq.h`)
  - `irq` (virq), `hwirq`, `chip`, `chip_data`, `domain`, `parent_data` (hierarchy), `common` (`irq_common_data`: affinity, effective_affinity, msi_desc, handler_data, node, state flags)
  - Why `irq_data` is the chip-facing view and `irq_desc` the core-facing view
- **3.3 `struct irq_chip`** (`include/linux/irq.h`)
  - The controller-driver vtable: `irq_mask` / `irq_unmask` / `irq_mask_ack` / `irq_ack` / `irq_eoi`, `irq_set_type`, `irq_set_affinity`, `irq_set_wake`, `irq_retrigger`, `irq_compose_msi_msg` / `irq_write_msi_msg`, `irq_request_resources`, `flags` (`IRQCHIP_*`)
  - `irq_chip_generic` / generic chip helpers (`kernel/irq/generic-chip.c`)
- **3.4 `struct irqaction`** (`include/linux/interrupt.h`)
  - `handler`, `thread_fn`, `thread`, `secondary`, `irq`, `flags`, `name`, `dev_id`, `percpu_dev_id`, `next` (shared-IRQ chain), `thread_flags`, `thread_mask`
- **3.5 Status & settings flags**
  - `IRQD_*` (state on `irq_data`), `IRQS_*` (internal desc state), `_IRQ_*`/`IRQ_*` settings, the accessor discipline (`irqd_*`), `settings.h`
- **3.6 Sparse vs legacy IRQ storage**
  - `CONFIG_SPARSE_IRQ`, the maple/radix lookup, `irq_to_desc`, `irq_desc_tree`, allocation via `irq_alloc_descs`
- **3.7 Per-IRQ statistics:** `kstat_irqs`, how `/proc/interrupts` is built

---

## Topic 04 — IRQ Domains: hwirq → virq Translation

How a hardware number on a specific controller becomes a kernel-global IRQ. The glue between Topic 02 and Topic 03.

- **4.1 The problem**
  - Multiple controllers, overlapping hwirq numbers, dynamic discovery → need a mapping and on-demand allocation
- **4.2 `struct irq_domain` & `irq_domain_ops`** (`include/linux/irqdomain.h`)
  - `ops` (`map`/`unmap`/`xlate`, and `alloc`/`free`/`translate` for hierarchy), `host_data`, `hwirq_max`, `revmap`, `fwnode`, parent
  - Mapping kinds: linear, tree, no-map; `irq_domain_add_*` / `irq_domain_create_*`
- **4.3 Creating and using mappings**
  - `irq_create_mapping`, `irq_find_mapping`, `irq_create_fwspec_mapping`, `irq_dispose_mapping`
  - `irq_of_parse_and_map` (DT), ACPI GSI path
- **4.4 Hierarchical IRQ domains**
  - Why: MSI and stacked controllers need a *chain* of domains (e.g. PCI-MSI → vector domain; GIC-ITS → GIC)
  - `irq_domain_alloc_irqs`, `irq_domain_set_hwirq_and_chip`, `parent_data` walking, `irq_chip_*_parent` helpers
- **4.5 firmware interface**
  - `fwnode_handle`, `struct irq_fwspec`, DT `#interrupt-cells`, ACPI
- **4.6 Observability:** `/sys/kernel/debug/irq/domains/*`

---

## Topic 05 — Flow Handlers: How One IRQ Is Processed

The heart of genirq: the per-trigger-type state machine that sits between "the controller delivered an IRQ" and "your handler runs".

- **5.1 Entry into the core**
  - `generic_handle_irq` / `generic_handle_irq_safe` / `generic_handle_domain_irq` / `handle_irq_desc`
  - From arch entry (Topic 01) to here
- **5.2 The flow handlers** (`kernel/irq/chip.c`)
  - `handle_simple_irq`, `handle_level_irq`, `handle_edge_irq`, `handle_fasteoi_irq`, `handle_edge_eoi_irq`, `handle_percpu_irq`, `handle_percpu_devid_irq`, `handle_fasteoi_nmi`
  - For each: the exact `mask`/`ack`/`eoi`/`unmask` ordering and *why* it matches that trigger type
- **5.3 Running the handlers**
  - `handle_irq_event` / `handle_irq_event_percpu` → walk the `action` list → `irqreturn_t`
  - `IRQ_HANDLED` / `IRQ_NONE` / `IRQ_WAKE_THREAD`
- **5.4 Shared interrupts**
  - `IRQF_SHARED`, the action chain, how "is it mine?" is decided, the cost
- **5.5 Masking states & deferred actions**
  - `IRQS_PENDING`, `irq_may_run`, masking during handling, `IRQCHIP_EOI_THREADED`
- **5.6 Spurious / unhandled detection**
  - `note_interrupt`, `irqs_unhandled`, the "nobody cared / disabling IRQ" path (`kernel/irq/spurious.c`)
- **5.7 Resend**
  - `check_irq_resend`, `kernel/irq/resend.c` (lost edge re-injection)

---

## Topic 06 — Registering & Managing Handlers (the driver API)

The surface 99% of driver authors touch. Now that you know the machinery, the API reads naturally.

- **6.1 Requesting**
  - `request_irq`, `request_threaded_irq`, `request_any_context_irq`, `request_nmi`, `request_percpu_irq`, `request_percpu_nmi`
  - `devm_request_irq` / `devm_request_threaded_irq` (resource-managed)
  - What `__setup_irq` does under the hood (chip setup, shared-IRQ checks, thread creation)
- **6.2 The `IRQF_*` flags** (`include/linux/interrupt.h`)
  - `IRQF_SHARED`, `IRQF_ONESHOT`, `IRQF_NO_THREAD`, `IRQF_PERCPU`, `IRQF_NOBALANCING`, `IRQF_TRIGGER_*`, `IRQF_NO_SUSPEND`, `IRQF_EARLY_RESUME`, `IRQF_NO_AUTOEN`, `IRQF_COND_SUSPEND`, `IRQF_IRQPOLL`
- **6.3 The top-half handler contract**
  - Runs with the IRQ (often) masked, in hardirq context: no sleeping, no `mutex`, no blocking allocation; keep it short
  - `irqreturn_t` semantics; returning `IRQ_WAKE_THREAD`
- **6.4 Releasing & gating**
  - `free_irq`, `disable_irq` (sync) vs `disable_irq_nosync`, `enable_irq`, `synchronize_irq`, the `depth` counter
  - `irq_set_irq_type`, `irq_set_affinity`, `irq_get_irqchip_state`/`set`
- **6.5 Pitfalls**
  - `dev_id` uniqueness for shared/free, the disable/enable balance, requesting before the device is quiesced

---

## Topic 07 — Threaded IRQs & Forced Threading

Moving handler work into a kernel thread — the default model under PREEMPT_RT and increasingly common everywhere.

- **7.1 Why thread an IRQ**
  - Long handlers, code that must sleep, RT determinism, priority
- **7.2 The split**
  - Primary handler (hardirq, decides `IRQ_WAKE_THREAD`) + `thread_fn` (the kthread)
  - `IRQF_ONESHOT` and why it's required when there's no primary handler (line stays masked until the thread completes)
- **7.3 The IRQ thread**
  - `irq_thread`, `irq_thread_fn`, `irq_forced_thread_fn`, per-action kthread, `thread_mask`, `IRQTF_*`
  - `wake_threads`/`irq_wake_thread`, `synchronize_hardirq` vs `synchronize_irq`
- **7.4 Forced threading**
  - `threadirqs` boot param, `CONFIG_IRQ_FORCED_THREADING`, what is exempt (`IRQF_NO_THREAD`, percpu, timer)
- **7.5 PREEMPT_RT**
  - Almost everything threaded; the few hardirq-context survivors; relation to softirq changes (forward ref: Topics 08, 14)
- **7.6 Choosing:** top-half-only vs threaded vs workqueue (decision guide)

---

## Topic 08 — Bottom Halves I: Softirqs

The oldest and lowest-level deferred-work mechanism; the substrate tasklets and timers ride on.

- **8.1 The model**
  - A fixed, small set of softirq vectors run with interrupts enabled but in atomic (bottom-half) context
  - `open_softirq`, `raise_softirq`, `raise_softirq_irqoff`, `__raise_softirq_irqoff`
- **8.2 The vector list** (`include/linux/interrupt.h`)
  - `HI`, `TIMER`, `NET_TX`, `NET_RX`, `BLOCK`, `IRQ_POLL`, `TASKLET`, `SCHED`, `HRTIMER`, `RCU` — who owns each and why the order matters
- **8.3 When softirqs run** (`kernel/softirq.c`)
  - On `irq_exit` (`invoke_softirq`), on `local_bh_enable`, in **`ksoftirqd`**
  - `__do_softirq`: the restart loop, `MAX_SOFTIRQ_RESTART`, time/budget limits, when it punts to `ksoftirqd`
- **8.4 `local_bh_disable` / `local_bh_enable`**
  - The `SOFTIRQ_OFFSET` bookkeeping in `preempt_count`, `in_softirq` vs `in_serving_softirq`
- **8.5 ksoftirqd**
  - Per-CPU thread, scheduling, the "softirq overload" interaction with latency
- **8.6 PREEMPT_RT changes**
  - Softirqs run in task context; `local_bh_disable` becomes a per-CPU lock
- **8.7 Observability:** `/proc/softirqs`, `softirq_entry/exit` tracepoints

---

## Topic 09 — Bottom Halves II: Tasklets, Workqueues, irq_work

The higher-level deferral mechanisms, and how to pick among all of them.

- **9.1 Tasklets** (`kernel/softirq.c`)
  - `struct tasklet_struct`, `tasklet_schedule` / `hi_tasklet`, run on `TASKLET_SOFTIRQ`/`HI_SOFTIRQ`
  - Serialization guarantee (a tasklet never runs concurrently with itself), `tasklet_disable/enable`, `tasklet_kill`
  - Why tasklets are discouraged for new code (latency, the move to threaded IRQs / BH workqueues)
- **9.2 Workqueues (as an interrupt consumer)**
  - Process context, *may sleep*; `schedule_work`, `queue_work`, the `system_wq` family
  - When a handler defers to a workqueue vs a threaded IRQ; the **BH workqueue** (softirq-context workqueue) as a tasklet replacement
  - (Deep workqueue internals are out of scope — pointer to `kernel/workqueue.c`)
- **9.3 `irq_work`** (`kernel/irq_work.c`)
  - Running a callback in hardirq context, *from* NMI/hardirq, via self-IPI; `irq_work_queue`, `irq_work_queue_on`, lazy vs hard
  - Users: perf, printk, RCU, sched
- **9.4 `irq_poll`** (`include/linux/irq_poll.h`)
  - NAPI-style polled completion for high-rate IRQs
- **9.5 Decision matrix:** top-half / softirq / tasklet / BH-workqueue / threaded-IRQ / workqueue / irq_work — latency, context, may-sleep, concurrency

---

## Topic 10 — Contexts, Masking, Atomicity & `preempt_count`

The concurrency rules that make all of the above correct. Placed here because it needs softirqs and threads to already exist.

- **10.1 `preempt_count` layout** (`include/linux/preempt.h`)
  - Sub-fields: preempt, softirq, hardirq, NMI counts; `PREEMPT_BITS`/`SOFTIRQ_BITS`/`HARDIRQ_BITS`/`NMI_BITS`
  - `in_hardirq()`, `in_serving_softirq()`, `in_task()`, `in_nmi()`, `in_interrupt()`, `in_atomic()`
- **10.2 IRQ masking primitives** (`include/linux/irqflags.h`)
  - `local_irq_disable/enable`, `local_irq_save/restore`, `local_irq_disable` vs disabling at the controller
  - Arch flags representation; `irqs_disabled()`
- **10.3 Locks and IRQs**
  - `spin_lock_irqsave` and *why* (deadlock between handler and holder), `spin_lock_bh`, the ordering rules
  - What a top half may take; what a threaded handler may take
- **10.4 Lockdep & IRQ state**
  - hardirq/softirq-safe vs -unsafe lock classes, `local_irq_*` tracking, why lockdep flags inconsistent usage
- **10.5 RCU and interrupts**
  - RCU read-side in IRQ context, `rcu_irq_enter/exit` (now via the generic entry layer), NMI-safe RCU
- **10.6 The "what's allowed where" master table**
  - hardirq / softirq / threaded / process / NMI × (sleep? alloc? which locks? which APIs?)

---

## Topic 11 — MSI and MSI-X (message-signaled interrupts)

The modern delivery method for PCIe and beyond. Built entirely on hierarchical IRQ domains (Topic 04).

- **11.1 Why messages instead of wires**
  - No shared lines, thousands of vectors, write-to-address semantics, DMA-write/IRQ ordering
  - MSI vs MSI-X (table size, per-vector masking/addressing)
- **11.2 The MSI core** (`kernel/irq/msi.c`, `include/linux/msi.h`)
  - `struct msi_desc`, `msi_domain_info`, `msi_domain_ops`, `platform_msi`, per-device MSI lists, `msi_alloc_irqs`
  - `irq_compose_msi_msg` / `irq_write_msi_msg` on the controller chip
- **11.3 PCI/MSI**
  - `pci_alloc_irq_vectors`, `pci_irq_vector`, capability structures, masking
- **11.4 The domain stack in practice**
  - x86: PCI-MSI domain → **vector domain** → APIC; remapping (Intel VT-d / AMD) interposed
  - arm64: PCI-MSI → **GIC ITS** (LPIs), the `its` device table
- **11.5 IMS** (Interrupt Message Store) for high-vector-count modern devices
- **11.6 Observability:** MSI rows in `/proc/interrupts`, `/sys/kernel/debug/irq/`

---

## Topic 12 — Per-CPU Interrupts, IPIs, and NMIs

Three special delivery modes that don't fit the "device line → one CPU" picture.

- **12.1 Per-CPU interrupts**
  - `request_percpu_irq`, `handle_percpu_devid_irq`, `enable_percpu_irq`, `__percpu` `dev_id`
  - Users: arch timer, PMU, GIC PPIs
- **12.2 IPIs (inter-processor interrupts)**
  - The genirq IPI layer (`kernel/irq/ipi.c`, `ipi-mux.c`): `irq_reserve_ipi`, `ipi_send_mask`, `__ipi_send_single`
  - The `smp_call_function` family and reschedule IPI as consumers
  - x86 vectors (RESCHEDULE_VECTOR, CALL_FUNCTION_VECTOR, …); arm64 SGIs
- **12.3 NMIs**
  - Where used: hardlockup/watchdog, perf, `nmi_panic`, KGDB, MCE-adjacent
  - `request_nmi` path in genirq; arch NMI entry; `nmi_enter/exit`, `in_nmi()`
  - The constraints: no locks others might hold, NMI-safe primitives only, re-entrancy
- **12.4 Observability:** IPI/NMI rows in `/proc/interrupts`

---

## Topic 13 — Affinity, Balancing, CPU Hotplug & the Matrix Allocator

Where interrupts land, who decides, and what happens when CPUs come and go.

- **13.1 Affinity basics**
  - `irq_set_affinity`, `irq_data->common->affinity` / `effective_affinity`, `affinity_hint`
  - `/proc/irq/N/smp_affinity[_list]`, `default_smp_affinity`
- **13.2 Managed affinity**
  - `IRQD_AFFINITY_MANAGED`, `irq_create_affinity_masks` (`kernel/irq/affinity.c`), multiqueue devices (blk-mq / NVMe / NICs), spreading across CPUs/nodes
- **13.3 Userspace balancing**
  - `irqbalance` (what it does and its limits), when to pin manually
- **13.4 The x86 vector matrix allocator** (`kernel/irq/matrix.c`)
  - Why vectors are scarce (256 per CPU minus reserved), `irq_matrix_*`, reservation vs assignment, managed reservations
- **13.5 CPU hotplug** (`kernel/irq/cpuhotplug.c`, `migration.c`)
  - `irq_migrate_all_off_this_cpu`, moving IRQs off a dying CPU, `IRQ_MOVE_PCNTXT`, pending-move handling, `irq_force_complete_move`
- **13.6 Observability:** `/proc/interrupts` distribution, `/sys/kernel/debug/irq/`

---

## Topic 14 — Time Accounting, Latency, and PREEMPT_RT

Interrupts and the clock: how time spent in IRQs is measured, where latency comes from, and the RT model.

- **14.1 IRQ time accounting**
  - `CONFIG_IRQ_TIME_ACCOUNTING`, `irqtime_account_irq`, hardirq vs softirq time
  - `/proc/stat` (`intr`, per-CPU `irq`/`softirq` fields), `/proc/interrupts`
- **14.2 Latency sources**
  - IRQ-disabled regions, long top halves, softirq floods, contended `spin_lock_irqsave`
  - The `irqsoff` / `preemptirqsoff` tracers; `hardirqs off` / `softirqs off` latency hooks
- **14.3 Interrupt storms**
  - Screaming/level-stuck IRQs, the spurious-disable safety net (Topic 05), polling fallback (`irq_poll`, NAPI)
- **14.4 PREEMPT_RT in full**
  - Forced-threaded IRQs, softirqs in task context, `local_bh_disable` as a lock, the surviving hardirq paths, priority inheritance
- **14.5 Tuning:** `threadirqs`, affinity isolation (`isolcpus`/`nohz_full` interplay), threaded-IRQ priorities

---

## Topic 15 — Suspend/Resume, Wakeup, and IRQ Power Management

What happens to interrupts across system sleep, and how a device wakes the machine.

- **15.1 Wakeup interrupts**
  - `enable_irq_wake` / `disable_irq_wake`, `IRQD_WAKEUP_STATE`, `irq_set_wake` on the chip
  - Wakeup sources and the `/sys/.../power/wakeup` surface
- **15.2 IRQ suspend/resume core** (`kernel/irq/pm.c`)
  - `suspend_device_irqs` / `resume_device_irqs`, `IRQF_NO_SUSPEND`, `IRQF_EARLY_RESUME`, `IRQF_COND_SUSPEND`
  - syscore ordering, what stays armed during suspend
- **15.3 Resend / lost interrupts across resume** (`kernel/irq/resend.c`)
- **15.4 Controller PM**
  - Saving/restoring distributor/redirection state (GIC, IO-APIC), `irq_chip` PM hooks
- **15.5 Pitfalls:** missed wakeups, shared-IRQ-with-NO_SUSPEND (`IRQF_COND_SUSPEND`)

---

## Topic 16 — Debugging & Observability (practitioner topic)

A complete reference for diagnosing interrupt problems. Each tool: principle → how to read it → which symptom it fits.

- **16.1 The procfs/sysfs surface**
  - `/proc/interrupts` (every column explained), `/proc/softirqs`, `/proc/stat`
  - `/proc/irq/N/*` (`smp_affinity`, `node`, `spurious`, chip name), `/sys/kernel/irq/N/*`
- **16.2 The debugfs surface** (`kernel/irq/debugfs.c`)
  - `/sys/kernel/debug/irq/irqs/N`, `/sys/kernel/debug/irq/domains/*` — state flags, hierarchy, hwirq
- **16.3 Tracepoints**
  - `irq:irq_handler_entry/exit`, `irq:softirq_entry/exit/raise`, `irq:tasklet_*`, `ipi:*`
  - bpftrace one-liners for per-IRQ latency and rate
- **16.4 Latency tracers**
  - `irqsoff`, `preemptirqsoff`, function_graph on the entry path
- **16.5 The classic symptoms → diagnosis**
  - "nobody cared / disabling IRQ#" — shared-IRQ misregistration or screaming line
  - IRQ with a zero count — wrong affinity / never firing / wrong hwirq mapping
  - softirq/`ksoftirqd` at 100% — NET_RX flood, needs NAPI/affinity tuning
  - high IRQ-off latency — long handler or lock; locate with `irqsoff`
  - missing wakeup — `enable_irq_wake`/`IRQF_NO_SUSPEND` gap
  - vector exhaustion (x86) — matrix allocator failure; reduce MSI vectors
- **16.6 `irq_sim` / `irq_test`** for writing tests without hardware

---

## Topic 17 — Initialization Walk-through (boot → interrupts live)

The capstone: re-thread every prior topic onto the boot timeline.

- **17.1 Earliest setup**
  - `early_irq_init` (`kernel/irq/irqdesc.c`): allocate the initial `irq_desc`s, sparse-IRQ tree
  - Arch IDT/vector table install: x86 `idt_setup_*`; arm64 `VBAR_EL1`/`vectors`
- **17.2 `init_IRQ` and the controller bring-up**
  - x86: `native_init_IRQ`, APIC setup, legacy PIC handling
  - arm64/DT/ACPI: `irqchip_init`, `IRQCHIP_DECLARE` probing, GIC init, ITS init
- **17.3 When IRQs are first enabled**
  - `local_irq_enable()` in `start_kernel`; what must be ready first
- **17.4 Secondary CPU bring-up**
  - Per-CPU controller setup (local APIC / GIC redistributor), per-CPU IRQ enable, `ksoftirqd` and IRQ-thread creation
- **17.5 The first device interrupt**
  - Timer IRQ as the canonical first consumer; from controller to `handle_irq` to softirq
- **17.6 Timeline summary:** which layer is live at which moment, and which APIs are legal pre/post each milestone

---

## Appendix A — Source-code Locations

| Subsystem | Primary files |
|---|---|
| genirq core | `kernel/irq/irqdesc.c`, `kernel/irq/manage.c`, `kernel/irq/chip.c`, `kernel/irq/handle.c`, `kernel/irq/internals.h` |
| IRQ domains | `kernel/irq/irqdomain.c`, `include/linux/irqdomain.h` |
| MSI | `kernel/irq/msi.c`, `include/linux/msi.h`, `drivers/pci/msi/` |
| Affinity / matrix / hotplug | `kernel/irq/affinity.c`, `kernel/irq/matrix.c`, `kernel/irq/cpuhotplug.c`, `kernel/irq/migration.c` |
| IPI | `kernel/irq/ipi.c`, `kernel/irq/ipi-mux.c` |
| Spurious / resend / PM / proc / debugfs | `kernel/irq/spurious.c`, `resend.c`, `pm.c`, `proc.c`, `debugfs.c` |
| Generic chip / devres / sim | `kernel/irq/generic-chip.c`, `devres.c`, `irq_sim.c`, `dummychip.c` |
| Softirq / tasklet | `kernel/softirq.c`, `include/linux/interrupt.h` |
| irq_work | `kernel/irq_work.c`, `include/linux/irq_work.h` |
| Generic entry | `include/linux/irq-entry-common.h`, `kernel/entry/` |
| Core headers | `include/linux/irq.h`, `irqdesc.h`, `irqhandler.h`, `irqreturn.h`, `irqflags.h`, `irqnr.h`, `hardirq.h`, `preempt.h` |
| x86 | `arch/x86/kernel/irq.c`, `irqinit.c`, `idt.c`, `apic/`, `include/asm/idtentry.h`, `include/asm/hardirq.h` |
| arm64 | `arch/arm64/kernel/irq.c`, `entry.S`, `entry-common.c` |
| irqchip drivers | `drivers/irqchip/` (`irq-gic*.c`, `irq-apple-aic.c`, …) |

## Appendix B — Per-topic Article Format (the 8 steps)

1. **Position in the subsystem** (where this sits on the Topic-00 spine)
2. **Core data structures** (definition + every meaningful field + relationships)
3. **Key flows** (call chains, critical branches, x86 + arm64)
4. **Key algorithms / design rationale** (why this design, alternatives)
5. **Upstream / downstream interfaces** (who calls in, who it calls)
6. **Observability** (procfs / sysfs / debugfs / tracepoints / BPF)
7. **Common pitfalls & debugging tips**
8. **Further reading & source pointers**

Every topic header also carries a **"Standing requirements check (R1/R2/R3)"** line.

## Appendix C — Suggested Reading Paths by Role

- **Driver author (the common case):** 00 → 03 → 05 → 06 → 07 → 08 → 09 → 13 → 16
- **Embedded / SoC / irqchip author:** 00 → 01 → 02 → 03 → 04 → 05 → 11 → 12 → 17
- **x86 platform / virtualization:** 00 → 01 → 02 → 03 → 04 → 11 → 12 → 13 → 15
- **RT / latency engineer:** 00 → 05 → 07 → 08 → 09 → 10 → 14 → 16
- **Read-it-all (recommended):** 00 → 17 in order; Appendix A as a working reference

## Appendix D — Glossary

IRQ, hwirq, virq, vector, IDT, APIC/IO-APIC/x2APIC, GIC/ITS/LPI/SGI/PPI/SPI, irq_chip, irq_domain, flow handler, top half / bottom half, softirq, tasklet, threaded IRQ, IPI, NMI, MSI/MSI-X, EOI, level/edge trigger, affinity, managed IRQ, IRQ storm, spurious IRQ.

---

**Confirm the plan, then I will write the topics in order (00 first).**
Recommended start: **Topic 00 (overview)** → **Topic 01 (CPU entry)** → **Topic 03 (data model)**.
