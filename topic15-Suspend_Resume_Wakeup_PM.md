# Topic 15 — Suspend/Resume, Wakeup, and IRQ Power Management

> **Standing requirements check (R1/R2/R3):** R1 — wakeup interrupts (`enable_irq_wake`/`irq_set_irq_wake`, the `IRQD_WAKEUP_*` states), the IRQ suspend/resume core (`suspend_device_irqs`/`resume_device_irqs`, the `IRQF_NO_SUSPEND`/`EARLY_RESUME`/`COND_SUSPEND` flags), resend-on-resume, and controller PM. R2 — system-management band, after latency (Topic 14); reuses the flags introduced in Topic 06 and the resend mechanism of Topic 05. R3 — verified against `kernel/irq/pm.c`, `kernel/irq/manage.c` (`irq_set_irq_wake`), `include/linux/irq.h` (`IRQD_WAKEUP_*`) in `v7.1-rc6`. Re-read R1/R2/R3 in the plan.
>
> **Builds on:** Topic 03 (`IRQD_*` state, `wake_depth`, the PM_SLEEP desc fields), Topic 05 (resend), Topic 06 (`IRQF_NO_SUSPEND`/`EARLY_RESUME`/`COND_SUSPEND`, `enable_irq_wake`).
>
> **Position in the plan:** the sleep/wake lifecycle. After this only debugging (16) and the boot capstone (17) remain.
>
> **Source:** `kernel/irq/pm.c`, `kernel/irq/manage.c` (`irq_set_irq_wake`), `include/linux/irq.h` (`IRQD_WAKEUP_STATE`/`IRQD_WAKEUP_ARMED`), `kernel/irq/resend.c`, `drivers/irqchip/irq-gic-pm.c`.

---

## 1. Position: Interrupts Across System Sleep

When the system suspends (S2idle/S3), most interrupts must be **silenced** so a spurious IRQ doesn't abort or corrupt the suspend, but a few must stay **armed** — the timer that keeps time, and the **wakeup sources** that bring the machine back (power button, RTC alarm, NIC wake-on-LAN, a lid switch). This topic is how genirq draws that line.

```
suspend:  freeze tasks → suspend devices → suspend_device_irqs() ──► mask everything
          EXCEPT IRQF_NO_SUSPEND and armed wakeup IRQs
   ... CPU sleeps; an armed wakeup IRQ (or NO_SUSPEND timer) can fire ...
resume:   resume_device_irqs() ──► re-enable; replay any that fired while masked (resend)
```

---

## 2. Wakeup Interrupts

A driver marks its device's IRQ as able to wake the system:

```c
int enable_irq_wake(unsigned int irq);    /* = irq_set_irq_wake(irq, 1) */
int disable_irq_wake(unsigned int irq);
```

Mechanics (`irq_set_irq_wake`, `kernel/irq/manage.c`):
- It is **counted** via `desc->wake_depth` (like `disable_irq`'s depth) — balanced enable/disable.
- It sets **`IRQD_WAKEUP_STATE`** on the `irq_data` (Topic 03) and calls the chip's **`irq_set_wake`** (Topic 03 §4) so the controller programs the line as wakeup-capable (e.g. a GPIO/PMIC routes it to the always-on wake logic). Chips that don't need per-line config set `IRQCHIP_SKIP_SET_WAKE`.
- During suspend, a wakeup IRQ is **armed** (`IRQD_WAKEUP_ARMED`) rather than masked — it stays live to wake the system.

Userspace side: a device's `/sys/devices/.../power/wakeup` (`enabled`/`disabled`) gates whether the driver's wakeup capability is honored; `/sys/power/wakeup_count` and the wakeup-source accounting (`/sys/kernel/debug/wakeup_sources`) track who woke the machine.

---

## 3. The IRQ Suspend/Resume Core (`kernel/irq/pm.c`)

Called from the PM core late in suspend / early in resume (as a syscore-ish step, after devices):

```c
void suspend_device_irqs(void)
{   for each irq_desc:
        sync = suspend_device_irq(desc);   /* mask unless NO_SUSPEND or armed-wakeup */
        if (sync) synchronize_irq(irq);     /* ensure no handler still running */
}
void resume_device_irqs(void) { ... resume_irq(desc) for each ... }   /* unmask + resend */
```

`suspend_device_irq` decides per line:
- **`IRQF_NO_SUSPEND`** actions (Topic 06): the line stays **enabled** through suspend — for the timer (`IRQF_TIMER` bundles `NO_SUSPEND`) and other infrastructure that must keep running during the suspend path itself.
- **Armed wakeup** (`IRQD_WAKEUP_ARMED`, §2): stays enabled, *as a wakeup*.
- **Everything else**: masked, and `synchronize_irq` ensures its handler isn't mid-flight.

The per-desc PM counters from Topic 03 drive the bookkeeping: **`nr_actions`**, **`no_suspend_depth`** (how many actions are `NO_SUSPEND`), **`cond_suspend_depth`** (`IRQF_COND_SUSPEND`), **`force_resume_depth`** (`IRQF_FORCE_RESUME`).

### The flag interplay (the subtle part)
- **`IRQF_NO_SUSPEND`** keeps the *line* armed, but on a **shared** line it does **not** stop the *other* actions' devices from suspending — which is dangerous if one sharer wants NO_SUSPEND and another doesn't. **`IRQF_COND_SUSPEND`** is the resolution: a sharer that can tolerate the line staying enabled (its handler is suspend-safe) sets it, and the core allows the mix; without it, mixing `NO_SUSPEND` and normal actions on one line is rejected.
- **`IRQF_EARLY_RESUME`** resumes the line **early** (in `resume_device_irqs`'s early phase, before normal devices) — for IRQs needed to bring other devices back.
- **`IRQF_FORCE_RESUME`** forces resume even for a `NO_SUSPEND` line (rare, for lines that were left armed but must be re-initialized).

---

## 4. Resend / Lost Interrupts Across Resume (`kernel/irq/resend.c`)

A masked edge IRQ that fired *during* suspend would be lost. On resume, `resume_irq` re-enables the line and, if it was pending, **replays** it via `check_irq_resend` (Topic 05 §6): either the chip's `irq_retrigger`, or the software resend list (`CONFIG_HARDIRQS_SW_RESEND`, `desc->resend_node`). `rearm_wake_irq` re-arms a wakeup line. This is what prevents "the device interrupted while we were asleep and we never noticed" after resume.

---

## 5. Controller PM

The interrupt **controller** itself loses state in deep sleep and must be saved/restored:
- `irq_chip` provides **`irq_suspend`/`irq_resume`/`irq_pm_shutdown`** hooks (Topic 03 §4) and `IRQCHIP_MASK_ON_SUSPEND`.
- A controller driver saves its distributor/redirection/priority state on suspend and restores it on resume — e.g. GIC PM (`drivers/irqchip/irq-gic-pm.c`) saves the distributor and redistributor configuration; the IO-APIC saves its redirection table. Without this, IRQ routing would be garbage after resume.
- `pm.c`'s `irq_pm_*` helpers coordinate this with the per-IRQ suspend.

---

## 6. Observability

- **`/sys/power/wakeup_count`**, **`/sys/kernel/debug/wakeup_sources`** — which source woke the system and how many times.
- **`/sys/devices/.../power/wakeup`** — per-device wakeup enable.
- **`/proc/interrupts`** — compare counts before sleep and after wake to see which line fired the wakeup.
- dmesg around suspend/resume — `PM: suspend`/`resume` and any "irq N: ... while suspended" warnings.
- `/sys/kernel/debug/irq/irqs/N` shows `IRQD_WAKEUP_STATE`/`IRQD_WAKEUP_ARMED`.

---

## 7. Common Pitfalls

- **Forgetting `enable_irq_wake`** — the device can't wake the system; it sleeps through the event. The classic "my USB/RTC/GPIO won't wake the laptop" bug.
- **Unbalanced wake enable/disable** — `wake_depth` counting means a missing `disable_irq_wake` leaves the line armed; an extra one warns.
- **Mixing `IRQF_NO_SUSPEND` and normal handlers on a shared line without `IRQF_COND_SUSPEND`** — rejected, or (if forced) a "suspended" device's handler runs anyway. Use `COND_SUSPEND` deliberately.
- **Assuming `IRQF_NO_SUSPEND` makes a line a wakeup source** — it doesn't; it keeps the line *running during the suspend sequence*, which is different from *waking the system from sleep* (`enable_irq_wake`). Two distinct mechanisms.
- **Controller state not restored** — if a custom irqchip lacks suspend/resume hooks, IRQs misroute after resume; symptom is "device dead only after a suspend cycle".
- **Lost edge across resume** — without resend, an edge during sleep vanishes; ensure `irq_retrigger`/SW resend is available for critical edge lines.

---

## 8. Further Reading & Source Pointers

### Documentation
- `Documentation/power/suspend-and-interrupts.rst` — the authoritative write-up of `NO_SUSPEND`/`COND_SUSPEND`/wakeup and the suspend ordering.
- `Documentation/driver-api/pm/` — device wakeup framework.

### Source to read
- `kernel/irq/pm.c` — `suspend_device_irq` / `resume_irq` (the per-line decision).
- `kernel/irq/manage.c` `irq_set_irq_wake` — the wake-state machine.
- `drivers/irqchip/irq-gic-pm.c` — a real controller save/restore.

### Mental model
> On suspend, genirq masks every line *except* `IRQF_NO_SUSPEND` infrastructure lines and *armed wakeup* sources (`enable_irq_wake` → `IRQD_WAKEUP_ARMED`). On resume it unmasks and **resends** any edge that fired while masked, and the controller driver restores its routing state. "Keep running during suspend" (`NO_SUSPEND`) and "wake the system from sleep" (`enable_irq_wake`) are two different switches — set the right one.

---

**Next: Topic 16 — Debugging & Observability** (a practitioner's reference: every procfs/sysfs/debugfs/tracepoint surface and a symptom→diagnosis playbook).
