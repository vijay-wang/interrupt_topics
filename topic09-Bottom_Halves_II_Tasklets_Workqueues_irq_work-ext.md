# Topic 09 扩展版 — 下半部 II：tasklet、工作队列、irq_work

> **用法同前**。行号基于 `~/repo/linux`（v7.1-rc7）：`tasklet_schedule` 在
> `include/linux/interrupt.h:758`、`tasklet_struct` 在 `interrupt.h:692` 附近、
> `WQ_BH` 在 `include/linux/workqueue.h:372`、`system_bh_wq` 在 `workqueue.h:473`、
> `irq_work_queue` 在 `kernel/irq_work.c:116`、`tasklet_action_common` 在
> `kernel/softirq.c:916`。
>
> **承接**：Topic 08 讲了最底层的软中断地基。本章讲**建在地基之上的几种更灵活的
> 延迟机制**，并给出贯穿 Topic 05~09 的"我到底该用哪一个"完整决策矩阵。

---

## 1. 先建立画面感：延迟工作的"五个去处"

到目前为止，硬中断 handler（top half，Topic 06）干完紧急的事，剩下的活可以扔给
**五个去处**。先把全景画出来（这张表是本章的骨架）：

```
                  跑在什么上下文          能睡吗   建在什么之上
线程化 IRQ  ───►  进程（专属kthread,FIFO）  能      Topic 07
工作队列    ───►  进程（共享worker池）       能      kernel/workqueue.c
BH工作队列  ───►  软中断                     不能    WQ_BH（tasklet 的现代替代品）
tasklet     ───►  软中断（TASKLET/HI）        不能    Topic 08（遗留机制）
irq_work    ───►  硬中断（自 IPI）            不能    可从 NMI 里排队！
irq_poll    ───►  软中断（IRQ_POLL）          不能    NAPI 式轮询
```

Topic 07 已讲第一个（线程化）。本章讲剩下的非线程化的四个 + 工作队列，§6 用一张矩阵
统一它们。**核心心智**：选延迟机制就是回答两个问题——"**要不要睡眠？**" 和
"**从什么上下文排队的？**"

---

## 2. 逐句拆解 tasklet（先理解，但新代码别用）

### 晦涩点 ①：tasklet 比软中断多给的唯一保证

tasklet 跑在软中断上下文（TASKLET_SOFTIRQ 或 HI_SOFTIRQ 向量），所以**不能睡**
（继承软中断的约束，Topic 08）。它比裸软中断多给一个保证：

> **同一个 tasklet 永不与自己并发**——即使在不同 CPU 上。

回忆 Topic 08：同一个软中断向量可以在 8 个 CPU 上同时跑。对 net_rx 这没问题
（它内部做了 per-CPU 处理）。但如果你的延迟函数访问一个**全局**数据结构，"同时在多核
跑同一个函数"就会打架。tasklet 用一个 `TASKLET_STATE_RUN` 状态位保证"全局只有一个
实例在跑"——省了你自己加锁。

```c
struct tasklet_struct {
    struct tasklet_struct *next;   /* per-CPU 挂起链表 */
    unsigned long state;           /* SCHED（已排队）/ RUN（正在跑）*/
    atomic_t count;                /* != 0 表示被禁用 */
    bool use_callback;
    union {
        void (*func)(unsigned long data);       /* 旧风格 */
        void (*callback)(struct tasklet_struct *t);  /* 新风格（推荐）*/
    };
    unsigned long data;
};
```

运行机制：`tasklet_schedule(t)`（`interrupt.h:758`）把 `t` 挂到本 CPU 的 tasklet 链表，
然后 `raise_softirq(TASKLET_SOFTIRQ)`。软中断跑时（Topic 08），`tasklet_action_common`
（`softirq.c:916`）遍历链表，用 `tasklet_trylock` 实现"不与自己并发"，`count==0` 是
"已启用"判断。

### 晦涩点 ②：为什么新代码不推荐 tasklet？

原文 §2 末段三条理由，翻译：

1. **不能睡却延迟更差**：它跑在软中断里（和裸软中断一样不能睡），但比裸软中断**多了一层
   链表/锁的折腾**，延迟反而更高；又比工作队列**少了灵活性**（不能 flush/cancel/命名）。
   两头不讨好。
2. **那个并发保证常被误解、且很少真用到**：多数人其实不需要"永不自并发"，只是不知道。
3. **内核正在迁移**：把 tasklet 用户迁到 **BH 工作队列**（§3）或线程化 IRQ。
   tasklet 没被删（大量旧代码还用），但**当遗留机制对待**。

**结论**：读到 tasklet 要认识（旧代码遍地是），但**自己写新代码用 §3 的 BH 工作队列**。

---

## 3. 逐句拆解工作队列（中断消费者视角）

### 晦涩点 ③：工作队列 = "能睡的延迟"

```c
INIT_WORK(&my_work, my_work_fn);
schedule_work(&my_work);     /* 把 my_work_fn 排队，稍后在 worker 线程里跑 */
```

工作队列把 `work_struct` 回调跑在**进程上下文**的 worker 线程里——**可以睡眠**
（mutex、阻塞分配、慢总线）。

原文明确划界：**工作队列内部那套**（worker 池、`max_active`、`WQ_UNBOUND`、并发管理）
是独立的大子系统，**本章不展开**。从**中断**的角度只需记住：当延迟的活**要睡眠、但
不值得专门开一个线程化 IRQ 的 kthread**（Topic 07），或者只是偶发的后续跟进，
就用工作队列。

### 晦涩点 ④：BH 工作队列 —— tasklet 的官方接班人

`include/linux/workqueue.h:372`：

```c
WQ_BH = 1 << 0,    /* 让 work 跑在 bottom-half（软中断）上下文 */
extern struct workqueue_struct *system_bh_wq;        /* :473 */
extern struct workqueue_struct *system_bh_highpri_wq;
```

普通工作队列跑在进程上下文（能睡）。**BH 工作队列**特殊：它让 work **跑在软中断
上下文**——**不能睡**，但有和 tasklet 一样的时序（软中断时机）。于是：

```c
queue_work(system_bh_wq, &work);   /* 推荐的 tasklet_schedule 替代品 */
```

**画面**：tasklet 是一个老旧的、专用的小工具；BH 工作队列是"用成熟的瑞士军刀
（工作队列 API：flush/cancel/命名池）做同样的活"。**软中断上下文的延迟，新代码用它，
不用 tasklet。**

---

## 4. 逐句拆解 irq_work：唯一能从 NMI 排队的机制

### 晦涩点 ⑤：为什么需要一个"能从 NMI 里用"的机制？

前面所有机制（tasklet、工作队列）**排队动作本身**用了锁或可能睡眠的操作，
所以**不能从最受限的上下文（NMI、硬中断）里调用**。但有时你**就在 NMI 里**
（Topic 12），却需要安排一点后续工作——比如 perf 的 NMI 采样处理器需要唤醒
环形缓冲区的消费者，可这件事在 NMI 里干不了（会用到锁）。

**irq_work** 填这个空：它能从 **NMI 或硬中断上下文安全地排队**一个回调，让回调稍后
在**硬中断上下文**跑。秘诀是**无锁的 per-CPU 链表 + 自 IPI**（给自己 CPU 发一个
处理器间中断来触发执行，Topic 12）。

```c
struct irq_work { struct __call_single_node node; void (*func)(struct irq_work *); ...; };
bool irq_work_queue(struct irq_work *work);             /* 在本 CPU 尽快跑（自 IPI）*/
bool irq_work_queue_on(struct irq_work *work, int cpu); /* 在指定 CPU 跑（IPI）*/
```

### 晦涩点 ⑥：三个标志的区别

- **默认**：自 IPI，尽快在硬中断上下文跑。
- **`IRQ_WORK_LAZY`**：不立刻发 IPI，**等下一次定时器滴答**再跑（批处理、省开销）。
  适合不急的活。
- **`IRQ_WORK_HARD_IRQ`**：在 PREEMPT_RT 上**强制真正的硬中断上下文**（RT 上普通
  irq_work 可能被线程化）。

**谁用它**（原文）：**perf**（NMI→唤醒消费者）、**printk**（把控制台输出从 NMI/硬中断
里挪出来）、**RCU**、**调度器**。它是**从 NMI 升级到"普通硬中断工作"的唯一安全途径**。
Topic 12 重度依赖它。

---

## 5. 逐句拆解 irq_poll：NAPI 式轮询

### 晦涩点 ⑦：为什么"中断太多反而要关掉中断"？

回忆 Topic 00/02：中断的好处是"不用轮询、有事才打扰"。但当事件率**极高**时
（万兆网卡每秒上百万包），"每个包一次中断"的开销（进出中断、cache 抖动）**超过了
轮询的开销**。这时反而该**临时关掉中断、主动去轮询**。

**irq_poll** 是块/存储设备的通用 NAPI 机制（网络栈有自己的 NAPI）：

```c
void irq_poll_sched(struct irq_poll *);     /* top half 里调：切换到轮询模式 */
void irq_poll_complete(struct irq_poll *);  /* 队列排空了：重新开启中断 */
```

流程：第一个中断到来 → top half 调 `irq_poll_sched` **关掉设备中断** → 在
IRQ_POLL_SOFTIRQ 软中断里**带预算地轮询**完成项 → 队列排空 → `irq_poll_complete`
**重新武装中断**。

**画面**：门铃响个不停（高频中断），你干脆**先把门铃断电**，自己定期去门口收一批快递，
收完没人了再把门铃接上。**核心教训**：大规模下，**中断缓解（coalescing + 轮询）是中断
子系统的一部分，不是 hack**。

---

## 6. 逐句拆解决策矩阵（本章最实用的产出）

原文 §6 的矩阵建议**贴在墙上**。我把"快速规则"用决策树重写，更好记：

```
要睡眠吗？
├─ 是 → 是"服务这条线本身"的固有工作、要 RT 优先级？
│       ├─ 是 → 线程化 IRQ（Topic 07）
│       └─ 否（偶发跟进）→ 工作队列
└─ 否 → 从 NMI / 硬中断里排队的吗？
        ├─ 是 → irq_work（唯一选项）
        └─ 否 → 事件率极高想轮询？
                ├─ 是 → NAPI / irq_poll
                └─ 否 → 热点静态子系统（网/块/定时器）？
                        ├─ 是 → 真正的软中断向量（你加不了新的，那些是保留的）
                        └─ 否（普通软中断上下文延迟）→ BH 工作队列（别用 tasklet）
```

三条最常用的规则：
- **要睡 →** 线程化 IRQ 或工作队列。
- **从 NMI 排队 →** irq_work（没有别的选择）。
- **新的软中断上下文延迟 →** BH 工作队列，不是 tasklet。

---

## 7. 动手任务

### 任务 0：在 /proc/softirqs 和 ps 里找这些机制的痕迹（10 分钟，无需 root）

```bash
grep -E "TASKLET|HI:" /proc/softirqs    # tasklet 和 BH-wq 都计在这两列
ps -e | grep -E "kworker|ksoftirqd"     # 工作队列的 worker 和软中断线程
ls /sys/devices/virtual/workqueue/ 2>/dev/null | head
```

回答（答案 §8.0）：

1. TASKLET / HI 列有计数吗？说明你机器上还有代码在用 tasklet 或 BH-wq。
2. `kworker/*` 线程有多少个？它们是工作队列的什么角色？
3. `system_bh` 相关的工作队列在 `/sys/.../workqueue/` 里能找到吗？

### 任务 1：源码走读——tasklet 怎么坐在软中断上（25 分钟）

```
① include/linux/interrupt.h:692   tasklet_struct + DECLARE_TASKLET 宏
② include/linux/interrupt.h:758   tasklet_schedule：找到它调 raise_softirq 的地方
③ kernel/softirq.c:916            tasklet_action_common：找到三件事——
                                   遍历链表、tasklet_trylock（防自并发）、count 检查
④ include/linux/workqueue.h:372   WQ_BH 标志 + system_bh_wq 声明
⑤ kernel/irq_work.c:116           irq_work_queue：找到 LAZY 分支和自 IPI 触发
```

回答（答案 §8.1）：

1. `tasklet_schedule` 把 tasklet 挂链后 raise 的是哪个软中断向量？（联系 Topic 08 §2。）
2. `tasklet_trylock` 是怎么保证"同一 tasklet 不自并发"的？（看它操作哪个 state 位。）
3. `irq_work_queue` 在 `IRQ_WORK_LAZY` 和默认情况下触发方式有什么不同？

### 任务 2：写一个对比四种延迟机制上下文的模块（60 分钟）⭐

注册一个中断（用 Topic 04 骨架），在 top half 里同时触发四种延迟，各自打印上下文：

```c
static void my_tasklet_fn(struct tasklet_struct *t) {
    pr_info("TASKLET: in_serving_softirq=%lu, in_hardirq=%lu, can_sleep=NO\n",
            in_serving_softirq(), in_hardirq());
}
static void my_work_fn(struct work_struct *w) {
    pr_info("WORK: in_serving_softirq=%lu, pid=%d comm=%s (process ctx, can sleep)\n",
            in_serving_softirq(), current->pid, current->comm);
    msleep(1);   /* 合法！工作队列在进程上下文 */
}
static void my_bh_work_fn(struct work_struct *w) {
    pr_info("BH-WORK: in_serving_softirq=%lu (softirq ctx, NO sleep)\n",
            in_serving_softirq());
}
static void my_irq_work_fn(struct irq_work *w) {
    pr_info("IRQ_WORK: in_hardirq=%lu (hardirq ctx)\n", in_hardirq());
}
// top half 里：
//   tasklet_schedule(&my_tasklet);
//   schedule_work(&my_work);
//   queue_work(system_bh_wq, &my_bh_work);
//   irq_work_queue(&my_irq_work);
```

回答（答案 §8.2）：

1. 四个函数打印的 `in_serving_softirq` / `in_hardirq` / `current->comm` 各是什么？
   验证它们各自的运行上下文。
2. 只有 `my_work_fn` 里的 `msleep` 合法——把 msleep 加到另外三个里会怎样？
3. 哪个回调最先执行、哪个最后？（irq_work 自 IPI 最急、工作队列要等调度。）

### 任务 3：观察真实驱动用了哪种机制（20 分钟）

```bash
# 网络栈用 NAPI（软中断），看你网卡驱动：
grep -rln "napi_schedule\|irq_poll_sched\|tasklet_schedule\|queue_work\|irq_work_queue" \
     ~/repo/linux/drivers/net/ethernet/intel/ 2>/dev/null | head
# 看 perf 怎么用 irq_work（从 NMI 升级）：
grep -rn "irq_work_queue" ~/repo/linux/kernel/events/core.c | head
```

回答（答案 §8.3）：

1. Intel 网卡驱动主要用哪种延迟机制收包？为什么不是 tasklet 或工作队列？
2. `kernel/events/core.c`（perf）在哪里用 irq_work？联系 §4 晦涩点 ⑤"从 NMI 升级"。

### 任务 4：思考题（答案 §8.4）

1. tasklet 保证"不自并发"，BH 工作队列没有这个保证。那从 tasklet 迁移到 BH-wq 时，
   如果原代码**依赖**了这个并发保证，需要补什么？
2. 为什么 irq_work 能从 NMI 排队，而 tasklet/工作队列不能？它们排队时分别用了什么
   不能在 NMI 里用的东西？
3. `schedule_work` 不适合延迟敏感的路径，原文说"worker 调度延迟在高负载下无界"。
   为什么线程化 IRQ（也是进程上下文）就没这个问题？（提示：SCHED_FIFO vs
   普通 worker。）
4. irq_poll/NAPI 在高负载下关中断改轮询。如果忘了调 `irq_poll_complete` 会怎样？
   这种 bug 的表现是什么（联系 Topic 05 的 screaming 反面）？

---

## 8. 任务参考答案

### 8.0 任务 0

1. 多数机器 TASKLET/HI 有少量计数（仍有驱动在用）。2. `kworker/*` 是工作队列的
   **worker 线程**，从共享池里取 work 执行，数量随 CPU 数和负载变化。
3. 能看到 `events`、`events_highpri` 等；BH 类工作队列在较新内核也会列出。

### 8.1 任务 1

1. `tasklet_schedule` raise `TASKLET_SOFTIRQ`（`tasklet_hi_schedule` raise `HI_SOFTIRQ`）。
2. `tasklet_trylock` 用原子操作设置/检测 `TASKLET_STATE_RUN` 位：已置位说明别处正在跑，
   本次放回链表稍后再试——从而保证全局唯一实例。
3. 默认：立即发**自 IPI**让回调尽快在硬中断跑；LAZY：只挂链，等**下一次定时器滴答**
   顺带处理，不发 IPI（省中断开销）。

### 8.2 任务 2

1. tasklet：`in_serving_softirq=1, in_hardirq=0`；work：`in_serving_softirq=0`、
   comm=`kworker/...`（进程上下文）；bh-work：`in_serving_softirq=1`（软中断上下文）；
   irq_work：`in_hardirq=1`（硬中断上下文）。
2. 另外三个里 msleep 都会触发 "sleeping function called from invalid context" BUG
   ——它们都在原子上下文（软中断/硬中断）。
3. 通常 irq_work 最先（自 IPI 立即），tasklet/bh-work 在下次软中断（尾巴），
   工作队列最后（要等 worker 被调度）。

### 8.3 任务 3

1. NAPI（软中断 NET_RX，本质是 §5 的 irq_poll 思想用在网络）。因为收包是**超高频
   热点**，tasklet/工作队列的开销和延迟都太大，必须用带预算的软中断轮询。
2. perf 在 NMI 采样处理路径里调 `irq_work_queue` 唤醒 ring buffer 消费者——
   正是"NMI 里干不了的事，升级到硬中断 irq_work 去干"。

### 8.4 任务 4

1. 需要**自己补锁**（如 spinlock）保护原来靠"不自并发"保护的数据，因为 BH-wq 的多个
   实例可能在不同 CPU 并发跑同一回调。这也是为什么迁移时要审视原代码是否真依赖了
   该保证。
2. irq_work 用**无锁 per-CPU 链表 + 自 IPI**，全程不拿锁、不睡眠，NMI 安全。
   tasklet 排队也基本无锁但其软中断运行路径和某些状态操作不保证 NMI 安全；
   工作队列排队要操作池的锁、可能涉及唤醒——这些在 NMI 里都不安全。
3. 线程化 IRQ 是 **SCHED_FIFO 实时优先级**，优先于所有普通任务，高负载下也能及时
   被调度；普通 worker 是 SCHED_OTHER，和海量用户任务公平竞争，负载高时可能排队
   很久——所以延迟无界。
4. 设备中断会**一直保持禁用**（因为只有 `irq_poll_complete` 才重新武装它），表现为
   **设备静默停摆**（不再产生中断、I/O 卡死）。这是 screaming 的反面——不是中断太多，
   而是中断被关掉再没打开，同样难查。

---

## 9. 学完自测

1. 五个延迟去处各跑在什么上下文、能否睡眠、建在什么之上？
2. tasklet 比裸软中断多给什么保证？为什么新代码仍不推荐它、该用什么替代？
3. irq_work 凭什么能从 NMI 排队？谁是它的典型用户？
4. irq_poll/NAPI 解决什么问题？"高频时关中断改轮询"的道理是什么？
5. 默写决策树：要睡？从 NMI 排队？高频？分别导向哪个机制？

下一章 Topic 10：把 Topic 05~09 所有机制**串起来的并发规则**——上下文、屏蔽、
原子性、preempt_count，让这些机制互不打架的底层法则。
