# Topic 13 — Affinity, Balancing, CPU Hotplug & the Matrix Allocator

> **Standing requirements check (R1/R2/R3):** R1 — affinity (requested vs effective), managed affinity for multiqueue devices, userspace `irqbalance`, the x86 vector matrix allocator, and CPU-hotplug IRQ migration — end to end. R2 — placed in the "system management" band after the delivery topics (11/12): you must know MSI/per-CPU before affinity matters, and the matrix allocator is the vector-scarcity resolution promised in Topics 02/11. R3 — verified against `kernel/irq/{manage.c,affinity.c,matrix.c,cpuhotplug.c,migration.c}` in `v7.1-rc6`. Re-read R1/R2/R3 in the plan.
>
> **Builds on:** Topic 03 (`affinity`/`effective_affinity`, `IRQD_AFFINITY_MANAGED`, `pending_mask`), Topic 11 (MSI affinity = rewrite the message; x86 vector domain), Topic 12 (per-CPU IRQs are exempt).
>
> **Position in the plan:** the placement-and-lifecycle topic. Topic 14 then covers the time/latency consequences of placement.
>
> **Source:** `kernel/irq/manage.c` (`irq_set_affinity`, `irq_setup_affinity`), `kernel/irq/affinity.c` (`irq_create_affinity_masks`), `kernel/irq/matrix.c`, `kernel/irq/cpuhotplug.c` + `migration.c`.

---

## 1. Affinity Basics: Requested vs Effective

**Affinity** = the set of CPUs allowed to receive an IRQ. Two masks live in `irq_common_data` (Topic 03 §3), and confusing them is the #1 affinity bug:

- **`affinity`** — what was *requested* (by the driver, by `/proc/irq/N/smp_affinity`, or the default). A *set* of allowed CPUs.
- **`effective_affinity`** — where the interrupt *actually* lands. Many controllers deliver to a *single* CPU even when the requested mask has several (e.g. x86 picks one CPU+vector); `effective_affinity` is that single CPU. `CONFIG_GENERIC_IRQ_EFFECTIVE_AFF_MASK`.

Setting affinity:
```c
int irq_set_affinity(unsigned int irq, const struct cpumask *mask);   /* → chip->irq_set_affinity */
int irq_force_affinity(unsigned int irq, const struct cpumask *mask); /* override managed/locked */
```
`__irq_set_affinity` calls the chip's `irq_set_affinity` (Topic 03). For MSI this *rewrites the message* (Topic 11 §4); for an IO-APIC/GIC it reprograms the routing. If the IRQ is in flight or the chip needs the change deferred, it's stashed in `desc->pending_mask` (`CONFIG_GENERIC_PENDING_IRQ`) and applied at the next interrupt (`irq_move_masked_irq`) — that's the "interrupt moves on its next occurrence" behavior.

`irq_setup_affinity` picks the initial mask at request time (honoring `irq_default_affinity` and node locality).

---

## 2. Managed Affinity (multiqueue devices)

Manually pinning hundreds of NVMe/NIC queue IRQs is hopeless. **Managed affinity** lets the kernel spread them automatically and keep them sane across hotplug:

- A driver requests it via `pci_alloc_irq_vectors_affinity(..., PCI_IRQ_AFFINITY, ...)` (Topic 11 §3) or by passing a `struct irq_affinity` spec.
- **`irq_create_affinity_masks`** (`kernel/irq/affinity.c`) computes one CPU mask per vector, **spreading** vectors evenly across CPUs/NUMA nodes, optionally reserving leading vectors for non-queue (admin) interrupts.
- Such IRQs are marked **`IRQD_AFFINITY_MANAGED`**: userspace **cannot** change their affinity via `/proc/irq` (writes are rejected), because the kernel owns the queue↔CPU mapping. `irqbalance` skips them.
- On CPU hotplug, managed IRQs whose CPUs all go offline are *shut down* and *restored* automatically (`irq_affinity_online_cpu`, §4), rather than crammed onto a survivor.

This is why, on a modern server, each NVMe queue's IRQ sits on its "own" CPU and you can't (and shouldn't) move it.

---

## 3. Userspace Balancing: `irqbalance`

For *unmanaged* IRQs, the userspace **`irqbalance`** daemon periodically reads `/proc/interrupts` and rewrites `/proc/irq/N/smp_affinity` to spread interrupt load across CPUs (with policy hints from `/proc/irq/N/affinity_hint`, which drivers set via `irq_set_affinity_hint`).

- Pros: keeps a busy device from hammering CPU0; NUMA-aware.
- Cons/limits: it's reactive and coarse; for latency-critical or isolated-CPU setups you usually **disable irqbalance and pin manually** (or use managed affinity). `IRQF_NOBALANCING` / `IRQD_AFFINITY_MANAGED` exclude IRQs from it.
- `/proc/irq/default_smp_affinity` sets the default mask for newly created IRQs.

Manual pinning: `echo <cpumask> > /proc/irq/N/smp_affinity` (or `..._list`). Combine with `isolcpus`/`nohz_full` to keep interrupts off latency-critical cores (Topic 14).

---

## 4. The x86 Vector Matrix Allocator (`kernel/irq/matrix.c`)

x86's scarce per-CPU vector space (Topic 02 §2.4, Topic 11 §4) needs careful bookkeeping: each device IRQ on a CPU must own one of ~256 vectors, minus exceptions and fixed system vectors. The **matrix allocator** is a per-CPU bitmap manager:

```c
struct cpumap   { ... unsigned long alloc_map[]; unsigned long managed_map[]; ... };  /* per CPU */
struct irq_matrix { ... struct cpumap __percpu *maps; ... };
```

Operations:
- **Reserve vs allocate.** `irq_matrix_reserve` reserves a *global* slot count (so the system guarantees it can place an IRQ somewhere) without binding a CPU yet; `irq_matrix_alloc` binds an actual `(CPU, vector)` from the least-loaded eligible CPU.
- **Managed reservations** (`irq_matrix_reserve_managed` / `irq_matrix_alloc_managed`) back **managed-affinity** IRQs (§2): vectors are pre-reserved on the target CPUs so a managed IRQ is guaranteed a vector when its CPU comes online.
- **`irq_matrix_online`/`offline`** track CPUs entering/leaving (hotplug, §5).
- **`irq_matrix_available`** reports headroom — the source of "vector allocation failed" when a system over-commits MSI-X across CPUs (Topic 11 §7).

This whole allocator is **x86-specific**; arm64 (GIC-ITS LPIs) has no equivalent per-CPU vector pressure, so it doesn't use it. The vector domain (`arch/x86/kernel/apic/vector.c`) is its consumer in the MSI/IOAPIC hierarchy.

---

## 5. CPU Hotplug (`kernel/irq/cpuhotplug.c`, `migration.c`)

When a CPU goes offline, every IRQ that could land on it must move:

```c
void irq_migrate_all_off_this_cpu(void)         /* called late in CPU-down */
{   for each desc affined to this CPU:
        affinity_broken = migrate_one_irq(desc);
}
static bool migrate_one_irq(struct irq_desc *desc)
{   ...
    irq_force_complete_move(desc);              /* finish any pending affinity move first */
    /* re-affine to a still-online CPU in the mask; if none, break affinity / shut down managed */
}
```

- **Unmanaged IRQs** are re-affined to a surviving CPU in their mask (or, if none remain, affinity is "broken" to any online CPU so the device keeps working).
- **Managed IRQs** (§2) whose entire mask went offline are **shut down** (`irq_shutdown`) rather than relocated — the queue is simply parked until one of its CPUs returns. On CPU-up, **`irq_affinity_online_cpu`** restores them.
- **`irq_force_complete_move`** (the chip op from Topic 03) resolves a half-finished affinity change before migrating, so an in-flight interrupt isn't lost. `IRQD_MOVE_PCNTXT` marks chips that can move affinity from process context immediately rather than deferring to the next interrupt.
- Per-CPU IRQs (Topic 12) aren't migrated — they're disabled on that CPU and re-enabled when it returns.

This is the bridge to Topic 17 (init/bring-up): the same `irq_affinity_online_cpu` runs for secondary CPUs at boot.

---

## 6. Observability

- **`/proc/interrupts`** — the per-CPU columns show the *actual* distribution; an IRQ landing entirely on one CPU despite a broad `smp_affinity` means single-target `effective_affinity`.
- **`/proc/irq/N/smp_affinity` / `smp_affinity_list`** — read/write the requested mask (rejected for managed IRQs).
- **`/proc/irq/N/effective_affinity[_list]`** — where it actually lands.
- **`/proc/irq/N/node`** — the NUMA node; `affinity_hint` — the driver's suggestion.
- **`/sys/kernel/debug/irq/domains/VECTOR`** (x86) — matrix usage and headroom; **`/sys/kernel/debug/irq/irqs/N`** shows `IRQD_AFFINITY_MANAGED`.
- `irqbalance --debug`, and dmesg `vector allocation failed` / `IRQ N: no longer affine to CPU` messages.

---

## 7. Common Pitfalls

- **Editing `effective_affinity` expectations** — reading `smp_affinity` and being surprised the IRQ only hits one CPU; check `effective_affinity`.
- **Trying to pin a managed IRQ** — `/proc/irq/N/smp_affinity` write returns `-EIO`; managed IRQs are kernel-owned. Use the driver's queue count instead.
- **Leaving `irqbalance` on for a latency/RT setup** — it will move IRQs onto your isolated cores. Disable it and pin (or rely on managed affinity + `isolcpus`).
- **x86 vector exhaustion under hotplug** — bringing CPUs offline concentrates vectors; managed reservations exist to prevent this, but over-committed MSI-X can still fail (`irq_matrix_available` near zero).
- **Assuming affinity changes apply instantly** — for many chips the move happens on the IRQ's *next* occurrence (via `pending_mask`); a silent device won't move until it fires.
- **Forgetting per-CPU IRQs aren't balanced** — trying to set affinity on a per-CPU line is meaningless (Topic 12).

---

## 8. Further Reading & Source Pointers

### Documentation
- `Documentation/core-api/irq/irq-affinity.rst` — managed vs unmanaged, the spreading algorithm.
- `Documentation/PCI/msi-howto.rst` — `PCI_IRQ_AFFINITY`.
- `irqbalance(1)` man page.

### Source to read
- `kernel/irq/affinity.c` `irq_create_affinity_masks` — the spreading algorithm.
- `kernel/irq/matrix.c` — `irq_matrix_reserve`/`alloc`/`alloc_managed`.
- `kernel/irq/cpuhotplug.c` `migrate_one_irq` / `irq_affinity_online_cpu`.

### Mental model
> Affinity is a *requested* CPU set (`affinity`) that the chip reduces to where it *actually* lands (`effective_affinity`). Multiqueue devices use **managed** affinity — the kernel spreads queue IRQs across CPUs and owns the mapping (userspace can't touch it). On x86 each placement consumes a scarce per-CPU **vector** tracked by the matrix allocator. CPU hotplug migrates unmanaged IRQs to survivors and parks/restores managed ones.

---

**Next: Topic 14 — Time Accounting, Latency, and PREEMPT_RT** (the temporal consequences of everything so far: how IRQ time is measured, where latency comes from, and the full RT model).
