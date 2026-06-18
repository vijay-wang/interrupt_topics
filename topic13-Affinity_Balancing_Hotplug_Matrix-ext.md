# Topic 13 扩展版 — 亲和性、均衡、CPU 热插拔与矩阵分配器

> **用法同前**。行号基于 `~/repo/linux`（v7.1-rc7）：`irq_set_affinity` 在
> `kernel/irq/manage.c:481`、`irq_setup_affinity` 在 `manage.c:592`、
> `irq_create_affinity_masks` 在 `kernel/irq/affinity.c:26`、矩阵分配器在
> `kernel/irq/matrix.c`（`irq_matrix_alloc:383`、`reserve:352`、`alloc_managed:292`）、
> `migrate_one_irq` 在 `kernel/irq/cpuhotplug.c:53`、`irq_affinity_online_cpu` 在
> `cpuhotplug.c:233`。
>
> **承接**：前面讲了中断怎么投递（MSI、IPI 等）。本章讲**投到哪个 CPU、谁决定、
> CPU 上下线时怎么办**。也兑现了 Topic 02/11 反复提到的"x86 向量稀缺由矩阵分配器解决"。

---

## 1. 先建立画面感：亲和性是什么

### 1.1 "这个中断该让哪个 CPU 接"

多核机器上，一个中断来了，**该由哪个 CPU 处理？** 这就是**亲和性（affinity）**——
允许接收某中断的 CPU 集合。

**画面**：呼叫中心有 8 个接线员（CPU）。一通电话进来（中断），可以规定"只许 3 号接"
或"1~4 号谁有空谁接"。这个"许可名单"就是亲和性。为什么要管？因为：
- 让一个忙设备总打扰 CPU0 会让 CPU0 过载（其他核闲着）；
- 把中断和处理它的应用放同一个 NUMA 节点能减少跨节点访存（Topic 14）；
- 实时任务的核要保持"干净"，不能被中断打扰。

### 1.2 全章头号坑：requested vs effective（你点的 vs 实际端上的）

Topic 03 晦涩点 ⑧ 已经预告过，这里讲透。`irq_common_data` 里有**两个**掩码：

- **`affinity`（请求的）**：你**要求**的 CPU 集合（驱动设的、`/proc/irq/N/smp_affinity`
  写的、或默认）。是一个**集合**（"1~4 号都行"）。
- **`effective_affinity`（实际的）**：中断**真正落在**哪。很多控制器**一次只能投给一个
  CPU**（x86 选一个 CPU+向量），所以哪怕你要求"1~4 号"，实际只落在某一个（比如 2 号）。

**画面**：你点菜说"川菜粤菜都行"（affinity 是集合），但厨房实际给你做了一道宫保鸡丁
（effective 是单个）。调试"为什么中断都打在 CPU2"时，看错这两个文件
（`smp_affinity` vs `effective_affinity`）是头号 bug。

### 晦涩点 ①：改亲和性为什么"下次中断才生效"？

```c
int irq_set_affinity(unsigned int irq, const struct cpumask *mask);  /* → chip->irq_set_affinity */
```

`__irq_set_affinity` 调芯片的 `irq_set_affinity`（Topic 03）。但如果**中断正在飞行中**
或芯片要求延迟变更，新掩码会被**暂存在 `desc->pending_mask`**（Topic 03 见过这字段，
`CONFIG_GENERIC_PENDING_IRQ`），**等这个中断下次发生时才应用**（`irq_move_masked_irq`）。

**为什么？** 正在投递途中改路由可能丢中断或路由到半路。安全做法是"等它这次落地，
下次再换路"。后果（原文 Pitfalls）：**一个不触发的设备，改了亲和性也不会动**——
因为它没有"下次中断"来触发切换。这很反直觉，记住它。

---

## 2. 逐句拆解托管亲和性（多队列设备的自动铺开）

### 晦涩点 ②：为什么需要"托管"？

一块 NVMe 有 64 个队列、64 个中断。手动把它们一个个 pin 到 64 个 CPU 上？没人受得了，
而且 CPU 上下线时还得重新算。**托管亲和性（managed affinity）** 让内核**自动铺开并
在热插拔中保持合理**。

机制（原文 §2）：

- 驱动通过 `pci_alloc_irq_vectors_affinity(..., PCI_IRQ_AFFINITY, ...)`（Topic 11 §3）申请。
- **`irq_create_affinity_masks`**（`affinity.c:26`）为每个向量算出一个 CPU 掩码，
  把向量**均匀铺开**到各 CPU/NUMA 节点，可选地保留开头几个向量给非队列（管理）中断。
- 这些中断被标记 **`IRQD_AFFINITY_MANAGED`**：**用户空间不能**через `/proc/irq` 改它们的
  亲和性（写入被拒绝，返回 -EIO）——因为**队列↔CPU 映射归内核所有**。`irqbalance` 也跳过它们。
- CPU 热插拔时，托管中断的所有 CPU 都下线了就**关闭**它（而不是硬塞给幸存的 CPU），
  CPU 回来再恢复（§5）。

**画面**：托管亲和性像"公司统一排班"——HR（内核）把 64 个接线员和 64 条线一一配好，
**员工本人不能私自换线**（写 smp_affinity 被拒）。这就是为什么现代服务器上每个 NVMe
队列的中断"钉"在自己的 CPU 上，你动不了也不该动。

---

## 3. 逐句拆解用户态均衡 irqbalance

### 晦涩点 ③：irqbalance 管的是"非托管"中断

对**非托管**的中断，用户态守护进程 **irqbalance** 定期读 `/proc/interrupts`、
重写 `/proc/irq/N/smp_affinity`，把中断负载铺开到各 CPU（参考驱动通过
`irq_set_affinity_hint` 设的 `/proc/irq/N/affinity_hint` 提示）。

- **好处**：防止一个忙设备死磕 CPU0；NUMA 感知。
- **局限**：被动、粗粒度。对**延迟敏感/CPU 隔离**的场景，通常**关掉 irqbalance 手动 pin**
  （或用托管亲和性）。`IRQF_NOBALANCING`/`IRQD_AFFINITY_MANAGED` 的中断被它排除。

手动 pin：`echo <cpumask> > /proc/irq/N/smp_affinity`。配合 `isolcpus`/`nohz_full`
把中断挡在实时核之外（Topic 14）。

**画面**：irqbalance 像个"自动调度的领班"，定期看哪个接线员忙、重新分配。但它反应慢、
不够精细——要求严格的场合（实时、低延迟）宁可解雇它、手动排班。

---

## 4. 逐句拆解 x86 向量矩阵分配器

### 晦涩点 ④：兑现"向量稀缺"的承诺

Topic 02 §2.4、Topic 11 §4 反复说 x86 向量稀缺（每 CPU ~256 个，减去异常和系统向量）。
**矩阵分配器（matrix allocator，`kernel/irq/matrix.c`）** 就是管理这个稀缺资源的
**每 CPU 位图管理器**。

**画面**：一张二维表格，行是 CPU、列是向量号（0~255），每个格子标记"已用/空闲"。
分配器就是管这张"CPU × 向量"棋盘的管理员，确保不冲突、负载均衡。

### 晦涩点 ⑤：reserve（预订）vs allocate（落实）—— 为什么分两步

```c
irq_matrix_reserve(m);                /* 预订一个全局名额，但还不绑定具体 CPU */
irq_matrix_alloc(m, msk, ...);        /* 从负载最轻的合格 CPU 上落实一个 (CPU, 向量) */
```

- **reserve**：先在**全局**占一个名额——"系统保证能在**某处**放下这个中断"，但还没定
  哪个 CPU。
- **alloc**：真正绑定一个 `(CPU, 向量)`，从**负载最轻的合格 CPU** 上挑。

为什么分两步？和 Topic 04 的 alloc/activate 两段式同理——**先确保资源够（预订）、
真正启用时才占具体格子（落实）**。这样驱动申请 64 个但只启用 8 个时，没启用的不占
实际向量格子。

### 晦涩点 ⑥：托管预订——保证 CPU 回来时有向量

`irq_matrix_reserve_managed`/`alloc_managed`（`matrix.c:292`）支撑**托管亲和性**中断：
在目标 CPU 上**预先保留**向量，这样一个托管中断在它的 CPU 上线时**保证有向量可用**
（不会出现"CPU 回来了但向量被别人抢光了"）。`irq_matrix_available`（headroom，余量）
就是 Topic 11 "vector allocation failed" 的判据——余量为零就分配失败。

**这整个分配器是 x86 专属**；arm64（GIC-ITS LPI）没有每 CPU 向量压力，不用它。

---

## 5. 逐句拆解 CPU 热插拔

### 晦涩点 ⑦：一个 CPU 下线，它的中断怎么办？

```c
void irq_migrate_all_off_this_cpu(void)  /* CPU 下线晚期调用 cpuhotplug.c:171 */
{   for each desc affined to this CPU:
        migrate_one_irq(desc);            /* cpuhotplug.c:53 */
}
```

一个 CPU 要下线，所有可能落在它上面的中断**必须搬走**，否则中断没人接收。
`migrate_one_irq` 的处理分两类：

- **非托管中断**：**重新亲和**到掩码里还在线的某个 CPU；若掩码里的 CPU 全下线了，
  就"打破亲和性"（broken）随便找个在线 CPU，**保证设备继续工作**。
- **托管中断**（§2）：整个掩码都下线了就**关闭**（`irq_shutdown`）而非搬走——队列直接
  **停泊**，等它的某个 CPU 回来。CPU 上线时 **`irq_affinity_online_cpu`**（`cpuhotplug.c:233`）
  恢复它。

**画面**：3 号接线员下班（CPU 下线）。普通电话线**改派给别的在线接线员**（非托管，
保证有人接）；但 VIP 专线（托管，绑定特定接线员）**直接停用挂牌"暂停服务"**，
等 3 号回来再开——而不是硬塞给不熟悉这条 VIP 线的别人。

### 晦涩点 ⑧：`irq_force_complete_move` —— 搬家前先把上一次没搬完的搬完

原文 §5：`irq_force_complete_move`（Topic 03 的 chip 回调）在迁移**前**先解决一个
**没完成的亲和性变更**（还记得 §1 晦涩点 ① 那个"下次中断才生效"的 pending 状态吗），
确保飞行中的中断不丢。`IRQD_MOVE_PCNTXT` 标记那些能**从进程上下文立即改亲和性**
（而非延迟到下次中断）的芯片。

**联系 Topic 17**：boot 时给次级 CPU 上线跑的也是同一个 `irq_affinity_online_cpu`
——热插拔和初始化共用一套机器。

---

## 6. 动手任务（针对你的 x86_64 机器）

### 任务 0：观察 requested vs effective 亲和性（10 分钟，无需 root 读）

```bash
N=$(grep -m1 -E "nvme|enp|eth|wlp" /proc/interrupts | awk -F: '{print $1}' | tr -d ' ')
echo "irq $N:"
cat /proc/irq/$N/smp_affinity_list
cat /proc/irq/$N/effective_affinity_list
# 看这个中断实际打在哪些 CPU（/proc/interrupts 该行哪几列非零）：
grep "^\s*$N:" /proc/interrupts
```

回答（答案 §7.0）：

1. `smp_affinity_list`（请求）和 `effective_affinity_list`（实际）一样吗？
   如果请求是多个 CPU 但 effective 是一个，印证 §1.2 哪个画面？
2. `/proc/interrupts` 里这个中断的计数是均摊在多个 CPU 还是集中在一个？和
   effective_affinity 对得上吗？

### 任务 1：尝试 pin 一个普通中断 vs 一个托管中断（15 分钟，需要 root）

```bash
# 找一个非托管中断（如键盘/鼠标/USB），试着改它：
N=普通中断号
echo 2 | sudo tee /proc/irq/$N/smp_affinity     # 绑到 CPU1（掩码 0x2）
cat /proc/irq/$N/smp_affinity_list               # 看是否生效
# 找一个 NVMe 队列中断（多半是托管的），试着改它：
M=nvme队列中断号
echo 2 | sudo tee /proc/irq/$M/smp_affinity     # 观察是否被拒绝
```

回答（答案 §7.1）：

1. 普通中断改成功了吗？托管中断改时报什么错？（应该是 -EIO 或 "write error"。）
2. 怎么从 `/sys/kernel/debug/irq/irqs/M` 确认某中断是 `IRQD_AFFINITY_MANAGED`？
3. 为什么内核**禁止**你 pin 托管中断？（联系 §2 的"队列↔CPU 映射归内核所有"。）

### 任务 2：观察矩阵分配器的向量使用（15 分钟，需要 root + debugfs）

```bash
sudo cat /sys/kernel/debug/irq/domains/VECTOR
# 看每 CPU 向量使用和系统余量
```

回答（答案 §7.2）：

1. 输出里能看到每个 CPU 已用了多少向量、还剩多少（available）吗？
2. 你机器总共大约能分配多少个设备向量？（CPU 数 × 每 CPU 可用向量数。）
3. 如果这个 `available` 接近 0，再插多队列网卡会发生什么？（联系 Topic 11
   "vector allocation failed"。）

### 任务 3：实测 CPU 热插拔时的中断迁移（30 分钟，需要 root，虚拟机）⭐

> ⚠️ 虚拟机里做。下线一个 CPU，观察中断怎么搬。**不要下线 CPU0**（很多东西绑它）。

```bash
N=某个非托管中断号
cat /proc/irq/$N/effective_affinity_list           # 记下当前落在哪个 CPU，假设是 CPU3
# 把那个中断 pin 到 CPU3：
echo 8 | sudo tee /proc/irq/$N/smp_affinity        # 0x8 = CPU3
# 下线 CPU3：
echo 0 | sudo tee /sys/devices/system/cpu/cpu3/online
dmesg | tail -5                                     # 看 "IRQ N: ... affinity" 类消息
cat /proc/irq/$N/effective_affinity_list           # 现在落在哪了？
# 恢复：
echo 1 | sudo tee /sys/devices/system/cpu/cpu3/online
```

回答（答案 §7.3）：

1. 下线 CPU3 后，原本 pin 在 CPU3 的非托管中断搬到哪了？dmesg 有相关消息吗？
2. 如果对一个**托管**中断做同样实验（且它只绑 CPU3），下线后它是被搬走还是被关闭？
   （从 /proc/interrupts 看它是否消失/停止计数。）
3. CPU3 重新上线后，中断回到 CPU3 了吗？托管和非托管的恢复行为有何不同？

### 任务 4：思考题（答案 §7.4）

1. 为什么 x86 改一个 MSI 中断的亲和性可能**失败**，而 arm64 不会？
   （联系 §4 矩阵分配器 + Topic 11 §4 两个 domain 栈。）
2. 托管中断在所有绑定 CPU 下线时被"关闭"而非"搬到幸存 CPU"。为什么这个设计
   比"硬塞给幸存 CPU"更好？（提示：队列和 CPU 的对应关系、缓存局部性。）
3. 一个延迟敏感的实时应用独占 CPU4-7。你要确保中断不打扰这几个核。
   有哪几种手段（至少三种）？它们分别管托管还是非托管中断？
4. 改亲和性"下次中断才生效"。一个几乎不触发的设备（如很少用的传感器），
   你改了它的亲和性但它迟迟不动——这是 bug 还是正常？怎么强制立即生效？

---

## 7. 任务参考答案

### 7.0 任务 0

1. x86 上经常不一样：请求可能是一组 CPU，effective 是其中一个（x86 单目标投递）。
   印证"点了川菜粤菜都行，实际端上一道"。
2. 应集中在 effective_affinity 指向的那个 CPU 上（该列计数远高于其他）。对得上。

### 7.1 任务 1

1. 普通中断改成功（smp_affinity_list 变为 1）；托管中断写入被拒，
   `tee: ...: Input/output error`（-EIO）。
2. `sudo cat /sys/kernel/debug/irq/irqs/M | grep -i managed` 或看 IRQD 标志位里有
   `AFFINITY_MANAGED`。
3. 因为内核根据队列数精心计算了 queue↔CPU 映射（每队列对一个 CPU 以获得最佳
   缓存局部性和无锁路径），用户随意改会破坏这个映射、损害性能甚至正确性。

### 7.2 任务 2

1. 能。VECTOR domain 转储列出每 CPU 的 allocated/managed/available 计数。
2. 约 = 在线 CPU 数 × (256 - 异常 32 - 固定系统向量约 20+) ≈ CPU 数 × 约 200。
3. 新中断分配会失败（`vector allocation failed`），设备可能拿不到足够中断、
   退化到更少队列或 INTx。托管预订（§4 晦涩点 ⑥）就是为缓解这个而存在。

### 7.3 任务 3

1. 搬到掩码里其他在线 CPU（若掩码只有 CPU3，则"broken"到任意在线 CPU 如 CPU0）。
   dmesg 通常有 "IRQ N: no longer affine to CPU3" 类消息。
2. 托管中断会被**关闭**（irq_shutdown），/proc/interrupts 里它停止计数（停泊），
   而不是搬到别的 CPU。
3. CPU3 上线后：托管中断由 `irq_affinity_online_cpu` **恢复到 CPU3**；非托管中断
   **不会自动搬回**（除非 irqbalance 或你手动重设）——这是两者的关键区别。

### 7.4 任务 4

1. x86 改亲和性要在**目标 CPU 上分配一个新向量**（稀缺资源），目标 CPU 向量耗尽时
   失败；arm64 改亲和性只是**重写 ITS 映射**（LPI 空间充裕），无稀缺问题。
   根源还是 Topic 01/02 的硬件 demux（向量有限）vs 软件翻译（表在内存）差异。
2. 托管中断的 CPU 和**特定队列**绑定（缓存、NUMA 局部性最优）。硬塞给幸存 CPU 会
   让那个 CPU 既处理自己的队列又处理这个外来队列，破坏局部性、可能过载；而队列
   本身在它的 CPU 下线后通常也不再产生工作（应用也迁走了），停泊它更干净，
   CPU 回来再原样恢复。
3. ① 关掉 irqbalance 并手动把非托管中断 pin 到 CPU0-3（管非托管）；
   ② `isolcpus=4-7` / `nohz_full=4-7` 启动参数（让调度器和 tick 避开这些核）；
   ③ 托管中断通过驱动的队列数控制 + 内核自动避开隔离核（管托管）；
   ④ `IRQF_NOBALANCING` 让特定中断不被均衡。前两种最常用。
4. **正常，不是 bug**。亲和性变更暂存在 pending_mask，靠"下次中断"触发实际迁移
   （§1 晦涩点 ①）。很少触发的设备迟迟不迁移是预期行为。要立即生效，可手动触发
   一次该中断（如让设备产生事件），或对支持 `IRQD_MOVE_PCNTXT` 的芯片它本就立即生效。

---

## 8. 学完自测

1. requested affinity 和 effective affinity 的区别？x86 为什么常常 effective 是单个 CPU？
2. 托管亲和性解决什么问题？为什么用户不能 pin 托管中断？
3. 矩阵分配器管什么资源？reserve 和 alloc 为什么分两步？它是哪个架构专属的？
4. CPU 下线时，非托管中断和托管中断的处理有何不同？为什么这样设计？
5. 改亲和性为什么常常"下次中断才生效"？这会导致什么反直觉现象？

下一章 Topic 14：前面所有放置决策的**时间后果**——中断时间怎么计量、延迟从哪来、
PREEMPT_RT 的完整模型。
