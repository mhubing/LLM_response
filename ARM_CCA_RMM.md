# CCA REALM 中的 RMM

让我把 ARM CCA 里**到底有几个 hypervisor、各自是什么**讲清楚。

---

## 一、关键澄清:Realm world 里**没有传统意义上的 "hypervisor"**

你说"Realm 中运行的 hypervisor 完全不是 KVM"——**这里其实有个概念混淆**:

> **Realm world 里跑的 RMM,严格来说不是一个 hypervisor。**

### RMM 到底是什么?

- RMM(**Realm Management Monitor**)跑在 Realm world 的 EL2,看似占据了"hypervisor 的位置"。
- 但它**不调度 VM、不分配内存、不做 I/O 虚拟化、不模拟设备** —— 这些"hypervisor 的传统工作"它都不做。
- 它只做一件事:**为 Realm VM 强制访问控制**,确保 Normal world 的 KVM 看不到 Realm VM 的数据。
- 真正"调度 Realm VM 在哪个物理 CPU 上跑、给它多少内存、什么时候让它跑"这些**调度/资源管理决策,还是由 Normal world 的 KVM 通过 RMI 调用来"请求"RMM 执行的**。

### 类比更准确的说法

| 组件 | 角色比喻 | 类比 HypSec |
|---|---|---|
| Normal world KVM(Linux) | "主管调度的 hypervisor" | **hostvisor**(不可信大块) |
| Realm world RMM | "VM 数据的门卫,只管隔离不管调度" | **corevisor**(可信小核) |
| Realm Guest Linux | 被保护的机密 VM | (HypSec 里的 guest VM) |

> **RMM 更像是 HypSec 的 corevisor,不是一个完整 hypervisor。** ARM 文档里也很谨慎地称它为 "Monitor",不叫 "Hypervisor"。

---

## 二、所以正确的理解应该是

### ✅ 你说对的部分
- **Normal world 的 hypervisor 是 KVM**,这没错。
- **Realm 里的 guest OS 可以是修改后的 Linux**,这也对。
- **Realm 里跑的那个东西(RMM)不是 KVM**,这个观察也对。

### ⚠️ 需要修正的部分
- **Realm world 里跑的 RMM 不是 hypervisor**,它是个**可信小核 / 访问控制层**,职能很窄。
- 所以更准确的说法不是 "Realm 中跑的 hypervisor 不是 KVM",而是 **"Realm 中根本不跑传统意义上的 hypervisor,只跑一个叫 RMM 的可信组件"**。
- **整个 CCA 系统里只有一个传统意义的 hypervisor —— 就是 Normal world 的 KVM**。RMM 只是辅助它做安全隔离的"门卫"。

---

## 三、修正后的完整图景

```
┌─────────────────────────────────────────────────────────────┐
│                       Normal World                          │
│                                                             │
│  EL0: QEMU、应用                                            │
│  EL1: Linux user/kernel(VHE 模式下也算 EL2)               │
│  EL2: Linux + KVM ← 这是真正的 hypervisor                   │
│                                                             │
│        ↓ 调度/资源请求(通过 RMI SMC 调用)                  │
└─────────────────────────────────────────────────────────────┘
                              ↓
              ┌───────────────────────────────┐
              │      EL3: Monitor (TF-A)       │
              │   仲裁 Normal ↔ Realm 切换     │
              └───────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                       Realm World                           │
│                                                             │
│  EL2: RMM ← 不是 hypervisor,是访问控制可信小核              │
│        ↓ 启动/调度 Realm VM(被 KVM 请求驱动)              │
│                                                             │
│  EL1: Realm Guest OS(修改过的 Linux,带 CCA guest patch)   │
│  EL0: Realm 应用(完全不变)                                 │
└─────────────────────────────────────────────────────────────┘
```

### 谁做什么(关键!)
| 任务 | 谁做 | 备注 |
|---|---|---|
| 决定 Realm VM 在哪个 CPU 跑、跑多久 | **KVM(Normal world)** | KVM 是真正的调度者 |
| 给 Realm VM 分配物理内存 | **KVM(Normal world)** | 但分配后必须告诉 RMM 把这些页"标记为 Realm 私有" |
| 给 Realm VM 提供 I/O 后端(virtio) | **KVM + QEMU(Normal world)** | I/O 数据走加密 / Shared 内存窗口 |
| 保证 KVM 看不到 Realm VM 的内存 | **RMM(Realm world)** | 通过 GPT(Granule Protection Table)硬件机制 |
| 保证 KVM 看不到 Realm VM 的 CPU 寄存器 | **RMM(Realm world)** | 在 world switch 时 RMM 清/存寄存器 |
| 远程证明(attestation) | **RMM(Realm world)** | 给 guest 签发 attestation token |

---

## 四、和 HypSec 的对应非常清晰

| HypSec | ARM CCA |
|---|---|
| hostvisor(EL1 Linux + KVM)| Normal world EL2 的 KVM + Linux |
| corevisor(EL2 小核) | Realm world EL2 的 RMM |
| Guest VM(普通 Linux) | Realm Guest(改过的 Linux) |
| 隔离机制:shadow NPT + IOMMU + 软件检查 | 隔离机制:GPT 硬件 + RMM 强制 |

> **ARM CCA 就是把 HypSec 的拆分思路用硬件 + 标准化的方式做出来了**:hostvisor → 留在 Normal world(就是 KVM);corevisor → 标准化为 RMM(放在 Realm world);两者之间的接口标准化为 RMI/RSI。

---

## 五、所以修正后的一句话

> **整个 ARM CCA 系统里只有一个传统意义上的 hypervisor —— Normal world 的 KVM(就是普通 Linux 加 CCA patch)。Realm world 里跑的 RMM 不是 hypervisor,而是一个可信访问控制小核,职能仅限于"防 KVM 偷窥/篡改 Realm VM 的数据"。Realm VM 里跑的可以是修改过的 Linux 内核(带 CCA guest patch),应用层完全不变。所以整个生态最大限度地复用了 Linux:host 是 Linux + KVM,guest 也是 Linux,唯一"专用"的组件就是 RMM(ARM 提供开源参考实现 TF-RMM)。**

你的整体直觉非常对,只是把 RMM 当成了"另一个 hypervisor",其实它更准确地说是 "hypervisor 的可信安全协处理器"。
