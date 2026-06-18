# Topic 17 扩展版 — 初始化全程（从开机到中断上线）

> **用法同前**。行号基于 `~/repo/linux`（v7.1-rc7）：`init/main.c` 里
> `early_irq_init():1123`、`init_IRQ():1124`、`softirq_init():1130`、
> `local_irq_enable():1148`；x86 `native_init_IRQ` 在 `arch/x86/kernel/irqinit.c:95`、
> `idt_setup_apic_and_irq_gates` 在 `irqinit.c:105`；arm64 `init_IRQ` 在
> `arch/arm64/kernel/irq.c:110`（`init_irq_stacks:54`、`irqchip_init:114`）；
> `spawn_ksoftirqd` 在 `kernel/softirq.c:1165`；`CPUHP_AP_IRQ_AFFINITY_ONLINE` 在
> `kernel/cpu.c:2197`。
>
> **承接**：这是**收官篇**。前 16 章拆解了中断子系统的每个零件，本章把它们**重新串到
> 启动时间线上**——从第一个 IDT/向量安装，到第一个设备中断触发。读完你能把任何一个
> IRQ 符号定位到启动时刻，并知道"此刻调哪个 API 合法"。

---

## 1. 先建立画面感：开机是一场"按顺序通电"

### 1.1 为什么顺序至关重要

中断子系统不能一下子全部上线——它有**严格的依赖顺序**。就像装修新房通电：
得先有电箱（irq_desc 存储），再接总闸和分线（控制器/domain），再装报警系统
（软中断），最后才能合闸通电（开中断）。**顺序错了 = 短路**——一个在控制器还没建好
时就触发的中断，会让内核"BUG: NULL handle_arch_irq"崩溃。

### 1.2 `start_kernel` 的四个里程碑

`init/main.c` 的关键切片（行号已核对）：

```c
asmlinkage void start_kernel(void)
{
    ...
    early_irq_init();      /* (1) 1123：分配最初的 irq_desc 存储          (Topic 03) */
    init_IRQ();            /* (2) 1124：架构+控制器上线：IDT/向量、GIC、domain (Topic 01/02/04) */
    ...
    softirq_init();        /* (3) 1130：软中断机器                          (Topic 08) */
    ...
    local_irq_enable();    /* (4) 1148：★中断第一次被打开                   (Topic 10) */
    ...
}
```

**四个里程碑，四道门槛**：
- (1) 之前：**没有 irq_desc**（档案室还没建）。
- (2) 之前：**没有控制器、没有 `handle_arch_irq`**（门卫还没上岗）。
- (4) 之前：CPU **一直关着中断**（开机默认状态就是 `local_irq_disable`）。

下面逐个展开。

---

## 2. 逐句拆解最早期设置（init_IRQ 之前）

### 晦涩点 ①：x86 的 IDT 是"分阶段"填的

IDT（Topic 01 §3.1 的 256 门铃面板）**不是一次填满**，而是随着内核"能处理的东西越来越多"
逐步填（`arch/x86/kernel/idt.c`）：

1. **`idt_setup_early_handler`**：最早期——一个兜底处理器，让早期错误能**打印点东西**
   而不是直接三重故障（triple fault，CPU 直接重启）。
2. **`idt_setup_early_traps`**：内存还没起来时就需要的异常门（#DB 调试、#BP 断点）。
3. **`idt_setup_early_pf`**：缺页异常门（一旦分页/内存初始化开始就需要）。
4. **（稍后，在 init_IRQ 里）`idt_setup_apic_and_irq_gates`**：设备/APIC/IPI 向量（Topic 12）。

**关键认知**：x86 上 IDT 的**异常那一半**（CPU 自己的故障）远早于**设备中断那一半**上线。
**画面**：先装好"自家烟雾报警器"（异常），房子能住人了，再接"外部门铃"（设备中断）。

### 晦涩点 ②：arm64 的向量表早早就指好了

`VBAR_EL1` 在早期 CPU 设置（`__primary_switched`/`cpu_setup`）时就指向 `vectors`
（Topic 01 §4.1），所以**同步异常和入口桩在 init_IRQ 之前就能工作**。但 IRQ 那几个槽位
路由到 `handle_arch_irq`——**它在控制器注册之前一直是 NULL**（§3）。
**画面**：门铃线路早就拉好了，但"门铃响了找谁"这张纸还空着，等门卫上岗才填。

### 晦涩点 ③：`early_irq_init` —— 建档案室（里程碑 1）

`kernel/irq/irqdesc.c`：

```c
int __init early_irq_init(void)
{   /* 分配第一批 irq_desc（稀疏 IRQ 树 或 静态数组，Topic 03 晦涩点⑭），
       初始化每个 desc 的 lock/状态，然后 arch_early_irq_init(); */ }
```

这是里程碑 (1)：之后 **`irq_to_desc()` 能用了**，genirq 核心（Topic 03~06）**有地方放
描述符了**。但**还没有任何中断被投递**——档案室建好了，但门卫没上岗、电没合闸。

---

## 3. 逐句拆解 init_IRQ：控制器与根处理器（里程碑 2）

### 晦涩点 ④：x86 —— 装上设备向量门 + APIC 上线

`arch/x86/kernel/irqinit.c:95`：

```c
void __init native_init_IRQ(void)
{   ...
    idt_setup_apic_and_irq_gates();   /* :105 在 IDT 里装设备/IPI/系统向量 */
    /* APIC 上线（apic_intr_mode_init 在别处）：LAPIC、IO-APIC、x2APIC（Topic 02）*/
}
```

之后 IDT 的设备区指向 `common_interrupt`/sysvec 桩（Topic 01 §3.2），APIC/IO-APIC 编程完成。
每 CPU 的 `vector_irq[]` demux（Topic 01）**准备好被填充**——等驱动来 request_irq。

### 晦涩点 ⑤：arm64 —— 建 IRQ 栈 + 探测控制器 + 发布根处理器

`arch/arm64/kernel/irq.c:110`：

```c
void __init init_IRQ(void)
{
    init_irq_stacks();    /* :54 每 CPU 的 IRQ 栈（Topic 01 §5 的便签纸）*/
    ...
    irqchip_init();       /* :114 探测控制器、建 domain、set_handle_irq() */
}
```

`irqchip_init()`（`drivers/irqchip/irqchip.c`）遍历 **`IRQCHIP_DECLARE`** 表（设备树）/
ACPI MADT（Topic 02 §5 的"新生报到处"），运行每个匹配控制器的 init——后者**创建它的
`irq_domain`**（Topic 04），并且对**根控制器**调 **`set_handle_irq(gic_handle_irq)`**
（Topic 02 §3.2）。

**关键时刻**：一旦 `handle_arch_irq` 非 NULL，arm64 的 IRQ 向量槽位**就有去处了**
（晦涩点 ② 那张空纸填上了）。

**里程碑 (2) 之后的状态**：控制器在、domain 在、根处理器发布了、IRQ 栈备好了——
**但 CPU 仍关着中断**。门卫上岗了，电箱接好了，就差合闸。

---

## 4. 逐句拆解首次开中断与启动期消费者（里程碑 3-4）

### 晦涩点 ⑥：`softirq_init` + ksoftirqd（里程碑 3）

`softirq_init()` 建立软中断/tasklet 机器（Topic 08）；`spawn_ksoftirqd()`
（`kernel/softirq.c:1165`，一个 `early_initcall`）通过 smpboot 注册每 CPU 的
**ksoftirqd 线程** + `CPUHP_SOFTIRQ_DEAD` 热插拔状态。**之后** `raise_softirq` 才有意义
（原文 Pitfalls：在里程碑 3 之前 raise 软中断是无意义的）。

### 晦涩点 ⑦：`local_irq_enable` —— 合闸（里程碑 4），第一个中断必是定时器

`local_irq_enable()`（`init/main.c:1148`）是 boot CPU **第一次接受中断**。此时定时器
（经典的"第一个设备中断"）已用 `IRQF_TIMER`（= `NO_SUSPEND | NO_THREAD`，
Topic 06 晦涩点 ⑧）申请好，`tick_init`/clockevent 驱动已武装它。

**第一个设备中断几乎总是定时器滴答**，它走完整条你学过的链：

```
控制器投递 → handle_arch_irq / common_interrupt    (Topic 01)
          → generic_handle_domain_irq               (Topic 04)
          → 定时器的 flow handler（arch timer 用 handle_percpu_devid_irq，Topic 05/12）
          → tick 处理器
          → irq_exit_rcu 跑起被 raise 的软中断       (Topic 08，Topic 01 晦涩句㉑那道缝)
```

**这一刻，你前 16 章学的所有东西第一次合体运转。** 之后的 `initcall` 探测设备，
它们 `request_irq`（Topic 06）填充 `vector_irq[]`（x86）/domain 反查表（arm64）；
MSI domain（Topic 11）随 PCI 总线上线。

---

## 5. 逐句拆解次级 CPU 上线

### 晦涩点 ⑧：每个 CPU 都要"重放"一遍自己的中断部分

boot CPU 之外的每个 CPU 上线时（SMP 启动，以及后来的 CPU 热插拔——**同一条路径**，
Topic 13 §5）：

1. **每 CPU 控制器初始化**：x86 拉起该 CPU 的 Local APIC；arm64 启用它的 GIC
   **redistributor** 和每 CPU 的 PPI/SGI。
2. **每 CPU 中断**（Topic 12）**在该 CPU 上**通过热插拔启动回调使能（arch timer/PMU
   从它们的热插拔状态调 `enable_percpu_irq`——回忆 Topic 12 晦涩点 ② "必须逐 CPU enable"）。
3. **`irq_affinity_online_cpu`** 运行（`CPUHP_AP_IRQ_AFFINITY_ONLINE`，`kernel/cpu.c:2197`，
   Topic 13）：恢复那些现在 CPU 可用了的**托管中断**，重新均衡。
4. 该 CPU 的 **`ksoftirqd/N`** 和任何 **`irq/N-name`** 线程（Topic 07）被创建/绑定。
5. 最后该 CPU 开本地中断，加入中断服务。

**关键认知**：Topic 01/12/13 的每 CPU 部分，**在启动时和每次热插拔上线时都重放一遍**
——启动和热插拔共用一套机器（Topic 13 晦涩点 ⑧ 的回响）。

---

## 6. 逐句拆解时间线总表：什么时候能调什么 API

原文 §6 这张表是**整个系列的"何时合法"速查**，是收官的精华：

| 启动点 | 上线了什么 | 此刻合法调用 |
|---|---|---|
| `early_irq_init` 前 | 啥都没有 | — |
| `early_irq_init` (1) 后 | irq_desc 存储 | `irq_to_desc`、分配描述符（还不能投递）|
| `init_IRQ` (2) 后 | 控制器、domain、根处理器、IRQ 栈 | `irq_domain_*`、`irq_create_mapping`（Topic 04）；芯片已注册 |
| `softirq_init` (3) 后 | 软中断/tasklet | `open_softirq`、`raise_softirq`（Topic 08）|
| `local_irq_enable` (4) 后 | **投递** | 中断真正触发；定时器滴答开始跑 |
| initcall 期间 | 设备探测 | `request_irq`/`pci_alloc_irq_vectors`（Topic 06/11）|
| 每个次级 CPU 上线 | 该 CPU 的控制器+每 CPU 中断 | 在该 CPU 上 `enable_percpu_irq`（Topic 12）|

### 晦涩点 ⑨：启动期那些 "NULL handle_arch_irq" 崩溃的含义

原文最后那句很重要：启动时的 "BUG: Bad ... / NULL `handle_arch_irq`" 崩溃**几乎总是
意味着某个中断在它的里程碑之前就被投递了**——例如：

- 一条线在 `init_IRQ` 之前就没被屏蔽（设备开机就在叫）；
- 一个驱动在它的 domain 还没建好时就 request_irq。

**理解了这张时间线，这类崩溃一眼就能定位"是哪个里程碑被越过了"。**

---

## 7. 动手任务（针对你的 x86_64 机器）

### 任务 0：从 dmesg 重建启动时间线（15 分钟，无需 root）

```bash
dmesg | grep -iE "APIC|IO-APIC|IRQ|GIC|irq_stack|vector" | head -30
dmesg | grep -iE "Booting|smpboot|CPU[0-9]" | head -20    # 次级 CPU 上线
dmesg -T | grep -iE "x2apic|Switched APIC|enabled APIC" | head
```

回答（答案 §8.0）：

1. 从 dmesg 能看到 APIC/IO-APIC 上线的消息吗？它出现在"smp: Bringing up secondary
   CPUs"之前还是之后？对应 §3 哪个里程碑？
2. 找到各个 CPU 上线的消息（"smpboot: Booting Node ... CPU"）——你机器几个 CPU？
   每个上线时是否重放了 §5 的步骤？
3. 第一个被 request_irq 的设备中断大概是什么？（搜定时器/timer 相关。）

### 任务 1：源码走读启动主线（30 分钟，核心）⭐

```
① init/main.c:1123-1148   start_kernel 里四个里程碑的顺序（亲眼确认 early_irq_init →
                          init_IRQ → softirq_init → local_irq_enable）
② kernel/irq/irqdesc.c    early_irq_init：看它分配 irq_desc 存储
③ arch/x86/kernel/irqinit.c:95  native_init_IRQ → idt_setup_apic_and_irq_gates
   （arm64 对应 arch/arm64/kernel/irq.c:110 init_IRQ → irqchip_init）
④ kernel/softirq.c:1165   spawn_ksoftirqd（early_initcall）
⑤ kernel/cpu.c:2197       CPUHP_AP_IRQ_AFFINITY_ONLINE 热插拔状态
```

回答（答案 §8.1）：

1. `local_irq_enable`（里程碑 4）在 `init/main.c` 里的行号是多少？它在
   `softirq_init` 之后吗？为什么开中断必须排在软中断机器建好之后？
2. x86 的 `idt_setup_apic_and_irq_gates` 装的是 IDT 的哪一半（异常还是设备）？
   异常那一半是什么时候装的（§2）？
3. `CPUHP_AP_IRQ_AFFINITY_ONLINE` 在 cpuhotplug 状态机里——它在 CPU 上线流程的
   哪个阶段跑？（AP = application processor，次级 CPU。）

### 任务 2：观察 CPU 热插拔重放每 CPU 中断（25 分钟，需要 root，虚拟机）⭐

> ⚠️ 虚拟机里做，别下线 CPU0。

```bash
# 看每 CPU 中断（如 arch timer/LOC）在所有 CPU 上的计数：
grep LOC /proc/interrupts
# 下线一个 CPU，再上线，观察它的每 CPU 中断重新使能：
echo 0 | sudo tee /sys/devices/system/cpu/cpu2/online
grep LOC /proc/interrupts                     # CPU2 列还涨吗？
echo 1 | sudo tee /sys/devices/system/cpu/cpu2/online
dmesg | tail -10                               # 看 CPU2 重新上线、控制器重新初始化
grep LOC /proc/interrupts                     # CPU2 列恢复涨了吗？
```

回答（答案 §8.2）：

1. 下线 CPU2 后，它的 LOC（本地定时器）计数停止增长了吗？（每 CPU 中断在该 CPU
   下线时被禁用。）
2. 重新上线后，CPU2 的 LOC 恢复增长了吗？这印证 §5 哪个步骤（enable_percpu_irq
   在热插拔启动回调里重放）？
3. dmesg 里有 CPU2 的 APIC/控制器重新初始化的消息吗？

### 任务 3：在启动时间线上定位所有前序 Topic（30 分钟，纸面综合）⭐⭐

这是**整个系列的综合复习**。不写代码，做一张**对照表**：把 Topic 01~16 的核心概念
各自钉到 §6 时间线的某个里程碑上。模板（填完即通关）：

| Topic | 核心概念 | 在哪个里程碑上线/相关 |
|---|---|---|
| 01 CPU 入口 | IDT/向量表、IRQ 栈、通用入口层 | ? |
| 02 控制器 | APIC/GIC、IRQCHIP_DECLARE | ? |
| 03 数据模型 | irq_desc 存储 | ? |
| 04 domain | irq_create_mapping | ? |
| 05 flow handler | desc->handle_irq | ? |
| 06 request_irq | 注册 handler | ? |
| 08 软中断 | open_softirq/ksoftirqd | ? |
| 11 MSI | pci_alloc_irq_vectors | ? |
| 12 每CPU/IPI | enable_percpu_irq | ? |
| 13 亲和性/热插拔 | irq_affinity_online_cpu | ? |

回答见 §8.3。

### 任务 4：思考题（答案 §8.4）

1. 为什么 `local_irq_enable`（开中断）必须排在 `init_IRQ`（建控制器）**和**
   `softirq_init`（建软中断）**之后**？如果在 init_IRQ 前就开中断会怎样？
   在 softirq_init 前开又会怎样？
2. x86 的异常门（#PF 缺页）远早于设备中断门安装。为什么缺页异常必须这么早就能处理？
   （提示：内核自己初始化时会不会缺页？）
3. 一个设备开机时就在拉中断线（固件没清它）。内核启动到 `local_irq_enable` 时会
   发生什么？怎么避免？（联系 Topic 05 nobody cared。）
4. 启动和 CPU 热插拔"共用一套每 CPU 上线机器"。这个设计带来什么好处？
   （提示：代码复用、一致性、测试。）

---

## 8. 任务参考答案

### 8.0 任务 0

1. 能看到 "APIC routing"、"IO-APIC ... entries"、"Switched APIC routing" 等，
   出现在次级 CPU 上线**之前**（init_IRQ 是里程碑 2，远早于 SMP bring-up）。
2. "smpboot: Booting Node 0 Processor N" 每个次级 CPU 一条；每个上线时重放
   APIC 拉起、每 CPU 中断使能等（§5）。
3. 定时器（local timer / clockevent），它是第一个 request 且第一个触发的设备中断。

### 8.1 任务 1

1. `local_irq_enable` 在 `init/main.c:1148`，在 `softirq_init`(1130) 之后。
   开中断必须在软中断机器之后，因为第一个中断（定时器）的尾巴会 `irq_exit_rcu`
   跑软中断（§4 晦涩点 ⑦）——软中断机器没建好就跑会崩。
2. 装的是**设备/APIC/IPI 那一半**。异常那一半（早期 trap、缺页门）在 init_IRQ
   之前的 `idt_setup_early_*` 阶段就装了（§2 晦涩点 ①）。
3. 在 AP（次级 CPU）上线流程的较晚阶段（CPU 即将开始服务中断前），恢复托管亲和性
   中断、重新均衡——是 CPU-up 路径的一环。

### 8.2 任务 2

1. 是，CPU2 下线后它的 LOC 停止增长（每 CPU 定时器在该 CPU 下线时禁用）。
2. 是，重新上线后恢复——印证 §5 步骤 2：arch timer 从热插拔启动回调重新
   `enable_percpu_irq`。
3. 有，dmesg 通常显示 CPU2 重新上线、本地 APIC 重新初始化的消息。

### 8.3 任务 3（综合复习答案）

- 01 → 里程碑 2（init_IRQ：IDT/向量、IRQ 栈）；但通用入口层在更早期就编译进内核，
  异常入口在里程碑 2 之前（§2）。
- 02 → 里程碑 2（init_IRQ/irqchip_init 建控制器）。
- 03 → 里程碑 1（early_irq_init 建 irq_desc 存储）。
- 04 → 里程碑 2 之后（domain 在 irqchip_init 里建，irq_create_mapping 此后合法）。
- 05 → 里程碑 4 之后才真正运行（投递开始）；flow handler 在 request_irq 时安装。
- 06 → initcall 期间（设备探测时 request_irq）。
- 08 → 里程碑 3（softirq_init/ksoftirqd）。
- 11 → initcall 期间，随 PCI 总线（MSI domain 上线）。
- 12 → 每个 CPU 上线时（enable_percpu_irq 逐 CPU）。
- 13 → 每个 CPU 上线时（irq_affinity_online_cpu）+ 运行期。

### 8.4 任务 4

1. 在 init_IRQ 前开中断：没有控制器、`handle_arch_irq` 为 NULL，来个中断直接
   "NULL handle_arch_irq" 崩溃。在 softirq_init 前开：第一个中断的尾巴要跑软中断，
   但软中断机器/ksoftirqd 还没建，崩溃或未定义行为。所以开中断必须排在两者之后
   ——这就是里程碑顺序不可乱的根本原因。
2. 内核自己初始化时会访问尚未建立映射的内存（按需建页表、vmalloc 区等），
   这些会触发缺页异常，由缺页处理器**正常补上映射**。所以缺页处理必须在内存
   初始化**开始前**就能工作——它是内核内存管理正常运转的一部分，不是错误。
3. 启动到 local_irq_enable 时，这条一直拉着的线立刻触发，但如果对应 handler 还没
   注册（驱动还没探测），会被判为无人认领 → 可能很快 "nobody cared" 被禁用
   （Topic 05）。避免：固件/早期代码应在交给内核前**让设备安静**（quiesce），
   或用 `IRQF_NO_AUTOEN`（Topic 06）+ 设备就绪后再 enable。
4. 好处：① 代码复用（一套 CPU-up 路径同时服务启动和热插拔，不用写两份）；
   ② 一致性（启动期次级 CPU 和运行期热插拔 CPU 行为完全相同，不会有"只在某种
   情况下出现"的 bug）；③ 可测试性（热插拔一个 CPU 就等于在运行的系统上重演了
   启动期的 CPU 上线逻辑，便于验证）。

---

## 9. 学完自测（兼整个系列的收官自测）

1. 默写 `start_kernel` 的四个中断里程碑及其顺序，每个之后什么变得合法？
2. x86 IDT 为什么分阶段填？异常半和设备半各在哪个里程碑？arm64 的向量表和
   `handle_arch_irq` 分别什么时候就绪？
3. 第一个设备中断通常是谁？它走完的完整链条经过哪些前序 Topic？
4. 次级 CPU 上线重放哪五个步骤？为什么启动和热插拔共用这套机器？
5. 启动期 "NULL handle_arch_irq" 崩溃说明什么？怎么用时间线定位？

---

## 10. 全系列回顾：你现在掌握了什么

走完 Topic 00~17（扩展版），你应该能：

- **把任意一个中断 IRQ 符号**（`common_interrupt`、`gic_handle_irq`、`handle_fasteoi_irq`、
  `request_threaded_irq`、`pci_alloc_irq_vectors`……）**定位到它在子系统里的位置和启动
  时刻**。
- **追一条中断的完整生命**：硬件拉线/写消息 → CPU 入口（01）→ 控制器 demux（02）→
  domain 翻译（04）→ flow handler（05）→ 你的 handler（06）→ 下半部（07/08/09），
  全程遵守并发规则（10）。
- **诊断真实问题**：用 Topic 16 的手册，从症状（nobody cared / 计数为 0 / ksoftirqd 烫 /
  延迟高 / 唤不醒）定位到原因和修法。
- **理解架构差异**：x86 硬件 demux（向量稀缺、矩阵分配器）vs arm64 软件翻译
  （GIC/ITS、LPI 充裕）——这条主线贯穿 01/02/11/13。
- **理解 PREEMPT_RT**：线程化（07）+ 任务上下文软中断（08）+ 可睡眠锁（10）如何
  把延迟有界化（14）。

**建议下一步**：挑一个真实驱动（你机器的网卡或 NVMe），从它的 `request_irq`/
`pci_alloc_irq_vectors` 出发，对照这 18 篇扩展文档把它的中断路径**完整走一遍源码**
——把"读得懂"变成"讲得出"。
