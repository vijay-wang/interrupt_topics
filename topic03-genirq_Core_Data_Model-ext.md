# Topic 03 扩展版 — genirq 核心数据模型：四个结构体的"户口本"

> **用法同前**：第 1 章画面感 → 中间逐句拆解 → 动手任务 → 答案。
> 行号基于 `~/repo/linux`（v7.1-rc7）：`irq_desc` 在 `include/linux/irqdesc.h:80`、
> `irq_common_data` 在 `include/linux/irq.h:149`、`irq_data` 在 `irq.h:181`、
> `irq_chip` 在 `irq.h:498`、`irqaction` 在 `include/linux/interrupt.h:123`。
>
> **这一章的性质**：前两章讲的是"事件怎么发生"（动词），这一章讲"内核用什么名词来记录
> 这一切"。它是后面所有章节的**词汇表**——没有它，Topic 05/06/07 的每句话都没有主语。

---

## 1. 先建立画面感：为什么需要四个结构体？

### 1.1 一个"客户档案系统"的比喻

继续用宅子比喻。管家（CPU）应门多了，发现必须建一套**档案系统**来管理所有"会按门铃的人"。
对每一路中断，需要记录四类完全不同性质的信息：

1. **这路中断的总档案** —— 它现在禁用了吗？被按了多少次？出过几次"无人认领"？
   用哪套接待流程？→ **`irq_desc`**（描述符，"档案夹"本体）
2. **门卫眼里它是谁** —— 在 GIC/APIC 的编号系统里它是几号线？归哪个门卫管？
   → **`irq_data`**（芯片视角的"工牌"）
3. **门卫的操作手册** —— 想静音这路、想回执、想改触发方式，具体怎么拨弄硬件寄存器？
   → **`irq_chip`**（Topic 02 那些真实硬件操作的**函数指针表**）
4. **认领了这路中断的住户** —— 哪个驱动说"这路铃响了找我"？找谁？怎么找？
   → **`irqaction`**（"认领登记条"）

**为什么不合成一个大结构体？** 因为四类信息的**生命周期和归属者**不同：
档案夹归内核核心管；工牌每过一层门卫就有一张（见 1.2 的惊人事实）；操作手册整个 GIC
共用一份（几百路中断共享同一张函数表）；认领条则随驱动加载/卸载来来去去，
共享中断时一路线上还能挂好几张。强行合并，要么浪费要么混乱。

### 1.2 全章最容易懵的一句话，先在这里讲透

原文 §1 那句 *"`irq_desc` is the core-facing descriptor, while `irq_data` is the
chip-facing descriptor"* 以及后面的 *"Hierarchical domains stack multiple `irq_data`
for one `irq_desc`"*——这是全章的"题眼"：

**画面**：一封挂号信（一路中断）在邮政系统里只有**一份总档案**（`irq_desc`：寄没寄到、
投递了几次、谁签收）。但它途经**好几个网点**（PCI-MSI 层 → 中断重映射层 → APIC 层，
Topic 04/11 的层级 domain），**每个网点给它贴一张自己的条码**（`irq_data`：本网点编号
hwirq、本网点的操作规程 chip）。条码用 `parent_data` 指针串成一串——撕开最外层条码
能看到上一站的。

所以：**一路中断 = 1 个 `irq_desc` + N 张 `irq_data`（N = 途经的控制器层数）+
每层一份共享的 `irq_chip` + 0~多张 `irqaction`。** 把这个等式记住，本章就通了。

判别口诀（原文 Pitfalls 也强调了）：看函数签名——**参数是 `struct irq_data *` 的，
是给门卫（芯片驱动）回调用的，每层一个视角；参数是 `struct irq_desc *` 或 virq 的，
是内核核心的事务**。

---

## 2. 逐句拆解 `irq_desc`（档案夹）的关键字段

打开 `include/linux/irqdesc.h:80` 对照。不逐字段抄一遍（原文已列全），
只拆解几处真正晦涩的：

### 晦涩点 ①：`handle_irq` —— "THE FLOW HANDLER"

档案夹第一页写着**接待流程**：这路中断来了走哪套程序（电平的 mask→处理→unmask？
边沿的 ack→处理？）。Topic 01 里 x86 的 `handle_irq(desc, regs)` 最终就是调
`desc->handle_irq(desc)`（见 `irqdesc.h:184` 的 `generic_handle_irq_desc`）。
注意原文那个提示框：**这棵树里 flow handler 只有一个参数 `desc`**，老书老博客里的
`(irq, desc)` 双参数签名已成历史——读旧资料时留神。

### 晦涩点 ②：`depth` —— "nested disable_irq() depth; 0 ⇒ enabled"

> 嵌套禁用深度；0 表示启用。

**为什么禁用一个中断需要"深度"而不是一个开关位？** 想象代码路径 A 调了
`disable_irq(n)`，干活时调用的子函数 B 也调了 `disable_irq(n)`，B 干完调
`enable_irq(n)`——如果是开关位，**B 的 enable 把 A 还需要的禁用状态破坏了**。
所以用计数器：disable 加一，enable 减一，**减到 0 才真正开闸**。
像会议室门口挂"使用中"的牌子换成了签到表：所有人都签退了才算空闲。
（`wake_depth` 同理，管的是"保持可唤醒"的嵌套计数，Topic 15。）

### 晦涩点 ③：`irq_count` / `irqs_unhandled` / `last_unhandled` —— 假中断侦探的工作台

三个字段合起来是一个**测谎仪**：每来一次中断 `irq_count++`；如果 action 链全员回答
`IRQ_NONE`（"不是我的"），`irqs_unhandled++`。当大约 10 万次里有 9.9 万次没人认领，
内核断定这条线疯了（硬件故障/驱动 bug/触发方式配错——Topic 02 的尖叫中断），
打出著名的 `irq N: nobody cared`，**强制禁用这条线**保命。`last_unhandled` 是时间戳，
防止把"偶尔丢一次"累积成冤案（隔太久就既往不咎）。细节在 Topic 05 的 spurious.c。

### 晦涩点 ④：`lock` 为什么是 `raw_spinlock_t`？

普通 `spinlock_t` 在 PREEMPT_RT 实时内核里会被偷偷替换成**可睡眠的互斥锁**（Topic 14）。
但 `desc->lock` 要在**硬中断上下文**里拿（那里绝不能睡，Topic 10），所以必须用
"raw"前缀声明"我是真自旋锁，RT 也别动我"。推论就是原文 Pitfalls 那条：
**持有 `desc->lock` 期间做任何可能睡眠的事 = bug**。

### 晦涩点 ⑤：`____cacheline_internodealigned_in_smp` 这串咒语

结构体定义末尾那个长后缀的意思：**按缓存行对齐**。两个 CPU 频繁访问的两个 `irq_desc`
如果挤在同一条缓存行里，会发生**伪共享（false sharing）**——硬件层面互相踩缓存，
性能莫名下降。对齐让每个档案夹独占自己的"抽屉"。读内核代码常见此类后缀，认识即可。

---

## 3. 逐句拆解 `irq_data` 与 `irq_common_data`（工牌与公共栏）

### 晦涩点 ⑥：`hwirq` + `chip` + `domain` 三件套

原文说这是 *"this interrupt, as known to this controller level"*。工牌上的三行字：
**我在本网点的编号**（hwirq）、**本网点的操作规程**（chip）、**本网点的编号簿**
（domain，Topic 04 主角）。`irq_chip_mask_parent` 这类 helper 做的事就一行——
拿 `parent_data` 这张上一层工牌，调上一层规程里的同名操作：**"这事我不管，问我上级"**。

### 晦涩点 ⑦：为什么还要再分出一个 `irq_common_data`？

既然 `irq_data` 每层一张，那"亲和性设到哪些 CPU""这是不是 MSI"这类**全线只有一份**的
信息放哪？放任何一层的工牌上都不对。于是单独立了一块**公共信息栏** `irq_common_data`
（`irq.h:149`），嵌在 `irq_desc` 里，每张工牌的 `common` 指针都指回它。
层数再多，公共栏只有一块——原文那句 *"note both live here, once per `irq_desc`,
regardless of hierarchy depth"* 说的就是这个设计。

### 晦涩点 ⑧：`affinity` vs `effective_affinity` —— "你点的菜"和"端上来的菜"

- `affinity`：**你要求的**——"这个中断请绑到 CPU 0-3"。
- `effective_affinity`：**硬件实际做到的**——x86 一个向量一次只能指到一个 CPU，
  你要求 0-3，硬件可能实际落在 CPU2 一个上。

调试"为什么中断都打在 CPU2"时读错这两个文件（`/proc/irq/N/smp_affinity` vs
`effective_affinity`）是 Topic 13 的经典坑，这里先把概念立住。

---

## 4. 逐句拆解 `irq_chip`（操作手册/函数指针表）

### 晦涩点 ⑨："The core never pokes APIC/GIC registers directly; it calls these"

> 内核核心从不直接戳 APIC/GIC 寄存器，它只调这些回调。

这就是面向对象里的**多态/虚函数表（vtable）**，用 C 实现。Topic 02 任务 1 你已亲眼
见过证据：8259 的 mask 是 `outb` 端口、I/O APIC 是改表项 bit、GICv3 是写
`GICD_ICENABLER`——三种实现都叫 `.irq_mask`，核心代码一句
`chip->irq_mask(data)` 通吃。**换硬件不换上层代码**，Topic 02 自测题 5 的官方答案。

### 晦涩点 ⑩：`irq_bus_lock` / `irq_bus_sync_unlock` —— 给"慢门卫"的特殊通道

原文说这是 *"a subtle but important design point"*，值得用画面讲透：

有些门卫**住得很远**——比如挂在 I2C 总线上的 GPIO 扩展芯片（Topic 02 任务 2 见过的
pca953x）。想 mask 它的一路中断，得发 I2C 报文，**而 I2C 通信会睡眠**。
可 mask 通常发生在硬中断上下文/持锁状态——不准睡！（Topic 10 的铁律。）

解法是**记账-延迟生效**：在不能睡的地方，mask 操作只改一个内存里的影子寄存器（记账）；
等回到可以睡的上下文，核心调 `irq_bus_sync_unlock`，驱动这时才把积攒的修改
**一次性通过 I2C 刷到真芯片**。像你在地铁里（不能打电话）先把要办的事记在备忘录，
出站后（可以打了）一个电话全办掉。

### 晦涩点 ⑪：`IRQCHIP_*` flags 是"门卫的脾气备注"

例如 `IRQCHIP_SET_TYPE_MASKED` = "改触发方式前请先把我静音"（有的硬件改配置时会
误发中断）；`IRQCHIP_EOI_THREADED` = "回执请等线程化 handler 干完再发"（Topic 07）。
核心代码在相应时机检查这些位并迁就硬件怪癖。**flags 是声明式的：硬件作者标注脾气，
核心负责伺候**——比每个驱动自己写特殊逻辑可维护得多。

---

## 5. 逐句拆解 `irqaction`（认领登记条）

### 晦涩点 ⑫：共享中断与 `dev_id` 的双重身份

`interrupt.h:123`。一条线可以被多个设备共享（老 PCI 时代的常态）：每个
`request_irq(IRQF_SHARED)` 挂一张登记条，用 `->next` 串成链。铃响时核心**挨个问**：
"是你吗？"各 handler 检查自家设备的状态寄存器，回答 `IRQ_HANDLED`（是我）或
`IRQ_NONE`（不是我）。

`dev_id` 的两重身份：① 调 handler 时传给它的**身份饼干**（cookie），让同一个 handler
函数服务多块同型号网卡时知道"这次是哪块卡"；② `free_irq(irq, dev_id)` 注销时的
**查找钥匙**——链上好几张登记条，靠 dev_id 找到该撕哪张。所以共享线上 dev_id
**必须唯一且非 NULL**，否则注销时会撕错条子。

### 晦涩点 ⑬：`handler` vs `thread_fn` —— 一张登记条上的两个电话

`handler` 是**紧急联系电话**（硬中断上下文立刻接，必须秒挂）；`thread_fn` 是
**工单电话**（转给专属内核线程慢慢办，可以睡眠）。Topic 07 全章讲这对搭档。
`secondary`/`thread_mask`/`threads_oneshot` 等字段都是为线程化服务的记账，
现在见个面即可。

---

## 6. 三族标志位——"三本账，三把钥匙"

原文 §6 的表格信息密度极高，给一个记忆框架：

| 前缀 | 记在哪本账上 | 性质 | 钥匙（访问函数） |
|---|---|---|---|
| `IRQD_*` | 工牌公共栏（irq_data/common）| **现在时**：此刻 masked 吗？此刻 disabled 吗？ | `irqd_irq_masked()` 等 |
| `IRQ_*` | 档案夹设置页（desc.status_use_accessors）| **配置**：允不允许均衡？是 percpu 吗？ | `irq_settings_*()` |
| `IRQS_*` | 档案夹内部页（desc.core_internal_...）| **核心私房账**：PENDING（屏蔽期有人敲门）、ONESHOT… | 核心自己 |

那个故意起得很难看的字段名 `core_internal_state__do_not_mess_with_it`
（"核心内部状态——别乱碰"）是内核的一种**社会工程学防御**：让你在写出
`desc->core_internal_state__do_not_mess_with_it |= ...` 这行代码时自己都觉得羞耻。
规矩一句话：**认前缀、用配套的访问函数，永不裸写**。

---

## 7. 稀疏存储与生命周期

### 晦涩点 ⑭："irq_descs live in a maple tree, looked up by irq_to_desc(virq)"

**背景**：老内核用一个固定大小数组 `irq_desc[NR_IRQS]`。MSI 时代（Topic 11）
一块网卡就可能要几百个中断、虚拟机热插拔时中断成千上万地来去——固定数组要么
浪费内存要么不够用。于是 `CONFIG_SPARSE_IRQ`：档案夹**按需创建**，放进一棵
**maple tree**（内核的范围索引树，可以理解为"高效的稀疏数组"）里，
`irq_to_desc(virq)` 查树取档案。真身在 `kernel/irq/irqdesc.c:169`：

```c
static struct maple_tree sparse_irqs = MTREE_INIT_EXT(...);
```

**画面**：档案室从"预先钉死一万个格子的柜子"换成"用多少买多少的活动抽屉"。

### 晦涩点 ⑮："RCU frees them so lockless lookups in flow paths are safe"

删除档案夹时的难题：中断热路径正**无锁地**读着这个 `irq_desc`，你不能说删就删——
读的人手里的指针会突然指向已释放内存。RCU 的解法（呼应 topic01 §5 的值班表）：
先把档案从树上摘下（新查询查不到了），**等所有正在进行的无锁读者都走完**
（一个宽限期），才真正释放内存。`desc->rcu` 字段就是为这次"延迟火化"准备的。

---

## 8. 动手任务

### 任务 0：把 `/proc/interrupts` 的每一列对应到结构体字段（15 分钟，无需 root）

挑 `/proc/interrupts` 里你网卡的那一行（或任意 PCI-MSI 行），写出五元组：

```
virq │ 各CPU计数 │ chip名 │ hwirq+流程名 │ 设备名
```

回答（答案 §9.0）：每一段分别来自本章哪个结构体的哪个字段？
（提示：五段对应 5 个不同字段，分布在 3 个结构体里。）

### 任务 1：用 /sys 和 /proc 给一路中断"查户口"（15 分钟）

```bash
N=你网卡的中断号
grep -r . /sys/kernel/irq/$N/ 2>/dev/null
ls /proc/irq/$N/
cat /proc/irq/$N/smp_affinity /proc/irq/$N/effective_affinity 2>/dev/null
```

1. `actions` 文件的内容对应哪个结构体链？
2. 两个 affinity 文件如果内容不同，说明什么？（对照晦涩点 ⑧。）
3. `per_cpu_count` 对应 `irq_desc` 的哪个字段？

### 任务 2：源码寻宝——验证"一个 desc，多张工牌"（30 分钟）

```
① include/linux/irqdesc.h:80     通读 irq_desc，找到嵌在里面的那张 irq_data
② include/linux/irq.h:181        irq_data：找到 parent_data 字段和它的 #ifdef 条件
③ include/linux/irq.h:498        irq_chip：数一数有多少个回调以 irq_ 开头；
                                  找到 irq_bus_lock/irq_bus_sync_unlock 那对
④ kernel/irq/chip.c              搜 irq_chip_mask_parent，看它的函数体是不是
                                  真的只有"问上级"一行
⑤ kernel/irq/irqdesc.c:169       sparse_irqs maple tree + 往下找 irq_to_desc
```

回答（答案 §9.2）：

1. `irq_desc` 里嵌的 `irq_data` 是指针还是实体？这意味着最底层那张工牌的内存
   是怎么来的？那 hierarchy 上层的工牌内存又是谁分配的？（后半题可先猜，
   Topic 04 会证实。）
2. `irq_chip_mask_parent` 的完整函数体是什么？
3. `irq_to_desc` 在 SPARSE_IRQ 下最终调了 maple tree 的什么操作？

### 任务 3：用 bpftrace 或 drgn 活体解剖一个 irq_desc（30 分钟，需要 root；二选一）

**方案 A（drgn，推荐）**：`pip install drgn` 后 `sudo drgn`：

```python
from drgn.helpers.linux import *
desc = irq_to_desc(prog, N)        # N 换成你的网卡中断号；老版本接口或为
                                    # prog['sparse_irqs'] 查询，参考 drgn 文档
desc.handle_irq                     # 是哪个 flow handler？
desc.action.name                    # 认领者
desc.irq_data.hwirq                 # 工牌号
desc.irq_data.chip.name             # 门卫名
desc.depth                          # 0 = 启用
```

**方案 B（无 drgn 时）**：开 `CONFIG_GENERIC_IRQ_DEBUGFS` 的内核可
`sudo cat /sys/kernel/debug/irq/irqs/N`——一次性打印 IRQD/IRQS 状态、domain 层级、
chip 名。两种方案任选，回答：

1. 你的中断用的 flow handler 是哪个函数？和 `/proc/interrupts` 里的 `-edge`/`-fasteoi`
   后缀对上了吗？
2. `irq_data` 的 hwirq 和 virq 一样吗？（MSI 中断通常差很远——为什么？提前想想
   Topic 04 要解决的问题。）

### 任务 4：思考题（答案 §9.4）

1. 为什么 `irq_chip` 是几百路中断**共享一份**，而 `irq_data` 每路每层一张？
   用"操作手册 vs 工牌"说明哪些信息属于哪边。
2. 共享中断线上，如果两个驱动的 handler 都回答 `IRQ_HANDLED` 会怎样？都答
   `IRQ_NONE` 又会怎样（短期/长期）？
3. `disable_irq()` 用 depth 计数。那 `free_irq()` 之后 depth 会怎样？一个 depth>0
   且 action 链为空的 desc 处于什么状态？
4. 如果把 `desc->lock` 换成普通 `spinlock_t`，在 PREEMPT_RT 内核上第一个炸点
   会出现在哪？（提示：硬中断上下文里拿锁，而 RT 把普通自旋锁变成了什么？）

---

## 9. 任务参考答案

### 9.0 任务 0

`virq` ← 就是 `irq_to_desc` 的索引（也存在 `irq_data.irq`）；各 CPU 计数 ←
`desc->kstat_irqs`（per-CPU）；chip 名 ← `desc->irq_data.chip->name`；
hwirq+流程名 ← `irq_data.hwirq` + `desc->handle_irq` 对应的流程名缩写；
设备名 ← `desc->action->name`（共享线则逗号列出链上每张登记条的 name）。

### 9.1 任务 1

1. `actions` = 遍历 `desc->action` 链打印每个 `irqaction.name`。
2. 说明"要求的"和"硬件做到的"不一致——常见于 x86 单目标投递、或 managed affinity
   接管（Topic 13）。
3. `kstat_irqs`（per-CPU 计数数组）。

### 9.2 任务 2

1. **实体**（嵌入）。最底层工牌随档案夹一次性分配，零额外开销——这是为最常见的
   "无层级"情形优化。上层工牌由 domain 的 alloc 流程动态挂上（Topic 04 的
   `irq_domain_alloc_irqs_*` 一族）。
2. 基本就是两行：`data = data->parent_data; data->chip->irq_mask(data);`
   ——字面意义的"问我上级"。
3. `mt_find`（`irqdesc.c:190` 附近）——maple tree 的查找原语。

### 9.3 任务 3

1. 多数 MSI 网卡是 `handle_edge_irq`（`-edge` 后缀）。对上即可。
2. 通常不一样。MSI 的 hwirq 是 PCI-MSI domain 编号空间里的大数（含设备/功能编码），
   virq 是核心顺手分配的小数——**两个编号空间需要翻译**，翻译官就是 Topic 04 的
   irq_domain。

### 9.4 任务 4

1. 操作手册写的是**方法**（怎么 mask 任何一路），与具体哪一路无关 → 共享；
   工牌写的是**身份**（这一路在这一层的编号、私有数据 chip_data）→ 每路每层独立。
   方法/数据分离，就是 vtable 模式的本质。
2. 都答 HANDLED：功能上没事（各自处理了自家事件），统计上正常。都答 NONE：
   这次中断"无人认领"，`irqs_unhandled++`；**长期**如此（≈99k/100k）触发
   spurious 保护，整条线被禁——共享线上一个失职驱动会连累所有室友，
   这是共享中断口碑差的原因之一。
3. `free_irq` 不动 depth（它管 action 链）。depth>0 + 无 action = "被禁用的空线"：
   合法状态，等下一个 request_irq 时 `irq_startup` 会重新拉起。说明 disable
   计数和认领登记是**两套独立账本**。
4. RT 把 `spinlock_t` 变成可睡眠的 rtmutex。第一个炸点在硬中断入口路径第一次
   `raw_spin_lock(&desc->lock)` 的地方（如 flow handler 进门）——在不可睡眠
   上下文里调用可睡眠锁，lockdep 直接报 "BUG: sleeping function called from
   invalid context"。这正是 raw_ 前缀存在的全部意义（Topic 14 详述）。

---

## 10. 学完自测

1. 默写等式：一路中断 = 几个 desc + 几张 irq_data + 几份 chip + 几张 irqaction？
   每一项的归属者是谁？
2. 看到函数参数 `struct irq_data *d`，你立刻知道这是给谁用的函数？
3. `depth` 为什么是计数不是布尔？`dev_id` 的两重身份是什么？
4. 三族标志位 IRQD_/IRQ_/IRQS_ 各记在哪、用什么访问？
5. `irq_bus_lock` 那对回调解决什么矛盾？用"地铁里记备忘录"复述一遍。

下一章 Topic 04：hwirq 怎么翻译成 virq——给"工牌号"和"档案号"之间配一个翻译官。
