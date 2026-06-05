# Topic 03 — The genirq Core Data Model

> **Standing requirements check (R1/R2/R3):** R1 — every structure and the *purpose of each meaningful field* is covered (this is the reference all later topics cite). R2 — placed before any flow/API topic because `irq_desc`/`irq_data`/`irq_chip`/`irqaction` are the nouns every later verb operates on; you cannot read Topic 05 (flow) or Topic 06 (API) without them. R3 — every field below is copied from `include/linux/irqdesc.h`, `include/linux/irq.h`, `include/linux/interrupt.h` in `v7.1-rc6`; layout is config-dependent and that is flagged. Re-read R1/R2/R3 in the plan.
>
> **Builds on:** Topic 00 (hwirq vs virq; contexts), Topic 01 (how control reaches `desc->handle_irq`), Topic 02 (the controllers that `irq_chip` abstracts).
>
> **Position in the plan:** the vocabulary chapter. Topics 04 (domains), 05 (flow), 06 (API), 07 (threads) are all operations on these four structs.
>
> **Source:** `include/linux/irqdesc.h` (`irq_desc`), `include/linux/irq.h` (`irq_data`, `irq_common_data`, `irq_chip`), `include/linux/interrupt.h` (`irqaction`), `kernel/irq/irqdesc.c` (lifecycle), `kernel/irq/settings.h` (flags).

---

## 1. Position & the Big Picture

Four structures, four roles:

```
   virq ──index──►  irq_desc          "the core's view of one interrupt line"
                      │  ├─ handle_irq ─────────────► a FLOW HANDLER (Topic 05)
                      │  ├─ action ────────────────► irqaction ─► irqaction ─► …  (the handlers, Topic 06)
                      │  ├─ irq_common_data ───────► affinity, msi_desc, state, node
                      │  └─ irq_data ──────────────► "the chip's view"
                      │         ├─ hwirq             (controller-local number, Topic 00/04)
                      │         ├─ chip ───────────► irq_chip   (mask/unmask/eoi/…, the controller driver, Topic 02)
                      │         ├─ domain ─────────► irq_domain (Topic 04)
                      │         └─ parent_data ────► irq_data   (next chip up the hierarchy, Topic 04/11)
```

The split that confuses everyone at first: **`irq_desc` is the *core-facing* descriptor** (the flow handler, the action list, statistics, locking), while **`irq_data` is the *chip-facing* descriptor** (hwirq, chip, domain) — passed to every `irq_chip` callback. They are joined: `irq_desc.irq_data` is embedded, and `irq_data.common` points back to `irq_desc.irq_common_data`. Helpers convert: `irq_data_to_desc(d)`, `irq_desc_get_irq_data(desc)`.

> **Why two views?** Hierarchical domains (Topic 04, MSI in Topic 11) stack *multiple* `irq_data` for one `irq_desc` — one per controller level (e.g. PCI-MSI → vector → APIC). Each level's chip sees its own `irq_data` via `parent_data`, but there is still exactly one `irq_desc`. Separating the chip view from the core view is what makes that stacking possible.

---

## 2. `struct irq_desc` — the core descriptor (`include/linux/irqdesc.h`)

```c
struct irq_desc {
    struct irq_common_data  irq_common_data;
    struct irq_data         irq_data;
    struct irqstat __percpu *kstat_irqs;     /* per-CPU handled count → /proc/interrupts */
    irq_flow_handler_t      handle_irq;      /* THE FLOW HANDLER (Topic 05); called as handle_irq(desc) */
    struct irqaction        *action;         /* the handler chain (shared IRQs link via ->next) */
    unsigned int            status_use_accessors;  /* IRQ_* settings; touch only via irq_settings_* */
    unsigned int            core_internal_state__do_not_mess_with_it; /* IRQS_* internal state */
    unsigned int            depth;           /* nested disable_irq() depth; 0 ⇒ enabled */
    unsigned int            wake_depth;      /* nested enable_irq_wake() depth (Topic 15) */
    unsigned int            tot_count;       /* aggregated count for non-percpu fast path */
    unsigned int            irq_count;       /* used by the spurious detector (Topic 05) */
    unsigned long           last_unhandled;  /* aging timer for unhandled count */
    unsigned int            irqs_unhandled;  /* consecutive IRQ_NONE → "nobody cared" */
    atomic_t                threads_handled; /* threaded-IRQ accounting (Topic 07) */
    int                     threads_handled_last;
    raw_spinlock_t          lock;            /* THE per-desc lock; raw because taken in hardirq ctx */
    struct cpumask          *percpu_enabled; /* per-CPU IRQ enable mask (Topic 12) */
#ifdef CONFIG_SMP
    struct irq_redirect     redirect;        /* pending-affinity redirect bookkeeping */
    const struct cpumask    *affinity_hint;  /* driver hint for irqbalance (Topic 13) */
    struct irq_affinity_notify *affinity_notify;
#ifdef CONFIG_GENERIC_PENDING_IRQ
    cpumask_var_t           pending_mask;    /* deferred affinity change target (Topic 13) */
#endif
#endif
    unsigned long           threads_oneshot; /* ONESHOT thread mask (Topic 07) */
    atomic_t                threads_active;
    wait_queue_head_t       wait_for_threads;/* synchronize_irq() waits here */
#ifdef CONFIG_PM_SLEEP
    unsigned int            nr_actions, no_suspend_depth,
                            cond_suspend_depth, force_resume_depth;  /* Topic 15 */
#endif
    /* proc/debugfs/sparse: dir, debugfs_file, dev_name, rcu, kobj */
    struct mutex            request_mutex;   /* serializes request_irq/free_irq for this line */
    int                     parent_irq;
    struct module           *owner;
    const char              *name;           /* shown in /proc/interrupts */
#ifdef CONFIG_HARDIRQS_SW_RESEND
    struct hlist_node       resend_node;     /* on the software-resend list (Topic 05/15) */
#endif
} ____cacheline_internodealigned_in_smp;
```

Fields you will actually touch when debugging: `handle_irq` (which flow handler), `action` (who registered), `depth` (is it disabled?), `irqs_unhandled`/`irq_count` (spurious detection), `lock` (the thing every genirq path takes), `kstat_irqs` (the counts).

> **`irq_flow_handler_t` is single-argument in this tree:** `generic_handle_irq_desc()` calls `desc->handle_irq(desc)`. (Older kernels passed `(irq, desc)`.) Verify: `grep generic_handle_irq_desc include/linux/irqdesc.h`.

---

## 3. `struct irq_data` & `irq_common_data` — the chip view (`include/linux/irq.h`)

```c
struct irq_data {
    u32                     mask;        /* chip-private mask bit cache */
    unsigned int            irq;         /* the virq (same as desc) */
    irq_hw_number_t         hwirq;       /* the CONTROLLER-LOCAL number (Topic 00 §3) */
    struct irq_common_data  *common;     /* → back to irq_desc.irq_common_data */
    struct irq_chip         *chip;       /* the controller driver vtable (this level) */
    struct irq_domain       *domain;     /* the domain that owns this mapping (Topic 04) */
#ifdef CONFIG_IRQ_DOMAIN_HIERARCHY
    struct irq_data         *parent_data;/* the NEXT chip up the stack (Topic 04/11) */
#endif
    void                    *chip_data;  /* controller-driver private per-IRQ data */
};
```

`irq_data` is what every `irq_chip` callback receives. Its `hwirq`+`chip`+`domain` triple is "this interrupt, as known to *this* controller level". Walking `parent_data` climbs the hierarchy (e.g. helpers `irq_chip_mask_parent`, `irq_chip_eoi_parent` forward an op to the parent chip — Topic 04).

```c
struct irq_common_data {
    unsigned int __private  state_use_accessors;  /* IRQD_* state; access via irqd_*() only */
#ifdef CONFIG_NUMA
    unsigned int            node;                 /* preferred NUMA node */
#endif
    void                    *handler_data;        /* per-IRQ data for the flow handler */
    struct msi_desc         *msi_desc;            /* set for MSI IRQs (Topic 11) */
#ifdef CONFIG_SMP
    cpumask_var_t           affinity;             /* requested affinity (Topic 13) */
#endif
#ifdef CONFIG_GENERIC_IRQ_EFFECTIVE_AFF_MASK
    cpumask_var_t           effective_affinity;   /* where it ACTUALLY lands */
#endif
#ifdef CONFIG_GENERIC_IRQ_IPI
    unsigned int            ipi_offset;           /* base for IPI ranges (Topic 12) */
#endif
};
```

`affinity` (what you asked for) vs `effective_affinity` (what the hardware does) is a frequent source of confusion in Topic 13 — note both live here, once per `irq_desc`, regardless of hierarchy depth.

---

## 4. `struct irq_chip` — the controller driver interface (`include/linux/irq.h`)

The vtable a controller driver (Topic 02) fills. The core never pokes APIC/GIC registers directly; it calls these:

```c
struct irq_chip {
    const char *name;                              /* shown in /proc/interrupts */
    unsigned int (*irq_startup)(struct irq_data *);  void (*irq_shutdown)(struct irq_data *);
    void (*irq_enable)(struct irq_data *);           void (*irq_disable)(struct irq_data *);
    void (*irq_ack)(struct irq_data *);              /* acknowledge receipt */
    void (*irq_mask)(struct irq_data *);             void (*irq_mask_ack)(struct irq_data *);
    void (*irq_unmask)(struct irq_data *);
    void (*irq_eoi)(struct irq_data *);              /* end-of-interrupt (level/fasteoi) */
    int  (*irq_set_affinity)(struct irq_data *, const struct cpumask *, bool force);  /* Topic 13 */
    int  (*irq_retrigger)(struct irq_data *);        /* re-inject a lost edge (Topic 05) */
    int  (*irq_set_type)(struct irq_data *, unsigned int flow_type);  /* level/edge/polarity */
    int  (*irq_set_wake)(struct irq_data *, unsigned int on);          /* wakeup source (Topic 15) */
    void (*irq_bus_lock)(struct irq_data *);  void (*irq_bus_sync_unlock)(struct irq_data *); /* slow-bus chips (I2C/SPI) */
    void (*irq_suspend)/(*irq_resume)/(*irq_pm_shutdown)(struct irq_data *);            /* Topic 15 */
    int  (*irq_request_resources)(struct irq_data *);  void (*irq_release_resources)(struct irq_data *);
    void (*irq_compose_msi_msg)(struct irq_data *, struct msi_msg *);  /* Topic 11 */
    void (*irq_write_msi_msg)(struct irq_data *, struct msi_msg *);
    int  (*irq_get_irqchip_state)/(*irq_set_irqchip_state)(struct irq_data *, enum irqchip_irq_state, …);
    int  (*irq_set_vcpu_affinity)(struct irq_data *, void *vcpu_info);                 /* KVM passthrough */
    void (*ipi_send_single)(struct irq_data *, unsigned int cpu);                      /* Topic 12 */
    void (*ipi_send_mask)(struct irq_data *, const struct cpumask *dest);
    int  (*irq_nmi_setup)(struct irq_data *);  void (*irq_nmi_teardown)(struct irq_data *);  /* Topic 12 */
    void (*irq_force_complete_move)(struct irq_data *);                                /* Topic 13 hotplug */
    unsigned long flags;                            /* IRQCHIP_* (below) */
};
```

Not every chip implements every op — the core checks for NULL and has sane defaults. The **minimum** a simple chip provides is `irq_mask`/`irq_unmask` (and usually `irq_ack` or `irq_eoi` depending on trigger). The `irq_bus_lock`/`irq_bus_sync_unlock` pair exists for controllers reached over a sleeping bus (an I2C GPIO expander): the flow handler can't poke I2C in hardirq context, so masking is *batched* and flushed in `bus_sync_unlock` from a sleepable context — a subtle but important design point.

### `IRQCHIP_*` flags (the `flags` field)
Behavioural hints the core honours, e.g.: `IRQCHIP_SET_TYPE_MASKED` (mask while changing type), `IRQCHIP_EOI_IF_HANDLED`, `IRQCHIP_EOI_THREADED` (defer EOI until the threaded handler finishes, Topic 07), `IRQCHIP_MASK_ON_SUSPEND`, `IRQCHIP_SKIP_SET_WAKE`, `IRQCHIP_ONESHOT_SAFE`, `IRQCHIP_AFFINITY_PRE_STARTUP`. Read them in `include/linux/irq.h` — several change flow-handler behaviour in Topic 05.

### Generic chip
`kernel/irq/generic-chip.c` (`struct irq_chip_generic`, `irq_alloc_generic_chip`) provides a ready-made chip for the common "one MMIO register with a mask bit per IRQ" controller, so SoC drivers don't reimplement mask/unmask. Many `drivers/irqchip/` drivers use it.

---

## 5. `struct irqaction` — a registered handler (`include/linux/interrupt.h`)

```c
struct irqaction {
    irq_handler_t       handler;        /* the TOP HALF (Topic 06); returns irqreturn_t */
    union { void *dev_id; void __percpu *percpu_dev_id; };  /* cookie; also the free_irq key */
    const struct cpumask *affinity;
    struct irqaction    *next;          /* SHARED-IRQ chain: multiple actions on one line */
    irq_handler_t       thread_fn;      /* the THREADED handler (Topic 07), or NULL */
    struct task_struct  *thread;        /* the per-action kthread */
    struct irqaction    *secondary;     /* internal: the "other" handler for forced-threading */
    unsigned int        irq;            /* the virq */
    unsigned int        flags;          /* IRQF_* (Topic 06) */
    unsigned long       thread_flags;   /* IRQTF_* runtime thread state (Topic 07) */
    unsigned long       thread_mask;    /* bit in desc->threads_oneshot for this action */
    const char          *name;          /* the device name in /proc/interrupts */
    struct proc_dir_entry *dir;
};
```

One `irqaction` per `request_irq()`. A **shared** line (`IRQF_SHARED`) chains several via `->next`; the flow handler walks the chain and each handler returns `IRQ_NONE` ("not mine") or `IRQ_HANDLED`. `dev_id` must be unique per action on a shared line — it is both the per-device cookie and the key `free_irq(irq, dev_id)` uses to find which action to remove. (Note: `request_irq()` in this tree folds in `IRQF_COND_ONESHOT` — see the inline definition — relevant to Topic 06/07.)

---

## 6. Status & Settings Flags (three families — keep them straight)

genirq state lives in **three** different words with **three** access disciplines:

| Where | Flags | Meaning | Access via |
|---|---|---|---|
| `irq_common_data.state_use_accessors` | **`IRQD_*`** | live `irq_data` state: `IRQD_IRQ_MASKED`, `IRQD_IRQ_DISABLED`, `IRQD_PER_CPU`, `IRQD_LEVEL`, `IRQD_AFFINITY_SET`, `IRQD_WAKEUP_STATE`, `IRQD_AFFINITY_MANAGED`, `IRQD_MOVE_PCNTXT`, … | `irqd_*()` helpers |
| `irq_desc.status_use_accessors` | **`IRQ_*`** | configurable settings: `IRQ_PER_CPU`, `IRQ_NOBALANCING`, `IRQ_NESTED_THREAD`, `IRQ_NO_BALANCING`, `IRQ_LEVEL`, trigger type | `irq_settings_*()` (`kernel/irq/settings.h`) |
| `irq_desc.core_internal_state__do_not_mess_with_it` | **`IRQS_*`** | core runtime state: `IRQS_PENDING`, `IRQS_REPLAY`, `IRQS_ONESHOT`, `IRQS_WAITING`, `IRQS_SUSPENDED` | core only |

The naming is unfortunate but the rule is simple: **never write these by hand** — use the accessor for the right family. The names tell you which: `IRQD_` → on `irq_data`, `IRQ_` → desc settings, `IRQS_` → core internal. You'll meet `IRQS_PENDING` in Topic 05 (edge re-trigger), `IRQD_AFFINITY_MANAGED` in Topic 13, `IRQD_WAKEUP_STATE` in Topic 15.

---

## 7. Sparse IRQ Storage & Lifecycle (`kernel/irq/irqdesc.c`)

- **`CONFIG_SPARSE_IRQ=y`** (the norm): `irq_desc`s live in a **maple tree** (`sparse_irqs`), looked up by `irq_to_desc(virq)`. virqs are allocated on demand by `irq_alloc_descs()` / `__irq_alloc_descs()` — there is *no* fixed `NR_IRQS`-sized array. This is essential for MSI, where thousands of vectors appear and vanish dynamically.
- **`CONFIG_SPARSE_IRQ=n`** (small/embedded): a static `struct irq_desc irq_desc[NR_IRQS]` array.
- `early_irq_init()` (Topic 17) creates the initial descriptors at boot; `alloc_desc()`/`free_desc()` manage them; RCU (`desc->rcu`) frees them so lockless lookups in flow paths are safe.
- Each `irq_desc` gets a `kobject` (`/sys/kernel/irq/N`) and a proc dir (`/proc/irq/N`).

### Statistics
`kstat_irqs` is per-CPU per-IRQ; `kstat_irqs_cpu()` / `irq_desc_kstat_cpu()` read it; this is exactly what `/proc/interrupts` prints. `tot_count` aggregates for the non-percpu fast path.

---

## 8. Observability, Pitfalls, Further Reading

### Observability
- `/proc/interrupts` = `virq | per-CPU kstat_irqs | chip name + hwirq + flow type + action name`. Every column maps to a field above.
- `/sys/kernel/irq/N/{chip_name,hwirq,name,type,actions,per_cpu_count}` — the `irq_desc` exposed.
- `/sys/kernel/debug/irq/irqs/N` (`CONFIG_GENERIC_IRQ_DEBUGFS`) — dumps `IRQD_*`/`IRQS_*` state, the domain, hwirq, and the hierarchy. The single best place to inspect this model live (Topic 16).
- `drgn`/`crash`: `irq_to_desc(N)` then walk `->action`, `->irq_data.chip`, `->irq_data.hwirq`.

### Pitfalls
- **Editing the state words directly** — always use `irqd_*` / `irq_settings_*`; the raw fields are deliberately ugly-named (`core_internal_state__do_not_mess_with_it`) to stop you.
- **Confusing `irq_desc` and `irq_data`** — if a function takes `struct irq_data *` it is a chip callback (per hierarchy level); if it takes `struct irq_desc *` or a virq it is core-facing.
- **Assuming one `irq_data` per IRQ** — under hierarchy there are several; walk `parent_data` (Topic 04/11).
- **`affinity` vs `effective_affinity`** — reading the wrong one when debugging "why did this land on CPU X" (Topic 13).
- **Holding `desc->lock` and then sleeping** — it is a `raw_spinlock_t` taken in hardirq context; nothing under it may sleep.

### Further reading
- `Documentation/core-api/genericirq.rst` (the conceptual companion to this struct tour).
- Source: `include/linux/irq.h` (read the `irqd_*` accessor block right after the state flags), `kernel/irq/irqdesc.c` (`alloc_desc`, `irq_to_desc`).

### Mental model
> `irq_desc` is the *line*; `irq_data` is the *line as one controller sees it* (stacked under hierarchy); `irq_chip` is the *controller's hands* (mask/unmask/eoi); `irqaction` is *a driver's claim* on the line. The flow handler (Topic 05) is the function that, given a `desc`, drives the `chip` through the right mask/ack/eoi dance and then runs the `action` chain.

---

**Next: Topic 04 — IRQ Domains** (how a controller's hwirq becomes the virq that indexes the `irq_desc` you just learned).
