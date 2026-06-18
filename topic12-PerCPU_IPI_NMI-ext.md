# Topic 12 扩展版 — 每 CPU 中断、IPI、NMI：三种"不走寻常路"的投递

> **用法同前**。行号基于 `~/repo/linux`（v7.1-rc7）：`request_nmi` 在
> `kernel/irq/manage.c:2271`、`request_percpu_nmi` 在 `manage.c:2584`、
> `irq_reserve_ipi` 在 `kernel/irq/ipi.c:23`、`__ipi_send_single` 在 `ipi.c:227`；
> x86 IPI 向量在 `arch/x86/include/asm/irq_vectors.h:63`（`RESCHEDULE_VECTOR=0xfd`
> 等）；`register_nmi_handler` 在 `arch/x86/include/asm/nmi.h:78`。
>
> **承接**：Topic 11 讲了 MSI 这种"非电线"投递。本章讲其余三种都不符合"设备线 →
> 一个 CPU"模型的投递。前置：Topic 05（`handle_percpu_devid_irq`、`handle_fasteoi_nmi`）、
> Topic 09（irq_work）、Topic 10（NMI/原子规则那一行）。

---

## 1. 先建立画面感：三种"破例"的中断

到目前为止，中断模型都是"某个设备拉一条线 → 控制器 → 一个 CPU"。但有三种中断
**各自打破一个假设**：

| 模式 | 谁发起 | 发给谁 | 打破的假设 |
|---|---|---|---|
| **每 CPU 中断** | 每个 CPU 私有的线（定时器、PMU）| 只发给本 CPU | 中断不是全局的（亲和性无意义）|
| **IPI** | **另一个 CPU**（软件） | 一个/多个 CPU | 来源不是设备，是对等 CPU 上的软件 |
| **NMI** | 特殊线/perf/看门狗 | 一个 CPU，**不可屏蔽** | `local_irq_disable` 关不掉它 |

**画面**：
- **每 CPU 中断**像"每个工位自带的桌面闹钟"——只吵自己，谈不上"派给谁"。
- **IPI**像"同事之间互相发的内部短信"——不是外部访客，是同事让你立刻做件事。
- **NMI**像"全楼强制火警"——你戴着降噪耳机（关中断）也必须听见。

三者都复用 Topic 01~10 的机器，但各弯折一个假设。

---

## 2. 逐句拆解每 CPU 中断

### 晦涩点 ①：为什么有些中断"每 CPU 一份"？

架构定时器（系统滴答）、PMU（性能计数器）、GIC 的 PPI（Topic 02 §3 的 16~31 号段）
**物理上每个 CPU 一个**。CPU0 的定时器和 CPU1 的定时器是**两个不同的硬件**，
号相同但实体不同（Topic 02 晦涩点 ⑤ 埋的伏笔）。

如果把它们当全局中断处理就全错了：

- **亲和性无意义**——你不能把"CPU0 的定时器"绑到 CPU3（它就是 CPU0 的）。
- **dev_id 和使能状态必须每 CPU 独立**——每个 CPU 各自启用/禁用自己那份。

### 晦涩点 ②：API 与普通 `request_irq` 的三个关键区别

```c
int request_percpu_irq(unsigned int irq, irq_handler_t handler,
                       const char *devname, void __percpu *percpu_dev_id);
void enable_percpu_irq(unsigned int irq, unsigned int type);  /* ★必须在每个 CPU 上各调一次 */
```

1. **`dev_id` 是 `void __percpu *`**——每个 CPU 看到自己那份 cookie，handler 拿到的是
   **本地 CPU** 的实例。（回忆 Topic 03 的 percpu_dev_id union。）
2. **使能是每 CPU 的**：一次 `request_percpu_irq` 只注册了 action，但
   `enable_percpu_irq` **必须在每个想接收的 CPU 上分别调用**（通常从 CPU 热插拔
   启动回调里调，Topic 13/17）。这是原文 Pitfalls 第一条的来源——只在一个 CPU 上
   enable，结果只有那个 CPU 收得到。
3. **flow handler 是 `handle_percpu_devid_irq`**（Topic 05 §2.4）：**不拿 `desc->lock`**
   ——因为每 CPU 线没有跨 CPU 竞争，省掉锁，更快。

这些线带 `IRQF_PERCPU` 语义，**豁免强制线程化**（Topic 07）和**亲和性/均衡**（Topic 13）。

---

## 3. 逐句拆解 IPI

### 晦涩点 ③：IPI 是"CPU 对 CPU 说：立刻干这件事"

**IPI（Inter-Processor Interrupt，处理器间中断）** 是一个 CPU 通知另一个 CPU 立即做事。
它支撑着这些关键操作：

- **TLB shootdown**（地址映射改了，让其他 CPU 刷新它们缓存的页表项）；
- **重新调度**（CPU A 发现该让 CPU B 跑别的任务）；
- **在远程 CPU 上跑函数**（`smp_call_function`）；
- 定时器广播、irq_work 的跨 CPU 投递（Topic 09）。

### 晦涩点 ④：两层结构——genirq IPI 层 vs 高层用户

**底层**（`kernel/irq/ipi.c`）：把 IPI 当作 IRQ 暴露的控制器（主要是 GIC 的 SGI）注册它们，
核心提供 `__ipi_send_single`（`ipi.c:227`）等，最终调芯片的 `ipi_send_single`
（Topic 03 的回调）——arm64 上就是往 GIC 的 **SGI 寄存器**写、目标是那些 CPU。
接收 CPU 像收普通 IRQ 一样收下并跑 handler。

**高层**（`kernel/smp.c`）：**大多数内核代码不直接碰底层**，而是用：

```c
smp_call_function_single(cpu, func, info, wait);  /* 在某个远程 CPU 上跑 func() */
on_each_cpu(func, info, wait);                     /* 在所有 CPU 上跑 */
smp_send_reschedule(cpu);                           /* 戳一下某 CPU 的调度器 */
```

这些函数把 `func` 打包成一个工作项，触发一个**call-function IPI**，远程 CPU 的 IPI
handler 把排队的 `func` 跑掉。

**画面**：高层 API 像"给同事发条短信：'帮我做 X'"；底层 IPI 层像"短信底层的信号发射"。
你平时只发短信，不用管基站怎么发射。

### 晦涩点 ⑤：为什么 x86 和 arm64 的 /proc/interrupts 里 IPI 长得不一样？

这是 Topic 01 架构差异的又一次显形：

- **x86**：IPI 用**固定的高位向量**（`irq_vectors.h:63`）：`RESCHEDULE_VECTOR=0xfd`、
  `CALL_FUNCTION_VECTOR=0xfc`、`CALL_FUNCTION_SINGLE_VECTOR=0xfb`……发送方写自己的
  Local APIC **ICR** 寄存器（Topic 02 §2.2 的"寄信口"）。这些走
  **`DEFINE_IDTENTRY_SYSVEC`** 入口（Topic 01 §3.2），**不是** `vector_irq[]`/`irq_desc`
  设备中断——这正是 `common_interrupt` 注释里说"特殊 SMP 中断有自己的入口"的原因。
- **arm64**：IPI 用 GIC 的 **SGI**（0~15 号），经 GIC 走正常的 `gic_handle_irq` 路径
  → 一个 IPI 的 `irq_desc`。

所以 /proc/interrupts 里 x86 把 IPI 显示成底部一组**有名字的系统中断**
（"Rescheduling interrupts"等），arm64 显示成 **SGI 行**。

---

## 4. 逐句拆解 NMI（全章最需要敬畏的部分）

### 晦涩点 ⑥：NMI 凭什么"不可屏蔽"，为什么需要它

**NMI（Non-Maskable Interrupt）** 不能被 `local_irq_disable()` 挡住——**即使在关中断的
临界区里它也会触发**。这让它成为"**监视一个卡死或关着中断的 CPU**"的唯一工具，
但代价极其苛刻。

用途（原文 §4.1）：

- **硬死锁检测/看门狗**：NMI 周期性检查 CPU 还在前进。普通中断查不了——如果 CPU 卡在
  关中断的死循环里，普通中断根本进不去，只有 NMI 能。
- **perf 采样**：常通过 NMI 投递，这样能采样到**关中断的代码**（否则那段代码永远
  采不到）。
- `nmi_panic`/KGDB/串口魔术键——调试死掉的系统。

### 晦涩点 ⑦：两条注册路径，千万别搞混

原文反复强调这是个混淆点：

1. **x86 架构 NMI**（`arch/x86/include/asm/nmi.h:78`）：x86 的 NMI **不是普通 IRQ 线**，
   有自己的注册表：
   ```c
   register_nmi_handler(NMI_LOCAL, handler, flags, name);  /* 不是 irq_desc！ */
   ```
   handler 按类型（NMI_LOCAL/NMI_UNKNOWN/...）串成链，`arch/x86/kernel/nmi.c` 派发，
   返回 `NMI_HANDLED`/`NMI_DONE`。
2. **genirq `request_nmi`**（`manage.c:2271`，Topic 06）：给那些**能把普通 IRQ 线当 NMI
   投递**的控制器用——主要是 **arm64 伪 NMI**（GIC 优先级，`CONFIG_ARM64_PSEUDO_NMI`，
   Topic 01/02 晦涩句 ⑯）。芯片实现 `irq_nmi_setup`/`irq_nmi_teardown`，flow handler 是
   `handle_fasteoi_nmi`（Topic 05 §2.4）。

**记忆**：x86 NMI 走自己的小注册表（不是 irq_desc）；arm64 伪 NMI 走 genirq 的
`request_nmi`（是 irq_desc）。两个不同的子系统。

### 晦涩点 ⑧：NMI 里的铁律（Topic 10 那一行硬约束的展开）

NMI handler 内部：

- **不能睡、不能用普通锁**。为什么？因为 NMI 可能在**本 CPU 上某段代码正持有一把普通
  spinlock 时**打断它。如果 NMI handler 去拿同一把锁 → 它自旋等一个**永远回不去放锁**
  的持有者（持有者被 NMI 打断了）→ **瞬间死锁**（Topic 10 任务 4 问题 4 的同款）。
  只能用**无锁算法**或 `raw_spin_trylock`（拿不到就放弃）。
- **只能用 NMI 安全的原语**：NMI 安全的 RCU（`rcu_nmi_enter`，Topic 01 §5/Topic 10 §5）、
  `this_cpu_*`、`cmpxchg`。`printk` 被特殊处理成 NMI 安全（延迟输出），其余大多不安全。
- **"升级，别干活"**：标准模式（perf、printk 都这么做）是——在 NMI 里做**最小限度的
  NMI 安全捕获**，然后 `irq_work_queue()`（Topic 09 §4）把剩下的活推到**普通硬中断
  上下文**完成。**这就是 irq_work 存在的根本原因。**

**画面**：NMI 像闯进手术室的火警——你能做的极少（不能碰任何可能正被别人用着的工具），
唯一安全的动作是"按下警报转给外面的人处理"（irq_work 升级出去）。

---

## 5. 动手任务（针对你的 x86_64 机器）

### 任务 0：在 /proc/interrupts 底部认出三种特殊中断（10 分钟，无需 root）

```bash
cat /proc/interrupts | tail -25
```

回答（答案 §6.0）：

1. 找到 `LOC`（每 CPU 本地定时器）——它是哪种模式（§1 三种之一）？为什么每个 CPU
   列都在涨？
2. 找到 `RES`（重新调度）、`CAL`（function call）、`TLB`——它们是哪种模式？
   是谁发给谁的？
3. 找到 `NMI` 行——你机器上 NMI 计数是 0 还是在涨？涨说明什么在用 NMI
   （看门狗或 perf）？

### 任务 1：实测 IPI——用 perf 或 taskset 制造跨 CPU 活动（20 分钟，需要 root）

```bash
# 看 IPI 计数基线：
grep -E "RES|CAL|TLB" /proc/interrupts
# 制造重新调度 IPI：把一个忙进程在 CPU 间反复迁移
taskset -c 0 yes >/dev/null & PID=$!
sleep 1; for c in 1 2 3 0; do taskset -cp $c $PID; sleep 0.3; done
# 制造 TLB shootdown：大量 mmap/munmap（多线程）
grep -E "RES|CAL|TLB" /proc/interrupts   # 再看，哪个涨了
kill $PID
```

回答（答案 §6.1）：

1. 迁移进程后 `RES`（reschedule IPI）涨了吗？这印证 §3 的什么用途？
2. 用 `perf bench` 或多线程程序制造大量内存映射变更，`TLB` 行涨吗？为什么改页表
   要发 IPI 给别的 CPU？
3. `CAL`（function call）涨说明什么内核操作在跨 CPU 调函数？

### 任务 2：源码走读三条路径（35 分钟）

```
① kernel/irq/manage.c:2526   request_percpu_irq_affinity：看 percpu_dev_id 的处理
② kernel/irq/ipi.c:227       __ipi_send_single：找到它最终调 chip->ipi_send_single
③ kernel/smp.c               smp_call_function_single：看它怎么打包 call_single_data
                             并触发 IPI（搜 send_call_function_single_ipi）
④ arch/x86/include/asm/irq_vectors.h:63  确认三个 IPI 向量号
⑤ arch/x86/kernel/nmi.c      看 NMI 派发：default_do_nmi / nmi_handle，
                             handler 链怎么遍历
```

回答（答案 §6.2）：

1. `__ipi_send_single` 和发送普通中断有什么本质不同？（提示：它不经过设备，
   直接调芯片的发送回调。）
2. x86 的 reschedule IPI 走 `DEFINE_IDTENTRY_SYSVEC` 还是 `common_interrupt`？
   为什么它"有自己的入口"（Topic 01 §3.2）？
3. `arch/x86/kernel/nmi.c` 里 NMI handler 是怎么组织的（链表？按类型？）？
   返回 `NMI_HANDLED` 和 `NMI_DONE` 的区别？

### 任务 3：观察 NMI 看门狗（20 分钟，需要 root）

```bash
cat /proc/sys/kernel/nmi_watchdog          # 1 = 开启
cat /proc/sys/kernel/watchdog_thresh        # 阈值秒数
grep NMI /proc/interrupts                    # NMI 计数
# 临时关闭再开启，观察 NMI 计数变化：
echo 0 | sudo tee /proc/sys/kernel/nmi_watchdog; sleep 2; grep NMI /proc/interrupts
echo 1 | sudo tee /proc/sys/kernel/nmi_watchdog; sleep 5; grep NMI /proc/interrupts
```

回答（答案 §6.3）：

1. 关闭 nmi_watchdog 后 NMI 计数停止增长了吗？说明看门狗确实在用 NMI 周期性触发。
2. 看门狗为什么必须用 NMI 而不是普通定时器中断来检测 CPU 卡死？
   （回忆 §4：卡死的 CPU 关着中断时，普通中断进不去。）
3. （了解即可，别真做）如果一个 CPU 真的硬死锁，NMI 看门狗检测到后会打印什么、
   可能 panic 吗？（看 `hardlockup_panic`。）

### 任务 4：思考题（答案 §6.4）

1. 为什么每 CPU 中断的 flow handler（`handle_percpu_devid_irq`）可以不拿
   `desc->lock`，而普通中断必须拿？
2. `smp_call_function_single(cpu, func, info, wait=1)` 里 wait=1 会让**发送方**阻塞
   等待。如果 func 在远程 CPU 上跑得很慢会怎样？这对"IPI handler 要短"的要求
   意味着什么？
3. 一段数据被普通进程代码和一个 NMI handler 共享。用 `spin_lock_irqsave` 保护够吗？
   为什么不够？应该怎么做？
4. perf 的 NMI 采样处理器为什么不能直接在 NMI 里唤醒用户态的 perf 工具，
   而要先 `irq_work_queue`？irq_work 跑在什么上下文，比 NMI 宽松在哪？

---

## 6. 任务参考答案

### 6.0 任务 0

1. `LOC` 是**每 CPU 中断**（每个 CPU 的本地 APIC 定时器）。每列都涨因为每个 CPU
   都有自己的定时器在产生滴答。
2. `RES`/`CAL`/`TLB` 都是 **IPI**——CPU 发给 CPU。RES=重新调度、CAL=远程函数调用、
   TLB=TLB shootdown（页表变更通知）。
3. NMI 计数涨通常意味着 nmi_watchdog 或 perf 在用 NMI；为 0 说明都没启用或用的
   是非 NMI 路径。

### 6.1 任务 1

1. 涨。进程在 CPU 间迁移触发 reschedule IPI，印证"CPU A 让 CPU B 跑别的任务"。
2. 涨。改页表（munmap）后，其他 CPU 的 TLB 里可能缓存了**旧的**映射，必须发 IPI
   让它们刷新（shootdown），否则它们用陈旧映射访问已释放的页 → 内存损坏。
3. 内核在跨 CPU 同步某些状态（如刷某 CPU 的缓存、读远程 CPU 的寄存器）时用
   `smp_call_function` 在目标 CPU 上跑函数。

### 6.2 任务 2

1. `__ipi_send_single` 不经过任何设备/物理线，直接调芯片的 `ipi_send_single` 让
   硬件（GIC SGI / APIC ICR）往目标 CPU 投递——**软件主动发起**，这是 IPI 的本质。
2. 走 `DEFINE_IDTENTRY_SYSVEC`。因为 IPI 向量是固定的系统向量，不走 `vector_irq[]`
   设备查表路径——它们语义固定（重新调度就是重新调度），不需要 irq_desc 那套
   通用机制，直接专用入口更快。
3. 按 NMI 类型（NMI_LOCAL 等）组织成 handler 链，`nmi_handle` 遍历链逐个调用。
   `NMI_HANDLED`="这个 NMI 是我处理的"，`NMI_DONE`="不是我的"——类似普通中断的
   HANDLED/NONE，用于多个 NMI 源共享时判定归属。

### 6.3 任务 3

1. 是，关闭后 NMI 计数停止增长（或大幅放缓），证明看门狗周期性产生 NMI。
2. 卡死的 CPU 往往是**关着中断**死循环或死锁。普通定时器中断被屏蔽进不去，
   无法检测。NMI 不可屏蔽，是唯一能"敲进"这种 CPU 的手段。
3. 打印 "NMI watchdog: Watchdog detected hard LOCKUP on cpu N" + 调用栈；
   若 `hardlockup_panic=1` 则直接 panic（生产环境常这么配，便于拿到 crash dump）。

### 6.4 任务 4

1. 每 CPU 线只在它所属的那个 CPU 上触发，**不存在跨 CPU 并发访问同一个 desc**，
   所以无需锁保护。普通中断可能在任意 CPU 上触发、亲和性可变、还可能与
   set_affinity 等操作并发，必须用 `desc->lock` 串行化。
2. 发送方会一直阻塞到远程 func 跑完。func 慢 → 发送方 CPU 也跟着干等，两个 CPU
   都被拖住。所以 IPI handler/`smp_call_function` 的回调**必须极短**，
   wait=1 时尤其——它把"远程慢"传染给了本地。
3. **不够**。`local_irq_disable`/`irqsave` 关不掉 NMI（不可屏蔽），NMI 仍可能在你
   持锁时插入。如果 NMI 也碰这数据，普通锁会死锁。正确做法：用 **NMI 安全的无锁
   结构**（per-CPU + cmpxchg），或在 NMI 侧只用 `raw_spin_trylock`，并尽量
   通过 irq_work 把对共享数据的操作移出 NMI。
4. 唤醒用户态涉及调度器/等待队列操作，这些用锁、可能调度——在 NMI 里不安全。
   `irq_work` 跑在**普通硬中断上下文**：虽然也不能睡，但可以拿普通锁、做唤醒
   （它不会像 NMI 那样打断一个正持锁的对等上下文，因为它经由自 IPI 在中断退出
   的正常时机运行）。所以是从"几乎啥都不能干"安全升级到"普通硬中断能干的事"。

---

## 7. 学完自测

1. 三种特殊投递各打破"设备线→一个 CPU"模型的哪个假设？各举一个真实例子。
2. 每 CPU 中断的 API 和普通 request_irq 的三个区别？为什么 enable 要逐 CPU 调？
3. IPI 的两层结构是什么？x86 和 arm64 的 IPI 在 /proc/interrupts 里为什么长得不同？
4. NMI 的两条注册路径分别给谁用？NMI 里为什么连普通自旋锁都不能用？
5. "在 NMI 里捕获最小信息，然后 irq_work 升级"——为什么这是 NMI 的标准模式？

下一章 Topic 13：中断落在哪个 CPU、谁决定、CPU 上下线时怎么办——亲和性、均衡、
热插拔与矩阵分配器。
