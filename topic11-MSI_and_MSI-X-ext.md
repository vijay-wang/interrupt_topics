# Topic 11 扩展版 — MSI 与 MSI-X：用"写内存"代替"拉电线"

> **用法同前**。行号基于 `~/repo/linux`（v7.1-rc7）：`struct msi_desc` 在
> `include/linux/msi.h:183`、`pci_alloc_irq_vectors` 在 `include/linux/pci.h:1767`、
> `pci_irq_vector` 在 `pci.h:1779`、`its_irq_compose_msi_msg` 在
> `drivers/irqchip/irq-gic-v3-its.c:1814`、`irq_compose/write_msi_msg` 回调在
> `include/linux/irq.h:534`。
>
> **承接**：Topic 04 讲层级 domain 时，MSI 一直是那个"动机例子"。本章把它**落地**——
> 用一个完整的端到端例子重新走一遍层级 domain。前置：Topic 04（层级/alloc/activate）、
> Topic 02（APIC、GIC-ITS）、Topic 03（`irq_compose_msi_msg` 回调）。

---

## 1. 先建立画面感：为什么用"消息"代替"电线"

### 1.1 老办法（INTx）的麻烦

Topic 02 讲的传统中断是一根**物理电线**：设备把线拉高，控制器看到。问题：

- **线少**：一块 PCI 卡传统上只有 4 根 INTx 线（A/B/C/D），插满设备只能**共享**
  （Topic 05 §4 的共享开销、"nobody cared"歧义）。
- **多队列没法玩**：现代 NVMe/网卡有几十上百个队列，每个队列想要自己的中断——
  4 根线远远不够。
- **顺序竞争**：数据用 DMA 写进内存，中断走另一条电线。可能**中断先到、数据还没写完**
  ——驱动读到半截数据（INTx 的真实历史 bug）。

### 1.2 新办法（MSI）：中断就是一次内存写

**MSI（Message-Signaled Interrupt）** 把"拉电线"换成"**往一个特定地址写一个特定的数**"：
设备通过 DMA 往平台规定的地址写一个值，**平台硬件把这次写翻译成一个中断**。

**画面**：老办法是"拉一根专线报警"（线有限、会串）；新办法是"**往一个特定信箱投一张
写好编号的纸条**"——信箱地址编码了"送给哪个 CPU"，纸条内容编码了"是哪个中断"。
信箱想要多少有多少，纸条互不干扰。

四大好处（原文 §1）：

1. **没有共享线**：每个 MSI 是自己独立的向量——没有共享 demux 开销、没有归属歧义。
2. **向量极多**：一个设备可以有几百上千个（每队列一个）——多队列 NVMe/网卡的命根子。
3. **DMA/中断顺序天然正确**：因为中断**本身就是一次写**，和数据 DMA 走**同一条路**
   （PCIe 保证写顺序）——中断绝不会跑到它所宣告的数据前面。**老 INTx 的竞争被根治了。**
4. **可路由**：目标地址编码了目的 CPU，所以改亲和性（Topic 13）就是"**重写这张纸条**"。

### 1.3 MSI vs MSI-X：一句话区别

| | MSI | MSI-X |
|---|---|---|
| 每设备向量数 | 最多 32，**必须连续** | 最多 2048，**各自独立** |
| 每向量地址/数据 | 共享基址、低位变化 | **完全独立** |
| 每向量屏蔽 | 可选/受限 | **每条都能单独屏蔽** |
| 表放哪 | 配置空间 | 设备 MMIO（BAR）里的 **MSI-X 表** |

**记忆**：MSI 是简化的老形式（连续、受限）；MSI-X 是现代高队列设备用的（独立、灵活）。
**两者投递后走的是同一套 genirq flow（Topic 05）**——只是"武装设备"的方式不同。

---

## 2. 逐句拆解 MSI 核心结构

### 晦涩点 ①：`msi_desc` —— 设备上每个 MSI 向量的"出生证"

`include/linux/msi.h:183`：

```c
struct msi_desc {
    unsigned int   irq;       /* 这个 MSI 向量对应的 virq（Topic 03）*/
    struct device  *dev;
    struct msi_msg msg;       /* ★组装好的"消息"：{address_hi/lo, data} */
    u16            msi_index;  /* 设备内第几条（0..N-1）*/
    union { struct pci_msi_desc pci; struct msi_desc_data data; };  /* 总线相关 */
};
```

`msi_desc` 挂在设备的 MSI 列表里，从中断侧通过 `irq_common_data.msi_desc`
（Topic 03 晦涩点 ⑦ 的公共栏）可达。**那个 `msg` 字段就是要写进设备的 MSI 能力寄存器/
MSI-X 表里的"纸条内容"**——写进去之后，设备触发中断时就照着它往那个地址写那个数。

**画面**：`msi_desc` 是这条 MSI 中断的完整档案，核心是 `msg` 这张"投递单"
（投到哪个地址 address、写什么内容 data）。

### 晦涩点 ②：compose（组装）vs write（写入）——两段式编程

这是本章和 Topic 04 的核心连接点。控制器的 `irq_chip`（Topic 03）提供两个回调
（`include/linux/irq.h:534`）：

- **`irq_compose_msi_msg(irq_data, msg)`** —— **算出**那张投递单的内容
  （`{address, data}` 指向选定的 CPU/向量）。发生在 **alloc/activate** 时。
- **`irq_write_msi_msg(irq_data, msg)`** —— 把投递单**写进设备**（MSI 写配置空间，
  MSI-X 写 BAR 里的 MSI-X 表）。这一步才**真正武装**设备。

**回忆 Topic 04 晦涩点 ⑧**：domain 的 `alloc`（预订/组装）和 `activate`（入住/写入）
两段式——**MSI 就是它的典范用户**：

```
alloc 阶段   → irq_compose_msi_msg：算好投递单（房卡信息写好，但还没给设备）
activate 阶段 → irq_write_msi_msg：把投递单写进设备（房卡塞进门锁，设备从此能触发）
```

原文 Pitfalls 那条"activate 前读设备 MSI 寄存器读到旧值"（Topic 04 同款）就是这个
两段式的直接后果——投递单还没写进设备呢。

---

## 3. 逐句拆解 PCI 驱动 API（驱动作者真正调的）

### 晦涩点 ③：四行代码搞定 MSI

```c
int n = pci_alloc_irq_vectors(pdev, min_vecs, max_vecs,
                              PCI_IRQ_MSIX | PCI_IRQ_MSI | PCI_IRQ_INTX);  /* ① 申请 */
int virq = pci_irq_vector(pdev, queue_index);   /* ② 取第 i 个向量的 virq */
request_irq(virq, handler, 0, name, dev);         /* ③ 之后就是普通中断（Topic 06）*/
pci_free_irq_vectors(pdev);                        /* ④ 释放 */
```

逐行（`pci.h:1767`/`1779`）：

1. `pci_alloc_irq_vectors`：按掩码里的**优先顺序**尝试最好的类型（先 MSI-X，再 MSI，
   再退回传统 INTx），分配向量（通过 MSI domain 创建 `msi_desc` 和 virq），
   返回**实际拿到几个**。
2. 驱动对返回的 virq 注册 handler——**从这一步起，MSI 就是个普通中断了**（Topic 06）。
3. `PCI_IRQ_AFFINITY`（用 `pci_alloc_irq_vectors_affinity`）开启**托管亲和性**铺开
   （Topic 13）——每个队列的中断自动落在不同 CPU 上。

**关键认知**（原文）：**驱动从不直接碰 APIC/ITS**——MSI domain 层级把它们全藏起来了。
驱动眼里只有"申请 N 个中断、拿到 N 个 virq、像普通中断一样用"。

---

## 4. 逐句拆解层级 domain 实战（Topic 04 落地）

这是把 Topic 04 §4 的抽象图填上真实内容。

### 晦涩点 ④：x86 的 domain 栈 + 向量稀缺

```
PCI-MSI domain     hwirq=MSI索引；irq_chip 组装并写设备的 MSI 消息
   │ parent
IRQ-remap domain   （仅当开了 Intel VT-d / AMD-Vi 重映射时存在）
   │ parent        消息 → 重映射表索引 → 向量
x86 vector domain  hwirq=CPU向量；★通过矩阵分配器(Topic 13)分配稀缺的每CPU向量
   │ parent        亲和性 = 选一个 CPU+向量
APIC (根)
```

**重点**（原文）：在 x86 上，**分配一个 MSI 就消耗一个 CPU 向量**（Topic 02 §2.4 的
稀缺资源）。海量 MSI 设备可能**耗尽向量**——这个失败就暴露在 vector domain 这一层
（dmesg 里 `irq N: vector allocation failed`）。源码：`arch/x86/kernel/apic/vector.c`
是 vector domain，`apic/msi.c` 是 PCI-MSI domain。

### 晦涩点 ⑤：arm64 的 domain 栈 + ITS 翻译

```
PCI-MSI domain
   │ parent
GIC-ITS domain     ★把 (DeviceID, EventID) 翻译成 LPI；用 ITS 的设备表/ITT 表
   │ parent        (drivers/irqchip/irq-gic-v3-its.c)
GIC (根，LPI 经 redistributor 投递)
```

**对比 x86**：arm64 **没有每 CPU 向量稀缺**问题（Topic 02 晦涩句 ⑧ 讲过，ITS 翻译表
在普通内存里，LPI 空间巨大）。它的限制是 ITS 表大小和 LPI 号空间。
`its_irq_compose_msi_msg`（`irq-gic-v3-its.c:1814`）在这里写 ITS 的门铃地址 + EventID，
ITS 把它映射成投给某 redistributor 的 LPI。

**这正是 Topic 01 晦涩句的回响**：x86 硬件 demux（向量直接编码源）vs arm64 软件翻译
（ITS 查表）——同一个架构哲学差异，在 MSI 这一层再次显形。

调试两者都用 `/sys/kernel/debug/irq/domains/` 看完整父链（Topic 04 §6）。

---

## 5. 逐句拆解 IMS

### 晦涩点 ⑥：IMS 是什么、和 MSI-X 什么关系

现代加速器（GPU、DPU、智能网卡）想要**远超 MSI-X 表上限（2048）**的中断数，
而且想用**设备自己的内存结构**管理，而不是标准 MSI-X 表。**IMS（Interrupt Message
Store，中断消息存储）** 让设备把 MSI 消息存在自己的私有结构里，通过同一套 MSI domain
框架**动态分配**。

**对学 genirq 的要点**（原文）：IMS **仍然是走层级 domain 栈的 MSI**——只是把"消息存储"
从 PCI 标准的 MSI-X 表换成了设备私有的存储。**同样的 `msi_desc`、同样的 compose/write
两段式、同样的 flow handler。** 换汤不换药，理解了 MSI 就理解了 IMS。

---

## 6. 动手任务（针对你的 x86_64 机器）

### 任务 0：在 /proc/interrupts 和 /sys 里观察 MSI（15 分钟，无需 root 大部分）

```bash
grep -E "PCI-MSI|MSIX|nvme|eth|enp|wlp" /proc/interrupts | head -20
# 看一个 PCI 设备拥有的 MSI 向量：
ls /sys/bus/pci/devices/*/msi_irqs/ 2>/dev/null | head
# 挑一个 NVMe 或网卡，数它有几个 MSI 中断：
for d in /sys/bus/pci/devices/*/; do
  c=$(ls $d/msi_irqs 2>/dev/null | wc -l)
  [ "$c" -gt 0 ] && echo "$(basename $d): $c MSI vectors"
done | sort -t: -k2 -rn | head
```

回答（答案 §7.0）：

1. 哪个设备拥有最多 MSI 向量？数量和它的队列数/你的 CPU 核数有关系吗？
   （联系 §3 的 managed affinity。）
2. `/proc/interrupts` 里 MSI 中断的命名（如 `nvme0q3`）告诉你什么？
   `q3` 是什么意思？
3. 找一个中断行，它右列写的是 `PCI-MSI` 还是 `PCI-MSIX`？和 §1.3 的区别对得上吗？

### 任务 1：看 MSI 的层级 domain 栈（15 分钟，需要 root + debugfs）

```bash
sudo ls /sys/kernel/debug/irq/domains/ | grep -iE "PCI-MSI|VECTOR|IR-"
N=$(grep -m1 nvme /proc/interrupts | awk -F: '{print $1}' | tr -d ' ')
sudo cat /sys/kernel/debug/irq/irqs/$N
```

回答（答案 §7.1）：

1. 你的 NVMe/网卡 MSI 中断的层级有几层？最顶是 PCI-MSI、最底是 VECTOR 吗？
   有没有 IR（中断重映射）层？（有 = 你开了 VT-d。）
2. 每层的 hwirq 各不相同——顶层（MSI 索引）和底层（CPU 向量号）分别是多少？
   这印证 Topic 04 §1.2 哪个画面（每站一张单号）？
3. `VECTOR` domain 那一层的 hwirq 落在 32~255 之间吗？（它就是 Topic 01 §3.1 的
   设备向量号。）

### 任务 2：源码走读 compose/write 两段式（30 分钟）

```
① include/linux/msi.h:183           msi_desc：找到 msg 字段
② include/linux/irq.h:534           irq_chip 的 compose/write 两个回调声明
③ arch/x86/kernel/apic/msi.c        找到 x86 的 irq_compose_msi_msg 实现
                                     （搜 compose），看它怎么把 CPU+向量编码进 address/data
④ drivers/irqchip/irq-gic-v3-its.c:1814  its_irq_compose_msi_msg：
                                     对比 arm64 怎么编码（ITS 门铃地址 + EventID）
⑤ kernel/irq/msi.c                  搜 activate，确认 write 发生在 activate 阶段
```

回答（答案 §7.2）：

1. x86 的 compose 把"目标 CPU"编码进了 message 的哪部分（address 还是 data）？
   "向量号"又编码进哪部分？
2. arm64 的 compose 写的 address 指向什么？（提示：ITS 的 GITS_TRANSLATER 寄存器
   的物理地址，即"门铃"。）
3. compose 和 write 在代码里分别由 domain 的哪个 ops 触发（alloc 还是 activate）？

### 任务 3：用 pci_alloc_irq_vectors 写一个查询小工具（45 分钟，可选挑战）

不真的注册中断（避免干扰真设备），而是写个模块**遍历一个 PCI 设备的 msi_desc**，
打印它们的 virq 和组装好的 msg：

```c
// 给定一个已绑定驱动的 pci_dev（或自己 pci_get_device 找一个）
struct msi_desc *desc;
msi_for_each_desc(desc, &pdev->dev, MSI_DESC_ALL) {
    pr_info("msi_index=%u virq=%u addr=%#llx data=%#x\n",
            desc->msi_index, desc->irq,
            ((u64)desc->msg.address_hi << 32) | desc->msg.address_lo,
            desc->msg.data);
}
```

更安全的**纯观察版**（不写模块）：

```bash
# MSI-X 表/能力可以从 lspci 看（不解码 message 内容，但能看结构）：
sudo lspci -vv -s <你的设备BDF> | grep -A8 -E "MSI-X|MSI:"
```

回答（答案 §7.3）：

1. `lspci -vv` 里 MSI-X 的 `Enable+` 和 `Count=N` 分别什么意思？N 和任务 0 数到的
   向量数一致吗？
2. （写模块的话）打印出的 `address` 高位是不是都一样、低位/data 区分不同向量？
   这印证 §1 的"地址编码目标 CPU、data 编码哪个中断"。

### 任务 4：思考题（答案 §7.4）

1. MSI 为什么天然解决了 INTx 的"中断先于数据到达"竞争？（提示：中断和数据现在走
   同一条 PCIe 路径，PCIe 保证什么？）
2. x86 上申请 4096 个 MSI-X 向量可能失败，arm64 上同样请求却能成功。从 §4 的两个
   domain 栈解释根本原因。这和 Topic 01/02 反复出现的什么架构差异是同一个？
3. 改一个 MSI 中断的亲和性（绑到另一个 CPU），底层发生了什么三步操作？
   （提示：compose→write，且 x86 还多一步。）为什么这在高频改亲和性时可能失败？
4. MSI 每个向量独占，那给 MSI 中断写 `IRQF_SHARED` 的 handler 有意义吗？会怎样？

---

## 7. 任务参考答案

### 7.0 任务 0

1. 通常是 NVMe 或多队列网卡向量最多；数量一般 ≈ min(设备队列数, CPU 数)——因为
   managed affinity 让每队列对一个 CPU，多了没用。
2. `nvme0q3` = nvme0 设备的第 3 号队列的中断。每个队列独立中断是 MSI-X 多向量的
   直接体现。
3. 现代设备基本是 `PCI-MSIX`（独立、可单独屏蔽）；老/简单设备可能 `PCI-MSI`。

### 7.1 任务 1

1. 开了 VT-d 的机器是 3 层（PCI-MSI → IR → VECTOR），没开是 2 层（PCI-MSI → VECTOR）。
2. 顶层 hwirq 是 MSI 索引（小数，0..N-1）；底层是 CPU 向量号（32~255）。印证
   "同一路中断每站一张不同单号"。
3. 是，VECTOR 层 hwirq 就是 Topic 01 §3.1 那个 32~255 的设备向量。

### 7.2 任务 2

1. x86：目标 CPU 编码进 **address**（APIC 地址的目的字段），向量号编码进 **data**。
   这就是"地址决定送谁、数据决定是哪个中断"。
2. arm64：address 指向 **ITS 的 GITS_TRANSLATER 物理地址**（门铃寄存器），
   data 是 EventID。设备写门铃，ITS 据 (DeviceID 隐含于来源, EventID) 查表得 LPI。
3. compose 在 `alloc`（或首次 activate 前）触发，write 在 `activate` 触发——
   正是 Topic 04 的两段式。

### 7.3 任务 3

1. `Enable+` = MSI-X 已启用；`Count=N` = 该设备 MSI-X 表有 N 个条目（向量上限）。
   实际用到的可能少于 N（驱动按需申请）。
2. 是：同一目标 CPU 的向量 address 相同（都指向那个 CPU 的 APIC），data 字段
   不同（区分向量号）。印证 §1 的编码方式。

### 7.4 任务 4

1. MSI 的中断**本身就是一次内存写**，和数据 DMA 走**同一条 PCIe 通道**。PCIe 保证
   同一通道上的写**保持顺序**——所以宣告数据的中断写必然排在数据写之后到达，
   驱动收到中断时数据一定已落地。INTx 的中断走独立电线，没有这个顺序保证。
2. x86 每个 MSI 消耗一个**稀缺的每 CPU 向量**（vector domain 经矩阵分配器分），
   4096 个可能耗尽；arm64 的 ITS 把映射放在**普通内存表**里，LPI 空间巨大，不稀缺。
   这与 Topic 01/02 的"x86 硬件 demux（向量编码源，数量有限）vs arm64 软件翻译
   （ITS 查表，空间大）"是**同一个**架构差异。
3. ① compose 新消息（指向新 CPU）；② write 写进设备；x86 还要 ③ 在新 CPU 上
   **分配一个新向量**（旧的释放）。高频改亲和性时，第③步可能因目标 CPU 向量耗尽
   而失败（Topic 13）。
4. 没意义。MSI 每向量独占，不存在共享，写 `IRQF_SHARED` 既无必要、也可能让核心
   拒绝或行为异常。MSI 的 handler 直接是独占的，无需"是不是我的"判断。

---

## 8. 学完自测

1. 用"投信箱"比喻讲清 MSI 相比 INTx 电线的四个好处。
2. MSI vs MSI-X 的四点区别？现代多队列设备用哪个、为什么？
3. compose 和 write 分别做什么、各在 domain 生命周期哪个阶段？对应 Topic 04 什么概念？
4. 画出 x86 和 arm64 的 MSI domain 栈，各层 hwirq 是什么？x86 的稀缺点在哪层？
5. 驱动用哪三个函数就能用上 MSI？它为什么完全不用碰 APIC/ITS？

下一章 Topic 12：其余三种"非电线"投递方式——每 CPU 私有中断、CPU 间中断（IPI）、
不可屏蔽中断（NMI）。
