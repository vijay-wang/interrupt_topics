# Topic 04 — IRQ Domains: hwirq → virq Translation

> **Standing requirements check (R1/R2/R3):** R1 — covers the mapping problem, the `irq_domain`/`irq_domain_ops` structures field-by-field, all mapping kinds, the full hierarchical-domain model (the foundation MSI in Topic 11 needs), and the DT/ACPI firmware path. R2 — sits between the data model (Topic 03, which introduced `virq` and `hwirq`) and the flow handlers (Topic 05, which need a `virq`→`desc`): this is the missing link that turns a controller's hwirq into the descriptor you just learned. R3 — structs copied from `include/linux/irqdomain.h` in `v7.1-rc6`. Re-read R1/R2/R3 in the plan.
>
> **Builds on:** Topic 00 (hwirq vs virq), Topic 02 (controllers produce hwirqs), Topic 03 (`irq_desc`/`irq_data`, and why hierarchy stacks multiple `irq_data`).
>
> **Position in the plan:** the translation layer. After this, "controller delivers hwirq H → `generic_handle_domain_irq(domain, H)` → the right `irq_desc`" is fully explained, and you are ready for the flow handlers.
>
> **Source:** `kernel/irq/irqdomain.c`, `include/linux/irqdomain.h`, `include/linux/irqdomain_defs.h`; consumers in `drivers/irqchip/`.

---

## 1. The Problem

Topic 00 introduced two numbers; the domain layer is the machinery that maps between them. Why it must exist:

1. **hwirq numbers collide.** Every GIC has an SPI 32; every IO-APIC has a pin 0. The hwirq is only meaningful relative to *one* controller. The kernel needs a *global* token — the virq — that indexes `irq_desc`.
2. **Discovery is dynamic.** Controllers are found at boot (DT/ACPI) or hotplugged (PCI MSI). You cannot statically pre-assign virqs; they must be allocated on demand.
3. **Controllers stack.** A PCI-MSI interrupt passes through the PCI-MSI controller, then the CPU's vector/APIC (x86) or the GIC ITS (arm64). Each level has its *own* hwirq for the same logical interrupt. The mapping must be a *chain*, not a single pair.

An **`irq_domain`** is "one controller's hwirq namespace plus its reverse map to virqs". Each controller (Topic 02) creates one (or more) at init.

---

## 2. `struct irq_domain` & `irq_domain_ops`

```c
struct irq_domain {
    struct list_head        link;          /* on the global domain list */
    const char              *name;
    const struct irq_domain_ops *ops;       /* the per-controller behaviour (below) */
    void                    *host_data;     /* controller-private (often the chip's struct) */
    unsigned int            flags;          /* IRQ_DOMAIN_FLAG_* (hierarchy, MSI, …) */
    unsigned int            mapcount;       /* live mappings */
    struct mutex            mutex;
    struct irq_domain       *root;          /* top of the hierarchy this domain is in */
    struct fwnode_handle    *fwnode;        /* the firmware node (DT/ACPI) that named it */
    enum irq_domain_bus_token bus_token;    /* disambiguates multiple domains on one fwnode (e.g. wired vs MSI) */
    struct irq_domain_chip_generic *gc;     /* optional generic-chip backing */
#ifdef CONFIG_IRQ_DOMAIN_HIERARCHY
    struct irq_domain       *parent;        /* the NEXT controller up (NULL at the root) */
#endif
#ifdef CONFIG_GENERIC_MSI_IRQ
    const struct msi_parent_ops *msi_parent_ops;   /* Topic 11 */
#endif
    /* reverse map: small hwirqs use the inline linear array; large/sparse use the radix tree */
    irq_hw_number_t         hwirq_max;
    unsigned int            revmap_size;
    struct radix_tree_root  revmap_tree;
    struct irq_data __rcu   *revmap[] __counted_by(revmap_size);   /* hwirq → irq_data (linear part) */
};
```

The **reverse map** (`hwirq → irq_data`) is the hot lookup done on every interrupt: the inline `revmap[]` array handles small dense hwirq ranges in O(1); the `revmap_tree` radix tree handles large/sparse ranges. `irq_resolve_mapping()` / `irq_find_mapping()` read it; it is RCU-protected so the delivery path is lockless.

```c
struct irq_domain_ops {
    int  (*match)(struct irq_domain *d, struct device_node *node, enum irq_domain_bus_token);
    int  (*select)(struct irq_domain *d, struct irq_fwspec *fwspec, enum irq_domain_bus_token);
    int  (*map)(struct irq_domain *d, unsigned int virq, irq_hw_number_t hw);   /* v1: set chip+handler for a new mapping */
    void (*unmap)(struct irq_domain *d, unsigned int virq);
    int  (*xlate)(struct irq_domain *d, struct device_node *node,
                  const u32 *intspec, unsigned int intsize,
                  unsigned long *out_hwirq, unsigned int *out_type);            /* v1: DT specifier → hwirq+trigger */
#ifdef CONFIG_IRQ_DOMAIN_HIERARCHY
    int  (*alloc)(struct irq_domain *d, unsigned int virq, unsigned int nr_irqs, void *arg); /* v2 */
    void (*free)(struct irq_domain *d, unsigned int virq, unsigned int nr_irqs);
    int  (*activate)(struct irq_domain *d, struct irq_data *irqd, bool reserve);
    void (*deactivate)(struct irq_domain *d, struct irq_data *irqd);
    int  (*translate)(struct irq_domain *d, struct irq_fwspec *fwspec,
                      unsigned long *out_hwirq, unsigned int *out_type);        /* v2 replacement for xlate */
#endif
};
```

Two API generations live here:
- **v1 (`map`/`xlate`)** — flat domains. `xlate` parses a DT interrupt specifier into `(hwirq, trigger)`; `map` sets the new virq's `chip` and flow handler.
- **v2 (`alloc`/`free`/`activate`/`translate`)** — hierarchical domains (next section). `translate` is the `xlate` successor; `alloc` builds the whole chain of `irq_data` down the parents; `activate` programs the hardware (e.g. writes the MSI message) when the IRQ is actually started.

---

## 3. Mapping Kinds & Creating Flat Mappings

A non-hierarchical controller creates one domain via one of:

- **Linear** (`irq_domain_create_linear`, formerly `irq_domain_add_linear`): a fixed-size `revmap[]` indexed by hwirq. Best when hwirqs are `0..N` dense. The common case (GIC SPIs, an SoC GPIO block).
- **Tree** (`irq_domain_create_tree`): radix-tree revmap for large/sparse hwirq spaces (e.g. MSI before hierarchy, or a controller with 32-bit hwirqs).
- **No-map** (`irq_domain_create_nomap`): the hwirq *is* the virq (rare; some legacy controllers where the hardware number is already global).

Using a flat domain:

```c
unsigned int virq = irq_create_mapping(domain, hwirq);  /* alloc a virq, call ops->map, return it */
unsigned int virq = irq_find_mapping(domain, hwirq);    /* lookup only, used on the delivery path */
irq_dispose_mapping(virq);                              /* tear down */
```

On the hot path, the controller's root handler (Topic 02) does **`generic_handle_domain_irq(domain, hwirq)`**, which internally `irq_find_mapping`s the virq, fetches the `irq_desc`, and calls `desc->handle_irq(desc)` (the flow handler, Topic 05). That single call is the join point of Topics 02→04→05.

---

## 4. Hierarchical IRQ Domains (the MSI foundation)

When interrupts pass through stacked controllers, the domains form a **parent chain** (`domain->parent`), and one `irq_desc` gets a *stack* of `irq_data` (Topic 03 §3), one per level, linked by `irq_data->parent_data`.

Example, x86 PCI-MSI:
```
   PCI-MSI domain   (hwirq = MSI index)        ← top: composes the MSI message
        │ parent
   (IRQ remap domain, if VT-d/AMD-Vi)
        │ parent
   x86 vector domain (hwirq = vector)          ← allocates a CPU vector (Topic 13 matrix)
        │ parent
   (APIC)                                       ← root
```
arm64 PCI-MSI: `PCI-MSI domain → GIC ITS domain (LPI) → GIC` .

Allocation walks **top-down**, then each level fills its `irq_data` **bottom-up**:

```c
irq_domain_alloc_irqs(domain, nr_irqs, node, arg);
  → domain->ops->alloc(domain, virq, nr, arg)
       → irq_domain_alloc_irqs_parent(...)        /* recurse to parent first */
       → irq_domain_set_hwirq_and_chip(domain, virq, hwirq, chip, chip_data)  /* set THIS level */
```

Helpers let a child forward an operation to its parent chip without knowing the hardware: **`irq_chip_mask_parent`, `irq_chip_unmask_parent`, `irq_chip_eoi_parent`, `irq_chip_set_affinity_parent`, `irq_chip_retrigger_hierarchy`**. So a GIC-ITS chip's `irq_eoi` can just call `irq_chip_eoi_parent` to let the GIC do the EOI. `ops->activate` is the moment a level commits hardware state — e.g. the MSI domain writes the MSI address/data to the device in `activate` (which is why MSI IRQs are *composed* at alloc but *written* at activate/startup).

`irq_domain_set_hwirq_and_chip`, `irq_domain_create_hierarchy`, and `IRQ_DOMAIN_FLAG_HIERARCHY` are the core primitives; Topic 11 walks a full MSI example end to end.

---

## 5. The Firmware Interface (DT / ACPI)

How a *device* says "I use interrupt X on controller Y":

- **`struct fwnode_handle`** abstracts a DT node or ACPI device; a domain is keyed by its `fwnode` (plus `bus_token` to separate, say, a controller's wired domain from its MSI domain on the same node).
- **`struct irq_fwspec`** = `{ fwnode, param_count, param[] }` — the generic form of an interrupt specifier (the DT `interrupts = <...>` cells, or an ACPI GSI). `irq_create_fwspec_mapping(fwspec)` resolves it: finds the domain (`ops->select`/`match`), translates (`ops->translate`/`xlate`) to `(hwirq, trigger)`, and allocates the mapping (flat or hierarchical).
- **Device-tree driver shortcut:** `irq_of_parse_and_map(np, index)` reads `np`'s `interrupts` property and returns a ready virq — what most platform drivers call.
- **`#interrupt-cells`** in the controller's DT node tells `xlate`/`translate` how many u32s describe one interrupt (e.g. GIC uses 3: type, number, flags).
- **ACPI:** GSIs are mapped via `acpi_register_gsi()` → the IO-APIC/GIC domain.

So the chain is: **DT/ACPI specifier → `irq_fwspec` → domain `select`+`translate` → `(hwirq,trigger)` → allocate virq + `irq_desc`**, and the driver gets back a virq to hand to `request_irq()` (Topic 06).

---

## 6. Observability

- **`/sys/kernel/debug/irq/domains/`** (`CONFIG_GENERIC_IRQ_DEBUGFS`): one file per domain plus a `default`; shows name, size, mapped count, flags, and — crucially — the **hierarchy** (parent chain) and each level's hwirq/chip. The first place to look when an MSI/hierarchy mapping is wrong.
- **`/sys/kernel/debug/irq/irqs/N`**: for one virq, prints every level's `(domain, hwirq, chip)` up the stack.
- `cat /proc/interrupts` right column shows the *leaf* chip name; debugfs shows the whole stack.

---

## 7. Common Pitfalls

- **Using `irq_find_mapping` where you meant `irq_create_mapping`** (or vice-versa): `find` is lookup-only (delivery path); `create` allocates. A driver that `find`s before anyone `create`d gets virq 0.
- **`#interrupt-cells` mismatch** between the controller and the consumer's `interrupts` property → `translate` returns garbage hwirq → IRQ never fires or hits the wrong line. The #1 DT interrupt bug.
- **Wrong `bus_token`** when a controller has multiple domains on one fwnode (wired + MSI): `select` picks the wrong domain.
- **Forgetting `activate` semantics for MSI**: the message isn't written to the device until activation; reading the device's MSI registers before startup shows stale data.
- **Hierarchy depth confusion**: calling `irq_chip_mask_parent` from the *root* chip (no parent) — only intermediate chips forward.

---

## 8. Further Reading & Source Pointers

### Documentation
- `Documentation/core-api/irq/irq-domain.rst` — the authoritative domain write-up (linear/tree/nomap, hierarchy, the alloc/activate split).

### Source to read
- `kernel/irq/irqdomain.c` — `irq_create_mapping`, `irq_domain_alloc_irqs`, `irq_find_mapping`, `irq_create_fwspec_mapping`.
- `drivers/irqchip/irq-gic-v3.c` `gic_irq_domain_ops` (`translate`/`alloc`/`activate`) — a real hierarchical-capable domain.
- `drivers/irqchip/irq-gic-v3-its.c` — the MSI/LPI child domain (pairs with Topic 11).

### Mental model
> A domain is a controller's private hwirq→virq phonebook. Flat controllers keep one pair per line; stacked controllers chain domains (and `irq_data`) so one logical interrupt is described once per level. Delivery calls `generic_handle_domain_irq(domain, hwirq)` to turn the controller's local number into the `irq_desc` and its flow handler — which is Topic 05.

---

**Next: Topic 05 — Flow Handlers** (now that `generic_handle_domain_irq` found the `irq_desc`, the per-trigger-type state machine that drives the chip and runs your handler).
