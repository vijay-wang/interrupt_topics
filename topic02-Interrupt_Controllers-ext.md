# Topic 02 扩展版 — 中断控制器：门卫本卫的故事

> **用法同 topic01-ext**：第 1 章建画面 → 第 2~6 章逐句拆解原文 → 第 7 章动手任务 →
> 第 8 章答案。所有 `文件:行号` 均在 `~/repo/linux`（v7.1-rc7）核对过。
>
> **承接 topic01 的比喻**：上一章管家（CPU）听到门铃后做了什么；这一章讲**门铃系统本身**——
> 谁把几百个访客的呼叫汇聚成门铃声、谁决定按响哪个 CPU 的门铃。这个角色就是**中断控制器**，
> 也就是 topic01 里反复出现的"门卫"。

---

## 1. 先建立画面感：控制器是干什么的？

### 1.1 没有控制器会怎样？

一台机器有几百个中断源（网卡每个队列、每块磁盘、每个 USB 口……），而 CPU 核心上
**物理中断引脚只有一两根**。几百根线不可能直接焊到 CPU 上。所以必须有个中间设备：
收下所有设备的呼叫，排好队，再通过那一两根线（或一条消息总线）通知 CPU。

**画面**：一栋写字楼的**前台总机**。几百个房间都有呼叫按钮，全部接到前台；
前台小姐姐做六件事——这六件事正是原文 §1 列的清单，逐条翻译：

| 原文 | 人话 | 画面 |
|---|---|---|
| Aggregation 汇聚 | 几百路呼叫并成几路 | 所有房间的线都接到总机 |
| Masking 屏蔽 | 单独静音某个源 | "302 房间的按钮先别理，维修中" |
| Prioritization 优先级 | 同时呼叫时谁先 | 火警永远插队 |
| Routing/affinity 路由 | 送给哪个 CPU | "这单派给 3 号管家" |
| Ack/EOI 应答 | "我收到了/我处理完了"的回执 | 不回执，总机就一直认为这单没人管，**不派下一单** |
| Trigger config 触发方式 | 电平还是边沿（见 1.2） | 房客是"一直按着按钮"还是"按一下就松手" |

内核把所有控制器统一抽象成 `irq_chip`（Topic 03）挂在 `irq_domain`（Topic 04）上。
**本章是在看抽象背后的真实硬件**——先认识几个真门卫，后面看抽象层才不觉得空洞。

### 1.2 全文最重要的一对概念：电平触发 vs 边沿触发

原文说 *"These two words drive the entire flow-handler design in Topic 05"*——不夸张，
这一对词是后面好几章的分水岭，必须在这里建立牢固的画面：

**电平触发（level-triggered）= 一直按着门铃不松手。**
设备有事，就把线拉高，**并且一直拉着**，直到它的事被处理掉（handler 去设备上把状态清了），
线才放下。
- 好处：**绝不丢事件**。门卫睡着了也没关系，醒来时铃还在响。
- 坏处：处理期间必须先把这个源**屏蔽**掉，否则"铃一直响"会让 CPU 刚出来又被拽进去，
  无限循环。这就是 Topic 05 `handle_level_irq` 要做 mask→处理→unmask 三步舞的原因。
- 另一个坏处：如果 handler 没把设备的状态真正清掉（驱动 bug），unmask 之后铃**立刻又响**，
  循环往复——这叫 **screaming interrupt（尖叫中断）**，内核日志里那句著名的
  `irq N: nobody cared` 多半是它（Topic 16）。

**边沿触发（edge-triggered）= 敲一下门就松手。**
设备有事，把线从低拉到高（一个**跳变**），然后就不管了。
- 好处：便宜、干脆，没有"一直占着线"的问题；同一根线还能快速连敲多下表示多个事件。
- 坏处：**敲门瞬间没人听见，这一下就永远丢了**。如果敲门时这个源恰好被屏蔽着，
  跳变发生过就发生过，硬件不会替你记住。所以 Topic 05 的 `handle_edge_irq` 要用
  `IRQS_PENDING` 软件标志补记"屏蔽期间有人敲过门"，`kernel/irq/resend.c` 负责事后补送。

**一句话对照**：电平像"水位高了就报警、水退了才停"（状态）；边沿像"听到一声爆竹"（事件）。
状态可以随时再看一眼，事件错过就没了。

---

## 2. 逐句拆解原文 §2：x86 的控制器们

### 2.1 8259A PIC——爷爷辈门卫

### 晦涩句 ①："Two cascaded 8259A chips → 15 usable lines"

> 两片级联的 8259A → 15 根可用线。

**背景故事**：1981 年的 IBM PC 用一片 8259A，只有 8 根中断线。后来不够用了，
工程师的办法不是换芯片，而是**再加一片，把第二片的输出接到第一片的 2 号输入上**——
这就是**级联（cascade）**：第二片的 8 个访客共享第一片的一个门铃位。
8 + 8 = 16，减去被占用的 2 号线，**15 根可用**。

为什么今天还要提它？因为**遗产编号还活着**：IRQ0 = 定时器、IRQ1 = 键盘、IRQ14/15 = IDE
硬盘……这些数字就是 8259 时代的线号，至今出现在 `/proc/interrupts` 的开头几行。
而且它的编程方式古老得有教学价值——直接往 I/O 端口写字节
（`arch/x86/kernel/i8259.c:68` 附近，满屏 `outb(..., PIC_MASTER_IMR)`）：
对硬件编程，本质就是**往规定的地址写规定的数**，8259 是这件事最裸露的标本。

### 2.2 Local APIC——每个 CPU 的私人信箱

### 晦涩句 ②："It is the CPU's personal interrupt gateway"

**画面**：现代 x86 不再是"一个总机管全楼"，而是**每个管家（CPU 核）随身带一个私人信箱**，
这就是 Local APIC（LAPIC）。所有要给这个 CPU 的中断，最终都投进它的信箱，
由信箱按优先级递给 CPU。原文列的四样东西，逐个翻译：

- **LVT（Local Vector Table，本地向量表）**：信箱**自带的几个本地呼叫源**的配置表——
  APIC 定时器（就是 `/proc/interrupts` 里的 `LOC`！）、温度报警、性能计数器溢出。
  这些源不经过任何外部总机，是信箱自己的功能。
- **ICR（Interrupt Command Register）**：**寄信口**。CPU A 往自己的 ICR 写一条命令，
  就能往 CPU B 的信箱投一封信——这就是 **IPI（处理器间中断）** 的实现（Topic 12）。
  topic01 任务里看到的 `RES`（重新调度）就是这么寄出去的。
- **EOI 寄存器**：**回执口**。写一下表示"刚才那封信处理完了，可以递下一封"。
  topic01 的 `common_interrupt` 里那个 `apic_eoi()` 写的就是它。
- **TPR/PPR（任务/处理器优先级寄存器）**：信箱的"勿扰级别"旋钮——低于此优先级的信先压着。

### 晦涩句 ③："x2APIC is the MSR-based, larger-APIC-ID successor to the MMIO xAPIC"

> x2APIC 是 xAPIC 的继任者：用 MSR 访问、APIC ID 更大。

两个术语：**MMIO**（memory-mapped I/O）= 把硬件寄存器伪装成一段内存地址，读写那段
"内存"就是读写寄存器；**MSR**（model-specific register）= 用专门的 `rdmsr/wrmsr` 指令
访问的 CPU 内部寄存器，比走内存总线快。

翻译：老的 xAPIC 信箱要走"假内存"访问（慢），且信箱编号（APIC ID）只有 8 位——
**最多 255 个 CPU**。服务器核数爆炸后不够了，于是 x2APIC：改用 MSR 指令直接访问（快），
ID 扩到 32 位（百万级 CPU）。你机器上 `dmesg | grep x2apic` 多半能看到它已启用。

### 2.3 I/O APIC——外部设备的总机

### 晦涩句 ④："Its redirection table entry per pin selects: destination CPU(s), vector, delivery mode, trigger/polarity, mask bit"

LAPIC 管"递到 CPU 跟前的最后一步"，那外部设备的线接到哪？接到 **I/O APIC**——
它才是接替 8259 的"全楼总机"。它的核心就是一张**重定向表（redirection table）**：
每个输入引脚一行，行里写着"这根线上的呼叫，包装成几号向量、送给哪个 CPU、什么触发方式"。

这张表的"一行"长什么样？不用猜，内核源码里有精确定义
（`arch/x86/include/asm/io_apic.h:58`）：

```c
struct IO_APIC_route_entry {
    union {
        struct {
            u64 vector            : 8,   /* 送达时用几号向量（哪个门铃按钮）*/
                delivery_mode     : 3,
                dest_mode_logical : 1,
                delivery_status   : 1,
                active_low        : 1,   /* 极性：低电平有效？ */
                irr               : 1,
                is_level          : 1,   /* ★ 电平还是边沿 */
                masked            : 1,   /* ★ 屏蔽位 */
                ...
                destid_0_7        : 8;   /* 目的 CPU 的 APIC ID */
        };
        ...
    };
} __attribute__((packed));
```

§1.1 表格里的六件事，全部浓缩在这 64 个 bit 里：`vector`（路由成什么）、`destid`（送给谁）、
`is_level`/`active_low`（触发方式）、`masked`（静音开关）。**读懂这一个结构体，
"控制器到底配置些什么"就再也不抽象了。**

原文最后一句 *"MSI bypasses it"*：MSI 中断（Topic 11）不走 I/O APIC 的物理引脚，
设备直接"写一条消息"投递——所以现代机器上 I/O APIC 只剩下传统设备（PS/2、SATA 等）在用。

---

## 3. 逐句拆解原文 §3：GIC 家族（arm64）

### 晦涩句 ⑤：SGI / PPI / SPI / LPI——"小区门牌号段"

GIC 的中断号（hwirq）空间是**按号段划分功能**的，像小区门牌：

| 类别 | 号段 | 是什么 | 对应 x86 的什么 |
|---|---|---|---|
| **SGI** (Software-Generated) | 0–15 | CPU 之间互相敲门 = IPI | LAPIC 的 ICR 寄信 |
| **PPI** (Private Peripheral) | 16–31 | **每个 CPU 私有**的设备线（每核的 arch timer、PMU）| LAPIC 的 LVT 本地源 |
| **SPI** (Shared Peripheral) | 32–1019 | 普通外设中断，可路由到任意 CPU | I/O APIC 管的外部线 |
| **LPI** (Locality-specific) | 8192+ | **基于消息**的中断，给 MSI 用 | MSI（Topic 11）|

注意 PPI 的"每 CPU 私有"：30 号 PPI 在 CPU0 和 CPU1 上是**两个不同的中断**（各自的本地定时器），
号相同但实体不同——这是 Topic 12 percpu 中断的伏笔。

### 晦涩句 ⑥："the Distributor (global) and per-CPU CPU Interface" / GICv3 的 "Redistributors"

GIC 内部分块，用电力系统比喻：

- **Distributor（分发器）**：**总配电室**，全局唯一。SPI 的使能、优先级、"送给哪个 CPU"
  都在这里配置。
- **CPU Interface（CPU 接口）**：**每户的电表箱**，每个 CPU 一个。ack（读 IAR 取号）、
  EOI（回执）、优先级屏蔽（PMR）都在这。
- **Redistributor（GICv3 新增，再分发器）**：总配电室和电表箱之间**每户加装的配电箱**，
  存放该 CPU 私有的 SGI/PPI 配置和 LPI 配置表指针。v3 把"每 CPU 的杂事"从总配电室
  挪到这里，总配电室只管 SPI。

### 晦涩句 ⑦："System-register CPU interface (ICC_*_EL1) instead of MMIO — faster ack/EOI"

GICv2 时代，CPU 取中断号要读一个 **MMIO 地址**（`GICC_IAR`）——走内存总线，慢，
而且**每次中断都要走一遍**。GICv3 把电表箱直接**焊进 CPU 内部**，变成系统寄存器
（`ICC_IAR1_EL1`），用 `mrs` 指令一条读出——这就是 topic01 任务 2 里
`gic_read_iar()` 读的那个寄存器。**中断路径上最热的一次访问从"跑楼下看电表"
变成"看手边仪表盘"**，这是 v2→v3 最实在的提速。

### 晦涩句 ⑧："The ITS translates an MSI (DeviceID, EventID) write into an LPI"

> ITS 把一次 MSI 写 `(DeviceID, EventID)` 翻译成投递给某 CPU 的 LPI。

**画面：邮局的地址翻译柜台。** MSI 中断的本质是"设备往一个魔法地址写一个数"（Topic 11）。
arm64 上这个魔法地址属于 **ITS（Interrupt Translation Service）**。设备写来的内容相当于
信封上写着"我是 3 号设备（DeviceID），这是我的第 2 类事件（EventID）"；ITS 查它管理的
两张内存中的表（device table → event 映射表），翻译成"LPI 编号 8195，投给 CPU2"。
**翻译表放在普通内存里**（不是芯片内部寄存器），所以 LPI 数量几乎无限——
对照 x86 "每 CPU 只有 ~200 个向量格子"的紧张，这是 topic01 任务 4 思考题 3 的答案来源。

### 晦涩句 ⑨：GICv4 的 "direct injection of virtual LPIs without a hypervisor trap"

虚拟化场景：客户机（VM）的虚拟设备来中断了，传统路径是——物理中断 → 宿主机内核 →
KVM 算出该给哪个 vCPU → **人为往 VM 里注入**一个虚拟中断。每次都要"惊动宿主机"
（hypervisor trap），贵。GICv4 让硬件学会了直接把 LPI 投进**正在运行的 VM**，
宿主机全程不知情。一句话：**给虚拟机的信不再经过房东转交，直接塞进租客门缝**。
不做虚拟化可以先跳过。

### 晦涩句 ⑩：GICv5 "the IRS and a redesigned ITS replace the v3 distributor/redistributor split"

只需要记两点：① 这棵源码树里 GICv5 驱动**已经存在**（`drivers/irqchip/irq-gic-v5.c`
及 `-irs.c/-its.c/-iwb.c`，可以 `ls` 验证）；② 无论 GIC 怎么改朝换代，对内核其余部分
**永远表现为 `irq_chip` + `irq_domain`**——抽象层（Topic 03/04）的价值就在于此：
硬件翻新，上层代码不动。

---

## 4. 逐句拆解原文 §4：级联控制器

### 晦涩句 ⑪："A controller's output can be a single line into another controller"

**画面：小区大门的门卫 + 每栋楼的楼栋门禁。** SoC 上 GIC 是大门门卫，但一个 GPIO
控制器下面挂着 32 个引脚，每个引脚都可能产生中断——GPIO 控制器自己就是个小门卫，
它的**总输出**只占 GIC 的**一根** SPI 线。GIC 只知道"GPIO 那栋楼有人叫"，
具体是 32 个引脚里的哪个，要问楼栋门禁（读 GPIO 控制器的状态寄存器）。
这和 8259 级联是同一个思想，而且**和 topic01 的 x86/arm64 之争同构**：
又一层"软件 demux"。

### 晦涩句 ⑫：两种接法——"chained handler" vs "plain request_irq demux"

楼栋门禁的中断处理代码怎么挂到 GIC 那根线上？两种方式：

1. **链式处理器（chained handler）**，`kernel/irq/chip.c:1025`
   `irq_set_chained_handler_and_data()`：把父中断的 flow handler **整个换成**子控制器的
   分发函数。中断来了直接进分发函数 → 读子控制器状态寄存器 → 对真正出事的子 hwirq 调
   `generic_handle_domain_irq()`。**绕过了**父中断的 irqaction 机制（没有统计、没有线程化、
   不可共享）——轻，但裸奔。
2. **普通 `request_irq`**：子控制器像普通驱动一样注册 handler，在 handler 里自己分发。
   走完整的中断处置流程——重，但规矩齐全。

**类比**：① 是"门禁系统直连大门门卫的内线电话"（快但没有登记记录）；
② 是"门禁公司也作为一个普通住户走正常报备流程"。GPIO/pinctrl 这类高频小门卫几乎都用 ①。
无论哪种，子控制器都有**自己的 `irq_domain`**——它的 0~31 号引脚和 GIC 的 0~31 号
是两个世界的编号，不能混（这正是 Topic 04 domain 存在的理由，提前留个钩子）。

---

## 5. 逐句拆解原文 §5：驱动注册胶水

### 晦涩句 ⑬："IRQCHIP_DECLARE(name, compat, init_fn) ties a device-tree compatible string to an init function"

**背景**：arm64 内核镜像是通用的，启动时靠**设备树（DT）**或 ACPI 表得知"这块板子上有
什么硬件"。设备树里中断控制器节点写着 `compatible = "arm,gic-v3"` 之类的身份字符串。

`IRQCHIP_DECLARE` 宏（`include/linux/irqchip.h:42`）做的事：把"身份字符串 → 初始化函数"
这对关系**登记进内核镜像的一张静态表**。真实用例（`drivers/irqchip/irq-gic-v3.c:2271`）：

```c
IRQCHIP_DECLARE(gic_v3, "arm,gic-v3", gic_of_init);
/* 含义："如果设备树里发现 compatible 为 arm,gic-v3 的节点，请调用 gic_of_init" */
```

启动时 `irqchip_init()`（Topic 17）扫描设备树，逐个匹配这张表——**像新生入学报到处**：
每个控制器驱动提前把名牌放在桌上，设备树点到名，对应的初始化函数就被叫起来干活。
干的活就是原文那句链条：**建 domain → 填 irq_chip（mask/unmask/eoi/set_type/set_affinity
几个回调）→ 根控制器额外调 `set_handle_irq()` 把自己挂成 topic01 见过的
`handle_arch_irq`**。

---

## 6. 原文 §7 Pitfalls 的两条重点展开

- **"Wrong trigger type in DT/ACPI"**：设备树声明错触发方式是嵌入式开发第一大坑。
  把边沿写成电平 → 设备敲一下门，控制器以为"线一直拉着"，处理完一看线是低的，
  又没新跳变 → 后续事件**全丢**。把电平写成边沿 → 设备拉着线不放，控制器只在跳变瞬间
  报一次，处理后线还高着却再无跳变 → 也是丢事件；或者反过来狂报 → "nobody cared"。
  对照 §1.2 的两个画面想一遍，这段话就不绕了。
- **"Forgetting EOI semantics differ"**：四代门卫四种回执方式——8259 写端口、LAPIC 写
  EOI 寄存器、GICv2 写 MMIO、GICv3 写系统寄存器。**驱动作者不需要记这些**，因为 flow
  handler 统一调 `irq_chip->irq_eoi`，多态分发（Topic 03 的核心思想）。需要记的人是
  写控制器驱动的人——以及读这份文档的你。

---

## 7. 动手任务（针对你的 x86_64 机器）

### 任务 0：给你机器上的每个中断"验明正身"（15 分钟，无需 root）

```bash
cat /proc/interrupts
```

这次看**右侧的文字**（topic01 看的是左侧数字）。每行形如
`IO-APIC 16-fasteoi`、`PCI-MSI-0000:00:1f.2 ...-edge`。回答（答案 §8.0）：

1. 找出至少一个走 **IO-APIC** 的中断和一个走 **PCI-MSI** 的中断。
   两类设备分别是什么？哪类更"老"？
2. 文字里的 `edge` / `fasteoi` / `level` 是什么？对照 §1.2 说出它们的含义。
3. 运行 `ls /sys/kernel/irq/` 然后挑一个号 N：
   `grep -r . /sys/kernel/irq/N/ 2>/dev/null`——`chip_name`、`hwirq`、`type` 各是什么？
   和 `/proc/interrupts` 该行对得上吗？
4. `dmesg | grep -iE "x2apic|apic"` ——你的机器用 xAPIC 还是 x2APIC？

### 任务 1：读三段控制器源码，见识三代门卫（40 分钟）

```
① arch/x86/kernel/i8259.c:60 附近            爷爷辈：找到 mask 操作是怎么 outb 到
                                              PIC_MASTER_IMR 端口的
② arch/x86/include/asm/io_apic.h:58          叔叔辈：通读 IO_APIC_route_entry，
                                              给每个字段标注 §1.1 表格里的哪件事
③ drivers/irqchip/irq-gic-v3.c               当代（arm64）：找到 gic_mask_irq /
                                              gic_unmask_irq / gic_eoi_irq 三个函数
```

回答（答案 §8.1）：

1. 8259 的 mask 是"往端口写一个 8 位掩码字节"；I/O APIC 的 mask 是结构体里 1 个 bit；
   GICv3 的 mask 是什么操作？（提示：找 `gic_poke_irq`，看它写哪个寄存器——
   `GICD_ICENABLER` 的 IC 是 "Interrupt Clear-enable"。）
2. 三者的 EOI 各自怎么做？（8259: `outb` 0x20；LAPIC: `apic_eoi()`；GICv3: 找
   `gic_eoi_irq` 里写的寄存器名。）
3. 这三套完全不同的操作，最后都被装进同一个结构体的同名回调里——这个结构体叫什么？
   （在 gic-v3.c 里搜 `.irq_mask`，看它挂在什么类型的变量上。）

### 任务 2：找一个真实的级联控制器（20 分钟）

```bash
grep -rn "irq_set_chained_handler" ~/repo/linux/drivers/gpio/ | head -10
```

挑一个文件打开（推荐 `gpio-pca953x.c` 或任意眼熟的），找到它调用
`irq_set_chained_handler_and_data`（或 `devm_request_threaded_irq`）的位置，回答：

1. 它是 §4 的方式 ① 还是方式 ②？
2. 它的分发函数里是怎么"问楼栋门禁是谁敲的门"的？（找读状态寄存器后的循环，
   一般形如 `for_each_set_bit(...) generic_handle_domain_irq(...)`。）

### 任务 3：观察 MSI 绕过 I/O APIC（15 分钟，需要 sudo）

```bash
sudo lspci -v | grep -A2 -B4 "MSI" | head -40
```

1. 找一个 `Capabilities: [..] MSI-X: Enable+` 的设备，再到 `/proc/interrupts`
   里找它的中断行——右侧写的是 `IO-APIC` 还是 `PCI-MSI`？
2. 数一数你机器上 MSI(-X) 中断的总行数 vs IO-APIC 的总行数：
   `grep -c "PCI-MSI" /proc/interrupts; grep -c "IO-APIC" /proc/interrupts`。
   现代机器哪边占绝对多数？这印证了 §2.3 哪句话？

### 任务 4：思考题（答案 §8.4）

1. 电平触发"绝不丢事件"，为什么还要发明边沿触发？反过来，边沿触发更简单，
   为什么定时器、热插拔检测这类信号偏要用电平？
2. 如果一个 handler 处理完忘了 EOI，会发生什么？是"这一个中断再也不来"还是
   "所有中断都不来"？（提示：回执是对"信箱"说的，想想信箱递信的规则。）
3. GPIO 楼栋门禁用 chained handler 直连。如果楼里某个引脚的设备疯狂报警
   （screaming），从 `/proc/interrupts` 看，涨的是 GPIO 子中断的计数还是
   GIC 父中断的计数？这对排查问题意味着什么？
4. x86 为什么不需要 `IRQCHIP_DECLARE` 这种"看设备树点名"的机制？
   （提示：PC 平台的硬件发现靠什么？）

---

## 8. 任务参考答案

### 8.0 任务 0

1. IO-APIC 后面通常是 PS/2 键盘鼠标、SATA(部分)、ACPI、串口等**传统/低速设备**；
   PCI-MSI 后面是 NVMe、网卡、显卡、USB 控制器等**现代 PCIe 设备**。IO-APIC 一类更老
   （走物理引脚），MSI 是消息中断（Topic 11）。
2. 那是**触发方式 + flow handler** 的名字：`edge` = 边沿触发（`handle_edge_irq`）、
   `fasteoi` = 电平/直通式、回执一次即可的流程（`handle_fasteoi_irq`）、
   `level` = 经典电平流程。Topic 05 正式登场。
3. `chip_name` 是 `irq_chip` 的名字（如 `IO-APIC` / `PCI-MSI`），`hwirq` 是控制器
   编号空间里的号（IO-APIC 的引脚号 / MSI 的消息号），`type` 是触发类型——
   三者合起来就是本章讲的"哪个门卫、哪根线、什么触发"。
4. 多数近十年的机器 dmesg 会有 `x2apic enabled`。没有的话你的固件可能禁了它或
   CPU 较老——都正常，xAPIC 模式照样工作。

### 8.1 任务 1

1. GICv3 的 mask：往 Distributor 的 `GICD_ICENABLER` 寄存器对应 bit 写 1
   （"Clear-enable"，写 1 清使能）。注意三代硬件三种风格：写掩码字节 / 改表项 bit /
   写 set-clear 成对寄存器——最后一种的好处是**不用读-改-写**，天然原子。
2. 8259：`outb(0x20, 端口)`（"非特定 EOI"命令字）；LAPIC：写 EOI 寄存器
   （`apic_eoi()`）；GICv3：`gic_eoi_irq` 写系统寄存器 `ICC_EOIR1_EL1`
   （见 `gic_complete_ack`，irq-gic-v3.c:800 附近）。
3. `struct irq_chip`。在 gic-v3.c 里搜 `.irq_mask = gic_mask_irq` 能看到完整的回调表。
   **三代门卫，一张接口**——这就是 Topic 03 的主角。

### 8.2 任务 2

1. 用 `irq_set_chained_handler_and_data` 的是方式 ①；用 `request_threaded_irq`
   注册普通 handler 再分发的是方式 ②（I2C 接口的 GPIO 扩展芯片如 pca953x 因为
   要睡眠读 I2C，只能走 ②+线程化，这是 Topic 07 的伏笔）。
2. 典型形态：读中断状态寄存器得到一个位图 → `for_each_set_bit(bit, &status, ngpio)`
   → 对每个置位的引脚调 `generic_handle_domain_irq(child_domain, bit)`。
   这就是"软件 demux"第三次出现（GIC 读 IAR、x86 查 vector_irq、GPIO 读状态位图）。

### 8.3 任务 3

1. `PCI-MSI`。启用了 MSI-X 的设备不再走 I/O APIC 引脚。
2. 现代机器 MSI 行数通常是 IO-APIC 的几倍到几十倍（多队列网卡/NVMe 每队列一行）。
   印证 §2.3 末句 "This is the chip behind legacy/INTx device IRQs; MSI bypasses it"。

### 8.4 任务 4

1. 边沿的价值：① 不占线，一根线可以传"多次事件"；② 设备侧电路简单，不需要维持状态
   直到被服务；③ MSI 本质上就是边沿语义（一条消息=一个事件），天然契合。
   而定时器/插拔检测表达的是**状态**（"现在到时间了/卡现在插着"），状态天然适合电平——
   而且电平的"绝不丢"对这类不容错过的信号更重要。
2. 取决于控制器，但通常是**灾难性的后者**：LAPIC/GIC 都有"服务中（in-service）"的概念，
   没收到 EOI 就认为你还在处理，**同优先级及更低优先级的中断全部压住不发**——
   表现为"处理了一个中断后整个机器的中断都哑了"。这就是为什么 EOI 被做进 flow handler
   统一管理，而不是信任每个驱动自己写。
3. **两个计数都涨**：父中断（GIC 那根 SPI）每次都先响，子中断计数在分发后增加。
   排查意义：如果只见父涨不见任何子涨，问题在分发/映射（domain 没建好）；
   父子同涨则去看具体设备。Topic 16 的实战技巧。
4. PC 平台硬件靠 **ACPI 表 + PCI 总线枚举**自描述，且核心中断硬件（LAPIC/IOAPIC）
   是架构标准、代代兼容，内核直接内置驱动按 ACPI MADT 表初始化（x86 也有
   `IRQCHIP_ACPI_DECLARE` 的等价物用于 arm64 ACPI 服务器）。设备树是给"硬件不会
   自我介绍"的嵌入式世界准备的。

---

## 9. 学完自测

1. 用"前台总机"比喻说出控制器的六件事，并在 `IO_APIC_route_entry` 里指认出对应字段。
2. 电平 vs 边沿：各自怎么丢事件/怎么防丢？`nobody cared` 通常和哪种有关？
3. 画出 GICv3 的三块结构（Distributor/Redistributor/CPU interface）并标注各管什么。
4. (DeviceID, EventID) → LPI 的翻译者是谁？翻译表放在哪？这和 x86 向量稀缺有什么对比关系？
5. 楼栋门禁（GPIO 控制器）接入 GIC 的两种方式是什么？各自的代价？

答得出来，进 Topic 03——看 Linux 怎么把这一章所有门卫装进四个结构体。
