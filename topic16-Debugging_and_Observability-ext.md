# Topic 16 扩展版 — 调试与可观测性（实战手册）

> **用法同前**。行号基于 `~/repo/linux`（v7.1-rc7）：观测面源码在 `kernel/irq/proc.c`
> （/proc）、`kernel/irq/debugfs.c`（/sys/kernel/debug/irq）、`kernel/irq/spurious.c`
> （nobody cared）、`kernel/irq/irq_sim.c`（中断模拟器）；tracepoint 定义在
> `include/trace/events/irq.h:53` 起。
>
> **⚠️ 一处订正**：原文档示例里的 `irq_sim_fire(d, hwirq)` 在本内核树**不存在**。
> 模拟器实际触发中断的方式是 `irq_set_irqchip_state(virq, IRQCHIP_STATE_PENDING, true)`
> ——`drivers/gpio/gpio-sim.c:95` 就是这么用的。下文 §6 用正确写法。
>
> **承接**：这一章把 Topic 01~15 每章末尾的"Observability"小节**汇总成一套实战工具 +
> 症状→诊断手册**。它复用前面所有章节的知识，所以放在倒数第二。

---

## 1. 先建立画面感：四个观测面，各看什么

调试中断问题，你有四个"窗口"，各自回答不同问题。先记住这张全局图，后面就知道
"遇到 X 问题该开哪个窗口"：

| 窗口 | 回答什么问题 | 一句话 |
|---|---|---|
| **`/proc/interrupts`** | 什么在触发、在哪个 CPU | "事件计数 + 分布快照" |
| **`/sys/kernel/debug/irq/`** | 某个中断的**完整状态和层级** | "一个中断的全身 X 光" |
| **tracepoint** | **何时**触发、跑了**多久** | "逐次事件的录像" |
| **irqsoff/duration_warn** | **什么在害延迟** | "谁关了中断 800 微秒" |

**口诀（原文 Mental model）**：计数为 0 → 查 debugfs 映射；"nobody cared" → 查触发方式/
共享；ksoftirqd 烫 → 查软中断/NAPI；延迟高 → irqsoff；唤不醒 → wakeup_sources。

---

## 2. 逐句拆解 procfs/sysfs 观测面

### 晦涩点 ①：`/proc/interrupts` 是整个模型的快照

这是你**第一个该读**的东西。每一列都对应前面学的某个结构（Topic 03~05、13）：

```
            CPU0       CPU1
  24:    1048576          0   PCI-MSI 524288-edge      nvme0q1
  30:        512        498   GICv3 30 Level           arch_timer
 LOC:     900123     901240   Local timer interrupts
 RES:       1203       1190   Rescheduling interrupts        ← IPI（Topic 12）
```

逐列翻译：
- `24` = virq（Topic 04 的工号）；
- `1048576 0` = 每 CPU 的 `kstat_irqs` 计数（Topic 03），这里全压在 CPU0（亲和性，Topic 13）；
- `PCI-MSI` = chip 名（Topic 02/03）；`524288` = hwirq（分机号）；
- `edge` = flow handler 类型（Topic 05）；`nvme0q1` = action 名（Topic 06，驱动起的名）。

**读这一行 = 一次性看到 Topic 02→06 的全链路。** 底部带名字的块（LOC/RES/CAL/NMI）
是系统 IPI/NMI（Topic 12）。

### 晦涩点 ②：必须读 delta，不能读一次

原文 Pitfalls 第一条：`/proc/interrupts` 是**累计值**。看一次只知道"开机到现在总共多少次"，
看不出**速率**。要诊断"现在什么中断在狂涨"，必须**取差**：

```bash
watch -n1 'cat /proc/interrupts'           # 肉眼看哪行涨得快
# 或精确算每秒速率：
A=$(grep "^ *24:" /proc/interrupts); sleep 1; B=$(grep "^ *24:" /proc/interrupts)
echo "CPU0 delta: $(( $(echo $B|awk '{print $2}') - $(echo $A|awk '{print $2}') )) /s"
```

### 晦涩点 ③：其余 procfs/sysfs 速查

- **`/proc/softirqs`**：每 CPU 每软中断向量计数（Topic 08）。NET_RX 涨 + ksoftirqd 烫
  = 收包过载。
- **`/proc/stat`** 的 `intr`/`softirq` 总数；开了 `CONFIG_IRQ_TIME_ACCOUNTING` 还有每 CPU
  中断时间（Topic 14）。
- **`/proc/irq/N/`**：`smp_affinity`/`effective_affinity`（Topic 13）、`spurious`
  （未处理统计，Topic 05）、chip 名。
- **`/sys/kernel/irq/N/`**：`chip_name`/`hwirq`/`name`/`type`/`actions`/`per_cpu_count`/
  `wakeup`——`irq_desc` 的暴露（Topic 03）。

---

## 3. 逐句拆解 debugfs：一个中断的"全身 X 光"

### 晦涩点 ④：`/sys/kernel/debug/irq/irqs/N` 是最强单点诊断

需要 `CONFIG_GENERIC_IRQ_DEBUGFS`（`kernel/irq/debugfs.c`）。对一个 virq，它一次性打印：

- 安装的 `handle_irq`（flow handler，Topic 05）；
- **`IRQD_*`/`IRQS_*` 状态标志**（masked？disabled？affinity-managed？wakeup-armed？
  ——Topic 03/13/15）；
- **chip + hwirq + 完整 domain 层级**（每个父级的 hwirq/chip，Topic 04/11）。

**画面**：`/proc/interrupts` 是"体检表上的一行数字"，`debugfs/irqs/N` 是"这一个中断的
全身 CT"——状态、层级、芯片一览无余。**诊断单个中断问题的首选。**

`domains/` 子目录：每个 `irq_domain` 一个节点 + `default`，含名字、大小、映射数、flags、
**父链**（Topic 04 §6）。确认 MSI 映射、发现 vector domain 耗尽（Topic 11/13）就看这里。

---

## 4. 逐句拆解 tracepoint 与延迟工具

### 晦涩点 ⑤：tracepoint 看"何时、多久"

`include/trace/events/irq.h:53` 起定义。关键几个：`irq_handler_entry/exit`（每个 action
handler 前后，带返回值）、`softirq_raise/entry/exit`（Topic 08）、`tasklet_entry/exit`
（Topic 09）、`ipi:*`（Topic 12）。

bpftrace 三个最实用的配方（原文 §3，已验证字段名）：

```sh
# 每中断速率
bpftrace -e 'tracepoint:irq:irq_handler_entry { @[args->irq, str(args->name)] = count(); }'
# 每 handler 耗时直方图
bpftrace -e 'tracepoint:irq:irq_handler_entry { @s[tid]=nsecs }
             tracepoint:irq:irq_handler_exit /@s[tid]/ {
               @us[args->irq]=hist((nsecs-@s[tid])/1000); delete(@s[tid]); }'
# 每软中断向量耗时
bpftrace -e 'tracepoint:irq:softirq_entry { @s[cpu]=nsecs }
             tracepoint:irq:softirq_exit /@s[cpu]/ { @[args->vec]=sum(nsecs-@s[cpu]); delete(@s[cpu]); }'
```

### 晦涩点 ⑥：延迟工具（Topic 14 的工具在此汇总）

- **`irqsoff`/`preemptirqsoff`**：最长关中断/关抢占区间 + 代码路径（Topic 14 §3）。
- **`irqhandler.duration_warn_us=<N>`** 启动参数（Topic 14 §2）：top half 超 N 微秒就
  带函数名告警。
- **`function_graph`** 过滤 `common_interrupt`/`gic_handle_irq`/具体 handler：看时间
  花在哪（topic01/05 任务用过）。
- **`cyclictest`**：端到端唤醒延迟（Topic 14 §5）。

---

## 5. 症状→诊断手册（本章最有价值的产出）

原文 §5 的表是**贴墙级**参考。我按"先看哪个窗口"重排，遇到问题照查：

| 症状 | 可能原因 | 怎么确认/修 |
|---|---|---|
| `irq N: nobody cared` 后中断被禁 | 尖叫线：触发方式错，或共享 handler 返回值错（Topic 02/05）| 查 DT/ACPI 触发方式；审 handler 的 `IRQ_NONE` 逻辑；`/proc/irq/N/spurious` |
| 某中断计数恒为 0 | 从不触发：hwirq 映射错、亲和性错、设备不产生、被屏蔽 | `debugfs/irqs/N`（hwirq/domain 对吗？masked？）；查 `effective_affinity`；查设备 |
| 中断只落 CPU0，尽管 smp_affinity 很宽 | 单目标 effective_affinity（很多芯片正常），或关了 irqbalance 又没 pin | 读 `effective_affinity_list`；pin 或用托管亲和性（Topic 13）|
| 写 `/proc/irq/N/smp_affinity` 报 -EIO | 托管中断（多队列），内核所有（Topic 13 §2）| 正常；改设备队列数 |
| `ksoftirqd/N` ~100% | 软中断过载，通常 NET_RX 风暴（Topic 08/14）| `/proc/softirqs` delta；开 NAPI/调亲和性/`ethtool -C` |
| 关中断延迟高/RT 错过 | 长 handler 或 `spin_lock_irqsave` 区间（Topic 10/14）| `irqsoff` tracer；`duration_warn_us`；活儿挪线程/软中断 |
| x86 `vector allocation failed` | 每 CPU 向量耗尽（Topic 02/11/13）| `debugfs/domains/VECTOR`；减 MSI-X；用托管亲和性 |
| 设备只在 suspend/resume 后死 | 控制器状态没还原，或唤醒丢失（Topic 15）| 查 chip `irq_suspend/resume`；`enable_irq_wake`；`wakeup_sources` |
| 设备唤不醒系统 | 缺 `enable_irq_wake` 或 sysfs wakeup 禁用（Topic 15）| 两者都开；debugfs 确认 `IRQD_WAKEUP_ARMED` |
| `scheduling while atomic` BUG | 在硬/软中断里睡眠（Topic 10）| `DEBUG_ATOMIC_SLEEP` 栈；挪到线程/工作队列 |
| lockdep `inconsistent {HARDIRQ-ON-W}` | 中断上下文拿锁没用 `_irqsave`（Topic 10）| 按 lockdep 打印的两个栈改用 `spin_lock_irqsave`/`_bh` |
| "Function call interrupts" 高 | TLB shootdown/`smp_call_function` 风暴（Topic 12/14）| trace `ipi:*`；通常是 mm/扩展性问题 |
| 线程化 handler 似乎慢 | 负载下唤醒到运行延迟（Topic 07）| 查 `irq/N-name` 线程优先级/CPU；考虑挪回 top half |

---

## 6. 逐句拆解 irq_sim：无硬件测试中断

### 晦涩点 ⑦：怎么在 CI 里测试中断逻辑而不需要真设备

写中断**消费方**的测试时，**中断模拟器**（`kernel/irq/irq_sim.c`）创建一个软件
`irq_domain`，你可以**用代码触发**它的中断：

```c
struct irq_domain *d = devm_irq_domain_create_sim(dev, fwnode, num_irqs);
int virq = irq_create_mapping(d, hwirq);    /* 像普通中断一样 request_irq */
/* ★触发：注意不是 irq_sim_fire（那个不存在），而是： */
irq_set_irqchip_state(virq, IRQCHIP_STATE_PENDING, true);  /* 置 pending → 触发 handler */
```

`drivers/gpio/gpio-sim.c:95` 就是这么干的——GPIO 子系统的 selftest 用它在没有真 GPIO
芯片的情况下验证中断路径。`kernel/irq/irq_test.c` 测试核心。

**画面**：irq_sim 像一个"中断的假人模特"——你能在它身上练习 request_irq、线程化、
亲和性的全套逻辑，CI 里跑测试，不用插真硬件。

---

## 7. 动手任务（针对你的 x86_64 机器）

### 任务 0：把 /proc/interrupts 读成"全链路快照"（10 分钟，无需 root）

挑一行 PCI-MSI 中断，把每一列对应回前面学的概念：

```bash
grep -m1 "PCI-MSI" /proc/interrupts
```

回答（答案 §8.0）：写出这一行的 virq、chip、hwirq、flow 类型、设备名分别是什么，
各来自哪个 Topic 的哪个结构。（这是 Topic 03 任务 0 的复习 + 升级。）

### 任务 1：算中断速率（delta），找最忙的中断（15 分钟）

```bash
# 写个小脚本算每秒速率：
paste <(awk '/[0-9]:/{s=0;for(i=2;i<=NF-2;i++)s+=$i;print $1,s}' /proc/interrupts) \
      <(sleep 1; awk '/[0-9]:/{s=0;for(i=2;i<=NF-2;i++)s+=$i;print s}' /proc/interrupts) \
  | awk '{print $1, $3-$2"/s"}' | sort -t' ' -k2 -rn | head
```

回答（答案 §8.1）：

1. 你机器上每秒中断最多的是哪个？空闲时通常是定时器（LOC）；制造负载（网络/磁盘）
   后变成谁？
2. 为什么必须算 delta 而不能直接读？（原文 Pitfalls 第一条。）

### 任务 2：用 debugfs 给一个中断做"全身 CT"（15 分钟，需要 root + debugfs）

```bash
N=$(grep -m1 nvme /proc/interrupts | awk -F: '{print $1}' | tr -d ' ')
sudo cat /sys/kernel/debug/irq/irqs/$N
```

回答（答案 §8.2）：

1. 输出里能看到哪些 `IRQD_*`/`IRQS_*` 状态标志？这个中断 masked 吗？是 managed 吗？
2. domain 层级有几层？和 Topic 11 任务 1 看到的一致吗？
3. 对比 `/proc/interrupts` 的那一行——debugfs 多告诉了你什么 `/proc` 看不到的信息？

### 任务 3：用 bpftrace 抓中断速率和 handler 耗时（20 分钟，需要 root + bpftrace）⭐

```bash
# 每中断速率（跑 5 秒）：
sudo bpftrace -e 'tracepoint:irq:irq_handler_entry { @[args->irq, str(args->name)] = count(); }
                  interval:s:5 { exit(); }'
# 每 handler 耗时直方图：
sudo bpftrace -e 'tracepoint:irq:irq_handler_entry { @s[tid]=nsecs }
                  tracepoint:irq:irq_handler_exit /@s[tid]/ {
                    @us[str(args->irq)]=hist((nsecs-@s[tid])/1000); delete(@s[tid]); }
                  interval:s:8 { exit(); }'
```

回答（答案 §8.3）：

1. 哪个中断的 handler 调用最频繁？哪个的 handler 耗时最长？
2. 耗时直方图大多落在几微秒？有没有超过 100 微秒的离群值？（联系 Topic 14
   duration_warn。）
3. `args->name` 和 `/proc/interrupts` 里的设备名对得上吗？

### 任务 4：实战演练手册——故意制造一个症状去诊断（30 分钟，需要 root，虚拟机）

挑一个**安全可控**的症状来走一遍诊断流程。推荐：**让一个中断只落在一个 CPU，
然后用手册定位**。

```bash
N=某非托管中断号
echo 1 | sudo tee /proc/irq/$N/smp_affinity      # pin 到 CPU0
cat /proc/irq/$N/smp_affinity_list                # 请求：0
cat /proc/irq/$N/effective_affinity_list          # 实际：?
grep "^\s*$N:" /proc/interrupts                    # 计数集中在 CPU0 吗？
```

回答（答案 §8.4）：

1. 按 §5 手册"中断只落 CPU0"这一行，你的确认步骤是什么？effective_affinity 怎么读？
2. 如果想让它落在 CPU2，怎么改？改完 `effective_affinity` 立刻变了吗？
   （回忆 Topic 13 晦涩点 ①：下次中断才生效。）
3. （进阶）写一个 irq_sim 模块，用 `irq_set_irqchip_state(virq,
   IRQCHIP_STATE_PENDING, true)` 触发，在 `/sys/kernel/debug/irq/domains/` 里
   找到你的模拟 domain。

### 任务 5：思考题（答案 §8.5）

1. 一个中断 `/proc/interrupts` 计数为 0。列出至少四种可能原因，以及用哪个观测面
   分别排除它们。
2. `/proc/irq/N/spurious` 和"高速率合法中断"怎么区分？为什么这两者的修法完全不同？
3. 为什么调试中断延迟问题，开 `CONFIG_IRQ_TIME_ACCOUNTING` 之前看到的 "system CPU 高"
   是误导性的？
4. lockdep 报 `inconsistent {HARDIRQ-ON-W}`，它打印的两个调用栈分别告诉你什么？
   你该改哪一处、怎么改？

---

## 8. 任务参考答案

### 8.0 任务 0

virq = 行首数字（Topic 04 工号 / Topic 03 索引）；chip = `PCI-MSI`（Topic 02/03 的
`irq_chip.name`）；hwirq = chip 后的大数字（Topic 04 分机号）；flow 类型 = `-edge`/
`-fasteoi`（Topic 05 的 `desc->handle_irq`）；设备名 = 最右（Topic 06 的 `action->name`）。
对应 Topic 03 晦涩点（五元组对应五个字段）。

### 8.1 任务 1

1. 空闲时通常 LOC（本地定时器）或 RES 最多；网络负载后变成网卡队列中断（如
   `nvme`/`enp` 行）或 NET_RX 相关。
2. 因为计数是开机至今的累计；读一次无法区分"历史上很忙"和"现在很忙"。delta
   才反映当前速率。

### 8.2 任务 2

1. 常见 `IRQD_ACTIVATED`、`IRQD_AFFINITY_SET`、可能有 `IRQD_AFFINITY_MANAGED`
   （NVMe 多为托管）、`IRQD_IRQ_MASKED`（处理中短暂）等。masked 一般为否（除非正处理）。
2. 通常 2~3 层（PCI-MSI → [IR →] VECTOR），和 Topic 11 任务 1 一致。
3. 多了：完整状态标志、domain 父链每层的 hwirq/chip、是否托管/可唤醒——
   `/proc/interrupts` 只给计数和叶子层。

### 8.3 任务 3

1. 通常定时器或网卡中断调用最频繁；handler 耗时最长的可能是某些慢设备或虚拟化路径。
2. 多数落在几微秒（top half 本就短）；超 100us 的离群值值得用 duration_warn 进一步盯。
3. 应对得上（args->name 就是 action 名）。

### 8.4 任务 4

1. 读 `effective_affinity_list`——若它是单个 CPU（如 0），说明芯片单目标投递，这是
   "只落 CPU0"的正常解释，不是 bug。计数集中在 CPU0 列与之一致即确认。
2. `echo 4 | sudo tee /proc/irq/$N/smp_affinity`（0x4=CPU2）。effective_affinity
   **不一定立刻变**——很多芯片要等该中断**下次触发**才迁移（Topic 13 pending_mask）。
   少触发的设备会迟迟不变。
3. 模块里 `irq_domain_create_sim` 建 domain、`irq_create_mapping` 建映射、
   `request_irq` 注册、`irq_set_irqchip_state(virq, IRQCHIP_STATE_PENDING, true)` 触发；
   `sudo ls /sys/kernel/debug/irq/domains/` 能看到你的（可能无名/带 fwnode 名）domain。

### 8.5 任务 5

1. ① hwirq 映射错（debugfs/irqs/N 看 hwirq/domain 对不对）；② 亲和性指向了离线/
   错误 CPU（看 effective_affinity）；③ 被屏蔽/disabled（debugfs 看 IRQD_IRQ_MASKED/
   depth）；④ 设备根本没产生中断（查设备本身/驱动初始化）。每种用括号里的观测面排除。
2. `/proc/irq/N/spurious` 显示 unhandled 计数——高 unhandled = 假中断/尖叫（修触发
   方式或共享 handler 返回值）；高速率但低 unhandled = 真实高负载（修法是缓解/NAPI/
   合并）。前者是"配置/逻辑错"，后者是"量太大"，方向完全相反。
3. 没开 IRQ_TIME_ACCOUNTING 时，中断时间被错记进被打断任务的 system（甚至 user）
   时间（Topic 14 晦涩点 ①），让你以为是某进程在耗 system CPU，实际是中断/软中断。
   开了才能把中断时间单独分出来。
4. 两个栈：一个是"这把锁曾在硬中断上下文（HARDIRQ）被安全获取"的位置，另一个是
   "这把锁在开着硬中断（HARDIRQ-ON，即可能被中断打断）时被获取"的位置。后者是隐患
   点——把那一处的 `spin_lock` 改成 `spin_lock_irqsave`（或视情况 `_bh`），
   让持锁期间中断不能插入，配方被破坏（Topic 10 任务 3）。

---

## 9. 学完自测

1. 四个观测面各回答什么问题？遇到"计数为 0"/"nobody cared"/"ksoftirqd 烫"/"延迟高"
   分别先开哪个？
2. 为什么 `/proc/interrupts` 必须读 delta？effective_affinity 为什么比 smp_affinity
   更能说明"中断在哪跑"？
3. `debugfs/irqs/N` 比 `/proc/interrupts` 多给哪些信息？
4. 默写症状手册里至少五条：症状 → 原因 → 确认手段。
5. irq_sim 怎么在无硬件下触发一个中断？（正确的函数是什么？）

下一章 Topic 17（收官）：把前面所有 Topic 重新串到**启动时间线**上——从第一个 IDT/
向量安装，到第一个设备中断。
