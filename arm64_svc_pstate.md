# ARM64 PSTATE：硬件、SVC/IRQ 路径、QEMU 内部表示

本文围绕 **PSTATE** 这一根主线讲清楚：
1. PSTATE 是什么、各位干嘛
2. SVC（同步异常）和 IRQ（异步异常）进入时硬件怎么处理 PSTATE
3. `eret` 怎么逆向恢复
4. Linux `kernel_entry/exit` 软件接力做什么
5. QEMU 把 PSTATE 拆成几个字段存的内部表示（`pstate_read/pstate_write`）

---

## 0. PSTATE 是什么

ARMv8 的 PSTATE = "Processor State"，是一组**散落在硬件里的处理器状态位**的统称。它**不是**一个真正的寄存器——CPU 内部 NZCV、DAIF、M、SP、SS、IL 等是物理上独立的标志/字段。"PSTATE" 这个名字只在两个场合作为"打包整体"出现：

1. 异常入口时 **`SPSR_ELx ← PSTATE`**（保存）
2. `eret` 时 **`PSTATE ← SPSR_ELx`**（恢复）
3. `mrs x?, NZCV` / `msr DAIFSet, #1` 这类局部访问指令也按字段名读写

平时软件想"读 PSTATE"，只能用 `mrs x?, NZCV/DAIF/CurrentEL/SPSel/...` 分别取。SPSR 是 PSTATE 唯一被"装在一个 64 位寄存器里"的形态。

### PSTATE 主要字段（AArch64）

| 位 | 字段 | 含义 |
|---|---|---|
| 31..28 | NZCV | 算术标志（Negative/Zero/Carry/oVerflow） |
| 25 | TCO | MTE 标签检查覆盖 |
| 24 | DIT | 数据无关时间（侧信道缓解） |
| 23 | UAO | 用户访问覆盖（让 ldtr/sttr 等同 ldr/str） |
| 22 | PAN | 特权访问禁止（内核不能直接访问用户态地址） |
| 21 | SS | 单步调试黏性位 |
| 20 | IL | Illegal Execution State 黏性位 |
| 13 | ALLINT | NMI mask |
| 12 | SSBS | 投机存储绕过状态 |
| 11..10 | BType | 分支目标类型（BTI） |
| 9..6 | DAIF | 异常 mask：D=Debug, A=SError, I=IRQ, F=FIQ |
| 4 | nRW | 0=AArch64, 1=AArch32 |
| 3..0 | M | mode/EL+SP：EL0t=0, EL1t=4, **EL1h=5**, EL2t/h=8/9, EL3t/h=12/13 |

**`M` 字段的特殊重要性**：低 2 位编码 EL（0/1/2/3），bit0 同时是 SP 选择（0=SP_EL0=`t`hread, 1=SP_ELN=`h`igh）。所以 `EL1h = 0b0101` 就是"EL1 + SP_EL1"。

---

## 1. SVC：硬件自动完成的切换动作

场景：用户态 EL0 执行 `svc #0` → 进入 EL1。

`svc` 是**同步异常**，硬件按 *Exception Entry* 流程一次性做完下面 8 件事，**没有任何软件介入窗口**：

| 步骤 | 寄存器 | 内容 |
|---|---|---|
| 1 | `ELR_EL1` ← 下一条指令地址 | `svc` 之后那条指令的 PC（不是 svc 本身） |
| 2 | `SPSR_EL1` ← `PSTATE` | **整个 PSTATE 打包存进 SPSR**：NZCV/DAIF/M/SS/IL/nRW/PAN/UAO/... 全在里面。SP 和 PC 不在 |
| 3 | `ESR_EL1` ← syndrome | EC=0x15 (`SVC from AArch64`)，IL=1，ISS=svc 立即数 16 位 |
| 4 | `FAR_EL1` | SVC 不用，是 UNKNOWN |
| 5 | `PSTATE` ← 新值 | **M=EL1h** (0b0101)，**DAIF=0b1111** 全 mask，**SS=0**，**IL=0**，nRW=AArch64；NZCV/PAN/UAO 等不变 |
| 6 | SP 切换 | 因为 M 设成 EL1h，SP 自动从 SP_EL0 切到 SP_EL1 |
| 7 | `PC` ← 异常向量 | `VBAR_EL1 + offset`（见下） |
| 8 | TLB / 内存系统调整 | 切换 TTBR 选择规则、清 exclusive monitor 等 |

**PSTATE 视角下的关键变化**：

- 步骤 2 拍下 PSTATE 快照（snapshot）进 SPSR。
- 步骤 5 把活跃 PSTATE **不是全部改写**，而是只动几位：
  - `M ← EL1h`（提权到内核，自动改 SP 选择）
  - `DAIF ← 0b1111`（屏蔽所有异步事件，避免 handler 被打断）
  - `SS ← 0` / `IL ← 0`（清两个"黏性异常源"，否则 handler 第一条指令立刻又触发异常 → 无限递归）
  - `nRW ← 0`（AArch64）
  - 其余位（NZCV、PAN、UAO、BType、PAC keys 等）保持原样

> **不变量**：handler 进来后 `mrs spsr_el1` 读到的是"被打断瞬间的完整 PSTATE"；handler 自己跑的时候 PSTATE 是"内核态默认值"。两者由硬件原子切换，软件看不到中间态。

### 异常向量怎么选 (VBAR_EL1 + offset)

`offset` 4×4 = 16 项表，单 EL 内 4 项（每项 0x80 字节，可容纳 32 条指令）：

```
Curr EL / SP0   :  +0x000  Sync   +0x080  IRQ   +0x100  FIQ   +0x180  SError
Curr EL / SPx   :  +0x200  Sync   +0x280  IRQ   +0x300  FIQ   +0x380  SError
Lower EL / 64   :  +0x400  Sync   +0x480  IRQ   +0x500  FIQ   +0x580  SError
Lower EL / 32   :  +0x600  Sync   +0x680  IRQ   +0x700  FIQ   +0x780  SError
```

EL0 → EL1 的 AArch64 SVC 走 **`VBAR_EL1 + 0x400`**（Lower EL / 64bit Sync）。

---

## 2. IRQ：异步异常的同款 PSTATE 处理

场景：CPU 任意时刻收到外部中断（IRQ）。

**机制和 SVC 一模一样**，区别只在：

| 维度 | SVC（同步） | IRQ（异步） |
|---|---|---|
| 触发 | 软件 `svc #imm` | 外部信号 |
| 时机 | 指令边界确定（svc 那条） | 任何指令边界都可能 |
| ESR | EC=0x15, ISS=立即数 | EC=0（IRQ 无 syndrome），细节看 GIC |
| `ELR_EL1` 存什么 | svc 的下一条 PC | **被打断的下一条 PC**（不是被打断那条本身） |
| vector offset | Sync 那列 | IRQ 那列（+0x080/0x280/0x480/...） |

PSTATE 处理 100% 相同：SPSR_EL1 ← PSTATE 快照；活跃 PSTATE 改 M/DAIF/SS/IL/nRW。

也就是说，**进入 handler 时 PSTATE 永远 mask 全 DAIF**——这是为什么 Linux IRQ handler 默认"不可被另一个 IRQ 嵌套打断"。要嵌套必须显式 `msr DAIFClr, #2`。

---

## 3. eret：硬件原子逆操作

```asm
eret
```

硬件一次性做：

| 步骤 | 动作 |
|---|---|
| 1 | `PC ← ELR_EL1` |
| 2 | `PSTATE ← SPSR_EL1`（**整组打包还原**，包括 M / SP 选择） |
| 3 | 若 PSTATE.M 表示更低 EL，自动切换 SP 库（如 EL1h → EL0 → SP_EL0） |
| 4 | 清 exclusive monitor 等微架构状态 |

**PSTATE 视角**：

- 进入异常时拍了一张 PSTATE 快照存进 SPSR。
- 软件可以读 SPSR（`mrs x?, spsr_el1`）也可以改它（`msr spsr_el1, x?`）。
- `eret` 把改后的 SPSR 整组写回活跃 PSTATE，**原子**。

> 关键：`eret` 是 ARMv8 里**唯一**能正确切换 EL/SP 库的指令。`msr DAIFSet, #1` 只能改 DAIF，`msr SPSel, #0` 只能改 SP 选择，**没有任何 `msr` 能改 M 字段**——M 只能通过"陷入更高 EL 由硬件设" + "eret 由 SPSR 设"两条路径改变。

---

## 4. 软件接力（Linux `kernel_entry`）

硬件只保住了"切换瞬间"那 4 个系统寄存器（ELR/SPSR/ESR/FAR）。GPR、原 SP_EL0、TPIDR 这些**必须由 EL1 的 vector 入口立即抢救**，否则一旦在内核里再发生任何上下文调用就丢了。

ARM64 Linux 在 `arch/arm64/kernel/entry.S::kernel_entry` 里做这件事。简化骨架（删了 PAN/UAO/BTI/SDEI 的分支）：

```asm
SYM_CODE_START(vectors)
    ...
    kernel_ventry  0, t, 64, sync      // EL0 64-bit sync (SVC 走这里)
    ...
SYM_CODE_END(vectors)

.macro kernel_entry, el, regsize = 64
    sub  sp, sp, #PT_REGS_SIZE          // 在 SP_EL1 上腾出 struct pt_regs

    stp  x0, x1,   [sp, #16 *  0]       // ① 把所有 GPR 存进 pt_regs
    stp  x2, x3,   [sp, #16 *  1]
    ...
    stp  x28, x29, [sp, #16 * 14]

    .if \el == 0
        mrs  x21, sp_el0                // ② 原用户态 SP
    .else
        add  x21, sp, #PT_REGS_SIZE     //    EL1->EL1 异常：原 SP_EL1
    .endif

    mrs  x22, elr_el1                   // ③ 把硬件存好的 SPSR/ELR 读回来
    mrs  x23, spsr_el1                  //    存进 pt_regs.pc / .pstate
    stp  lr,  x21, [sp, #S_LR]          //    顺手把 lr + sp 也存
    stp  x22, x23, [sp, #S_PC]
.endm
```

也就是把这张表搬到内存里：

```
struct pt_regs {              ┐
    u64 regs[31];   x0..x30   │   ← kernel_entry stp 进来
    u64 sp;                   │   ← 来自 mrs sp_el0 或 EL1 上一个 SP
    u64 pc;                   │   ← 来自 mrs elr_el1   ⬅ 硬件存到 ELR_EL1 的那一拍
    u64 pstate;               │   ← 来自 mrs spsr_el1  ⬅ 硬件存到 SPSR_EL1 的那一拍
    ...                       │
};                            ┘   栈上 PT_REGS_SIZE = 304B
```

**`pt_regs.pstate` 装的是 SPSR_EL1 的原始值**——即"被打断瞬间 PSTATE 的完整快照"。内核要修改用户态 PSTATE（比如 ptrace、signal 处理），就改这个字段，`kernel_exit` 时会写回 SPSR_EL1，`eret` 自动生效。

`esr_el1` 在 SVC 路径里**不会被压进 pt_regs**，而是 vector 分派时直接 `mrs x0, esr_el1` 然后 `b el0t_64_sync_handler` 里 `read_sysreg(esr_el1)` 用一次就丢——因为 `ec=0x15` 已经决定了"这是 SVC"，不需要再保留到返回时。

`kernel_exit` 走相反路径：

```asm
.macro kernel_exit, el
    ldp  x21, x22, [sp, #S_PC]         // 取回 pc / pstate
    msr  elr_el1,  x21                 // 写回 ELR_EL1
    msr  spsr_el1, x22                 // 写回 SPSR_EL1
    ldp  ... (恢复 GPR / sp_el0) ...
    add  sp, sp, #PT_REGS_SIZE
    eret                               // 硬件原子用 ELR_EL1/SPSR_EL1 还原 PC/PSTATE，回 EL0
.endm
```

---

## 5. 为什么"硬件只挂 1 份就够 Linux 用"——嵌套靠 stack

只有 1 套 `SPSR_EL1/ELR_EL1/ESR_EL1`，理论上看到第二个异常就该覆盖。Linux 之所以能跑嵌套异常，关键是：

1. **EL0 → EL1 的同步异常进入时 DAIF 自动 mask**，所以 SVC handler 起跑前不会被另一个 IRQ 抢；
2. handler 内**软件抢救 PSTATE 进了 pt_regs**（即 SPSR_EL1 的快照值）；
3. 之后真要再发生异常（典型场景：`local_irq_enable()` 后的硬中断、或缺页），新的异常**硬件再覆盖一次** `SPSR_EL1 / ELR_EL1`，但**旧那一份快照已经在内核栈上 pt_regs 里了**，不丢；
4. 内层异常返回前，先把内层 `pt_regs.pc / .pstate` 写回 `ELR_EL1 / SPSR_EL1`，再 `eret`——外层那份 pt_regs 依然安然躺在外层栈帧里。

简单说：**SPSR_EL1/ELR_EL1 = 当前在飞的 1 帧的 PSTATE 镜像；嵌套 N 帧 = N 份 `pt_regs.pstate` 在内核栈上。** 软件接力的代价就是 vector 入口必须能在"被覆盖之前"完成那 4 行 `mrs`。

---

## 6. QEMU 内部 PSTATE 表示

QEMU 模拟 PSTATE 时**不**用一个 64 位字段——为了 TCG 性能，**把 PSTATE 拆散到 5 个独立字段**（`CPUARMState env` 内）：

| 物理位 | QEMU 字段 | 备注 |
|---|---|---|
| NZCV | `env->NF / ZF / CF / VF` | 拆成 4 个独立 `uint32_t`，TCG 改 NZCV 时不用 RMW 整个 PSTATE |
| DAIF | `env->daif` | 独立 `uint64_t`，bit 6..9 有效 |
| BType | `env->btype` | 独立 `uint32_t`，2 位有效 |
| 其余所有位 | `env->pstate` | M / nRW / SP / SS / IL / PAN / UAO / DIT / TCO / SSBS / ALLINT / EXLOCK 等 |
| —— | （SP 选择见 `env->pstate` bit0 + `aarch64_save_sp` 机制） | |

`CACHED_PSTATE_BITS = NZCV | DAIF | BType` —— 这些位被"缓存"到独立字段。

### 拼包：`pstate_read`

```c
// cpu.h
static inline uint64_t pstate_read(CPUARMState *env)
{
    int ZF;
    ZF = (env->ZF == 0);
    return (env->NF & 0x80000000) | (ZF << 30)
        | (env->CF << 29) | ((env->VF & 0x80000000) >> 3)
        | env->pstate | env->daif | (env->btype << 10);
}
```

把 5 个字段重新拼成 64 位 PSTATE 值，**等同于硬件 SPSR_EL1 的 snapshot**。

### 拆包：`pstate_write`

```c
static inline void pstate_write(CPUARMState *env, uint64_t val)
{
    env->ZF = (~val) & PSTATE_Z;
    env->NF = val;
    env->CF = (val >> 29) & 1;
    env->VF = (val << 3) & 0x80000000;
    env->daif = val & PSTATE_DAIF;
    env->btype = (val >> 10) & 3;
    env->pstate = val & ~CACHED_PSTATE_BITS;
}
```

把一个 64 位 PSTATE 值**整组**写进 5 个字段。语义对应**"硬件把 SPSR_EL1 还原成活跃 PSTATE"**——即 `eret` 内部做的事。

### 局部更新：直接写字段

如果只想动一两位，**不需要** `pstate_read + 改 + pstate_write` 这个 round-trip——直接写对应字段：

```c
env->daif |= PSTATE_DAIF;                // mask DAIF
env->pstate &= ~(PSTATE_SS | PSTATE_IL); // 清 SS/IL
```

QEMU 在所有"异常入口"路径（`take_aarch64_exception`、modecall helper、shpl_try_capture_irq）都用这种方式——因为入口动作就是固定的"mask DAIF + 清 SS/IL"，没必要走 RMW。

### SP 银行（重要细节）

ARMv8 的 SP 库（SP_EL0/SP_EL1/...）在 QEMU 里是**独立数组** `env->sp_el[4]`。"当前活跃的 SP"则放在 `env->xregs[31]`——AArch64 没有 `x31` 这个 GPR（编码 31 是 XZR/SP），QEMU 借用这个槽位存 live SP。

`pstate_write` 改 M[0]（SP 选择位）**不会**自动切换 `xregs[31]`！想正确切换必须显式调：

```c
// internals.h
static inline void aarch64_save_sp(CPUARMState *env, int el) {
    if (env->pstate & PSTATE_SP)
        env->sp_el[el] = env->xregs[31];
    else
        env->sp_el[0] = env->xregs[31];
}
static inline void aarch64_restore_sp(CPUARMState *env, int el) { ... }
```

`eret` 的 QEMU 实现走的是 `take_aarch64_exception` 的逆流程，里面**显式**调用 `aarch64_save_sp/aarch64_restore_sp` 配合 `pstate_write` 完成"PSTATE 改 + SP 切"原子操作。

> **这就是为什么"用 pstate_write 直接还原 SPSR 不安全"** —— 如果 SPSR.M 跟当前 M 不一致（涉及 SP 库变化），单独 `pstate_write` 不会同步 `xregs[31]`，必然出错。所以 PSTATE 里的 **M、nRW、SP 这三类字段必须由 eret 路径处理**，不能用 pstate_write 还原。

---

## 7. 一张总表把 PSTATE 处理路径串起来

| 操作 | ARMv8 硬件 | QEMU 实现 |
|---|---|---|
| 异常入口保存 PSTATE | `SPSR_EL1 ← PSTATE`（整组） | `env->banked_spsr[1] = pstate_read(env)` |
| 异常入口设新 PSTATE | M/DAIF/SS/IL/nRW 改值，其余不变 | 直接写 `env->daif/pstate/...` 字段；M 改变要带 `aarch64_save_sp/restore_sp` |
| handler 内读 SPSR | `mrs x?, spsr_el1` | `env->banked_spsr[1]` |
| handler 内改 SPSR | `msr spsr_el1, x?` | 写 `env->banked_spsr[1]` |
| `eret` 还原 PSTATE | `PSTATE ← SPSR_EL1`（整组 + SP 切换） | `take_aarch64_exception` 反向流程：`pstate_write` + `aarch64_restore_sp` |
| 局部读 PSTATE 位 | `mrs x?, NZCV/DAIF/CurrentEL` 等 | 直接读对应字段 |
| 局部写 PSTATE 位 | `msr DAIFSet/Clr, #imm` 等 | 直接写对应字段（不调 pstate_write） |

---

## 8. 想自己看代码

- ARM ARM (DDI 0487J)：
  - D1.10 *Synchronous exception taken to AArch64* — 8 步入口流程
  - D1.11 *Asynchronous exception (IRQ/FIQ/SError)* — IRQ 路径
  - C5.2 *PSTATE fields* — 各字段位定义
  - C6.2.107 *ERET* — eret 行为
  - G1.16 *SPSR_EL1 format*
- Linux：
  - `arch/arm64/kernel/entry.S` 里 `kernel_ventry`、`kernel_entry`、`kernel_exit`
  - `arch/arm64/kernel/entry-common.c::el0t_64_sync_handler` → `el0_svc` → `do_el0_svc`
  - `arch/arm64/include/asm/ptrace.h` 看 `struct pt_regs` 布局
- QEMU：
  - `target/arm/cpu.h::pstate_read / pstate_write` —— PSTATE 拼/拆
  - `target/arm/internals.h::aarch64_save_sp / restore_sp` —— SP 银行
  - `target/arm/helper.c::take_aarch64_exception` —— 异常入口的完整 QEMU 实现
  - `target/arm/tcg/helper-a64.c::HELPER(exception_return)` —— eret 实现
