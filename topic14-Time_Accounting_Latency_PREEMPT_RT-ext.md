# Topic 14 扩展版 — 时间计量、延迟与 PREEMPT_RT

> **用法同前**。行号基于 `~/repo/linux`（v7.1-rc7）：`irqtime_account_irq` 在
> `kernel/sched/cputime.c:57`、handler 时长检查在 `kernel/irq/handle.c:147` 起
> （`irqhandler.duration_warn_us=` 启动参数在 `handle.c:171`）、irqsoff tracer 在
> `kernel/trace/trace_irqsoff.c`。
>
> **承接**：前面所有章节讲的是中断怎么工作、放在哪。本章讲它们的**时间后果**——
> 中断花了多少时间、延迟从哪来、怎么测、怎么调，以及把 Topic 07/08/10 零散介绍的
> PREEMPT_RT 拼成完整图景。前置：Topic 07（线程化）、Topic 08（软中断/ksoftirqd）、
> Topic 10（关中断区间）。

---

## 1. 先建立画面感：为什么要单独计量"中断时间"

### 1.1 默认计时为什么会"算错账"

内核传统上靠**定时器滴答**来统计 CPU 时间："滴答响的那一刻，CPU 在干嘛，就把刚过去
的那一小段算给谁。" 问题：中断处理是**借用某个倒霉任务的执行流**跑的（Topic 00）。
如果滴答恰好落在中断处理中，这段中断时间就被**错记到那个被打断的任务头上**——
明明是网卡在狂收包，账却记在你的编译进程上。

**画面**：合租房水电费按"抄表那一刻谁在用"来分摊——但室友 A 半夜偷偷开了大功率设备
（中断），抄表时却记在恰好醒着的室友 B 头上。不公平，也看不出真凶。

### 晦涩点 ①：`CONFIG_IRQ_TIME_ACCOUNTING` 精确计量

开启这个配置后，内核在**每个硬中断/软中断的进出边界**读一次高精度时钟
（`sched_clock`），把这段时间精确归到 **hardirq** 或 **softirq** 桶里，
**从被打断任务的运行时间里扣掉**（`kernel/sched/cputime.c:57`）：

```c
void irqtime_account_irq(struct task_struct *curr, unsigned int offset)  /* 每次进出上下文调用 */
```

结果暴露在 **`/proc/stat`** 的每 CPU `irq`/`softirq` 列，`top`/`mpstat` 显示成
`%irq`/`%soft`。**没有它**，重中断负载会被错算成虚高的 "system" 甚至 "user" 时间；
**有了它**，你能直接看到"某个核 30% 时间花在 softirq"（NET_RX 风暴）。

由 Topic 01 的通用入口层驱动进出钩子（`vtime_account_hardirq/softirq`），所以跨架构一致。

---

## 2. 逐句拆解延迟从哪来（本章核心）

### 晦涩点 ②："中断延迟"有两个含义，都重要

很多人说"中断延迟"时含糊，其实是两件事：

**含义 1：处理一个中断要多久（设备拉线 → handler 开跑）。** 被什么拖长：

- **关中断区间**（Topic 10）：**任何** `local_irq_disable`/`spin_lock_irqsave` 区间都会
  **推迟该 CPU 上所有中断**直到它结束。**这是头号元凶。**
- **过长的 top half**（Topic 06）：一个循环的 handler 关着中断不放，把排在后面的所有
  中断都堵住。
- **更高优先级中断/软中断风暴**霸占 CPU。
- handler 要拿的 `spin_lock_irqsave` 的**跨 CPU 争用**。

**含义 2：延迟的活儿要多久才跑（handler → 软中断/线程完成）。** 被什么拖长：

- **软中断预算耗尽 → ksoftirqd** 的调度延迟（Topic 08 §5）。
- **线程化 IRQ 的唤醒到运行延迟**（SCHED_FIFO 但仍要被调度，Topic 07 §4）。
- 关抢占区间推迟调度器。

**画面**：含义 1 是"救护车到现场要多久"（被路上的关中断区间堵车），含义 2 是
"到了现场后处理伤员要多久排上队"（被 ksoftirqd 调度排队拖延）。两段都算"响应慢"。

### 晦涩点 ③：内置的 handler 超时告警

`kernel/irq/handle.c:147` 起有个可选检查，专抓**过长的 top half**：

```c
__setup("irqhandler.duration_warn_us=", irqhandler_duration_check_setup);  /* :171 启动参数 */
/* __handle_irq_event_percpu 里给每个 handler 计时，超阈值就告警 */
```

启动加 `irqhandler.duration_warn_us=200`，任何 handler 跑超过 200 微秒就打印告警
（带罪魁函数名 `%pS`）。**零配置抓"慢 handler"的利器**，生产环境可常开一个宽松阈值
防回归。注意它用 static key（Topic 07 晦涩点 ⑧），不开时零开销。

---

## 3. 逐句拆解延迟测量工具

原文 §3 的工具表，补充"它到底测什么、什么时候用"：

### 晦涩点 ④：`irqsoff` tracer —— "谁把中断关了 800 微秒"

最经典的工具。它记录**最长的一段连续关中断区间**及**导致它的代码路径**。
当你怀疑"有人长时间关中断害我延迟"，就用它：

```bash
echo irqsoff > /sys/kernel/tracing/current_tracer
# 跑一会儿负载，然后：
cat /sys/kernel/tracing/trace        # 看最长关中断区间的调用栈
cat /sys/kernel/tracing/tracing_max_latency   # 最大值（微秒）
```

它直接把 §2 含义 1 的头号元凶（关中断区间）的**位置和栈**揪出来。
`preemptoff`/`preemptirqsoff` 是它的变体（测关抢占/二者合并）。

### 晦涩点 ⑤：`cyclictest` —— 端到端的"唤醒能晚多少"

`cyclictest`（rt-tests 包）是 RT 调优的**黄金标准测量**：它睡一个精确的时长，
测"实际被唤醒的时刻 vs 应该被唤醒的时刻"差多少——这就是应用能感受到的最坏延迟。
RT 内核调好后，cyclictest 最坏值能压到**两位数微秒**。

工具速记：**`irqsoff` 找"谁关了中断"（原因），`cyclictest` 测"唤醒晚了多少"（结果）。**

---

## 4. 逐句拆解中断风暴

### 晦涩点 ⑥：三种风暴，三种应对

**风暴 = 中断来的速度远超能有效处理的速度，把 CPU 饿死。** 三类（呼应前面章节）：

1. **尖叫电平中断**（线卡在高位，常因触发方式配错，Topic 02/05）：genirq 的假中断侦探
   在约 10 万次未处理后用 `irq N: nobody cared` 禁用它（Topic 05 §5）——**安全网**。
2. **高速但合法的中断**（网卡遭遇包风暴）：不是假中断，但每个中断的开销超过收益。
   解法是**中断缓解**：硬件合并（设备攒一批再中断）+ **轮询**（网络用 NAPI、块用
   irq_poll，Topic 09 §5）——高负载下从"每事件一中断"切到"带预算轮询"。
   **症状：ksoftirqd 100%**（Topic 08 §9）。
3. **IPI 风暴**（Topic 12）：过多 TLB shootdown/`smp_call_function`，表现为
   "Function call interrupts" 高——通常是内存管理或扩展性问题，不是中断层的锅。

**画面**：① 是门铃坏了一直响（禁掉它）；② 是真的来了太多快递（改成定时批量取件=轮询）；
③ 是同事之间内部消息刷屏（去查为什么消息这么多，不是消息系统的错）。

---

## 5. 逐句拆解 PREEMPT_RT 完整模型

### 晦涩点 ⑦：把前面零散的 RT 改动拼成一张图

RT 的目标：**让内核几乎处处可抢占**，从而把高优先级任务的唤醒延迟**有界化**。
前面分散讲过的改动，在这里合体：

| 机制 | 普通内核 | PREEMPT_RT | 来源 |
|---|---|---|---|
| IRQ handler | 多数跑硬中断 | **几乎全部线程化**（SCHED_FIFO 的 irq/N 线程，可抢占）| Topic 07 §5 |
| 软中断 | 硬中断尾巴上原子运行 | **任务上下文**，可被高优先级 RT 任务抢占 | Topic 08 §6 |
| `spin_lock` | 真自旋 | **可睡眠的 RT mutex（带优先级继承）** | Topic 10 §7 |
| `raw_spinlock` | 真自旋 | **仍真自旋**（所以 desc->lock 是 raw）| Topic 10 |
| `local_bh_disable` | 加 preempt_count | **每 CPU 睡眠锁** | Topic 08 |

**幸存者**（RT 上仍真正原子/硬中断）：`IRQF_NO_THREAD`（定时器）、每 CPU 中断、
`IRQ_WORK_HARD_IRQ`、少数标了 no-thread 的线、以及 `raw_spinlock` 路径。

**净效果**（原文）：最长不可抢占区间从"一个长 handler + 软中断风暴"缩到
"几个 raw_spinlock/硬中断专属路径"——这才让 cyclictest 能跑出两位数微秒最坏值。

### 晦涩点 ⑧：优先级继承——RT 的关键拼图

`spin_lock` 变成**带优先级继承（priority inheritance）**的 RT mutex 是什么意思？
场景：低优先级任务 L 拿着锁，高优先级任务 H 要这把锁被迫等待——这叫**优先级反转**。
如果此时中等优先级任务 M 抢占了 L，L 跑不了就放不了锁，H 就被 M 间接卡住了
（**无界优先级反转**，火星探测器 Pathfinder 著名 bug）。

**优先级继承**：H 等 L 的锁时，**临时把 L 提升到 H 的优先级**，让 L 尽快跑完放锁。
这把"无界反转"变成"有界"。这是 RT 内核能保证延迟上界的核心机制之一。

---

## 6. 逐句拆解调优旋钮

原文 §6 的旋钮，按"想达到什么目的"重组：

**目的 A：让中断处理可调度、可调优先级** → `threadirqs`（强制线程化，Topic 07）+
`chrt` 调 `irq/N` 线程的 RT 优先级。

**目的 B：保护实时核不被打扰** →
- `isolcpus=`/`nohz_full=`/`rcu_nocbs=` 把 tick/RCU/调度器赶出选定核；
- **配合 IRQ 亲和性**（Topic 13：把中断 pin **离开**隔离核 + 关 irqbalance）。
- ⚠️ 原文 Pitfalls：**只隔离不 pin 中断 = 半拉子**（中断仍堆 CPU0 或乱跑）。两者要一起做。

**目的 C：减少中断数量（吞吐优先）** → 中断合并（`ethtool -C` 网卡、设备特定存储）。
⚠️ 合并太狠会**增加单事件延迟**——延迟敏感负载不要过度合并。

**目的 D：定位/接近最优** → `irqhandler.duration_warn_us=` 常开抓回归；
NUMA 局部性（中断放在消费其数据的线程同节点）。

---

## 7. 动手任务（针对你的 x86_64 机器）

### 任务 0：用 mpstat 看中断时间占比（10 分钟，需要 sysstat 包）

```bash
mpstat -P ALL 1 3        # 看 %irq 和 %soft 列
# 制造网络负载再看：
( curl -s -o /dev/null http://speedtest.tele2.net/100MB.zip & ) ; mpstat -P ALL 1 5
cat /proc/stat | grep -E "^cpu[0-9]" | head   # irq/softirq 列（第 7、8 个数字）
```

回答（答案 §8.0）：

1. 空闲时 `%irq`/`%soft` 接近 0 吗？制造网络流量后，哪个核的 `%soft` 上去了？
   （联系 Topic 08 NET_RX。）
2. 你的内核开了 `CONFIG_IRQ_TIME_ACCOUNTING` 吗？
   （`grep IRQ_TIME_ACCOUNTING /boot/config-$(uname -r)`）没开的话 mpstat 的 %irq
   还准吗？
3. `%soft` 高的那个核，是不是也是网卡中断亲和性指向的核（Topic 13）？

### 任务 1：用 irqsoff tracer 抓最长关中断区间（25 分钟，需要 root）⭐

```bash
sudo -i; cd /sys/kernel/tracing
cat available_tracers | tr ' ' '\n' | grep -E "irqsoff|preempt"   # 确认有这些 tracer
echo 0 > tracing_on
echo irqsoff > current_tracer
echo 0 > tracing_max_latency
echo 1 > tracing_on
sleep 10                          # 跑点负载（编译、IO）
echo 0 > tracing_on
cat tracing_max_latency           # 最长关中断区间（微秒）
head -40 trace                    # 谁干的、调用栈
echo nop > current_tracer
```

回答（答案 §8.1）：

1. 最长关中断区间多少微秒？trace 里栈顶的函数是谁？是某个 `spin_lock_irqsave`
   还是某段 `local_irq_disable`？
2. 这段区间发生时，该 CPU 上**所有**其他中断都被推迟了——这印证 §2 含义 1 的头号元凶。
3. 换成 `preemptoff` tracer 再测一次，最长关抢占区间和关中断区间一样长吗？为什么可能不同？

### 任务 2：用 duration_warn 抓慢 handler（20 分钟，需要 root + 改启动参数）

```bash
# 临时改 GRUB 加 irqhandler.duration_warn_us=100，重启后：
cat /proc/cmdline | grep -o "irqhandler.duration_warn_us=[0-9]*"
dmesg | grep -i "irq handler\|duration" | head    # 看有没有慢 handler 告警
```

回答（答案 §8.2）：

1. 设成 100us 后，dmesg 里有 handler 超时告警吗？是哪个驱动的 handler？
2. 如果一个都没有，把阈值降到 20us 重启再看——现在抓到了吗？这说明你机器上
   handler 普遍多快？
3. 这个机制用 static key（`handle.c:147`），不开时零开销——为什么内核不默认开着？

### 任务 3：跑 cyclictest 测唤醒延迟（30 分钟，需要 root + rt-tests）⭐

```bash
sudo apt install rt-tests 2>/dev/null || echo "装一下 rt-tests"
# 测基线唤醒延迟（普通内核）：
sudo cyclictest -l 100000 -m -S -p 90 -i 200 -h 400 -q
# -S 每 CPU 一个线程, -p 90 RT优先级, 输出 Min/Avg/Max
# 同时制造干扰，再测一次对比：
sudo sh -c 'stress-ng --cpu 4 --io 2 & sleep 1; cyclictest -l 100000 -m -p 90 -q; kill %1' 2>/dev/null
```

回答（答案 §8.3）：

1. 空闲时的 Max 延迟是多少微秒？加干扰负载后 Max 涨到多少？
2. 你跑的是普通内核还是 PREEMPT_RT？普通内核的 Max 通常在什么量级
   （几十 us？几百 us？毫秒级？）？
3. 如果有 RT 内核可对比：同样干扰下 RT 的 Max 比普通内核低多少？这印证 §5 哪句话？

### 任务 4：思考题（答案 §8.4）

1. 为什么"关中断区间"比"长 handler"更隐蔽、更难查？（提示：handler 有 duration_warn
   抓，关中断区间藏在任意 `spin_lock_irqsave` 里。）
2. PREEMPT_RT 把 spin_lock 变成可睡眠锁，那为什么 `desc->lock` 必须保持 raw？
   如果它也变成可睡眠锁，第一个炸点在哪？（回忆 Topic 03/10。）
3. 中断合并（coalescing）减少中断数，降低 CPU 开销，但增加单事件延迟。
   对"高吞吐文件服务器"和"高频交易系统"，分别该怎么设？
4. 优先级继承解决无界优先级反转。如果一个 RT 内核**没有**优先级继承，
   一个低优先级任务持有 `desc->lock`（raw，不参与继承）被中优先级任务抢占，
   会发生什么？（这正是 raw_spinlock 区间必须极短的原因。）

---

## 8. 任务参考答案

### 8.0 任务 0

1. 空闲接近 0；网络流量下处理网卡中断的那个核 `%soft` 显著上升（NET_RX 软中断）。
2. 多数发行版**开了** IRQ_TIME_ACCOUNTING。没开的话 %irq 会把中断时间混入 system，
   mpstat 的 %irq 不准。
3. 通常是——网卡中断的 effective_affinity 指向哪个核，那个核就承担 NET_RX 软中断
   （除非开了 RPS/RFS 分散）。

### 8.1 任务 1

1. 视负载而定，常见几十到几百微秒；栈顶常是某个持 `spin_lock_irqsave` 的内核路径
   或一段显式 `local_irq_disable`。
2. 对——关中断期间该 CPU 收不到任何中断，全部推迟到区间结束，这是延迟的头号来源。
3. 可能不同。关抢占区间 ⊇ 关中断区间的某些部分，但 `preempt_disable` 不关中断、
   `local_irq_disable` 不一定关抢占（虽然通常连带），两者覆盖的代码段不完全重合。

### 8.2 任务 2

1. 取决于机器；可能抓到老旧驱动或虚拟化下的慢 handler。
2. 降到 20us 通常能抓到一些；说明现代 handler 大多在几us~几十us（top half 本就该短）。
3. 因为它给**每个中断**都加一次计时（即使 static key 开销小，热路径上仍有微小成本），
   且会刷日志；默认关闭，按需开启抓问题/防回归。

### 8.3 任务 3

1. 空闲 Max 常几十 us；加干扰后普通内核可能跳到几百 us 甚至毫秒级。
2. 普通 `PREEMPT`/`PREEMPT_VOLUNTARY` 内核干扰下 Max 通常几百 us 到 ms 级；
   `PREEMPT_NONE` 更差。
3. RT 内核同样干扰下 Max 常压到几十 us——印证"RT 把最长不可抢占区间缩到只剩
   raw_spinlock/硬中断路径"。

### 8.4 任务 4

1. 长 handler 有 `irqhandler.duration_warn_us` 直接点名罪魁；关中断区间可能藏在
   **任意一处** `spin_lock_irqsave`（包括第三方驱动、内核深处），没有"名字"，
   只能靠 irqsoff tracer 抓栈。分散、无主、靠工具反查——更隐蔽。
2. `desc->lock` 在**硬中断上下文**被拿（flow handler 进门即拿，Topic 05）。
   硬中断上下文不可睡眠（Topic 10）。若它变成可睡眠锁，第一次在硬中断里拿它就
   "sleeping function called from invalid context" BUG。所以必须 raw。
3. 文件服务器（吞吐优先）：**开大合并**（`ethtool -C` 调大 rx-usecs），用延迟换
   更少中断、更高吞吐。高频交易（延迟优先）：**关闭或最小化合并**，每包立即中断，
   宁可多花 CPU 也要最低延迟。
4. 中优先级任务会一直占着 CPU，低优先级持锁者跑不了、放不了 `desc->lock`，
   任何要这把锁的高优先级中断处理都被卡住——无界延迟。因为 raw_spinlock 不参与
   优先级继承，所以持有它的区间**必须极短**（绝不在里面做耗时操作或被抢占），
   这是内核对 raw_spinlock 的硬性纪律。

---

## 9. 学完自测

1. `CONFIG_IRQ_TIME_ACCOUNTING` 解决什么"算错账"问题？账记在哪几个桶里？
2. 中断延迟的两个含义各是什么？头号元凶是什么？分别用什么工具查？
3. 三种中断风暴各怎么应对？ksoftirqd 100% 是哪种的信号？
4. PREEMPT_RT 的五项主要改动是什么？哪些是"幸存者"仍保持原子？
5. 优先级继承解决什么问题？为什么 raw_spinlock 区间必须极短？

下一章 Topic 15：系统睡眠时中断怎么办、设备如何唤醒机器——挂起/恢复与中断电源管理。
