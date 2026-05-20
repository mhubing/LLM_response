# 如何读论文

看论文这件事，"快速掌握"和"深入理解"是两套不同的读法，要分开。我按你平时的工作（系统/安全/TEE 方向）讲，这类偏 system 的论文有比较固定的套路。

## 一、先想清楚：你这次读这篇论文是为了什么

这是最关键的一步，决定了下面所有动作。读论文的目的大致分四类，每类的读法不一样：

- **找方法借鉴**：你正在做隔离/中断设计，想看别人怎么解决。重点读 design 和 implementation，跳过大部分 motivation。
- **写 related work**：你要在自己论文里引用它。重点读 abstract、intro、和它自己 related work 怎么定位自己，提取一句话总结。
- **追踪一个领域**：第一次接触这个方向，要建立 mental map。重点读 intro、background、和它批评/对比的工作。
- **复现 / 攻击 / 改进**：要动它的实现。重点读 implementation、threat model、evaluation 的 setup。

你之前提到"不要结合之前的论文，单独讲一篇"，其实就是你自己已经在区分这两种模式：建立 mental map 时要对比，但深入吃透一篇时不要对比，否则会混淆这篇自己的 claim。

## 二、三遍读法（system 论文的标准做法）

这是 Keshav 那篇 *How to Read a Paper* 的经典方法，对系统类论文特别适用：

**第一遍（5–10 分钟，决定要不要继续读）**：只看 title、abstract、intro 的最后一段（contribution list）、所有 section 标题、conclusion、references 扫一眼有没有你认识的工作。读完能回答：这篇在解决什么问题？claim 是什么？大致用了什么方法？是不是我要的？

**第二遍（1 小时，掌握主要内容）**：读 figures、tables、design 的 high-level 描述、evaluation 的主图。**跳过证明、跳过实现细节、跳过你看不懂的公式**。读完能回答：它的核心 insight 是什么？关键设计选择是什么？evaluation 支不支持它的 claim？

**第三遍（几小时，吃透）**：逐字读，包括你不懂的部分；尝试在脑子里"重新发明"它的设计——如果让你做这个题，你会怎么做，它和你的差在哪。读完能找出它的弱点和未明说的假设。

**绝大多数论文你只需要做到第一遍或第二遍**。第三遍只留给和你工作直接相关、要深入引用或对抗的论文。

## 三、System / 安全类论文的提问清单

读 system 论文（OSDI/SOSP/ASPLOS/USENIX Security/CCS 这类）时，按下面这几个问题去"扫"，比从头读到尾高效得多：

**问题与动机**
- 它解决什么问题？这个问题之前为什么没解决？
- 它的 motivating example 是什么？（system 论文一般会有一个具体场景，这个场景往往就是它的整个 design 的"原型"）

**威胁模型 / 假设**（安全类论文必看，看不清这个就读不懂 design）
- 攻击者能力是什么？（能不能发中断？能不能改页表？是不是可信 firmware？）
- 信任谁？（TCB 包括什么？）
- 不防什么？（侧信道？拒绝服务？物理攻击？）

**核心机制**
- 它的 key insight 用一句话能不能讲清？
- 关键设计选择有哪些？每个选择的 trade-off 是什么？为什么不选另一种？
- 哪些是新东西，哪些是已有技术的组合？

**实现栈**
- 跑在什么上面？（QEMU / FPGA / 真实硬件 / gem5？）
- 改了什么层？（硬件 / firmware / kernel / userspace？）
- 这个实现栈本身就决定了它能投什么会——这正是你之前想问的。

**Evaluation**
- 它的 baseline 是什么？baseline 选得公平吗？
- 主要 metric 是什么？（性能开销？TCB 大小？安全性证明？）
- 哪些场景没测？（这往往是它的弱点）

**局限与未来工作**
- 作者自己承认的局限是什么？
- 哪些是它没承认但你看出来的局限？

## 四、几个加速技巧

**先看图**。System 论文的 Figure 1 通常是整个 architecture overview，吃透这一张图就掌握了 60%。Figure 2 通常是关键流程（attack flow / protocol flow / control flow），吃透这张图再加 60%。

**先看 evaluation 的主表**。Evaluation 的第一张表/图通常直接告诉你"它和谁比、赢了多少"，这往往比 intro 更准确地反映它的 claim。

**用它的 related work 当地图**。一篇好论文的 related work 会把整个领域的工作分类，告诉你它在哪个分支。第一次进一个领域时，找 2–3 篇近年顶会论文，把它们的 related work 拼起来，比读 survey 还快。

**带着假设读**。读之前先猜：如果是我做这个题，我会怎么设计？然后看它和你猜的差在哪。这样读完印象最深。

**记 one-liner**。每篇读完用一句话总结："X 通过 Y 方法解决了 Z 问题，代价是 W"。比如 SGX 的 one-liner："SGX 通过 CPU 扩展 + enclave 内存加密，在不可信 OS 下保护 user-space 代码，代价是不能直接发 syscall 且受侧信道攻击。" 这个 one-liner 是你以后写 related work 的素材。

## 五、针对你现在的工作的具体建议

你现在在做特权指令隔离 + protect 机制，对应的读论文模式应该是：

- **SKEE/TIDE/CKI 这种和你直接对标的**：第三遍读，逐字吃透，特别是 implementation section 和 threat model。这些是你要在 related work 里精确引用、并 claim "我们和它们的区别是 X"的论文。
- **SGX/TrustZone/Keystone 这种背景对比**：第二遍读就够，掌握 high-level 机制和它处理中断/隔离的思路，能在你 paper 的 background section 里写一段对比。
- **顶会上随便扫一眼的**：第一遍读，记 one-liner，建索引。哪天发现相关再回去深读。

## 六、论文的 Insight 是什么

"Insight" 这个词在论文里是个**高频但很虚**的词，刚开始读会觉得"作者总说 our key insight is...，到底什么算 insight？"。我分几个层次讲清楚。

### 一、Insight 的核心含义

Insight 在论文语境下，指的是**作者发现的一个"别人没注意到、或没用好"的关键观察，这个观察让原本难解的问题变得可解**。

它不是：
- 不是"我们做了什么"（那是 contribution）
- 不是"我们怎么做的"（那是 design / mechanism）
- 不是"我们解决了什么问题"（那是 motivation）

它是：**"为什么我们能这么做" 背后的那个支点**。

一个好的 insight 通常能用一句话讲清，听完之后让人有"哦，对啊，这样确实可以"的感觉。它像是杠杆的支点——支点找对了，design 就自然顺出来了。

### 二、Insight 通常出现在哪几类地方

在 system / 安全论文里，insight 一般是下面几类观察之一：

**(1) 对硬件/系统机制的"非典型用法"**

发现某个硬件特性原本是为 A 设计的，但可以拿来做 B。

- 例：SGX-Step 的 insight 是"APIC timer 可以被配置成极短的间隔，使 enclave 在执行单条指令后就被打断"。APIC timer 本来是给 OS 调度用的，作者发现它的精度足以做指令级单步。
- 例：很多用 PMU (performance counter) 做安全检测的工作，insight 是"PMU 计数器的某个事件在攻击发生时会有特征模式"。

**(2) 对工作负载/行为的关键观察**

发现实际系统里某种行为的分布、模式，让通用方案可以被特化。

- 例："99% 的 syscall 集中在 50 个之内"——于是可以做 syscall filter 加速。
- 例："enclave 进出非常频繁但每次切换数据量很小"——于是可以省掉某些保存/恢复开销。

**(3) 把两个看似无关的领域 / 机制连起来**

发现 A 领域的技术可以解决 B 领域的问题。

- 例：用编译器的 dataflow 分析做 kernel 安全检查。
- 例：用虚拟化的 EPT/二级页表做 intra-kernel 隔离（SKEE 就属于这类）。

**(4) 找到一个"不变量" (invariant)**

发现某个性质在所有相关场景下都成立，于是可以作为安全/正确性的基础。

- 例：SKEE 的 insight 之一是"如果能保证某段代码运行时 MMU 配置无法被修改，就能在同特权级实现隔离"。这个"MMU 配置不可修改"是 invariant。

**(5) 否定一个被默认接受的假设**

发现领域里大家默认的某个前提其实不必要，于是可以打开新的设计空间。

- 例："大家默认隔离必须靠特权级，但其实在同特权级下用 X 也能做到"。
- 例："大家默认 TEE 中断处理必须切世界，但其实可以..."

### 三、怎么从论文里识别 insight

几个识别信号：

**关键句式**。论文里通常会有这些句式标记 insight：
- "Our key insight is that..."
- "We observe that..."
- "The key observation is..."
- "We leverage the fact that..."
- "Based on this observation, ..."

看到这些句子要**停下来读三遍**——这往往是整篇论文最值钱的一句话。

**位置**。Insight 通常出现在 intro 的中后段（讲完 motivation、讲 contribution 之前），或者 design section 的开头。如果论文有 "Overview" 或 "Approach" section，这个 section 的第一段往往就是 insight。

**和 design 的关系**。读完 insight 之后再读 design，应该有"design 是 insight 的自然推论"的感觉。如果 design 看起来还是很复杂、和 insight 关系不大，那要么是你没读懂 insight，要么这个 insight 其实没那么核心。

### 四、Insight ≠ Contribution

这两个经常被混淆，但严格来说差别很大：

| | Insight | Contribution |
|---|---|---|
| 性质 | 观察 / 认识 | 产出 / 成果 |
| 数量 | 通常 1 个（最多 2 个）| 通常 3–5 个 |
| 形式 | "我们发现 X" | "我们设计了 / 实现了 / 评估了 Y" |
| 作用 | 支撑 design | 总结贡献 |

举例：
- **Insight**: "EPT 可以在不切换特权级的前提下提供另一套地址空间"
- **Contributions**: 设计了基于此 insight 的隔离系统 X、实现于 KVM、评估表明开销 < 5%

### 五、对你写论文的启发

你之前提到要"按 SKEE/TIDE/CKI 的口径写"，这里有个写作技巧：**每篇相关工作你都应该能用一句 insight 概括它**。比如：

- SKEE 的 insight：在同特权级用受保护的 MMU 切换提供隔离环境
- TIDE 的 insight：（按它自己的描述提取）
- CKI 的 insight：（按它自己的描述提取）

然后**你自己工作的 insight 要能和它们形成对照**——"它们靠 X，我们的 insight 是 Y，因此我们能做到 Z 而它们不能"。这是 related work 段落最有力的写法。
