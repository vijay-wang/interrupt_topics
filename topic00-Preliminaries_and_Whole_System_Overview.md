# Topic 00 — Preliminaries and Whole-System Overview

> **Standing requirements check (R1/R2/R3):** This topic is the orientation map, written to be **comprehensive** at the survey level (R1 — it names every later mechanism and where it lives), in **learnable order** (R2 — it defines the vocabulary and the end-to-end spine that all later topics are coordinates on), and **against this source tree** (R3 — every file path and symbol below is from `v7.1-rc6`; verify with `grep`). The three requirements live in `interrupt-topics-plan.md`; re-read them before each topic.
>
> **Position in the plan:** the entry chapter. Read it once now and again after Topic 03 — once you know `irq_desc`, the map below snaps into focus.
>
> **Goals after reading:**
> 1. State precisely what an interrupt is, and how it differs from an exception, a trap, a syscall, and a fault.
> 2. Draw the end-to-end path from "a device asserts a wire" to "a C function runs" to "deferred work finishes". This path is the spine of the whole series.
> 3. Never again confuse **hwirq** with the **Linux IRQ number (virq)**.
> 4. Name the four execution contexts and the one word (`preempt_count`) that encodes which one you are in.
> 5. Place every later topic on the map and know which file to open.
>
> **Source files referenced:** `include/linux/interrupt.h`, `include/linux/irqreturn.h`, `include/linux/preempt.h`, `include/linux/irqdesc.h`, `include/linux/irq-entry-common.h`, `kernel/irq/`, `kernel/softirq.c`, `arch/x86/...`, `arch/arm64/...`, `drivers/irqchip/`.

---

## 1. What an Interrupt Is (and Is Not)

An **interrupt** is an asynchronous signal from hardware that diverts a CPU from whatever it was running into a kernel handler. "Asynchronous" is the key word: it is unrelated to the instruction the CPU happened to be executing.

Distinguish the whole family — Linux and the architectures lump several things under "the interrupt machinery", but they are *not* the same:

| Event | Synchronous? | Source | Example | Linux entry |
|---|---|---|---|---|
| **Interrupt (IRQ)** | No (async) | external device via a controller | NIC packet arrived, disk DMA done | flow handler → `irqaction` |
| **Exception / fault** | Yes (caused by an instruction) | the CPU itself | page fault, #GP, alignment | a dedicated handler (e.g. `do_page_fault`) |
| **Trap** | Yes | a deliberate instruction | `int3` breakpoint, `#DB` | debug/trace handlers |
| **Syscall** | Yes | `syscall`/`svc` instruction | `read(2)` | the syscall entry path |
| **NMI** | No (async, unmaskable) | special line / perf / watchdog | hardlockup detector | NMI entry (Topic 12) |
| **IPI** | No (async) | another CPU | reschedule, TLB shootdown | IPI handler (Topic 12) |
| **MSI/MSI-X** | No (async) | a device *memory write* | modern PCIe device | same flow handlers, different delivery (Topic 11) |

This series is about the **asynchronous** side (IRQ, NMI, IPI, MSI) and the kernel infrastructure that dispatches them. Synchronous exceptions/syscalls share the *low-level arch entry* (Topic 01) but then diverge; we cover only the shared entry, not the fault handlers themselves.

> **Why interrupts at all?** The alternative is polling — spinning while you ask "is the device done yet?". Interrupts let the CPU do useful work and be told when something happens. The cost is latency and concurrency complexity, which is what the rest of this series manages. (At very high event rates the kernel deliberately switches *back* to polling: NAPI and `irq_poll` — Topic 09 — because at that point interrupts cost more than they save.)

---

## 2. The End-to-End Path (the spine of the series)

Memorize this pipeline. Every topic is a zoom-in on one stage.

```
   ┌─────────┐  wire / message   ┌──────────────────┐   vector/   ┌────────────────────┐
   │ device  │ ────────────────► │ interrupt        │ ──pin────►  │ CPU core           │
   │ (NIC,   │   asserts IRQ     │ controller       │             │ (stops, masks,     │
   │  disk)  │                   │ (APIC / GIC …)    │             │  vectors to entry) │
   └─────────┘                   └──────────────────┘             └─────────┬──────────┘
        Topic 02 (controllers)                                              │ Topic 01
                                                                            ▼
   ┌─────────────────────────────────────────────────────────────────────────────────┐
   │ ARCH LOW-LEVEL ENTRY  (arch/x86 idtentry / arch/arm64 vectors)                    │
   │   save regs → switch to IRQ stack → irqentry_enter() (generic entry, Topic 01)    │
   └───────────────────────────────────────────┬─────────────────────────────────────┘
                                                ▼
   ┌─────────────────────────────────────────────────────────────────────────────────┐
   │ GENERIC IRQ CORE  (kernel/irq/)                                                   │
   │   generic_handle_irq(virq)  ── uses the IRQ DOMAIN (Topic 04) to map hwirq→virq   │
   │      → irq_desc->handle_irq  = a FLOW HANDLER (Topic 05: level/edge/fasteoi/…)    │
   │          → handle_irq_event() walks irq_desc->action  (the irqaction chain)       │
   │              → your handler(irq, dev_id)  ── the TOP HALF (Topic 06)              │
   │                  returns IRQ_HANDLED / IRQ_NONE / IRQ_WAKE_THREAD                 │
   └───────────────────────────────────────────┬─────────────────────────────────────┘
                                                ▼ defer the heavy work
   ┌─────────────────────────────────────────────────────────────────────────────────┐
   │ BOTTOM HALF (deferred work)                                                       │
   │   softirq (Topic 08) / tasklet · BH-workqueue · irq_work (Topic 09) /             │
   │   threaded IRQ kthread (Topic 07) / workqueue (process context)                   │
   └───────────────────────────────────────────┬─────────────────────────────────────┘
                                                ▼
   ┌─────────────────────────────────────────────────────────────────────────────────┐
   │ RETURN FROM INTERRUPT  (irqentry_exit, Topic 01)                                  │
   │   run pending softirqs · check need_resched · deliver signals on return-to-user   │
   └─────────────────────────────────────────────────────────────────────────────────┘
```

The **top half / bottom half** split is the single most important idea: the handler that runs in interrupt context must be *short* (interrupts may be masked, nothing may sleep), so it does the urgent minimum and *defers* the rest to a context where more is allowed. Topics 06–09 are entirely about that split.

---

## 3. Two Number Namespaces You Must Never Confuse

A single physical interrupt has **two** numbers in Linux:

- **hwirq** (`irq_hw_number_t`) — the number the *controller* uses. GIC SPI 42, IO-APIC pin 16, an MSI index. Meaningful only relative to one controller, and these collide across controllers (every GIC has a hwirq 30).
- **Linux IRQ number / `virq`** (the plain `unsigned int irq`) — the kernel-global token. It indexes `irq_desc`. This is what `request_irq()` takes and what `/proc/interrupts` shows in the left column.

The **IRQ domain** layer (Topic 04) owns the `hwirq → virq` mapping. A controller driver says "I have hwirq 42"; the domain allocates a free virq, creates an `irq_desc`, and records the reverse map. When the controller later delivers hwirq 42, `generic_handle_domain_irq(domain, 42)` looks up the virq and dispatches.

> **Rule of thumb:** if a number came from device tree, ACPI, or a controller register, it is an **hwirq**. If it indexes `irq_desc` or you pass it to `request_irq`/`free_irq`/`enable_irq`, it is a **virq**. The columns in `/proc/interrupts` are virqs; the "edge/level + controller" text on the right describes the hwirq side.

---

## 4. Execution Contexts (and the one word that names them)

At any instant a CPU is in exactly one of these contexts, and the rules differ sharply:

| Context | May sleep? | May call blocking alloc? | Typical work | How to detect |
|---|---|---|---|---|
| **process / task** | yes | yes (`GFP_KERNEL`) | syscalls, kthreads, threaded IRQ handlers | `in_task()` |
| **softirq / bottom-half** | **no** | no (`GFP_ATOMIC` only) | network RX, timers, tasklets | `in_serving_softirq()` |
| **hardirq (interrupt)** | **no** | no | top-half handlers, flow handlers | `in_hardirq()` |
| **NMI** | **no** (and almost nothing else) | no | watchdog, perf | `in_nmi()` |

These predicates are literally bit-tests on one per-CPU word, **`preempt_count`** (`include/linux/preempt.h`):

```c
#define in_nmi()             (nmi_count())
#define in_hardirq()         (hardirq_count())
#define in_serving_softirq() (softirq_count() & SOFTIRQ_OFFSET)
#define in_interrupt()       (irq_count())     /* hardirq | softirq | nmi */
#define in_task()            (!(preempt_count() & (NMI_MASK|HARDIRQ_MASK|SOFTIRQ_OFFSET)))
```

`preempt_count` packs four sub-counters (preempt-disable depth, softirq depth, hardirq depth, NMI depth) into one word. Entering a hardirq adds `HARDIRQ_OFFSET`; serving a softirq adds `SOFTIRQ_OFFSET`; etc. That is how `might_sleep()`, lockdep, and `GFP_KERNEL` know they are in a forbidden context. **Topic 10** dissects this word bit by bit; for now, just hold the mental model: *one word tells the kernel "where am I, and what am I allowed to do".*

The handler return type ties into this (`include/linux/irqreturn.h`):

```c
enum irqreturn { IRQ_NONE = 0, IRQ_HANDLED = 1, IRQ_WAKE_THREAD = 2 };
```

`IRQ_NONE` = "not my device" (shared IRQs), `IRQ_HANDLED` = done, `IRQ_WAKE_THREAD` = "wake my threaded handler to finish in task context" (Topic 07).

---

## 5. Architecture Realizations at a Glance

Linux hides two very different hardware worlds behind one abstraction (`irq_chip` + `irq_domain`). You still need to recognize both:

**x86_64** (Topics 01, 02, 11, 13):
- The **IDT** (Interrupt Descriptor Table) has 256 entries; an interrupt's *vector* (0–255) indexes it. Vectors 0–31 are CPU exceptions; the rest are device/IPI vectors — a **scarce, per-CPU resource** managed by the matrix allocator (Topic 13).
- The **Local APIC** (per-CPU) and **I/O APIC** route external lines; **MSI** writes go through the APIC too. `idtentry` macros (`arch/x86/include/asm/idtentry.h`) generate the entry stubs.

**arm64** (Topics 01, 02):
- A 16-entry **exception vector table** (`VBAR_EL1` → `vectors` in `arch/arm64/kernel/entry.S`) split by exception level and sync-vs-async; IRQ and FIQ are async entries.
- The **GIC** (Generic Interrupt Controller, v2/v3/v4) distributes SPIs/PPIs/SGIs; GICv3+ adds the **ITS** for MSI (LPIs). No scarce per-CPU vector space like x86 — the model is different, which is exactly why the `irq_domain` abstraction earns its keep.

The payoff of the abstraction: a driver calls `request_irq(virq, …)` and a flow handler runs `irq_desc->handle_irq`, *identically* on both. Only the `irq_chip` (mask/unmask/eoi) and the domain differ underneath.

---

## 6. Source-Tree Geography

Keep this handy; each topic drills into one row.

```
kernel/irq/                 the generic IRQ core ("genirq")
  irqdesc.c                 irq_desc lifecycle, sparse-IRQ storage, generic_handle_*   (Topic 03/05)
  irqdomain.c               hwirq<->virq mapping, hierarchical domains                 (Topic 04)
  chip.c                    irq_chip helpers + the FLOW HANDLERS                        (Topic 05)
  handle.c                  handle_irq_event, the action-chain walk                    (Topic 05)
  manage.c                  request_irq/free_irq/threaded IRQs/affinity                 (Topic 06/07/13)
  msi.c                     MSI framework                                              (Topic 11)
  ipi.c, ipi-mux.c          inter-processor interrupts                                 (Topic 12)
  matrix.c                  x86 vector bitmap allocator                                (Topic 13)
  cpuhotplug.c, migration.c moving IRQs across CPUs                                    (Topic 13)
  spurious.c                "nobody cared" detection                                   (Topic 05/16)
  resend.c                  re-injecting lost edges                                    (Topic 05/15)
  pm.c                      suspend/resume of irq_descs                                (Topic 15)
  proc.c, debugfs.c         /proc/irq, /sys/kernel/debug/irq                           (Topic 16)
  generic-chip.c, devres.c, irq_sim.c, dummychip.c   helpers/tests

kernel/softirq.c            softirqs, tasklets, local_bh_*                             (Topic 08/09)
kernel/irq_work.c           hardirq/NMI-safe callbacks                                 (Topic 09)

include/linux/
  irq.h        irq_chip, irq_data, irq_domain accessors        irqdesc.h  irq_desc
  interrupt.h  request_irq, IRQF_*, softirq vectors, tasklets  irqreturn.h irqreturn_t
  irqflags.h   local_irq_* / lockdep IRQ state                 preempt.h   preempt_count, in_*()
  irq-entry-common.h  irqentry_enter/exit (generic entry)      msi.h       msi_desc

arch/x86/kernel/    irq.c, irqinit.c, idt.c, apic/      arch/x86/include/asm/idtentry.h
arch/arm64/kernel/  irq.c, entry.S, entry-common.c
drivers/irqchip/    irq-gic*.c, irq-apple-aic.c, … (the real controller drivers)       (Topic 02)
```

---

## 7. Common Pitfalls (at this orientation level)

- **Confusing hwirq and virq** (see §3). The #1 source of "my IRQ never fires" — you mapped or printed the wrong number.
- **Thinking the handler is the whole story.** The top half is deliberately tiny; the real work is the bottom half. If you only read `request_irq`, you have seen ~20% of the path.
- **Assuming `/proc/interrupts` counts hardware events.** It counts *handled* events per virq per CPU; a shared IRQ lumps several devices into one row, and coalescing/threading changes the count's meaning.
- **Treating x86 and arm64 as the same.** The vector-scarcity model (x86) vs the GIC model (arm64) leads to genuinely different code in the controller and affinity topics. The plan requires both be covered (R1) — so do not skip the arch you don't use.
- **"Interrupts off" ≠ "preemption off" ≠ "in interrupt".** Three different states, three different `preempt_count` bits (Topic 10). Mixing them up produces both deadlocks and `scheduling while atomic` bugs.

---

## 8. Further Reading and Source Pointers

### Kernel documentation (in tree)
- `Documentation/core-api/genericirq.rst` — the canonical genirq overview (flow handlers, chips, domains).
- `Documentation/core-api/irq/` — `irq-domain.rst`, `irq-affinity.rst`, `concepts.rst`, `irqflags-tracing.rst`.
- `Documentation/PCI/msi-howto.rst` — MSI from the driver side.

### Source entry points to read first
- `include/linux/interrupt.h` — the driver-facing API surface in one header.
- `kernel/irq/chip.c` — the flow handlers; the clearest place to *see* the mask/ack/eoi state machine.
- `include/linux/preempt.h` — the `in_*()` predicates and `preempt_count` layout.

### Mental model to walk away with
> A device pulls a wire; a controller turns that into a vector; the CPU jumps to a stub; the generic core maps **hwirq → virq**, picks a **flow handler**, and runs your **top half**; the top half defers heavy work to a **bottom half** in a context where more is allowed; on the way out, pending softirqs run and the scheduler gets a chance. Everything else in this series is detail bolted onto that sentence.

---

**Next: Topic 01 — CPU-Level Interrupt Mechanics** (what the silicon and the lowest kernel layer do, on x86_64 and arm64).
