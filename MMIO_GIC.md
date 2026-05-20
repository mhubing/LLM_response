# MMIO 与内核对 GIC 的操作

## 一、MMIO 是什么?最基础的概念

### 1. CPU 和外设之间的"对话"问题

CPU 需要和各种外设打交道:键盘、网卡、显卡、定时器、磁盘控制器、中断控制器…… 但 CPU 只会做一件事 —— **读写内存**(执行 `load` / `store` 指令)。它根本不知道什么叫"键盘"。

那 CPU 怎么"告诉网卡发个包"或者"问中断控制器现在有几个中断挂起"?

历史上有两种方案:

#### 方案 A:Port I/O(PIO)—— x86 的老传统
CPU 提供专用指令 `in` / `out`,用一组"端口号"和外设通信。

```
in  al, 0x60   ; 从键盘端口读一个字节
out 0x60, al   ; 写一个字节到键盘端口
```

- 端口号空间和内存空间**完全分开**。
- 只有 x86 有,且现在用得少了。

#### 方案 B:Memory-Mapped I/O(MMIO)—— 现代主流
**把外设的寄存器"映射"到物理内存地址空间的某段范围里**,CPU 用普通的 `load` / `store` 指令就能访问。

```
ldr x0, [0xFE100000]   ; 从 0xFE100000 这个"地址"读一个 word
                       ; 实际上是读 GIC 中断控制器的某个寄存器
str x1, [0xFE100004]   ; 往 0xFE100004 写,实际上是配置 GIC
```

- CPU **以为**自己在读写内存,但 **0xFE100000 这个物理地址不指向 DRAM**,而是被硬件路由到了 GIC 设备。
- 写这个地址 = 给设备发命令;读这个地址 = 查询设备状态。
- ARM 几乎所有外设都用 MMIO。

### 2. 一张图直观理解物理地址空间

```
0x00000000 ┌──────────────────┐
           │                  │
           │     DRAM         │   ← 真正的内存(读写就是访存)
           │                  │
0x80000000 ├──────────────────┤
           │   设备 MMIO 区    │
           │   ─ GIC          │   ← 中断控制器寄存器
           │   ─ UART         │   ← 串口寄存器
           │   ─ Timer        │   ← 定时器寄存器
           │   ─ NIC          │   ← 网卡寄存器
           │   ─ SMMU         │   ← IOMMU 寄存器
0xFFFFFFFF └──────────────────┘
```

CPU 看到的是一整片"物理地址空间",DRAM 占一段,各种设备 MMIO 各占一段。**读不同地址,硬件电路把请求路由到不同地方**。

### 3. MMIO 寄存器和内存的关键区别

虽然访问指令一样,但 MMIO 寄存器和 DRAM 行为完全不同:

| | DRAM | MMIO 寄存器 |
|---|---|---|
| 读两次同一地址 | 拿到一样的值 | 可能拿到不同的值(比如"取出一个挂起的中断") |
| 写一个值 | 之后能读出来 | 可能根本读不出来,但**触发了某个动作**(比如发包) |
| 是否能缓存? | 可以 | 不可以(必须 uncached 访问) |
| 行为本质 | 存储 | **通信** |

> 关键认知:**MMIO 表面上是"内存",本质上是"硬件接口"**。每次 load/store 不仅仅是搬数据,而是和设备"对话"。

### 4. 软件怎么用 MMIO?

操作系统启动时会通过 **设备树(Device Tree)** 或 **ACPI 表** 知道每个外设的 MMIO 物理地址范围;然后在内核虚拟地址空间里建立一段映射(通常用 `ioremap()` 把物理 MMIO 地址映射成内核虚拟地址,**且标成 uncached**),从此驱动就可以用普通的指针读写来操作设备。

Linux 中典型用法:
```c
void __iomem *gic_base = ioremap(0xFE100000, 0x10000);  // 映射 GIC 的 MMIO
writel(value, gic_base + GICD_SGIR);                    // 写 GIC 的某个寄存器
u32 v = readl(gic_base + GICC_IAR);                     // 读 GIC 的某个寄存器
```

`writel` / `readl` 不只是普通的 store/load —— 它们插入内存屏障、绕过缓存,保证写真的被设备看到、读真的去拿设备的当前状态。

---

## 二、内核通过 MMIO 对 GIC 做什么操作?

### 1. 先认识 GIC

**GIC(Generic Interrupt Controller)** 是 ARM 平台上**统一管理所有中断**的硬件模块。它接收来自所有外设的中断信号,按优先级排队,然后投递给某个 CPU 核去处理。

GIC 的内部结构(以 GICv2/v3 为例)分成两大部分:

| 组件 | 名字 | 功能 | MMIO 地址段(示例) |
|---|---|---|---|
| **Distributor** | GICD | 全局视角:管所有中断的使能、优先级、目标 CPU | `0xFE100000`(每机器一份) |
| **CPU Interface** | GICC | per-CPU 视角:某个 CPU 怎么 ACK / 处理中断 | `0xFE110000`(每 CPU 一份)|
| **Redistributor**(GICv3 新增) | GICR | per-CPU 配置 SGI / PPI 等 | per-CPU 一份 |

两部分都是 **MMIO 寄存器组**,内核就是通过对这些 MMIO 地址的读写来"指挥"GIC 工作。

### 2. 内核开机时:初始化 GIC

启动早期,内核(或 KVM 等驱动)读 device tree 找到 GIC 的 MMIO 基址,然后做下面这些**写操作**:

| 操作 | 写哪个寄存器 | 含义 |
|---|---|---|
| 关闭整个 GIC,先清场 | `GICD_CTLR ← 0` | 暂时禁用分发器,准备配置 |
| 关闭每个中断 | `GICD_ICENABLERn ← 0xFFFFFFFF` | 把所有中断默认禁用 |
| 清掉所有挂起中断 | `GICD_ICPENDRn ← 0xFFFFFFFF` | 防止启动残留 |
| 设置每个中断的优先级 | `GICD_IPRIORITYRn ← 0xA0...` | 默认中等优先级 |
| 设置每个中断的目标 CPU | `GICD_ITARGETSRn` | 决定中断送给哪个核 |
| 配置中断触发方式 | `GICD_ICFGRn` | 边沿触发 / 电平触发 |
| 启用分发器 | `GICD_CTLR ← 1` | 开启 GIC |
| per-CPU 启用 CPU Interface | `GICC_CTLR ← 1` | 让当前 CPU 接收中断 |
| 设置 CPU 优先级掩码 | `GICC_PMR ← 0xFF` | 接受所有优先级 |

### 3. 运行时:启用一个具体设备的中断(例如网卡)

网卡驱动调用 `request_irq()` 注册中断处理函数,在底层最终触发:

```
写 GICD_ISENABLERn 的某个 bit
  → GIC 开始让该中断信号"传得过去"
```

如果想换 CPU 处理这个中断(SMP affinity):
```
写 GICD_ITARGETSRn 的对应字段(GICv2)
或 GICD_IROUTERn(GICv3)
  → 改变中断的目标 CPU
```

### 4. 中断真的来了:CPU 怎么处理?

中断到达 CPU,CPU 跳到异常向量,**通用流程**(从中断处理函数里):

#### Step 1:读 IAR(Interrupt Acknowledge Register)拿到中断号
```c
u32 irqstat = readl(gic_cpu_base + GICC_IAR);
u32 irqnr = irqstat & 0x3FF;
```
- 这是个**读操作**,但语义不是"看一眼" —— 它**告诉 GIC**:"我接住了这个中断,把它从挂起队列移到激活队列"。
- 这就是前面说的"MMIO 不是普通内存":同一个寄存器读两次,拿到的值不一样(第二次会拿到下一个中断,或拿到 1023 表示无中断)。

#### Step 2:调用具体中断处理程序
内核根据 `irqnr` 找到对应设备的处理函数,运行(比如网卡处理收包、定时器更新 jiffies)。

#### Step 3:写 EOIR(End Of Interrupt Register)告诉 GIC "处理完了"
```c
writel(irqstat, gic_cpu_base + GICC_EOIR);
```
- 一个**写操作**,告诉 GIC 这个中断处理结束,从激活队列移除,允许后续同一中断再次触发。

> 一次中断处理 = **2 次 MMIO 操作**(读 IAR + 写 EOIR)。完整的内核中断流程,从硬件接收到处理完成,全程都是通过 MMIO 与 GIC 通信。

### 5. 给别的 CPU 发 IPI(进程间中断)

多核系统里,某个 CPU 想通知另一个 CPU 做事(比如调度、TLB shootdown),通过 SGI(Software Generated Interrupt):

```c
writel(target_cpu_mask << 16 | irq_id, gic_dist_base + GICD_SGIR);
```

写一次 `GICD_SGIR`,GIC 就触发目标 CPU 的中断。

### 6. 屏蔽 / 取消屏蔽中断

```
写 GICD_ICENABLERn ← bit_n   → 屏蔽中断 n
写 GICD_ISENABLERn ← bit_n   → 取消屏蔽中断 n
```

### 7. 整体一张表:内核对 GIC 的常见 MMIO 操作

| 场景 | 操作 | 寄存器 |
|---|---|---|
| 启动初始化 | 关 / 开 GIC | `GICD_CTLR`, `GICC_CTLR` |
| 启动初始化 | 清挂起、设优先级、设触发方式 | `GICD_ICPENDRn`, `GICD_IPRIORITYRn`, `GICD_ICFGRn` |
| 注册设备中断 | 启用某个中断号 | `GICD_ISENABLERn` |
| 改 IRQ affinity | 设置目标 CPU | `GICD_ITARGETSRn` (v2) / `GICD_IROUTERn` (v3) |
| 中断到达 | 取中断号 | `GICC_IAR` (读) |
| 中断处理完 | 通知 GIC 结束 | `GICC_EOIR` (写) |
| 发 IPI | 触发软件中断 | `GICD_SGIR` |
| 屏蔽中断 | 禁用某个中断号 | `GICD_ICENABLERn` |
| 优先级阈值 | 设当前 CPU 接受的优先级范围 | `GICC_PMR` |

---

## 三、一句话总结

> **MMIO(Memory-Mapped I/O)是把外设的硬件寄存器映射到物理地址空间,让 CPU 用普通的 load/store 指令来"伪装成读写内存"地访问设备。它表面像内存,本质是硬件通信接口 —— 读写的副作用不是搬数据,而是触发设备动作或获取设备状态。**
>
> **内核操作 GIC 完全通过 MMIO 完成:启动时配置分发器和 CPU 接口、运行时启用/屏蔽中断、中断到达时通过 IAR 取中断号、处理完通过 EOIR 通知 GIC、多核间通过 SGIR 发 IPI。整个 ARM 中断管理流程,本质上就是一连串对 GIC MMIO 寄存器的读写。**
