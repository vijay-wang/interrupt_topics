# Topic 05 扩展版 — Flow Handler：按触发方式跳舞的状态机

> **用法同前**。行号基于 `~/repo/linux`（v7.1-rc7）：`kernel/irq/chip.c` 里
> `handle_simple_irq:612`、`handle_level_irq:687`、`handle_fasteoi_irq:738`、
> `handle_edge_irq:825`；`__handle_irq_event_percpu` 在 `kernel/irq/handle.c:185`；
> "nobody cared" 在 `kernel/irq/spurious.c:160`。
>
> **承接**：Topic 04 的 `generic_handle_domain_irq` 已经找到了 `irq_desc`，并调用了
> `desc->handle_irq(desc)`。**这个被调用的函数就是 flow handler**，本章主角。
> **前置知识**：Topic 02 §1.2 的电平 vs 边沿（如果忘了，回去重读那一对画面，
> 本章一半内容是它的代码化身）。

---

## 1. 先建立画面感：flow handler 是什么角色？

### 1.1 "接待流程模板"

回到宅子比喻。一路中断的档案夹（`irq_desc`）里写着用哪套**接待流程**
（Topic 03 晦涩点 ①：`handle_irq` 字段）。flow handler 就是这套流程模板。

为什么需要"模板"而不是每个驱动自己写？因为**不同触发方式的访客，接待规矩完全不同**：

- **电平触发的访客**（一直按着门铃不放，Topic 02）：你必须**先把这个门铃静音**
  （mask），否则你去招待他时门铃一直响，你刚要坐下又被铃声拽起来——死循环。
  招待完（他的事办好了，手松开门铃），再解除静音。
- **边沿触发的访客**（敲一下就走）：不用静音（他本来就不按着了），但你招待 A 时
  B 又敲了一下门——**这一下不能漏**，得记下来"还有人敲过"，招待完 A 立刻招待 B。
- **聪明门卫递来的访客**（fasteoi，现代 GIC/APIC）：门卫已经在硬件层面把优先级和
  屏蔽都处理好了，你只需要招待完**说一声"这位办完了"**（EOI），别的不用管。

内核把这三套规矩写成三个久经测试的函数，挂在档案夹上。中断设置时
（Topic 06 的 `irq_set_handler` 或芯片的 `map`）根据触发类型选一个挂上去。

### 1.2 这一章的"题眼"：顺序就是一切

原文 §2 那句 *"The differences are entirely in the order of chip operations"*
（差异完全在于芯片操作的**顺序**）是本章最重要的一句话。三个 handler 调的都是
那几个动作——`irq_mask` / `irq_ack` / `irq_eoi` / `irq_unmask`（Topic 03 的
`irq_chip` 回调）——区别只在**先做哪个、什么时候做、要不要循环**。
所以读本章的正确方法是**三个 handler 并排对比着读**，看顺序差异，而不是孤立背诵。

---

## 2. 逐句拆解三大 flow handler

### 2.1 `handle_level_irq`：电平——"先静音再招待"

`kernel/irq/chip.c:687`：

```c
void handle_level_irq(struct irq_desc *desc)
{
    guard(raw_spinlock)(&desc->lock);   /* 见晦涩点① */
    mask_ack_irq(desc);                 /* ★第一步：先 mask！线还拉着呢 */
    if (!irq_can_handle(desc)) return;  /* 禁用了/没人认领/正在处理 → 退出 */
    kstat_incr_irqs_this_cpu(desc);
    handle_irq_event(desc);             /* 招待：handler 去设备上清状态，线放下 */
    cond_unmask_irq(desc);              /* ★最后：设备安静了才解除静音 */
}
```

### 晦涩点 ①：`guard(raw_spinlock)(&desc->lock)` 是什么写法？

这是较新内核的 **cleanup guard** 语法（基于 `__attribute__((cleanup))`）。
意思是：**在这里拿锁，函数返回时自动放锁**——无论从哪个 `return` 出去都不会忘记解锁。
等价于老代码里满地的 `raw_spin_lock(); ...; raw_spin_unlock();` 配对，但杜绝了
"某个分支忘了解锁"的经典 bug。读到 `guard(...)` 就读作"进入此作用域上锁，离开自动解锁"。
（`raw_spinlock` 的"raw"含义见 Topic 03 晦涩点 ④。）

### 晦涩点 ②：为什么 mask 必须在 handle 之前？

这就是 Topic 02 §1.2 "电平=一直按着门铃"的代码后果，原文 §2.1 讲得清楚，再强化画面：
电平线在设备被服务**之前一直是高的**。假如你不先 mask 就去跑 handler——handler 还没
让设备把线放下来，这条**仍然为高**的线会立刻再次触发中断，CPU 刚迈出门又被拽回来，
**无限中断风暴**。所以铁律：mask+ack → handle（handler 把设备条件清掉、线落下）→ unmask。
原文管这叫 "the canonical mask-around-the-handler dance"（围绕 handler 的经典屏蔽舞步）。

### 2.2 `handle_edge_irq`：边沿——"不静音，但记住漏掉的敲门"

`kernel/irq/chip.c:825`（结构简化）：

```c
void handle_edge_irq(struct irq_desc *desc)
{
    guard(raw_spinlock)(&desc->lock);
    if (!irq_can_handle(desc)) {            /* 现在不能处理（如被禁用）*/
        desc->istate |= IRQS_PENDING;       /* ★记一笔："欠着一次"*/
        mask_ack_irq(desc);
        return;
    }
    kstat_incr_irqs_this_cpu(desc);
    desc->irq_data.chip->irq_ack(...);      /* ★只 ack，不 mask！*/
    do {
        if (unlikely(!desc->action)) { mask_irq(desc); return; }
        if (desc->istate & IRQS_PENDING)    /* 处理期间又有人敲了门 */
            ... unmask_irq(desc);
        handle_irq_event(desc);
    } while (desc->istate & IRQS_PENDING && !disabled);  /* ★欠着就再来一轮 */
}
```

### 晦涩点 ③：为什么边沿"只 ack 不 mask"，还要 `do-while` 循环？

边沿是**瞬间事件**（敲一下就松手），不存在"线一直拉着"的问题，所以**不需要 mask**
来防重入。但有另一个危险：**handler 正在跑的时候，设备又敲了一下门**——这一下如果
丢了，对应的事件（比如一个网络包）就永远丢了。

解法是软件补记：硬件每次敲门会置一个 `IRQS_PENDING` 标志（Topic 03 §6 的 `IRQS_*`
家族成员）。handler 跑完，检查这个标志——如果处理期间有人敲过，**循环再跑一遍**。
这个 `IRQS_PENDING` 重跑循环，原文说 *"is the whole reason edge IRQs aren't dropped"*
（是边沿中断不丢失的全部原因）。

对照记忆：**电平靠 mask 防风暴（线会自己提醒你），边沿靠 PENDING 标志防丢失
（事件不会自己重来，得软件记账）。** 这一对正是 Topic 02 §1.2 "状态 vs 事件"的回响。

### 2.3 `handle_fasteoi_irq`：聪明门卫——"招待完说一声就行"

`kernel/irq/chip.c:738`（结构简化）：

```c
void handle_fasteoi_irq(struct irq_desc *desc)
{
    struct irq_chip *chip = desc->irq_data.chip;
    guard(raw_spinlock)(&desc->lock);
    if (!irq_can_handle_pm(desc)) { ...cond_eoi_irq(...); return; }  /* 挂起态等 */
    ...
    kstat_incr_irqs_this_cpu(desc);
    if (desc->istate & IRQS_ONESHOT) mask_irq(desc);  /* ★线程化的钩子，见下 */
    handle_irq_event(desc);
    cond_unmask_eoi_irq(desc, chip);    /* ★EOI（非 oneshot 时顺便 unmask）*/
    ...
}
```

### 晦涩点 ④：fasteoi 为什么"通常一个 mask 都没有"？

原文："modern controllers do the masking/priority in hardware and only need a single
EOI at the end"。GICv3/APIC 这类聪明门卫，在**硬件层面**就用优先级机制保证了
"正在处理这个中断时，同级中断不会进来"（Topic 12 的优先级屏蔽）。所以软件不用费劲
去 mask/unmask，只要招待完**写一下 EOI 寄存器**（"这位办完了，下一位"）。这是
**服务器上最常见的 flow handler**——原文让你 "read it twice"，因为生产环境的中断
99% 走这里。

### 晦涩点 ⑤：`IRQS_ONESHOT` 那行 mask 是给谁留的钩子？

`if (desc->istate & IRQS_ONESHOT) mask_irq(desc);`——这行是 fasteoi 里**唯一**的 mask，
它是给**线程化中断（Topic 07）**预留的接口。含义：当这个中断要交给内核线程慢慢处理时，
**从这里开始把线屏蔽住，一直到那个线程干完才解除**（解除发生在 Topic 07 的线程收尾，
不在这个硬中断里）。现在记住这个钩子的存在即可——它是 Topic 05 和 Topic 07 的接缝，
读到 Topic 07 时回来看这一行。

### 2.4 其余专用 handler（快速扫一眼）

原文 §2.4 列了一串，给个"什么时候用"的速查：

| handler | 用于 | 一句话 |
|---|---|---|
| `handle_simple_irq` | 软件中断/纯 demux | 啥都不 ack 不 eoi，没硬件可应答（Topic 04 任务 3 用的就是它）|
| `handle_percpu_devid_irq` | 每 CPU 线（arch timer、PMU）| 不拿 desc->lock（per-CPU 无竞争），Topic 12 |
| `handle_fasteoi_nmi` | 伪 NMI | 只跑 NMI 安全的操作，Topic 12 |
| `handle_nested_irq` | I2C GPIO 扩展器这种"父线程里再分发" | 在线程上下文里跑，Topic 07 |

---

## 3. 逐句拆解 action 链的执行

`kernel/irq/handle.c:185` 的 `__handle_irq_event_percpu` 是真正调用**你的 handler**
的地方：

```c
irqreturn_t __handle_irq_event_percpu(struct irq_desc *desc)
{
    irqreturn_t retval = IRQ_NONE;
    for_each_action_of_desc(desc, action) {           /* 遍历共享链 */
        res = action->handler(irq, action->dev_id);   /* ★调用你写的 top half */
        WARN_ONCE(!irqs_disabled(), "...handler enabled interrupts");
        if (res == IRQ_WAKE_THREAD) __irq_wake_thread(desc, action);  /* Topic 07 */
        retval |= res;
    }
    return retval;
}
```

### 晦涩点 ⑥："Handlers run with interrupts disabled" + 那个 `WARN_ONCE`

你的中断 handler（top half）运行时**中断是关闭的**（硬中断上下文）。那个
`WARN_ONCE(!irqs_disabled(), ...)` 是一道岗哨：如果你的 handler 居然把中断重新打开了
（几乎总是 bug），跑完一检查发现中断开着，立刻报警。**推论（很重要）**：handler 里
**不能睡眠、不能做耗时操作**——因为它霸占着 CPU 且关着中断，拖久了整个系统的中断
响应都会变差（Topic 14 延迟）。要做慢活/睡眠的活，返回 `IRQ_WAKE_THREAD` 交给线程
（Topic 07）或丢给 bottom half（Topic 08/09）。

### 晦涩点 ⑦：`retval |= res` 这个按位或在干什么？

共享线上每个 handler 返回 `IRQ_HANDLED`（是我的）或 `IRQ_NONE`（不是我的）。
`retval |= res` 把所有返回值**或**起来：只要**有一个人**认领了（HANDLED），
最终 retval 就是 HANDLED；只有**全员都说不是我的**，retval 才保持 IRQ_NONE。
这个 "全员 NONE" 的结果会喂给 §5 的假中断侦探——这正是 Topic 03 晦涩点 ③
那个测谎仪的输入来源。

---

## 4. 共享中断与假中断侦探（§4 + §5 合讲）

这两节其实是 Topic 02/03 已铺垫概念的代码落地，快速串一遍：

**共享中断**（§4）：一条线挂多个 `irqaction`（Topic 03 晦涩点 ⑫）。flow handler 每次
都把**整条链全跑一遍**，每个 handler 自检"是我的吗"。代价：① 别人的中断也害你的
handler 被调用（延迟）；② 一个耍流氓的 handler 无脑返回 HANDLED，会**掩盖**其他设备的
"没人理"，破坏侦探判案。MSI（Topic 11）的出现部分就是为了**消灭共享**——每个设备
独占自己的中断。

**假中断侦探**（§5，`spurious.c:160`）：§3 算出的 retval 若为 IRQ_NONE，
`note_interrupt()` 让 `irqs_unhandled++`。短时间内堆积约 10 万次没人认领，断定这条线
**疯了**（screaming，Topic 02 §1.2 见过这个词），打印：

```
irq N: nobody cared (try booting with the "irqpoll" option)
```

然后**禁用这条线**保命。这是触发方式配错（Topic 02/04 那些坑）或硬件故障的兜底。

---

## 5. Resend——补送丢失的中断（§6）

### 晦涩点 ⑧：为什么需要"补送"？

边沿中断在 `disable_irq` 期间敲了门（§2.2 那个 `IRQS_PENDING` 分支），或 fasteoi
遇到亲和性切换竞争——这些"该处理但当时没法处理"的中断被记成 PENDING。
`check_irq_resend()` 事后补送，两条路：

1. **优先硬件补送**：调 `irq_chip->irq_retrigger`，让硬件重新拉一次这条线（Topic 03
   的 irq_chip 回调之一）。
2. **软件补送兜底**（`CONFIG_HARDIRQS_SW_RESEND`）：把 desc 挂到 resend 链上，
   软件路径重跑 handler。挂起/恢复时（Topic 15）和硬件不支持重触发时用这条。

一句话：**resend 是边沿中断穿越 disable/enable 窗口和挂起/恢复而不丢事件的保险。**

---

## 6. 动手任务

### 任务 0：从 `/proc/interrupts` 给每个中断认出 flow handler（10 分钟，无需 root）

```bash
cat /proc/interrupts | awk '{print $NF, $(NF-1)}' | sort | uniq -c | sort -rn | head
# 更直接地看 fasteoi/edge/level 后缀：
grep -oE "(fasteoi|edge|level)" /proc/interrupts | sort | uniq -c
```

回答（答案 §7.0）：

1. 你机器上哪种 flow handler 最多？符合原文"fasteoi 是服务器最常见"的说法吗？
2. 找一个 `edge` 的和一个 `fasteoi` 的中断，分别是什么设备？回忆 §2 的画面，
   这两个设备的中断处理顺序有什么不同？

### 任务 1：三个 handler 并排读，列"顺序差异表"（30 分钟，核心任务）⭐

打开 `kernel/irq/chip.c`，把这三个函数**并排**放在屏幕上：

```
handle_level_irq    :687
handle_fasteoi_irq  :738
handle_edge_irq     :825
```

填写下面这张表（答案 §7.1）——这张表填完，本章就掌握了：

| 步骤 | level | edge | fasteoi |
|---|---|---|---|
| 进门第一个芯片操作是？ | | | |
| 跑 handler 前 mask 吗？ | | | |
| 跑 handler 后做什么收尾？ | | | |
| 有没有"重跑"循环？ | | | |
| IRQS_PENDING 在哪用到？ | | | |

### 任务 2：用 ftrace 抓 flow handler 实际运行（20 分钟，需要 root）

```bash
sudo -i; cd /sys/kernel/tracing
echo function_graph > current_tracer
# 抓最常见的那个（先用任务0确认你机器主力是哪个）：
echo handle_fasteoi_irq > set_graph_function
echo 1 > tracing_on; sleep 1; echo 0 > tracing_on
head -80 trace
echo nop > current_tracer; echo > set_graph_function
```

回答（答案 §7.2）：

1. 在 `handle_fasteoi_irq` 的调用树里找到 `handle_irq_event` → 再往下找到具体的驱动
   handler 函数名（如 `e1000_intr`、`nvme_irq`）。这就是 §3 说的 "your top half"。
2. 找到 EOI 相关的调用（`cond_unmask_eoi_irq` 或芯片的 eoi 回调）——它在驱动 handler
   **之前**还是**之后**？印证 §2.3 哪句话？
3. 整个 `handle_fasteoi_irq` 耗时多少微秒？handler 占了多大比例？

### 任务 3：故意制造 "nobody cared"（40 分钟，需要 root，写内核模块）⭐⭐

> ⚠️ 在虚拟机里做。这个任务会故意触发内核保护机制，安全但会刷日志。

写一个模块，用 `IRQF_SHARED` 在一条**已存在的共享中断**上注册一个**永远返回
`IRQ_NONE`** 的 handler，然后……其实更安全、更可控的做法是用软件触发一个孤儿中断。
推荐做下面这个**受控版**：

```c
// 复用 Topic 04 任务 3 的 domain demo 骨架，但 handler 永远返回 IRQ_NONE：
static irqreturn_t bad_handler(int irq, void *dev) {
    return IRQ_NONE;   /* 永远"不是我的" */
}
// 然后在一个 timer 或循环里反复 generic_handle_domain_irq() 触发它
// （注意关中断上下文要求），观察 irqs_unhandled 累积。
```

更简单的**纯观察版**（不写模块，强烈推荐先做这个）：

```bash
# 找一条计数为 0 或很低的中断，看它的 spurious 统计
N=某中断号
sudo cat /sys/kernel/debug/irq/irqs/$N 2>/dev/null | grep -iE "unhandled|count|istate"
```

回答（答案 §7.3）：

1. `note_interrupt` 在 `spurious.c` 里判定"疯了"的阈值是多少次？用什么字段做时间
   老化（防止偶发误判）？（读 `spurious.c` 找 `99900` / `100000` 和 `last_unhandled`。）
2. 触发保护后，内核对这条线做了什么？(`__disable_irq` / 改 flow handler 成 poll？)
3. `irqpoll` 启动选项让内核改用什么策略？为什么这是"牺牲一个设备保全系统"？

### 任务 4：思考题（答案 §7.4）

1. 假如把一个**边沿**设备错配成 `handle_level_irq`，会发生什么？反过来把**电平**
   设备配成 `handle_edge_irq` 呢？（分别对照 §2.1/§2.2 的死循环和丢事件画面。）
2. fasteoi 为什么不需要 `IRQS_PENDING` 重跑循环，而 edge 需要？（提示：聪明门卫
   在硬件做了什么，让"处理期间又来一个"这件事不需要软件操心？——但 §2.3 末尾那个
   `check_irq_resend` 又是为什么存在？）
3. 共享中断链上有 5 个 handler，第 3 个无脑返回 `IRQ_HANDLED`。第 4、5 个设备真正
   出问题（线卡住）时，"nobody cared"还会触发吗？后果是什么？
4. `guard(raw_spinlock)` 相比手写 lock/unlock，除了"不会忘记解锁"，对**多个 return
   分支**的函数还有什么特别价值？（数一数 `handle_fasteoi_irq` 有几个 return。）

---

## 7. 任务参考答案

### 7.0 任务 0

1. 绝大多数现代 x86 机器 **fasteoi 最多**（所有 IO-APIC 和很多 MSI 走它），edge 次之
   （部分 MSI），level 很少。符合原文。
2. edge 常见于部分 MSI/GPIO；fasteoi 常见于 IO-APIC 设备和 GIC。处理顺序差异见 §2：
   edge 是"ack→招待→查PENDING可能重跑"，fasteoi 是"招待→EOI"，level（若有）
   是"mask→招待→unmask"。

### 7.1 任务 1

| 步骤 | level | edge | fasteoi |
|---|---|---|---|
| 进门第一个芯片操作 | `mask_ack_irq`（mask+ack）| `irq_ack`（仅 ack）| 无（直接判断）|
| 跑 handler 前 mask 吗 | **是**（必须）| 否 | 否（除非 ONESHOT）|
| 跑 handler 后收尾 | `cond_unmask_irq` | 检查 PENDING 决定是否重跑 | `cond_unmask_eoi_irq`（EOI）|
| 重跑循环 | 无 | **有**（do-while PENDING）| 无（但有 resend 兜底）|
| IRQS_PENDING 用处 | 基本不用 | 记录漏掉的边沿，驱动重跑 | 仅亲和性竞争时补送 |

### 7.2 任务 2

1. 调用树形如 `handle_fasteoi_irq → handle_irq_event → handle_irq_event_percpu →
   __handle_irq_event_percpu → <驱动handler>`。最后那个就是你的 top half。
2. EOI 在驱动 handler **之后**（`handle_irq_event` 返回后才 `cond_unmask_eoi_irq`）。
   印证 "just run the handlers and irq_eoi"——招待完才说"下一位"。
3. 典型几微秒到几十微秒；handler 通常占大头。超过几百微秒的要警惕（Topic 14）。

### 7.3 任务 3

1. 阈值在 `spurious.c`：约 100000 次中断里 99900 次未处理就判定 spurious
   （`note_interrupt` 里的计数逻辑）；`last_unhandled` 时间戳做老化——超过
   `HZ/10` 没新的未处理事件就重置计数，避免把"久久偶发一次"误判成风暴。
2. 触发后调 `__disable_irq` 类路径**关掉这条线**，并把统计标记上，日志打印
   "nobody cared"。
3. `irqpoll` 让内核**定时轮询所有 handler**（而非依赖中断触发），用 CPU 换取
   "即使某条线疯了也能继续收别的中断"。牺牲在于轮询有 CPU 开销和延迟，但比整机
   被中断风暴卡死强——"一个死设备 vs 活系统"的取舍。

### 7.4 任务 4

1. 边沿错配成 level：level handler 进门就 mask 并期待 handler 清设备状态，但边沿
   设备没有"持续拉高的线"可清，且 level 的 unmask 逻辑与边沿语义不符——可能丢事件
   或卡住。电平错配成 edge：edge 只 ack 不 mask，电平线**一直高着**没被屏蔽 →
   handler 跑完线还高 → 取决于硬件可能立即风暴或被 ack 压住后再不触发而丢事件。
   两个方向都坏，这就是原文反复强调"触发方式必须配对"的原因。
2. fasteoi 不需要重跑：聪明门卫用硬件优先级保证"我正处理时同级不进来"，所以不存在
   "处理期间又来一个被丢"的窗口。末尾的 `check_irq_resend` 处理的是**另一个**问题
   ——亲和性切换时，中断在新旧 CPU 间竞争，可能出现一次需要补送的边角情况，
   不是常规重跑。
3. 不会触发。第 3 个无脑 HANDLED 让 retval 永远是 IRQ_HANDLED，侦探看到"有人认领"
   就不计 unhandled。后果：第 4/5 个设备的线卡住却**永远不会被"nobody cared"保护**
   ，表现为中断风暴拖垮 CPU 而无任何日志——这就是原文 Pitfalls 第一条"无脑返回
   HANDLED 会掩盖别人"的真实危害。
4. `handle_fasteoi_irq` 有 4~5 个 return 分支（PM 不能处理、无 action、正常、resend）。
   手写 lock/unlock 要在**每个** return 前都记得解锁，漏一个就死锁；`guard` 一次声明，
   所有出口自动解锁——分支越多，价值越大。这是现代内核大量采用 guard 的原因。

---

## 8. 学完自测

1. 并排默写 level/edge/fasteoi 的操作顺序，各自为什么是这个顺序？
2. 电平防风暴靠什么、边沿防丢失靠什么？这对应 Topic 02 哪对概念？
3. `retval |= res` 和 "nobody cared" 之间是什么因果链？无脑 HANDLED 为什么危险？
4. fasteoi 里那行 `IRQS_ONESHOT` mask 是给哪一章留的钩子？
5. resend 保护边沿中断穿越哪两种场景？

下一章 Topic 06：这些 flow handler 和 action 链，最初是怎么通过 `request_irq` 装上去的。
