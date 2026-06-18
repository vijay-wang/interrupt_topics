# Topic 10 扩展版 — 上下文、屏蔽、原子性与 preempt_count：并发规则手册

> **用法同前**。行号基于 `~/repo/linux`（v7.1-rc7）：`include/linux/preempt.h` 里
> 位布局 `PREEMPT_BITS:33` 起、`in_nmi()/in_hardirq()/...:126` 起；屏蔽原语在
> `include/linux/irqflags.h`，锁在 `include/linux/spinlock.h`。
>
> **承接**：Topic 05~09 介绍了一堆机制（硬中断、软中断、tasklet、线程、工作队列）。
> 本章是**让它们互不打架的法则手册**——什么上下文能做什么、锁怎么选、为什么。
> 这是全系列的"宪法"，Topic 11~17 全部默认你已掌握它。

---

## 1. 先建立画面感：一个词回答"我现在在哪儿"

### 1.1 问题：代码怎么知道"此刻能不能睡觉"？

内核里同一个函数可能在好几种处境下被调用：可能在普通系统调用里（悠闲，能睡觉），
可能在硬中断里（紧急，绝不能睡），可能在软中断里（半紧急，也不能睡）。
**这个函数怎么知道自己此刻处于哪种处境，从而决定能不能 `kmalloc(GFP_KERNEL)`、
能不能拿 mutex？**

答案：每个 CPU 当前任务身上挂着一个**计数器 `preempt_count`**，它像一块**叠放的
"身份牌"**——记录"我现在嵌套在几层什么样的上下文里"。

### 1.2 一个词，四个计数器叠在一起

`include/linux/preempt.h:33` 把一个 32 位整数切成几段：

```
 bit:  31            24      20      16          8           0
       │ NEED_RESCHED │ NMI(4)│HARDIRQ(4)│ SOFTIRQ(8) │ PREEMPT(8) │
```

**画面**：一摞托盘，每进入一种上下文就往对应格子里**加一个筹码**，离开就**减一个**
（由 Topic 01 的通用入口层自动加减）。读这个数就知道现在叠了哪些上下文、叠了多深：

- **PREEMPT（8 位）**：`preempt_disable()` 的嵌套深度。非零 = 此刻不可被抢占。
- **SOFTIRQ（8 位）**：软中断深度。正在跑软中断 +1 份（`SOFTIRQ_OFFSET`）；
  `local_bh_disable()` +2 份（`SOFTIRQ_DISABLE_OFFSET`）——这个**双重用途**正是
  Topic 08 晦涩点 ⑦ 讲的 `in_serving_softirq()` ≠ `in_softirq()` 的根源。
- **HARDIRQ（4 位）**：硬中断嵌套深度（`irq_enter` 加 `HARDIRQ_OFFSET`）。
- **NMI（4 位）**：NMI 嵌套深度。

### 晦涩点 ①：那些 `in_xxx()` 判断其实就是查位

`preempt.h:126` 起，全是纯粹的位测试（Topic 00 §4 见过，这里看实现）：

```c
in_nmi()             = nmi_count()        /* NMI 那 4 位非零吗 */
in_hardirq()         = hardirq_count()    /* HARDIRQ 那 4 位非零吗 */
in_serving_softirq() = softirq_count() & SOFTIRQ_OFFSET  /* 正在跑软中断（只看那 1 份）*/
in_softirq()         = softirq_count()    /* 正在跑 或 被 bh_disable（看全部）*/
in_interrupt()       = irq_count()        /* 硬中断 | 软中断 | NMI 任一 */
in_task()            = 上面全为 0          /* 在普通任务上下文 */
in_atomic()          = preempt_count() != 0  /* 任何一种，或抢占被禁 */
```

**这一个词就是 `might_sleep()`、`GFP_KERNEL` 的隐式判断、lockdep、调度器共同查询的
"此刻睡眠/抢占是否合法"的唯一真相源。** 进出每种上下文 = 加减对应的 `*_OFFSET`，
仅此而已。

---

## 2. 逐句拆解三个**互相独立**的状态（最大的混淆点）

原文 §2 反复强调："三个不同的状态，别混"。这是新手（甚至老手）最常搞错的地方。

### 晦涩点 ②：关中断 ≠ 关抢占 ≠ 在中断里

| 状态 | 谁设的 | 含义 | 怎么查 |
|---|---|---|---|
| **IRQ 关闭** | `local_irq_disable` | 本 CPU 上**硬件不再投递中断** | `irqs_disabled()` |
| **抢占关闭** | `preempt_disable` | 调度器**不会在这里切换任务** | `preempt_count() & PREEMPT_MASK` |
| **在中断中** | 硬件/`irq_enter` | **正在处理**某个中断/软中断/NMI | `in_interrupt()` |

**这三者可以任意组合**。最反直觉的例子（原文给的）：软中断运行时——**中断是开着的**
（Topic 08 晦涩点 ④）、抢占实际是关的、`in_serving_softirq()` 为真。三个状态各不相同。

**画面**：① 关中断 = 把家门口的门铃断电（别人没法打扰你）；② 关抢占 = 告诉调度器
"现在别换我下场"（但门铃还能响）；③ 在中断里 = 你正在应门（你自己就是被门铃叫来的）。
三件事彼此无关，可以同时成立或单独成立。

### 晦涩点 ③：`local_irq_save` vs `local_irq_disable` —— 为什么常用前者

```c
local_irq_disable(); ... local_irq_enable();        /* 无条件：直接关、直接开 */
local_irq_save(flags); ... local_irq_restore(flags); /* 保存原状态后关、恢复到原状态 */
```

区别在**嵌套安全**。假设函数 F 用了 `disable...enable`，而调用 F 的人**本来就关着中断**。
F 末尾的 `enable` 会**把中断错误地打开**——破坏了调用者的假设，可能引发竞争。
`save/restore` 则记住进来时的状态，出去时**恢复成原样**（原本关着就还关着）。

**规则**：可能被"已关中断"的代码调用的函数，**一律用 save/restore**。这也是为什么内核
里 `_irqsave` 满天飞。

注意原文强调：`local_irq_disable` 屏蔽的是**本 CPU 所有可屏蔽中断**（x86 的 `cli`、
arm64 的 DAIF），**不是某一条具体中断线**——关单条线是芯片的 `irq_mask`（Topic 03/05）
或 `disable_irq`（Topic 06）。两个层次别混。而且它是**最重的锤子**：拉高整个 CPU 的
中断关闭延迟（Topic 14），临界区要尽量小。

---

## 3. 逐句拆解锁与中断（你必须防的那个死锁）

### 晦涩点 ④：经典的中断死锁，用画面讲透

原文 §3 描述的死锁，是**所有中断编程里最重要的一个**，慢慢看：

**场景**：任务 T 在 CPU0 上拿了锁 L（此时中断开着）。正干着活，**同一个 CPU0 上来了
个中断**。中断 handler 也想拿锁 L —— 但 L 被 T 拿着呢，handler 只能**自旋等 T 放锁**。
可问题是：**T 没法继续运行**（CPU0 正卡在 handler 里自旋），T 永远放不了锁，
handler 永远等不到 —— **死锁**。

**画面**：你（T）在用会议室（锁 L），把钥匙揣兜里。这时你的上司（中断）冲进办公室说
"我现在就要用那间会议室"，并且**站在你面前不走、不让你回工位**。你回不了工位就没法
去还钥匙，上司就一直站着等……两人卡死。

**解法**：拿锁 L 期间，**把"也会拿 L 的那种中断"在本 CPU 上关掉**——这样持锁期间那个
中断根本不会来，也就不会自旋等你。

```c
spin_lock_irqsave(&L, flags);    /* 锁 + 关硬中断：当 L 也被硬中断 handler 拿时用 */
spin_lock_bh(&L);                /* 锁 + 关软中断：当 L 也被软中断/tasklet 拿时用 */
spin_lock(&L);                   /* 裸锁：L 从不在中断上下文被拿时用 */
```

### 晦涩点 ⑤：到底该用哪个变体？决策规则

原文 §3 的规则，配"为什么"：

- **和硬中断 handler 共享的锁 → `spin_lock_irqsave`**。关硬中断**顺带**也挡住了软中断
  （关中断时软中断跑不了），所以 `_irqsave` 是"最强保护"，覆盖软中断情况。
- **只和软中断/tasklet 共享 → `spin_lock_bh`**。更便宜，只关软中断不关硬中断。
- **只在进程上下文之间共享 → `spin_lock`**（甚至可以用能睡的 `mutex`）。
- **已经在 top half 里（中断本来就关着）→ 用裸 `spin_lock`**，别重复关（多此一举）。
- **在线程化 handler 里（进程上下文，Topic 07）→ 可以用 mutex**——这是线程化的全部
  意义所在。

**口诀**：**"谁会在中断里碰这个数据，就在持锁时关掉谁。"** 硬中断会碰 → irqsave；
只有软中断会碰 → bh；都不会 → 裸锁。

这也解释了 Topic 03 晦涩点 ④：`desc->lock` 是 `raw_spinlock_t`，因为它在硬中断
上下文被拿，**即使在 PREEMPT_RT 上也绝不能变成可睡眠的锁**。

---

## 4. 逐句拆解 lockdep：死锁的自动侦探

### 晦涩点 ⑥：lockdep 怎么在死锁**发生前**抓到它？

手动检查"每个锁该不该 irqsave"在大型内核里不可能。`lockdep`（`CONFIG_PROVE_LOCKING`）
**自动建模每把锁的中断安全性**：

- 一把锁**曾在硬中断里被拿过** → 标记为 **hardirq-safe**（中断里用过它）。
- 一把锁**曾在开着硬中断时被拿过**（意味着拿着它时可能被中断打断、中断又来拿它）
  → 标记为 **hardirq-unsafe**。
- 如果 lockdep 发现一种**可能导致 §3 那个死锁**的加锁顺序，立刻打印
  `inconsistent {IN-HARDIRQ-W} -> {HARDIRQ-ON-W}` 告警——**哪怕这次没真死锁**。

**画面**：lockdep 是个全程跟拍的侦探，把每把锁"曾在什么中断状态下被谁拿过"记成档案，
一旦发现两份档案拼起来构成死锁配方，当场吹哨——不用等真死锁发生（死锁在生产环境
往往几个月才偶发一次，极难复现）。

实用结论（原文）：**用开了 lockdep 的 debug 内核跑测试**，它会在你"该用 irqsave 却
用了裸 spin_lock"时当场抓住你，而不是等上线后随机死机。这跟 Topic 01 联动：
通用入口层负责把"中断开/关"的状态变化喂给 lockdep（`trace_hardirqs_on/off`），
所以**绕过通用入口层的自定义入口会让 lockdep 失灵**。

---

## 5. 逐句拆解 RCU 与中断

### 晦涩点 ⑦：RCU 读侧在中断里合法，但 `synchronize_rcu` 不行

回忆 Topic 01 晦涩句 ⑰ 的 RCU"值班表"。几条规则：

- **RCU 读侧（`rcu_read_lock`）在硬中断、软中断里合法**——它不阻塞，只是标记"我在读"。
- 中断里 CPU 必须"RCU-watching"，通用入口层（Topic 01 的 `irqentry_enter`）保证了这点
  ——这就是 `common_interrupt` 里那句 `RCU_LOCKDEP_WARN(!rcu_is_watching())` 岗哨的来历。
- **NMI 需要特殊 RCU**（`rcu_nmi_enter/exit`），因为 NMI 可能打断 RCU 自己的记账过程
  ——NMI handler 只能用 NMI 安全的 RCU 操作。
- **`synchronize_rcu()` 会睡眠**（它要等一个宽限期）→ **绝不能在任何原子/中断上下文调用**。
  要在中断里延迟释放，用**不阻塞的 `call_rcu()`** 代替。

**口诀**：中断里**读** RCU 没问题，**等** RCU（synchronize）绝对不行——等 = 睡，
睡在中断里就是 BUG。

---

## 6. 逐句拆解"主表"：什么上下文能做什么（全章精华）

原文 §6 这张表建议**背下来或贴墙上**。重排成"从自由到受限"，更易记：

| 你在… | 能睡/mutex？ | `GFP_KERNEL`？ | 用哪种 spinlock | RCU 读？ | `synchronize_rcu`？ |
|---|---|---|---|---|---|
| **进程/任务**（含线程化 IRQ）| ✅ | ✅ | 任意（或 mutex）| ✅ | ✅ |
| **软中断/BH**（tasklet、BH-wq）| ❌ | ❌（用 GFP_ATOMIC）| `spin_lock`（和硬中断共享则 `_irqsave`）| ✅ | ❌ |
| **硬中断（top half）** | ❌ | ❌（GFP_ATOMIC）| `spin_lock`（中断已关）| ✅ | ❌ |
| **NMI** | ❌ | ❌ | 只能 `raw_spin_trylock`/无锁 | 仅 NMI 安全 RCU | ❌ |

**核心规律（原文加粗）**：**上下文越深/越原子，能用的原语越少。** "我能在这睡吗？"
等价于"`in_task()` 为真且抢占没被禁吗？" 不确定时，在 debug 内核里用 `might_sleep()`
——它会替你判断并报警。

---

## 7. 逐句拆解 PREEMPT_RT 的改动

### 晦涩点 ⑧：RT 改的是"上下文归属"，不是位布局

原文 §7 关键：RT **不改 preempt_count 的位布局**，改的是**很多东西跑在哪个上下文**：

- 多数 IRQ handler 和软中断搬进**任务上下文**（Topic 07/08）→ 变得可抢占、可拿 RT mutex。
- `spin_lock` 在 RT 上变成**可睡眠的锁**；`raw_spinlock_t` 保持真自旋（所以 `desc->lock`
  是 raw 的）。
- `local_bh_disable` 变成每 CPU 可睡眠锁。
- `local_irq_disable` 仍然关中断，但**极少代码在它持有期间运行**（都搬走了）。

于是 RT 上"我在软中断里，不能睡"变成了"我在线程里，能睡"。**真正必须原子的代码**
用 `raw_*` 原语和 `IRQ_WORK_HARD_IRQ`（Topic 09）守住最后阵地。

---

## 8. 动手任务

### 任务 0：在运行的内核里观察 preempt_count 配置（10 分钟）

```bash
zcat /proc/config.gz 2>/dev/null | grep -E "PREEMPT|PROVE_LOCKING|DEBUG_ATOMIC_SLEEP" \
  || grep -E "PREEMPT|PROVE_LOCKING|DEBUG_ATOMIC_SLEEP" /boot/config-$(uname -r)
```

回答（答案 §9.0）：

1. 你的内核是 `CONFIG_PREEMPT_NONE`、`PREEMPT_VOLUNTARY`、`PREEMPT` 还是 `PREEMPT_RT`？
   这影响"普通代码能否被抢占"。
2. `CONFIG_PROVE_LOCKING` 和 `CONFIG_DEBUG_ATOMIC_SLEEP` 开了吗？（发行版生产内核
   通常关，debug 内核开。）
3. 如果两个 debug 选项都没开，你做不了任务 3 的"自动抓 bug"——想想为什么。

### 任务 1：源码核对位布局与判断宏（20 分钟）

```
① include/linux/preempt.h:33   PREEMPT/SOFTIRQ/HARDIRQ/NMI 的 BITS 和 SHIFT
② include/linux/preempt.h:126  in_nmi/in_hardirq/in_serving_softirq/in_softirq/
                               in_interrupt/in_task 六个宏的实现
③ 对照 Topic 08 晦涩点⑦：in_serving_softirq 用 & SOFTIRQ_OFFSET，
   in_softirq 用整个 softirq_count() —— 亲眼确认这个差异
```

回答（答案 §9.1）：

1. SOFTIRQ 占 8 位、HARDIRQ 只占 4 位。为什么软中断需要比硬中断更多的嵌套位？
   （提示：`local_bh_disable` 能嵌套多深 vs 硬中断嵌套多深。）
2. `in_interrupt()` 包含哪三种上下文？它和 `in_task()` 是互补的吗？
3. `in_serving_softirq()` 和 `in_softirq()` 的实现差异具体在哪一个位运算上？

### 任务 2：写模块打印自己在各上下文的 preempt_count（40 分钟）⭐

在不同上下文里打印 `preempt_count()` 的十六进制值和各 `in_*()` 判断，**亲眼看到
筹码叠加**：

```c
static void dump_ctx(const char *where) {
    pr_info("%s: pc=%#x in_task=%d in_hardirq=%d in_serving_softirq=%d "
            "in_softirq=%d in_nmi=%d irqs_disabled=%d\n",
            where, preempt_count(), in_task(), in_hardirq(),
            in_serving_softirq(), in_softirq(), in_nmi(), irqs_disabled());
}
static int __init m_init(void) {
    dump_ctx("module_init (process ctx)");          /* 应 in_task=1 */
    local_irq_disable(); dump_ctx("after local_irq_disable"); local_irq_enable();
    preempt_disable();   dump_ctx("after preempt_disable");   preempt_enable();
    local_bh_disable();  dump_ctx("after local_bh_disable (softirq count!)"); local_bh_enable();
    return 0;
}
```

再结合 Topic 07/09 的模块，在 top half、tasklet、irq_work 里也调 `dump_ctx`。
回答（答案 §9.2）：

1. `module_init` 里 `in_task` 是 1 吗？`local_irq_disable` 后 `irqs_disabled` 变 1 但
   `in_interrupt` 仍是 0 ——这印证 §2 哪个要点（关中断 ≠ 在中断里）？
2. `local_bh_disable` 后 `in_softirq` 变 1 但 `in_serving_softirq` 仍是 0 ——
   为什么？（晦涩点 ① 的双重用途。）
3. `preempt_disable` 后 `pc` 的哪 8 位变了？`local_bh_disable` 又是哪 8 位、加了几？

### 任务 3：故意触发 lockdep / atomic-sleep 告警（40 分钟，需要 debug 内核）⭐⭐

> 需要 `CONFIG_PROVE_LOCKING` + `CONFIG_DEBUG_ATOMIC_SLEEP`。没有的话只能读不能跑。

```c
// 违规①：在硬中断 handler / tasklet 里睡眠
static void bad_tasklet(struct tasklet_struct *t) {
    msleep(1);   /* DEBUG_ATOMIC_SLEEP 会报 BUG */
}
// 违规②：一把锁，一处用 spin_lock（进程），另一处在中断 handler 里也拿它，
//         但进程侧没用 _irqsave —— lockdep 报 inconsistent lock state
```

回答（答案 §9.3）：

1. 违规① 触发的 BUG 信息里，关键字是什么？它打印了 `preempt_count` 和
   `irqs_disabled` 吗？
2. 违规② 的 lockdep 告警里有**两个**调用栈——分别代表什么（安全获取 vs 不安全获取）？
3. 把违规② 的进程侧锁改成 `spin_lock_irqsave`，告警消失吗？为什么？

### 任务 4：思考题（答案 §9.4）

1. 软中断运行时"中断开着但不能睡"。既然中断开着，为什么还不能睡？到底是什么阻止了
   睡眠？（提示：不是中断状态，是 preempt_count 里哪一段。）
2. 你在进程上下文持有 `spin_lock_irqsave`。此时 `in_interrupt()` 是真还是假？
   `irqs_disabled()` 呢？这说明这俩判断的独立性。
3. 一把锁只被软中断和进程上下文共享（硬中断从不碰）。进程侧用 `spin_lock_bh`。
   现在有人**新增**了一处硬中断 handler 也拿这把锁但没改进程侧——会死锁吗？
   lockdep 会抓到吗？
4. 为什么 NMI 里连 `spin_lock`（普通自旋锁）都不能用，只能 `raw_spin_trylock`？
   （提示：NMI 能打断任何东西，包括一个**正持有这把锁**的普通上下文。）

---

## 9. 任务参考答案

### 9.0 任务 0

1. 多数桌面/服务器发行版是 `PREEMPT_VOLUNTARY` 或 `PREEMPT`（较新 Ubuntu 桌面是
   PREEMPT_DYNAMIC，运行时可调）。RT 内核才是 PREEMPT_RT。
2. 生产内核通常**都关**（有性能开销）；只有 debug/测试内核开。
3. 没开就没有"自动建模和报警"的机制——bug 仍会发生，只是不会被当场抓住，
   会变成生产环境的随机死机。这就是为什么开发要用 debug 内核。

### 9.1 任务 1

1. `local_bh_disable` 可以**深度嵌套**（一个函数禁 BH 里调用另一个也禁 BH 的函数），
   且每次加 2（DISABLE_OFFSET），需要更多位计数；硬中断嵌套层数受限（向量数、
   优先级），4 位（最多 15 层）足够。
2. `in_interrupt()` = 硬中断 | 软中断 | NMI。它和 `in_task()` **基本互补**——
   在中断中就不在任务里，反之亦然（注意 bh_disable 让 in_softirq 真但其实仍在
   进程上下文，是个细微边界）。
3. `in_serving_softirq()` = `softirq_count() & SOFTIRQ_OFFSET`（只测最低那 1 份）；
   `in_softirq()` = `softirq_count()`（测全部 8 位，含 bh_disable 加的 2 份）。

### 9.2 任务 2

1. 是 1。`local_irq_disable` 后 `irqs_disabled=1` 但 `in_interrupt=0`——证明
   "关中断"和"在中断里"是两回事（§2 晦涩点 ②）。
2. `local_bh_disable` 加的是 `SOFTIRQ_DISABLE_OFFSET`（2 份），让 `softirq_count()`
   非零 → `in_softirq` 真；但没加那"正在服务"的标志位语义，所以 `in_serving_softirq`
   （只看 1 份 OFFSET 的服务态）为假。
3. `preempt_disable` 改最低 8 位（PREEMPT 段）+1；`local_bh_disable` 改 8~15 位
   （SOFTIRQ 段）+2（即 +0x200，因 SOFTIRQ_OFFSET=0x100）。

### 9.3 任务 3

1. `BUG: sleeping function called from invalid context at ...`，并打印
   `in_atomic(): 1, irqs_disabled(): X, ... preempt_count` 等状态。
2. 两个栈：一个是"这把锁曾在硬中断上下文/中断安全地被获取"的位置，另一个是
   "这把锁在开中断（hardirq-unsafe）情况下被获取"的位置——lockdep 指出这两种获取
   组合可能死锁。
3. 消失。`spin_lock_irqsave` 在进程侧持锁期间关了硬中断，硬中断 handler 不可能在
   持锁期间插进来拿同一把锁，死锁配方被破坏，lockdep 满意。

### 9.4 任务 4

1. 阻止睡眠的不是中断状态，而是 **preempt_count 里 SOFTIRQ 段非零**（`in_serving_softirq`
   为真，是原子上下文）。调度器看到 preempt_count != 0 就拒绝切换任务——睡眠需要
   切换任务，所以不行。中断开着只是说"硬件能打断我"，与"我能否主动让出 CPU"无关。
2. `in_interrupt()` = **假**（你在进程上下文，没在处理任何中断）；
   `irqs_disabled()` = **真**（irqsave 关了中断）。完美展示两者独立：关着中断但不在中断里。
3. **会死锁**（新增硬中断 handler 在 CPU 上拿锁时，若进程侧正持锁且只 _bh 没关硬中断，
   硬中断插入自旋等待 → §3 死锁）。**lockdep 会抓到**——它发现这把锁现在既在硬中断
   上下文被拿、又在 hardirq-unsafe（只 bh）情况下被拿，立即报 inconsistent state。
   这正是 lockdep 的价值：新增代码引入的隐患当场暴露。
4. NMI 不可屏蔽，能打断**任何**上下文——包括一个**正持有这把普通 spinlock** 的进程/
   中断上下文。如果 NMI handler 也去 `spin_lock` 同一把锁，它会自旋等一个**永远不会
   放锁**的持有者（持有者被 NMI 自己打断了，回不去放锁）→ 死锁。所以 NMI 里只能用
   `raw_spin_trylock`（拿不到就放弃，不自旋）或彻底无锁的数据结构。这也是 irq_work
   （Topic 09）存在的部分原因——从 NMI 安全地"升级"出去。

---

## 10. 学完自测

1. 画出 preempt_count 的四段位布局，各段什么时候加减筹码？
2. "关中断、关抢占、在中断里"三个状态各自独立，举一个"关中断但不在中断里"的例子。
3. §3 的经典中断死锁怎么发生？`spin_lock_irqsave` / `_bh` / 裸锁怎么选？口诀是什么？
4. lockdep 怎么在死锁发生前抓到它？为什么要用 debug 内核测试？
5. 默写主表：硬中断/软中断/NMI 里，能睡吗、用哪种锁、能否 synchronize_rcu？

下一章 Topic 11：现代的消息信号中断 MSI/MSI-X——建在 Topic 04 的层级 domain 之上，
在你刚学的这套并发规则下投递。
