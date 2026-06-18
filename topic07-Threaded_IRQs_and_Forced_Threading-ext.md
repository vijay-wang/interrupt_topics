# Topic 07 扩展版 — 线程化中断与强制线程化

> **用法同前**。行号基于 `~/repo/linux`（v7.1-rc7）：`kernel/irq/manage.c` 里
> `force_irqthreads_key:28`、`irq_finalize_oneshot:1086`、`irq_thread_fn:1141`、
> `irq_forced_thread_fn:1158`、`irq_thread:1244`；`threadirqs` 注册在 `manage.c:35`；
> `IRQTF_*` 在 `kernel/irq/internals.h:36` 起。
>
> **承接**：Topic 06 的 top half 铁律说"不能睡眠、必须短"。但很多设备的处理就是要睡、
> 要长——本章讲怎么破这个限制。也是 Topic 05 晦涩点 ⑤ 那个 `IRQF_ONESHOT` 钩子的下文。

---

## 1. 先建立画面感：为什么要把中断"线程化"？

### 1.1 top half 的紧身衣

Topic 06 反复强调：硬中断 handler 像穿着紧身衣——关着中断、不能睡、必须秒退。
可现实中很多设备的处理**做不到**：

- **要睡眠**：处理过程要读一个挂在 I2C/SPI 慢总线上的寄存器（通信本身会睡），
  或要拿 mutex、等一个 completion。
- **要花时间**：一次中断要处理一批数据。
- **会拖累别人**：handler 在硬中断里跑得久，**这个 CPU 上所有其他中断都被它堵着**
  （Topic 14 延迟）。

### 1.2 解法：把重活搬进一个"专属工人"

**画面**：前台（硬中断 handler）只负责"收下包裹、登记一下"，**真正拆包裹、处理内容
的活交给后台一个专属员工**（内核线程）。这个员工坐在办公室里（进程上下文），
**可以喝咖啡（睡眠）、可以慢慢干、被打断了还能让位**（可抢占）。

于是中断处理被劈成两半：

- **硬中断部分缩到最小**："应答设备 + 喊一声让线程来"（返回 `IRQ_WAKE_THREAD`）。
- **重活搬到一个专属内核线程**（你在 `ps` 里看到的 `irq/N-name`），跑在进程上下文。

PREEMPT_RT 实时内核把这变成**几乎所有中断的默认形态**（§5）——这是本章和 Topic 14
的核心连接点。

---

## 2. 逐句拆解"一劈两半"

```c
request_threaded_irq(irq, primary_handler, thread_fn, IRQF_ONESHOT, name, dev);
```

两个函数，两个世界：

| | `primary_handler` | `thread_fn` |
|---|---|---|
| 上下文 | 硬中断（紧身衣）| IRQ 内核线程（进程上下文，自由）|
| 能睡吗 | 不能 | **能** |
| 做什么 | 应答设备、读个状态、**返回 `IRQ_WAKE_THREAD`** | 真正的处理（睡眠/慢总线/批处理）|
| 何时跑 | 中断一来立刻（Topic 05 的 action 链）| 被 primary 唤醒后，由调度器安排 |

### 晦涩点 ①：primary 传 NULL 会怎样？——"默认 primary"

原文 §2 末段：如果你 `handler == NULL`，核心装一个**默认 primary handler**
（`irq_default_primary_handler`），它什么都不做、只返回 `IRQ_WAKE_THREAD`——
即"每次中断都直接叫线程"。这是**纯线程化驱动**最常见的写法：

```c
request_threaded_irq(irq, NULL, my_thread_fn, IRQF_ONESHOT, "mydev", dev);
/*                        ↑ 默认 primary：来了就唤醒线程 */
```

注意原文那句："这正是为什么此时 `IRQF_ONESHOT` 是必须的"——下一节解释。

---

## 3. 逐句拆解 `IRQF_ONESHOT`：把线一直屏蔽到线程干完

### 晦涩点 ②：ONESHOT 解决的竞争

这是本章最需要画面感的一点。场景：**电平触发线 + NULL primary handler**。

电平线"一直按着不放"直到设备被服务（Topic 02/05）。但现在"服务"发生在**线程里**，
而线程要等调度器安排才跑——**从硬中断返回到线程真正开跑，中间有一段时间**。
这段时间里：硬中断返回了 → 线还高着（设备没被服务）→ **立刻又触发中断** →
又唤醒线程 → 又返回 → 又触发……**线程一次都还没跑，中断已经风暴了**。

`IRQF_ONESHOT` 的解法：让 flow handler（Topic 05 晦涩点 ⑤ 那行
`if (IRQS_ONESHOT) mask_irq`）在**硬中断触发时就把线屏蔽**，并**一直保持屏蔽到线程跑完**，
线程结束才解屏蔽 + EOI。**"一次触发，屏蔽到底，处理完才开闸"**——ONESHOT 名字的由来。

### 晦涩点 ③：`threads_oneshot` 位图与共享线

`irq_thread_fn`（`manage.c:1141`）：

```c
static irqreturn_t irq_thread_fn(struct irq_desc *desc, struct irqaction *action)
{
    irqreturn_t ret = action->thread_fn(action->irq, action->dev_id);  /* 你的 thread_fn */
    if (ret == IRQ_HANDLED) atomic_inc(&desc->threads_handled);
    irq_finalize_oneshot(desc, action);   /* ★清掉本 action 在 threads_oneshot 里的位；
                                              当最后一位被清，才 UNMASK + EOI */
    return ret;
}
```

`desc->threads_oneshot`（Topic 03 见过这个字段）是个**位图**，共享线上每个 action
占一位（通过 `action->thread_mask`）。**只有当这条线上所有 oneshot 线程都跑完
（所有位都清零），才解除屏蔽**。

**画面**：会议室门口的签到板，几个部门（共享线上的多个驱动）都在用。最后一个人签退，
管理员才锁门开灯。任何一个还在里面，门就不锁。

**铁律**（核心强制）：`thread_fn` 非空且无 primary handler 时，**必须**设 ONESHOT。
（Topic 06 晦涩点 ⑥ 讲的 `IRQF_COND_ONESHOT` 就是让共享伙伴能"传染"这个要求。）

---

## 4. 逐句拆解 IRQ 线程本体

`irq_thread`（`manage.c:1244`）是那个专属工人的主循环（简化）：

```c
static int irq_thread(void *data)
{
    struct irqaction *action = data;
    irq_thread_set_ready(desc, action);     /* 置 IRQTF_READY，告诉创建者"我就绪了" */
    sched_set_fifo(current);                 /* ★设为 SCHED_FIFO 实时优先级！见晦涩点④ */
    handler_fn = forced ? irq_forced_thread_fn : irq_thread_fn;

    while (!irq_wait_for_interrupt(desc, action)) {  /* ★睡眠，直到被唤醒（晦涩点⑤）*/
        irqreturn_t r = handler_fn(desc, action);    /* → 你的 thread_fn */
        wake_threads_waitq(desc);                    /* 唤醒等在 synchronize_irq 上的人 */
    }
    return 0;
}
```

### 晦涩点 ④：为什么线程是 SCHED_FIFO（实时优先级）？

普通线程跑在 SCHED_OTHER（公平调度），会和你的浏览器、编译任务**抢 CPU**。
但中断处理代表"硬件有事相求"，不能被一堆普通任务饿死。所以 IRQ 线程默认是
**SCHED_FIFO**（实时调度类，`sched_set_fifo`）——优先级高于所有普通任务。
这也意味着：**IRQ 线程的优先级和 CPU 亲和性是延迟调优的关键旋钮**（Topic 14）。

### 晦涩点 ⑤：`irq_wait_for_interrupt` 和 `IRQTF_RUNTHREAD` 唤醒链

线程平时**睡着**（`irq_wait_for_interrupt` 阻塞）。primary handler 返回
`IRQ_WAKE_THREAD` 时，`__irq_wake_thread`（`kernel/irq/handle.c`）做两件事：
**置 `IRQTF_RUNTHREAD` 标志 + 唤醒线程**。线程醒来看到这个标志，跑一遍 handler_fn，
然后继续睡。

**画面**：员工戴着传呼机睡觉，前台收到包裹按一下传呼（置 RUNTHREAD + 唤醒），
员工醒来干活，干完接着睡。`IRQTF_*` 这族标志（`internals.h:36`）就是传呼机的几种信号：
`RUNTHREAD`（快起来干活）、`READY`（我准备好了）、`FORCED_THREAD`（你是被强制线程化的）、
`WARNED`（你返回了 WAKE_THREAD 却没 thread_fn，已经骂过你一次了）。

### 晦涩点 ⑥：`free_irq` 怎么安全地关掉这个线程？

原文：`free_irq` → `synchronize_hardirq`（等硬中断部分跑完）→ `kthread_stop`（停线程）；
`irq_thread_dtor` 这个收尾任务清掉残留的 RUNTHREAD/oneshot 位，**确保线不会被永久
屏蔽着**。如果不清，线程被强杀时 threads_oneshot 的位没清零 → 那条线永远不解屏蔽 →
设备假死。这是个微妙但重要的清理。

### 晦涩点 ⑦：`secondary` action 是干嘛的？

原文 §4 末："The secondary action exists for forced threading"。场景：一个**没打算
线程化**的驱动，它的 handler 被**强制线程化**了（§5），但这个 handler 自己又返回了
`IRQ_WAKE_THREAD`（它以为自己在硬中断里，想再唤醒一个线程）。这时 `secondary` 这张
"备用登记条"承接那个本该被唤醒的线程。属于边角情况，知道这个字段为何存在即可。

---

## 5. 逐句拆解强制线程化与 PREEMPT_RT

### 晦涩点 ⑧：`threadirqs` 启动参数和 static key

```c
DEFINE_STATIC_KEY_FALSE(force_irqthreads_key);    /* manage.c:28，默认关 */
early_param("threadirqs", setup_forced_irqthreads); /* manage.c:35，启动参数翻开它 */
```

**static key（静态键）是什么？** 一种**零开销的运行时开关**：用代码自修改技术，
关闭时那段判断**几乎不耗 CPU**（不是普通 if，而是直接把分支 patch 掉）。
内核里大量"平时关、偶尔开"的特性用它。

用 `threadirqs` 启动后，这个键翻开，核心**强制把符合条件的中断都线程化**——
即使驱动只给了 top half。**豁免名单**（原文 `__setup_irq` 那段判断）：
`IRQF_NO_THREAD`（定时器，Topic 06 晦涩点 ⑧）、`IRQF_PERCPU`（每 CPU 线，Topic 12）、
已经是 oneshot 线程化的。

### 晦涩点 ⑨：`irq_forced_thread_fn` —— "假装还在硬中断里"的兼容魔术

`manage.c:1158`：

```c
static irqreturn_t irq_forced_thread_fn(struct irq_desc *desc, struct irqaction *action)
{
    local_bh_disable();                                   /* 硬中断 handler 本来就关着软中断 */
    if (!IS_ENABLED(CONFIG_PREEMPT_RT)) local_irq_disable(); /* 非 RT：也关硬中断 */
    ret = irq_thread_fn(desc, action);
    if (!IS_ENABLED(CONFIG_PREEMPT_RT)) local_irq_enable();
    local_bh_enable();
    return ret;
}
```

**问题**：一个为硬中断写的 handler，假设了"我运行时中断是关的、软中断是关的"
（它可能据此省略了某些锁）。强制线程化后它跑在线程里，这些假设**不再成立**——
直接搬过去可能有竞争 bug。

**魔术**：`irq_forced_thread_fn` 在调用这个 handler 前后，**人为重建那些隐含保证**
——关软中断（BH）、（非 RT 时）关硬中断。于是 handler 仍然看到"中断关、软中断关"的
环境，**对它而言透明**。这就是原文说的 "compatibility trick"：让旧 handler 在新机制下
不改一行也能正确运行。

### 晦涩点 ⑩：为什么 RT 上偏偏**不**关硬中断？

注意上面 `!IS_ENABLED(CONFIG_PREEMPT_RT)` 那个条件——**RT 内核里这个线程不关硬中断，
只关 BH**。因为 PREEMPT_RT 的整个目标就是**让中断处理可被抢占**（这样更高优先级的
实时任务能随时插队，保证低延迟）。关硬中断会破坏可抢占性，与 RT 宗旨相悖。所以 RT 下
handler 跑在可抢占的线程里、不关硬中断——这是 RT 内核中断模型的精髓，Topic 14 详述。

RT 上的"幸存者"（仍真正跑在硬中断的）：`IRQF_NO_THREAD` 线、每 CPU 中断、定时器。
连软中断（Topic 08）在 RT 上也搬进了任务上下文。

---

## 6. 逐句拆解"该用哪种延迟机制"

原文 §6 的决策表是全书最实用的速查表之一，加一列"为什么"：

| 需求 | 用什么 | 为什么 |
|---|---|---|
| 极小、不睡、最低延迟 | **纯 top half**（request_irq）| 没有调度延迟 |
| 要睡/慢总线/较长，想要 RT 优先级和专属线程 | **线程化 IRQ** + ONESHOT | 专属、实时优先、自动屏蔽线 |
| 要睡，活儿偶发/可批，不需要专属线程 | **工作队列**（Topic 09）| 共享 worker 池，省线程 |
| 硬/软中断/NMI 上下文里要回调 | **irq_work**（Topic 09）| 能在这些受限上下文安全排队 |
| 事件率极高，想轮询 | **NAPI / irq_poll**（Topic 09）| 高负载下轮询比逐个中断省 |

### 晦涩点 ⑪：线程化 IRQ vs 工作队列，到底怎么选？

两者都能睡，区别：

- **线程化 IRQ**：给你一个**专属、RT 优先级、绑定到该中断 CPU**的线程，外加自动的
  线屏蔽（ONESHOT）。适合"处理这条线**本身就需要**的工作"。
- **工作队列**：共享 worker 池，没有线屏蔽语义，调度延迟相对不可控。适合"中断的
  **后续跟进**工作，能容忍调度延迟"。

判断口诀：**服务这条线的固有工作 → 线程化 IRQ；可拖延的后续杂活 → 工作队列。**

---

## 7. 动手任务

### 任务 0：在 ps/top 里找到 IRQ 线程（10 分钟，无需 root）

```bash
ps -eo pid,cls,rtprio,comm | grep -E "irq/|CLS" | head -20
# cls=FF 表示 SCHED_FIFO，rtprio 是实时优先级
```

回答（答案 §8.0）：

1. 你机器上有 `irq/N-xxx` 命名的线程吗？它们的调度类（cls）是 FF（FIFO）吗？
   印证 §4 晦涩点 ④ 哪句话？
2. 如果**没有**任何 irq/ 线程，说明你的内核没开 `threadirqs` 且没有驱动主动线程化
   ——多数桌面发行版就是这样。那这些线程什么情况下才大量出现？
3. `cat /proc/cmdline | grep -o threadirqs` ——你的内核启动带 threadirqs 了吗？

### 任务 1：源码追"一劈两半 + ONESHOT 解屏蔽"（35 分钟）

```
① kernel/irq/manage.c:1244   irq_thread：找到 sched_set_fifo、while 循环、
                              forced ? 选择 handler_fn 的三个点
② kernel/irq/manage.c:1141   irq_thread_fn：看它调你的 thread_fn 后调 finalize
③ kernel/irq/manage.c:1086   irq_finalize_oneshot：找到清 threads_oneshot 位、
                              "最后一位清零才 unmask" 的逻辑
④ kernel/irq/handle.c        __irq_wake_thread：找到置 IRQTF_RUNTHREAD + wake 的两步
⑤ kernel/irq/internals.h:36  IRQTF_* 枚举：对照晦涩点⑤的"传呼机信号"
```

回答（答案 §8.1）：

1. `irq_finalize_oneshot` 里"最后一个 oneshot 线程跑完才 unmask"是怎么判断的？
   （看它检查 `threads_oneshot` 的什么条件。）
2. `__irq_wake_thread` 设的标志和 `irq_wait_for_interrupt` 等的标志是同一个吗？
3. `irq_thread` 里 `forced` 为真时选 `irq_forced_thread_fn`——这个分支什么情况成立？

### 任务 2：写一个线程化中断模块，对比 primary 和 thread 上下文（60 分钟）⭐

基于 Topic 04/06 的骨架，这次注册**线程化**中断，在两个函数里分别打印上下文信息：

```c
static irqreturn_t my_primary(int irq, void *dev) {
    pr_info("PRIMARY: in_irq=%lu, irqs_disabled=%d, can_sleep=NO\n",
            (unsigned long)in_hardirq(), irqs_disabled());
    return IRQ_WAKE_THREAD;       /* 把活交给线程 */
}
static irqreturn_t my_thread(int irq, void *dev) {
    pr_info("THREAD: in_irq=%lu, irqs_disabled=%d, pid=%d comm=%s\n",
            (unsigned long)in_hardirq(), irqs_disabled(),
            current->pid, current->comm);
    msleep(5);                    /* ★线程里睡眠——合法！ */
    pr_info("THREAD: woke up after sleep, still fine\n");
    return IRQ_HANDLED;
}
// request_threaded_irq(virq, my_primary, my_thread, IRQF_ONESHOT, "thr-demo", &dev);
```

提示：用 Topic 04 任务 3 的虚拟 domain + `generic_handle_domain_irq` 触发，
或直接 `irq_wake_thread()` 手动唤醒。回答（答案 §8.2）：

1. 两个函数打印的 `in_hardirq` / `irqs_disabled` 有什么不同？`current->comm` 在 thread
   里是什么？（应该看到 `irq/N-thr-demo`。）
2. thread 里的 `msleep(5)` 没有触发任何 BUG——把同样的 msleep 放进 primary 会怎样？
   （回忆 Topic 06 任务 2。）
3. 把 `IRQF_ONESHOT` 去掉、primary 改成返回 NULL（即纯线程化无 primary），
   编译/加载时核心会怎样？（验证 §3 的强制规则。）

### 任务 3：用 threadirqs 重启，观察中断线程化（30 分钟，需要 root + 改启动参数）

> ⚠️ 虚拟机里做。改 GRUB 加 `threadirqs` 启动参数。

```bash
# 临时改（GRUB 编辑界面在内核行末尾加 threadirqs），重启后：
cat /proc/cmdline | grep threadirqs
ps -eo cls,rtprio,comm | grep "irq/" | wc -l   # 现在应该一大堆
ps -eo cls,rtprio,comm | grep "irq/" | head -20
```

回答（答案 §8.3）：

1. 加 threadirqs 前后，`irq/` 线程数量差多少？
2. 找一个网卡或磁盘的 irq 线程，它的 rtprio 是多少？和普通用户进程（cls=TS）
   比，谁优先？
3. 哪些中断**即使** threadirqs 也没有对应线程？（找 timer 相关的，对照 §5 豁免名单。）

### 任务 4：思考题（答案 §8.4）

1. 为什么 `IRQF_ONESHOT` 对**电平**线是刚需，对**边沿**线却往往不必要？
   （回忆 Topic 02/05：边沿处理期间又来一个会怎样 vs 电平。）
2. IRQ 线程是 SCHED_FIFO。如果一个线程化 handler 里写了死循环（bug），
   在单核机器上会发生什么？比普通线程的死循环更危险吗？
3. `irq_forced_thread_fn` 在非 RT 上关硬中断、在 RT 上不关。一个驱动 handler 依赖
   "我运行时硬中断关着"来保护一个数据结构——它在 RT 强制线程化下还安全吗？
   应该改用什么？
4. 设备 ack 应该放在 primary 还是 thread_fn？对电平线，放错地方会怎样？
   （提示：电平线在被 ack/服务前一直拉高。）

---

## 8. 任务参考答案

### 8.0 任务 0

1. 有的话 cls 应为 FF——印证"IRQ 线程默认 SCHED_FIFO，不被普通任务饿死"。
2. 开 `threadirqs` 启动参数时、PREEMPT_RT 内核上、或驱动显式用 request_threaded_irq
   且带 thread_fn 时，才会出现专属 irq 线程。纯桌面非 RT 内核大多只有少数几个。
3. 多数发行版默认**没有** threadirqs。

### 8.1 任务 1

1. `irq_finalize_oneshot` 清掉 `action->thread_mask` 对应位后，检查
   `desc->threads_oneshot` 是否变为 0（所有 action 的位都清了），是则 unmask+eoi。
2. 是同一个：`IRQTF_RUNTHREAD`。唤醒方置位，线程方等待它被置位。
3. `forced` 为真当该 action 是被强制线程化的（带 `IRQTF_FORCED_THREAD`，
   即 threadirqs/RT 下把纯 top-half 驱动线程化的情形）。

### 8.2 任务 2

1. primary：`in_hardirq=1`、`irqs_disabled=1`（真硬中断上下文）。thread：
   `in_hardirq=0`、`irqs_disabled=0`、comm=`irq/N-thr-demo`（进程上下文）。
2. primary 里 msleep 会触发 "scheduling while atomic" / "sleeping function called
   from invalid context"——因为硬中断上下文不可睡。thread 里完全合法。
3. 核心强制要求：thread_fn 非空且 primary 为 NULL 时必须 ONESHOT，否则
   request_threaded_irq 返回 -EINVAL（拒绝注册），并可能 WARN。

### 8.3 任务 3

1. 通常从个位数暴增到几十上百个（几乎每条普通中断线一个）。
2. rtprio 通常是 50 左右（FIFO 实时优先级）；普通进程 cls=TS（SCHED_OTHER）
   没有 rtprio，**IRQ 线程优先**。
3. 定时器（带 IRQF_NO_THREAD via IRQF_TIMER）、每 CPU 中断（IRQF_PERCPU）
   ——没有对应 irq/ 线程。

### 8.4 任务 4

1. 电平线在被服务前**持续拉高**，不屏蔽就在线程跑之前无限风暴，所以 ONESHOT 刚需。
   边沿是瞬间事件，硬中断 ack 后线就不再"持续触发"，处理期间真来新边沿的话由
   `IRQS_PENDING` 机制（Topic 05）兜底，通常不需要 ONESHOT 全程屏蔽。
2. SCHED_FIFO 死循环在单核上极危险：它优先级高于所有普通任务，会**饿死整个系统**
   （普通任务永远抢不到 CPU，可能连 shell 都没响应）。比普通线程死循环危险得多
   ——普通线程至少会被公平调度让出。这是 RT 优先级的双刃剑。
3. **不安全**。RT 下该线程可被抢占、硬中断没关，"关硬中断保护数据"的假设失效。
   应改用**显式的锁**（spinlock，Topic 10）来保护数据，而不是依赖隐式的中断状态
   ——这正是 RT 改造内核时的大量工作。
4. 电平线的 ack 应放在 **primary handler**（或依赖 ONESHOT 的屏蔽）。若把 ack 拖到
   thread_fn，从硬中断返回到线程跑之间，电平线还高着且未被屏蔽（若没 ONESHOT）会
   持续触发——所以要么 primary 里 ack，要么 ONESHOT 屏蔽到线程 ack 为止。

---

## 9. 学完自测

1. 用"前台+后台员工"讲清 primary handler 和 thread_fn 的分工和各自能/不能做什么。
2. `IRQF_ONESHOT` 解决什么竞争？`threads_oneshot` 位图为什么是位图而不是布尔？
3. IRQ 线程为什么用 SCHED_FIFO？这给延迟调优带来什么旋钮？
4. `irq_forced_thread_fn` 的"兼容魔术"是什么？为什么 RT 上不关硬中断？
5. 线程化 IRQ vs 工作队列，选择口诀是什么？

下一章 Topic 08：最底层的延迟机制——软中断（softirq），以及"硬中断尾巴"上跑延迟工作的
那道缝（Topic 01 晦涩句 ㉑ 埋的钩子）。
