# System Information Collection
## CPU
```sh
$ lscpu
Architecture:                x86_64
  CPU op-mode(s):            32-bit, 64-bit
  Address sizes:             48 bits physical, 48 bits virtual
  Byte Order:                Little Endian
CPU(s):                      2
  On-line CPU(s) list:       0,1
Vendor ID:                   AuthenticAMD
  Model name:                AMD EPYC 7763 64-Core Processor
    CPU family:              25
    Model:                   1
    Thread(s) per core:      2
    Core(s) per socket:      1
    Socket(s):               1
    Stepping:                1
    BogoMIPS:                4890.85
    Flags:                   fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht s
                             yscall nx mmxext fxsr_opt pdpe1gb rdtscp lm constant_tsc rep_good nopl tsc_reliable nonstop_tsc cpuid
                             extd_apicid aperfmperf tsc_known_freq pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 movbe popcnt aes
                              xsave avx f16c rdrand hypervisor lahf_lm cmp_legacy cr8_legacy abm sse4a misalignsse 3dnowprefetch os
                             vw topoext vmmcall fsgsbase bmi1 avx2 smep bmi2 erms invpcid rdseed adx smap clflushopt clwb sha_ni xs
                             aveopt xsavec xgetbv1 xsaves user_shstk clzero xsaveerptr rdpru arat umip vaes vpclmulqdq rdpid fsrm
Virtualization features:
  Hypervisor vendor:         Microsoft
  Virtualization type:       full
Caches (sum of all):
  L1d:                       32 KiB (1 instance)
  L1i:                       32 KiB (1 instance)
  L2:                        512 KiB (1 instance)
  L3:                        32 MiB (1 instance)
NUMA:
  NUMA node(s):              1
  NUMA node0 CPU(s):         0,1
Vulnerabilities:
  Gather data sampling:      Not affected
  Ghostwrite:                Not affected
  Indirect target selection: Not affected
  Itlb multihit:             Not affected
  L1tf:                      Not affected
  Mds:                       Not affected
  Meltdown:                  Not affected
  Mmio stale data:           Not affected
  Old microcode:             Not affected
  Reg file data sampling:    Not affected
  Retbleed:                  Not affected
  Spec rstack overflow:      Vulnerable: Safe RET, no microcode
  Spec store bypass:         Vulnerable
  Spectre v1:                Mitigation; usercopy/swapgs barriers and __user pointer sanitization
  Spectre v2:                Mitigation; Retpolines; STIBP disabled; RSB filling; PBRSB-eIBRS Not affected; BHI Not affected
  Srbds:                     Not affected
  Tsa:                       Vulnerable: No microcode
  Tsx async abort:           Not affected
  Vmscape:                   Not affected
```
### CPU配置解析
从这份 `lscpu` 看，你这台“服务器”实际上是运行在 **Microsoft Hyper-V / Azure 风格虚拟化环境中的虚拟机**，宿主机 CPU 是 AMD EPYC 7763，但你真正拿到的计算资源只有 **2 个 vCPU**。

几个关键点：

* **CPU 型号**：AMD EPYC 7763，Zen 3，宿主物理 CPU 本身是很强的服务器 CPU，原生是 64 核 / 128 线程。
* **你的 VM**：只有 `CPU(s): 2`，而且显示 `Core(s) per socket: 1`、`Thread(s) per core: 2`，也就是暴露给虚拟机的是 **1 核 2 线程** 的拓扑。
* `Hypervisor vendor: Microsoft` 明确说明你不是裸机。
* 支持 `AVX2 / AES / SHA / SMEP / SMAP / PCID / INVPCID / user_shstk` 等，做一般 Linux 内核、编译、系统安全实验没问题。
* **没有 AVX-512**，这符合 EPYC Milan 的特点。
* L3 显示 `32 MiB`，不是说你独占 32 MiB L3；虚拟机中的 cache topology 不应理解成真实独占资源。
* `48 bits physical, 48 bits virtual` 是虚拟机给 guest 暴露的地址能力。
* 没看到 `svm` flag，所以 **AMD-V 没有透传给你的 VM**。如果你想在这台机器里面再跑 KVM，这一点尤其重要。

所以，如果你问“配置怎么样”，我的评价是：

> **CPU 本身很强，但当前 VM 规格非常低，瓶颈明显是只有 2 vCPU，而不是 CPU 微架构。**

拿来 SSH、编译小项目、Linux 内核调试、小型 QEMU 实验可以。拿来编译大型内核、跑 syzkaller、并行 fuzzing、大型 Rust/C++ build、VM 密集实验，就比较弱了。

尤其你如果准备做 KVM/QEMU，可以先运行：

```bash
grep -E 'vmx|svm' /proc/cpuinfo
```

你这份输出里实际上没有 `svm`，大概率会没有结果。也可以：

```bash
ls -l /dev/kvm
```

如果 `/dev/kvm` 不存在，说明没有 nested virtualization / KVM 可用。

---

## 查看整台服务器配置

我一般建议依次看下面这些。

**1. 内存**

```bash
free -h
```

更详细：

```bash
cat /proc/meminfo
```

如果想看 DIMM 等物理信息：

```bash
sudo dmidecode --type memory
```

不过云 VM 里的 `dmidecode` 信息通常没有裸机那么有意义。

---

**2. 磁盘**

最重要：

```bash
lsblk
```

推荐：

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL
```

看磁盘容量和使用情况：

```bash
df -hT
```

看具体硬盘型号：

```bash
sudo fdisk -l
```

如果是 NVMe：

```bash
sudo nvme list
```

没有 `nvme` 命令可以装：

```bash
sudo apt install nvme-cli
```

---

**3. PCI 设备**

这个对判断网卡、磁盘控制器、GPU、虚拟化设备非常有用：

```bash
lspci
```

详细一点：

```bash
lspci -nnk
```

比如只看：

```bash
lspci | grep -Ei 'vga|3d|network|ethernet|nvme'
```

---

**4. 网卡**

```bash
ip addr
```

```bash
ip link
```

查看网卡驱动：

```bash
sudo ethtool -i eth0
```

查看链路：

```bash
sudo ethtool eth0
```

云服务器显示出来的速率不一定等于实际云厂商限速。

---

**5. 操作系统**

```bash
cat /etc/os-release
```

内核：

```bash
uname -a
```

或者：

```bash
uname -r
```

---

**6. 虚拟化环境**

```bash
systemd-detect-virt
```

你的机器大概率会输出：

```text
microsoft
```

也可以：

```bash
sudo dmidecode -s system-manufacturer
sudo dmidecode -s system-product-name
```

如果这是 Azure VM，通常能看到 Microsoft/Azure 相关信息。

---

**7. NUMA**

虽然你现在只有一个 NUMA node：

```bash
numactl --hardware
```

安装：

```bash
sudo apt install numactl
```

你的 `lscpu` 已经显示：

```text
NUMA node(s): 1
NUMA node0 CPU(s): 0,1
```

所以当前 VM 没有什么 NUMA 调优空间。

---

**8. GPU**

```bash
lspci | grep -Ei 'vga|3d|display'
```

NVIDIA：

```bash
nvidia-smi
```

普通 Azure VM 大概率没有独立 GPU。

---

### 我建议直接跑这组命令

如果你想让我帮你完整判断这台服务器到底是什么规格，可以一次把下面结果贴给我：

```bash
echo '=== CPU ==='
lscpu

echo '=== MEMORY ==='
free -h

echo '=== DISK ==='
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL

echo '=== FILESYSTEM ==='
df -hT

echo '=== OS ==='
cat /etc/os-release
uname -a

echo '=== PCI ==='
lspci -nn

echo '=== NETWORK ==='
ip -br addr

echo '=== VIRTUALIZATION ==='
systemd-detect-virt
ls -l /dev/kvm 2>/dev/null || echo '/dev/kvm does not exist'
```

其中我最关心的是 **`free -h`、`lsblk` 和 `/dev/kvm`**。结合你现在这份 `lscpu`，就基本能判断它适不适合做内核编译、QEMU/KVM 和系统安全实验。

## DISK
```sh
$ lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL
NAME     SIZE TYPE FSTYPE MOUNTPOINTS
                                    MODEL
sda       64G disk                  Virtual Disk
├─sda1    63G part ext4   /
├─sda14    4M part
├─sda15  106M part vfat   /boot/efi
└─sda16  913M part ext4   /boot
sr0      630K rom                   Virtual DVD-ROM
```

### DISK配置解析
这份 `lsblk` 输出说明你的虚拟机目前有一块 **64 GB 虚拟磁盘**，外加一个虚拟光驱。系统根分区占了大约 63 GB。

命令：

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL
```

各列含义是：

* `NAME`：块设备名称。
* `SIZE`：容量。
* `TYPE`：设备类型，`disk` 是整块磁盘，`part` 是分区，`rom` 是只读光驱。
* `FSTYPE`：文件系统类型，例如 `ext4`、`vfat`。
* `MOUNTPOINTS`：该分区挂载到 Linux 文件系统的哪里。
* `MODEL`：设备型号。

你的磁盘结构可以画成：

```text
sda  64G  Virtual Disk
│
├── sda1    63G   ext4   /
├── sda14    4M
├── sda15  106M   vfat   /boot/efi
└── sda16  913M   ext4   /boot

sr0  630K  Virtual DVD-ROM
```

### `sda`：64 GB 虚拟硬盘

```text
sda       64G disk                  Virtual Disk
```

`/dev/sda` 是整块磁盘。

因为型号显示：

```text
Virtual Disk
```

再次说明这不是物理硬盘，而是 Hyper-V 给虚拟机暴露的一块虚拟磁盘。实际底层可能是 Azure 的远程块存储、SSD 等，从 guest 内部单凭这里看不出来。

---

### `sda1`：你的主分区

```text
sda1    63G part ext4   /
```

这是最重要的分区：

* 大小：约 63 GB
* 文件系统：`ext4`
* 挂载点：`/`

也就是说：

```text
/
├── home
├── usr
├── var
├── opt
├── tmp
...
```

这些目录基本都存在这一块 `sda1` 上。

比如：

```bash
df -h /
```

就可以看到这个分区实际用了多少空间。

例如可能显示：

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        62G   15G   44G  26% /
```

注意 `lsblk` 的 `63G` 是**分区大小**，不是“剩余空间”。

---

### `sda15`：EFI System Partition

```text
sda15  106M part vfat   /boot/efi
```

这是 **EFI System Partition，ESP**。

它使用 FAT 文件系统：

```text
FSTYPE = vfat
```

挂载在：

```text
/boot/efi
```

UEFI 固件启动 Linux 时，会从这个分区读取 EFI bootloader，例如：

```text
/boot/efi/EFI/
```

里面可能有：

```text
ubuntu/
BOOT/
```

之类的目录。

所以启动链大致是：

```text
Hyper-V UEFI firmware
        ↓
EFI System Partition
/boot/efi
        ↓
GRUB / shim
        ↓
Linux kernel
```

---

### `sda16`：独立 `/boot`

```text
sda16  913M part ext4   /boot
```

这是独立的 `/boot` 分区，大约 913 MB。

里面主要保存：

```text
vmlinuz-*
initrd.img-*
System.map-*
config-*
grub/
```

例如：

```bash
ls -lh /boot
```

可能会看到：

```text
vmlinuz-6.x.x-xx-generic
initrd.img-6.x.x-xx-generic
config-6.x.x-xx-generic
System.map-6.x.x-xx-generic
```

所以这里涉及的是 Linux kernel 和 initramfs，而普通系统文件放在 `/dev/sda1`。

---

### `sda14`：4 MB 小分区

```text
sda14    4M part
```

这个分区：

* 只有 4 MB；
* 没有文件系统；
* 没有挂载。

这种小分区通常是云镜像在磁盘初始化/引导布局中预留的分区，不是你的数据盘。

无需处理，更不要因为它“没挂载”就删除。

---

### `sr0`：虚拟光驱

```text
sr0      630K rom                   Virtual DVD-ROM
```

`sr0` 就是：

```text
/dev/sr0
```

它是 Hyper-V 提供的虚拟 CD/DVD-ROM。

`TYPE=rom` 表示只读设备。

云服务器有时会通过虚拟 CD-ROM 注入：

* provisioning 信息；
* cloud-init 数据；
* VM metadata；
* 初始化配置。

630 KB 非常小，所以显然不是实际的安装 DVD。

---

### 你的磁盘整体情况

目前可以概括为：

```text
VM
├── 2 vCPU
└── 64 GB Virtual Disk
    ├── ~63 GB root filesystem
    ├── ~913 MB /boot
    ├── ~106 MB EFI
    └── small reserved partition
```

这是非常典型的云 Linux VM 系统盘布局。

不过这里还不能回答两个比较重要的问题：

**第一，64 GB 还剩多少可用空间。**

运行：

```bash
df -hT
```

**第二，这块“Virtual Disk”性能怎么样。**

`lsblk` 看不出它底层到底对应 Premium SSD、Standard SSD 还是其他类型，也看不出 IOPS/吞吐上限。如果这是 Azure VM，这些往往由 **VM SKU + Managed Disk 类型**共同决定。

另外，你现在只有这一块 `sda`，没有类似：

```text
sdb
nvme0n1
```

因此从当前输出看，应该**没有额外挂载的数据盘**。
