# Topic 05 — Flow Handlers: How One IRQ Is Processed

> **Standing requirements check (R1/R2/R3):** R1 — covers every flow handler in `kernel/irq/chip.c`, the exact mask/ack/eoi ordering each performs and *why it matches its trigger type*, the action-chain walk, shared IRQs, spurious detection, and resend. R2 — placed right after the data model (Topic 03) and domains (Topic 04): you now have a `desc` with a `chip` and an `action` list, and this is the function that drives them; it must precede the driver API (Topic 06) because that API just *installs* what this topic *runs*. R3 — handler bodies quoted from `kernel/irq/chip.c` / `kernel/irq/handle.c` in `v7.1-rc6` (note the modern `guard(raw_spinlock)` style). Re-read R1/R2/R3 in the plan.
>
> **Builds on:** Topic 02 (level vs edge, EOI protocols), Topic 03 (`irq_desc`, `irq_chip`, `irqaction`, the `IRQS_*` state), Topic 04 (`generic_handle_domain_irq` got us here).
>
> **Position in the plan:** the engine room. Everything before this *delivered* an interrupt to `desc->handle_irq`; everything after this is about *who registers* handlers and *where the work goes*.
>
> **Source:** `kernel/irq/chip.c` (the flow handlers), `kernel/irq/handle.c` (`handle_irq_event*`, the action walk), `kernel/irq/spurious.c` (`note_interrupt`), `kernel/irq/resend.c`.

---

## 1. Position: From `generic_handle_*` to Your Handler

```
generic_handle_domain_irq(domain, hwirq)        (Topic 04)
   → irq_resolve_mapping → irq_desc
   → handle_irq_desc(desc)
        → desc->handle_irq(desc)                 ← THE FLOW HANDLER (this topic)
             drives desc->irq_data.chip through mask/ack/eoi
             → handle_irq_event(desc)
                  → walks desc->action chain → your handler(irq, dev_id)   (Topic 06)
```

A **flow handler** is the per-trigger-type state machine. The core picks *which* one when the IRQ is set up (`irq_set_handler` / the chip's `map`): a level line gets `handle_level_irq`, an edge line `handle_edge_irq`, a "smart" controller `handle_fasteoi_irq`, a per-CPU line `handle_percpu_devid_irq`. Its job: sequence the chip's `irq_mask`/`irq_ack`/`irq_eoi`/`irq_unmask` correctly for that trigger, run the registered handlers, and handle the corner cases (re-entrancy, lost edges, spurious).

---

## 2. The Flow Handlers (`kernel/irq/chip.c`)

All take just `struct irq_desc *desc` in this tree and take `desc->lock` via a cleanup guard. The differences are entirely in the *order* of chip operations.

### 2.1 `handle_level_irq` — level-triggered

```c
void handle_level_irq(struct irq_desc *desc)
{
    guard(raw_spinlock)(&desc->lock);
    mask_ack_irq(desc);                 /* MASK first: the line is still asserted! */
    if (!irq_can_handle(desc)) return;  /* disabled / no action / in-progress */
    kstat_incr_irqs_this_cpu(desc);
    handle_irq_event(desc);             /* run handlers; device de-asserts the line */
    cond_unmask_irq(desc);              /* UNMASK only after the device is quiet */
}
```

**Why this order:** a level line stays asserted until the device is serviced. If you didn't **mask** before running the handler, the same still-asserted line would immediately re-interrupt forever. So: mask+ack → handle (the handler clears the device condition, dropping the line) → unmask. This is the canonical "mask around the handler" dance.

### 2.2 `handle_edge_irq` — edge-triggered

```c
void handle_edge_irq(struct irq_desc *desc)
{
    guard(raw_spinlock)(&desc->lock);
    if (!irq_can_handle(desc)) {        /* can't run now → remember it */
        desc->istate |= IRQS_PENDING;
        mask_ack_irq(desc);
        return;
    }
    kstat_incr_irqs_this_cpu(desc);
    desc->irq_data.chip->irq_ack(&desc->irq_data);   /* ACK only — do NOT mask the line */
    do {
        if (unlikely(!desc->action)) { mask_irq(desc); return; }
        if (unlikely(desc->istate & IRQS_PENDING)) {  /* a new edge arrived while masked */
            if (!irqd_irq_disabled(...) && irqd_irq_masked(...)) unmask_irq(desc);
        }
        handle_irq_event(desc);
    } while ((desc->istate & IRQS_PENDING) && !irqd_irq_disabled(&desc->irq_data));
}
```

**Why this order:** an edge is a momentary event — masking the line isn't needed to stop re-assertion, but an edge that arrives *while the handler runs* must not be lost. So edge: **ack** (don't mask), run the handler, and if a new edge set `IRQS_PENDING` during handling, loop and run again. The `IRQS_PENDING` re-loop is the whole reason edge IRQs aren't dropped. If the IRQ couldn't run (disabled), it is remembered as pending and re-injected later (resend, §5).

### 2.3 `handle_fasteoi_irq` — "transparent" / modern controllers

```c
void handle_fasteoi_irq(struct irq_desc *desc)
{
    struct irq_chip *chip = desc->irq_data.chip;
    guard(raw_spinlock)(&desc->lock);
    if (!irq_can_handle_pm(desc)) {           /* suspended/in-progress: maybe resend */
        if (irqd_needs_resend_when_in_progress(&desc->irq_data)) desc->istate |= IRQS_PENDING;
        cond_eoi_irq(chip, &desc->irq_data);
        return;
    }
    if (!irq_can_handle_actions(desc)) { mask_irq(desc); cond_eoi_irq(chip,...); return; }
    kstat_incr_irqs_this_cpu(desc);
    if (desc->istate & IRQS_ONESHOT) mask_irq(desc);   /* threaded oneshot: keep masked till thread done */
    handle_irq_event(desc);
    cond_unmask_eoi_irq(desc, chip);          /* EOI (and unmask if not oneshot) */
    if (unlikely(desc->istate & IRQS_PENDING)) check_irq_resend(desc, false);  /* affinity-race resend */
}
```

**Why:** modern controllers (GIC, APIC) do the masking/priority in hardware and only need a single **EOI** at the end. So fasteoi issues *no* explicit mask in the common case — just run the handlers and `irq_eoi`. The `IRQS_ONESHOT` mask is the hook for threaded IRQs (Topic 07): the line stays masked from here until the IRQ thread finishes. The trailing `check_irq_resend` handles a subtle affinity-change race where the next interrupt beats the previous one's completion on the new CPU. This is the most common handler on server hardware — read it twice.

### 2.4 The specialised handlers
- **`handle_simple_irq`** — no chip mask/ack/eoi at all; for software/demux IRQs where there's nothing to acknowledge.
- **`handle_percpu_irq` / `handle_percpu_devid_irq`** — for per-CPU lines (arch timer, PMU; Topic 12). No `desc->lock` (per-CPU, no cross-CPU contention), minimal ack/eoi. `devid` variant passes the per-CPU `dev_id`.
- **`handle_fasteoi_nmi`** — NMI-safe fasteoi (Topic 12): runs the handler and EOIs, skipping anything that isn't NMI-safe.
- **`handle_nested_irq`** — for a handler that is itself called from a *parent* threaded handler (e.g. an I2C GPIO expander demux); runs the action in thread context.
- **`handle_fasteoi_ack_irq` / `handle_fasteoi_mask_irq`** — fasteoi variants that additionally ack/mask for controllers needing it.
- **`handle_untracked_irq`** — like simple but excluded from spurious accounting.

---

## 3. Running the Handlers — the Action Chain (`kernel/irq/handle.c`)

`handle_irq_event` → `handle_irq_event_percpu` → `__handle_irq_event_percpu`:

```c
irqreturn_t __handle_irq_event_percpu(struct irq_desc *desc)
{
    irqreturn_t retval = IRQ_NONE;
    struct irqaction *action;
    for_each_action_of_desc(desc, action) {          /* walk the shared-IRQ chain */
        trace_irq_handler_entry(irq, action);
        res = action->handler(irq, action->dev_id);  /* YOUR TOP HALF (Topic 06) */
        trace_irq_handler_exit(irq, action, res);
        WARN_ONCE(!irqs_disabled(), "irq %u handler %pS enabled interrupts\n", ...);
        if (res == IRQ_WAKE_THREAD) __irq_wake_thread(desc, action);  /* Topic 07 */
        retval |= res;
    }
    return retval;
}
```

Key facts visible here:
- **Handlers run with interrupts disabled**; the `WARN_ONCE` catches a handler that wrongly re-enabled them.
- **`IRQ_WAKE_THREAD`** wakes the action's `thread_fn` (Topic 07); a `WAKE_THREAD` with no `thread_fn` is a driver bug and is warned.
- The `trace_irq_handler_entry/exit` tracepoints are your per-handler observability (Topic 16).
- This tree also has an optional **handler-duration check** (`irqhandler_duration_check_enabled` static branch) that flags overlong handlers — a latency-debugging aid (Topic 14/16).

---

## 4. Shared Interrupts (`IRQF_SHARED`)

Several devices on one line → several `irqaction`s chained by `->next`. The flow handler runs the *whole* chain on every interrupt; each handler must:
1. Check its device's status register: *is this my interrupt?*
2. If not, return **`IRQ_NONE`** immediately.
3. If yes, service it and return **`IRQ_HANDLED`**.

The combined `retval` is `IRQ_NONE` only if *every* handler disowned it — which feeds spurious detection (§5). Costs: every device's handler runs on every shared interrupt (latency), and a buggy "always return HANDLED" handler hides others. MSI (Topic 11) exists partly to kill sharing.

---

## 5. Spurious / Unhandled Detection (`kernel/irq/spurious.c`)

After the action chain, `note_interrupt()` inspects the result:
- If `retval == IRQ_NONE`, increment `desc->irqs_unhandled`. If unhandled events pile up (100k in a short window via the `irq_count`/`last_unhandled` aging), the core concludes the line is **screaming** (stuck asserted, nobody owning it) and prints the famous:
  > `irq N: nobody cared (try booting with the "irqpoll" option)`
  then **disables the IRQ** to keep the machine alive, and (with `irqpoll`) falls back to polling all handlers.
- This is a safety net against a misconfigured trigger type or a device asserting a line no driver claims.

`IRQS_WAITING` / autoprobe (`kernel/irq/autoprobe.c`) use the same machinery to detect which line a legacy device pulses during `probe_irq_on/off`.

---

## 6. Resend — Re-injecting Lost Interrupts (`kernel/irq/resend.c`)

An edge (or a fasteoi affinity-race, §2.3) that couldn't be handled when it arrived is marked `IRQS_PENDING`. `check_irq_resend()` re-injects it later:
- Preferred: ask the chip to re-trigger via **`irq_chip->irq_retrigger`** (hardware re-assert).
- Fallback (`CONFIG_HARDIRQS_SW_RESEND`): queue the `desc` on `resend_node` and raise a software path that re-runs the handler (used on resume, Topic 15, and where hardware can't retrigger).

This is what saves edge interrupts across `disable_irq`/`enable_irq` windows and suspend/resume.

---

## 7. Design Rationale (the "why" R1 demands)

- **Why a family of flow handlers instead of one?** Trigger types have genuinely different correctness requirements: level must mask-around-handle (or loop forever); edge must ack-and-re-check (or lose events); smart controllers need only EOI. Encoding each as a separate, well-tested function keeps every controller from re-implementing the dance.
- **Why is the EOI sometimes deferred (`IRQCHIP_EOI_THREADED`, `cond_unmask_eoi_irq`)?** For threaded oneshot IRQs the line must stay masked until the *thread* finishes, so the EOI/unmask happens after the thread, not in the hardirq. The flow handler coordinates with Topic 07.
- **Why `IRQS_PENDING` rather than just re-running?** It bounds re-entrancy and cooperates with `disable_irq` — a pending edge taken while disabled is replayed on enable, not dropped or spun on.
- **Why the spurious-disable safety net?** A screaming level IRQ with no owner would otherwise wedge the CPU in an interrupt storm. Disabling it trades one dead device for a live system.

---

## 8. Observability, Pitfalls, Further Reading

### Observability
- `/proc/interrupts` — the count rising tells you the flow handler ran; the flow type appears in the right column ("fasteoi", "edge", "level").
- Tracepoints `irq:irq_handler_entry` / `irq:irq_handler_exit` (per action, with the return value).
- `/sys/kernel/debug/irq/irqs/N` — shows `istate` (`IRQS_PENDING`, etc.) and which `handle_irq` is installed.

### Pitfalls
- **Returning `IRQ_HANDLED` unconditionally on a shared line** — masks other devices' "nobody cared" and breaks spurious detection. Always check "is it mine?" and return `IRQ_NONE` when not.
- **A handler that sleeps or re-enables IRQs** — caught by `WARN_ONCE`; flow handlers run in hardirq context with IRQs off.
- **Wrong trigger type → wrong flow handler** — declaring an edge device as level (or vice-versa) yields lost events or a screaming "nobody cared". Cross-check the DT/ACPI trigger (Topic 02/04).
- **Expecting unmask in fasteoi** — there isn't an explicit one in the common path; the controller handles masking. Custom chips that need masking should use `handle_fasteoi_mask_irq`.
- **Long top halves** — the flow handler runs them with IRQs disabled; move work to a bottom half (Topics 07–09).

### Further reading
- `Documentation/core-api/genericirq.rst` — "Highlevel Driver API" and "Flow handlers" sections.
- Source: read `handle_level_irq`, `handle_edge_irq`, `handle_fasteoi_irq` back-to-back in `kernel/irq/chip.c` — the contrast *is* the lesson.

### Mental model
> The flow handler is a trigger-type-specific recipe for "mask/ack at the right moment, run the action chain, EOI/unmask at the right moment, and don't lose or spin on edges". Level masks around the handler; edge acks and re-checks pending; fasteoi just EOIs and lets hardware mask. Your handler is one entry in the action chain it runs.

---

**Next: Topic 06 — Registering & Managing Handlers** (`request_irq` and friends: how the `irqaction` and flow handler this topic ran got installed in the first place).
