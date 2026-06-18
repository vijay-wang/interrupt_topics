# Topic 01 扩展版 — CPU 层面的中断机制：把每句"黑话"掰开讲

> **这份文档怎么用：** 先读第 1 章建立画面感（不看任何代码）；再带着画面去读第 2~6 章，
> 每章对应原文档的一节，把里面晦涩的句子逐条翻译成"人话"并配上 `~/repo/linux` 里的真实源码；
> 最后做第 7 章的动手任务（按顺序做，每个任务都标了需要的时间和权限），第 8 章是任务参考答案。
> 文中所有 `文件:行号` 都基于你机器上 `~/repo/linux`（v7.1-rc7）实际核对过，可以直接打开看。

---

## 1. 先建立画面感：这一章到底在讲什么？

### 1.1 一个比喻：门铃与管家

把一台计算机想象成一栋大宅子：

- **CPU** 是管家，平时埋头干活（执行你的程序）。
- **设备**（网卡、磁盘、键盘）是访客，它们随时可能"有事找管家"。
- **中断**就是门铃。访客不需要管家时刻盯着门口（那叫**轮询 polling**，管家什么活都干不了了），
  而是按一下门铃，管家放下手里的活去应门。

这一章（Topic 01）讲的就是**从"门铃响"到"管家开始正经接待客人"之间发生的所有事情**：

1. 管家先把手头的活做到一个干净的段落（不能写信写到一半笔画断掉）；
2. 在门上挂"请勿打扰"牌子（屏蔽其他门铃，防止应门应到一半又被打断）；
3. 把手头工作的进度记在便签上（保存现场，回来才能继续）;
4. 走一条固定的路线到门口（跳到硬件规定的入口地址）；
5. **搞清楚到底是谁按的门铃**（x86 和 arm64 在这一步分道扬镳，见下）；
6. 在门口的登记簿上签到（通用入口层：告诉 RCU、lockdep 等子系统"我进来了"）；
7. 然后才把客人交给真正的接待流程（Topic 05 的 flow handler）。

### 1.2 x86 和 arm64 的根本区别，一句话版

原文档说 *"On x86, the vector already identifies the interrupt; on arm64, one root handler reads
the controller"*，这句话的画面是：

- **x86 的宅子有 256 个编了号的门铃按钮**。3 号按钮一响，管家不用开门就知道"是 3 号快递员来了"，
  直接查自己的通讯录（`vector_irq[3]`）就能找到对应的处理流程。**硬件已经替你分好类了**——
  这就是原文说的 "the hardware did the demux"（demux = demultiplex，**分发/分流**：
  把混在一根线上的多路信号区分开，搞清楚是哪一路）。
- **arm64 的宅子只有一个门铃**。铃一响，管家只知道"有人来了"，必须走到门口看一眼对讲机屏幕
  （读 GIC 控制器的 `ICC_IAR1_EL1` 寄存器）才知道是谁。**分类工作由软件来做**。

这个区别不是谁好谁坏，而是两种硬件设计哲学，它会一路影响到后面所有 topic
（控制器驱动怎么写、向量怎么分配、亲和性怎么管理）。记住这一对画面，后面读代码会顺畅很多。

### 1.3 这一层软件为什么存在？解决什么问题？

CPU 硬件做的事其实**极其少**：保存最低限度的状态（PC、状态寄存器），然后跳到一个固定地址。
仅此而已。剩下的全是操作系统的责任：

| 问题 | 谁来解决 | 在哪一节讲 |
|---|---|---|
| 跳过来之后，剩下的寄存器谁保存？ | 汇编入口桩（stub） | §3、§4 |
| 当前线程的栈快用完了怎么办？ | 切换到专用 IRQ 栈 | §3.3、§4.2 |
| RCU/调度器/锁检查器怎么知道"现在在中断里"？ | 通用入口层（generic entry） | §5 |
| 到底是哪个设备在叫？ | x86 查 `vector_irq`；arm64 问 GIC | §3.2、§4.2 |
| 处理完怎么安全地回去？ | `irqentry_exit` + `iret`/`eret` | §6 |

**历史背景**：早年这些事每个架构各写一份，写得大同小异但又各有微妙的 bug
（尤其是 RCU 和 lockdep 的进出顺序）。后来内核把公共部分抽成了**通用入口层**
（`include/linux/irq-entry-common.h` + `kernel/entry/`），x86 和 arm64 都从它过——
这就是原文反复出现 "generic entry" 的来历。

---

## 2. 逐句拆解原文 §2："CPU 在中断时做什么（硬件）"

### 晦涩句 ①："Finishes or abandons the in-flight instruction to reach a precise boundary"

> 完成或放弃正在执行的指令，以到达一个**精确的边界**。

**为什么这句难懂**：你可能以为 CPU 一次只执行一条指令。实际上现代 CPU 是**流水线 + 乱序执行**的，
同一时刻可能有几十条指令处于"飞行中"（in-flight）——有的刚取指，有的算到一半，有的等着提交结果。

**画面**：想象一条装配流水线上同时摆着 20 个半成品。这时门铃响了，工人要去应门。
他不能把 20 个半成品随手一扔——回来后就分不清哪个做到哪一步了。他必须选一条**干净的分界线**：
"第 N 件之前的全部做完入库，第 N 件之后的全部退回原料区当没做过"。
这样回来后从第 N 件重新开始，不多不少。

这就是**精确中断（precise interrupt）**：被打断时刻之前的指令效果全部生效，
之后的全部没发生。没有这个保证，"保存现场、回来继续"就无从谈起。
这是硬件给软件的承诺，软件层（也就是本 topic 的全部内容）建立在这个承诺之上。

### 晦涩句 ②："Masks further interrupts of equal/lower priority (so the handler isn't immediately re-entered)"

> 屏蔽同级/更低优先级的后续中断（这样 handler 不会刚进门就被再次打断）。

**画面**：管家应门时在门上挂"请勿打扰"。如果不挂，会发生什么？
门铃响 → 管家走向门口 → 还没到，门铃又响 → 管家又从头走一遍入口流程 → 又响……
入口流程本身被无限套娃，栈很快爆掉。所以硬件在跳转的**同一瞬间**自动屏蔽中断，
这是原子的，软件来不及参与。

注意"请勿打扰"牌子有级别：**更高优先级**的访客（x86 的 NMI、arm64 的 FIQ/pseudo-NMI）
可以无视牌子直接进来——这就是为什么 NMI 需要特殊对待（§3.3 的 IST）。

---

## 3. 逐句拆解原文 §3：x86_64 入口

### 3.1 IDT 和向量

**IDT（Interrupt Descriptor Table，中断描述符表）就是那块有 256 个编号按钮的门铃面板。**
每个"门铃按钮"（向量，vector）对应一个**门（gate）**，门里写着：响了之后跳到哪个地址。

向量空间的布局，配合真实源码看（`arch/x86/include/asm/irq_vectors.h`）：

```c
#define FIRST_EXTERNAL_VECTOR  0x20   /* 32 — 0~31 留给 CPU 异常，设备从 32 开始 */
#define RESCHEDULE_VECTOR      0xfd   /* 253 — "请重新调度"的 IPI，固定在高端 */
#define SPURIOUS_APIC_VECTOR   0xff   /* 255 — 最高位 */
#define NR_VECTORS             256
```

布局像一栋楼：**0~31 层是 CPU 自家用的**（页缺失 #PF、除零、NMI=2……这些是"异常"，
不是设备中断）；**32~255 层租给外部设备和系统服务**，其中顶部几层（0xfd、0xff 等）
是固定住户（重新调度 IPI、APIC 假中断等），中间大段是流动租户——设备来了现分配。

### 晦涩句 ③："Device vectors are a scarce per-CPU resource handed out by the matrix allocator"

> 设备向量是一种**稀缺的、每 CPU 独立的资源**，由矩阵分配器分配。

拆成三个点：

1. **稀缺**：每个 CPU 只有约 200 个可分配的向量格子（256 减去异常和固定系统向量）。
   一台服务器插几块多队列网卡，每个队列要一个向量，很容易逼近上限。
2. **per-CPU**：关键！**每个 CPU 都有自己独立的一份 256 格面板**。CPU0 的 35 号向量
   可以给网卡，CPU1 的 35 号向量可以给磁盘——互不冲突。所以总容量是 200 × CPU 数。
3. **矩阵分配器**：管理这张"CPU × 向量"二维表格的分配器（`kernel/irq/matrix.c`），
   Topic 13 详讲。现在只需要知道：有个管理员在管这张表，防止格子用冲突。

### 晦涩句 ④："an interrupt gate clears IF on entry; a trap gate does not"

> 中断门进入时清掉 IF 标志；陷阱门不清。

**IF（Interrupt Flag）就是 x86 上那块"请勿打扰"牌子**——RFLAGS 寄存器里的一个位，
IF=1 表示"可以打扰我"，IF=0 表示"勿扰"。

IDT 里每个门有个类型字段。**中断门（interrupt gate）**：穿过这扇门的瞬间，硬件自动把 IF 清零
（自动挂上勿扰牌）。**陷阱门（trap gate）**：不动 IF。设备中断全部用中断门，
原因就是 §2 句②讲的防套娃。

### 3.2 `idtentry` 宏机器

### 晦涩句 ⑤：原文贴的 `DEFINE_IDTENTRY_IRQ(common_interrupt)` 到底是个什么东西？

这是全文最劝退的部分之一。`DEFINE_IDTENTRY_IRQ` 是一个**代码生成宏**。
打开 `arch/x86/include/asm/idtentry.h:206`，宏的真身是：

```c
#define DEFINE_IDTENTRY_IRQ(func)                                  \
static void __##func(struct pt_regs *regs, u32 vector);           \
                                                                   \
__visible noinstr void func(struct pt_regs *regs,                  \
                            unsigned long error_code)              \
{                                                                  \
    irqentry_state_t state = irqentry_enter(regs);     /* ① */     \
    u32 vector = (u32)(u8)error_code;                  /* ② */     \
                                                                   \
    kvm_set_cpu_l1tf_flush_l1d();                                  \
    instrumentation_begin();                                       \
    run_irq_on_irqstack_cond(__##func, regs, vector);  /* ③ */     \
    instrumentation_end();                                         \
    irqentry_exit(regs, state);                        /* ④ */     \
}                                                                  \
                                                                   \
static noinline void __##func(struct pt_regs *regs, u32 vector)
```

所以当你在 `arch/x86/kernel/irq.c:326` 看到：

```c
DEFINE_IDTENTRY_IRQ(common_interrupt)
{
    /* ……原文贴的那段函数体…… */
}
```

预处理器展开后实际生成了**两个函数**：

- 外层 `common_interrupt()`：宏自动生成的"三明治面包"——先 `irqentry_enter()`（①，
  §5 的通用入口签到），再切到 IRQ 栈（③），最后 `irqentry_exit()`（④，签退）。
- 内层 `__common_interrupt()`：你写在花括号里的代码（"三明治的肉"）。
  注意宏的最后一行没有分号没有函数体——你写的 `{...}` 正好接在它后面，成为内层函数的函数体。

**为什么要搞这么绕？** 因为 x86 有几十个中断/异常入口，每个都需要一模一样的
"签到→干活→签退"三明治。手写几十遍必然有人漏一步（历史上真的出过这种 bug），
宏保证了**结构上不可能漏**。

还有个隐藏彩蛋（②那行）：`u32 vector = (u32)(u8)error_code;` ——
CPU 异常会往栈上压一个"错误码"，而设备中断没有错误码，于是汇编桩**把向量号塞进了
错误码的位置**，借这个现成的通道把"是几号门铃"传给 C 代码。这是个聪明的复用。

### 晦涩句 ⑥："vector → `vector_irq[vector]` → `irq_desc` → flow handler"（demux 的真身）

`vector_irq` 就是管家的**通讯录**。看 `arch/x86/include/asm/hw_irq.h:130`：

```c
typedef struct irq_desc* vector_irq_t[NR_VECTORS];   /* 256 个指针的数组 */
DECLARE_PER_CPU(vector_irq_t, vector_irq);           /* 每个 CPU 一份！ */
```

每个 CPU 有一张 256 项的表，第 N 项指向"N 号门铃对应的客户档案"（`irq_desc`，Topic 03 详讲）。
查表的动作就在 `arch/x86/kernel/irq.c:281`：

```c
static __always_inline bool call_irq_handler(int vector, struct pt_regs *regs)
{
    struct irq_desc *desc = __this_cpu_read(vector_irq[vector]);  /* 查通讯录 */

    if (likely(!IS_ERR_OR_NULL(desc))) {
        handle_irq(desc, regs);          /* 找到档案，进入正式接待流程 */
        return true;
    }
    ...
```

一次数组索引，完事。这就是"x86 的 demux 在硬件做"的软件侧代价：**几乎为零**。

### 晦涩句 ⑦："stale/unassigned vector → re-check under vector_lock, else EOI & drop"

> 过期/未分配的向量 → 持有 vector_lock 重查一次，否则 EOI 并丢弃。

这句话压缩了一个**并发竞争的故事**。`arch/x86/kernel/irq.c:290` 的注释原原本本画了时序图，
我把它翻译成剧情：

> 设备 X 用着 CPU1 的 35 号门铃。某一刻：
> 1. 设备 X 按下门铃（中断已挂起在 APIC 里），但 CPU1 还没来得及应门；
> 2. 与此同时另一个 CPU 上有人调用 `free_irq()` 注销了设备 X——通讯录第 35 项被写成
>    `VECTOR_SHUTDOWN`（"该住户已搬走"）；
> 3. 紧接着又有人 `request_irq()` 注册新设备 Y，恰好也分到 35 号——通讯录第 35 项
>    正在被改写成 Y 的档案；
> 4. CPU1 这才去应门，一查通讯录——**取决于查的瞬间**，可能看到旧值、空值或新值。

处理办法（`irq.c:311`）：第一眼没查到有效档案时，**拿住 `vector_lock`（通讯录的编辑锁）
再查一次**——锁保证没人正在改写。还查不到，说明真是无主中断：回一个 EOI
（End Of Interrupt，对 APIC 说"这事我知道了"，否则 APIC 会一直等）然后丢弃。

`hw_irq.h:126` 里三个特殊标记就是通讯录格子的三种"非住户"状态：

```c
#define VECTOR_UNUSED       NULL          /* 空格子 */
#define VECTOR_SHUTDOWN     ((void *)-1L) /* 住户刚搬走 */
#define VECTOR_RETRIGGERED  ((void *)-2L) /* 搬家过程中铃响过，待补送 */
```

### 3.3 IRQ 栈与 IST

### 晦涩句 ⑧："Kernel thread stacks are small (often 16 KiB). A deeply nested call chain that then takes an interrupt could overflow."

先看真实数字（`arch/x86/include/asm/page_64_types.h:15`）：

```c
#define THREAD_SIZE_ORDER  (2 + KASAN_STACK_ORDER)   /* 2^2 = 4 页 */
#define THREAD_SIZE  (PAGE_SIZE << THREAD_SIZE_ORDER) /* 4KB × 4 = 16KB */
#define IRQ_STACK_SIZE (PAGE_SIZE << IRQ_STACK_ORDER) /* IRQ 栈也是 16KB */
```

**画面**：内核里每个线程的栈只有 16KB——一张便签纸大小（用户态线程栈动辄 8MB，差 500 倍）。
现在想象某个线程正在执行一条很深的调用链（文件系统 → 块层 → 驱动，已用掉 14KB），
这时中断来了。如果中断处理直接压在当前栈上，再借走 3KB——**爆了**。
而中断什么时候来、来时谁在跑，完全随机，等于"任何线程随时可能被借走几 KB 栈"，
所有内核代码都得为最坏情况留余量，太浪费。

**解法**：每个 CPU 准备一块专用的 16KB **IRQ 栈**。中断进来，第一件事就是把栈指针
切到这块专用区域。线程栈从此不用为中断留余量。

### 晦涩句 ⑨："switched … but only if not already on it (nested IRQs reuse the stack)"

> 只在"还不在 IRQ 栈上"时才切换（嵌套中断复用同一块栈）。

为什么要判断？因为中断处理中某些窗口会重新开中断，可能发生**中断嵌套**：
处理 A 时来了 B。如果 B 也无脑"切到 IRQ 栈顶部"，会把 A 的现场覆盖掉！
所以规则是：**已经在 IRQ 栈上就原地继续压栈**，像同一张纸往下接着写，而不是换一张新纸从头写。
x86 的实现在 `arch/x86/include/asm/irq_stack.h:196`（`run_irq_on_irqstack_cond`，
名字里的 `_cond` 就是"有条件地"），arm64 的等价判断是 `entry-common.c:157` 的
`if (on_thread_stack())`。

### 晦涩句 ⑩："Some exceptions need a guaranteed-good stack regardless of the current one"（IST）

> 有些异常需要一块**保证可用**的栈，不管当前栈是什么状态。

上面的 IRQ 栈方案有个前提：**切栈的代码本身需要当前栈还能用**。但有两类灾难恰恰破坏这个前提：

- **双重错误（#DF）**：常见原因就是栈溢出——栈已经坏了，你还想往栈上保存现场？越救越糟。
- **NMI**：不可屏蔽，可能在**任何瞬间**到来，包括"栈指针正在切换、瞬间值非法"的窗口。

**画面**：消防通道。平时的楼梯（线程栈/IRQ 栈）再宽，火灾时可能恰好被堵死。
所以消防法规要求一条**永远畅通的独立通道**。x86 的 **IST（Interrupt Stack Table）**
就是消防通道：在 IDT 门里标记"这类异常一律切到第 N 号预留栈"，由**硬件**完成切换，
不依赖软件、不依赖当前栈的死活。`arch/x86/include/asm/cpu_entry_area.h` 里的
`ESTACK_DF/NMI/DB/MCE/VC` 就是五条消防通道。普通设备中断**不用** IST——
消防通道是给灾难用的，平时走它反而有问题（IST 不支持嵌套）。

---

## 4. 逐句拆解原文 §4：arm64 入口

### 4.1 异常向量表

### 晦涩句 ⑪："a 16-entry table = 4 exception sources × 4 origins"

x86 的门铃面板有 256 个按钮、按"是谁"分类；arm64 的面板只有 **16 个格子**，
按"**什么事 × 从哪来**"分类——一个 4×4 的矩阵（`arch/arm64/kernel/entry.S:517`）：

```
                 │ sync     irq      fiq      error
─────────────────┼─────────────────────────────────
EL1t (内核,SP_EL0)│  ...      ...      ...      ...
EL1h (内核,SP_EL1)│  ...    el1h_64_irq  ...    ...   ← 内核态被设备中断打断走这格
EL0 64位 (用户态) │  ...    el0t_64_irq  ...    ...   ← 用户态被打断走这格
EL0 32位 (兼容)   │  ...      ...      ...      ...
```

**四列（什么事）**：sync = 同步异常（页缺失、系统调用等，是当前指令"自己惹的事"）；
irq = 普通设备中断；fiq = 快速中断；error = SError（系统级错误，如总线错误）。

**四行（从哪来）**：被打断时 CPU 处于什么状态。EL0 = 用户态（64 位和 32 位程序各一行）；
EL1 = 内核态，又分 **t** 和 **h** 两种——这是下一句晦涩点。

### 晦涩句 ⑫："same-EL with SP_EL0 (`t`), same-EL with SP_ELx (`h`)" —— EL1t 和 EL1h 是什么鬼？

arm64 的 CPU 在 EL1（内核态）时可以选用两个栈指针之一：`SP_EL0` 或 `SP_EL1`。
用哪个栈指针，决定了异常落到 t 行还是 h 行：

- **t** = thread 模式（用 SP_EL0）
- **h** = handler 模式（用 SP_EL1）

**你现在只需要知道**：Linux 内核正常运行时用 SP_EL1，所以**内核态收到的中断
几乎都落在 `el1h_64_irq` 这一格**；EL1t 那一行正常情况下不该被走到，走到了说明出了怪事
（内核把它们接到报错处理上）。这个 t/h 区分是架构提供的机制，Linux 只是没怎么用它而已——
读代码时看到 `el1t` 系列函数直接 panic，不要困惑，那是"此路不通"的告示牌。

### 晦涩句 ⑬："IRQs are asynchronous so they don't use ESR — they go straight to the controller"

> IRQ 是异步的，所以不用 ESR——直接去问控制器。

**同步异常**（页缺失、非法指令）是当前指令自己造成的，CPU 自己最清楚原因，
所以把原因写进 `ESR_EL1`（Exception Syndrome Register，"症状寄存器"），
把出事地址写进 `FAR_EL1`。处理代码读这两个寄存器就知道发生了什么。

**异步中断**是外部设备干的，**CPU 根本不知道是谁**——知道这件事的是中断控制器（GIC）。
所以 IRQ 路径不读 ESR（里面没有有用信息），而是去读 GIC 的寄存器。
一句话：**同步异常问 CPU 自己，异步中断问门卫（GIC）**。

### 4.2 从向量到 C handler

### 晦涩句 ⑭："`handle_arch_irq` is a function pointer the root interrupt controller registered at init"

**为什么是函数指针，而不是直接调用？** 因为同一个 arm64 内核镜像要跑在成百上千种 SoC 上，
不同 SoC 用不同的中断控制器（GICv2/GICv3/各种厂商魔改）。内核启动时探测到实际硬件，
由对应的驱动把自己的处理函数**注册**进来。GICv3 的注册点在
`drivers/irqchip/irq-gic-v3.c:2045`：

```c
set_handle_irq(gic_handle_irq);    /* "门口对讲机的使用说明，由门卫厂商提供" */
```

**画面**：宅子的入口流程是固定的（entry.S），但"怎么问出访客身份"取决于装的是哪家的
对讲机。装修完（启动初始化，Topic 17），对讲机厂商把使用手册（函数指针）插进固定的插槽。

### 晦涩句 ⑮："reads `ICC_IAR1_EL1` to learn the hwirq, then calls `generic_handle_domain_irq` — that is the demux x86 did in hardware"

这是 arm64 软件 demux 的现场。看 `drivers/irqchip/irq-gic-v3.c:855`：

```c
static void __gic_handle_irq_from_irqson(struct pt_regs *regs)
{
    ...
    irqnr = gic_read_iar();        /* ★ 问对讲机："刚才是谁按的铃？" */
    ...                            /*   读 ICC_IAR1_EL1，同时表示"我接手了" */
}

static void __gic_handle_irq(u32 irqnr, struct pt_regs *regs)
{
    ...
    gic_complete_ack(irqnr);                              /* 给 GIC 回执 */
    if (generic_handle_domain_irq(gic_data.domain, irqnr)) /* ★ hwirq→virq 查表，
                                                              进入通用中断核心 */
        ...
}
```

对照记忆：**x86 的 demux 是 `vector_irq[vector]` 一次数组索引（硬件已给出编号）；
arm64 的 demux 是先读一次 GIC 寄存器拿到 hwirq 编号，再经 irq domain（Topic 04）查表。**
两条路最终汇合到同一个地方：通用中断核心 `kernel/irq/`。

### 晦涩句 ⑯：pseudo-NMI 是什么？

> arm64 历史上没有 x86 那样独立的 NMI 线（NMI = Non-Maskable Interrupt，
> 不可屏蔽中断——连"请勿打扰"牌子都拦不住的最高级警报，用于看门狗、性能采样等
> "就算内核卡死也必须响应"的场景）。

arm64 的变通方案：GIC 支持中断优先级，而且**优先级屏蔽（PMR 寄存器）和总开关（DAIF）
是两套机制**。于是内核玩了个花招——平时"关中断"不再真关总开关，而是用 PMR 把
普通优先级全部屏蔽、唯独放行一个超高优先级。这个超高优先级的中断就成了**伪 NMI
（pseudo-NMI）**：在"关中断"区域内照样能进来。`entry-common.c:518` 的
`el1_interrupt` 里那个 `IS_ENABLED(CONFIG_ARM64_PSEUDO_NMI)` 分支就是在区分这两条路。
细节在 Topic 12，现在有这个概念即可。

---

## 5. 逐句拆解原文 §5：通用入口/出口层

这是全文最抽象的一节，因为它服务的对象（RCU、lockdep、context tracking）你可能都还不熟。
先给每个"服务对象"一个一句话画像，再讲为什么它们需要入口钩子。

### 晦涩句 ⑰："tell RCU the CPU is no longer idle/quiescent"

> 告诉 RCU 这个 CPU 不再是空闲/静止状态。

**RCU 的一句话画像**：一种内核同步机制，核心问题是"我想释放这块旧数据，
但怎么知道没有任何 CPU 还在读它？"答案是等所有 CPU 都经过一次"静止点"。
为此 **RCU 维护着一张全体 CPU 的值班表**：哪些 CPU 正在干活（可能在读数据，必须等它），
哪些在睡觉（idle，肯定没在读，不用等）。

**问题来了**：一个在睡觉的 CPU 被中断叫醒，跑了一段 handler 代码——这段代码可能读 RCU
保护的数据！如果值班表上它还登记着"睡觉中"，RCU 就会误判"不用等它"，提前释放数据，
handler 读到已释放的内存——内核崩溃级别的 bug。

**所以**：每次中断入口必须先到值班表签到（"我醒了，开始算我"），出口签退。
这就是 `irqentry_enter()`/`irqentry_exit()` 干的核心事情之一。
而 `arch/x86/kernel/irq.c:331` 那行：

```c
RCU_LOCKDEP_WARN(!rcu_is_watching(), "IRQ failed to wake up RCU");
```

是一道**自检岗哨**：进到 C 函数体时，签到应该已经完成（外层三明治面包做的，见句⑤）；
如果发现没签到（`!rcu_is_watching()`），说明有人绕过了正规入口（通常是自定义入口路径写错了），
立刻报警。原文 Pitfalls 里说的 *"IRQ failed to wake up RCU splat"* 就是这个警报响了。

### 晦涩句 ⑱："update context tracking (for nohz_full)" 和 "maintain lockdep's hardirq on/off state"

两个一句话画像：

- **context tracking / nohz_full**：有些用户（高频交易、实时音频）希望某些 CPU 上
  **连时钟滴答中断都别来打扰**。要做到这点，内核必须精确知道每个 CPU 此刻在用户态还是
  内核态——这本账就叫 context tracking，而中断入口/出口正是账本必须更新的时刻。
- **lockdep**：内核内置的**死锁侦探**，记录"哪些锁曾在中断开/关的状态下被拿过"来推断
  潜在死锁。它的推理依赖准确知道"现在中断是开是关"，所以入口处硬件自动关中断这件事，
  必须有人通知 lockdep（硬件不会通知，软件得补这一笔账）。

**§5 的中心思想用一句话总结**：内核里好几个子系统的正确性都依赖一本"CPU 进出内核"的精确账本，
通用入口层就是**所有人共用的记账处**——每次中断必经此地，账就不会错。
原文说 *"every interrupt passes through one common gate that keeps RCU, context-tracking,
lockdep, and preemption honest"*，honest 在这里就是"账实相符"。

代码层面，账本接口就是（`include/linux/irq-entry-common.h:576`）：

```c
irqentry_state_t noinstr irqentry_enter(struct pt_regs *regs);  /* 签到，返回一张回执 */
void irqentry_exit(struct pt_regs *regs, irqentry_state_t state); /* 凭回执签退 */
```

那个 `irqentry_state_t`（回执）记着"签到时改了哪些账"，签退时按回执原样撤销——
因为中断可能嵌套，每层只能撤自己改的那部分。

### 晦涩句 ⑲：`noinstr` 是什么？

你会在所有入口函数上看到 `noinstr` 标记（如句⑤宏展开里的外层函数）。
它的意思是 **no instrumentation——禁止任何插桩**（ftrace、KASAN、kcov 等一律不准碰这个函数）。

**为什么**：插桩代码自己可能用到 RCU 或触发缺页，而入口函数运行在"RCU 还没签到"的危险窗口里——
让追踪器在这个窗口里跑，等于让记账员在账本打开之前花钱。所以从入口到 `irqentry_enter()`
完成之间的代码必须"绝对干净"。宏展开里的 `instrumentation_begin()/end()` 就是在标记
"从这里开始可以插桩了/到这里为止"。**这个知识点在任务 3 里你会亲手撞上**。

---

## 6. 逐句拆解原文 §6：中断返回

### 晦涩句 ⑳："if preemption is enabled and need_resched is set, call the preemption path"

**画面**：管家接待完客人，准备回去继续之前的活。但门口贴了一张新便条（`need_resched` 标志）：
"有更要紧的活等着干"。**中断返回点是内核检查这张便条的黄金时机**——
事实上，"时钟中断返回时发现该换人了"正是抢占式多任务的实现基础：
正因为每个 CPU 隔几毫秒必然进一次时钟中断，内核才有机会强制收回 CPU 使用权。
一个死循环的程序不会主动让出 CPU，但它挡不住中断。

### 晦涩句 ㉑："Pending softirqs … are run earlier, in `irq_exit_rcu()` (the hardirq tail) — that is the seam where Topic 08 attaches"

> 待处理的软中断在更早的 `irq_exit_rcu()`（硬中断收尾处）运行——那是 Topic 08 接入的"缝"。

**背景**：中断 handler 必须快（它霸占着 CPU 且常常屏蔽着其他中断）。所以内核的策略是
**handler 里只做紧急的事，把耗时的活记在待办清单上**（"raise softirq"），出门时再处理。

"seam（缝）"是作者的比喻：两个 topic 的知识在代码里的**衔接点**。具体说：
arm64 路径里 `__el1_irq()`（`entry-common.c:505`）的结构是：

```c
irq_enter_rcu();
do_interrupt_handler(regs, handler);   /* 硬中断 handler，Topic 01~05 的内容 */
irq_exit_rcu();                        /* ← 这就是"缝"：里面会顺手把软中断清掉，
                                            Topic 08 从这里开讲 */
```

读到 Topic 08 时回到这一行，你就知道软中断是从哪里"长出来"的。

---

## 7. 动手任务

> 按顺序做。任务 0~2 零风险；任务 3 需要 root；任务 4 是纯思考题。
> 做完每个任务再对照第 8 章的参考答案。

### 任务 0：认识你机器上的"门铃面板"（10 分钟，无需 root）

你的机器是 x86_64，正好走本文的 x86 路径。

```bash
cat /proc/interrupts | head -25     # 看前 25 行
cat /proc/interrupts | tail -20     # 看最后 20 行
```

回答以下问题（答案见 §8.0）：

1. 最左边一列的数字/缩写是什么？它是"向量号"吗？
2. 找到 `LOC`（Local timer）这一行。运行
   `watch -n1 'grep -E "LOC|RES" /proc/interrupts'` 观察 10 秒，
   哪个 CPU 的计数涨得最快？为什么每个 CPU 的 LOC 都在涨，而普通设备中断往往只有一列在涨？
3. 找到 `RES`（Rescheduling interrupts）。对照 §3.1 的 `RESCHEDULE_VECTOR 0xfd`，
   说出它是干什么的、谁发给谁。
4. 底部的 `NMI`、`MCE` 行：它们会出现在 0~31 还是 32~255 的向量区间？

### 任务 1：徒手追一条 x86 中断入口链（30~45 分钟，只读源码）

目标：把 §3 的整条链在真实源码里走一遍，每一跳都亲眼看到。起点和路标：

```
① arch/x86/kernel/irq.c:326        DEFINE_IDTENTRY_IRQ(common_interrupt)
② arch/x86/include/asm/idtentry.h:206  宏定义（对照句⑤手工展开一遍）
③ arch/x86/kernel/irq.c:281        call_irq_handler() —— 读 :290 的 race 注释
④ arch/x86/include/asm/hw_irq.h:126    VECTOR_* 三个特殊值
⑤ arch/x86/include/asm/irq_stack.h:196 run_irq_on_irqstack_cond
```

边走边回答（答案见 §8.1）：

1. `common_interrupt` 的第二个参数叫 `error_code`，但设备中断没有错误码。
   这个参数里实际装的是什么？在宏的哪一行被取出来？
2. `call_irq_handler` 里 `__this_cpu_read(vector_irq[vector])` ——
   为什么是 `__this_cpu_read` 而不是普通的全局数组读取？如果两个 CPU 同时各自收到
   自己的 35 号向量，会互相干扰吗？
3. 在 `irq_stack.h` 里找到 `run_irq_on_irqstack_cond` 最终展开用到的判断逻辑，
   它怎么知道"当前已经在 IRQ 栈上"？（提示：跟着宏找到 `call_on_irqstack_cond`，
   看它检查了什么。）
4. 假设 `vector_irq[vector]` 读出来是 `VECTOR_SHUTDOWN`，完整说出接下来发生的事，
   直到这次中断被处置完毕。

### 任务 2：对照阅读 arm64 路径，画出双路汇合图（30 分钟，只读源码）

依次打开：

```
① arch/arm64/kernel/entry.S:517            向量表本体（数一数是不是 16 项）
② arch/arm64/kernel/entry-common.c:529     el1h_64_irq_handler
③ arch/arm64/kernel/entry-common.c:505     __el1_irq
④ arch/arm64/kernel/entry-common.c:152     do_interrupt_handler
⑤ drivers/irqchip/irq-gic-v3.c:915         gic_handle_irq
⑥ drivers/irqchip/irq-gic-v3.c:855         __gic_handle_irq_from_irqson（找到读 IAR 的那行）
⑦ drivers/irqchip/irq-gic-v3.c:2045        set_handle_irq(gic_handle_irq)
```

然后**在纸上**（或一个新 markdown 文件里）画出这张图并填空，两条路各自走到哪一步获得了
"这是几号中断"这个信息（答案见 §8.2）：

```
x86:   APIC 送来向量号 ──► common_interrupt ──► [填空A] ──► irq_desc ──► handle_irq
arm64: GIC 拉起 IRQ 线 ──► el1h_64_irq ──► __el1_irq ──► gic_handle_irq ──► [填空B] ──► generic_handle_domain_irq
```

附加题：`__el1_irq` 里 `irq_enter_rcu()` 和 `arm64_enter_from_kernel_mode()` 都出现了，
x86 的宏展开里却只看到 `irqentry_enter()`——x86 的 `irq_enter_rcu` 等价物藏在哪里？
（提示：去 `irq_stack.h` 的 `run_irq_on_irqstack_cond` 宏附近找，或者 grep
`DEFINE_IDTENTRY_SYSVEC` 对比普通 IRQ 和系统向量的不同包装。不要求完全搞清，
能发现"x86 把它藏进了更深的宏"即可。）

### 任务 3：用 ftrace 亲手抓一次中断入口（20 分钟，需要 root）⭐ 最重要的任务

这个任务设计了一个**你必然会撞墙的环节**，撞墙本身就是知识点。

```bash
sudo -i
cd /sys/kernel/tracing            # 老内核在 /sys/kernel/debug/tracing

# 第 1 步：尝试追踪 common_interrupt 本体
grep -c . available_filter_functions          # 看看有多少个可追踪函数
grep common_interrupt available_filter_functions
```

**你会发现 `common_interrupt` 本体不在列表里，只有 `__common_interrupt`。**
回到句⑤和句⑲想一想为什么（答案见 §8.3 问 1）。然后继续：

```bash
# 第 2 步：追踪内层函数 __common_interrupt
echo function_graph > current_tracer
echo __common_interrupt > set_graph_function
echo 1 > tracing_on; sleep 1; echo 0 > tracing_on
head -60 trace
```

观察 trace 输出并回答（答案见 §8.3）：

1. 为什么 `common_interrupt` 追踪不到而 `__common_interrupt` 可以？
2. 在输出里找到 `handle_irq` 之后的调用链，你能认出哪个是 flow handler
   （形如 `handle_edge_irq`/`handle_fasteoi_irq`）？记下名字，Topic 05 会正式见到它。
3. 看每次 `__common_interrupt` 的总耗时（行尾的 us 数字），最长的一次花了多久？
   它和"中断 handler 必须快"的要求对得上吗？

```bash
# 第 3 步：换一个视角——用现成的 tracepoint 看向量号
echo nop > current_tracer
echo 1 > events/irq_vectors/enable 2>/dev/null || echo "(老内核可能没有这组事件)"
echo 1 > tracing_on; sleep 1; echo 0 > tracing_on
grep -m 20 vector= trace
echo 0 > events/irq_vectors/enable
```

4. 输出里 `local_timer_entry: vector=236` 之类的行——这个 vector 数字落在 §3.1
   布局图的哪一段？和 `/proc/interrupts` 里的 `LOC` 行是什么关系？

### 任务 4：思考题（不动手，写下你的答案再看 §8.4）

1. 如果完全去掉 IRQ 栈，让中断直接用被打断线程的栈，内核还能正确运行吗？
   会立刻崩溃，还是"通常没事、偶尔炸"？哪种更可怕？
2. NMI 为什么不能像普通中断一样靠"进门自动挂勿扰牌"来防止重入？
   （提示：NMI 的 N 是什么意思。）
3. x86 把 demux 做在硬件（256 向量），arm64 做在软件（读 IAR）。
   设想一台有 4096 个中断源的服务器，两种方案各自的瓶颈在哪里？
   （这道题是 Topic 11 MSI 和 Topic 13 矩阵分配器的预告。）
4. `irqentry_enter()` 返回的 `irqentry_state_t` 为什么必须由调用者保管、
   原样传回 `irqentry_exit()`，而不能存在一个全局变量里？

---

## 8. 任务参考答案

### 8.0 任务 0

1. 最左列是 **Linux IRQ 号（virq）**，不是向量号。同一个设备中断：virq 是内核全局的
   逻辑编号，向量号是某个 CPU 门铃面板上的格子号，两者由 `vector_irq` 表关联
   （x86 上向量号在 `/proc/interrupts` 里不直接显示）。底部的 `NMI`/`LOC`/`RES` 等
   大写缩写行不是设备中断，是 x86 的系统向量统计。
2. `LOC` 是每 CPU 自带的本地 APIC 定时器——**每个 CPU 都有一个自己的**，所以每列都涨；
   普通设备中断的亲和性通常指向一个 CPU（Topic 13），所以往往只有一列涨。
   哪列涨最快取决于哪个 CPU 最忙（忙的 CPU 不进省电状态，滴答更密）。
3. `RES` = 重新调度 IPI（处理器间中断）：CPU A 发现有任务该去 CPU B 上跑时,
   往 CPU B 的 0xfd 号门铃按一下，把 B 从当前工作中唤起去跑调度器。是 CPU 发给 CPU 的。
4. `NMI` 是向量 2，`MCE`（machine check）是向量 18——都在 **0~31 的 CPU 异常区**。
   这正是"它们不是设备 IRQ、不走 `vector_irq` 通讯录"的体现。

### 8.1 任务 1

1. 装的是**向量号**。汇编桩把向量号压在"错误码"的栈位上借道传入；
   宏展开的 `u32 vector = (u32)(u8)error_code;`（`idtentry.h:213`）取出。
   `(u8)` 截断是因为向量本来就只有 8 位。
2. `vector_irq` 是 **per-CPU 变量**——每个 CPU 一份独立的表，`__this_cpu_read`
   读的是"当前这个 CPU 自己那份"。两个 CPU 的 35 号格子是两块不同的内存，互不干扰。
   这正是 §3.1 "per-CPU resource" 的代码落点。
3. `call_on_irqstack_cond` 用当前栈指针和本 CPU 的 IRQ 栈范围做比较（宏里的
   `assert_arg_type`/`irq_stack_in_use` 一带，不同版本实现细节略有出入）：
   已在 IRQ 栈内 → 直接调用函数（继续在本栈压栈）；不在 → 先切栈指针再调用。
   对应句⑨"嵌套复用同一块栈"。
4. `IS_ERR_OR_NULL` 判真（-1L 是 ERR 指针范围）→ 不走快路径 → `lock_vector_lock()` →
   `reevaluate_vector(vector)` 在锁内重读：若已被并发的 `request_irq` 填上新档案，
   正常调用 `handle_irq` 处理；若仍是 `VECTOR_SHUTDOWN`，把格子改写成 `VECTOR_UNUSED`
   并返回 NULL → `call_irq_handler` 返回 false → `common_interrupt` 调 `apic_eoi()`
   给 APIC 回执，中断被静默丢弃（设备已注销，无人认领）。

### 8.2 任务 2

- **填空 A**：`call_irq_handler` 里的 `__this_cpu_read(vector_irq[vector])`——
  拿着硬件给的向量号查 per-CPU 通讯录。x86 在**进入软件之前**就已经知道编号了。
- **填空 B**：`gic_read_iar()`（读 `ICC_IAR1_EL1`）——arm64 直到**软件主动去问 GIC**
  这一刻才知道是几号 hwirq。
- 汇合点：两条路都终结于通用中断核心（x86 `handle_irq` → `desc->handle_irq`；
  arm64 `generic_handle_domain_irq` → 同样到 `desc->handle_irq`）。Topic 03/05 从汇合点继续。
- 附加题：x86 的 `irq_enter_rcu/irq_exit_rcu` 藏在 `run_irq_on_irqstack_cond` 展开后
  实际执行体的外皮里（`irq_stack.h` 中 `ASM_CALL_IRQ` 路径外层的 `irq_enter_rcu()`/
  `irq_exit_rcu()` 调用，可在该文件 :180~:210 区域看到 sysvec/irq 两种包装的对比）。
  结论：**两个架构做的账完全一样，只是 x86 包进了宏，arm64 写成了明文**。

### 8.3 任务 3

1. `common_interrupt` 外层函数标记了 **`noinstr`**（句⑲）：它运行在 RCU 签到完成之前的
   危险窗口，禁止 ftrace 插桩，所以根本没被编译进可追踪函数列表。`__common_interrupt`
   是宏展开的内层函数，运行在 `instrumentation_begin()` 之后的安全区，可以追踪。
   ——你刚才撞的墙，就是 §5 整节内容在工具链上的投影。
2. 典型输出里会看到 `handle_edge_irq`（边沿触发设备）或 `handle_fasteoi_irq`
   （APIC 直通的 fasteoi 流程），这就是 Topic 05 的 flow handler。
3. 典型值是几微秒到几十微秒。如果你看到几百微秒以上的，记下是哪个设备——
   这正是 Topic 14（延迟）和 Topic 16（调试）要处理的问题。
4. 236 (0xec) 落在 32~255 的**系统向量高段**（`LOCAL_TIMER_VECTOR`）。
   它和 `/proc/interrupts` 的 `LOC` 行是同一件事的两个观察窗口：
   tracepoint 给你逐次事件，`/proc/interrupts` 给你累计计数。

### 8.4 任务 4

1. 能跑，而且**通常没事、偶尔炸**——这更可怕。崩溃取决于"恰好在栈最深的线程上来了
   最深的中断嵌套"这种小概率叠加，意味着问题在测试中几乎不出现，上线后随机炸，
   且每次炸的现场都不同（栈溢出会踩坏相邻内存，症状千奇百怪）。IRQ 栈把这个
   "概率性灾难"变成"确定性安全"。
2. NMI 的 N 是 Non-maskable：**"勿扰牌"对它无效**——这是它存在的意义（看门狗就是要在
   中断被屏蔽死的场景下也能进来）。所以防 NMI 重入不能靠硬件屏蔽，x86 用 IST 专用栈 +
   软件标志的复杂舞步来处理嵌套 NMI（Topic 12 详讲）。
3. x86 方案瓶颈在**向量空间**：每 CPU 约 200 个格子，4096 个源必须靠"多 CPU 分摊 +
   矩阵分配器精打细算"（Topic 13），极端时格子不够。arm64 方案瓶颈在**每次中断多一次
   控制器寄存器读**（IAR 读相对慢），但编号空间几乎无限（GIC 支持上千 SPI + LPI 更多）。
   MSI（Topic 11）的出现部分是为了让中断号空间摆脱物理引脚限制。
4. 因为**中断会嵌套**：A 进来签到，处理中 B 进来又签到。如果回执存在全局变量里，
   B 的回执会覆盖 A 的；B 签退时用自己的回执没问题，A 签退时拿到的却是 B 的旧回执，
   账就错了。回执放在各自的栈上（局部变量），天然每层一份——这是"用栈保存嵌套状态"
   的经典手法，和"中断现场保存在栈上"是同一个道理。

---

## 9. 学完自测：你应该能用自己的话回答

1. 用门铃比喻完整讲一遍：x86 上一次网卡中断从 APIC 到 `handle_irq` 的全程。
2. "demux"在 x86 和 arm64 上分别发生在哪一行代码？
3. `DEFINE_IDTENTRY_IRQ` 生成的两个函数各干什么？为什么外层是 `noinstr`？
4. IRQ 栈和 IST 分别解决什么问题？为什么普通中断不用 IST？
5. `irqentry_enter()` 替哪三个子系统记了账？不记会出什么事故？

答得出来，就可以进 Topic 02（中断控制器——APIC 和 GIC 这两个"门卫"本身的故事）。
