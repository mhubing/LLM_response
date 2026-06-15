# Monitor 研究方向待读资料

## 一、Cache 分区 / 资源治理

### 入门：理解 Intel RDT / CAT
- Intel 官方白皮书：搜 "Intel® Resource Director Technology Architecture Specification"
- Linux 内核文档：`Documentation/x86/resctrl.rst`
- LWN 博客："Reworking the cache resctrl filesystem"
  - https://lwn.net/Articles/772562/

### 经典论文（Google 生产环境部署经验）
- **Heracles** (ISCA'15) — "Improving Resource Efficiency at Scale"
  - Google 用 CAT + DVFS 做 co-location，cache 分区落地标杆
- **CLITE** (HPCA'20) — "Efficient and QoS-Aware Co-Location of Multiple Latency-Critical Jobs for Warehouse Scale Computers"
  - Heracles 升级版
- **Bubble-Up** (MICRO'11) — co-location 性能干扰预测开山作
- **Bubble-Flux** (ISCA'13) — Bubble-Up 续作

### 安全角度（对 monitor 方向最对口）
- **CATalyst** (HPCA'16) — "Defeating Last-Level Cache Side Channel Attacks in Cloud Computing"
  - 用 CAT 当侧信道防御，cache 分区安全用代表作
- **MASCAT** (ASIA CCS'17) — "Stopping Microarchitectural Attacks Before Execution"
- **CacheBar** (CCS'16) — cache 共享下的 channel mitigation

### NFV / 多租户
- **ResQ** (NSDI'18) — "Enabling SLOs in Network Function Virtualization"
  - 用 CAT 隔离 NFV 性能

### 备选轻量方向（如果 CAT 太重）
- **PMU 访问控制**（推荐切入点）
  - 比 CAT 简单一个数量级，能 cover SLA 审计 + 侧信道防御两个角度
- IOMMU 策略 monitor 化
- 中断路由保护

---

## 二、SLA 可验证性 / 可信审计

### 必读：可信审计 / tamper-evident logging
- **Hardlog** (S&P'22) — "Practical Tamper-Proof System Auditing Using a Novel Audit Device"
  - **必读**，专门做防篡改审计的硬件辅助
- **Custos** (NDSS'20) — "Practical Tamper-Evident Auditing of Operating Systems Using Trusted Execution"
  - TEE-based 审计
- **PillarBox** (RAID'14) — TEE 审计鼻祖

### 远程证明 (remote attestation) 系列
- **TrustVisor** (S&P'10) — "Efficient TCB Reduction and Attestation"
- **Flicker** (EuroSys'08) — "An Execution Infrastructure for TCB Minimization"
  - attestation 极简化代表
- **Town Crier** (CCS'16) — "An Authenticated Data Feed for Smart Contracts"
  - 区块链上下文，思想通用

### 云计算 SLA 验证（论文少，工业实践多）
- Confidential Computing Consortium 白皮书系列：https://confidentialcomputing.io
- **AWS Nitro Attestation Document** — 产业落地范本
- **Azure Confidential VM Attestation** 文档
- **Google Confidential Computing** docs

### 可验证计算（更野心的方向）
- **Pinocchio** (S&P'13) — verifiable computation 早期工作
- **Geppetto** (S&P'15)
- libsnark / circom — zk-SNARK 工程库

### 工业博客（PPT 引用方便）
- Cloudflare Geo Key Manager
- Apple Private Relay
- Signal Sealed Sender

### 搜索关键词
- "trusted monitoring" + cloud
- "attested performance counter"
- "verifiable telemetry"
- "tamper-evident audit" + kernel

---

## 三、行动建议

1. **优先读**：Hardlog (S&P'22) + Heracles (ISCA'15) + CATalyst (HPCA'16)
2. **入门顺序**：先看 Intel RDT 白皮书 + Linux resctrl 文档建立背景
3. **方向选择**：考虑用 PMU 访问控制替代 CAT 切入，复杂度低、覆盖面广（可同时支撑 SLA 审计 + 侧信道防御两条 paper 线）
