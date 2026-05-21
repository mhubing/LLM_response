# Runtime 的概念与常见类型

## 一、Runtime 的定义

**Runtime（运行时）** 这个词在计算机领域有两层含义，需要先区分清楚：

1. **Runtime（时间阶段）**：指程序"正在运行"的那个时间阶段，与 compile-time（编译时）、link-time（链接时）、load-time（加载时）相对。例如"runtime error"指程序运行期间才出现的错误。

2. **Runtime（运行时系统 / runtime system）**：指支撑程序运行所需的**软件基础设施**——一套在程序执行期间提供环境、服务、资源管理的代码和机制。这是你问的"sandbox runtime / eBPF runtime"中的含义。

更精确地说，一个 runtime 通常负责以下若干职责的组合：
- **执行模型**：解释/JIT/AOT 后的代码如何被驱动执行（指令分派、调用约定、栈帧管理）
- **资源管理**：内存分配、GC、线程/协程调度、I/O
- **隔离与安全**：地址空间、权限、能力（capability）、沙箱边界
- **ABI/系统接口**：暴露给被托管程序的 syscall/host function/intrinsics
- **生命周期**：加载、验证、链接、卸载

可以把 runtime 理解为 **"被托管程序与底层硬件/OS 之间的中间层"**，它定义了"程序在什么世界里运行"。

---

## 二、常见 Runtime 分类

### 1. 语言运行时（Language Runtime）

为某种编程语言的程序提供执行环境，通常包含 GC、调度、标准库胶水等。

- **JVM（Java Virtual Machine）**：执行 Java 字节码，含 GC、JIT（HotSpot/C2）、类加载器
- **CLR（Common Language Runtime）**：.NET 的运行时，支持 C#/F# 等
- **Go runtime**：goroutine 调度器、GC、channel、netpoller，**直接编进二进制**
- **Node.js / V8**：JavaScript 的执行环境（V8 是引擎，Node 在其上加 libuv 提供 I/O）
- **Python（CPython）/ Ruby（YARV）/ Erlang BEAM**：解释器 + 内置调度/GC
- **Rust**：常说"Rust 没有运行时"，更准确的说是**极小运行时**——只有 panic、栈展开等少量机制，无 GC 无绿色线程

### 2. 容器与 OS 级 Runtime

- **OCI Runtime（runc、crun、youki）**：底层执行容器进程，操纵 namespace/cgroup
- **High-level Container Runtime（containerd、CRI-O）**：管理镜像、生命周期，调用底层 OCI runtime
- **Kata Containers、gVisor、Firecracker**：**沙箱化容器运行时**，用 VM 或用户态内核做强隔离
  - gVisor 在用户态用 Go 实现了一个"应用内核"拦截 syscall
  - Firecracker 是 microVM monitor

### 3. 沙箱 / 虚拟化 Runtime（你提到的 sandbox runtime）

核心职责是**隔离 + 受控执行**：

- **WebAssembly Runtime**：Wasmtime、Wasmer、WAMR、V8（Wasm 部分）。执行 Wasm 字节码，提供能力式的 host import（WASI）
- **eBPF Runtime**：Linux 内核中的 verifier + JIT/解释器，运行受限字节码，提供 helper functions 和 maps 作为 ABI
- **浏览器 JS Sandbox**：V8/SpiderMonkey/JavaScriptCore，加上同源策略、Site Isolation
- **TEE Runtime**：SGX SDK（Intel）、OP-TEE（ARM TrustZone）、Keystone（RISC-V）——在 enclave/secure world 里提供精简运行时（你做隔离工作时熟悉这一类）

### 4. 序列化/RPC/AI Runtime

- **gRPC runtime**：处理 stub 调用、流、拦截器
- **ONNX Runtime / TensorRT / TVM runtime / PyTorch runtime**：张量调度、算子分派、kernel 选择、设备内存管理
- **CUDA Runtime（libcudart）**：相对 CUDA Driver API 的高层封装，管理 context、stream、kernel launch

### 5. Serverless / Function Runtime

- **AWS Lambda Runtime**（Node/Python/Custom Runtime API）
- **Cloudflare Workers Runtime**（基于 V8 isolate）
- **Knative**：在 K8s 上的 serverless runtime

### 6. 其他常见 Runtime

- **MPI runtime（OpenMPI、MPICH）**：进程组、集合通信
- **OpenMP runtime（libgomp、libomp）**：fork-join、任务调度
- **Actor runtime（Akka、Orleans）**：消息传递、位置透明

---

## 三、辨析两个易混点

**Runtime vs. Library**：库通常是被动调用的函数集合；runtime 通常**主动控制执行流**（调度、GC、事件循环），并定义被托管代码必须遵守的协议。但二者边界模糊——`libcudart` 既叫 runtime 也是 library。

**Runtime vs. VM/Engine**：VM/Engine（如 V8、HotSpot）通常指**执行引擎**本身（解释 + JIT）；Runtime 是更外层的概念，通常 = Engine + 运行库 + 系统接口胶水。Node.js = V8（engine）+ libuv + 标准库 = Node runtime。

**Runtime vs. OS**：OS 给所有进程提供 syscall；Runtime 给**特定一类托管程序**提供更高层、更受限或更便利的接口。eBPF runtime 在内核里，反而限制了 eBPF 程序能做什么——这就是你做特权指令隔离时类似的"宿主对受控代码的边界定义"。

---

## 四、一个统一视角

无论是 JVM、Wasmtime 还是 eBPF runtime，它们的**通用骨架**几乎一致：

$$
\text{Runtime} = \underbrace{\text{Loader/Verifier}}_{\text{加载验证}} + \underbrace{\text{Executor (interp/JIT/AOT)}}_{\text{执行}} + \underbrace{\text{Host ABI}}_{\text{对外接口}} + \underbrace{\text{Resource Mgmt}}_{\text{内存/调度/隔离}}
$$

<img width="787" height="69" alt="image" src="https://github.com/user-attachments/assets/ee431ce2-16c6-41ad-b80d-f89ca6e87f2e" />


理解了这个骨架，再看任何新的 runtime（比如最近的 confidential computing runtime、各种 AI agent runtime），都可以快速对位它在四个维度上各做了什么取舍。
