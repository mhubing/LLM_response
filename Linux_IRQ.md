# ARM64 Linux 中断处理笔记

记录与 SHPL 中断委派相关的两块 ARM64 Linux 知识：
1. **硬中断完整调用链**——从 `el1h_64_irq` 入口到驱动 handler
2. **pseudo-NMI 机制**——GICv3 优先级掩码，唯一能"嵌套"硬中断的途径

> 全部对照 `linux-aarch64/` 当前源码核过（基线是仓库 HEAD）。代码行号引自 6.x 主线，重构后可能漂移，以函数名为准。

---

## 1. 硬中断完整调用链

### 1.1 调用流程图
linux 6.19

```
[硬件] IRQ 触发，跳到 VBAR_EL1 + 0x280
        │
        ▼
entry.S  el1h_64_irq 标号
   ├─ kernel_entry 1                       (保存现场到 pt_regs)
   └─ bl  el1h_64_irq_handler              (跳到 C)
        │
        ▼
entry-common.c:513  el1h_64_irq_handler(regs)
   └─ el1_interrupt(regs, handle_arch_irq)            [行 502]
        ├─ write_sysreg(DAIF_PROCCTX_NOIRQ, daif);    (确认关 IRQ)
        └─ __el1_irq(regs, handler)                   [行 489]
              ├─ enter_from_kernel_mode(regs)         (RCU/上下文记账)
              ├─ irq_enter_rcu()                      ← 进入 hard IRQ 上下文
              ├─ do_interrupt_handler(regs, handler)  [行 129]
              │     ├─ on_thread_stack() ? call_on_irq_stack : 直接调
              │     └─ handler(regs)                  (handler == handle_arch_irq)
              │           │
              │           ▼
              │   handle_arch_irq 是全局函数指针 [arch/arm64/kernel/irq.c:87]
              │   __ro_after_init，启动时被 set_handle_irq() 装上
              │   GICv3 在 probe 时调:
              │     set_handle_irq(gic_handle_irq);  [drivers/irqchip/irq-gic-v3.c:2037]
              │           │
              │           ▼
              │   gic_handle_irq(regs)               [irq-gic-v3.c:915]
              │     └─ __gic_handle_irq_from_irqson(regs) 或 _irqsoff(regs)
              │           ├─ irqnr = gic_read_iar();
              │           └─ __gic_handle_irq(irqnr, regs)         [行 818]
              │                ├─ gic_complete_ack(irqnr);
              │                └─ generic_handle_domain_irq(domain, irqnr)
              │                      │
              │                      ▼
              │              kernel/irq/irqdesc.c:723
              │                generic_handle_domain_irq()
              │                  └─ handle_irq_desc(desc)          [行 658]
              │                        └─ generic_handle_irq_desc(desc)
              │                              └─ desc->handle_irq(desc)   ← ★ 虚函数
              │                                    │
              │                                    ▼
              │                       per-IRQ flow handler，
              │                       GICv3 在 domain_map 时装的:
              │                         · handle_percpu_devid_irq  (PPI/SGI)
              │                         · handle_fasteoi_irq       (SPI)
              │                                    │
              │                                    ▼
              │                       handle_fasteoi_irq()  [kernel/irq/chip.c:736]
              │                          └─ handle_irq_event(desc)
              │                                └─ handle_irq_event_percpu
              │                                      └─ __handle_irq_event_percpu
              │                                            └─ 遍历 desc->action[]，
              │                                               逐个调 action->handler(...)
              │                                               (driver request_irq 装的回调)
              ├─ irq_exit_rcu()                     ← 退出 hard IRQ，可能触发 softirq
              └─ exit_to_kernel_mode(regs, state)   ← 内含 arm64_preempt_schedule_irq()
        │
        ▼
回到 entry.S，ret_to_kernel → ERET
```

### 1.2 三层 handler

| 层级 | 字段 | 谁装 | 默认值 / 典型值 |
|------|------|------|---------|
| 架构根 handler | `handle_arch_irq`（全局函数指针）| `set_handle_irq()`，GICv3 driver 在 `gic_init_bases()` 调 | `gic_handle_irq` |
| per-IRQ flow handler | `desc->handle_irq` | chip driver 在 `irq_domain_set_info()` 时装 | PPI/SGI → `handle_percpu_devid_irq`；SPI → `handle_fasteoi_irq` |
| driver 回调 | `desc->action->handler` | `request_irq()` / `request_percpu_irq()` | 由各 driver 提供（arch_timer / pl011 / virtio 等）|

### 1.3 没人 `request_irq` 的 IRQ 怎么处理

`handle_fasteoi_irq` 进来后：
```c
if (!irq_can_handle_actions(desc)) {     // desc->action == NULL
    mask_irq(desc);                       // 把这根线 mask 掉
    cond_eoi_irq(chip, &desc->irq_data);  // 告诉 GIC 处理完了
    return;
}
```
后果：该 IRQ 被永久 mask + GIC 收到 EOI，不会刷屏崩内核，但这一根线相当于"哑掉"。

### 1.4 `irq_exit_rcu()` 与 softirq

`__el1_irq` 调的是 `irq_exit_rcu()`（不是 `irq_exit()`），两者最终都进 `__irq_exit_rcu()`：

```c
// kernel/softirq.c:713
static inline void __irq_exit_rcu(void)
{
    local_irq_disable();                 // 先关 IRQ
    account_hardirq_exit(current);
    preempt_count_sub(HARDIRQ_OFFSET);   // 退出硬中断上下文
    if (!in_interrupt() && local_softirq_pending())
        invoke_softirq();                 // ← 立即调用，同栈
    ...
}
```

`invoke_softirq()` → `__do_softirq()` → `handle_softirqs(false)`，里面 `restart:` 块会 `local_irq_enable()` 跑 softirq handlers，结束再 `local_irq_disable()`。

**含义**：硬 IRQ handler 全程 `PSTATE.I=1`；softirq 阶段 `PSTATE.I=0`，但已经不在硬中断上下文里（`in_interrupt()` 返回 true 阻止 softirq 自身嵌套）。

例外：`force_irqthreads()` 启用时，`invoke_softirq()` 改为 `wakeup_softirqd()`，把 softirq 推给 ksoftirqd 内核线程跑。

---

## 2. pseudo-NMI 机制

ARM64 没有真正的 NMI 异常向量，Linux 用 GICv3 的优先级掩码寄存器 `ICC_PMR_EL1` 实现"伪 NMI"——**这是唯一能打断正在跑的硬 IRQ handler 的机制**。

### 2.1 工作原理

| 状态 | 关 IRQ 的实现 |
|------|------------|
| 普通模式（默认） | `PSTATE.I = 1` |
| pseudo-NMI 模式 | `PSTATE.I = 0`，但 `ICC_PMR_EL1` 设到只放高优先级中断进 CPU |

`request_nmi()` 注册的中断在 GIC 被配成"高于 PMR 阈值"的优先级，因此即使在普通 IRQ handler 里也能进 CPU。典型用户：perf PMU 中断、kgdb、SError 风格的硬错处理。

### 2.2 启用条件（缺一不可）

| 条件 | 检查办法 |
|------|---------|
| (a) 编译时 `CONFIG_ARM64_PSEUDO_NMI=y` | `grep PSEUDO_NMI build-aarch64/.config` |
| (b) 启动参数 `irqchip.gicv3_pseudo_nmi=1` | `cat /proc/cmdline` |
| (c) GICv3 CPU 接口可用 | `dmesg \| grep GICv3` |
| (d) 启动时未被黑名单 quirk 关掉（MediaTek 固件 bug 等）| `dmesg \| grep -i "Pseudo-NMI"` |

四个都满足时，`ARM64_HAS_GIC_PRIO_MASKING` cap 才会被注册：

```c
// arch/arm64/kernel/cpufeature.c:2848
#ifdef CONFIG_ARM64_PSEUDO_NMI
{
    .desc = "IRQ priority masking",
    .capability = ARM64_HAS_GIC_PRIO_MASKING,
    .type = ARM64_CPUCAP_STRICT_BOOT_CPU_FEATURE,
    .matches = can_use_gic_priorities,
},
#endif
```

`can_use_gic_priorities()`（cpufeature.c:2271）三件套：`enable_pseudo_nmi` && GICv3 CPUIF 存在 && 不是坏固件。

