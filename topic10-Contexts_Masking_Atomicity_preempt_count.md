# Topic 10 — Contexts, Masking, Atomicity & `preempt_count`

> **Standing requirements check (R1/R2/R3):** R1 — the complete concurrency model: the `preempt_count` bit layout, the `in_*()` predicates, IRQ masking primitives, the lock-vs-IRQ rules (`spin_lock_irqsave`/`_bh`), lockdep IRQ-state tracking, RCU-in-IRQ, and a master "allowed where" table. R2 — placed *after* the bottom halves (Topics 07–09) exactly as the plan's principle #4 requires: `preempt_count`'s softirq/hardirq fields and `local_bh_disable` only make sense once those mechanisms exist; this topic then governs all of them. R3 — bit layout quoted from `include/linux/preempt.h` in `v7.1-rc6`; masking from `include/linux/irqflags.h`. Re-read R1/R2/R3 in the plan.
>
> **Builds on:** Topic 00 (the four contexts, intro to `preempt_count`), Topic 01 (generic entry maintains lockdep/RCU IRQ state), Topic 05/06 (handlers run IRQs-off), Topic 08 (`SOFTIRQ_OFFSET`, `local_bh_*`).
>
> **Position in the plan:** the rulebook. Everything in Topics 05–09 is *correct* only under these rules; everything in Topics 11–17 assumes them.
>
> **Source:** `include/linux/preempt.h` (counts, `in_*`), `include/linux/irqflags.h` (`local_irq_*`, lockdep hooks), `include/linux/spinlock.h`, `include/linux/rcupdate.h`.

---

## 1. One Word Encodes "Where Am I": `preempt_count`

Every CPU's current task carries a `preempt_count` whose bitfields say which contexts are active and how deeply nested (`include/linux/preempt.h`):

```c
#define PREEMPT_BITS  8     #define PREEMPT_SHIFT  0
#define SOFTIRQ_BITS  8     #define SOFTIRQ_SHIFT  (PREEMPT_SHIFT + PREEMPT_BITS)   /* 8  */
#define HARDIRQ_BITS  4     #define HARDIRQ_SHIFT  (SOFTIRQ_SHIFT + SOFTIRQ_BITS)   /* 16 */
#define NMI_BITS      4     #define NMI_SHIFT      (HARDIRQ_SHIFT + HARDIRQ_BITS)   /* 20 */
```

```
 bit:  31            24      20      16          8           0
       │ NEED_RESCHED │ NMI(4)│HARDIRQ(4)│ SOFTIRQ(8) │ PREEMPT(8) │
```

- **PREEMPT (8 bits)** — preemption-disable nesting (`preempt_disable()` depth). Non-zero ⇒ this task can't be preempted.
- **SOFTIRQ (8 bits)** — softirq depth. `+SOFTIRQ_OFFSET` while *serving* a softirq; `+SOFTIRQ_DISABLE_OFFSET` (2×) while `local_bh_disable()`. That double-use is why `in_serving_softirq()` ≠ `in_softirq()` (Topic 08 §4).
- **HARDIRQ (4 bits)** — hardirq nesting (`irq_enter()` adds `HARDIRQ_OFFSET`).
- **NMI (4 bits)** — NMI nesting.
- Top bit: `PREEMPT_NEED_RESCHED` (folded in on some arches) — "a reschedule is pending".

The predicates are pure bit-tests (Topic 00 §4):
```c
in_nmi()             = nmi_count()
in_hardirq()         = hardirq_count()
in_serving_softirq() = softirq_count() & SOFTIRQ_OFFSET
in_softirq()         = softirq_count()                  /* serving OR bh-disabled */
in_interrupt()       = irq_count()                      /* hardirq | softirq | nmi */
in_task()            = !(preempt_count() & (NMI_MASK|HARDIRQ_MASK|SOFTIRQ_OFFSET))
in_atomic()          = preempt_count() != 0             /* any of the above, or preempt disabled */
```

This single word is how `might_sleep()`, `GFP_KERNEL` (vs the implicit "can I reclaim?" decision), lockdep, and the scheduler all answer "is sleeping/preemption legal right now?". Entering/leaving each context just adds/subtracts the matching `*_OFFSET` (done by the generic entry layer, Topic 01).

---

## 2. IRQ Masking Primitives (`include/linux/irqflags.h`)

Disabling interrupts on the **local CPU** is different from disabling preemption and different from "being in an interrupt":

```c
local_irq_disable();  local_irq_enable();          /* unconditional */
local_irq_save(flags); local_irq_restore(flags);   /* save+disable / restore prior state — nestable */
bool irqs_disabled();                              /* are local IRQs currently off? */
```

- These mask **all** maskable interrupts on the current CPU at the CPU level (x86 `IF`/`cli`/`sti`; arm64 `DAIF`). They do **not** mask a *specific* IRQ line — that's the chip's `irq_mask` (Topic 03/05) / `disable_irq` (Topic 06).
- **Use `save`/`restore`, not bare `disable`/`enable`, inside code that may itself be called with IRQs already off** — bare `enable` would wrongly re-enable.
- Disabling IRQs is the heaviest hammer: it raises IRQ-off latency for the *whole CPU* (Topic 14). Keep regions tiny.

Three independent states to keep straight (a frequent confusion):
| State | Set by | Means | Predicate |
|---|---|---|---|
| IRQs disabled | `local_irq_disable` | no interrupts on this CPU | `irqs_disabled()` |
| preemption disabled | `preempt_disable` | scheduler won't switch tasks here | `preempt_count() & PREEMPT_MASK` |
| in interrupt | hardware/`irq_enter` | currently servicing an IRQ/softirq/NMI | `in_interrupt()` |

You can be in any combination (e.g. a softirq runs with IRQs *on*, preemption effectively off, `in_serving_softirq()` true).

---

## 3. Locks and Interrupts (the deadlock you must prevent)

The classic interrupt deadlock: task holds lock L (IRQs on); an interrupt fires on the *same CPU*; its handler tries to take L → spins forever (the holder can't progress because the CPU is stuck in the handler). Prevention = **disable on the CPU whatever context also takes the lock, while holding it**:

```c
spin_lock_irqsave(&L, flags);   /* lock + local_irq_save: use when a HARDIRQ handler also takes L */
   ...
spin_unlock_irqrestore(&L, flags);

spin_lock_bh(&L);               /* lock + local_bh_disable: use when a SOFTIRQ/tasklet also takes L */
   ...
spin_unlock_bh(&L);

spin_lock(&L);                  /* plain: only when L is never taken in IRQ/softirq context */
```

Rules:
- **Lock shared with a hardirq handler → `spin_lock_irqsave`** (covers softirq too, since IRQs-off implies BH can't run).
- **Lock shared with a softirq/tasklet only → `spin_lock_bh`** (cheaper; doesn't disable hardirqs).
- **Lock shared only among process-context paths → `spin_lock`** (and it may even be a sleeping `mutex`).
- Inside a **top half** (already IRQs-off) use plain `spin_lock` — don't double-disable.
- Inside a **threaded handler** (process context, Topic 07) you may take **mutexes** — that's the whole point of threading.

`desc->lock` (Topic 03) is a `raw_spinlock_t` precisely because it's taken in hardirq context and must not become a sleeping lock even on PREEMPT_RT.

---

## 4. Lockdep & IRQ-State Tracking

`lockdep` (`CONFIG_PROVE_LOCKING`) models each lock's **IRQ-safety**:
- A lock ever taken in hardirq context is **hardirq-safe**; a lock ever taken with hardirqs enabled (so an IRQ could interrupt and re-take it) is **hardirq-unsafe**. Taking a hardirq-unsafe lock while holding a hardirq-safe one (or vice-versa in a way that could deadlock) triggers the `inconsistent {IN-HARDIRQ-W} -> {HARDIRQ-ON-W}` splat.
- The generic entry layer (Topic 01) feeds lockdep the on/off transitions (`trace_hardirqs_on/off`, `lockdep_hardirqs_*`) so it knows the current IRQ state at every lock op. This is why a custom entry path that skips the generic layer corrupts lockdep.
- Softirq equivalents track BH-safe/unsafe similarly.

Practical payoff: lockdep catches "you used `spin_lock` where you needed `spin_lock_irqsave`" *before* it deadlocks in production. Run debug kernels with it.

---

## 5. RCU and Interrupts

- **RCU read-side is legal in hardirq and softirq context** (`rcu_read_lock()` there is fine; it's non-blocking). Reclaim/callbacks are deferred.
- The CPU must be "RCU-watching" inside an interrupt; the generic entry layer (`irqentry_enter` → `ct_irq_enter`/`rcu_irq_enter` semantics, Topic 01) guarantees it — hence the `RCU_LOCKDEP_WARN(!rcu_is_watching())` in `common_interrupt` (Topic 01 §3.2).
- **NMI needs special RCU** (`rcu_nmi_enter`/`exit`, via `irqentry_nmi_enter`) because an NMI can interrupt RCU's own bookkeeping; NMI handlers may use only NMI-safe RCU operations.
- `synchronize_rcu()` **sleeps** → never from any atomic/IRQ context; use `call_rcu()` to defer instead.

---

## 6. The Master "Allowed Where" Table (R1)

| You are in… | Sleep / mutex? | Blocking alloc (`GFP_KERNEL`)? | Which spinlock variant | RCU read? | `synchronize_rcu`? | Typical code |
|---|---|---|---|---|---|---|
| **process / task** | **yes** | **yes** | any (`spin_lock`, or mutex) | yes | yes | syscalls, kthreads, **threaded IRQ** (Topic 07) |
| **softirq / BH** | no | no (`GFP_ATOMIC`) | `spin_lock` (vs hardirq: `_irqsave`) | yes | no | net/block softirqs, tasklets, BH-wq |
| **hardirq (top half)** | no | no (`GFP_ATOMIC`) | `spin_lock` (IRQs already off) | yes | no | `request_irq` handlers, flow handlers |
| **NMI** | no | no | only `raw_spin_trylock`/lockless | NMI-safe RCU only | no | watchdog, perf; escalate via `irq_work` |

Reading rule: **the deeper/more-atomic the context, the fewer primitives are legal.** "Can I sleep here?" = "is `in_task()` and preemption not disabled?". When unsure, `might_sleep()` in a debug kernel tells you.

---

## 7. PREEMPT_RT Deltas (so the rules don't surprise you)

RT changes *which* context things run in (Topics 07/08/14), not the bit layout:
- Most IRQ handlers and softirqs run in **task context** → they become preemptible and may take RT mutexes ("sleeping spinlocks").
- `spin_lock` becomes a sleeping lock; `raw_spinlock_t` stays a true spinlock (that's why `desc->lock` is raw).
- `local_bh_disable` becomes a per-CPU sleeping lock.
- `local_irq_disable` still disables IRQs, but far less code runs with it held.

So on RT, "I'm in a softirq, I can't sleep" becomes "I'm in a thread, I can". Code that must be truly atomic uses `raw_*` and `IRQ_WORK_HARD_IRQ` (Topic 09).

---

## 8. Observability, Pitfalls, Further Reading

### Observability
- **lockdep splats** (`INCONSISTENT`, `possible irq lock inversion`) — the primary tool; read the two stack traces (the safe and unsafe acquisition).
- `CONFIG_DEBUG_ATOMIC_SLEEP` → "BUG: sleeping function called from invalid context" with the `in_atomic`/`irqs_disabled` state printed.
- The `irqsoff`/`preemptirqsoff` tracers (Topic 14/16) measure how long IRQs/preemption stayed off and where.
- `preempt_count()` is visible in oops dumps and via tracing.

### Pitfalls
- **`spin_lock` where `spin_lock_irqsave` was needed** — deadlocks when the IRQ handler re-takes the lock on the same CPU. The #1 interrupt-locking bug; lockdep catches it.
- **`local_irq_disable` instead of `local_irq_save` in nestable code** — re-enables IRQs that a caller had disabled.
- **Sleeping in atomic context** — `mutex_lock`, `kmalloc(GFP_KERNEL)`, `copy_from_user`, `msleep`, `synchronize_rcu` in a hardirq/softirq → `DEBUG_ATOMIC_SLEEP` BUG.
- **Assuming "IRQs off" ⇒ "preemption off" ⇒ "in interrupt"** — three different states (§2). E.g. you can hold `spin_lock_irqsave` in process context (IRQs off, *not* in interrupt).
- **Long IRQs-off / BH-disabled regions** — latency killers (Topic 14). Minimize.
- **Custom entry paths** that skip the generic layer — corrupt lockdep/RCU IRQ-state and cause spurious or missed warnings.

### Further reading
- `Documentation/locking/*` (esp. `lockdep-design.rst`), `Documentation/core-api/local_ops.rst`, `Documentation/RCU/checklist.rst`.
- LWN: "Atomic context and kernel API design", the PREEMPT_RT locking series.
- Source: `include/linux/preempt.h`, `include/linux/irqflags.h`, `kernel/locking/lockdep.c` (IRQ-state machine).

### Mental model
> `preempt_count` is one word saying "how deep am I in preempt/softirq/hardirq/NMI". The deeper you are, the less you may do: hardirq/softirq/NMI can't sleep; process context can. Locks shared with an IRQ context need `spin_lock_irqsave` (hardirq) or `_bh` (softirq) so a same-CPU handler can't deadlock the holder. lockdep models this and catches the mistakes; PREEMPT_RT moves most handlers to threads so the "can't sleep" rules relax.

---

**Next: Topic 11 — MSI and MSI-X** (modern message-signaled interrupts, built on the hierarchical domains of Topic 04, delivered under all the rules you just learned).
