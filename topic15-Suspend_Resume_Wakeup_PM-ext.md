# Topic 15 扩展版 — 挂起/恢复、唤醒与中断电源管理

> **用法同前**。行号基于 `~/repo/linux`（v7.1-rc7）：`kernel/irq/pm.c` 里
> `suspend_device_irq:65`、`suspend_device_irqs:126`、`resume_irq:144`、
> `resume_device_irqs:246`；`irq_set_irq_wake` 在 `kernel/irq/manage.c:868`；
> `IRQD_WAKEUP_STATE`(BIT14)/`IRQD_WAKEUP_ARMED`(BIT19) 在 `include/linux/irq.h:235/239`；
> GIC PM 在 `drivers/irqchip/irq-gic-pm.c`。
>
> **承接**：复用 Topic 06 的 `IRQF_NO_SUSPEND`/`COND_SUSPEND` 标志、Topic 05 的 resend、
> Topic 03 的 `IRQD_WAKEUP_*` 状态和 `wake_depth`。

---

## 1. 先建立画面感：系统睡觉时，中断怎么办？

### 1.1 睡觉时的两难

系统挂起（笔记本合盖、手机锁屏）时，绝大多数中断**必须噤声**——否则一个乱来的中断会
中止挂起过程或弄坏它。但**有几个必须保持清醒**：

- **维持系统时间的定时器**（睡着也要知道过了多久）；
- **唤醒源**（电源键、RTC 闹钟、网卡的 Wake-on-LAN、笔记本翻盖开关）——它们的任务
  **就是把机器叫醒**。

**画面**：你睡觉前关掉家里所有电器（中断噤声），但**留着门铃**（唤醒源）和**闹钟**
（NO_SUSPEND 定时器）。门铃响了你就醒（唤醒），闹钟保证你知道睡了多久。本章就是
genirq 怎么划这条"谁睡谁醒"的线。

```
挂起：冻结任务 → 挂起设备 → suspend_device_irqs() ──► 屏蔽一切
      除了 IRQF_NO_SUSPEND 和已武装的唤醒中断
   ... CPU 睡眠；一个武装的唤醒中断（或 NO_SUSPEND 定时器）可以触发 ...
恢复：resume_device_irqs() ──► 重新使能；补送睡眠中被屏蔽时触发过的（resend）
```

### 1.2 全章最重要的区分：两个完全不同的"开关"

原文反复强调（Pitfalls 第 4 条尤其）：**"睡眠期间保持运行" 和 "把系统从睡眠中唤醒"
是两个不同的开关**，新手最容易混：

| 开关 | 函数/标志 | 含义 | 画面 |
|---|---|---|---|
| **保持运行** | `IRQF_NO_SUSPEND` | 在**挂起流程本身**进行时这条线还活着 | 闹钟（睡觉时仍走时）|
| **唤醒系统** | `enable_irq_wake()` | 能把已经睡死的机器**叫醒** | 门铃（睡死了也能把你吵醒）|

设错了开关是"我的 USB/RTC 唤不醒笔记本"这类经典 bug 的根源。记住：**要唤醒用
`enable_irq_wake`，不是 `IRQF_NO_SUSPEND`。**

---

## 2. 逐句拆解唤醒中断

### 晦涩点 ①：`enable_irq_wake` 做了什么

```c
int enable_irq_wake(unsigned int irq);    /* = irq_set_irq_wake(irq, 1) */
int disable_irq_wake(unsigned int irq);
```

机制（`irq_set_irq_wake`，`manage.c:868`）：

1. **计数**（`desc->wake_depth`，Topic 03 见过，和 `disable_irq` 的 depth 同理）——
   enable/disable 要配对。
2. 设置 `irq_data` 上的 **`IRQD_WAKEUP_STATE`**（`irq.h:235`），并调芯片的 **`irq_set_wake`**
   （Topic 03 §4）——让控制器把这条线配成**可唤醒**（比如 GPIO/PMIC 把它接到
   **常开的唤醒逻辑**上）。不需要逐线配置的芯片设 `IRQCHIP_SKIP_SET_WAKE`。
3. 挂起时，唤醒中断被**武装**（`IRQD_WAKEUP_ARMED`，`irq.h:239`）而不是屏蔽——
   保持活着以唤醒系统。

**画面**：`enable_irq_wake` 像告诉门禁系统"这个按钮要接到主卧的紧急呼叫器上"
（`irq_set_wake` 配置常开唤醒逻辑），睡觉时这个按钮**带电武装**（ARMED），按下就叫醒你。

### 晦涩点 ②：用户态的闸门

驱动声明"我能唤醒"还不够，用户/系统策略可以**否决**它：
`/sys/devices/.../power/wakeup`（enabled/disabled）控制是否**采纳**驱动的唤醒能力。
`/sys/power/wakeup_count` 和 `/sys/kernel/debug/wakeup_sources` 记录谁唤醒了机器。
这就是为什么你能在系统设置里"允许此设备唤醒计算机"——那个勾就是写这个 sysfs。

---

## 3. 逐句拆解挂起/恢复核心

### 晦涩点 ③：`suspend_device_irq` 的逐线决策

`kernel/irq/pm.c:126`，PM 核心在挂起晚期/恢复早期调用（设备之后）：

```c
void suspend_device_irqs(void)
{   for each irq_desc:
        sync = suspend_device_irq(desc);   /* 屏蔽，除非 NO_SUSPEND 或已武装唤醒 */
        if (sync) synchronize_irq(irq);     /* 确保没有 handler 还在跑 */
}
```

`suspend_device_irq`（`pm.c:65`）对每条线判断：

- **`IRQF_NO_SUSPEND`** 的 action（Topic 06）：线**保持使能**穿过挂起——给定时器
  （`IRQF_TIMER` 自带 NO_SUSPEND）和挂起路径本身需要运行的基础设施用。
- **已武装唤醒**（`IRQD_WAKEUP_ARMED`，§2）：保持使能，**作为唤醒源**。
- **其余一切**：屏蔽，并 `synchronize_irq`（Topic 06 晦涩点 ⑬）确保它的 handler 不在
  半途。

### 晦涩点 ④：`synchronize_irq` 在这里为什么关键？

屏蔽一条线只是"以后不再触发"，但**此刻可能正有个 handler 在另一个 CPU 上跑**。
如果不等它跑完就继续挂起，可能在 handler 访问设备的过程中把设备断电 → 崩溃。
所以屏蔽后 `synchronize_irq` 等待"清场"——这正是 Topic 06 那个"不关闸只等清场"的
工具在挂起场景的应用。

### 晦涩点 ⑤：标志的微妙互动（共享线的坑）

原文 §3 这段是全章最 subtle 的：

- **`IRQF_NO_SUSPEND` 保持的是"线"**，但在**共享线**上它**不能阻止其他 action 的设备
  挂起**——如果一个共享者要 NO_SUSPEND 而另一个不要，就**危险**了（线还活着，
  但某个设备已经被挂起、其 handler 访问已断电的设备）。
- **`IRQF_COND_SUSPEND`** 是解药：一个**能容忍线保持使能**的共享者（它的 handler 是
  挂起安全的）设这个标志，核心就允许这种混合；**不设的话，在一条线上混用 NO_SUSPEND
  和普通 action 会被拒绝**。
- **`IRQF_EARLY_RESUME`**：让线**提前恢复**（在 resume 早期阶段，普通设备之前）——
  给那些"要靠它才能把其他设备唤回来"的中断用。
- **`IRQF_FORCE_RESUME`**：强制恢复，即使是 NO_SUSPEND 线（罕见，用于必须重新初始化的
  保活线）。

这些由 Topic 03 的逐 desc PM 计数器记账：`nr_actions`、`no_suspend_depth`、
`cond_suspend_depth`、`force_resume_depth`。

---

## 4. 逐句拆解恢复时的补送

### 晦涩点 ⑥：睡眠中触发的边沿中断不能丢

一个被屏蔽的边沿中断在**挂起期间触发**了，会丢（边沿是瞬间事件，Topic 02/05）。
恢复时 `resume_irq`（`pm.c:144`）重新使能这条线，**如果它处于 pending 状态，
就补送**（`check_irq_resend`，Topic 05 §6）：要么芯片的 `irq_retrigger`，要么软件补送链
（`CONFIG_HARDIRQS_SW_RESEND`，`desc->resend_node`）。`rearm_wake_irq` 重新武装唤醒线。

**这就是防止"设备在我们睡觉时中断了，醒来却完全没注意到"的机制。** 把 Topic 05 §6 的
resend 和本章的睡眠场景连起来——resend 不只服务 disable/enable 窗口，也服务挂起/恢复。

---

## 5. 逐句拆解控制器电源管理

### 晦涩点 ⑦：控制器自己也会"失忆"

深睡眠时，**中断控制器本身**会丢失状态（GIC 的分发器配置、IO-APIC 的重定向表
都在断电域里）。如果不管，恢复后**整个中断路由就是垃圾**——所有中断送错地方。

所以 `irq_chip` 提供 `irq_suspend`/`irq_resume`/`irq_pm_shutdown` 钩子（Topic 03 §4）+
`IRQCHIP_MASK_ON_SUSPEND`。控制器驱动在挂起时**保存**自己的分发/路由/优先级配置，
恢复时**还原**——GIC PM（`drivers/irqchip/irq-gic-pm.c`）保存分发器和再分发器配置，
IO-APIC 保存重定向表。

**画面**：控制器像个会断电失忆的电话总机。睡前**把接线图抄在纸上**（save），
醒来**照着重新接好**（restore）。少了这步，醒来后所有电话都串线。这是
"设备只在挂起一次后才坏"这类怪 bug 的根源（原文 Pitfalls）。

---

## 6. 动手任务（针对你的 x86_64 机器，多数需 root + 可挂起的环境）

### 任务 0：看你机器的唤醒源（10 分钟，无需 root 读）

```bash
cat /sys/power/wakeup_count
sudo cat /sys/kernel/debug/wakeup_sources 2>/dev/null | head -20
# 哪些设备被允许唤醒系统：
for f in /sys/devices/**/power/wakeup; do
  v=$(cat "$f" 2>/dev/null); [ "$v" = "enabled" ] && echo "$f"
done 2>/dev/null | head
# 更简单：
grep enabled /sys/devices/*/*/power/wakeup 2>/dev/null | head
cat /proc/acpi/wakeup 2>/dev/null      # ACPI 唤醒设备列表（老接口）
```

回答（答案 §7.0）：

1. 哪些设备被允许唤醒系统？电源键、RTC、网卡、USB 在其中吗？
2. `wakeup_sources` 里 `wakeup_count` 最高的是谁？它唤醒过系统几次？
3. `/proc/acpi/wakeup` 里哪些设备是 `*enabled`？（S3/S4 唤醒设备。）

### 任务 1：实测一次挂起/恢复，找出唤醒源（25 分钟，需要 root + 能挂起）⭐

> ⚠️ 笔记本/能 S2idle 的机器做。台式机/某些虚拟机可能不支持，跳到任务 2。

```bash
# 记录挂起前的中断计数：
cat /proc/interrupts > /tmp/irq_before.txt
cat /sys/power/wakeup_count
# 挂起（用 RTC 定时唤醒，避免叫不醒）：
sudo rtcwake -m mem -s 15        # 挂起 15 秒后 RTC 唤醒；-m freeze 用 s2idle
# 醒来后：
cat /proc/interrupts > /tmp/irq_after.txt
diff <(awk '{print $1,$NF}' /tmp/irq_before.txt) <(awk '{print $1,$NF}' /tmp/irq_after.txt)
# 找哪条中断在睡眠期间涨了（就是唤醒源）：
sudo cat /sys/kernel/debug/wakeup_sources | sort -k3 -rn | head
```

回答（答案 §7.1）：

1. 对比前后 `/proc/interrupts`，哪条中断（RTC）的计数涨了？它就是这次的唤醒源吗？
2. dmesg 里 `PM: suspend`/`PM: resume` 之间有多少时间？有没有 "irq N while
   suspended" 类告警？
3. `/sys/kernel/debug/irq/irqs/<RTC中断号>` 里能看到 `IRQD_WAKEUP_ARMED` 吗？

### 任务 2：源码走读逐线挂起决策（30 分钟）

```
① kernel/irq/pm.c:65     suspend_device_irq：找到三个分支——NO_SUSPEND 保持、
                         WAKEUP_ARMED 保持、其余屏蔽
② kernel/irq/pm.c:126    suspend_device_irqs：找到屏蔽后 synchronize_irq 的调用
③ kernel/irq/pm.c:144    resume_irq：找到 check_irq_resend（补送）的调用
④ kernel/irq/manage.c:868 irq_set_irq_wake：看 wake_depth 计数 + 调 chip irq_set_wake
⑤ include/linux/irq.h:235 IRQD_WAKEUP_STATE 和 IRQD_WAKEUP_ARMED 两个状态位
```

回答（答案 §7.2）：

1. `suspend_device_irq` 返回的 `sync` 布尔决定了什么？什么情况下需要 synchronize？
2. `IRQD_WAKEUP_STATE` 和 `IRQD_WAKEUP_ARMED` 有什么区别？（一个是"配置成可唤醒"，
   一个是"此刻武装中"——分别什么时候设置？）
3. `irq_set_irq_wake` 里如果芯片设了 `IRQCHIP_SKIP_SET_WAKE`，会跳过什么？

### 任务 3：观察 NO_SUSPEND 定时器 vs 唤醒源的区别（20 分钟，需要 root）

```bash
# 定时器中断带 IRQF_NO_SUSPEND/IRQF_TIMER：
grep -iE "timer|LOC|hpet|rtc" /proc/interrupts
# 看哪些中断标了可唤醒（debugfs）：
for n in $(ls /sys/kernel/debug/irq/irqs/ 2>/dev/null); do
  sudo grep -l WAKEUP /sys/kernel/debug/irq/irqs/$n 2>/dev/null
done 2>/dev/null
# 或直接：
sudo grep -rl "IRQD_WAKEUP" /sys/kernel/debug/irq/irqs/ 2>/dev/null | head
```

回答（答案 §7.3）：

1. 定时器中断（LOC/HPET）有 `IRQD_WAKEUP_STATE` 吗？它靠什么穿过挂起（§1.2 哪个开关）？
2. RTC 或电源键中断有 `IRQD_WAKEUP_STATE` 吗？它们靠哪个开关？
3. 用 §1.2 的表说出：定时器和 RTC 唤醒源，分别设的是"保持运行"还是"唤醒系统"？

### 任务 4：思考题（答案 §7.4）

1. `IRQF_NO_SUSPEND` 和 `enable_irq_wake` 为什么是两个不同的机制而不能合一？
   举一个"需要 NO_SUSPEND 但不该唤醒系统"的例子，和一个反过来的例子。
2. 共享线上一个 action 要 NO_SUSPEND、另一个不要，为什么危险？`IRQF_COND_SUSPEND`
   凭什么能让这种混合变安全？（提示：handler 的"挂起安全性"。）
3. 恢复时为什么边沿中断需要 resend 而电平中断通常不需要？（回忆 Topic 02/05：
   电平线在挂起期间状态还在，边沿是瞬间事件。）
4. 一个自定义 irqchip 驱动忘了实现 `irq_suspend`/`irq_resume`。它的设备在正常运行时
   完全正常，但"挂起一次后就不工作了"。为什么？怎么诊断到是这个原因？

---

## 7. 任务参考答案

### 7.0 任务 0

1. 通常电源键、RTC、笔记本翻盖、部分 USB、网卡（WoL）被允许唤醒；具体看机型和
   BIOS/系统设置。
2. 常见是定时器/RTC 或电源管理相关源；`wakeup_count` 是它触发唤醒的累计次数。
3. `/proc/acpi/wakeup` 里带 `*enabled` 的设备（如 PWRB 电源键、RTC、PCI 网卡槽）
   能从 S3/S4 唤醒。

### 7.1 任务 1

1. RTC 对应的中断计数会涨；它就是 rtcwake 设的唤醒源。
2. dmesg 显示 suspend 到 resume 约 15 秒（你设的）；正常不应有 "while suspended"
   告警（有的话说明某条线没被正确屏蔽）。
3. 能——RTC 作为本次唤醒源在挂起期间处于 `IRQD_WAKEUP_ARMED`（醒来后可能已清除，
   但 `IRQD_WAKEUP_STATE` 配置位仍在）。

### 7.2 任务 2

1. `sync` 决定屏蔽这条线后是否要 `synchronize_irq` 等待在跑的 handler——
   对有 action 且被真正屏蔽的线需要（确保挂起设备时 handler 不在访问它）。
2. `IRQD_WAKEUP_STATE` = "这条线被配置成可唤醒"（`enable_irq_wake` 时设，长期保持）；
   `IRQD_WAKEUP_ARMED` = "此刻在挂起中、作为唤醒源带电武装"（挂起时设，恢复时清）。
   前者是能力，后者是当前状态。
3. 跳过调用芯片的 `irq_set_wake`——这类芯片不需要逐线配置唤醒（整个控制器或更上层
   统一处理唤醒路由），只需在 genirq 层记录 WAKEUP_STATE。

### 7.3 任务 3

1. 定时器中断通常**没有** `IRQD_WAKEUP_STATE`（它不是唤醒源）；它靠 **`IRQF_NO_SUSPEND`**
   （"保持运行"开关）穿过挂起。
2. RTC/电源键**有** `IRQD_WAKEUP_STATE`；它们靠 **`enable_irq_wake`**（"唤醒系统"开关）。
3. 定时器 = 保持运行（NO_SUSPEND）；RTC 唤醒源 = 唤醒系统（enable_irq_wake）。
   完美对应 §1.2 的两个开关。

### 7.4 任务 4

1. 两件事时机和目的都不同。NO_SUSPEND 是"挂起**流程进行中**这条线还要工作"
   （线还没真正断电睡死），enable_irq_wake 是"机器**已经睡死**后还能把它叫醒"
   （接到常开唤醒逻辑）。例：① 需要 NO_SUSPEND 但不该唤醒——挂起路径里维持计时的
   定时器（它要在挂起过程中走，但不该把刚睡下的机器立刻叫醒）；② 需要唤醒但不需要
   NO_SUSPEND——电源键（挂起流程中它不需要工作，但睡死后按下要叫醒机器）。
2. 危险在于：线保持使能（因 NO_SUSPEND），但不要 NO_SUSPEND 的那个共享者的设备
   已经被挂起/断电，如果此时线触发、核心遍历 action 链调到那个设备的 handler，
   它会访问已断电的设备 → 崩溃。`COND_SUSPEND` 表示"我的 handler 是挂起安全的
   （即使设备挂起了也能安全运行/会自检跳过）"，核心据此允许混合。
3. 电平线在挂起期间**状态持续存在**（线还拉着），恢复后线仍是高的会再次触发，
   不丢；边沿是**瞬间跳变**，挂起期间被屏蔽时跳变发生过就没了，必须靠 resend 补送。
4. 控制器在深睡眠中丢失路由配置（分发器/重定向表），没有 resume 钩子就不会还原，
   恢复后中断路由全乱 → 设备的中断送不到 CPU → 设备"死了"。但正常运行（没经历
   挂起）时控制器配置一直在，所以一切正常。诊断：现象是"suspend 一次后才坏"，
   检查该 irqchip 是否实现了 irq_suspend/irq_resume，对比 GIC PM 等正确实现。

---

## 8. 学完自测

1. 挂起时哪些中断保持活着、哪些被屏蔽？这条线怎么划？
2. "保持运行"（NO_SUSPEND）和"唤醒系统"（enable_irq_wake）的区别？各举一例。
3. 共享线混用 NO_SUSPEND 和普通 action 为什么危险？COND_SUSPEND 怎么解决？
4. 恢复时为什么要 resend？边沿和电平的处理为什么不同？
5. 控制器为什么需要 suspend/resume 钩子？少了会出什么"只在挂起后出现"的 bug？

下一章 Topic 16：实战调试参考——所有 procfs/sysfs/debugfs/tracepoint 观测面 +
症状到诊断的排查手册。
