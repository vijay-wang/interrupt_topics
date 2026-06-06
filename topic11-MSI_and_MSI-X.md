# Topic 11 — MSI and MSI-X (message-signaled interrupts)

> **Standing requirements check (R1/R2/R3):** R1 — why messages replaced wires, MSI vs MSI-X, the MSI core structures (`msi_desc`, `msi_domain_info`/`ops`), the PCI driver API, the hierarchical domain stack on **both** arches (x86 vector/remap, arm64 GIC-ITS/LPI), and IMS. R2 — placed after hierarchical domains (Topic 04, which MSI *is* the motivating example for) and after the concurrency rules (Topic 10); it revisits domains with a concrete, end-to-end example. R3 — structs/APIs from `include/linux/msi.h`, `include/linux/pci.h`, `kernel/irq/msi.c`, `drivers/irqchip/irq-gic-v3-its.c` in `v7.1-rc6`. Re-read R1/R2/R3 in the plan.
>
> **Builds on:** Topic 02 (APIC/GIC-ITS controllers), Topic 03 (`irq_chip.irq_compose_msi_msg`/`irq_write_msi_msg`, `irq_common_data.msi_desc`), Topic 04 (hierarchical domains, `alloc`/`activate`), Topic 13 forward-ref (x86 vector scarcity).
>
> **Position in the plan:** the first "advanced delivery" topic. Topic 12 covers the other non-wire deliveries (per-CPU, IPI, NMI).
>
> **Source:** `kernel/irq/msi.c`, `include/linux/msi.h`, `drivers/pci/msi/`, `arch/x86/kernel/apic/{vector.c,msi.c}`, `drivers/irqchip/irq-gic-v3-its.c`.

---

## 1. Why Messages Instead of Wires

A classic interrupt is a *wire* asserted toward a controller (Topic 02). **MSI** (Message-Signaled Interrupt) replaces the wire with a **memory write**: the device DMA-writes a specific *data value* to a specific *address*, and the platform turns that write into an interrupt. Advantages:

- **No shared lines.** Each MSI is its own vector — no `IRQF_SHARED` demux cost (Topic 05 §4), no "nobody cared" ambiguity.
- **Many vectors.** A device can have hundreds/thousands (one per queue) — essential for multiqueue NVMe/NICs.
- **DMA/IRQ ordering.** Because the interrupt *is* a write on the same path as the data DMA, the interrupt cannot arrive before the data it announces (PCIe ordering) — fixing a real race that wired INTx had.
- **Routability.** The target address encodes the destination CPU, so affinity (Topic 13) is just "rewrite the message".

**MSI vs MSI-X:**
| | MSI | MSI-X |
|---|---|---|
| Vectors per device | up to 32, **contiguous** | up to 2048, **independent** |
| Per-vector address/data | shared base, low bits vary | fully independent per entry |
| Per-vector mask | optional/limited | **yes**, per entry |
| Table location | config space | a **MSI-X table** in device MMIO (BAR) |

MSI-X is what modern high-queue devices use; MSI is the simpler legacy form. Both deliver to the *same* genirq flow (Topic 05) — only the delivery/programming differs.

---

## 2. The MSI Core (`kernel/irq/msi.c`, `include/linux/msi.h`)

### `struct msi_desc` — one per MSI vector on a device
```c
struct msi_desc {
    unsigned int    irq;            /* the virq (Topic 03) for this MSI vector */
    unsigned int    nvec_used;
    struct device   *dev;
    struct msi_msg  msg;            /* the COMPOSED message: {address_hi/lo, data} */
    struct irq_affinity_desc *affinity;
    void (*write_msi_msg)(struct msi_desc *entry, void *data);
    u16             msi_index;      /* which entry within the device (0..N-1) */
    union { struct pci_msi_desc pci; struct msi_desc_data data; };  /* bus-specific */
};
```
`msi_desc` lives in the device's MSI list and is reachable from the IRQ via `irq_common_data.msi_desc` (Topic 03). The `msg` is what gets written into the device's MSI capability / MSI-X table to make it fire that vector.

### `struct msi_domain_info` / `struct msi_domain_ops`
An **MSI irq_domain** sits at the top of the hierarchy (Topic 04 §4). `msi_domain_info` describes it (its `irq_chip`, flags like `MSI_FLAG_*`, and `ops`); `msi_domain_ops` provides MSI-specific hooks (`msi_init`, `msi_prepare`, `set_desc`, …). The platform creates it with `msi_create_irq_domain(fwnode, info, parent)` (or PCI/platform wrappers). `msi_parent_ops` (referenced from `irq_domain`, Topic 04) lets a parent domain (the vector/ITS domain) advertise that it can host MSI children.

### Compose vs write (the two-phase programming)
The controller `irq_chip` (Topic 03) provides:
- **`irq_compose_msi_msg(irq_data, msg)`** — compute the `{address, data}` that targets the chosen CPU/vector. Done at **alloc/activate** time (Topic 04 §4: `ops->activate`).
- **`irq_write_msi_msg(irq_data, msg)`** — write that message into the *device* (config space for MSI, the MSI-X table for MSI-X). This is what arms the device.

That compose-then-write split is exactly the domain `alloc`(compose) / `activate`(write) lifecycle from Topic 04 — MSI is its canonical user.

---

## 3. The PCI/MSI Driver API (`include/linux/pci.h`, `drivers/pci/msi/`)

What a PCI driver actually calls:

```c
int n = pci_alloc_irq_vectors(pdev, min_vecs, max_vecs,
                              PCI_IRQ_MSIX | PCI_IRQ_MSI | PCI_IRQ_INTX);
int virq = pci_irq_vector(pdev, queue_index);     /* get the virq for vector i */
request_irq(virq, handler, 0, name, dev);          /* then it's an ordinary IRQ (Topic 06) */
...
pci_free_irq_vectors(pdev);
```

- `pci_alloc_irq_vectors` tries the best available type in the mask (MSI-X, then MSI, then legacy INTx), allocates the vectors (creating `msi_desc`s and virqs via the MSI domain), and returns how many it got. The driver then requests handlers on the returned virqs — from there it's just Topic 06.
- `PCI_IRQ_AFFINITY` (via `pci_alloc_irq_vectors_affinity`) enables **managed affinity** spreading (Topic 13) so each queue's IRQ lands on a different CPU automatically.
- Masking individual MSI-X vectors, reading back, etc., go through `pci_msi_*` helpers but are rarely needed directly.

The driver never touches the APIC/ITS — the MSI domain hierarchy hides it.

---

## 4. The Domain Stack in Practice (Topic 04 made concrete)

### x86
```
   PCI-MSI domain          hwirq = MSI index;   irq_chip composes/writes the device MSI message
        │ parent
   IRQ-remap domain        (present iff Intel VT-d / AMD-Vi remapping is on)
        │ parent           translates message → remap-table index → vector
   x86 vector domain       hwirq = CPU vector;  allocates a scarce per-CPU vector via the
        │ parent           MATRIX allocator (Topic 13); affinity = pick a CPU+vector
   APIC (root)
```
So on x86, allocating one MSI consumes a **CPU vector** (the scarce resource of Topic 02 §2.4). Mass-MSI devices can exhaust vectors; that failure surfaces here. `arch/x86/kernel/apic/vector.c` is the vector domain; `apic/msi.c` the PCI-MSI domain.

### arm64
```
   PCI-MSI domain
        │ parent
   GIC-ITS domain          translates (DeviceID, EventID) → LPI;  the ITS device/ITT tables
        │ parent           (drivers/irqchip/irq-gic-v3-its.c)
   GIC (root, LPIs via redistributor)
```
No per-CPU vector scarcity; the analog limits are the ITS table sizes and the LPI number space. `irq_compose_msi_msg` here writes the ITS doorbell address + EventID; the ITS maps it to an LPI delivered to a redistributor (Topic 02 §3.2).

To debug either, `/sys/kernel/debug/irq/domains/` shows the whole parent chain (Topic 04 §6).

---

## 5. IMS — Interrupt Message Store

Modern accelerators want *far more* interrupts than the PCI MSI-X table cap (2048), and want to manage them in **device-specific memory** rather than a standard table. **IMS** (Interrupt Message Store) lets a device store MSI messages in its own structures, allocated dynamically through the same MSI domain framework (`msi.c` platform-MSI/IMS paths, `MSI_FLAG_*`). For studying genirq the takeaway is: IMS is *still* MSI through the hierarchical domain stack — just with a device-private message store instead of the PCI MSI-X table. Same `msi_desc`, same compose/write, same flow handler.

---

## 6. Observability

- **`/proc/interrupts`** — MSI IRQs show as `PCI-MSI` / `PCI-MSIX` / `ITS-MSI` in the right column, often many rows per device (one per queue), each named like `nvme0q3`.
- **`/sys/kernel/debug/irq/domains/`** — the MSI→vector/ITS hierarchy; the place to confirm a mapping or diagnose vector exhaustion.
- **`/sys/bus/pci/devices/<dev>/msi_irqs/`** — the MSI vectors a PCI device owns.
- x86 vector pressure: watch for `irq N: vector allocation failed` in dmesg; `/sys/kernel/debug/irq/domains/VECTOR` shows usage.
- Tracepoints: the standard `irq:irq_handler_*` fire on the delivered virq like any IRQ.

---

## 7. Common Pitfalls

- **x86 vector exhaustion** — requesting thousands of MSI-X vectors across many devices/CPUs can fail at the vector domain. Use **managed affinity** (`PCI_IRQ_AFFINITY`) and request only as many queues as CPUs you'll use. (arm64 doesn't hit this.)
- **Reading the device's MSI message before `activate`** — the message is *composed* at alloc but only *written* to the device at activate/startup (Topic 04 §4). Premature reads show stale/zero.
- **Assuming MSI is shared** — it isn't; each vector is exclusive. Don't write `IRQF_SHARED` handlers for MSI.
- **Forgetting `pci_free_irq_vectors`** — leaks `msi_desc`s/virqs; use the device-managed `pcim_*` variants where possible.
- **Affinity confusion** — for MSI, changing affinity *rewrites the message* (`irq_set_affinity` → compose → write). On x86 that also reallocates a vector on the target CPU; under heavy churn this can fail (Topic 13).
- **MSI-X table in a BAR you didn't map** — the table lives in device MMIO; the PCI core maps it, but custom drivers poking it manually can corrupt entries.

---

## 8. Further Reading & Source Pointers

### Documentation
- `Documentation/PCI/msi-howto.rst` — the driver-facing guide (`pci_alloc_irq_vectors`, affinity).
- `Documentation/core-api/irq/irq-domain.rst` — the hierarchy this rides on.

### Source to read
- `kernel/irq/msi.c` — `msi_domain_alloc_irqs_*`, `msi_create_irq_domain`, the compose/activate path.
- `arch/x86/kernel/apic/vector.c` + `apic/msi.c` — the x86 vector + PCI-MSI domains.
- `drivers/irqchip/irq-gic-v3-its.c` — the arm64 ITS MSI domain (`its_irq_domain_ops`, `its_irq_compose_msi_msg`).

### Mental model
> MSI turns "assert a wire" into "DMA-write {addr,data}", so each interrupt is its own routable vector. The kernel models it as a hierarchical domain stack: a PCI-MSI domain on top *composes* the message and *writes* it to the device, with parents (x86 vector+remap, or arm64 GIC-ITS) supplying the real delivery resource. A driver calls `pci_alloc_irq_vectors` + `pci_irq_vector` + `request_irq`, and from the flow handler on, an MSI is just an ordinary IRQ.

---

**Next: Topic 12 — Per-CPU Interrupts, IPIs, and NMIs** (the other three non-wire delivery modes: lines private to one CPU, CPU-to-CPU interrupts, and the unmaskable NMI).
