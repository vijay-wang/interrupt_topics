# Topic 00 扩展版 — 预备知识与全局总览（整个系列的地图）

> **用法**：这是**导览章**，先读它建立全局画面，再去读 01~17 的扩展版；读完 Topic 03
> 后**回头再读一遍本章**——等你认识了 `irq_desc`，这张地图会突然变清晰。
> 行号基于 `~/repo/linux`（v7.1-rc7）：`in_*()` 判断在 `include/linux/preempt.h:126`、
> `irqreturn` 枚举在 `include/linux/irqreturn.h`、驱动 API 在 `include/linux/interrupt.h`。
>
> **本章和别章不同**：它不深挖某个机制，而是**搭骨架、立比喻、给地图**。所有比喻
> （门铃/管家、分机号/工号、前台总机……）在这里第一次出现，后面各章反复使用。

---

## 1. 先建立画面感：一个贯穿全系列的比喻

整个 18 章，我用**一栋大宅子**的比喻串起来，先在这里立好，后面每章都在用：

| 比喻 | 对应的真实东西 | 哪章详讲 |
|---|---|---|
| 管家（埋头干活，听到门铃去应门）| CPU 核心 | Topic 01 |
| 门铃系统（汇聚几百个访客的呼叫）| 中断控制器（APIC/GIC）| Topic 02 |
| 客户档案夹 | `irq_desc`（中断描述符）| Topic 03 |
| 分机号 → 集团工号的翻译 | hwirq → virq（IRQ domain）| Topic 04 |
| 不同访客的接待流程模板 | flow handler | Topic 05 |
| "这路铃响找我"的认领登记 | `request_irq`/`irqaction` | Topic 06 |
| 前台收件 + 后台员工拆件 | 硬中断 + 线程化处理 | Topic 07 |
| 快递员打勾、关门时统一处理 | 软中断 | Topic 08 |
| 几种"延迟干活"的去处 | tasklet/工作队列/irq_work | Topic 09 |
| "此刻我在哪、能做什么"的身份牌 | preempt_count / 上下文 | Topic 10 |
| 投信箱代替拉电线 | MSI 消息中断 | Topic 11 |
| 同事间内部短信、全楼火警 | IPI、NMI | Topic 12 |
| 这通电话派给哪个接线员 | 亲和性、均衡 | Topic 13 |
| 救护车多久到、伤员多久排上队 | 中断延迟 | Topic 14 |
| 睡觉留门铃和闹钟 | 挂起/唤醒 | Topic 15 |
| 四个观测窗口 + 排查手册 | 调试 | Topic 16 |
| 装修通电的固定顺序 | 启动初始化 | Topic 17 |

**读完本章你应能**：用这套比喻把"一个网络包到达 → 网卡中断 → 内核处理 → 唤醒应用"
讲一遍。

---

## 2. 逐句拆解"中断是什么、不是什么"

### 晦涩点 ①：为什么"异步"是关键词？

**中断**是硬件发来的**异步**信号，把 CPU 从手头的事拽进内核处理函数。"异步"是题眼：
它**和 CPU 正在执行的指令毫无关系**——网卡的包什么时候到，跟你的程序算到第几行，
没有任何关联。

对比"同步"事件（缺页、系统调用）：那些是当前指令**自己引发的**（你这条指令访问了
没映射的内存 → 缺页）。**画面**：异步中断像"突然有人按门铃"（与你在干嘛无关）；
同步异常像"你自己打翻了杯子"（你的动作直接导致的）。

### 晦涩点 ②：一张表分清"中断大家庭"

人们把好几样东西都叫"中断机器"，但它们不同（原文 §1 的表，加画面）：

| 事件 | 同步？ | 来源 | 画面 |
|---|---|---|---|
| **中断（IRQ）** | 否（异步）| 外部设备经控制器 | 门铃响 |
| **异常/fault** | 是 | CPU 自己 | 自己打翻杯子（缺页）|
| **trap** | 是 | 故意的指令 | 主动喊"暂停"（断点）|
| **系统调用** | 是 | `syscall`/`svc` 指令 | 主动打内线电话求助 |
| **NMI** | 否（异步、不可屏蔽）| 特殊线/perf/看门狗 | 全楼强制火警 |
| **IPI** | 否（异步）| 另一个 CPU | 同事发的内部短信 |
| **MSI** | 否（异步）| 设备**写内存** | 往信箱投纸条 |

**本系列讲异步那半**（IRQ/NMI/IPI/MSI）。同步异常/系统调用只和中断**共享最底层的
架构入口**（Topic 01），之后就分道扬镳——我们只讲共享的入口，不讲缺页处理器本身。

### 晦涩点 ②.5：常见困惑——"系统调用、trap、异常不也是中断吗？为什么本系列不讲它们？"

这是初学者**最常见、也最关键**的一个困惑，单独讲透。先纠正一个容易说反的点：

> **系统调用、trap、异常/fault 都是「同步」事件，不是异步的。这正是它们和中断的
> 根本区别——它们和中断不是同一种东西。**

**判别只有一条线：触发源在不在 CPU 之外、发生时刻和当前指令流有没有关系。**

| 事件 | 谁触发的 | 和当前指令的关系 | 同步/异步 |
|---|---|---|---|
| 异常/fault（缺页、除零、#GP）| **当前这条指令自己** | 是它直接引起的 | **同步** |
| trap（int3 断点、单步）| **当前这条故意的指令** | 是它主动触发的 | **同步** |
| 系统调用（syscall/svc）| **程序主动执行的一条指令** | 就是这条指令的目的 | **同步** |
| 中断（IRQ/网卡/磁盘）| **外部设备** | **毫无关系** | **异步** |

**画面**：异常/trap/syscall 像"**你自己**打翻杯子 / 自己按暂停键 / 自己打内线电话"
——都是**你当前动作的直接后果**，发生时刻和指令流严格绑定，重放这条指令必然再来一次。
中断像"**别人**按门铃"——和你在干嘛**完全无关**，不可预测。"异步"的精确含义就是
**触发源在 CPU 之外、发生时刻与当前指令无关**。

**那为什么硬件手册常把它们混一起讲？** 因为它们**共享最底层的"陷入内核"管道**——
都要保存现场 → 切内核态/内核栈 → 跳到一个入口。x86 上异常和中断都走 IDT（0~31 异常、
32~255 中断）；arm64 都走那张 16 项异常向量表。**硬件用同一套机制把它们"陷进"内核**，
所以放一起。但**进了内核立刻分道扬镳**走完全不同的子系统：

```
缺页    → do_page_fault（读 CR2/ESR，去补页表）        ── 内存管理子系统 mm/
系统调用 → 系统调用分发表（syscall entry）              ── 系统调用子系统
中断    → generic_handle_irq → flow handler → 你的handler ── 中断子系统 kernel/irq/ ★本系列
```

**所以本系列只讲中断、不讲那三个**，因为主题是 **Linux 中断子系统（`kernel/irq/`、
genirq）**——专门处理**异步**事件的那套基础设施（控制器、domain、irq_desc、flow
handler、上下半部……）。缺页/异常属于内存管理子系统（`arch/*/mm/fault.c`），系统调用
属于系统调用子系统，各自成体系、和 irq_desc 那套**毫无关系**。本系列**唯一覆盖**它们的
地方就是 Topic 01 讲的"共享的最底层架构入口"，之后就只跟着中断这条路走。

**一个立竿见影的验证**：缺页**永远不会**出现在 `/proc/interrupts` 里（topic01-ext 的
Pitfalls 专门提过）——因为它根本不走中断子系统，不存在对应的 irq_desc。你在
`/proc/interrupts` 看到的每一行，都是异步中断，没有一行是异常或系统调用。

### 晦涩点 ③：为什么要中断？为什么有时又关掉它改轮询？

**为什么要中断**：替代方案是**轮询（polling）**——CPU 不停问"设备好了没？好了没？"，
什么正事都干不了。中断让 CPU 该干嘛干嘛，有事了被通知。代价是**延迟和并发复杂度**
（本系列剩下的内容就是管理这个代价）。

**为什么高频时反而关中断改轮询**：当事件率极高（万兆网卡每秒百万包），"每个事件一次
中断"的开销超过收益，这时内核**故意切回轮询**（NAPI/irq_poll，Topic 09）。
**画面**：门铃平时好用（有事才打扰），但门口排了一百个快递员时，干脆关掉门铃、
自己定期去门口批量收（轮询）。这是 Topic 09 的伏笔。

---

## 3. 逐句拆解全系列脊柱：端到端路径

### 晦涩点 ④：把这条流水线刻进脑子（每章都是它的放大）

这是整个系列**最重要的一张图**，原文 §2 给了完整版，我用比喻重画一遍：

```
设备拉线/写消息          →  控制器(门铃系统)demux  →  CPU(管家)跳到入口
(Topic 02)                  搞清是谁按的铃           保存现场、切IRQ栈
                            (Topic 02)              (Topic 01)
                                                         ↓
架构底层入口 + 通用入口层(签到)                          (Topic 01)
                                                         ↓
通用 IRQ 核心：
  generic_handle_irq(virq) ── 用 domain 把 hwirq→virq  (Topic 04)
     → irq_desc->handle_irq = flow handler             (Topic 05)
        → 遍历 action 链 → 你的 handler(top half)       (Topic 06)
           返回 HANDLED / NONE / WAKE_THREAD
                                                         ↓ 把重活推迟
下半部（延迟工作）：
  软中断(08) / tasklet·BH工作队列·irq_work(09) /
  线程化IRQ(07) / 工作队列（进程上下文）
                                                         ↓
中断返回(irqentry_exit)：跑挂起的软中断、查 need_resched、
返回用户态时送信号                                       (Topic 01)
```

### 晦涩点 ⑤：上半部/下半部分割——全系列最重要的一个思想

原文加粗强调："The top half / bottom half split is the single most important idea."

**为什么要分割**：中断处理函数（top half，上半部）运行时**中断可能被屏蔽、不能睡眠、
必须极短**（否则拖累整个系统）。所以它只做**最紧急的最小动作**（应答设备、抓一点数据），
把剩下的重活**推迟到一个限制更少的上下文**去做（bottom half，下半部）。

**画面**：前台（上半部）只负责"收下包裹、打个勾"，绝不当场拆包裹研究——因为前台
一忙，所有访客都堵在门口。拆包裹的重活交给后台（下半部）慢慢干。**Topic 06~09
整整四章都在讲这个分割。** 如果你只读了 `request_irq`，你只看到了整条路径的约 20%。

---

## 4. 逐句拆解两个绝不能混的编号

### 晦涩点 ⑥：hwirq vs virq（分机号 vs 集团工号）

这是**整个系列最常见的 bug 来源**，必须在第 0 章就钉死。一个物理中断在 Linux 里有
**两个**号：

- **hwirq**（`irq_hw_number_t`）：**控制器**用的号。"GIC SPI 42"、"IO-APIC 引脚 16"、
  "某个 MSI 索引"。**只在单个控制器内有意义，跨控制器会撞**（每个 GIC 都有 hwirq 30）。
- **virq**（Linux IRQ 号，普通的 `unsigned int irq`）：**内核全局**的号。它索引
  `irq_desc`，是 `request_irq()` 的参数、`/proc/interrupts` 最左列的数字。

**比喻**（Topic 04 详讲）：hwirq 是**分公司的分机号**（北京和上海都有分机 32），
virq 是**集团唯一工号**。IRQ domain 层（Topic 04）就是"分机号→工号"的翻译官。

### 晦涩点 ⑦：一条判别口诀

> **从设备树、ACPI、控制器寄存器来的数 = hwirq；索引 `irq_desc` 或传给
> `request_irq`/`free_irq`/`enable_irq` 的数 = virq。**

`/proc/interrupts` 左列是 virq；右边"edge/level + 控制器名"那串描述的是 hwirq 那一侧。
记住这条，"我的中断为什么不触发"这类问题先排除"用错号了"。

---

## 5. 逐句拆解四个执行上下文与那个"身份牌"

### 晦涩点 ⑧：四个上下文，规则天差地别

任一时刻 CPU 恰好处于这四个上下文之一，**能做什么差别极大**（原文 §4 表）：

| 上下文 | 能睡？ | 能阻塞分配？ | 典型工作 | 怎么检测 |
|---|---|---|---|---|
| **进程/任务** | 能 | 能（GFP_KERNEL）| 系统调用、kthread、线程化 handler | `in_task()` |
| **软中断/下半部** | **不能** | 不能（只 GFP_ATOMIC）| 网络收包、定时器、tasklet | `in_serving_softirq()` |
| **硬中断** | **不能** | 不能 | top half、flow handler | `in_hardirq()` |
| **NMI** | **不能**（几乎啥都不能）| 不能 | 看门狗、perf | `in_nmi()` |

### 晦涩点 ⑨：一个词 `preempt_count` 编码"我在哪"

这些 `in_xxx()` 判断**就是对一个每 CPU 的字 `preempt_count` 做位测试**
（`include/linux/preempt.h:126`）：

```c
#define in_nmi()             (nmi_count())
#define in_hardirq()         (hardirq_count())
#define in_serving_softirq() (softirq_count() & SOFTIRQ_OFFSET)
#define in_interrupt()       (irq_count())     /* 硬中断 | 软中断 | nmi */
#define in_task()            (!(preempt_count() & (NMI_MASK|HARDIRQ_MASK|SOFTIRQ_OFFSET)))
```

`preempt_count` 把四个子计数器（抢占禁用深度、软中断深度、硬中断深度、NMI 深度）
打包进一个字。进硬中断 +HARDIRQ_OFFSET，跑软中断 +SOFTIRQ_OFFSET……
这就是 `might_sleep()`、lockdep、`GFP_KERNEL` 判断"我在不在禁区"的依据。

**心智模型（现在记住即可，Topic 10 逐位详解）**：**一个词告诉内核"我在哪、能干什么"。**

### 晦涩点 ⑩：handler 返回值的三种含义

`include/linux/irqreturn.h`：

```c
enum irqreturn { IRQ_NONE = 0, IRQ_HANDLED = 1, IRQ_WAKE_THREAD = 2 };
```

- `IRQ_NONE`：不是我的设备（共享中断时用，Topic 05 的侦探依赖它）；
- `IRQ_HANDLED`：处理完了；
- `IRQ_WAKE_THREAD`：唤醒我的线程化 handler 去进程上下文里完成剩下的（Topic 07）。

---

## 6. 逐句拆解两个架构世界

### 晦涩点 ⑪：x86 和 arm64 是两个硬件世界，藏在一个抽象后面

Linux 用 `irq_chip` + `irq_domain` 把两个差异极大的硬件世界藏在一个接口后。但你**两个
都得认识**（这条主线贯穿 01/02/11/13）：

**x86_64**：
- **IDT**（256 个门铃按钮），中断的**向量**（0~255）索引它。0~31 是 CPU 异常，
  其余是设备/IPI 向量——**稀缺的每 CPU 资源**，由矩阵分配器管（Topic 13）。
- **Local APIC**（每 CPU 私人信箱）+ **I/O APIC**（外部总机），MSI 写也走 APIC。

**arm64**：
- **16 项异常向量表**（`VBAR_EL1` → `vectors`），按异常级别和同步/异步分。IRQ/FIQ 是
  异步入口。
- **GIC** 分发 SPI/PPI/SGI，GICv3+ 加 **ITS** 处理 MSI（LPI）。**没有 x86 那样的
  稀缺每 CPU 向量空间**——模型不同，这正是 `irq_domain` 抽象的价值所在。

**抽象的回报**：驱动调 `request_irq(virq, …)`，flow handler 跑 `irq_desc->handle_irq`，
**两个架构上完全一样**。只有底下的 `irq_chip`（mask/unmask/eoi）和 domain 不同。

> **核心主线（记住它，贯穿全系列）**：**x86 硬件 demux（向量已编码源、但向量稀缺）
> vs arm64 软件翻译（读控制器问是谁、但编号空间充裕）。** 这一对差异在 Topic 01
> （入口）、02（控制器）、11（MSI）、13（亲和性/矩阵）反复出现——认准它，很多
> "为什么 x86 这样 arm64 那样"的问题迎刃而解。

---

## 7. 源码地图（每章钻一行）

把这张表存着，每读一章对照它打开对应文件（原文 §6 已全，这里给学习路径建议）：

```
kernel/irq/                  通用 IRQ 核心（genirq）
  irqdesc.c    irq_desc 生命周期、稀疏存储           (Topic 03/05)
  irqdomain.c  hwirq↔virq 映射、层级 domain          (Topic 04)
  chip.c       irq_chip helper + flow handlers ★最清晰 (Topic 05)
  handle.c     handle_irq_event、action 链遍历        (Topic 05)
  manage.c     request_irq/free_irq/线程化/亲和性     (Topic 06/07/13)
  msi.c        MSI 框架                               (Topic 11)
  ipi.c        处理器间中断                           (Topic 12)
  matrix.c     x86 向量位图分配器                     (Topic 13)
  cpuhotplug.c CPU 间迁移中断                         (Topic 13)
  spurious.c   "nobody cared" 检测                    (Topic 05/16)
  pm.c         irq_desc 的挂起/恢复                   (Topic 15)
  debugfs.c    /sys/kernel/debug/irq                  (Topic 16)
kernel/softirq.c   软中断、tasklet、local_bh_*        (Topic 08/09)
kernel/irq_work.c  硬中断/NMI 安全回调                (Topic 09)
include/linux/  irq.h irqdesc.h interrupt.h preempt.h irq-entry-common.h msi.h
arch/x86/kernel/   irq.c irqinit.c idt.c apic/
arch/arm64/kernel/ irq.c entry.S entry-common.c
drivers/irqchip/   irq-gic*.c …（真实控制器驱动）     (Topic 02)
```

**新手最佳起步三个文件**（原文建议）：`include/linux/interrupt.h`（驱动 API 全在一个头）、
`kernel/irq/chip.c`（flow handler，最能"看到"状态机）、`include/linux/preempt.h`
（in_* 判断和 preempt_count 布局）。

---

## 8. 动手任务（针对你的 x86_64 机器，整个系列的热身）

### 任务 0：用一行 /proc/interrupts 验证 §3 §4 的所有概念（15 分钟，无需 root）

```bash
cat /proc/interrupts | head -5
grep -m1 "PCI-MSI" /proc/interrupts
```

回答（答案 §9.0）：

1. 左列数字是 hwirq 还是 virq？右边"PCI-MSI 524288-edge"里的 524288 是哪个？
   用 §4 的判别口诀说明。
2. 这一行体现了端到端路径（§2）的哪几个阶段？（提示：chip 名、flow 类型、设备名
   各对应一个阶段。）

### 任务 1：在源码里确认四个上下文判断（15 分钟）

```
① include/linux/preempt.h:126   in_nmi/in_hardirq/in_serving_softirq/in_interrupt/in_task
② include/linux/irqreturn.h      irqreturn 三个值
③ include/linux/interrupt.h      request_irq 的签名（确认它收 virq）
```

回答（答案 §9.1）：

1. `in_interrupt()` 包含哪三种上下文？它和 `in_task()` 互补吗？
2. `request_irq` 的第一个参数类型是什么？它是 hwirq 还是 virq？（联系 §4。）

### 任务 2：把"网络包到达"讲成一个完整故事（20 分钟，纸面）⭐

用 §1 的比喻 + §2 的脊柱图，写一段话描述：一个网络包到达网卡后，从硬件到你的应用
被唤醒，**完整经过哪些阶段、各属于哪个 Topic**。要求点到：控制器 demux、CPU 入口、
hwirq→virq、flow handler、top half、软中断（NAPI）、上下文限制。

回答见 §9.2（对照你写的，查漏补缺——这是检验你是否抓住全局的关键任务）。

### 任务 3：建立你的"学习路线"（10 分钟，规划）

根据原文 plan 的角色化阅读路径思想，结合你的目标，回答（答案 §9.3 给建议）：

1. 你的目标是"写设备驱动"、"调试中断问题"、还是"理解内核原理"？
2. 针对你的目标，18 章里哪几章是核心、哪几章可以先略读？

### 任务 4：思考题（答案 §9.4）

1. 为什么同步异常（缺页）和异步中断（网卡）共享"最底层入口"却又"分道扬镳"？
   它们的根本区别是什么？
2. 如果没有 hwirq/virq 两层编号，直接用 hwirq 当全局号，会出什么问题？
3. 上半部/下半部分割的本质，是在"两种上下文的能力差异"之间做权衡。
   用 §4 的表说明：为什么重活必须挪到下半部，而下半部又分好几种（07/08/09）？
4. 为什么 Linux 要用 `irq_chip`+`irq_domain` 抽象把 x86 和 arm64 藏起来，
   而不是各写一套？（联系 §6 的"换硬件不换上层代码"。）

---

## 9. 任务参考答案

### 9.0 任务 0

1. 左列是 **virq**（索引 irq_desc、request_irq 用它）；524288 是 **hwirq**
   （MSI 索引，来自控制器侧）。口诀：右边"控制器名 + 这个数"描述 hwirq 侧。
2. chip 名（`PCI-MSI`）= 控制器阶段（Topic 02）；flow 类型（`edge`）= flow handler
   阶段（Topic 05）；设备名（`nvme0q1`）= action/top half 阶段（Topic 06）。
   一行字串起了脊柱图的好几段。

### 9.1 任务 1

1. `in_interrupt()` = 硬中断 | 软中断 | NMI。和 `in_task()` 基本互补（在中断里就不在
   任务里），细微边界见 Topic 10（bh_disable 时 in_softirq 真但仍算进程上下文）。
2. `unsigned int irq`——是 **virq**。驱动永远传 virq，hwirq 由 domain 翻译，
   驱动基本不碰（Topic 04/06）。

### 9.2 任务 2（参考故事）

包到达网卡 → 网卡通过 MSI **写一条消息**（Topic 11）→ 控制器（APIC/GIC）收到并
**demux** 出是哪个中断（Topic 02，x86 硬件给向量 / arm64 读 GIC）→ CPU **停下当前任务、
保存现场、切到 IRQ 栈、跳到入口**（Topic 01）→ 通用入口层签到（RCU/lockdep）→
通用核心用 **domain 把 hwirq 翻译成 virq**（Topic 04）→ 找到 irq_desc、调它的
**flow handler**（Topic 05，多为 handle_edge_irq）→ 遍历 action 链调网卡驱动的
**top half**（Topic 06）→ top half **只做最小动作**（确认中断、`napi_schedule`）就返回
`IRQ_HANDLED` → 真正的收包在**软中断 NET_RX**（NAPI，Topic 08/09）里进行，
跑在不能睡眠的软中断上下文（Topic 10）→ 协议栈处理完把数据交给 socket →
最终**唤醒**阻塞在 `recv()` 的你的应用（进程上下文）。全程上半部极短、重活在下半部。

### 9.3 任务 3（路线建议）

1. 三种目标都常见。2. 建议：
- **写驱动**：核心 03、04、06、07、09、11；略读 01、02、13、16 够用，10 必读（锁规则）。
- **调试问题**：核心 16（手册）、05、08、10、13、14；01/02 建立背景，按症状回查。
- **理解原理**：按 00→17 顺序通读，01/03/04/05/10 是理解骨架的关键。

### 9.4 任务 4

1. 共享入口是因为两者都要"从用户/内核态切进异常处理、保存现场"——这套机制相同
   （Topic 01 通用入口层）。分道扬镳是因为根本区别：同步异常是**当前指令引发的**
   （CPU 知道原因，读 CR2/ESR 就知道是哪个地址缺页），异步中断是**外部设备引发的**
   （CPU 不知道是谁，要问控制器）。一个问 CPU 自己，一个问门卫。
2. hwirq 跨控制器撞号（每个 GIC 都有 hwirq 30），无法做全局唯一索引；且控制器是
   动态发现/热插拔的，没法静态预分配全局号。两层编号 + domain 翻译解决了撞号和
   动态分配（Topic 04）。
3. 上半部在硬中断上下文（不能睡、必须短、可能屏蔽中断），能力受限；重活（要睡眠、
   要花时间）放这里会拖垮整个 CPU 的中断响应。所以挪到下半部——而下半部又分多种是
   因为"重活"也有差别：要睡眠用线程化/工作队列（07/09），不睡但要快的热点用软中断
   （08），从 NMI 升级用 irq_work（09）。不同需求不同工具（Topic 09 决策矩阵）。
4. 抽象让**驱动和核心代码完全不关心**底下是 APIC 还是 GIC——`request_irq`/flow handler
   两边一模一样，只有 `irq_chip`/`domain` 实现不同。好处：① 一套驱动代码跑所有架构；
   ② 硬件翻新（GICv3→v5）上层不动；③ 新架构只需实现 chip+domain 接口就能接入。
   各写一套则要为每个架构重写整个中断栈，不可维护。

---

## 10. 学完自测（系列入门检验）

1. 中断和异常/系统调用的根本区别？"异步"为什么是关键词？
2. 默写端到端脊柱：从设备拉线到应用被唤醒，经过哪些阶段、各属哪个 Topic？
3. hwirq 和 virq 的区别？判别口诀是什么？谁负责翻译？
4. 四个执行上下文各能做什么不能做什么？用哪个词、哪个判断检测？
5. 上半部/下半部分割为什么是"最重要的思想"？x86 vs arm64 的主线差异是什么？

答得出来，你已经有了全局地图——现在从 Topic 01 扩展版开始逐章深入。
读完 Topic 03 记得**回头再读一遍本章**，地图会更清晰。

---

## 附：18 篇扩展文档已全部完成

| Topic | 主题 | 扩展文档 |
|---|---|---|
| 00 | 预备与总览 | `topic00-...-ext.md`（本篇）|
| 01 | CPU 层入口机制 | `topic01-...-ext.md` |
| 02 | 中断控制器 | `topic02-...-ext.md` |
| 03 | genirq 数据模型 | `topic03-...-ext.md` |
| 04 | IRQ domain | `topic04-...-ext.md` |
| 05 | flow handler | `topic05-...-ext.md` |
| 06 | 注册/管理 handler | `topic06-...-ext.md` |
| 07 | 线程化中断 | `topic07-...-ext.md` |
| 08 | 软中断 | `topic08-...-ext.md` |
| 09 | tasklet/工作队列/irq_work | `topic09-...-ext.md` |
| 10 | 上下文/原子性/preempt_count | `topic10-...-ext.md` |
| 11 | MSI/MSI-X | `topic11-...-ext.md` |
| 12 | 每CPU/IPI/NMI | `topic12-...-ext.md` |
| 13 | 亲和性/均衡/热插拔/矩阵 | `topic13-...-ext.md` |
| 14 | 时间计量/延迟/RT | `topic14-...-ext.md` |
| 15 | 挂起/恢复/唤醒 | `topic15-...-ext.md` |
| 16 | 调试与可观测性 | `topic16-...-ext.md` |
| 17 | 初始化全程 | `topic17-...-ext.md` |

每篇结构一致：**建立画面感 → 逐句拆解原文晦涩点（配真实源码与行号）→ 动手任务
（多数针对你的 x86_64 机器，标注是否需 root）→ 任务参考答案 → 学完自测**。
所有源码引用均在 `~/repo/linux`（v7.1-rc7）核对过。
