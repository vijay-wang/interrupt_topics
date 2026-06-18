# Topic 08 扩展版 — 下半部 I：软中断（softirq）

> **用法同前**。行号基于 `~/repo/linux`（v7.1-rc7）：`kernel/softirq.c` 里
> `MAX_SOFTIRQ_TIME:543`、`MAX_SOFTIRQ_RESTART:544`、`handle_softirqs:579`、
> `__do_softirq:654`、`raise_softirq:790`、`ksoftirqd_should_run:1063`、
> `run_ksoftirqd:1068`；softirq 向量枚举在 `include/linux/interrupt.h:550`。
>
> **承接**：Topic 07 把"要睡眠的重活"搬进了线程。本章讲**另一条延迟路径**——
> 不睡眠但要快的重活（网络收包、定时器、块 I/O 完成），它们走的是"硬中断尾巴"上的
> 软中断。这正是 Topic 01 晦涩句 ㉑（`irq_exit_rcu` 那道"缝"）埋的钩子的下文。

---

## 1. 先建立画面感：软中断是什么、解决什么问题

### 1.1 "硬中断不能久留，但活儿不能不干"

Topic 06/07 反复说硬中断 handler 必须秒退。但网络收包这类活儿：**量大、不能睡、
又必须尽快做**（拖久了网络吞吐就崩）。线程化（Topic 07）能睡但有调度延迟，
对这种"每秒上百万次、不睡眠但要极快"的场景太重。

需要一种**介于硬中断和线程之间**的机制：

**画面**：快递员（硬中断 handler）到你家门口，**不进屋**（不能久留），把包裹往门口一放、
在你的待办清单上打个勾（`raise_softirq`：置一个"有软中断待处理"的标志位）就走了。
等你**送快递员出门、关上门的那一刻**（中断退出的"尾巴"，`irq_exit_rcu`），
你顺手把门口堆的包裹都处理了——**这时门是开着的（中断已重新打开），但你还在玄关
没法坐下喝茶（仍是原子上下文，不能睡眠）**。

这就是软中断：**在硬中断刚结束的尾巴上、中断重新打开、但仍不能睡眠的上下文里，
批量处理延迟下来的重活。**

### 1.2 为什么软中断"又少又固定"？

原文 §1："Why fixed and few?" 软中断是**性能最高、最底层**的延迟机制，给内核里
最热的子系统用（网络、块、定时器、RCU）。代价是它**牺牲了灵活性换性能**：

- **静态编号**，不能动态注册（编译时就定死那十个槽位）；
- **每 CPU 无锁运行**；
- **被精心限制**，不能饿死系统。

任何更灵活的东西（tasklet、工作队列，Topic 09）都**建在它之上**。所以软中断是
"地基"，先理解它，Topic 09 才有根。

---

## 2. 逐句拆解向量列表

`include/linux/interrupt.h:550` 的枚举，**编号即优先级**（一趟处理里小号先跑）：

```c
enum {
    HI_SOFTIRQ = 0,    /* 高优先级 tasklet */
    TIMER_SOFTIRQ,     /* 定时器轮回调 */
    NET_TX_SOFTIRQ,    /* 网络发送 */
    NET_RX_SOFTIRQ,    /* 网络接收（NAPI）——通常最热 */
    BLOCK_SOFTIRQ,     /* 块 I/O 完成 */
    IRQ_POLL_SOFTIRQ,  /* irq_poll（Topic 09）*/
    TASKLET_SOFTIRQ,   /* 普通 tasklet（Topic 09）*/
    SCHED_SOFTIRQ,     /* 调度器负载均衡 */
    HRTIMER_SOFTIRQ,   /* 高精度定时器 */
    RCU_SOFTIRQ,       /* RCU 回调——故意放最后 */
    NR_SOFTIRQS
};
```

### 晦涩点 ①：顺序为什么重要？

一趟 `__do_softirq` 会**从小号到大号**依次处理所有挂起的向量。把 TIMER 放在 NET 前面、
RCU 放最后是有讲究的：定时器关乎系统时间基准，优先处理；RCU 回调（释放内存等）
不急，垫底，避免它挤占网络这种实时性更强的活。**`/proc/softirqs` 的列就是这十个**——
任务 0 你会亲眼看到。

### 晦涩点 ②：`open_softirq` 的 handler 为什么没有参数？

```c
void open_softirq(int nr, void (*action)(void));  /* 注意 action 不带参数！ */
/* 例：open_softirq(NET_RX_SOFTIRQ, net_rx_action); */
```

普通回调都带个 `void *data`，软中断 handler 偏偏**无参数**。因为软中断是**每 CPU**
的极致性能路径，它要的数据直接从**当前 CPU 的 per-CPU 变量**里取（如网络的
`softnet_data`），不需要通过参数传——省一次间接、贴合 per-CPU 模型。这是"为性能
牺牲通用性"的又一例。注册是**静态的、init 时一次性**完成。

---

## 3. 逐句拆解"置位"与"运行"

### 晦涩点 ③：`raise_softirq` 其实什么都不"跑"

```c
void raise_softirq(unsigned int nr);         /* 置位（IRQ 安全的包装）*/
void raise_softirq_irqoff(unsigned int nr);  /* 同上，但调用者已关中断 */
```

原文 §3 关键句："Raising just sets a bit ... It does **not** run anything itself."
`raise_softirq` 只做两件极轻的事：① 在当前 CPU 的 `local_softirq_pending()` 掩码里
**置一个位**（"NET_RX 有活待干"）；② 如果当前**不在**中断/软中断上下文，
**唤醒 ksoftirqd**。它**不执行任何 handler**——执行发生在后面的"尾巴"上。

**画面**：在待办清单上打勾 ≠ 现在就做。打勾极快（一个原子位操作），真正做活留到
合适时机批量进行。

### 晦涩点 ④：`handle_softirqs` 的"重启循环" + 预算

`kernel/softirq.c:579`，这是本章的心脏，逐行配画面：

```c
static void handle_softirqs(bool ksirqd)
{
    unsigned long end = jiffies + MAX_SOFTIRQ_TIME;  /* 预算：2ms（:543）*/
    int max_restart = MAX_SOFTIRQ_RESTART;           /* 预算：最多 10 趟（:544）*/
    __u32 pending = local_softirq_pending();

    softirq_handle_begin();          /* preempt_count 加 SOFTIRQ_OFFSET → in_serving_softirq() 为真 */
restart:
    set_softirq_pending(0);          /* ★清空掩码：处理期间新来的 raise 累积到下一趟 */
    local_irq_enable();              /* ★★软中断运行时中断是开着的！ */
    while ((softirq_bit = ffs(pending))) {
        h = &softirq_vec[vec_nr];
        h->action();                 /* 跑注册的 handler，如 net_rx_action */
        pending >>= softirq_bit;
    }
    local_irq_disable();
    pending = local_softirq_pending(); /* 处理期间又有人 raise 了吗？*/
    if (pending) {
        if (time_before(jiffies, end) && !need_resched() && --max_restart)
            goto restart;            /* 预算内，再来一趟 */
        wakeup_softirqd();           /* ★预算耗尽，剩下的甩给 ksoftirqd */
    }
    softirq_handle_end();            /* preempt_count 减回去 */
}
```

四个关键设计，全在这段里：

1. **中断开着跑**（`local_irq_enable`）：只有摆弄挂起掩码那几行关中断，handler 本身
   **中断是开的**。所以**软中断不会延长硬中断的"关中断窗口"**——这是它存在的核心价值
   （对照 Topic 01：关中断窗口越长，中断延迟越糟）。
2. **有界**：最多 10 趟或 2ms，且一旦 `need_resched()`（有任务等着调度）就提前收手。
   超出就把剩余工作**交给 ksoftirqd**——保证软中断**不能霸占 CPU 饿死用户空间**。
3. **每 CPU + 串行**：同一向量可以在**不同 CPU 上并发**跑（NET_RX 在 8 个核同时收包），
   但**单个 CPU 上软中断是串行的**。所以 handler 要自己做跨 CPU 的锁（Topic 10）。
4. **新来的活累积到下一趟**：开头 `set_softirq_pending(0)` 清空掩码，处理期间新 `raise`
   的位会重新累积，下一趟（或下次）再处理——避免一边处理一边无限追加导致永不退出。

### 晦涩点 ⑤：为什么 `set_softirq_pending(0)` 要先清空？

如果不先清空，handler 处理 NET_RX 时网卡又 raise 了 NET_RX，掩码里那一位还在，
`while` 循环可能永远停不下来。先清空、让新来的累积到 `pending = local_softirq_pending()`
的二次检查里，配合"最多 10 趟"的预算，就把"处理期间持续来活"这件事**有界化**了。
这是防止软中断风暴的关键技巧。

---

## 4. 逐句拆解 `local_bh_disable/enable` 与 preempt_count

### 晦涩点 ⑥：保护"和软中断共享的数据"

如果你有一段普通代码（进程上下文）和某个软中断 handler **共享数据**，怎么防止它们
打架？软中断会在"尾巴"或 `local_bh_enable` 时插进来。办法：

```c
local_bh_disable();   /* 暂时禁止软中断在本 CPU 运行 */
   ... 安全地碰共享数据 ...
local_bh_enable();    /* ★解禁的瞬间，顺手把这期间挂起的软中断跑掉 */
```

### 晦涩点 ⑦：为什么 `in_softirq()` 和 `in_serving_softirq()` 不一样？（全章最绕）

这俩名字像，含义不同，是 Topic 00/10 反复出现的坑。秘密在 `preempt_count` 里软中断
子计数的**双重身份**（`softirq.c` 里 SOFTIRQ_OFFSET 的注释）：

- **`SOFTIRQ_OFFSET`（1 份）**：软中断**正在被执行**时加上 → `in_serving_softirq()` 为真。
- **`SOFTIRQ_DISABLE_OFFSET`（2 份）**：`local_bh_disable()` 加上 → `in_softirq()` 为真。

翻译成人话：

- `in_serving_softirq()` = **"此刻真的有个软中断 handler 在跑"**。
- `in_softirq()` = **"软中断在这里跑不了"**（要么正在跑，要么被 bh_disable 禁着）。

**画面**：`in_serving_softirq` 是"厨师正在炒菜"；`in_softirq` 是"厨房现在不接新单"
（可能正在炒，也可能临时关灶）。两者范围不同。

### 晦涩点 ⑧：`local_bh_enable()` 是一个"软中断运行点"

原文加粗强调这句。`local_bh_enable()` 把子计数减到 0 的那一刻，会**顺手执行**这期间
积压的软中断。所以软中断的运行点不止"硬中断尾巴"一个，还有：**每次 `local_bh_enable`**、
**ksoftirqd**。记住这三个运行点，调试"我的软中断什么时候跑的"才不会懵。

---

## 5. 逐句拆解 ksoftirqd

### 晦涩点 ⑨：为什么需要一个线程来兜底？

`softirq.c:1068`，每 CPU 一个 `ksoftirqd/<cpu>` 线程。它存在的唯一理由：**接住硬中断
尾巴在预算内（§4 的 10 趟/2ms）没处理完的软中断**。

为什么不在尾巴上死磕到处理完？因为那样会**无限延长**这次中断、饿死用户空间。
把溢出的活交给 ksoftirqd——一个**可被调度**的实体——之后，调度器就能在"软中断 CPU
时间"和"用户任务"之间做权衡，而不是让 NET_RX 把整个 CPU 吃光。

**实战信号**（原文，Topic 14/16 详述）：`top` 里看到 `ksoftirqd/N` 接近 100% CPU
= **软中断过载**（典型是网络收包风暴）。这是排查"机器很忙但用户进程跑不动"的
头号线索。

---

## 6. 逐句拆解 PREEMPT_RT 的改动

### 晦涩点 ⑩：RT 上软中断不再"原子"

普通内核：软中断在硬中断尾巴上以**原子上下文**运行（不能睡）。PREEMPT_RT 改成：

- `local_bh_disable()` 变成一把**每 CPU 的睡眠锁**——于是"禁 BH 区域"变得**可抢占、
  可迁移**。
- 软中断 handler 跑在**任务上下文**（通常是 ksoftirqd 或触发它的线程），**可抢占、
  能拿 RT mutex**。

这是 RT 故事的"软中断那一半"，对应 Topic 07 的"硬中断那一半"（线程化）。两半在
Topic 14 合体。**普通内核的"软中断原子、不能睡"铁律对大多数人仍然成立**——只是
RT 通过搬进线程放宽了它。

---

## 7. 动手任务

### 任务 0：读懂 /proc/softirqs（10 分钟，无需 root）

```bash
cat /proc/softirqs
# 看哪个向量计数最大、压在哪些 CPU 上：
watch -n1 'cat /proc/softirqs'   # 一边用网络（curl 大文件/ping），观察 NET_RX 涨
```

回答（答案 §8.0）：

1. 列名和 §2 枚举对得上吗？哪个向量计数最大？（多数机器是 TIMER 或 NET_RX/RCU。）
2. 制造网络流量（`curl -o /dev/null http://大文件` 或持续 ping），哪一列涨得最猛？
   它压在所有 CPU 还是少数 CPU？（联系网卡中断亲和性，Topic 13。）
3. `grep softirq /proc/stat`——软中断总数和 /proc/softirqs 各列之和对得上吗？

### 任务 1：源码精读 handle_softirqs（30 分钟，核心）⭐

打开 `kernel/softirq.c:579`，**通读一遍 `handle_softirqs`**，标出这五处：

```
(a) :581-583  两个预算变量 end / max_restart
(b) restart 标签后  set_softirq_pending(0) —— 为什么先清空（晦涩点⑤）
(c) local_irq_enable() —— 软中断运行时中断状态（晦涩点④点1）
(d) while + ffs + h->action() —— 实际跑 handler 的循环
(e) goto restart vs wakeup_softirqd —— 预算耗尽的分叉
```

回答（答案 §8.1）：

1. 三个继续/停止的条件（`time_before`、`!need_resched`、`--max_restart`）是**与**关系
   ——任何一个不满足就 `wakeup_softirqd`。为什么要三个条件而不是一个？
2. `MAX_SOFTIRQ_TIME` 是多少毫秒？`MAX_SOFTIRQ_RESTART` 是多少趟？（:543/:544）
3. handler 跑在 `local_irq_enable()` 之后——这对"软中断能否被硬中断打断"意味着什么？

### 任务 2：用 ftrace/bpftrace 抓软中断（20 分钟，需要 root）

```bash
sudo bpftrace -e 'tracepoint:irq:softirq_entry { @[args->vec] = count(); }
                  interval:s:5 { print(@); clear(@); exit(); }'
# 或者 ftrace：
sudo -i; cd /sys/kernel/tracing
echo 1 > events/irq/softirq_entry/enable
echo 1 > events/irq/softirq_raise/enable
echo 1 > tracing_on; sleep 2; echo 0 > tracing_on
grep -E "softirq" trace | head -40
echo 0 > events/irq/softirq_entry/enable; echo 0 > events/irq/softirq_raise/enable
```

回答（答案 §8.2）：

1. `softirq_raise`（谁打勾）和 `softirq_entry`（谁真跑）的向量号对得上吗？
   有没有"raise 了但很久才 entry"的延迟？
2. 哪个向量出现最频繁？vec 数字对照 §2 枚举是哪个名字？
3. 抓到的 `softirq_entry` 大多发生在硬中断附近（尾巴）还是 ksoftirqd 上下文？
   （看 trace 行首的进程名。）

### 任务 3：观察 ksoftirqd 过载（30 分钟，需要 root，虚拟机）

制造软中断压力，看处理如何从"尾巴内联"迁移到 ksoftirqd：

```bash
# 一个终端打流量（如果有第二台机器更好；否则本地 loopback 压一压）：
iperf3 -s &   # 或 ping -f localhost（flood ping 需 root）
sudo ping -f -s 1400 127.0.0.1 &
# 另一个终端观察：
top -d1        # 找 ksoftirqd/N，看它 CPU%
watch -n1 'cat /proc/softirqs | grep NET'
```

回答（答案 §8.3）：

1. 压力够大时，`ksoftirqd/N` 的 CPU% 上去了吗？这印证 §5 哪句话？
2. ksoftirqd 是普通调度（SCHED_OTHER）的——它和你的用户进程**抢** CPU 吗？
   这和 §5 "schedulable entity, scheduler can balance" 怎么对应？
3. 停掉压力后 ksoftirqd CPU% 是否回落？

### 任务 4：思考题（答案 §8.4）

1. 软中断不能睡眠，但它运行时**中断是开着的**。那"不能睡眠"和"中断开着"矛盾吗？
   它到底受什么约束才"不能睡"？（提示：preempt_count 里那个 SOFTIRQ_OFFSET。）
2. `spin_lock_bh` 和 `spin_lock_irqsave` 的区别？一份数据被"进程上下文 + 软中断"
   共享用哪个？被"进程上下文 + 软中断 + 硬中断"共享又用哪个？为什么？
3. 为什么 tasklet（Topic 09）要建在软中断之上，而不是直接做成第 11 个软中断向量？
   （提示：软中断"静态、固定、为性能牺牲灵活性"。）
4. 假如把软中断的预算从"10 趟/2ms"改成"无限处理直到清空"，在网络风暴下会发生
   什么？用户空间还能动吗？

---

## 8. 任务参考答案

### 8.0 任务 0

1. 列名就是 §2 那十个。空闲机器通常 TIMER/RCU/SCHED 最大；有网络/磁盘负载时
   NET_RX/BLOCK 上来。
2. NET_RX 涨最猛，且通常集中在**处理网卡中断的那几个 CPU**（亲和性决定，Topic 13）
   ——不是均摊到所有核。
3. /proc/stat 的 softirq 行是所有 CPU 所有向量的总和，应约等于 /proc/softirqs 全表之和。

### 8.1 任务 1

1. 三个条件各防一种失控：`time_before` 防"单趟太久"（墙钟），`max_restart` 防
   "趟数太多但每趟很快"（计数），`!need_resched` 防"有更该跑的任务在等"（响应性）。
   单一条件挡不住所有失控模式，所以三管齐下。
2. 2 毫秒；10 趟。
3. 意味着软中断 handler **可以被硬中断打断**（中断开着）——但不会被另一个软中断
   或进程抢占（仍在原子上下文/preempt 关）。所以硬中断优先级高于软中断，符合
   "上半部最紧急"的层级。

### 8.2 任务 2

1. 一般对得上且延迟很小（尾巴上立即处理）；高负载时可能看到 raise 后延迟到
   ksoftirqd 才 entry。
2. 通常 TIMER 或 NET_RX 最频繁；vec=1 是 TIMER，vec=3 是 NET_RX（按 §2 编号）。
3. 轻负载时多在硬中断尾巴（trace 行首可能是被打断的任意进程名或 `<idle>`）；
   重负载时迁移到 `ksoftirqd/N`。

### 8.3 任务 3

1. 是，ksoftirqd CPU% 显著上升——印证"持续负载下软中断处理从尾巴迁移到 ksoftirqd"。
2. 抢。ksoftirqd 是 SCHED_OTHER（和 IRQ 线程的 SCHED_FIFO 不同！），所以它**公平地**
   和用户进程竞争 CPU——这正是设计目的：让调度器能平衡软中断和用户任务，
   而不是软中断在尾巴上无限霸占。
3. 是，压力停止后回落到接近 0。

### 8.4 任务 4

1. 不矛盾。"中断开着"说的是硬件中断使能（IF/DAIF），"不能睡眠"说的是**不能调用
   调度器让出 CPU**。软中断在原子上下文（preempt_count 里有 SOFTIRQ_OFFSET，
   `in_serving_softirq` 为真），调度器拒绝在这种上下文切换任务——所以不能睡。
   两者是正交的两件事。
2. `spin_lock_bh`：拿锁同时禁本 CPU 软中断；`spin_lock_irqsave`：拿锁同时关本 CPU
   硬中断（连带软中断）。进程+软中断共享 → `_bh` 够（挡住软中断即可）；
   进程+软中断+硬中断共享 → 必须 `_irqsave`（硬中断能打断软中断，`_bh` 挡不住它）。
   用错（数据被硬中断碰却只用 _bh）会导致硬中断在你持锁时插进来访问同一数据 → 竞争/死锁。
3. 软中断槽位静态固定、为极致性能牺牲灵活性，不可能给每个驱动都开一个向量
   （只有 10 个）。tasklet 建在 TASKLET_SOFTIRQ/HI_SOFTIRQ 之上，让**任意多个**
   驱动动态注册延迟任务，共享这两个软中断向量——用一层间接换灵活性。这是 Topic 09。
4. 网络风暴下软中断会**无限处理收到的包**，永不退出尾巴/ksoftirqd 死循环占满 CPU，
   用户空间**完全饿死**（shell 无响应、ssh 不上）。这种"livelock（活锁）"正是
   预算机制要防止的——宁可丢包也要让系统保持响应。

---

## 9. 学完自测

1. 用"快递员打勾 + 关门时处理包裹"讲清 raise_softirq 和实际运行的分离。
2. 软中断的三个运行点是哪三个？运行时中断开还是关？能睡吗？为什么？
3. `handle_softirqs` 的预算（10 趟/2ms）和 ksoftirqd 兜底解决什么问题？
   不要这个预算会怎样（活锁）？
4. `in_softirq()` vs `in_serving_softirq()` 区别？对应 preempt_count 里加几份 OFFSET？
5. `spin_lock_bh` 和 `spin_lock_irqsave` 各防谁？什么时候必须用后者？

下一章 Topic 09：建在软中断之上的更灵活机制——tasklet、工作队列、irq_work，
以及"我到底该用哪一个"的完整决策矩阵。
