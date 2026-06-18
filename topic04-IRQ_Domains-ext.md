# Topic 04 扩展版 — IRQ Domain：编号世界的翻译官

> **用法同前**。行号基于 `~/repo/linux`（v7.1-rc7）：`struct irq_domain` 在
> `include/linux/irqdomain.h:168`、`generic_handle_domain_irq` 在
> `kernel/irq/irqdesc.c:729`、GICv3 的 domain ops 在 `drivers/irqchip/irq-gic-v3.c:1767`。
>
> **承接**：Topic 03 留下的悬念——任务 3 里你看到 MSI 中断的 hwirq 和 virq 差很远，
> "两个编号空间需要翻译"。本章就是那个翻译官的全部故事。

---

## 1. 先建立画面感：为什么非要两套编号？

### 1.1 "电话分机号"问题

**画面**：一家集团公司有很多分公司（控制器）。每个分公司内部都用分机号 0、1、2……
——北京分公司有分机 32，上海分公司也有分机 32（原文："Every GIC has an SPI 32;
every IO-APIC has a pin 0"）。集团总部（内核核心）要建一本**全集团唯一**的通讯录
（`irq_desc` 档案室），不能用"分机 32"做索引——那是谁家的 32？

所以有两套号：

- **hwirq** = 分机号。只在**自己分公司**内有意义（"GIC 的 SPI 32"）。
- **virq** = 集团工号。全内核唯一，是 `irq_to_desc()` 的索引、`request_irq()` 的参数、
  `/proc/interrupts` 最左列的数字。

**irq_domain 就是一个分公司的"分机号 → 集团工号"对照表**，外加这个分公司的
翻译规则。每个控制器初始化时建一个（Topic 02 §5 那句"建 domain"的下文）。

### 1.2 为什么不能静态写死对照表？

原文 §1 的三条理由，配上画面：

1. **撞号**（上面讲了）。
2. **动态发现**："员工"是入职时才知道的——设备树启动时才解析、PCI 设备可以热插拔、
   一块 MSI 网卡运行时申请 64 个中断又释放。工号必须**按需发放**（呼应 Topic 03 的
   maple tree 活动抽屉）。
3. **控制器是叠起来的**：同一路 MSI 中断在 PCI-MSI 层有个编号、在重映射层有个编号、
   在 vector 层又有个编号——**一条快递经过三个中转站，每站给它贴一张本站单号**。
   翻译不是"一对一查表"，而是**一串**（Topic 03 晦涩点 ⑥ 的工牌链，本章 §4 给出
   制造工牌链的机器）。

---

## 2. 逐句拆解 `struct irq_domain`

打开 `include/linux/irqdomain.h:168` 对照，拆几个真正难懂的字段：

### 晦涩句 ①："The reverse map (hwirq → irq_data) is the hot lookup done on every interrupt"

> 反向映射是每次中断都要做的热查询。

**为什么叫"反向"？** 正向是"集团工号 → 档案"（`irq_to_desc(virq)`，Topic 03 讲过）。
但中断到来时硬件报的是**分机号**（GIC 读 IAR 得到 hwirq）！所以投递路径需要反着查：
hwirq → virq/irq_data。**这个查询每秒可能发生几十万次**，必须极快、必须无锁。

源码里的实现是个二级策略（`irqdomain.h:168` 结构体尾部）：

```c
    irq_hw_number_t       hwirq_max;
    unsigned int          revmap_size;
    struct radix_tree_root revmap_tree;                    /* 大/稀疏号段：基数树 */
    struct irq_data __rcu *revmap[] __counted_by(revmap_size); /* 小而密的号段：内联数组 */
```

**画面**：前台查分机表。分机号 0~287 的（GIC SPI 这种密集小号段）直接做一面
**钉在墙上的格子板**（`revmap[]` 数组，O(1) 一眼看到）；号段巨大且稀疏的
（32 位 hwirq 的 MSI）用**索引卡柜**（radix tree）。`__rcu` 标记说明读路径用 RCU
保护——又是"无锁读 + 延迟回收"（Topic 03 晦涩点 ⑮ 同款），投递热路径一把锁都不拿。

### 晦涩句 ②：`bus_token` —— "disambiguates multiple domains on one fwnode"

一个控制器节点可能**同时开两家翻译社**：GIC 既管有线中断（wired）又管 MSI（通过 ITS）。
两个 domain 都挂在同一个固件节点（fwnode）上，找翻译社时光说"GIC 那家"不够，
还得说明"我要办有线业务还是 MSI 业务"——`bus_token`（`DOMAIN_BUS_WIRED` /
`DOMAIN_BUS_PCI_MSI` …）就是**业务窗口号**。

### 晦涩句 ③：v1（map/xlate）和 v2（alloc/translate）两代 API 并存

`irq_domain_ops` 里一半回调标着 v1 一半标着 v2，初读极易混乱。历史脉络一句话：

- **v1 = 平面时代**：一个控制器一本对照表，`xlate` 负责"把设备树里那几个数字解析成
  (hwirq, 触发方式)"，`map` 负责"新工号办入职：设置 chip 和 flow handler"。
- **v2 = 层级时代**（为 MSI 而生）：办一个号要**全链路各站一起办**，所以 `alloc`
  取代 `map`（一次把整串 irq_data 建好），`translate` 取代 `xlate`（接口更通用），
  新增 `activate`（见 §4 晦涩句 ⑥）。

平面控制器至今仍用 v1，没毛病；凡是要当 MSI 父级或叠在别人之上的，用 v2。
**读任何 irqchip 驱动前先看它填的是哪代回调，就知道它是平面还是层级。**

---

## 3. 逐句拆解三种平面映射 + 热路径汇合点

### 晦涩句 ④：Linear / Tree / No-map 三种建表方式

就是"格子板 vs 索引卡柜"的延伸，选择题只看 hwirq 号段的形状：

| 类型 | 建法 | 适用 | 画面 |
|---|---|---|---|
| Linear | `irq_domain_create_linear` | hwirq 是 0..N 密集小号段（GIC SPI、GPIO 块）| 墙上格子板 |
| Tree | `irq_domain_create_tree` | 号段大且稀疏（32 位 hwirq）| 索引卡柜 |
| No-map | `irq_domain_create_nomap` | hwirq 本身全局唯一（极少数遗留硬件）| 分机号就是工号，不用翻译 |

### 晦涩句 ⑤："`generic_handle_domain_irq(domain, hwirq)` … That single call is the join point of Topics 02→04→05"

> 这一个调用是 Topic 02、04、05 的汇合点。

把前几章的线索在这里打个结。`kernel/irq/irqdesc.c:729`：

```c
int generic_handle_domain_irq(struct irq_domain *domain, irq_hw_number_t hwirq)
{
    return handle_irq_desc(irq_resolve_mapping(domain, hwirq));
    /*       │                    │
     *       │                    └ Topic 04：反查格子板/卡柜，hwirq → desc
     *       └ 内部调 desc->handle_irq(desc) → Topic 05 的 flow handler
     * 调用者是谁？Topic 02 的根处理函数（gic_handle_irq）和所有级联分发函数 */
}
```

你在 topic01 任务 2（gic 读 IAR 后）、topic02 任务 2（GPIO 楼栋门禁分发）里
已经两次见过这个函数被调用——现在你知道它里面是什么了。

---

## 4. 逐句拆解层级 domain（全书最抽象的机器，值得慢读）

### 晦涩句 ⑥："Allocation walks top-down, then each level fills its irq_data bottom-up"

> 分配自顶向下递归，然后各层自底向上填好自己的 irq_data。

**画面：跨国快递开单。** 你在顶层窗口（PCI-MSI domain）说"我要寄一票件"。
顶层窗口自己**先不开单**，先打电话给下一站（vector domain）："给这票件留个位置"；
下一站又打给再下一站（APIC）……电话一路打到底（**top-down 递归**，源码里就是
`ops->alloc` 里调 `irq_domain_alloc_irqs_parent`，"先递归到父级"）。
最底层先把自己的单号填好挂上，返回；倒数第二层再填自己的
（`irq_domain_set_hwirq_and_chip`，"设置本层"）……**回程时（bottom-up）每层贴一张
自己的运单**。最终成品就是 Topic 03 讲的那串 `irq_data` 工牌链——
**本章这台机器就是工牌链的制造车间**（Topic 03 任务 2 问题 1 后半题的答案）。

x86 PCI-MSI 的真实链条（原文 §4 的图，加上"各层管什么"）：

```
PCI-MSI domain     "这是设备的第几个 MSI"        ← 顶层：负责拼出 MSI 消息内容
   │ parent
中断重映射 domain    （VT-d，存在则插在中间）       ← 安全/虚拟化的转译层
   │ parent
x86 vector domain  "落在哪个 CPU 的几号向量"      ← 从矩阵分配器(Topic 13)领格子
   │ parent
APIC (根)           真正投递
```

### 晦涩句 ⑦：`irq_chip_*_parent` 一族 —— "层级里的甩锅链"

Topic 03 任务 2 你已看过 `irq_chip_mask_parent` 的两行函数体。现在看全景：
ITS 芯片的 `irq_eoi` 只写一行 `irq_chip_eoi_parent`——**"回执的事我不懂，问 GIC"**。
每层只实现自己真正懂的操作，不懂的全部上交。这让"在中间插一层"（比如突然要支持
中断重映射）变成纯粹的局部改动——层级设计的全部收益就在这里。

### 晦涩句 ⑧："MSI IRQs are composed at alloc but written at activate" —— alloc 和 activate 为什么要分两步？

> MSI 在 alloc 时拼好消息，在 activate 时才写进设备。

**画面**：alloc = **预订**（酒店给你留了房，写好了房卡信息，但房卡还没给你）；
activate = **入住**（真正把房卡写进门锁——把 MSI 地址/数据写进设备的配置空间，
往 APIC/ITS 的表里登记）。

为什么分开？① 驱动常常一次申请 64 个中断但只用 8 个——没 activate 的不占稀缺资源
（x86 向量格子）；② 资源的最终去向（哪个 CPU）可能到真正启用时才定。
原文 Pitfalls 那条 "reading the device's MSI registers before startup shows stale data"
（启用前去读设备的 MSI 寄存器，读到的是旧数据）就是这个两段式的直接后果——
房卡还没写进门锁，你去刷当然没反应。

---

## 5. 逐句拆解固件接口（设备怎么"报需求"）

### 晦涩句 ⑨：fwspec / xlate / `#interrupt-cells` 这一团

设备树里一个网卡节点写着：

```dts
interrupt-parent = <&gic>;
interrupts = <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>;   /* 三个 u32 */
```

这三个数字是**给 GIC 的翻译社看的密文**。流程拆开：

1. `interrupts` 的那几个 u32 + 指向 GIC 的引用，打包成 **`irq_fwspec`**
   （"翻译申请单"：fwnode 说明找哪家翻译社，param[] 是密文）。
2. 凭 fwnode（+ bus_token 窗口号）找到 GIC 的 domain。
3. 调 domain 的 `translate`/`xlate` ——**只有 GIC 自己知道**"GIC_SPI 42" 要加 32 偏移
   变成 hwirq 74、第三个 cell 是触发方式。`#interrupt-cells = <3>` 是 GIC 在自己
   节点里声明的"我家密文是三个数一组"。
4. 翻译出 (hwirq=74, level-high) → 走 §3/§4 的机器分配 virq + 建映射。
5. 驱动最终拿到的就是个普通 virq，传给 `request_irq()`（Topic 06）。
   多数驱动只调一个封装好的 `irq_of_parse_and_map()`，对全过程无感。

原文 Pitfalls 第一名 *"#interrupt-cells mismatch"* 现在很好懂了：消费者按两个数一组写，
翻译社按三个数一组读——**密文错位，翻出来的 hwirq 是垃圾**，中断永远不来或者
打到别人家线上。这是设备树中断问题的头号病因。

### 晦涩句 ⑩：virq 0 的含义

原文 Pitfalls："A driver that finds before anyone created gets virq 0"。
genirq 的约定：**0 不是合法 virq，它是"没有/失败"**。`irq_find_mapping` 只查不建，
查不到返回 0；驱动不检查就拿去 `request_irq` 会得到 -EINVAL 或更怪的错。
看到代码里 `if (!virq)` 的判断，读作"映射不存在"。

---

## 6. 动手任务

### 任务 0：参观你机器上的所有翻译社（15 分钟，需要 root + debugfs）

```bash
sudo ls /sys/kernel/debug/irq/domains/ 2>/dev/null \
  || echo "内核未开 CONFIG_GENERIC_IRQ_DEBUGFS，跳到任务 1"
sudo cat /sys/kernel/debug/irq/domains/default
# 挑一个名字带 MSI / VECTOR 的：
sudo cat "/sys/kernel/debug/irq/domains/$(sudo ls /sys/kernel/debug/irq/domains/ | grep -i -m1 vector)"
```

回答（答案 §7.0）：

1. 你机器上有几个 domain？名字里能认出哪几类（VECTOR / PCI-MSI / IO-APIC / DMAR…）？
2. 找一个有 `parent:` 字段的 domain——它的父链有几层？和 §4 那张 x86 MSI 链条图
   对得上几层？
3. `mapped` 数字是什么？（对照结构体的哪个字段？）

### 任务 1：给一个 MSI 中断画"工牌链"（20 分钟，需要 root + debugfs）

```bash
N=你网卡或NVMe的中断号
sudo cat /sys/kernel/debug/irq/irqs/$N
```

输出里有一段 `domain:` 开头、逐层缩进的层级转储。把它抄成你自己的图：
每层写出 (domain 名, hwirq, chip 名)。回答：

1. 一共几层？每层的 hwirq 各是多少——**同一路中断，几个不同的"本站单号"**？
2. 最底层（root）的 chip 是什么？最顶层的呢？
3. 没有 debugfs 的备选方案：`grep . /sys/kernel/irq/$N/{hwirq,chip_name}`
   只能看到**叶子层**——印证原文 §6 哪句话？

### 任务 2：源码走读——把 generic_handle_domain_irq 拆到底（30 分钟）

```
① kernel/irq/irqdesc.c:729   generic_handle_domain_irq 本体（就两行）
② kernel/irq/irqdomain.c     __irq_resolve_mapping：找到 revmap[] 数组命中和
                              radix tree 兜底的分支
③ include/linux/irqdomain.h:168  对照 revmap_size / revmap_tree / revmap[] 三个字段
④ drivers/irqchip/irq-gic-v3.c:1767  gic_irq_domain_ops：它填了哪几个回调？
                              v1 还是 v2？（找 .translate / .alloc）
⑤ 同文件 gic_irq_domain_translate：亲眼看 "GIC_SPI x → hwirq x+32" 的偏移在哪行
```

回答（答案 §7.2）：

1. `__irq_resolve_mapping` 里判断走数组还是走树的条件是什么？
2. GICv3 的 domain 是 v1 还是 v2 风格？这说明它能不能当 ITS 的父级？
3. `gic_irq_domain_translate` 里 SPI 的偏移量是多少？PPI 呢？——现在你能解释
   设备树里 `GIC_SPI 42` 最终为什么变成 hwirq 74 了。

### 任务 3：写一个最小的"虚拟翻译社"内核模块（60~90 分钟，可选挑战）⭐

这是第一个真正写代码的任务。目标：创建一个假的 linear domain，手工建一条映射，
触发它，观察 `/proc/interrupts` 出现你的中断。骨架：

```c
// virq_domain_demo.c — 在 ~/repo/linux 之外建个目录，配个最简 Kbuild Makefile
#include <linux/module.h>
#include <linux/irq.h>
#include <linux/irqdomain.h>
#include <linux/interrupt.h>

static struct irq_domain *demo_domain;
static int demo_virq;

static irqreturn_t demo_handler(int irq, void *dev) {
    pr_info("demo irq fired! virq=%d\n", irq);
    return IRQ_HANDLED;
}

static int demo_map(struct irq_domain *d, unsigned int virq, irq_hw_number_t hw) {
    /* 入职手续：给新 virq 配上"什么都不做"的 chip 和最简流程 */
    irq_set_chip_and_handler(virq, &dummy_irq_chip, handle_simple_irq);
    return 0;
}
static const struct irq_domain_ops demo_ops = { .map = demo_map };

static int __init demo_init(void) {
    demo_domain = irq_domain_create_linear(NULL, 8, &demo_ops, NULL);
    demo_virq = irq_create_mapping(demo_domain, 3);   /* 分机 3 → 领工号 */
    pr_info("hwirq 3 mapped to virq %d\n", demo_virq);
    request_irq(demo_virq, demo_handler, 0, "domain-demo", NULL);
    /* 软件触发它： */
    generic_handle_domain_irq(demo_domain, 3);  /* 需在关中断上下文，见下方提示 */
    return 0;
}
static void __exit demo_exit(void) {
    free_irq(demo_virq, NULL);
    irq_dispose_mapping(demo_virq);
    irq_domain_remove(demo_domain);
}
module_init(demo_init); module_exit(demo_exit);
MODULE_LICENSE("GPL");
```

提示：① `generic_handle_domain_irq` 要求中断关闭的上下文，模块 init 里直接调会触发
告警——用 `local_irq_save/restore` 包住，或改用 `irq_set_irqchip_state(demo_virq,
IRQCHIP_STATE_PENDING, true)` 试试软重发路径；② 编译用标准的外部模块 Makefile
（`make -C /lib/modules/$(uname -r)/build M=$PWD modules`）——注意编译目标是
**你正在运行的内核**，不是 ~/repo/linux 的 v7.1（除非你跑的就是它，接口可能略有出入，
比如老内核需要用 `irq_domain_add_linear`）；③ 做完 `dmesg` 看打印、
`grep domain-demo /proc/interrupts` 看你的登记条、（有 debugfs 的话）
`sudo ls /sys/kernel/debug/irq/domains/` 找到你的无名 domain。
**做完这个任务，§2~§3 的所有概念就都摸过一遍实物了。**

### 任务 4：思考题（答案 §7.4）

1. 为什么反向映射要 RCU 保护、而不是拿 `desc->lock`？（想想是谁、在什么上下文查它。）
2. 层级分配为什么必须先递归到父级、再填本层？反过来（先填本层再问父级）会出什么问题？
3. 一个 GPIO 楼栋门禁（Topic 02 §4 的级联控制器）的 domain 需要是层级式的吗？
   它和 GIC 的关系与"ITS 和 GIC 的关系"有什么本质不同？
4. 为什么 virq 用 0 表示"无"而不是 -1？（提示：看 `irq_of_parse_and_map` 的
   返回类型。）

---

## 7. 任务参考答案

### 7.0 任务 0

1. 典型 x86 机器十几到几十个：`VECTOR`（根）、`IO-APIC-*`、`PCI-MSI*`、
   `DMAR-MSI`/`IR-*`（开了 VT-d 时）、`IOMMU-*` 等。
2. PCI-MSI 类 domain 的 parent 通常指向中断重映射 domain 或直接 VECTOR——
   能对上 §4 链条图的 2~3 层（APIC 不单独成 domain，VECTOR 即是根）。
3. `mapped` ≈ 结构体的 `mapcount`——这家翻译社当前活跃的"分机→工号"对数。

### 7.1 任务 1

1. 常见 2~3 层。例：顶层 PCI-MSI domain 的 hwirq 是设备/功能编码出的大数，
   底层 VECTOR domain 的 hwirq 就是 CPU 向量号（几十到两百多）。
   **同一路中断在每层有完全不同的 hwirq**——这就是 1.2 第 3 条"每站一张单号"。
2. 底层 chip 是 `APIC`/`IR-IO-APIC` 类；顶层是 `PCI-MSI`/`IR-PCI-MSI`。
3. 印证 "`/proc/interrupts` right column shows the leaf chip name; debugfs shows
   the whole stack"——/proc 和 /sys/kernel/irq 只见叶子，debugfs 见全链。

### 7.2 任务 2

1. `if (hwirq < domain->revmap_size)` 走内联数组，否则
   `radix_tree_lookup(&domain->revmap_tree, hwirq)`。
2. v2：填了 `.translate/.alloc/.free`（同时兼容地提供 select）。所以它**能**当父级
   ——ITS 的 domain 的 parent 正是它。
3. SPI 加 32，PPI 加 16（`gic_irq_domain_translate` 里 `*hwirq = fwspec->param[1] + 32`
   一带）。设备树写的是"SPI 空间内第 42 根"，GIC 全局 hwirq 空间里 SPI 从 32 起跳。

### 7.4 任务 4

1. 查它的人是**中断投递热路径**（每次中断、硬中断上下文、性能敏感），写它的人是
   慢路径（建/拆映射，每次设备初始化才发生一次）。读多写极少、读方不能慢不能睡
   ——教科书级的 RCU 适用场景。拿 desc->lock 不仅慢，而且此时 desc 还没找到呢
   ——锁本身就在查询结果里，鸡生蛋问题。
2. 本层的配置往往**依赖父级分配的结果**：MSI 消息内容里要写"目标 CPU 的哪个向量"，
   而向量是 vector 父域分配的——父级没办完，本层没法填表。先父后己是数据依赖
   决定的，不是风格选择。
3. 不需要层级。GPIO 门禁和 GIC 是**级联**（GPIO 的输出占 GIC 一根普通输入线，
   两个 domain 各自独立、平面，靠 chained handler 衔接）；ITS 和 GIC 是**层级**
   （同一路中断在两层各有一张 irq_data，操作逐层上交）。判别法：级联是
   "两路不同的中断接力"，层级是"同一路中断的多层描述"。
4. 因为返回类型是 `unsigned int`，-1 表示不了；且 0 在 Linux 传统里 IRQ0 是
   PIT 定时器这种不该被驱动申请的存在，牺牲它当哨兵值代价最小。内核文档明文规定
   "IRQ 0 is invalid as a driver-visible interrupt number"。

---

## 8. 学完自测

1. 用"分机号/集团工号"讲清 hwirq、virq、domain 三者关系，以及为什么必须按需分配。
2. 反向映射的两级结构（数组+基数树）各服务什么形状的号段？为什么用 RCU？
3. 层级 alloc 的"先递归后填表"顺序由什么决定？alloc/activate 两段式解决什么问题？
4. 设备树 `interrupts = <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>` 到驱动手里的 virq，
   中间经过哪五步？`#interrupt-cells` 错配会怎样？
5. `generic_handle_domain_irq` 的两行代码各连接着哪个 Topic？

下一章 Topic 05：找到 `irq_desc` 之后，那台"按触发方式跳舞"的状态机——flow handler。
