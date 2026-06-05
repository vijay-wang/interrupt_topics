# Topic 02 — Interrupt Controllers (the routing hardware)

> **Standing requirements check (R1/R2/R3):** R1 — covers the full controller landscape on both arches (x86 8259/LAPIC/IOAPIC/x2APIC; arm GICv2/v3/v4, and note: **GICv5 drivers exist in this tree**), trigger types, cascading, and the driver glue. R2 — placed right after CPU entry (Topic 01) and before the genirq data model (Topic 03): you must know what real hardware does before seeing how Linux abstracts it. R3 — verified against `drivers/irqchip/` and `arch/x86/kernel/apic/` in `v7.1-rc6`. Re-read R1/R2/R3 in the plan.
>
> **Builds on:** Topic 00 (the spine), Topic 01 (how the vector/IRQ reaches kernel C; the x86-demuxes-in-hardware vs arm64-reads-the-controller contrast).
>
> **Position in the plan:** the hardware layer between device and CPU. Topic 03 will show how Linux models *every* controller below as one `irq_chip`; this topic is so you can read the real drivers without drowning.
>
> **Source files:** `arch/x86/kernel/apic/{apic.c,io_apic.c,apic_flat_64.c,x2apic_*.c}`, `arch/x86/kernel/i8259.c`; `drivers/irqchip/irq-gic.c` (v2), `irq-gic-v3.c`, `irq-gic-v3-its.c`, `irq-gic-v4.c`, `irq-gic-v5*.c`; `drivers/irqchip/irqchip.c`.

---

## 1. Position in the Subsystem & What a Controller Does

A device cannot talk to a CPU core directly; an **interrupt controller** sits between them and provides:

- **Aggregation** — many device lines funnel into a few CPU inputs.
- **Masking/enabling** — per-source on/off, so the kernel can silence one device.
- **Prioritization** — decide which pending source wins.
- **Routing/affinity** — choose *which* CPU receives the interrupt.
- **Acknowledgement / EOI** — a handshake telling the controller "I've taken this; you may deliver the next".
- **Trigger configuration** — level vs edge, polarity.

The kernel models all of this as an `irq_chip` (Topic 03) hung off an `irq_domain` (Topic 04). This topic is the *hardware* those abstractions describe.

### Trigger types (applies to every controller)
- **Level-triggered**: the line is asserted *as long as* the device needs service. The handler must clear the device's condition or the IRQ re-asserts. Safe against lost interrupts; requires the flow handler to mask-then-unmask (Topic 05 `handle_level_irq`).
- **Edge-triggered**: a *transition* signals the event. Cheaper, but an edge that arrives while masked can be *lost* unless re-checked (`handle_edge_irq` keeps `IRQS_PENDING` and re-runs; `resend.c` re-injects).

These two words drive the entire flow-handler design in Topic 05 — keep them in mind.

---

## 2. x86 Controllers

### 2.1 8259A PIC (legacy)
Two cascaded 8259A chips → 15 usable lines (IRQ0–15), the original PC interrupt controller (`arch/x86/kernel/i8259.c`). Modern systems still *emulate* it for early boot and legacy devices, but it is almost always superseded by the APIC. Programmed via I/O ports 0x20/0xA0; EOI is an explicit port write. Worth knowing only because legacy IRQ numbers (IRQ0 timer, IRQ1 keyboard, …) come from here.

### 2.2 Local APIC (LAPIC) — per CPU
Each CPU has a Local APIC (`arch/x86/kernel/apic/apic.c`). It is the CPU's personal interrupt gateway:
- **LVT** (Local Vector Table): the APIC timer, thermal, performance-counter, and error interrupts are configured here as local sources.
- **IPIs**: a CPU writes the **ICR** to send an inter-processor interrupt to another CPU (Topic 12).
- **EOI register**: writing it acknowledges the in-service interrupt (this is the `apic_eoi()` you saw in `common_interrupt`, Topic 01).
- **Priority** (TPR/PPR), spurious-interrupt vector.

**x2APIC** is the MSR-based, larger-APIC-ID successor to the MMIO **xAPIC** (`x2apic_phys.c` / `x2apic_cluster.c`), required for large CPU counts and used with interrupt remapping.

### 2.3 I/O APIC
The **I/O APIC** (`arch/x86/kernel/apic/io_apic.c`) routes *external* device lines. Its **redirection table** entry per pin selects: destination CPU(s), vector, delivery mode, trigger/polarity, mask bit. So an external level line on IOAPIC pin 16 becomes "vector V delivered to CPU N's LAPIC". This is the chip behind legacy/INTx device IRQs; MSI bypasses it (Topic 11).

### 2.4 The vector as a scarce resource
On x86 the LAPIC delivers a *vector* (Topic 01), and there are only ~256 per CPU, minus exceptions and system vectors. Every device IRQ on a CPU must own a distinct vector there. That scarcity is why x86 has the **matrix allocator** (Topic 13) and why mass-MSI devices can exhaust vectors. arm64 has no equivalent per-CPU vector limit — a structural difference.

---

## 3. arm/arm64 Controllers — the GIC family

The **GIC** (Generic Interrupt Controller) is ARM's standard. Interrupt classes (numbering is the GIC's hwirq space):

- **SGI** (Software-Generated Interrupt, 0–15): IPIs (Topic 12).
- **PPI** (Private Peripheral Interrupt, 16–31): per-CPU lines (arch timer, PMU).
- **SPI** (Shared Peripheral Interrupt, 32+): ordinary device interrupts, routable to any CPU.
- **LPI** (Locality-specific Peripheral Interrupt): message-based, used for **MSI** via the ITS (GICv3+).

### 3.1 GICv2
Two blocks: the **Distributor** (global: enable, priority, routing of SPIs) and per-CPU **CPU Interface** (ack/EOI, priority mask). `drivers/irqchip/irq-gic.c`. Acking reads `GICC_IAR`; EOI writes `GICC_EOIR`.

### 3.2 GICv3 / GICv4
`drivers/irqchip/irq-gic-v3.c`. Major changes:
- **System-register** CPU interface (`ICC_*_EL1`) instead of MMIO — faster ack/EOI. The root handler reads `ICC_IAR1_EL1`:
  ```c
  static void __gic_handle_irq(u32 irqnr, struct pt_regs *regs)
  {   ...
      if (generic_handle_domain_irq(gic_data.domain, irqnr)) { ... }   /* hwirq → virq dispatch */
  }
  /* registered at init: */  set_handle_irq(gic_handle_irq);
  ```
  This is the arm64 software demux promised in Topic 01.
- **Redistributors** (per-CPU) hold SGI/PPI and LPI configuration.
- **ITS** (Interrupt Translation Service, `irq-gic-v3-its.c`): translates an MSI `(DeviceID, EventID)` write into an LPI delivered to a CPU — the heart of arm64 MSI (Topic 11). It owns device/collection tables in memory.
- **GICv4** (`irq-gic-v4.c`) adds *direct injection* of virtual LPIs into a running VM guest without a hypervisor trap — virtualization-focused.
- **Pseudo-NMI**: GIC priorities let a high-priority IRQ behave like an NMI (`CONFIG_ARM64_PSEUDO_NMI`, Topic 12), since classic arm64 lacked a separate maskable NMI.

### 3.3 GICv5 (present in this tree)
`drivers/irqchip/irq-gic-v5*.c` (`irq-gic-v5.c`, `-irs.c`, `-its.c`, `-iwb.c`) — the next-generation GIC architecture is already supported here (R3: don't assume "v3/v4 is newest" — `grep drivers/irqchip` and see). The IRS (Interrupt Routing Service) and a redesigned ITS replace the v3 distributor/redistributor split. You don't need its register-level detail to study genirq, but know it exists and follows the same `irq_chip`/`irq_domain` contract.

### 3.4 SoC secondary controllers
Beyond the GIC, SoCs have pin-controller/GPIO interrupt blocks, "L2" combiners, etc. (e.g. `irq-bcm*`, `exynos-combiner.c`, `irq-apple-aic.c` for Apple Silicon). They appear as *child* controllers chained to the GIC — next section.

---

## 4. Cascaded / Chained Controllers

A controller's output can be a single line *into another controller*. Two ways Linux handles the child:

- **Chained handler** (`irq_set_chained_handler_and_data`): the parent IRQ's flow handler is replaced by the child controller's demux routine, which reads the child's status register and calls `generic_handle_domain_irq()` for the specific child hwirq. No separate `irqaction`; low overhead. Typical for GPIO/AIC-style controllers.
- **Plain `request_irq` demux**: the child registers a normal handler on the parent IRQ and demuxes in a top half. Simpler but heavier (full irqaction machinery per event).

Either way, the child gets its own `irq_domain` (Topic 04) so its hwirqs don't collide with the parent's.

---

## 5. The Driver Glue (how a controller registers)

Controller drivers live in `drivers/irqchip/` and self-register:

- **`IRQCHIP_DECLARE(name, compat, init_fn)`** ties a device-tree `compatible` string to an init function; `irqchip_init()` (called from `init_IRQ`, Topic 17) walks them. ACPI systems use `IRQCHIP_ACPI_DECLARE` / the MADT.
- The init function creates the `irq_domain`, fills an `irq_chip` (mask/unmask/eoi/set_type/set_affinity), and for the root controller calls **`set_handle_irq()`** to install `handle_arch_irq` (arm64) — the pointer Topic 01's `__el1_irq` invokes.
- `drivers/irqchip/irqchip.c` holds the registration plumbing.

So the chain is: **DT/ACPI describes a controller → `irqchip_init` runs its init → it builds a domain + chip → root controller publishes `handle_arch_irq`**. After that, every IRQ flows controller → `handle_arch_irq` → `generic_handle_domain_irq` → `irq_desc->handle_irq`.

---

## 6. Observability

- `/proc/interrupts` — the right-hand text names the controller and trigger ("IO-APIC 16-fasteoi", "GICv3 30 Level", "PCI-MSI"). Reading it tells you which chip and flow handler back each IRQ.
- `/sys/kernel/debug/irq/domains/*` — one node per `irq_domain`, showing the controller hierarchy (Topic 04/16).
- `/sys/kernel/irq/N/` — per-IRQ chip name, hwirq, type.
- arm64: `cat /proc/interrupts` shows IPIs (SGIs) in a separate block at the bottom; the GIC/ITS appear as the chips.

---

## 7. Common Pitfalls

- **Wrong trigger type in DT/ACPI** — declaring an edge line as level (or wrong polarity) yields either a screaming IRQ ("nobody cared", Topic 16) or lost events. The `irq_set_type` on the chip must match the board.
- **Forgetting EOI semantics differ** — GICv3 uses sysreg EOI, GICv2 MMIO, LAPIC an EOI register, 8259 a port write. The flow handler (via `irq_chip->irq_eoi`) hides this, but custom controller drivers get it wrong constantly.
- **Vector exhaustion is an x86-only failure mode** — don't look for it on arm64; there the analog is ITS table/LPI limits.
- **Assuming the GIC is the only controller** — GPIO/pinctrl interrupts are *chained* children; an IRQ "not firing" may be a missing child-domain mapping, not a GIC issue.
- **x2APIC vs interrupt remapping** — large x86 systems need x2APIC + IRQ remapping (VT-d/AMD); a misconfig shows as IRQs not reaching high-numbered CPUs.

---

## 8. Further Reading & Source Pointers

### Documentation
- `Documentation/arch/arm64/...`, the ARM GIC architecture specs (v2/v3/v4/v5) for register detail.
- Intel SDM Vol 3 (APIC); `Documentation/arch/x86/...`.

### Source to read
- `drivers/irqchip/irq-gic-v3.c` — `gic_handle_irq`, `gic_irq_domain_ops`, the chip callbacks: the cleanest end-to-end controller driver to study.
- `arch/x86/kernel/apic/io_apic.c` — redirection-table programming.
- `drivers/irqchip/irqchip.c` + an `IRQCHIP_DECLARE` user — the registration path.

### Mental model
> A controller turns "device needs service" into "deliver source S to CPU C, with this trigger and EOI protocol". Linux wraps each controller as an `irq_chip` on an `irq_domain`; the root controller publishes one entry point (`handle_arch_irq` on arm64; the per-vector `vector_irq[]` on x86). Everything controller-specific — APIC vs GIC vs ITS — is hidden behind those two abstractions, which Topics 03 and 04 build next.

---

**Next: Topic 03 — The genirq Core Data Model** (`irq_desc` / `irq_data` / `irq_chip` / `irqaction`: the four structures that turn all of the above into one uniform interface).
