# Topic 06 扩展版 — 注册与管理 handler：驱动作者的 API

> **用法同前**。行号基于 `~/repo/linux`（v7.1-rc7）：`request_threaded_irq` 在
> `kernel/irq/manage.c:2115`、`__setup_irq` 在 `manage.c:1471`、
> `request_any_context_irq` 在 `manage.c:2222`；`IRQF_*` 宏定义在
> `include/linux/interrupt.h:44` 起。
>
> **承接**：Topic 05 讲了 flow handler 怎么跑你的 handler、怎么遍历 action 链。
> 本章讲**这些 action 和 flow handler 最初是怎么装上去的**——也就是 99% 驱动作者
> 唯一会直接接触的中断 API。前五章是"机器内部"，这一章是"用户面板"。

---

## 1. 先建立画面感：一个函数撑起整个门面

### 1.1 "认领一路中断"要办哪些手续？

回到 Topic 03 的比喻：驱动想说"这路铃响了找我"，要去档案室办**认领登记**
（建一张 `irqaction` 挂到档案夹上）。这件事的唯一核心入口是
`request_threaded_irq()`，其他全是它的薄包装。最常用的 `request_irq` 的真身
（`include/linux/interrupt.h:173`）：

```c
static inline int request_irq(unsigned int irq, irq_handler_t handler,
                              unsigned long flags, const char *name, void *dev)
{
    return request_threaded_irq(irq, handler, NULL,   /* thread_fn = NULL */
                                flags | IRQF_COND_ONESHOT, name, dev);
}
```

看出来了吗——`request_irq` 就是"**thread_fn 传 NULL** 的 request_threaded_irq"。
一个函数，几种用法：

- 只传 `handler`（thread_fn=NULL）= 纯硬中断处理 = 本章；
- 同时传 `handler` 和 `thread_fn` = 线程化中断 = Topic 07。

### 1.2 `request_irq` 在幕后办了哪几道手续？

`__setup_irq`（`manage.c:1471`）干的事，串起了前几章所有概念：

1. **建一张登记条**（`irqaction`）：填上你的 handler、flags、name、dev_id（Topic 03）。
2. **共享线兼容性检查**：如果这条线已有别人认领，核对"大家的 IRQF_SHARED 和触发方式
   是否兼容"（不兼容返回 `-EBUSY`）。
3. **线程化则建内核线程**（Topic 07）。
4. **配置硬件**：调芯片的 `irq_request_resources`、设置触发类型、挂上/保留 flow handler
   （Topic 05 选哪个）、**默认使能这条线**（除非你给了 `IRQF_NO_AUTOEN`）。

### 晦涩点 ①：它收 virq，不是 hwirq

原文 §1 末句很关键：`request_irq` 的第一个参数是 **virq**（集团工号，Topic 04），
**绝不是 hwirq**（分机号）。这个 virq 你从哪拿？从 Topic 04 那些固件接口函数：
`irq_of_parse_and_map()`（设备树）、`pci_irq_vector()`（PCI/MSI）、
`platform_get_irq()`（平台设备）。**驱动作者基本不直接碰 hwirq**——翻译官（domain）
已经替你把分机号换成工号了。

---

## 2. 逐句拆解 request 家族

原文 §2 的表格已全，补充每个的"什么时候选它"判断：

### 晦涩点 ②：`request_any_context_irq` 是什么"any context"？

> 让核心根据配置决定走硬中断还是线程。

有些驱动**两种都能接受**——它的 handler 写得既能在硬中断上下文跑、也能在线程上下文跑。
这类驱动调 `request_any_context_irq`（`manage.c:2222`），让内核根据"这条线是否被
强制线程化"（Topic 07，PREEMPT_RT 下全部线程化）**自动选**。返回值会告诉你最终
选了哪种（`IRQC_IS_HARDIRQ` / `IRQC_IS_NESTED`）。GPIO 库这类要适配各种内核配置的
通用代码常用它。

### 晦涩点 ③：为什么强烈建议用 `devm_` 版本？

`devm_request_irq` 是**资源托管版**（`kernel/irq/devres.c`）：中断登记和**驱动的生命周期
绑定**，驱动卸载/probe 失败时**自动 free_irq**。原文说它消灭了头号清理 bug——
"忘记在 probe 失败路径上 free_irq"。

**画面**：普通 `request_irq` 是"自己借的东西自己记得还"；`devm_` 是"押金托管，
退房时自动结算"。probe 函数里常有十几个错误跳转分支，每个都手动 free 极易漏——
`devm_` 让你彻底不用管。新驱动**默认用 devm_**。

### 晦涩点 ④：`__must_check` —— 编译器逼你检查返回值

原文 §2 末："The `__must_check` attribute means you must check the return"。
`request_irq` 声明带了 `__must_check`，你要是写 `request_irq(...);` 不接返回值，
**编译器直接警告**。为什么这么严？因为忽略失败 = 设备静默地没有 handler，
中断来了无人处理，表现为"设备完全不工作但毫无报错"——最难查的一类 bug。
常见失败码：`-EBUSY`（共享 flag 不匹配）、`-ENOMEM`、`-EINVAL`。

---

## 3. 逐句拆解 `IRQF_*` 标志

`include/linux/interrupt.h:44` 起。原文按功能分了组，这里只深挖三个最常踩的：

### 晦涩点 ⑤：`IRQF_SHARED` 的连带义务

声明"这条线可共享"。但有**三个连带义务**（Topic 03/05 讲过后果，这里讲规则）：
① 所有共享者**都**必须设 IRQF_SHARED（一个没设就 -EBUSY）；② `dev_id` 必须
**唯一且非 NULL**（否则 free_irq 不知道撕哪张条子，Topic 03 晦涩点 ⑫）；
③ 大家的触发方式要兼容。MSI 时代共享越来越少，但老 PCI INTx 设备仍需要它。

### 晦涩点 ⑥：`IRQF_ONESHOT` —— Topic 05 那个钩子的开关

还记得 Topic 05 晦涩点 ⑤ 里 fasteoi 那行 `if (IRQS_ONESHOT) mask_irq`？
`IRQF_ONESHOT` 就是点亮那个钩子的开关。含义：**硬中断 handler 跑完后不要立刻解除屏蔽，
一直屏蔽到线程化 handler 也跑完**。

为什么需要？线程化中断里，硬中断只是"唤醒线程"就返回了（Topic 07），但设备的中断
条件还没被真正处理（要等线程去处理）。如果这时解除屏蔽，设备会**反复触发**同一个
还没处理的中断。ONESHOT 保证"一次中断只触发一次，等线程消化完再开闸"。
原文：当 `thread_fn` 非空且没有 primary handler 时，**ONESHOT 是强制的**。

### 晦涩点 ⑦：`IRQF_NO_AUTOEN` 解决"还没准备好就来中断"

默认 `request_irq` 会**立即使能**中断线。但有的设备：你刚注册完 handler、设备本身
还没初始化好，中断就可能来——handler 拿到半初始化的设备状态会崩。

`IRQF_NO_AUTOEN` 让 request 时**不自动使能**，你把设备彻底准备好后再手动
`enable_irq()`。这是原文 Pitfalls 里"请求 IRQ 的时机"问题的标准解法。

### 晦涩点 ⑧：`IRQF_TIMER` 是个"打包套餐"

`interrupt.h:90`：`IRQF_TIMER = __IRQF_TIMER | IRQF_NO_SUSPEND | IRQF_NO_THREAD`。
定时器中断的特殊需求打包：① `NO_SUSPEND`——系统挂起时定时器还得走（Topic 15）；
② `NO_THREAD`——绝不能线程化（调度器自己依赖定时器，把它线程化会造成鸡生蛋，Topic 07）。
一个 flag 表达"我是特殊的定时器中断，按特殊规矩伺候我"。

---

## 4. 逐句拆解 top half 契约（最重要的一节）

这一节是**每个新手驱动作者必须刻进肌肉记忆**的。Topic 05 晦涩点 ⑥ 讲了"为什么"，
这里是完整的"清单"。你的 handler：

```c
static irqreturn_t my_handler(int irq, void *dev_id) { ...; return IRQ_HANDLED; }
```

运行在**硬中断上下文**、**中断关闭**、**线常被屏蔽**。因此四条铁律：

### 晦涩点 ⑨：为什么"绝对不能睡眠"？

handler 借用着某个倒霉线程的执行流（Topic 00 的上下文），还关着中断。**睡眠 = 让出 CPU
给调度器**——但你现在身处中断上下文，调度器一旦在这里被调用，内核会直接 panic
（"scheduling while atomic"）。所以：

- **禁止**：`mutex_lock`、`msleep`、`wait_event`、`kmalloc(GFP_KERNEL)`（可能为等内存而睡）。
- **允许**：`spin_lock`（注意用普通版**不用** `_irqsave`，因为中断本来就关着）、
  `kmalloc(GFP_ATOMIC)`（不睡眠的分配）、非阻塞操作。

要睡眠/做慢活？→ 返回 `IRQ_WAKE_THREAD` 交给线程（Topic 07），或丢给工作队列（Topic 09）。

### 晦涩点 ⑩：四个返回值的含义

- `IRQ_HANDLED`：是我的，处理了。
- `IRQ_NONE`：不是我的（共享线上必须正确返回这个，Topic 05 §4 的侦探依赖它）。
- `IRQ_WAKE_THREAD`：紧急部分做完了，剩下的交给线程（Topic 07）。
- `IRQ_RETVAL(x)`：把一个 bool 转成 HANDLED/NONE 的便捷宏。

口诀（原文）：**"应答设备、抓取最少的数据（或仅唤醒线程）、赶紧出去"**。重活留给 bottom half。

---

## 5. 逐句拆解释放与门控

`manage.c` 里这组函数是 request 的逆操作和"闸门"：

### 晦涩点 ⑪：`disable_irq` vs `disable_irq_nosync` —— "sync"指等什么？

```c
void disable_irq(unsigned int irq);        /* 同步：还会等正在跑的 handler/线程跑完 */
void disable_irq_nosync(unsigned int irq); /* 异步：立刻返回 */
```

两者都在芯片层屏蔽这条线。区别在于 **`disable_irq` 还会等当前正在执行的 handler
彻底跑完才返回**（"sync"=同步等待）。

**为什么这个区别要命？** 假设你要释放一块 handler 会访问的数据。如果用
`disable_irq`（sync），返回后你**确定**没有 handler 正在碰这块数据，可以安全释放。
如果用 `disable_irq_nosync`，可能有个 handler 还在另一个 CPU 上跑着——你一释放它就
访问已释放内存。所以 nosync 之后想动共享数据，必须额外调 `synchronize_irq()` 等一等。

### 晦涩点 ⑫：为什么 `disable_irq` 不能在 handler 里调？

因为它会**阻塞等待 handler 跑完**——而你**就在 handler 里**。等你自己跑完？死锁。
原文 Pitfalls 明列：`free_irq`、同步版 `disable_irq` 都可能阻塞，**绝不能从
硬中断/软中断上下文调用**。

### 晦涩点 ⑬：`synchronize_irq` —— 不关闸，只等清场

```c
void synchronize_irq(unsigned int irq);  /* 不改变使能状态，只等"现在没有 handler 在跑" */
```

它在 `desc->wait_for_threads`（Topic 03 见过这个等待队列字段）上等待。用途：
"我只想确保此刻没有 handler 正在运行"，但不想禁用中断。比如更新某个 handler 会读的
配置，更新前 `synchronize_irq` 确保旧 handler 都退场了。

### 晦涩点 ⑭：`depth` 计数的实际后果

`disable_irq` 是**计数的**（Topic 03 晦涩点 ②的 `desc->depth`）：N 次 disable 需要 N 次
enable 才真正开闸。后果（原文 Pitfalls）：**漏一次 enable，这条线永远关着**
（设备静默死亡）；**多一次 enable，内核警告**（"Unbalanced enable for IRQ"）。
disable/enable 必须像括号一样严格配对。

---

## 6. 动手任务

### 任务 0：在 /proc 和 /sys 里观察 request_irq 的痕迹（10 分钟，无需 root）

```bash
# 找一个共享中断（actions 里有多个名字）：
for d in /sys/kernel/irq/*/; do
  n=$(cat $d/actions 2>/dev/null)
  [ "${n//,/}" != "$n" ] && echo "irq $(basename $d): $n"
done | head
cat /sys/kernel/irq/*/actions | tr ',' '\n' | grep -c .   # 总 action 数
```

回答（答案 §7.0）：1. 你机器上有共享中断吗（actions 含逗号）？是什么设备共享？
2. 随便选个中断 N，`cat /sys/kernel/irq/N/name`——这个名字是 `request_irq` 的哪个参数？

### 任务 1：源码追 request_irq → __setup_irq（30 分钟）

```
① include/linux/interrupt.h:173   request_irq 内联定义（确认它转调 threaded 版）
② kernel/irq/manage.c:2115        request_threaded_irq：找到它分配 action、
                                   调 __setup_irq 的地方
③ kernel/irq/manage.c:1471        __setup_irq：找到三件事——
                                   (a) 共享线的 flags 兼容性检查（搜 IRQF_SHARED / mismatch）
                                   (b) 线程创建（搜 setup_irq_thread / kthread）
                                   (c) 默认使能（搜 IRQF_NO_AUTOEN / irq_startup）
```

回答（答案 §7.1）：

1. 共享线兼容性检查在比较哪些字段不匹配就返回 -EBUSY？
2. `IRQF_NO_AUTOEN` 在 `__setup_irq` 里如何改变使能行为？
3. `request_irq` 偷偷加的 `IRQF_COND_ONESHOT` 是什么意思？（读 interrupt.h:71
   附近的注释。）

### 任务 2：写一个会被编译器骂的 handler，体会 top half 契约（40 分钟，写模块）⭐

基于 Topic 04 任务 3 的骨架，故意在 handler 里写违规代码，观察后果：

```c
static irqreturn_t bad_handler(int irq, void *dev) {
    msleep(10);              /* 违规①：硬中断里睡眠 */
    kmalloc(4096, GFP_KERNEL); /* 违规②：可能睡眠的分配 */
    return IRQ_HANDLED;
}
```

但**更安全的做法**（不会真把系统搞挂）是只**编译**它、用 sparse/检查工具看，
或者在模块里**只注册不真正触发**，然后阅读如果触发会发生什么。如果你想真看到崩溃，
**务必在虚拟机里**触发一次，观察 dmesg 里的 "scheduling while atomic" 或
"BUG: sleeping function called from invalid context"。回答（答案 §7.2）：

1. `request_irq` 不检查返回值时，编译会怎样？（把 `int ret = request_irq(...)` 改成
   `request_irq(...);` 试试，看 `__must_check` 警告。）
2. 如果 handler 真的睡眠了，内核报的 BUG 信息里关键字是什么？
3. 把违规代码改成"返回 IRQ_WAKE_THREAD + 在 thread_fn 里 msleep"，还会报错吗？
   为什么？（这是 Topic 07 的预告。）

### 任务 3：实测 disable_irq 的计数语义（30 分钟，需要 root，谨慎）

> ⚠️ 虚拟机里做。选一个**不影响系统存活**的设备中断（如一个空闲 USB 口）。

```bash
N=某个低风险中断号
# 观察当前 depth（debugfs）：
sudo cat /sys/kernel/debug/irq/irqs/$N 2>/dev/null | grep -i depth
```

或者用一个内核模块调 `disable_irq(N)` 两次、`enable_irq(N)` 一次，再读 depth，
观察"两次 disable 一次 enable 后线仍是关的"。回答（答案 §7.3）：

1. 两次 disable + 一次 enable 后，depth 是多少？这条线是开还是关？
2. 此时再 `enable_irq` 一次，depth 变成？线状态？
3. 如果 enable 比 disable 多一次，内核打印什么警告？

### 任务 4：思考题（答案 §7.4）

1. 为什么 `request_irq` 收 virq 而不是 hwirq？如果让驱动直接用 hwirq 会有什么灾难？
   （回忆 Topic 04 的撞号问题。）
2. `devm_request_irq` 自动释放，那它在**什么时刻**释放？如果你的设备在驱动卸载前就该
   停止中断，光靠 devm 够吗？
3. 共享线上 A 用 `IRQF_ONESHOT`、B 不用，能共享吗？`IRQF_COND_ONESHOT` 在这里起什么
   作用？
4. 为什么定时器中断要 `IRQF_NO_THREAD`？把它线程化会发生什么"鸡生蛋"问题？
   （提示：线程要靠谁来调度上 CPU？)

---

## 7. 任务参考答案

### 7.0 任务 0

1. 现代 MSI 机器共享中断已很少；若有，常见于老 PCI 设备、USB、ACPI。actions 字段
   含逗号即共享。2. `name` 就是 `request_irq(..., const char *name, ...)` 的第四个参数
   ——`/proc/interrupts` 右列显示的也是它。

### 7.1 任务 1

1. 主要比较触发类型（trigger）和 `IRQF_SHARED`/`IRQF_PERCPU`/`IRQF_ONESHOT` 等
   关键 flag 是否一致；不一致或有一方没设 SHARED → -EBUSY。
2. 设了 NO_AUTOEN 就**跳过** `irq_startup`/使能步骤，只把 action 挂好，线保持禁用
   （`__irq_disable` 状态），等驱动手动 `enable_irq`。
3. COND_ONESHOT = "如果这条共享线上**已经有人**要求了 ONESHOT，我就跟着接受
   ONESHOT；否则不强加"——让纯硬中断驱动能和线程化驱动共享同一条线而不冲突。

### 7.2 任务 2

1. `warning: ignoring return value of 'request_irq', declared with attribute
   warn_unused_result`（即 `__must_check`）。
2. `BUG: scheduling while atomic` 或 `BUG: sleeping function called from invalid
   context at ...`，并打印调用栈。
3. 不会报错。`thread_fn` 运行在**专属内核线程**（进程上下文，可被调度、可睡眠），
   所以那里 `msleep`/`mutex` 合法——这正是线程化中断（Topic 07）存在的全部意义。

### 7.3 任务 3

1. depth=1，线**关闭**（需要的 enable 次数还差一次）。
2. depth=0，线**开启**。
3. `Unbalanced enable for IRQ N`（`enable_irq` 检测到 depth 已经是 0 还来 enable）。

### 7.4 任务 4

1. virq 是全局唯一工号，hwirq 会撞号（每个 GIC 都有 SPI 32，Topic 04 §1.1）。
   若驱动用 hwirq，内核无法知道"是哪个控制器的 32 号"，`irq_to_desc` 索引到错误档案
   或根本无法索引。virq 是 domain 翻译后的唯一标识，这是 request_irq 收 virq 的根本原因。
2. devm 在**驱动与设备解绑时**（remove/probe 失败）自动释放。如果你需要在卸载**之前**
   （比如运行中要重配设备）停中断，仍需手动 `disable_irq`/`free_irq`——devm 只管最终
   清理，不管运行中的生命周期控制。
3. 能共享：B 通过 `IRQF_COND_ONESHOT`（request_irq 隐式带上）"同意在已有 ONESHOT 时
   也接受 ONESHOT"。于是整条线按 ONESHOT 处理，B 也接受这个约束。这就是
   COND_ONESHOT 的设计目的。
4. 内核线程要靠**调度器**才能跑上 CPU，而调度器的时间片、抢占都依赖**定时器中断**
   按时到来。如果定时器中断被线程化，它就要等调度器调度它的线程——可调度器又在等
   定时器中断……死锁式的鸡生蛋。所以定时器中断必须留在硬中断上下文，NO_THREAD 强制
   这一点。

---

## 8. 学完自测

1. `request_irq` 和 `request_threaded_irq` 的关系？它收的第一个参数是 virq 还是 hwirq，
   为什么？
2. top half 四条铁律是什么？为什么不能睡眠？要睡眠怎么办？
3. `disable_irq` 和 `disable_irq_nosync` 的"sync"指等什么？什么场景必须用 sync 版？
4. `IRQF_ONESHOT` / `IRQF_NO_AUTOEN` / `IRQF_TIMER` 各解决什么问题？
5. 为什么驱动里优先用 `devm_request_irq`？`__must_check` 在防什么 bug？

下一章 Topic 07：`request_threaded_irq` 的另一半——`thread_fn`，以及 PREEMPT_RT
如何把"线程化"变成所有中断的默认形态。
