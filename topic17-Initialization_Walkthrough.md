# Topic 17 — Initialization Walk-through (boot → interrupts live)

> **Standing requirements check (R1/R2/R3):** R1 — the complete boot path that brings interrupts up, on both arches, re-threading every prior topic onto the timeline (IDT/vectors → controllers → domains → descriptors → first enable → secondary CPUs → first device IRQ). R2 — the capstone, last by design: it only makes sense once you know *what* each piece is, so it re-orders Topics 01–16 along boot. R3 — call sequence verified in `init/main.c`, `kernel/irq/irqdesc.c`, `arch/x86/kernel/{irqinit.c,idt.c}`, `arch/arm64/kernel/irq.c`, `kernel/softirq.c`, `kernel/cpu.c` in `v7.1-rc6`. Re-read R1/R2/R3 in the plan.
>
> **Builds on:** everything — especially Topic 01 (entry), 02 (controllers), 03 (`irq_desc`), 04 (domains), 08 (softirq/ksoftirqd), 13 (hotplug/affinity online).
>
> **Position in the plan:** the final synthesis. After this you can place any IRQ symbol on the boot timeline and know which APIs are legal at each moment.
>
> **Source:** `init/main.c` (`start_kernel`), `kernel/irq/irqdesc.c` (`early_irq_init`), `arch/x86/kernel/irqinit.c`+`idt.c`, `arch/arm64/kernel/irq.c`, `drivers/irqchip/irqchip.c`, `kernel/softirq.c`, `kernel/cpu.c`.

---

## 1. The `start_kernel` Order (the spine of boot)

The relevant slice of `init/main.c`:

```c
asmlinkage void start_kernel(void)
{
    ...
    rcu_init();
    early_irq_init();      /* (1) allocate the initial irq_desc array / sparse tree  (Topic 03) */
    init_IRQ();            /* (2) arch + controller bring-up: IDT/vectors, GIC, domains (Topics 01,02,04) */
    tick_init();
    ...
    softirq_init();        /* (3) softirq machinery                                   (Topic 08) */
    ...
    local_irq_enable();    /* (4) INTERRUPTS ARE NOW ON for the first time            (Topic 10) */
    ...
}
```

Four milestones. Before (1), there is no `irq_desc`. Before (2), there is no controller and no `handle_arch_irq`. Before (4), the CPU runs with IRQs masked (`local_irq_disable` was the boot state). Everything below expands these.

---

## 2. Earliest Setup — before `init_IRQ`

### x86 IDT, in phases (`arch/x86/kernel/idt.c`)
The IDT (Topic 01 §3.1) is filled incrementally as the kernel becomes able to handle more:
- **`idt_setup_early_handler`** — very early: a catch-all so an early fault prints something instead of triple-faulting.
- **`idt_setup_early_traps`** — the exception gates needed before memory is up (#DB, #BP).
- **`idt_setup_early_pf`** — the page-fault gate (needed once paging/memory init begins).
- (later, in `init_IRQ`) **`idt_setup_apic_and_irq_gates`** — the device/APIC/IPI vectors (Topic 12).

So on x86 the *exception* side of the IDT is live well before the *device-IRQ* side.

### arm64 vector table
`VBAR_EL1` is pointed at `vectors` (Topic 01 §4.1) during early CPU setup (`__primary_switched`/`cpu_setup`), so synchronous exceptions (and the entry stubs) work before `init_IRQ`. The IRQ slots route to `handle_arch_irq`, which is still NULL until a controller registers it (§3).

### `early_irq_init` (`kernel/irq/irqdesc.c`)
```c
int __init early_irq_init(void)
{   ... allocate the first batch of irq_desc (sparse-IRQ tree or the static array),
        init each desc's lock/state, then arch_early_irq_init(); }
```
This is milestone (1): after it, `irq_to_desc()` works and the genirq core (Topics 03–06) has somewhere to put descriptors. No interrupts are delivered yet.

---

## 3. `init_IRQ` — Controllers and the Root Handler (milestone 2)

### x86 (`arch/x86/kernel/irqinit.c`)
```c
void __init native_init_IRQ(void)
{   ...
    idt_setup_apic_and_irq_gates();   /* install device/IPI/system vectors in the IDT */
    /* APIC bring-up (apic_intr_mode_init elsewhere): LAPIC, IO-APIC, x2APIC (Topic 02) */
}
```
After this the IDT's device range points at `common_interrupt`/sysvec stubs (Topic 01 §3.2), and the APIC/IO-APIC are programmed. The `vector_irq[]` per-CPU demux (Topic 01) is ready to be populated as drivers request IRQs.

### arm64 (`arch/arm64/kernel/irq.c`)
```c
void __init init_IRQ(void)
{
    init_irq_stacks();    /* per-CPU IRQ stacks (Topic 01 §5) */
    ...
    irqchip_init();       /* probe controllers, build domains, set_handle_irq() */
}
```
`irqchip_init()` (`drivers/irqchip/irqchip.c`) walks the **`IRQCHIP_DECLARE`** table (DT) / ACPI MADT (Topic 02 §5), runs each matching controller's init, which **creates its `irq_domain`** (Topic 04) and, for the root controller, calls **`set_handle_irq(gic_handle_irq)`** (Topic 02 §3.2). The moment `handle_arch_irq` is non-NULL, the arm64 IRQ vector slots have somewhere to go.

So after milestone (2): controllers exist, domains exist, the root handler is published, IRQ stacks are ready — but the CPU still has IRQs masked.

---

## 4. First Enable & the Boot-Time Consumers (milestones 3–4)

- **`softirq_init()`** (milestone 3) sets up the softirq/tasklet machinery (Topic 08); `spawn_ksoftirqd()` (an `early_initcall`) registers the per-CPU **`ksoftirqd`** threads via smpboot and the `CPUHP_SOFTIRQ_DEAD` hotplug state.
- **`local_irq_enable()`** (milestone 4) is the first time the boot CPU accepts interrupts. By now the timer (the canonical first device IRQ) has been requested with `IRQF_TIMER` (= `NO_SUSPEND | NO_THREAD`, Topic 06/15), and `tick_init`/the clockevent driver armed it. The **first device interrupt** is almost always the timer tick: controller delivers → `handle_arch_irq`/`common_interrupt` → `generic_handle_domain_irq` (Topic 04) → the timer's flow handler (`handle_percpu_devid_irq` for the arch timer, Topic 05/12) → the tick handler → `irq_exit_rcu` runs any raised softirqs (Topic 08).
- Later `initcall`s probe devices, which `request_irq` (Topic 06) and populate `vector_irq[]` (x86) / domain revmaps (arm64). MSI domains (Topic 11) come up with the PCI bus.

---

## 5. Secondary CPU Bring-up

When each secondary CPU comes online (SMP boot, and later CPU hotplug — same path, Topic 13 §5):
- Its **per-CPU controller** is initialized: x86 brings up that CPU's Local APIC; arm64 enables its GIC **redistributor** and per-CPU PPIs/SGIs.
- **Per-CPU interrupts** (Topic 12) are enabled *on that CPU* via the CPU-hotplug startup callbacks (the arch timer/PMU call `enable_percpu_irq` from their hotplug state).
- **`irq_affinity_online_cpu`** runs (`CPUHP_AP_IRQ_AFFINITY_ONLINE` in `kernel/cpu.c`, Topic 13): restores **managed** IRQs whose CPUs are now available, and rebalances.
- The CPU's **`ksoftirqd/N`** and any **`irq/N-name`** threads (Topic 07) are created/affined.
- Finally the CPU enables local IRQs and joins interrupt servicing.

So the per-CPU pieces of Topics 01/12/13 are replayed for every CPU, at boot and on every hotplug-online.

---

## 6. Timeline Summary — which APIs are legal when

| Boot point | Live | Legal to call |
|---|---|---|
| before `early_irq_init` | nothing genirq | — |
| after `early_irq_init` (1) | `irq_desc` storage | `irq_to_desc`, descriptor alloc (no delivery) |
| after `init_IRQ` (2) | controllers, domains, root handler, IRQ stacks | `irq_domain_*`, `irq_create_mapping` (Topic 04); chips registered |
| after `softirq_init` (3) | softirq/tasklet | `open_softirq`, `raise_softirq` (Topic 08) |
| after `local_irq_enable` (4) | **delivery** | interrupts actually fire; timer tick runs |
| during `initcall`s | device probing | `request_irq`/`pci_alloc_irq_vectors` (Topics 06/11) |
| per secondary CPU online | that CPU's controller + per-CPU IRQs | `enable_percpu_irq` on that CPU (Topic 12) |

"BUG: Bad ... / NULL `handle_arch_irq`" splats during boot almost always mean something delivered an interrupt *before* its milestone (e.g. an unmasked line before `init_IRQ`, or a driver requesting an IRQ before its domain exists).

---

## 7. Common Pitfalls (boot/init)

- **Requesting an IRQ before its domain/controller is up** — `irq_create_mapping` returns 0 / `request_irq` fails. Order driver init after `irqchip_init` (the `IRQCHIP_DECLARE` ordering and `of_irq` deferral handle this; custom code can get it wrong).
- **Leaving a line unmasked across `local_irq_enable`** — a device that was already asserting before the handler is installed → immediate "nobody cared" (Topic 05). Quiesce devices before enabling.
- **Assuming softirqs run before milestone 3** — `raise_softirq` before `softirq_init`/`ksoftirqd` is meaningless.
- **Per-CPU IRQ enabled on only the boot CPU** — must be enabled on each CPU via the hotplug startup callback (Topic 12/13), or it only fires on CPU0.
- **Controller state not (re)initialized on a hotplugged CPU** — its redistributor/APIC must be brought up in the CPU-up path, else IRQs misroute to it.

---

## 8. Further Reading & Source Pointers

### Source to read (in boot order)
- `init/main.c` `start_kernel` — the milestone sequence.
- `kernel/irq/irqdesc.c` `early_irq_init`.
- `arch/x86/kernel/idt.c` (`idt_setup_*`) + `arch/x86/kernel/irqinit.c` (`native_init_IRQ`); or `arch/arm64/kernel/irq.c` (`init_IRQ`) + `drivers/irqchip/irqchip.c` (`irqchip_init`).
- `kernel/softirq.c` `spawn_ksoftirqd`; `kernel/cpu.c` `CPUHP_AP_IRQ_AFFINITY_ONLINE`.

### Documentation
- `Documentation/core-api/genericirq.rst` (init section); `Documentation/admin-guide/kernel-parameters.txt` (`irqaffinity`, `threadirqs`, `irqpoll`, `irqhandler.duration_warn_us`).

### Mental model
> Boot brings interrupts up in strict order: allocate `irq_desc`s (`early_irq_init`) → install vectors/IDT and bring up controllers + domains + the root handler (`init_IRQ`/`irqchip_init`) → set up softirqs/ksoftirqd → `local_irq_enable`, after which the timer is the first IRQ to fire. Every secondary CPU replays its per-CPU slice (local controller, per-CPU IRQs, managed-affinity restore, ksoftirqd). Knowing this timeline tells you exactly which interrupt API is legal at any boot moment — and why an early IRQ splats.

---

**End of the Linux Interrupt Subsystem series (Topics 00–17), written against this `v7.1-rc6` tree under the three standing requirements (R1 comprehensive · R2 learnable order · R3 source-verified). Use Appendix A of the plan as a working source map, and Appendix C for role-based reading paths.**
