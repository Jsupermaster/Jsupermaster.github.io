---
title: "把 XRT 2024.2 移植到 Linux 6.17：Alveo U50 内核适配九处 API 变更处理"
date: 2026-07-02 12:00:00 +0800
description: 记录在 Ubuntu 24.04 + Linux 6.17 上把 XRT 2024.2 的 DKMS 内核模块（xocl / xclmgmt）编过并跑通 Alveo U50 的完整过程，覆盖九处内核 API 变更以及一个只在冷启动后复现的 EINVAL 运行时问题的定位。
categories:
  - FPGA
home_category: engineering-practice
series: alveo-development
tags:
  - FPGA
  - Alveo
  - XRT
  - Linux Kernel
  - DKMS
  - Debugging
article_kicker: FPGA NOTE
cover_image: /assets/images/alveo_u50/image-20260701223955214.png
word_count: 约 12k 字
read_time: 约 40 分钟
---

![Alveo U50 XRT kernel 6.17 cover](/assets/images/alveo_u50/image-20260701223955214.png)

## 一、前言

前段时间我在闲鱼上淘到一张 Alveo U50C。相较官方板卡，它只是缺少 QSFP 光口，芯片和 shell 与官方 U50 通用，可以直接使用 U50 的官方工具链。关于 Alveo 的整体开发流程可以参考我上一篇博客：<https://jsupermaster.github.io/posts/alveo-u50-software-stack.html>，其中的核心是安装 XRT 等 Xilinx 官方驱动和运行时。

我的主机是 Ubuntu 24.04.4，内核 `6.17.0-35-generic`。按官方文档，安装 XRT 2024.2 本应是一个非常短的流程：下载 deb、`apt install`、`xbutil validate`。

实际情况是，官方 24.04 版本的 XRT deb 装完之后会触发 DKMS 编译 `xocl` 与 `xclmgmt`，直接遇到一批 `implicit declaration` 编译错误。原因在于 AMD 打包时基于 Ubuntu 24.04 的默认内核 6.8，而从 6.10 到 6.17 的一系列内核 API 变更并没有在这套 DKMS 源码里跟上。GitHub issue [#8783](https://github.com/Xilinx/XRT/issues/8783) 中有人反馈过类似问题，AMD 官方的回复大致是"请切换回内核 6.8"。

但我并不希望为了驱动一张加速卡而反复切换主机内核，所以决定自己去修 DKMS 源。

**目标：主机环境保持不变，把 XRT 2024.2 在内核 6.17 上编过并跑通。**

---

## 二、快速开始（TL;DR）

如果只想在 Ubuntu 24.04 + Linux 6.17 上把 U50 跑通，跟着下面六步走即可，全程大约十分钟。技术细节、九处内核 API 变更是如何处理的，以及 `FOP_UNSIGNED_OFFSET` 这个运行时问题是怎么定位的，都放在后面的章节。

**前置检查**

- Ubuntu 24.04.x，内核在 `6.10` – `6.17` 之间均已验证（旧内核用 AMD 官方 deb 可直接安装，不需要本仓库）
- 已安装 `linux-headers-$(uname -r)`
- 磁盘空闲 ≥ 2 GB
- 具备 sudo 权限，可联网

### 1. 从 AMD 官网下载三个安装包

打开 **[Alveo U50 Downloads → Vitis 2024.2 tab](https://www.amd.com/en/support/downloads/alveo-previous-downloads.html/accelerators/alveo/u50.html#alveotabs-item-vitis-tab)**，页面上只有三个下载项，全部下载：

| 分类 | 文件名 | 大小 | 用途 |
|---|---|---|---|
| **Xilinx Runtime (XRT)** | `xrt_202420.2.18.179_24.04-amd64-xrt.deb` | 17.90 MB | 用户态运行时 + DKMS 内核模块源码 |
| **Deployment Target Platform** | `xilinx-u50-gen3x16-xdma_2024.1_2024_0522_2343-all.deb.tar.gz` | 33.95 MB | shell 位流 + validate xclbin + CMC / SC firmware（tar.gz 内含 4 个 deb） |
| **Development Target Platform**（可选） | `xilinx-u50-gen3x16-xdma-5-202210-1-dev_1-3499627_all.deb` | 159.02 MB | 使用 Vitis 编译 xclbin 时才需要 |

### 2. 安装 XRT 并通过补丁修复 DKMS

先把 XRT deb 放到本仓库的 `downloads/` 目录，然后运行一键脚本：

```bash
git clone https://github.com/Jsupermaster/xrt_ubuntu24_alveou50.git
cd xrt_ubuntu24_alveou50
mv ~/Downloads/xrt_202420.2.18.179_24.04-amd64-xrt.deb ./downloads/
./install.sh    # 默认 --mode fast
```

`install.sh` 内部完成的工作是：先 `apt install` 官方 XRT deb（用户态安装成功、DKMS 编译 `xocl` / `xclmgmt` 失败是预期结果），随后把 `xrt-overlay/` 下 11 个已修复的源文件覆盖到 `/usr/src/xrt-2.18.0/driver/` 对应位置（源码布局和 DKMS 扁平化布局不完全一致，脚本内做了路径映射），最后执行 `dkms remove xrt/2.18.0 --all && dkms install xrt/2.18.0` 重新编译。

执行完成后可以验证一下：

```bash
dkms status | grep xrt
# 期望：xrt/2.18.0, 6.17.0-35-generic, x86_64: installed
```

如果希望完全从 XRT 源码整体重编，而不是直接使用官方 deb，可以运行 `./install.sh --mode source`，耗时约 30 分钟，详见第三章。

### 3. 安装 Platform 与 Firmware deb

Deployment tar.gz 里已经打包了 base、validate、CMC、SC 共四个 deb。解压后一起安装即可：

```bash
tar xzf xilinx-u50-gen3x16-xdma_2024.1_2024_0522_2343-all.deb.tar.gz
sudo apt install ./xilinx-cmc-u50_*.deb \
                 ./xilinx-sc-fw-u50_*.deb \
                 ./xilinx-u50-gen3x16-xdma-base_5-*.deb \
                 ./xilinx-u50-gen3x16-xdma-validate_5-*.deb

# 如需要使用 Vitis 编译 xclbin，可以再安装 dev 平台包
sudo apt install ./xilinx-u50-gen3x16-xdma-5-202210-1-dev_1-3499627_all.deb
```

### 4. 物理安装板卡并冷启动

把 U50 插入 PCIe 插槽（或 USB4 / Thunderbolt PCIe 扩展坞），**完全断电**约 10 秒后再上电（不是 `reboot`）。

```bash
lspci -d 10ee:                              # 应该看到两个 function：mgmt PF + user PF
sudo /opt/xilinx/xrt/bin/xbmgmt examine     # 记下这里显示的 <bdf>，例如 0000:2f:00.0
```

### 5. 烧写 shell（第一次装卡必做）

```bash
sudo /opt/xilinx/xrt/bin/xbmgmt program --base --device 0000:2f:00.0
```

烧写完成后**必须再进行一次完全断电**。SC 只有在 12V 电源掉电时才会 latch 新的 shell metadata，`reboot` 不会切断电源，SC 状态不会更新，flash 就等同于没有生效。这个环节第一次操作时很容易忽略，一定要做物理断电。

### 6. validate

```bash
source /opt/xilinx/xrt/setup.sh
sudo xbutil validate --device 0000:2f:00.1
```

期望输出 6/6 PASSED：

```
Test 1 [0000:2f:00.1]  : aux-connection    PASSED
Test 2 [0000:2f:00.1]  : pcie-link         PASSED  （可能出现 warning，属正常）
Test 3 [0000:2f:00.1]  : sc-version        PASSED
Test 4 [0000:2f:00.1]  : verify            PASSED
Test 5 [0000:2f:00.1]  : dma               PASSED
Test 6 [0000:2f:00.1]  : iops              PASSED
```

`pcie-link` 报 warning 通常意味着链路协商没有达到板卡的 x16 上限（例如通过 USB4 扩展坞只能协商到 x4），并不影响后续 Vitis 使用。

到这里 U50 已经可以正常工作，可以执行 `xbutil examine` 查看板卡信息，或运行 Vitis Accel Examples，或自行开发 xclbin。想了解为什么需要这套 overlay、每个内核 API 变更是如何处理的，以及那个冷启动问题是如何定位的，可以继续往下读。

---

## 三、软件栈与自编 XRT 的选择

### 3.1 软件栈简图

Alveo 的软件栈大致可以分成四层：Vitis（host 编译器）→ **XRT 用户态 + XRT 内核模块** → Platform（shell 位流） → Firmware（CMC + SC）。本文关注的问题只出现在中间的 XRT 两层，也就是用户态和内核模块。Platform 和 Firmware 都直接使用 AMD 官方 deb，未做任何修改（安装方式见第二章）。

关于四层各自的职责，以及 `xclbin` / `xbutil` / `xbmgmt` 分别是什么，可以先阅读我上一篇 [《Alveo 软件栈与安装组成梳理》](https://jsupermaster.github.io/posts/alveo-u50-software-stack.html)，这里不再重复。

需要单独强调的是 XRT 内核模块有两个：

- **`xocl`**：挂在 user PF（`0000:2f:00.1`）上，负责用户空间对 kernel 的调度、BO 分配、xclbin 加载。它基于 DRM 框架，字符设备为 `/dev/dri/renderD128`。
- **`xclmgmt`**：挂在 mgmt PF（`0000:2f:00.0`）上，负责 flash shell、读取传感器以及与 SC 通信。字符设备为 `/dev/xclmgmt*`。

两个模块都由 DKMS 从 `/usr/src/xrt-2.18.0/` 编译得到，在安装 `xrt-*-xrt.deb` 时被 postinst 触发。**本文涉及的九处修改，全部集中在这两个模块的源码中。**

### 3.2 官方 24.04 deb 装完为什么还是不行

AMD [Alveo U50 下载页](https://www.amd.com/en/support/downloads/alveo-previous-downloads.html/accelerators/alveo/u50.html#alveotabs-item-vitis-tab) 的 Vitis 2024.2 tab 目前提供 `xrt_202420.2.18.179_24.04-amd64-xrt.deb`（约 18 MB）。安装它时会发生两件事：

- **用户态部分**（`/opt/xilinx/xrt/` 下的 `xbutil`、`xbmgmt`、`libxrt_core` 等）可以正常安装。
- **DKMS 部分**（`/usr/src/xrt-2.18.0/`）postinst 会立即触发 `dkms install xrt/2.18.0`，在编译 `xocl` / `xclmgmt` 时失败。

失败原因就是后续要展开的九处 API 变更。AMD 官方 deb 中的 DKMS 源码是按 Ubuntu 24.04 的默认内核（发布时为 6.8）打包的，从 6.10 到 6.17 的一系列 API 变更并未同步跟进。对着 6.17 内核运行 `dkms install`，会看到和直接编译 XRT 源码一模一样的报错，例如 `implicit declaration of 'iommu_present'`、`implicit declaration of 'vzalloc'` 等。

因此，本仓库真正修复的是**内核 API 兼容层**，与 XRT 用户态是否重编无关。基于这一点，有两种安装路径可选。

**Fast 路径（推荐，几分钟）**：

1. 安装 AMD 官方 XRT deb（用户态成功，DKMS 失败但不影响后续步骤）。
2. 用本仓库 `xrt-overlay/` 覆盖 `/usr/src/xrt-2.18.0/` 下的 11 个文件。
3. 执行 `dkms remove xrt/2.18.0 --all && dkms install xrt/2.18.0` 重新编译。

`./install.sh` 默认走这条路径，第二章的命令即为这一流程。

**Source 路径（约 30 分钟）**：

1. `git clone` XRT 的 tag `202420.2.18.179`。
2. 用同一套 overlay 覆盖到源码树上。
3. 依次执行 `xrtdeps.sh`、`build.sh -opt`、`make package`。
4. 安装自编的 deb，DKMS 从已打好补丁的源码中编译。

`./install.sh --mode source` 走这条路径，适合需要同时修改 userspace 代码，或希望从源头验证整条流水线的场景。

两种路径的版本三元组是相同的：

| 组件 | 选定版本 | 备注 |
|---|---|---|
| Vitis / Vivado | 2024.2 | 已安装 |
| XRT | tag `202420.2.18.179` | Official 2024.2 |
| U50 Platform | `xilinx_u50_gen3x16_xdma_5_202210_1` | 冻结在 2022.1，U50 已进入 maintenance mode |

需要特别提示的是，Platform 的 `5_202210_1` 是**平台版本号**，不是 XRT 版本号。这两个数字容易混淆，特此说明。

### 3.3 这套补丁是怎么写出来的

Fast 路径听起来简单，但**这些补丁本身是如何写出来的**才是本文重点。当初做这件事时，我走的是 source 路径，逐个问题定位、逐个修复。跟着 XRT 源码 clone、check out `202420.2.18.179`：

```bash
git clone https://github.com/Xilinx/XRT.git
cd XRT
git checkout 202420.2.18.179
git submodule update --init --recursive
```

安装编译依赖（这一步会 apt install 上百个包，网络较慢时需要耐心等待）：

```bash
sudo ./src/runtime_src/tools/scripts/xrtdeps.sh
```

编译用户态并打 deb：

```bash
cd build
./build.sh -opt
cd Release
make package
```

`build.sh -opt` 用户态可以顺利跑到 100%，`make package` 会生成 8 个 deb（xrt 主包、aws / azure / container 变体、xbflash 工具等）。安装 XRT：

```bash
sudo apt install ./xrt_*-xrt.deb
```

apt 前段过程顺利，直到最后触发 DKMS 编译 `xocl.ko` / `xclmgmt.ko` 时，编译器开始输出连续的错误信息。下一章就是这九处修改的完整记录。

---

## 四、内核 API 变更处理

以下是九处由内核 API 变更导致的编译问题，以及对应的修复过程。所有源码和补丁都已上传 GitHub，不想读细节的可以直接按 README 安装。

代码路径全部位于 XRT 源码树的 `src/runtime_src/core/pcie/driver/linux/xocl/` 下，下文简写为 `xocl/`。

### 4.1 kernel 6.8：`iommu_present()` 被移除

DKMS make.log 中的第一条报错：

```
xocl/subdev/p2p.c: In function 'p2p_probe':
error: implicit declaration of function 'iommu_present';
```

Mainline 中 `iommu_present()` 在 6.8 被移除，一并被移除的还有 `iommu_domain_type_str` 等。移除相关的 commit 是 `b869e18a1160`（Robin Murphy 的 "iommu: Retire bus ops"），在此之前 `iommu_present` 已经被标记为弃用一段时间。

Robin 的动机很直接：`iommu_present(bus)` 是 per-bus 的粗粒度检查，回答的是"这条总线上是否存在 IOMMU"；而实际业务几乎都需要的是 per-device 的"这个设备是否被 IOMMU map 了"。因此推荐的替代函数是 `device_iommu_mapped(&pdev->dev)`。

XRT 中 `p2p.c` 原本的代码是：

```c
if (iommu_present(&pci_bus_type)) {
    /* P2P 走的是物理地址，被 IOMMU 拦截时无法使用 */
    return -EOPNOTSUPP;
}
```

改为：

```c
if (device_iommu_mapped(&pcidev->dev)) {
    return -EOPNOTSUPP;
}
```

顺带把语义从"总线上有 IOMMU"精化到"这块 U50 被 IOMMU map 了"，反而更贴近本意。

### 4.2 kernel 6.10：`I2C_CLASS_SPD` + UART `circ_buf` 迁移到 kfifo

这一版内核带来了两处需要处理的变更。

**变更 1：`I2C_CLASS_SPD` 移除**（`xocl/subdev/xiic.c`）

Wolfram Sang 主导的 i2c 子系统清理，把老式 SPD（DIMM EEPROM）类别整体移除。相关 commit 标题类似 "i2c: remove I2C_CLASS_SPD support"，落在 6.10。

XRT 的 xiic 从未对接过 SPD 器件，这个 class flag 是从模板复制过来的历史遗留。加一个版本守卫直接去掉即可：

```c
adap->class =
#if LINUX_VERSION_CODE < KERNEL_VERSION(6, 10, 0)
    I2C_CLASS_HWMON | I2C_CLASS_SPD;
#else
    I2C_CLASS_HWMON;
#endif
```

不会带来任何功能影响。

**变更 2：UART tx 缓冲从 `struct circ_buf` 迁移到 kfifo**（`xocl/subdev/ulite.c`）

Jiri Slaby 的一系列 serial 子系统重构，把 `uart_state::xmit` 从 `circ_buf` 换成了 kfifo，配套 helper 全部相应改变：

| 旧（≤ 6.9） | 新（≥ 6.10） |
|---|---|
| `port->state->xmit` (`struct circ_buf`) | `&tport->xmit_fifo` (kfifo) |
| `uart_circ_empty(xmit)` | `kfifo_is_empty(&tport->xmit_fifo)` |
| `uart_circ_chars_pending(xmit)` | `kfifo_len(&tport->xmit_fifo)` |
| `xmit->buf[xmit->tail]; xmit->tail = (xmit->tail + 1) & (UART_XMIT_SIZE - 1);` | `uart_fifo_out(port, &ch, 1)` |

`ulite_transmit()` 中原本有 20 多行手写的 `circ_buf` 环形队列操作，改用 kfifo 之后大约只剩 6 行。这是本次涉及的所有修改中改动量最大的一处，也是唯一一处把函数体重写了的地方，其余基本都是宏或单行修补。

代码较长不完整贴出，思路是把整个函数拆成 `#if LINUX_VERSION_CODE < KERNEL_VERSION(6, 10, 0)` 与 `#else` 两个分支。

### 4.3 kernel 6.11：`platform_driver::remove` 从 int 改为 void

这是九处修改中最偏"设计哲学"的一处。

Uwe Kleine-König 从 2023 年开始推动 `remove` callback 改为返回 void。他的理由很直接：`platform_driver::remove` 返回 int 已经存在十几年，但内核代码路径**从来不会检查这个返回值**（remove 的语义就是"必须成功"）。既然是虚假承诺，不如干脆改成 void。

推进节奏：

1. 6.7：新增 `remove_new` 字段（返回 void），与老的 `remove`（返回 int）并存，逐步迁移各驱动。相关 commit 是 `0edb555a65d1` "platform: Provide a remove callback that returns no value"。
2. 6.7 到 6.11 之间：mainline 中所有驱动被一批批切换到 `remove_new`。
3. 6.11：删除老的 `remove`（返回 int），把 `remove_new` 改名回 `remove`（返回 void）。

XRT 中 platform_driver 的数量：

```bash
$ grep -r "\.remove\s*=" src/runtime_src/core/pcie/driver/linux/xocl/ | wc -l
61
```

61 处调用点，每一处的结构都类似：

```c
static int xocl_xdma_remove(struct platform_device *pdev)
{
    /* ... 40 行清理逻辑 ... */
    return 0;
}

static struct platform_driver xocl_xdma_driver = {
    .probe    = xocl_xdma_probe,
    .remove   = xocl_xdma_remove,     // ← 类型冲突
    /* ... */
};
```

处理方式有两种：

- **方案 A**：修改 61 个函数签名，并去掉 61 处 `return 0`。
- **方案 B**：让 gcc 知道这个类型不匹配是可接受的，不作为错误处理。

XRT 现有的 `return 0` 之前都没有基于返回值做任何清理判断，直接改成 void 语义完全等价，方案 A 的实际收益为零。方案 B 只需要一行改动：

```makefile
# xocl/userpf/Makefile 与 xocl/mgmtpf/Makefile 各加一行
ccflags-y += -Wno-error=incompatible-pointer-types
```

写博客时我特意查证了一下这条 warning 会不会掩盖其他潜在问题——它只影响函数指针赋值时的原型不匹配检查，其他类型问题该报仍然会报。这个取舍在本场景中是合理的。

### 4.4 kernel 6.12：`no_llseek` 被移除

Christoph Hellwig 的一次头文件收敛，`no_llseek` 从 6.12 起被彻底移除（此前已经被 `__deprecated` 标记两年）。

XRT 中有几个 debug fops 用到它：

```c
static const struct file_operations debug_fops = {
    .owner   = THIS_MODULE,
    .llseek  = no_llseek,     // ← 找不到符号
    /* ... */
};
```

`no_llseek` 的语义是"这个 fd 不支持 seek，返回 -ESPIPE"，`noop_llseek` 的语义是"seek 什么都不做，返回 0"。在 XRT 这几个 debug 端点中根本不会发生 seek，返回 0 还是 -ESPIPE 都不影响功能。一行宏 alias 即可解决：

```c
/* xocl/xocl_drv.h */
#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 12, 0)
#define no_llseek noop_llseek
#endif
```

### 4.5 kernel 6.13：`crc32c_le` 改名 + `MODULE_IMPORT_NS` 字符串化

两处独立的小变更，都是纯签名调整。

**改名 `crc32c_le` → `crc32c`**：Eric Biggers 的 crypto lib 清理。`crc32c_le` 意为"小端 CRC32C"，但内核中从未存在过大端版本，`_le` 后缀属于伪对称，去掉更清爽。

XRT 中只调用过一次 `crc32c_le`，加一个 alias 即可：

```c
/* xocl/xocl_drv.h */
#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 13, 0)
#define crc32c_le(seed, data, len) crc32c((seed), (data), (len))
#endif
```

**`MODULE_IMPORT_NS(NS)` 参数改为 string literal**：以前使用 `MODULE_IMPORT_NS(DMA_BUF)`，是 identifier 形式；6.13 起要求写成 `MODULE_IMPORT_NS("DMA_BUF")`。这是因为 symbol namespace 的 metadata 直接进 `.modinfo`，使用字符串形式更加规整。

XRT 中有两处调用（`xocl_drv.c`, `libxdma.c`），都需要加版本守卫：

```c
#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 13, 0)
MODULE_IMPORT_NS("DMA_BUF");
#else
MODULE_IMPORT_NS(DMA_BUF);
#endif
```

### 4.6 kernel 6.15：timer API 术语统一

Ingo Molnar 主导的一次 timer 子系统术语统一，涉及三个宏 / 函数：

| 旧 | 新（6.15+） |
|---|---|
| `from_timer(var, t, field)` | `timer_container_of(var, t, field)` |
| `del_timer(t)` | `timer_delete(t)` |
| `del_timer_sync(t)` | `timer_delete_sync(t)` |

术语上更加统一（对齐 `container_of` 家族和 `_delete` 前缀），语义完全等价。XRT 中相关调用分布在三个头文件（`xocl_drv.h`、`xrt_cu.h`、`qdma_compat.h`），全部通过 compat 宏收敛，调用点无须改动：

```c
#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 15, 0)
#define from_timer(var, callback_timer, timer_fieldname) \
    timer_container_of(var, callback_timer, timer_fieldname)
#define del_timer(t)      timer_delete(t)
#define del_timer_sync(t) timer_delete_sync(t)
#endif
```

其中 `qdma_compat.h` 是 QDMA IP 驱动自带的兼容层（XRT 通过 submodule 引入），需要重复一份宏定义，因为它作为一个"内嵌 out-of-tree 驱动"独立编译。

### 4.7 kernel 6.16：`<linux/pfn_t.h>` 移除 + `drm_driver::date` 移除

两处修改都在 `xocl/userpf/xocl_drm.c`。

**`pfn_t` 类型消失**。David Hildenbrand 主导的 memory management 清理。`pfn_t` 本质上是对 `unsigned long` 的一层薄封装，六年前引入是想为其增加区分内存类型的标签位（DEV / MAP / WC），但生态并未跟上，反而引入了新的 bug，最终决定移除，所有调用点回到裸 `unsigned long`。

XRT 在 `xocl_bo_vm_ops` 中有两处调用 `phys_to_pfn_t`，若严格跟随 mainline 修改，需要把 `pfn_t` 全部替换为 `unsigned long`，函数签名沿调用链一路传导，工作量较大且没有实质收益。

我选择的方式是**本地 emulate**。既然 `pfn_t` 就是 `unsigned long`，那么在 XRT 头文件中重新造一个：

```c
/* xocl/userpf/xocl_drm.c */
#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 16, 0)
typedef unsigned long pfn_t;
#define phys_to_pfn_t(phys, flags) ((phys) >> PAGE_SHIFT)
#else
#include <linux/pfn_t.h>
#endif
```

函数体一处都无需改动。

**`struct drm_driver::date` 字段移除**。DRM 子系统一直保留 `date` 字段供用户空间探测驱动版本使用，但实际上没有被使用。Simon Ser 直接将其移除。加一个 `#if` 守卫即可：

```c
static struct drm_driver mm_drm_driver = {
    .driver_features = DRIVER_GEM | DRIVER_RENDER,
    /* ... */
    .name = "xocl",
    .desc = XOCL_DRIVER_DESC,
#if LINUX_VERSION_CODE < KERNEL_VERSION(6, 16, 0)
    .date = driver_date,
#endif
    .major = XOCL_DRIVER_MAJOR,
    /* ... */
};
```

### 4.8 kernel 6.17：`<linux/vmalloc.h>` 不再被传递性 include

编译报错：

```
xocl/xocl_drv.h:465: error: implicit declaration of 'vzalloc'
xocl/xocl_subdev.c:272: error: implicit declaration of 'vfree'
xocl/xocl_subdev.c:2208: error: implicit declaration of 'vmalloc'
```

XRT 一直没有直接 include `<linux/vmalloc.h>`，而是通过 `<drm/*.h>` 或 `<linux/cdev.h>` 传递性带入。6.17 里头文件依赖被收敛，`<linux/vmalloc.h>` 不再被自动引入。

补一行即可：

```c
/* xocl/xocl_drv.h */
#include <linux/vmalloc.h>
```

问题本身很短，但它是一个明确的信号：**out-of-tree 驱动会定期受到内核头文件传递依赖收敛的影响**。以后遇到 `implicit declaration of 'xxx'` 类报错，第一反应应当是先查明它所属的头文件，然后直接 include。

### 4.9 kernel 6.17：`drm_open_helper` 强制要求 `FOP_UNSIGNED_OFFSET`

这一处放在下一章单独展开，因为它不是编译错误，而是运行时错误，而且是**只在冷启动后复现**的运行时错误。

到 4.8 结束，DKMS install 已经可以通过：

```bash
$ dkms status
xrt/2.18.0, 6.17.0-35-generic, x86_64: installed
```

编译层面的九处 API 变更处理到此结束。

---

## 五、冷启动后 user PF 打不开（EINVAL）

### 5.1 症状

执行 `xbutil examine`，正常情况下应该列出板卡并报告状态。实际输出却是：

```
$ sudo /opt/xilinx/xrt/bin/xbutil examine
System Configuration
  OS Name              : Linux
  Kernel Version       : 6.17.0-35-generic
  ...
Devices present
[0000:2f:00.1] : Device is not ready
```

"Device is not ready" 是 XRT 中一个通用错误。dmesg 中没有 xocl 相关的 error，模块状态也是已加载：

```
$ lsmod | grep -E 'xocl|xclmgmt'
xocl                 1245184  0
xclmgmt              229376   0
```

热重启（`sudo reboot`）之后，`xbutil examine` 一切正常；**再一次进行拔电冷启动，问题又出现**。

同一个问题在热重启和冷启动之间行为不一致，这种现象通常只在硬件故障场景下才会遇到。

### 5.2 定位链路

**第一层：strace**

`xbutil` 是 C++ 用户态程序，通过 `libxrt_core` 走 ioctl 与 xocl 通信。用 strace 缩小范围：

```bash
sudo strace -f -e trace=openat,ioctl,close -o /tmp/xbutil.strace \
    /opt/xilinx/xrt/bin/xbutil examine
```

trace 中最后一次 openat 是这样的：

```
openat(AT_FDCWD, "/dev/dri/renderD128", O_RDWR|O_CLOEXEC) = -1 EINVAL (Invalid argument)
```

`renderD128` 就是 xocl 的 DRM render node，XRT 所有 user PF 操作都需要先打开这个 fd。**它在 open 阶段就返回了 EINVAL**，ioctl 根本没有被触发。

初看这个错误，我一度以为是 fd 泄露或权限问题。`ls -la /dev/dri/renderD128` 显示 `crw-rw-rw-`，权限正常；单进程干净启动同样返回该错误。

**第二层：dmesg**

```bash
sudo dmesg | grep -Ei 'drm|xocl'
```

xocl probe 一切正常，DRM 子系统一切正常。但仔细往前翻，找到一条 WARN：

```
[   45.231891] ------------[ cut here ]------------
[   45.231893] WARNING: CPU: 3 PID: 4102 at drivers/gpu/drm/drm_file.c:243 drm_open_helper+0x14e/0x1c0
```

位置是 `drivers/gpu/drm/drm_file.c:243`。这个位置在 6.17 mainline 上是什么？

**第三层：内核源码**

翻 mainline `drivers/gpu/drm/drm_file.c` 的 `drm_open_helper`：

```c
static int drm_open_helper(struct file *filp, struct drm_minor *minor)
{
    struct drm_device *dev = minor->dev;
    struct drm_file *priv;
    int ret;

    if (filp->f_flags & O_EXCL)
        return -EBUSY; /* No exclusive opens */
    if (!drm_cpu_valid())
        return -EINVAL;

    if (WARN_ON_ONCE(!(filp->f_op->fop_flags & FOP_UNSIGNED_OFFSET)))
        return -EINVAL;                                            // ← 这里
    /* ... */
}
```

非常清楚：`WARN` + `return -EINVAL`。触发条件是**`filp->f_op->fop_flags` 中没有设置 `FOP_UNSIGNED_OFFSET` 位**。

那么 xocl 有没有设置这个位？

**第四层：bpftrace**

不想为了加 printk 重新编译内核，直接用 bpftrace 读取运行时字段：

```bash
# 挂在 drm_open_helper 入口，dump fop_flags
sudo bpftrace -e '
    kprobe:drm_open_helper {
        $filp = (struct file *)arg0;
        printf("fop_flags = 0x%x, comm=%s\n", $filp->f_op->fop_flags, comm);
    }
'
```

在另一个终端运行 `xbutil examine`，bpftrace 输出：

```
fop_flags = 0x0, comm=xbutil
```

**`fop_flags` 是 0**。`FOP_UNSIGNED_OFFSET` 定义为 `(1 << 5)`，没有被设置。

### 5.3 根因

查看 xocl 源码 `xocl/userpf/xocl_drm.c`：

```c
static const struct file_operations xocl_driver_fops = {
    .owner          = THIS_MODULE,
    .open           = xocl_drm_open,
    .release        = xocl_drm_release,
    .unlocked_ioctl = drm_ioctl,
    .compat_ioctl   = drm_compat_ioctl,
    .poll           = drm_poll,
    .read           = drm_read,
    .llseek         = noop_llseek,
    .mmap           = xocl_bo_mmap,
    /* 没有 .fop_flags */
};
```

对比常规 DRM 驱动使用的宏 `DEFINE_DRM_GEM_FOPS`：

```c
/* include/drm/drm_gem.h */
#define DEFINE_DRM_GEM_FOPS(name) \
    static const struct file_operations name = { \
        .owner       = THIS_MODULE, \
        .fop_flags   = FOP_UNSIGNED_OFFSET, \
        /* ... */ \
    }
```

可以看到 `FOP_UNSIGNED_OFFSET` 是**这个宏自动设置的**。xocl 因为需要提供自己的 `.open`、`.release`、`.mmap`，没有使用这个宏，而是手写了全部字段，恰好漏掉了这一位。

**这个字段是 kernel 6.12 引入的**，作者 Kent Overstreet 在 commit 中说明它是为 DRM / IOMMU 这类使用 `mmap` 大 offset 的驱动做的兼容性标记，目的是避免 `loff_t` 作为 signed 时溢出。6.12 引入时，`drm_open_helper` 中只有 `WARN`，不 return，不影响功能。

**6.17 把它升级为硬拦截**。commit 的原话大意是"warn 已经挂了三个 release cycle 无人修复，是时候强制约束了"。

至于为什么冷启动才复现、热重启不复现，这里需要说明的是，**我并没有查到确定性的答案**。较为合理的推测是：warm reboot 之后某些 kernel object 的初始化路径走了 fast path，跳过了这次检查（`drm_open_helper` 内部对已经 open 过的 device 有一些短路径）；cold boot 则走完整初始化。但这只是推测，能稳定重现是唯一可靠的事实，走完整路径这一假设是保守的。**无论冷热，这个问题本质都是"驱动配置缺失"，热重启没有触发只是巧合，不代表"配置正确"。**

### 5.4 修复

一行代码即可：

```c
static const struct file_operations xocl_driver_fops = {
    .owner          = THIS_MODULE,
    .open           = xocl_drm_open,
    /* ... 其他字段 ... */
#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 12, 0)
    .fop_flags      = FOP_UNSIGNED_OFFSET,
#endif
};
```

执行 `dkms remove xrt/2.18.0 --all && dkms install xrt/2.18.0`，冷启动后再运行 `xbutil examine`，板卡正常显示。

---

## 六、板卡 bring-up

DKMS 编译通过、user PF 可以打开之后，剩下的步骤基本按照 AMD 官方文档流程即可。

### 安装 U50 platform / firmware deb

从 [openDownload](https://www.xilinx.com/bin/public/openDownload) 搜索 "Alveo U50 Gen3x16 XDMA base"，选择 `202210_1` 平台版本。需要安装四个 deb（若使用 Vitis 编译 xclbin，还可以额外安装 dev deb）：

```bash
sudo apt install ./xilinx-cmc-u50_*.deb \
                 ./xilinx-sc-fw-u50_*.deb \
                 ./xilinx-u50-gen3x16-xdma-base_5-*.deb \
                 ./xilinx-u50-gen3x16-xdma-validate_5-*.deb
```

### 烧写 shell

第一次执行 `xbmgmt examine` 时，板卡处于 `GOLDEN_9` shell —— 这是 fallback 状态，说明出厂 shell 与当前 platform 版本不匹配。

```bash
sudo /opt/xilinx/xrt/bin/xbmgmt examine
sudo /opt/xilinx/xrt/bin/xbmgmt program --base --device 0000:2f:00.0
```

烧写完成后**必须做一次完全断电**（不是 `reboot`），断电约 10 秒后再上电。SC（satellite controller，板卡上的 MSP432 MCU）只有在 12V 电源掉电时才会 latch 新的 shell metadata。`reboot` 不切断电源，SC 状态不变，烧写不会真正生效。

### validate

```bash
source /opt/xilinx/xrt/setup.sh
sudo xbutil validate --device 0000:2f:00.1
```

输出：

```
Test 1 [0000:2f:00.1]  : aux-connection    PASSED
Test 2 [0000:2f:00.1]  : pcie-link         PASSED with warning
Test 3 [0000:2f:00.1]  : sc-version        PASSED
Test 4 [0000:2f:00.1]  : verify            PASSED
Test 5 [0000:2f:00.1]  : dma               PASSED
Test 6 [0000:2f:00.1]  : iops              PASSED
Validation completed. Please run the command '--verbose' option for more details
```

**6/6 全部通过**。`pcie-link` 的 warning 是链路协商到 Gen3 x4 而非 x16，原因在于我这里使用的是 USB4 显卡坞，PCIe 峰值带宽受限。

实测带宽：

- host ↔ FPGA（PCIe）：**~2.7 GB/s**（Gen3 x4 的实际上限约 3.5 GB/s，扣除 xdma overhead 大致就是这个水平）
- HBM 卡内单通道：**12.4 GB/s**（HBM2 单 pseudo-channel 理论 14 GB/s）
- IOPS：**377K/s**

---

## 七、开源与后续

整套工作已经开源，仓库地址：

```
https://github.com/Jsupermaster/xrt_ubuntu24_alveou50
```

目录结构：

```
├── install.sh              一键安装脚本
├── xrt-overlay/            11 个已修改的源文件（保留 XRT 目录结构）
└── patches/
    └── xrt-2024.2-linux-6.17.diff   完整合并 diff（review 用）
```

三行命令即可完成整个流程：

```bash
git clone https://github.com/Jsupermaster/xrt_ubuntu24_alveou50.git
cd xrt_ubuntu24_alveou50
./install.sh
# 脚本末尾会打印后续安装 U50 platform deb 与烧写 shell 的手动步骤
```

后续计划：

- **自定义 PCIe ↔ HBM loopback 测试**。已有 Vitis Accel Examples 中的 `hbm_bandwidth` 作为基线，接下来会写一个可配置的 loopback：M1 纯搬数、M2 kernel passthrough、M3 多 PC 并发，输出 JSON + CSV，出现 miscompare 时贴出 hex dump。
- **在 U50 shell 中寄生一颗 RISC-V CPU，跑 Linux**。CPU 选型尚未确定（NaxRiscv、CVA6、Rocket、Ibex 都在候选池中），大方向是把 CPU 放入 dynamic region，通过 XRT 的 mailbox 隧道把 UART 与 block device 引到 host。这一部分会在后续单独成文。

---

## 八、参考资料

**上游 XRT**

- 仓库：<https://github.com/Xilinx/XRT>
- 本次基线 tag：[`202420.2.18.179`](https://github.com/Xilinx/XRT/tree/202420.2.18.179)
- 相关 issue：[#8783 kernel 6.x DKMS 编译失败](https://github.com/Xilinx/XRT/issues/8783)
- AMD 官方 U50 下载页：[Alveo Previous Downloads - U50](https://www.amd.com/en/support/downloads/alveo-previous-downloads.html/accelerators/alveo/u50.html#alveotabs-item-vitis-tab)

**Linux 内核 API 变更**（按博客顺序，具体 commit hash 可通过 `git log <path> --grep=<keyword>` 检索）

- 6.8 `iommu_present` retire：Robin Murphy，iommu subsystem cleanup
- 6.10 `I2C_CLASS_SPD` remove：Wolfram Sang，i2c subsystem
- 6.10 UART `xmit` circ_buf → kfifo：Jiri Slaby，serial subsystem
- 6.11 `platform_driver::remove` returns void：Uwe Kleine-König，起点 commit `0edb555a65d1`
- 6.12 `no_llseek` remove：Christoph Hellwig，fs subsystem
- 6.12 `FOP_UNSIGNED_OFFSET` 引入：Kent Overstreet，`fs/file_operations`
- 6.13 `crc32c_le` rename：Eric Biggers，crypto lib
- 6.13 `MODULE_IMPORT_NS` stringify：Masahiro Yamada，kbuild
- 6.15 timer API 术语统一：Ingo Molnar，timer subsystem
- 6.16 `pfn_t` remove：David Hildenbrand，mm subsystem
- 6.16 `drm_driver::date` remove：Simon Ser，drm subsystem
- 6.17 `drm_open_helper` 强制 `FOP_UNSIGNED_OFFSET`：drm subsystem tightening

**AMD 官方文档**

- [UG1120 平台指南](https://docs.amd.com/r/en-US/ug1120-alveo-platforms)
- [UG1370 U50 安装手册](https://docs.amd.com/r/en-US/ug1370-u50-installation)

**本系列相关文章**

- 前文：[《Alveo 软件栈与安装组成梳理》](https://jsupermaster.github.io/posts/alveo-u50-software-stack.html)
- 后续：*RISC-V + Linux on Alveo U50（待发）*
