---
title: Xilinx(AMD) Alveo 软件栈与安装组成梳理：XRT、Platform、Firmware、Vitis 分别是什么
date: 2026-07-01 22:10:00 +0800
description: 梳理 Alveo U50 平台、固件、XRT 驱动与运行时、Vitis 开发工具之间的关系，明确部署环境与开发环境分别需要安装什么。
categories:
  - FPGA
home_category: engineering-practice
series: alveo-development
tags:
  - FPGA
  - AMD
  - Xilinx
  - Alveo
  - XRT
  - Vitis
article_kicker: FPGA NOTE
cover_image: /assets/images/alveo_u50/image-20260701223955214.png
word_count: 4.2k 字
read_time: 约 14 分钟
---

![Alveo U50 software stack cover](/assets/images/alveo_u50/image-20260701223955214.png)

## 一、前言

第一次接触 Alveo 时，想必很多人都会产生这些疑问：

- Alveo 是什么，和普通 FPGA 板卡有什么区别？
- Alveo 的开发流程是什么？
- `XRT`、`platform`、`shell`、`firmware`、`xclbin`、`Vitis` 分别是什么？

前段时间在闲鱼上淘到一张 Alveo U50C 板卡。相较官方板卡，它只是缺少 QSFP 光口，芯片和板卡主体与官方版本通用，可以直接识别并无缝使用 U50C 的官方工具链，性价比很高，因此我把这段学习过程整理成笔记，供后续参考。

这篇文章的目标，是把 Alveo U50 的软件栈和安装组成梳理清楚，回答“到底需要安装什么、每一层分别起什么作用”。

下一篇文章会进一步展开 Ubuntu 24.04 上的实际安装步骤、内核兼容性问题和补丁处理。

## 二、Alveo Platform

官方文档 `UG1120` 详细介绍了 Alveo 系列加速卡。这个系列覆盖多个型号，资源的大致区别如下表所示：

| 板卡 | 主机链路 | 板载外部内存 | HBM/DDR 组织 | 可用 LUT | 可用 FF | 可用 BRAM36 | BRAM 容量约合 | 可用 URAM | URAM 容量约合 | 可用 DSP |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| U50 | PCIe Gen3 x16 | 8 GB HBM2 | `HBM[0:31]` | 704K | 1410K | 1116 | 约 40.2 Mb | 544 | 约 156.7 Mb | 4920 |
| U55C | PCIe Gen3 x16 | 16 GB HBM2 | `HBM[0:31]` | 1146K | 2292K | 1776 | 约 63.9 Mb | 960 | 约 276.5 Mb | 8376 |
| U200 | PCIe Gen3 x16 | 64 GB DDR4 | `DDR[0:3]` | 978K | 1956K | 1860 | 约 67.0 Mb | 800 | 约 230.4 Mb | 5880 |
| U250 | PCIe Gen3 x16 | 64 GB DDR4 | `DDR[0:3]` | 1456K | 2915K | 2384 | 约 85.8 Mb | 1068 | 约 307.6 Mb | 10634 |
| U280 | PCIe Gen3 x16 | 32 GB DDR4 + 8 GB HBM2 | `DDR[0:1]` + `HBM[0:31]` | 1131K | 2265K | 1776 | 约 63.9 Mb | 960 | 约 276.5 Mb | 8304 |

> 注：
> 1. 表中的 `LUT / FF / BRAM / URAM / DSP` 依据 `UG1120` 的 `Available Resources After Platform Installation` 整理，表示平台安装后 `dynamic region` 实际可用资源，不是裸 FPGA 的全量资源。
> 2. `BRAM36` 按每块 `36 Kb` 折算，`URAM` 按每块 `288 Kb` 折算。

按照 `UG1120` 的定义，Alveo 平台可以理解为一套已经验证好的硬件和软件接口集合。它的目标不是直接承载某个具体业务，而是给后续加速应用提供一个稳定、可复用的运行底座。

和普通 FPGA 板卡相比，Alveo 更加强调整套软件工具链。用户不必自己处理 HBM、PCIe、XDMA 等底层驱动开发，而是把自己的设计打包成对外暴露 AXI 接口的 IP，通过 Xilinx 提供的工具链完成部署；相较 Zynq 系列板卡，Alveo 搭载的是纯 FPGA 逻辑芯片，不包含 Arm 硬核，而是通过预置的平台底座与主机协作完成任务加速。

官方实际上也提供了基于 Vivado 的设计流程，Alveo 加速卡也可以被当作普通 FPGA 开发板使用。但由于 Alveo U50 搭载了 HBM，驱动与平台约束更复杂，并且这张卡几乎没有用户 IO，因此把它当作普通板卡使用的难度远高于使用官方软件栈，也就是 `Vitis Design Flow`。以下讨论都建立在 `Vitis Design Flow` 的基础上。若采用 `Vivado Design Flow`，通常只需下载对应 `xdc` 文件后按常规 FPGA 流程开发。

从 FPGA 设计视角看，一个 Alveo 平台由两部分组成：

- `static region`：静态区域，负责卡作为一张 PCIe 加速卡正常工作的基础设施。
- `dynamic region`：动态区域，负责承载用户的加速逻辑，也就是最终会被应用加载和使用的部分。

可以把它理解成“底板 + 插件槽”的关系。`static region` 决定这张卡能否和主机通信、能否搬运数据、能否被管理和监控；`dynamic region` 决定用户真正跑什么加速逻辑。

因此，开发环境的部署，本质上是在打通这套“底板”，让我们自己的设计能够嵌入其中。部署环境时，重点落在 `static region` 对应的平台、驱动、固件和运行时是否完整可用。

## 三、静态区域（底板/基座）

`UG1120` 把平台静态区域里的基础能力拆成几个明确模块。理解这些模块，有助于理解为什么 Alveo 不是“插上就能直接当通用 FPGA 用”的设备。

- `HIF (Host Interface)`：主机接口，本质上就是卡和服务器之间的 PCIe 通道。
- `DMA`：负责主机和卡上内存之间的数据传输，是 Host 与设备数据交互的关键路径。
- `CRI (Clock, Reset, and Isolation)`：负责时钟、复位和隔离，保证卡在启动、重配置和异常场景下仍然可控。
- `CMP (Card Management Peripheral)`：板卡管理相关外设，用于健康状态、调试和编程等能力。
- `CMC (Card Management Controller)`：卡管理控制器，负责与卫星控制器、传感器等部件通信，也参与固件更新和板级管理。
- `ERT (Embedded RunTime Scheduler)`：嵌入式运行时调度器，负责计算单元调度和执行监控。

这些模块共同构成了平台底座。换句话说，用户最终写出来的 kernel 不是直接裸跑在 FPGA 上，而是运行在这样一套已经预置好的平台基础设施之上。

## 四、Xilinx 官方软件栈

把 Alveo 软件栈从上到下排开，大致可以分成五层。

### 1. 板卡平台层

这层对应官方文档里的 `target platform` 或 `deployment platform`。它定义了卡的基础硬件能力，包括 PCIe、DMA、时钟、管理、HBM/DDR/PLRAM 等资源如何组织，以及哪些资源留给用户逻辑使用。

对于 U50 来说，`UG1120` 里重点给出的平台是 `xilinx_u50_gen3x16_xdma_base_5`。这个名字本身就编码了卡型、链路形态、DMA 模式和平台代次。

### 2. 固件层

这层主要是 `SC` 和 `CMC` 固件。`SC` 是 Satellite Controller，可以理解成板卡上的辅助控制器/管理控制器固件模块。它们不直接承载用户计算，但负责板卡控制面能力，包括状态管理、传感器、控制器通信以及部分更新流程。

为了让 Alveo 加速卡正常工作，平台、驱动和固件需要协同配合，版本关系也必须匹配。

### 3. 内核驱动层

Alveo 在 Linux 下依赖 XRT 提供的内核驱动。常见的两个核心驱动是：

- `xclmgmt`：管理面驱动，负责板卡管理、平台加载、监控、时钟和复位等特权操作。
- `xocl`：用户面驱动，负责 buffer 管理、DMA、上下文、计算调度以及应用运行时的数据通路。

可以简单理解为：`xclmgmt` 更接近“管卡”，`xocl` 更接近“用卡”。

### 4. 用户态运行时层

这层就是 `XRT (Xilinx Runtime)` 的用户态部分。它向上提供命令行工具和编程接口，向下对接内核驱动和平台能力。

典型内容包括：

- `xbutil`：设备检查、诊断、验证、状态查看。
- `xbmgmt`：管理类操作，例如平台检查、配置和部分维护动作。
- XRT Native API
- OpenCL API
- 其他语言绑定和运行时库

Host 程序并不是直接操作 FPGA，而是通过 XRT 调用底层驱动和平台能力。

### 5. 开发工具层

如果只是部署环境、运行现成应用，到这里已经足够；如果要自己开发加速程序，还需要安装 `Vitis unified software platform`。

Vitis 负责的是开发工作，不是部署工作。它用于编译、链接和打包加速应用，最终生成面向目标平台运行的产物，例如 `xclbin`。

## 五、Platform、Shell、xclbin 分别是什么

这几个词经常被混用，但严格来说并不是一个层面的东西。

`platform` 是官方文档中的总体概念，强调的是一套经过验证的运行底座。`deployment platform` 更偏部署语境，强调“这张卡被配置成什么目标运行环境”。`shell` 在实际讨论里通常是平台底座的口语化说法，指承载 PCIe、DMA、时钟、管理等静态基础设施的那一层。

而 `xclbin` 则是应用侧产物。可以把它理解为“交付到卡上的加速逻辑容器”，里面不仅包含设备侧配置内容，也包含运行时需要识别的元数据。常规 FPGA 开发的最终产物往往是 `.bit` 或 `.bin`，而在 Alveo 开发中，常见交付物则是 `xclbin`。

所以，平台负责“让卡可用”，`xclbin` 负责“让应用可跑”。这两者不能互相替代。

## 六、官方安装包分成哪几类

`UG1120` 提到，Alveo 平台相关的 Linux 安装包主要可以分成三类。

### 1. Partition Package

这类包承载部署平台的一部分。官方命名里常见的分区名包括：

- `base`
- `shell`
- `validate`

需要注意的是，不同平台的组织方式并不完全一样。不能机械地把所有卡都理解成“永远都是 `base + shell` 两步安装”，具体还要看平台属于哪种 DFX 组织形式。

### 2. Validate Package

这类包提供平台安装后的验证程序，用于确认平台是否正确安装、板卡状态是否正常，以及环境是否达到了基本可运行条件。

后续常见的 `xbutil validate` 就属于这类验证路径的一部分。

### 3. Firmware Package

这类包提供 `SC` 和 `CMC` 固件。它们的作用不是提供用户计算能力，而是完善板卡控制面和平台管理能力。

## 七、包名的命名方式

官方包名并不是随便命名的，而是把关键信息编码进去了。例如：

`xilinx_u50_gen3x16_xdma_base_5`

这个名字至少能读出几层信息：

- `u50`：对应 U50 卡型
- `gen3x16`：对应 PCIe Gen3 x16 链路形态
- `xdma`：对应 DMA 架构
- `base_5`：对应平台代次和分区名称

如果再看安装包文件名，通常还会包含发布号、架构和发行版标记。理解命名规则的意义在于，后面看到多个平台包、验证包、固件包时，能快速判断它们是不是给同一类卡、同一类平台准备的。

## 八、部署环境和开发环境分别要装什么

这是最关键的一节，因为很多实际问题都来自“把部署环境和开发环境混在一起”。

### 只做部署和运行时

如果你的目标只是把卡装到机器里，运行已有程序，或者至少跑通 `xbutil validate`，那最小安装集合通常包括：

- `XRT`
- 与卡匹配的 `deployment platform`
- 对应的 `validate package`
- 必要时还包括 `SC/CMC firmware`

这一套解决的是“机器能不能识别卡、驱动能不能挂起来、平台能不能工作、运行时能不能看到设备”。

### 需要自己开发时

如果你要自己写 Host 程序、编译 kernel、生成 `xclbin`，那么在部署组件之外，还需要：

- `Vitis unified software platform`
- 面向目标卡的平台开发支持

这一套解决的是“能不能编、能不能链、能不能生成可交付的加速应用”。

所以，`deployment environment` 和 `development environment` 不是轻重不同的两个安装模式，而是目标完全不同的两套环境需求。

## 九、以 U50 为例，应该怎样理解实际安装对象

对 U50 而言，`UG1120` 中给出的重点平台之一是 `xilinx_u50_gen3x16_xdma_base_5`。官方同时给出了它的 `platform name`、`development name`、支持的 XRT 版本以及对应的卡型号信息。

以下是官方给出的 Alveo U50 支持下载页面：

<https://www.amd.com/en/support/downloads/alveo-downloads.html/accelerators/alveo/u50.html>

![Alveo U50 download page](/assets/images/alveo_u50/image-20260701224109869.png)

可以根据系统版本和 Vitis 版本选择对应的下载内容，核心通常就是这几个包。

在部署实战中，真正的工作顺序通常是：

1. 确认卡型号和目标平台名称。
2. 确认平台和 XRT 的版本关系。
3. 安装运行时、平台包、验证包和必要固件。
4. 最后再做设备检查和验证。

## 十、总结

如果只用一句话概括 Alveo U50 的官方软件栈，那么它本质上是“平台 + 固件 + 驱动 + 运行时 + 开发工具”组成的一套分层系统。

其中，平台负责让卡具备基础运行能力，固件负责控制面和板级管理，驱动和 XRT 负责把这些能力暴露给 Linux 和应用程序，Vitis 则负责把开发者写出的加速逻辑编译成可运行产物。

对于只做部署的人来说，最重要的不是先学会多少命令，而是先分清楚自己到底是在搭建 `deployment environment`，还是在搭建 `development environment`。这个边界一旦清楚，后面的安装路线就会清晰很多。

## 参考资料

- UG1120: <https://docs.amd.com/r/en-US/ug1120-alveo-platforms>
- XRT Documentation: <https://xilinx.github.io/XRT/master/html/index.html>
- Platforms and User Guides: <https://xilinx.github.io/XRT/master/html/platforms.html>
- XRT Installation Guide: <https://xilinx.github.io/XRT/master/html/install.html>
