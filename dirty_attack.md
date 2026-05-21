# Dirty Pipe / Copy Fail / Dirty Frag / Fragnesia —— 攻击逻辑入门

这四个 Linux 内核漏洞虽然分布在不同子系统（pipe、crypto、IPsec、TCP），但它们本质上是**同一种攻击模式的不同变体**。本文按"攻击者视角的逻辑链"组织，而不是按时间或漏洞编号排列，目的是让防护人员先理解通用模式，再看每个漏洞的差异。

---

## 一、攻击者的最终目标和流程

普通用户想拿 root，剧本三步：

```
[1] 拿到一个能写「文件 page cache」的内核漏洞原语
       ↓
[2] 用这个原语篡改某个高权限文件在内存中的副本
    (典型目标：/usr/bin/su、/etc/passwd、/etc/ld.so.preload)
       ↓
[3] 执行 / 触发该文件 → 内核以 root 加载被污染的内存页 → root shell
```

四个漏洞的差别只在第 [1] 步用的"撬棍"不同。第 [2][3] 步在四个漏洞里几乎一样。

下面先讲第 [1] 步（攻击者怎么写进 page cache），再讲 [2][3] 步（写进去之后怎么变 root）。

---

## 二、漏洞原语：攻击者如何写入 page cache

### 2.1 通用模式：splice() 把文件页"借"给内核某子系统

攻击者用一组**完全合法**的系统调用，把 `/usr/bin/su` 的 page cache 页**主动喂**给一个有 bug 的内核子系统：

```c
// 1. 普通用户对 /usr/bin/su 有读权限，open 合法
int su_fd = open("/usr/bin/su", O_RDONLY);

// 2. 创建一个有 bug 的内核接口（加密 socket / pipe / IPsec socket ...）
int target_fd = socket(AF_ALG, SOCK_SEQPACKET, 0);
// ...或者 pipe()、xfrm socket 等

// 3. 关键一步：splice 零拷贝，把文件页的指针挂到目标 fd 的队列
splice(su_fd, NULL, target_fd, NULL, 4096, 0);
```

`splice()` 的本质是**传指针不传数据**：它把 `/usr/bin/su` 对应的 `struct page` 引用挂到 `target_fd` 的 buffer 队列里。这块物理内存现在**有了两重身份**：

- `/usr/bin/su` 的 page cache
- 目标子系统的输入 buffer

**内核为什么不拦？** 每一步都合法：
- 攻击者有 `/usr/bin/su` 的读权限
- `splice()` 是为 nginx/kafka 这类零拷贝场景设计的标准接口

### 2.2 Bug 在哪：内核子系统把共享页当成"私有 scratch buffer"

接下来某个内核子系统会处理这个 buffer。它的代码（被引入漏洞的那部分）假设：**"调用者传给我的 buffer 是它自己的私有缓冲区，覆盖掉无所谓"**。

但这个 buffer 实际上是通过 splice 共享进来的文件页。内核没有去检查、也没意识到这一点，直接 `memcpy / xor / decrypt` 写下去 → 文件 page cache 被污染。

> **类比**：你借了图书馆一本书（只能看），但图书馆有个"装订机"接口能把书页和便签纸一起装订。攻击者把书页 splice 进装订机，装订机以为它在改便签纸，于是把字印到了书页上 —— 而图书馆本来不允许你写这本书。

### 2.3 关键观察

- **走的不是 `write()` 路径** → VFS 权限检查从未发生
- **dirty 标志没置上** → 磁盘永远不会被修改
- **写入内容由攻击者控制**（密文、IV、密钥、pipe 内容…均可控）

到这里攻击者已经能**任意写文件 page cache** —— 但这怎么就等于 root？下一章解释。

---

## 三、提权方式：page cache 被污染如何变 root

首先为新手补两块必要的 background：

- `/usr/bin/su` 是只读文件，为什么内核改 page cache 后能"生效"？
  - 软件层（§3.1）、物理内存层（§3.2）、硬件 MMU 层（§3.3）三层拆解
- 改了之后凭什么就能拿 root？
  - 以 setuid 二进制 `/usr/bin/su` 为例（§3.4）

### 3.1 "只读"只是给用户态的约束，不是给内核的

`/usr/bin/su` 的 mode 是 `-rwsr-xr-x`，意思是：

- VFS 在 `open(O_WRONLY)` 时检查权限位，普通用户被拒。
- 这个检查在 **syscall 入口**做，是一段 C 代码里的 `if`，不是 CPU 的 MMU 保护。

而内核运行在 ring 0，管理全部物理内存。**没有任何硬件机制阻止内核往一块内存上写东西**。"这个文件只读"对内核只是一条**代码自律规则**："我应该按规矩别去改它的 page cache 页"。第二章里的 bug，就是某个子系统"忘了自律"。

### 3.2 Page cache 就是普通的内核可写 RAM

#### 3.2.1 page cache 是干什么的

磁盘 I/O 比内存慢 4–6 个数量级（NVMe 微秒级 vs DRAM 纳秒级，机械盘更是毫秒级）。如果每次 `read("/usr/bin/su")` 都去捅磁盘，系统会卡到不能用。所以 Linux（和几乎所有现代 OS）都在 VFS 和块设备之间放了一层 **page cache**，加速以下几件事：

| 场景 | 没有 page cache | 有 page cache |
|---|---|---|
| 同一进程反复读同一文件 | 每次都查磁盘 | 第一次后命中内存 |
| **多进程读同一文件**（典型：`/usr/bin/su`、共享库 `.so`） | 每个进程都查一遍磁盘，且各自占一份内存 | **全部进程共享同一份 4KB 物理页** |
| `execve` 加载二进制 | 完整读磁盘再映射 | 直接把 page cache 页 mmap 进进程地址空间 |
| 顺序读 | 一次一块 | readahead 预读后续块 |
| `write()` 到普通文件 | 同步写磁盘 | 先改 page cache + 标 dirty，writeback 线程异步刷盘 |

**对本文最关键的是第二条**："多进程共享同一份物理页"是设计意图，不是 bug。这正是为什么攻击者只要污染了 `/usr/bin/su` 在 page cache 里那 4KB，**之后所有用户执行 `su` 看到的都是被改过的版本** —— 不需要重新感染，不需要每个用户单独中招。

> 简而言之：page cache 把"磁盘文件"虚拟化成"一份全局共享的内存副本"。这种共享是性能上的福音，也是这类漏洞的攻击放大器。

#### 3.2.2 读文件时 page cache 怎么落地

读文件时：

```
磁盘块 ──读入──► 内核分配 struct page (4KB物理内存)
                   ├── 挂到 inode 的 address_space 上（这就叫 page cache）
                   └── read() 把这4KB拷贝给用户态
```

这 4KB **就是普通内核 RAM**。任何内核里的 `memcpy(dst, src, n)`，只要 `dst` 落在这里，写入就会成功，MMU 不会报错。

**区别只在于这页是否被打上 dirty 标志**：dirty → writeback 线程会写回磁盘；不 dirty → 磁盘不会变，但 page cache 里已经是新内容。

四个漏洞的共同点：**改了 page cache 但没标 dirty**。后果：
- 磁盘文件**真的没动**（`sha256sum` 对比磁盘原文件 ≠ 内存里的内容）
- 内存中的副本被攻击者控制
- 任何进程 `read()` 或 `execve()` 这个文件，内核**优先从 page cache 取** → 拿到的是被改过的字节
- 重启或 `echo 3 > /proc/sys/vm/drop_caches` 会丢缓存重读磁盘，攻击痕迹"自愈"

> **防御者注意**：传统文件完整性监控（AIDE、Tripwire 这类对比磁盘 hash 的方案）**完全检测不到**这类攻击。事件响应时主动 `drop_caches` 是恢复正常状态的重要一步。

### 3.3 深入硬件层：内核页表里 page cache 也是 RW

§3.1、§3.2 讲了"软件层为什么挡不住"。再深入一层问：**既然 page cache 是只读文件的副本，为什么内核页表里它对应的 PTE 不是 RO？硬件 MMU 不能在那里把这一脚拦下来吗？** 答案是 **不能 —— page cache 在内核地址空间里就是 RW 的**。

#### 3.3.1 内核 direct map：所有物理 RAM 都映射为 RW

x86_64 Linux 启动时建立一个 **direct mapping / linear map**（从 `page_offset_base`，通常是 `0xffff888000000000` 开始），把**全部物理 RAM 一一映射到内核虚拟地址空间**，权限是 **RW + NX**。

```
物理地址 0x12345000 (某个 page cache 页)
        ↓
内核虚拟地址 0xffff888012345000   ← PTE 是 RW
```

所有 `struct page` 都能通过 `page_address(page)` / `kmap()` 拿到这个 direct map 地址。内核子系统拿到 `struct page *` 后做 `memcpy`，写的就是这个 RW 映射 —— MMU 不会拦。

**page cache 页在 direct map 里和栈页、slab 页、网络 skb 页没有任何区别**，都是 RW。

#### 3.3.2 用户态的 RO 跟内核没关系

你可能想到："那 `mmap(/usr/bin/su, PROT_READ)` 之后那块内存不是 RO 吗？"

是的，但 RO 只在**用户进程的页表**里：

```
用户 PTE  ──映射──► 物理页 P  (用户视角: RO, 写会 SIGSEGV)
内核 PTE  ──映射──► 物理页 P  (内核视角: RW, 直接写无障碍)
```

同一个物理页，两套页表，两种权限。内核改 P 的时候走的是自己的 PTE，用户态那个 RO 标记完全不在路径上。

> 这正是经典 **Dirty COW (CVE-2016-5195)** 的根：用户 PTE 是 RO + COW，但内核可以通过 `get_user_pages(FOLL_WRITE)` 拿到一个临时的 RW 映射往里写 —— 竞态下绕过了 COW 语义。本文这四个漏洞是它的精神续作。

#### 3.3.3 CR0.WP 不救你

x86 上 `CR0.WP = 1`（Linux 默认开）意味着即便是 ring 0，CPU 也要遵守 PTE 的 W 位。听起来很美好 —— 但前提是**那条 PTE 本身得是 RO**。direct map 的 PTE 是 RW 写死的，WP 检查通过了也照样能写。

WP 真正保护的是 `.rodata`、内核代码段这类**明确标 RO 的内核映射**，而不是 page cache。

#### 3.3.4 哪些内核页才是真正 MMU 级 RO？

打开 `CONFIG_STRICT_KERNEL_RWX` 后：

| 区域 | 内核 PTE 权限 |
|---|---|
| 内核 `.text`（代码） | RX |
| 内核 `.rodata` | R |
| 内核 `.data` / 栈 / slab | RW + NX |
| **direct map（含 page cache）** | **RW + NX** |
| 模块代码 | RX |

所以"写 `/usr/bin/su` 的 page cache 页"在 MMU 看来跟"写一个 `kmalloc` 出来的 buffer"是完全一样的操作 —— 没有硬件 trap，没有 fault，CPU 一路绿灯。

#### 3.3.5 为什么不把 page cache 设成 RO？

理论上可以：每次 writeback 前临时 `set_memory_rw()`，写完再 `set_memory_ro()`。但 page cache 命中是热路径，每次文件读写都要改 PTE + flush TLB → **性能直接崩盘**。所以内核选择"全 RW + 代码自律"。

结论回到 §3.1：**唯一的防线是内核代码的逻辑约束** —— 每个子系统拿到 `struct page *` 时要正确判断"这页是我私有的吗？是共享 page cache 吗？是 splice 来的吗？"。第四章的四个 bug，正是反复栽倒在这个判断上。

> **防御者注意**：既然 MMU 不会替你拦，**对关键二进制做内核度量**就成了能感知污染的少数手段之一。IMA-appraisal 在 `bprm_check_security` 这个 LSM hook 上对将要 execve 的内存页做哈希比对，能在 page cache 已被改的情况下拒绝加载 —— 详见 §5.2 / §5.6。

### 3.4 为什么挑 setuid 二进制：以 /usr/bin/su 为例

直觉上你可能想问："`/usr/bin/su` 是 root 的工具，普通用户不能执行吧？" —— **恰恰相反，普通用户不仅能执行，这正是它危险的根源**。

#### 3.4.1 setuid 让"普通用户可执行" + "执行时变 root"

看 `/usr/bin/su` 的权限：

```
$ ls -l /usr/bin/su
-rwsr-xr-x 1 root root ... /usr/bin/su
```

注意中间那个 **`s`**（在 owner 的执行位上），这是 **setuid 位**。它的含义是：

> 任何用户执行这个文件时，**进程的 effective UID 会变成文件 owner 的 UID**（这里 owner 是 root，所以 EUID = 0）。

`su` 必须有这个属性，否则普通用户根本没办法切到 root —— 因为校验密码、读 `/etc/shadow`、调 `setuid(0)` 这些操作本身就需要 root 权限。系统里类似的 setuid-root 二进制还有 `sudo`、`passwd`、`mount`、`ping`（老版本）、`pkexec` 等。

所以 setuid 文件的三条性质合起来：

- 文件 owner 是 root ✅
- 文件可被所有人执行 ✅（`x` 位对 group/other 都开着）
- 执行时进程 EUID = 0 ✅（setuid）

→ 一个**人人可触发、触发后跑在 root 上下文里的"合法跳板"**。

#### 3.4.2 正常 su 为什么不会让你随便变 root

虽然进程一启动 EUID 就是 0，但 `su` 自己的代码里写了一套"自律逻辑"：

1. 读取 `/etc/shadow` 里目标用户的密码哈希；
2. 提示输入密码、做哈希比对；
3. 比对失败就 `exit(1)`，比对成功才调用 `setuid(0)` / `execve("/bin/bash", ...)` 把 root shell 交给你。

也就是说，**root 权限从内核角度一开始就给了进程，是 `su` 自己选择"先验密码再交出来"**。这是一种 userspace 层面的自我约束。

#### 3.4.3 攻击者篡改的本质：去掉自律

理解了上面两点，篡改的目的就一目了然：**把"先验密码"那段自律逻辑去掉或绕过，让进程直接把 root shell 交出来**。

概念上等价于把：

```c
if (check_password() == OK) {
    setuid(0); exec_shell();
} else {
    exit(1);
}
```

改成：

```c
setuid(0); exec_shell();   // 不管谁来执行，直接给 root
```

或者插入一个后门分支：输入特定密码（比如 `"letmein"`）就直接放行。

具体到字节层面，第二章里的写原语（4 字节或 1 字节）通常用来覆盖入口附近的几条指令，跳到一段事先准备好的 shellcode，效果等同上面这段 C 代码。

#### 3.4.4 一次性漏洞 → 持久化提权后门

这才是这种攻击爱挑 setuid 二进制下手的真正原因：

1. 攻击者用某个**临时的、受限的**漏洞原语（Dirty Pipe 之类）改掉 `/usr/bin/su` 的字节；
2. 漏洞利用本身可能只是一次性的、不稳定的、留痕迹的；
3. 但 `/usr/bin/su` 是**长期存在、setuid root、所有用户随时可执行**的文件；
4. 之后攻击者（甚至以一个普通 shell 回连进来时）只要 `su` 一下就能稳定拿 root，**不再依赖原始漏洞**。

本质是**权限升级路径的固化**：把"一次性内存漏洞"转化成"持久化的提权后门"。
注意"持久化"指的是 page cache 寿命内有效（直到 evict / 重启 / `drop_caches`），不是磁盘级持久化。

### 3.5 其他常见篡改目标

`/usr/bin/su` 不是唯一选择，下表是另外两个高价值目标：

| 目标文件 | 利用方式 | 触发条件 |
|---|---|---|
| `/usr/bin/su` | 改入口指令为 spawn shell 的 shellcode | 用户执行 `su` |
| `/etc/passwd` | 改 root 行的密码哈希字段 | `su root` + 已知密码 |
| `/etc/ld.so.preload` | 加一行指向攻击者 `.so` 的路径 | **任何**动态链接程序运行；若被 SUID 程序读到则更可怕 |

`/etc/ld.so.preload` 那条触发面最大，且不依赖特定二进制 —— 任何动态链接二进制启动都会被 ld.so 读这个文件。

---

## 四、四个漏洞各自的"撬棍"在哪

四个漏洞都符合 §2 的模式，只是"借"给了不同子系统、利用了不同的逻辑 bug。

### 对比总表

| 漏洞 | 年份 | "借给"的子系统 | 写原语 | 必要权限 | 关键特点 |
|---|---|---|---|---|---|
| Dirty Pipe | 2022 | pipe | 写 page cache（有偏移/长度限制） | 普通用户 | 经典开山之作 |
| Copy Fail | 2026.04 | `algif_aead`（加密 socket） | 4-byte 任意写 | 普通用户 | 跨容器；PoC 几十行 Python |
| Dirty Frag | 2026.05 | xfrm-ESP (IPsec) + RxRPC | 4-byte 任意写 | `CAP_NET_ADMIN`（可自取） | 绕过 Copy Fail 缓解 |
| Fragnesia | 2026.05 | xfrm ESP-in-TCP | 1-byte 任意写 | `CAP_NET_ADMIN`（可自取） | Dirty Frag 补丁不修它 |

### 各自的 bug 细节

**Dirty Pipe (CVE-2022-0847)**
- pipe buffer 的 `PIPE_BUF_FLAG_CAN_MERGE` 标志在 `splice()` 后没被清零
- 攻击者构造一系列 pipe 操作让本不该可合并的页（只读文件页）被打上"可合并"标志
- 接着往 pipe 里 `write()`，内核就把数据直接写进了文件 page cache

**Copy Fail (CVE-2026-31431)**
- 2017 年加的 "in-place AEAD" 性能优化：`decrypt(src, dst=src, len)` 原地解密省一次 memcpy
- 当 src 是 splice 过来的文件页时，"in-place 写回"就是写文件 page cache
- 攻击者控制密文/密钥 → 控制解密后的 4 字节

**Dirty Frag (CVE-2026-43284 + CVE-2026-43500)**
- 同样模式，触发面换成网络栈
- xfrm-ESP (IPsec) 接收路径的"in-place 解密"，buffer 是 splice 来的文件页时写 page cache → 4-byte STORE 原语
- RxRPC 一处 page cache 写 bug，同时提供创建 namespace 所需的 `CAP_NET_ADMIN`
- 两者**链起来**：先用 RxRPC 在 user/net namespace 自取 `CAP_NET_ADMIN`，再用 ESP 精确写
- **重要**：即便管理员按 Copy Fail 建议拉黑了 `algif_aead`，Dirty Frag 走 `esp4/esp6/rxrpc`，照打

**Fragnesia (CVE-2026-46300)**
- 同源于 xfrm/ESP，但只用 ESP-in-TCP，不需要 RxRPC
- 当 TCP socket 先用 `splice()` 把文件页喂进 receive queue，再**切换**到 espintcp 模式时，内核把队列里的文件页当 ESP 密文做就地解密
- 攻击者控制 IV → 确定性的 1-byte 写
- 名字由来：socket buffer (skb) 在 coalescing 合并过程中"忘了"frag 是共享的（frag + amnesia）
- **重要**：Dirty Frag 的内核补丁**不修** Fragnesia，需要单独打

---

## 五、给安全防护人员的关键 takeaway

### 5.1 真正的"漏洞家族"看的是原语，不是名字

四个漏洞 CVE 不同、子系统不同，但本质都是：

> **"splice 进来的共享文件页 → 被内核某子系统当成自己私有的 scratch buffer 写回"**

防护策略应该围绕这个模式建立，而不是逐个 CVE 打补丁。

### 5.2 检测的盲区与方法

- **磁盘完整性扫描失效**：磁盘文件没改，AIDE/Tripwire 对比 hash 看不出问题
- **内存完整性才有效**：定期对比关键 SUID 文件的**内存映像**与磁盘内容；IMA-appraisal 这类基于内核度量的方案比纯 userspace hash 比对更可靠
- **包管理器校验**：`dpkg -V` / `rpm -Va` 在事件响应时也能用 —— 但同样要注意它读到的也可能是被污染的 page cache，必要时先 `drop_caches` 再校验
- **可疑系统调用组合**：监控 `splice()` + `AF_ALG / xfrm / pipe` 的非常规组合，尤其是源 fd 指向 `/usr/bin/su`、`/etc/passwd`、`/etc/ld.so.preload` 等敏感文件
- **Microsoft 观察到的活跃模式**：SSH 进入 → 落 ELF → 立即触发 `su` 提权（Dirty Frag / Copy Fail 共有特征）

### 5.3 容器并不是隔离边界

Page cache 是**宿主全局共享的**。一个容器内的攻击者污染了 `/usr/bin/su` 的 page cache，宿主和其他容器看到的也是被污染的版本 → 这类漏洞**天然就是容器逃逸 / K8s 节点接管原语**。

不要因为"运行在容器内"就降低风险评估。

### 5.4 事件响应要点

怀疑被打过：
1. **立即** `echo 3 > /proc/sys/vm/drop_caches` 把被污染的页驱逐
2. 检查关键文件的 page cache 是否与磁盘一致
3. 检查 `/etc/ld.so.preload` 是否被加了条目
4. 排查 SSH 登录后是否有可疑 `su` 行为（活跃利用的标志）

### 5.5 缓解措施要打**全**，不要只挡一个模块

如果暂时不能打补丁：

```
# /etc/modprobe.d/page-cache-write-prims.conf
install algif_aead /bin/false   # 挡 Copy Fail
install esp4 /bin/false         # 挡 Dirty Frag / Fragnesia
install esp6 /bin/false
install rxrpc /bin/false
```

只挡一类（比如只挡 `algif_aead`）会留下另外几条路。代价是 IPsec/AFS 等功能会不可用，需要业务评估。

### 5.6 长期防御思路

- **缩小攻击面**：seccomp 限制容器/服务能调用的 syscall（禁用 `splice` 或 `AF_ALG` 创建可显著降低风险）
- **MAC 强化**：SELinux/AppArmor 限定只有必要服务能访问 `AF_ALG` socket、IPsec 配置接口；限制即使是 root 进程也不能随便写 `/usr/bin/` 下的文件
- **User namespace 限制**：很多发行版默认开放，导致攻击者能自取 `CAP_NET_ADMIN` → 评估是否禁用 `kernel.unprivileged_userns_clone`
- **缩减 setuid 攻击面**：用户可写挂载点（`/home`、`/tmp`）一律 `nosuid`；持续用 capabilities 替代 setuid，把 `su`/`sudo` 收敛到 `pkexec` + polkit 路径上
- **关注同类新洞**：这类 bug 还会持续出现（同一模式在不同子系统里复现），防御侧的模式识别比 CVE 跟踪更可持续

---

## 附：参考链接

- Wiz —— Dirty Frag 技术拆解: https://www.wiz.io/blog/dirty-frag-linux-kernel-local-privilege-escalation-via-esp-and-rxrpc
- Wiz —— Copy Fail 拆解: https://www.wiz.io/blog/copyfail-cve-2026-31431-linux-privilege-escalation-vulnerability
- Xint —— Copy Fail "732 字节到 root": https://xint.io/blog/copy-fail-linux-distributions
- Microsoft Security Blog —— Dirty Frag 在野利用观察: https://www.microsoft.com/en-us/security/blog/2026/05/08/active-attack-dirty-frag-linux-vulnerability-expands-post-compromise-risk/
- Microsoft Security Blog —— Copy Fail: https://www.microsoft.com/en-us/security/blog/2026/05/01/cve-2026-31431-copy-fail-vulnerability-enables-linux-root-privilege-escalation/
- Tenable —— Fragnesia FAQ: https://www.tenable.com/blog/fragnesia-cve-2026-46300-faq-about-new-linux-kernel-xfrm-esp-in-tcp-priv-esc
- Red Hat —— RHSB-2026-003 Dirty Frag: https://access.redhat.com/security/vulnerabilities/RHSB-2026-003
- Huntress —— 三个漏洞综述: https://www.huntress.com/blog/linux-kernel-flaws-copyfail-dirty-frag-fragnesia
