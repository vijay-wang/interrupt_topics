# Topic 06 — Registering & Managing Handlers (the driver API)

> **Standing requirements check (R1/R2/R3):** R1 — the complete request/free/enable/disable surface, the full `IRQF_*` table from the header, the top-half contract, and the management calls. R2 — placed after the machinery (Topics 03–05) so the API reads as "installing the `irqaction` and choosing the flow handler you already understand", and before threaded IRQs (Topic 07), which is just one option of this same call. R3 — signatures and flag values from `include/linux/interrupt.h`; behaviour from `kernel/irq/manage.c` (`__setup_irq`, `request_threaded_irq`) in `v7.1-rc6`. Re-read R1/R2/R3 in the plan.
>
> **Builds on:** Topic 03 (`irqaction`, `irq_desc`, `dev_id`), Topic 05 (the flow handler that will call your handler; `IRQ_NONE`/`IRQ_HANDLED`/`IRQ_WAKE_THREAD`).
>
> **Position in the plan:** the surface 99% of driver authors touch. Topic 07 expands the threaded variant; Topics 08–09 cover where the deferred work goes.
>
> **Source:** `include/linux/interrupt.h` (API + `IRQF_*`), `kernel/irq/manage.c` (`request_threaded_irq`, `__setup_irq`, `free_irq`, `__disable_irq`/`enable_irq`, `synchronize_irq`), `kernel/irq/devres.c` (`devm_*`).

---

## 1. Position & the One Core Call

Everything funnels into **`request_threaded_irq()`**; the rest are thin wrappers:

```c
int request_threaded_irq(unsigned int irq, irq_handler_t handler,
                         irq_handler_t thread_fn, unsigned long flags,
                         const char *name, void *dev);

/* request_irq() is literally: */
static inline int request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
                              const char *name, void *dev)
{ return request_threaded_irq(irq, handler, NULL, flags | IRQF_COND_ONESHOT, name, dev); }
```

`request_irq` = a threaded request with no thread function (top-half only). `request_threaded_irq` with both a primary `handler` and a `thread_fn` is Topic 07. So *one* call, several shapes.

What it does under the hood (`__setup_irq` in `kernel/irq/manage.c`):
1. Allocate and fill an `irqaction` (`handler`, `thread_fn`, `flags`, `name`, `dev_id`).
2. For shared lines, verify the new action's flags/trigger are compatible with existing ones.
3. If threaded (or forced-threaded), create the kthread (Topic 07).
4. Call the chip's `irq_request_resources`, set the trigger type, install/keep the flow handler (Topic 05), and (unless `IRQF_NO_AUTOEN`) enable the line.

It takes a **virq** (from Topic 04: `irq_of_parse_and_map`, `pci_irq_vector`, `platform_get_irq`, …) — never an hwirq.

---

## 2. The Request Family

| Call | Use |
|---|---|
| `request_irq` | the common case: top-half-only handler |
| `request_threaded_irq` | primary handler + threaded `thread_fn` (Topic 07) |
| `request_any_context_irq` | let the core decide hardirq vs thread by config (used by drivers that work either way) |
| `request_percpu_irq` / `request_percpu_irq_affinity` | per-CPU lines; `dev_id` is `void __percpu *` (Topic 12) |
| `request_nmi` / `request_percpu_nmi` | register an NMI handler (Topic 12) — strict constraints |
| `devm_request_irq` / `devm_request_threaded_irq` | resource-managed: auto-freed on driver detach (`kernel/irq/devres.c`) |

Prefer the **`devm_`** variants in device drivers — they remove the #1 cleanup bug (forgetting `free_irq` on probe failure / remove).

Returns 0 on success; `-EBUSY` (shared-flag mismatch), `-ENOMEM`, `-ENOTCONN` (`irq == IRQ_NOTCONNECTED`), `-EINVAL`. The `__must_check` attribute means **you must check the return** — an ignored failure leaves the device with no handler.

---

## 3. The `IRQF_*` Flags (`include/linux/interrupt.h`)

```
Trigger (override the firmware-declared type; usually leave 0 and trust DT/ACPI):
  IRQF_TRIGGER_RISING / FALLING / HIGH / LOW / NONE

Sharing & probing:
  IRQF_SHARED        line may be shared; all sharers must set it; dev_id must be unique & non-NULL
  IRQF_PROBE_SHARED  tolerate mismatch during autoprobe
  IRQF_COND_ONESHOT  fold in ONESHOT only if a sharer needs it (set implicitly by request_irq)

Threading (Topic 07):
  IRQF_ONESHOT       keep the line masked until the threaded handler completes
                     (mandatory when thread_fn is set with no primary handler)
  IRQF_NO_THREAD     never force-thread this IRQ (timers, some low-latency lines)

CPU / balancing (Topic 12/13):
  IRQF_PERCPU        a per-CPU interrupt
  IRQF_NOBALANCING   exclude from irqbalance / affinity spreading

Power management (Topic 15):
  IRQF_NO_SUSPEND    keep armed across system suspend (timers, wakeup-critical)
  IRQF_EARLY_RESUME  resume this IRQ early (before normal devices)
  IRQF_FORCE_RESUME  force-resume even if NO_SUSPEND
  IRQF_COND_SUSPEND  for a shared line mixing NO_SUSPEND and normal actions

Misc:
  IRQF_IRQPOLL       eligible for the irqpoll watchdog (the "is anything alive?" timer)
  IRQF_NO_AUTOEN     do NOT auto-enable on request; caller enables later with enable_irq()
  IRQF_NO_DEBUG      exclude from some debug/lockup accounting
  IRQF_TIMER         = __IRQF_TIMER | IRQF_NO_SUSPEND | IRQF_NO_THREAD  (the timer-IRQ bundle)
```

The three you will set most: **`IRQF_SHARED`** (legacy/INTx lines), **`IRQF_ONESHOT`** (threaded handlers), and trigger flags only when the firmware can't be trusted.

---

## 4. The Top-Half Handler Contract

```c
static irqreturn_t my_handler(int irq, void *dev_id) { ... return IRQ_HANDLED; }
```

Runs in **hardirq context** (Topic 00 §4), called from the flow handler's action walk (Topic 05 §3), with **interrupts disabled** and often the line masked. Therefore:

- **Must not sleep.** No `mutex`, no `msleep`, no `wait_event`, no blocking allocation (`GFP_KERNEL`). Only `GFP_ATOMIC`, spinlocks (`spin_lock`, not `_irqsave` — IRQs are already off), and non-blocking work.
- **Must be short.** Long handlers raise IRQ-off latency (Topic 14) and delay every other interrupt on the CPU.
- **Must identify shared interrupts.** On a shared line, check your device and return **`IRQ_NONE`** if it wasn't you (Topic 05 §4).
- **Return value:** `IRQ_HANDLED` (serviced), `IRQ_NONE` (not mine), or `IRQ_WAKE_THREAD` (defer the rest to `thread_fn`, Topic 07). `IRQ_RETVAL(x)` maps a bool to HANDLED/NONE.
- **Do not re-enable interrupts** — the core `WARN_ONCE`s if you return with IRQs on (Topic 05 §3).

The discipline is "acknowledge the device, grab the minimal data (or just wake the thread), get out". Heavy lifting → bottom half (Topics 07–09).

---

## 5. Releasing & Gating

```c
const void *free_irq(unsigned int irq, void *dev_id);   /* returns the dev_id cookie; identifies the action on shared lines */
void disable_irq(unsigned int irq);        /* sync: also waits for any running handler/thread to finish */
void disable_irq_nosync(unsigned int irq); /* async: returns immediately */
bool disable_hardirq(unsigned int irq);    /* sync wrt the hardirq only (not the thread) */
void enable_irq(unsigned int irq);
void synchronize_irq(unsigned int irq);    /* wait until no handler/thread is running (no disable) */
```

- **`disable_irq` is nested/counted** via `desc->depth`: N disables need N enables. It masks at the chip *and* waits for in-flight handlers — safe to call before touching data a handler uses. `disable_irq_nosync` skips the wait (use when you can't block, but then you must `synchronize_irq` separately before freeing shared state).
- **`free_irq` removes one `irqaction`** (matched by `dev_id` on shared lines), waits for it to finish, and disables the line if it was the last action. Always pass the *same* `dev_id` you registered.
- **`synchronize_irq`** waits on `desc->wait_for_threads` / in-flight handlers without changing the enable state — the tool for "make sure the handler isn't running right now".
- There is a `DEFINE_LOCK_GUARD_1(disable_irq, …)` cleanup guard in this tree for scoped disable/enable.

Other management: `irq_set_irq_type` (change trigger), `irq_set_affinity_hint`/`irq_set_affinity` (Topic 13), `irq_get_irqchip_state`/`irq_set_irqchip_state` (query/force pending/masked at the chip), `irq_wake_thread` (manually wake the threaded handler), `enable_irq_wake` (Topic 15).

---

## 6. Observability

- After `request_irq`, the line appears in `/proc/interrupts` with your `name` in the right column and counts per CPU.
- `/sys/kernel/irq/N/actions` lists the registered action names (shared lines show several).
- `/proc/irq/N/` exists once an action is registered; `spurious` there shows unhandled/last-unhandled (Topic 05/16).
- Tracepoints `irq:irq_handler_entry/exit` fire around your handler.

---

## 7. Common Pitfalls

- **Ignoring the return value** (`__must_check`) — a failed `request_irq` you didn't notice = a silent dead device.
- **Non-unique or NULL `dev_id` on a shared line** — `request_irq` rejects NULL `dev_id` for `IRQF_SHARED`; a duplicate makes `free_irq` ambiguous. Use the device/private pointer.
- **Requesting the IRQ before the device can't generate one** — enable the line only after the device is quiesced, or take the first interrupt before you're ready. `IRQF_NO_AUTOEN` + later `enable_irq` solves ordering.
- **Unbalanced `disable_irq`/`enable_irq`** — the `depth` counter means a missing enable leaves the IRQ off forever; an extra enable warns.
- **Calling `free_irq` from within the handler / atomic context** — `free_irq` (and sync `disable_irq`) can block waiting for the handler/thread; never from hardirq/softirq.
- **Sleeping in the top half** — the single most common new-driver bug; if you need to sleep, use `request_threaded_irq` (Topic 07) or a workqueue (Topic 09).
- **Forgetting cleanup on probe error paths** — use `devm_request_irq`.

---

## 8. Further Reading & Source Pointers

### Documentation
- `Documentation/core-api/genericirq.rst` — "Highlevel Driver API".
- `Documentation/driver-api/basics.rst` and subsystem MSI/PCI docs for the virq sources.

### Source to read
- `kernel/irq/manage.c` — `request_threaded_irq` → `__setup_irq` (the shared-IRQ checks and thread setup), `free_irq` → `__free_irq`, `__disable_irq`/`enable_irq`, `synchronize_irq`.
- `kernel/irq/devres.c` — how `devm_request_irq` hooks device teardown.

### Mental model
> `request_irq(virq, handler, flags, name, dev_id)` builds an `irqaction`, hangs it on the `irq_desc`, picks/keeps the flow handler, and enables the line. Your `handler` runs in hardirq context — short, non-sleeping, shared-aware — and either finishes the work or returns `IRQ_WAKE_THREAD` to push it to a kthread. `free_irq`/`disable_irq`/`synchronize_irq` are the inverse and the gates.

---

**Next: Topic 07 — Threaded IRQs & Forced Threading** (the `thread_fn` half of `request_threaded_irq`, and the model PREEMPT_RT makes universal).
