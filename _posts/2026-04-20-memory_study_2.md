---
title: 内存与系统：第二章 DRAM实现与系统集成概述
date: 2026-04-20 19:30:00 +0800
description: 从软件模拟器、RTL 仿真到 SPICE 仿真，梳理 DRAM 研究与 PIM 系统设计中常见的建模、验证与集成路径。
categories:
  - Computer Architecture
tags:
  - Computer Architecture
  - DRAM
  - Simulator
  - Studing Note
article_kicker: MEMORY NOTE
cover_image: /assets/images/mem_study/Gemini_Generated_Image_cql4b1cql4b1cql4.png
word_count: 8.0k 字
read_time: 约 27 分钟
---

> 参考资料：
>
> gem5，Ramulator 2，DRAMSim3，Verilator 等官方仓库

![DRAM 实现与系统集成概述]({{ '/assets/images/mem_study/Gemini_Generated_Image_cql4b1cql4b1cql4.png' | relative_url }})

对于学术界来说，DRAM 的实现主要依靠仿真与模拟来评估。仿真与模拟主要可以分为三类，分别是**软件模拟器**（比如 gem5、Ramulator 2、DRAMSim3）、**SPICE 仿真**（提供真实电路级阵列模型）以及 **RTL 级仿真**（如利用 DRAM 的行为级模型、验证内存控制器逻辑等）。一般来说，一个完整的流程可能是：

1. `SPICE` 提取 DRAM 阵列的真实延迟、能耗、可靠性边界。
2. `RTL` 把这些边界转化为控制器等待周期、命令限制和数据路径结构。
3. `软件模拟器` 把 RTL / SPICE 产出的延迟、吞吐和能耗参数注入系统模型。

当然，对于当前成熟的 DRAM 芯片，这些开源的内存模拟器已经提供了比较精确的延迟、吞吐和能耗模型，但如果我们要做一些 PIM 相关研究，就不可避免地需要自己做 SPICE 仿真。由于博主的研究方向为 PIM 加速系统，因此在以下的探讨中特别地关注了内存模型是否更加容易进行 PIM 的拓展。

## 2.1 软件模拟器

这一层级是评估系统级性能和运行大型软件栈的主力。软件模拟器能够作为全系统评估的有利手段，进行指令集、软件栈、端到端系统的模拟与评估。对于软件模拟器来说，最大的意义是通过模拟器来评估一款 DRAM 芯片或者 PIM 加速器在整个系统中的延迟、调度、能耗以及端到端加速情况。我们主要关心以下几个问题：

### 2.1.1 全系统 / 软件栈集成能力

关注该工具能否支撑完整软件实验，而不是只跑一个 memory trace。

重点关注：

- 是否支持 `Full-System`，即能否启动完整 Linux。
- 是否支持 `RISC-V`、`x86` 或 `ARM` 等常见 ISA。
- 是否能运行真实用户态程序、驱动、运行时和 AI 框架产出的工作负载。
- 是否能通过 syscall emulation、checkpoint、KVM 或 sampling 降低系统启动成本。

这一点上，不同工具差异很大：

- `gem5` 是最典型的全系统模拟器，支持多 ISA 和 Linux 启动，适合做 OS、驱动、编译器、PIM ISA 的联合评估。
- `Ramulator 2` 不是完整系统模拟器，它更像是高保真的内存系统模型，适合独立研究 DDR/PIM 行为，或者被接入 gem5 一类系统模拟器中。
- `DRAMSim3` 也是以内存系统建模为主，既可独立 trace-driven 运行，也可和外部 simulator 协同。
- `ZSim` 的优势是快，但它本质上是高性能的用户态 x86 模拟框架，不是像 gem5 那样的完整全系统平台；如果研究目标是“启动 Linux 并跑完整 AI 栈”，它不能直接替代 gem5。

因此，如果我们希望“从 Linux 启动一路跑到 LLM kernel”，软件层的主平台通常应是 `gem5`，而 `Ramulator 2 / DRAMSim3` 更适合做内存子系统高保真建模或联合仿真。

> ### Gem5 自带的内存模型和 Ramulator 2 / DRAMSim3 外部模型的区别
>
> #### 1. 建模粒度与状态机的精确度（Fidelity & Accuracy）
>
> - **Gem5 自带模型（`DRAMCtrl` / `MemCtrl`）**：
>   - 它采用的是一种**基于队列和事件驱动（Event-driven）**的抽象模型。
>   - 它会追踪基本的 Bank 冲突、Row Buffer 命中/缺失，并应用基础的时序参数（如 $t_{RCD}$、$t_{CAS}$）。
>   - **缺点**：它并不维护一个严格的、周期精确（Cycle-accurate）的 DRAM 状态机。对于复杂的 DRAM 行为（如细粒度的刷新机制、Bank Group 级别的时序约束、总线反转等），它的模拟是近似的，这会导致在内存密集型任务中，测得的延迟和带宽与真实芯片有偏差。
> - **Ramulator 2 / DRAMSim3（外部专用模型）**：
>   - 它们是**严格的周期精确（Cycle-accurate）状态机模型**。
>   - 它们在代码底层完整复现了 JEDEC 标准定义的各种状态（Idle、Active、Precharged、Power-down 等），并严格遵循几百个时序参数（Timing Constraints）的依赖关系。
>   - **优势**：能够极其精准地模拟总线冲突、复杂的刷新策略（如 Per-Bank Refresh）、以及热耗散引起的降频等极度底层的硬件行为。
>
> #### 2. 支持的内存标准与更新速度
>
> - **Gem5 自带模型**：
>   - 内置了 DDR3、DDR4、LPDDR、HBM 的基础配置，但通常落后于业界最新的标准。
>   - 如果想要研究最新的 HBM3、DDR5 的高级特性，或者 CXL 内存池化，Gem5 的内置模型往往无能为力。
> - **Ramulator 2 / DRAMSim3**：
>   - 作为专门的内存研究工具，它们的更新非常快，紧跟学术界和工业界的前沿。
>   - **DRAMSim3**：支持 DDR3/4/5、LPDDR3/4、HBM、HMC，甚至 GDDR。它的强项在于自带了非常成熟的**功耗和温度模型**。
>   - **Ramulator 2**：这是目前学术界最火的工具。它不仅支持几乎所有最新的内存标准，其架构被设计成高度模块化，**原生提供了对近存计算（PIM）、存内计算和 CXL 的扩展支持**。
>
> #### 3. 仿真速度与系统开销
>
> - **Gem5 自带模型**：
>   - 因为抽象程度较高，且与 Gem5 的事件队列深度耦合，**仿真速度最快**。没有跨进程或跨模块通信的开销。
> - **外部模型（联合仿真）**：
>   - 当 Gem5 与 DRAMSim3 或 Ramulator 联合仿真（Co-simulation）时，Gem5 的内存控制器实际上变成了一个“接口（Port）”。每次内存请求都需要将包从 Gem5 转换格式，发送给外部模拟器，外部模拟器计算出精确的延迟后再返回给 Gem5。
>   - 这种跨界面的交互会增加额外的开销，导致**整体仿真速度变慢（通常会慢 10% 到 30% 不等）**。
>
> #### 4. 易用性与修改成本
>
> - **Gem5 自带模型**：
>   - 开箱即用，写几行 Python 配置文件就能跑。但由于 Gem5 源码庞大且耦合度高，想在 `DRAMCtrl.cc` 里面修改核心的调度算法或添加非标准指令，难度很大。
> - **外部模型**：
>   - 需要先单独编译外部模拟器，然后重新编译 Gem5 以建立耦合，通常通过修改 Gem5 的构建脚本并启用相应的 Flag。
>   - 但是，一旦耦合成功，由于 Ramulator 2 等工具代码结构清晰、专为内存设计，**在其中添加自定义的硬件逻辑（如计算单元）要比在 Gem5 中修改容易得多**。

### 2.1.2 DRAM 模型的保真度与扩展性

这一维度要回答的问题是：工具能否把 DDR 的真实结构限制体现出来，以及在现有抽象层中插入 PIM 逻辑到底难不难。

以下是我们重点关注的问题：

- 是否显式建模 `Channel / Rank / Bank / Bank Group / Row / Column`。
- 是否支持 JEDEC timing，如 `tRCD`、`tCL`、`tRP`、`tRAS`、`tFAW`、refresh 相关约束。
- 是否能观察 `row-buffer hit/miss`、bank 冲突、请求重排和调度策略影响。
- 是否提供相对清晰的命令层或控制器层接口，便于插入自定义 PIM 命令。
- 是否便于把 PIM 设计成：
  - 新的 memory-side opcode
  - 特殊 MMIO 寄存器触发
  - 自定义 DMA / command queue
  - logic-die 的黑盒计算单元

在可扩展性上，各工具可以这样看：

- `Ramulator 2` 的模块化程度较高，适合增加新命令、新状态机和新 timing 约束，是当前调研 `PIM-aware DRAM simulator` 时最值得重点看的工具之一。
- `DRAMSim3` 的协议实现和 timing 组织也较清晰，适合在现有 DDR 协议模型上增加扩展命令或实验性事务。
- `gem5` 的强项是系统级集成，但内存侧扩展通常涉及 SimObject、memory controller、packet/message path 的联动，改动面往往比单纯改 Ramulator 更大。
- `ZSim` 适合做快速性能探索，但如果要把 PIM 机制深入地嵌入内存协议层，它不是最自然的切入点。

### 2.1.3 仿真速度与加速技术

跑 LLM 或完整 AI 框架时，软件模拟器最大的现实瓶颈不是建模能力，而是时间。

我们重点关注以下问题：

- `Fast-forwarding`：是否支持快速跳过初始化阶段。
- `Checkpointing`：是否支持从 Linux 已启动、模型已加载的状态继续仿真。
- `Sampling / SimPoint`：是否支持对长程序只抽样关键片段。
- `KVM / Hardware-assisted acceleration`：是否能借助宿主机虚拟化能力加速前期执行。

这一维度上：

- `gem5` 具备 checkpoint 机制，也支持 SimPoint 流程；在合适宿主环境和 ISA 组合下，还可以使用 KVM CPU 加速前期执行。
- `Ramulator 2` 和 `DRAMSim3` 由于不承担完整 OS 模拟负担，天生更快，但它们通常需要外部 trace、外部 simulator 或者简化工作负载接口。
- `ZSim` 的主要优势正是速度，因此它适合作为快速体系结构探索平台，但不是替代 full-system 验证的平台。

如果研究对象是 LLM，实际调研中应该明确区分两类实验：

- `端到端实验`：用 full-system 平台跑从用户态到 kernel 的完整链路。
- `核心算子实验`：仅抽取 attention、matmul、KV-cache 访问等关键 memory trace，在更快的 memory simulator 上做大规模扫参。

这两类实验最好并行存在，而不是只选其中一种。

### 2.1.4 功耗与面积的粗粒度建模

软件模拟器在功耗方面通常不是“真值”，但它是做系统级能效估算的起点。

需要关注：

- 是否能导出命令计数、访存统计和活动窗口。
- 是否能与 `DRAMPower`、`McPAT` 或自定义能耗模型联动。
- 是否能把 `PIM 算子延迟`、`单位操作能耗`、`附加面积代价` 作为参数注入。
- 是否支持 thermal-aware 或至少可被外部 thermal model 消费。

这一层比较合理的研究产物不是“精确功耗”，而是：

- 系统级能效趋势
- 不同 PIM placement 的相对优劣
- 带宽节省与额外内存侧能耗之间的 tradeoff

内存模拟器已经提供了主流 DRAM 芯片的一些参数，但对于 PIM 这类需要修改 DRAM 芯片内部的改动仍需 SPICE 仿真才能提供精确的功耗与面积评估。

### 2.1.5 总结

对于大部分实验来说，软件模拟器使用 Gem5 + Ramulator 或 Gem5 + DRAMSim 的组合即可较好地完成既定目标。如果是像 PIM 这样需要对 DRAM 内部做修改的设计，则需要通过 SPICE 来获得更加精确的电路级参数和模型信息。

## 2.2 RTL 仿真

对于 RTL 仿真来说，目标往往不是跑完整 Linux，而是验证数字电路设计的正确性，比如自己设计的内存控制器、PIM 处理单元等。调研重点应放在 `DDR 协议合规性`、`控制器二次开发难度`、`跨层接口` 和 `综合后资源/频率`。

### 2.2.1 DDR 协议合规性与厂商模型获取

DRAM 本身是一个数模混合芯片，只有数字部分能够通过 RTL 模型进行仿真，其中主要是内存控制器、自己的处理单元等。为了能做 RTL 级仿真，必须获得 DDR 的仿真模型。

- 如何获取真实的 DDR4/DDR5 Verilog 仿真模型。
- 是否能在 testbench 中接入真实 timing check。
- PIM 命令如何与 `ACT / PRE / RD / WR / REF` 共存。
- 新命令是否会破坏 refresh、安全时序窗口和 data bus turnaround。

核心的关注点集中在以下层面：

- 新的 PIM 事务在协议上是否合法。
- 控制器是否能够在 refresh、bank 冲突和 backpressure 存在时稳定工作。
- 内存侧扩展是否要求修改标准 DDR 命令语义，还是只改控制器内部行为。

### 2.2.2 开源 IP 与内存控制器的二次开发

真实研究里，很少从零开始手写完整 DDR 控制器。更现实的路线是选择一个结构清晰、边界明确、便于插桩的开源框架。

重点关注：

- 控制器的调度器、命令生成器和 data path 是否模块化。
- 是否容易在 read/write path 上插入 PIM ALU、reduction、accumulate 或 format conversion。
- 是否有清晰的前端总线接口，便于和 SoC、DMA、CPU 对接。

一些开源的候选是：

- `LiteDRAM`：开源、结构清晰，适合教学和研究性改造，是“二次开发 DDR 控制器”的主力候选。
- `Chipyard`：它本身不是 DDR 控制器，但它提供的是 SoC 生成与仿真框架，可以把你修改过的 memory subsystem 挂进完整 RISC-V SoC 中。
- `FireSim`：依托 Chipyard / FPGA 的全系统平台，更适合后续原型和大规模软件验证，不是替代 RTL 本身，而是向上承接。

### 2.2.3 跨层验证与软硬协同接口

纯 RTL 仿真很慢，无法直接承担完整软件栈，因此必须研究如何与上层软件激励对接。

常见做法包括：

- 用 `DPI-C` 把 C/C++ 侧生成的 memory trace、命令流或算子请求送进 RTL。
- 用 `Verilator` 把 RTL 编译成 C++ 模型，再和软件测试框架或 trace replay 工具联动。
- 先在软件模拟器中提取关键算子的访存行为，再在 RTL 中只重放关键片段。
- 在 `Chipyard` 中做较小规模的软件-RTL 联动验证，再把更长的软件流交给 FireSim 或软件模拟器。

这一部分的核心问题是：

- 软件层想暴露给 RTL 的接口到底是什么。
- 是传 `ISA 指令`、`memory request`、`command trace`，还是 `算子级事务`。
- 哪一层负责把“指令”翻译成控制器动作。

### 2.2.4 硬件资源与时序评估

RTL 仿真只能说明逻辑正确，不能说明它“代价合理”。因此必须同步考虑综合和时序收敛问题。

应重点考虑：

- 是否能用 `Vivado`、`Design Compiler` 或等价工具做综合。
- PIM 逻辑插入后增加多少 `LUT / FF / BRAM` 或 ASIC `gate count / area`。
- 关键路径是否仍能满足目标频率。
- 若目标是 DDR PHY 附近的几 GHz 频率，PIM 逻辑是否必须 pipeline。

因此，RTL 层最有价值的产出之一，不只是“波形正确”，而是：

- 面积开销
- 时钟频率上限
- 需要多少 pipeline stage
- 是否必须改变控制器微架构

### 2.2.5 总结

RTL 级仿真与综合能够针对数字电路部分给出比较准确的面积、时序和详细的微架构评估。但由于 DRAM 本身是一个数模混合芯片，因此实际上必须进行联合仿真与设计。

> ### 数字电路和模拟电路如何进行联合仿真与实验
>
> #### 1. 自底向上（Bottom-Up）：从 SPICE 提取模型供 RTL 验证
>
> 这是在架构探索和数字逻辑验证阶段最常用的方法，目的是**为了让慢速的模拟电路“跟得上”快速的数字仿真**。
>
> - **时序与功耗特征提取（Characterization）**：
>   - **怎么做**：工程师会先用 SPICE 仿真极小块的模拟电路，例如一个 Sense Amplifier 或者一个 Row Decoder。通过在不同的温度、电压下跑千万次 SPICE 瞬态分析，得出它“输入信号到输出信号需要多少皮秒（ps）”“翻转一次消耗多少飞焦耳（fJ）”。
>   - **产出**：这些数据会被打包导出成一种标准的文本格式，叫做 **`.lib`（Liberty 格式）** 文件或时序模型。
>   - **供 RTL 使用**：RTL 仿真工具，如 VCS，或者综合工具在跑数字电路时，遇到这个模拟模块，就不会去算节点电压，而是直接查表：“哦，遇到读指令，延迟 15ns，功耗 2mW”，从而极大地加快仿真速度。
> - **行为级建模（Behavioral Modeling）**：
>   - **怎么做**：对于整个存储阵列（Array），工程师会使用 **Verilog-A / Verilog-AMS** 或 SystemVerilog 的**实数建模（Real Number Modeling, RNM）**，手写一套代码来“模仿”模拟电路的行为。
>   - **供 RTL 使用**：在数字验证人员的眼里，模拟阵列变成了一个可以瞬间返回数据的“黑盒”，从而让数字端能够跑通完整的 DDR 协议栈。
>
> #### 2. 自顶向下（Top-Down）：将 RTL 转换到 SPICE 级别进行模拟
>
> 这是在芯片即将流片（Tape-out）前的最终签核（Sign-off）阶段必须做的事，目的是**为了验证真实的物理效应**，比如数字时钟跳变会不会把模拟阵列里的微弱电荷干扰掉。
>
> - **RTL 到网表到 SPICE 的降维打击**：
>   - **怎么做**：真实的流程并不是把 RTL 代码直接扔进 SPICE，而是将 RTL 代码经过逻辑综合（Synthesis）和布局布线（Place & Route），变成由真实逻辑门构成的**晶体管级网表（Transistor-level Netlist）**，并提取出所有的寄生电阻和电容（RC Extraction）。
>   - **合并仿真**：这时，数字部分的庞大网表和模拟阵列的网表拼在一起，形成一个包含几千万甚至上亿个晶体管的超级大网表。
> - **应对手段：Fast-SPICE 仿真器**：
>   - 用传统的 HSPICE 跑这个级别的电路，可能跑到宇宙毁灭都跑不完。因此，原厂会使用 **Fast-SPICE 工具**，如 Synopsys CustomSim、Cadence Spectre XPS。这类工具会在内部做数学近似，比如把不活跃的电路模块休眠，或者降低精度，能在牺牲极小精度（约 1% 到 5%）的情况下，把 SPICE 仿真速度提升成百上千倍。
>
> #### 3. 最强杀器：数模混合协同仿真（Co-Simulation）
>
> 这是目前业界日常验证的主流方式。不再是互相倒出模型，而是**让 RTL 仿真器和 SPICE 仿真器“实时连麦”**。
>
> - **运行机制**：
>   - 准备两个工具，例如 Cadence 的 Xcelium 跑 RTL，Spectre 跑 SPICE。工具之间会通过一个叫做 **AMS（Analog Mixed-Signal）** 的底板连接。
>   - 当数字信号（0 或 1）需要传给模拟阵列时，仿真器会在边界上自动插入一个虚拟的 D2A（数字转模拟）转换器，把数字逻辑“1”瞬间转换成带有上升沿斜率的 1.2V 电压波形。反之亦然（A2D）。
> - **优势**：这实现了数字的高速和模拟的高精度的完美折中。

利用 Verilator 的一种思路是：将 RTL 代码转成 C++，之后利用 C++ 的行为级模型进行系统级实验，既能够加快速度，又能够保障数字电路部分的准确性；同时使用 C++ 开发行为级模型的难度略小于开发 Verilog 行为级模型，因为高端的 DDR4/5、HBM 等模型很难获得厂商的行为级模型。这种思路适合作为全系统评估的精确周期仿真。

专注于 DRAM 芯片的设计，则需要联合 RTL 模型与 SPICE 仿真，可以通过从 SPICE 提取模型，再给 RTL 提供行为级模型来实现；也可以利用 EDA 公司提供的工具将 RTL 综合出来的网表集成到 SPICE 中，或是直接利用 EDA 工具来做联合仿真。

## 2.3 SPICE 仿真

这一层用于探索物理实现。如果要深入到 `Bank 内部`、`Sense Amplifier`、`charge-sharing` 等，那么 SPICE 是决定方案能否成立的核心证据。

### 2.3.1 DRAM Cell 与阵列级建模方法

首先应回答一个问题：究竟要模拟到什么粒度。

常见粒度包括：

- 单个 `1T1C DRAM cell`
- 一条 wordline 和若干 bitline 的局部阵列
- 含 precharge、equalization、sense amplifier 的子阵列片段
- 带寄生提取的 post-layout 局部电路

这一部分应重点看：

- 如何建立 `1T1C` cell 的等效模型。
- 如何提取 `Wordline / Bitline` 的 RC 参数。
- 是否需要加入 process corner、voltage variation、temperature variation。
- 如果是 in-DRAM compute，是否需要 Monte Carlo 分析看误差分布和最坏情况。

这一步的目标不是追求大规模，而是找准“最小但足够解释机制”的电路边界。

### 2.3.2 算子级电路修改的物理影响

如果 PIM 逻辑深入到 DRAM 阵列内部，真正难的不是“算子能不能工作一次”，而是“它会不会破坏原有存储行为”。

这一部分应重点分析：

- 修改 sense amplifier 后，原始读写裕量是否下降。
- 位线电荷共享是否引入额外误差、读扰动或写回失败。
- 是否增加漏电、静态功耗或动态能耗。
- 是否因为附加晶体管或路由导致阵列密度下降。
- 是否对 retention、noise margin、refresh 周期造成副作用。

如果这些问题没有被认真分析，那么“在 SPICE 里做出一个逻辑运算”并不能证明该方案可用。

### 2.3.3 工艺节点与 PDK 的近似获取

这部分必须非常谨慎，因为 commodity DRAM 的真实工艺库通常拿不到。

学术界常见替代路径包括：

- 使用 `FreePDK45` 一类公开工艺做外围逻辑近似分析。
- 使用 `ASAP7` 这类预测性先进节点 PDK 评估外围控制逻辑的趋势。
- 对 DRAM cell 本体采用论文中公开的等效参数、经验值或简化模型。

但必须明确：

- `FreePDK45`、`ASAP7` 适合近似 `外围逻辑` 或趋势分析，不等价于真实 commodity DDR 的 DRAM 阵列工艺。
- 用这些 PDK 得出的绝对延迟和能耗可以作为 `研究近似值`，但不应直接宣称代表真实商品 DDR die。

### 2.3.4 向上层建筑的参数馈送

SPICE 的最终目的不是只画一张漂亮波形，而是把结果变成上层模型能用的参数。

最重要的调研任务是定义参数接口：

- 某个 PIM 算子的 `单位延迟`
- 单位操作 `动态能耗`
- 对 refresh、时钟周期或控制器时序的额外约束
- 可靠性边界，如最小电压裕量、误码率趋势、温度敏感性

这些结果应被打包成黑盒参数，提供给：

- `RTL`：决定 pipeline 深度、等待周期、控制器仲裁和非法状态约束
- `软件模拟器`：决定 memory-side opcode 的延迟、吞吐、功耗和热约束

如果 SPICE 结果不能上送到 RTL 或软件层，这部分工作就只能算机理展示，而不是系统研究闭环。

### 2.3.5 总结

SPICE 仿真是针对模拟存储阵列的电路级建模，能够最精准地反映真实的电路指标。如果我们对于 bank 内部进行了修改，则需要通过 SPICE 来模拟我们的电路，从而获得延迟、功耗等真实数据，再将其反馈到软件模拟器或 RTL 模型中，完成系统可靠性闭环。
