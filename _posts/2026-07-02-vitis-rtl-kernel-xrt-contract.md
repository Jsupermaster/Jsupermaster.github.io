---
title: "Vitis RTL Kernel 与 XRT 的驱动契约：为什么能打包成功却 `run.wait()` 卡死"
date: 2026-07-02 20:00:00 +0800
description: 从主机调用、AXI4-Lite 控制寄存器、AXI4 主端口和 kernel.xml 四个层面梳理 Vitis RTL kernel 必须满足的 XRT 契约，解释为什么很多设计能成功打包，却在运行阶段卡在 `run.wait()`。
categories:
  - FPGA
home_category: engineering-practice
series: alveo-development
tags:
  - FPGA
  - Alveo
  - XRT
  - Vitis
  - RTL
  - AXI
article_kicker: FPGA NOTE
cover_image: /assets/images/alveo_u50/image-20260701223955214.png
word_count: 约 7k 字
read_time: 约 24 分钟
---

![Vitis RTL kernel and XRT contract cover](/assets/images/alveo_u50/image-20260701223955214.png)

## 一、前言

上一篇文章我整理了 Alveo U50 的软件栈，解释了 `XRT`、platform、firmware、`Vitis` 分别是什么，以及部署环境和开发环境各自需要安装什么。那篇文章回答的是“卡怎样被装起来”；这一篇则进一步回答另一个更容易踩坑的问题：

**一份自写的 RTL，到底要满足什么条件，才能被 XRT 当成一个 kernel 正常启动、等待、收尾？**

这个问题表面上看像是“写个 `kernel.xml` 再 `package_xo` 就行”，但实际并没有那么简单。很多设计的问题都不是出在综合、布局布线或者 `package_xo` 阶段，而是出在**契约没有对齐**：

- 顶层端口名字看起来差不多，但 Vitis 不认。
- AXI4-Lite 能读能写，但 `ap_done` 语义不对，结果 `run.wait()` 永远不返回。
- `kernel.xml` 里的参数偏移和 RTL 内部译码不一致，主机传的地址到了硬件里全错位。
- `m_axi` 能发事务，但 burst、地址宽度或 `WLAST` 处理不符合平台预期，结果读写数据异常。

本文把我手头一份偏“内部参考文档”风格的笔记，整理成博客版说明。目标不是逐条复述官方手册，而是给出一个更实用的视角：

**从 XRT 实际怎样驱动 RTL kernel 出发，反推 RTL 顶层、寄存器和元数据必须长成什么样。**

---

## 二、先看本质：XRT 眼中的 RTL kernel 是什么

一个 Vitis RTL kernel，本质上不是“任意一段 RTL 逻辑”，而是：

1. 一份**接口符合约定的 Vivado IP**；
2. 通过 `package_xo` 打包进 `.xo`；
3. 再由 `v++ -l` 链接进目标平台，形成最终的 `xclbin`；
4. 最后由主机侧 XRT 通过 MMIO 和中断把它当作一个 compute unit 调用。

从驱动视角看，XRT 对 RTL kernel 主要做两件事：

- 通过 **AXI4-Lite 控制口**写参数、写启动位、读状态寄存器；
- 通过 **`interrupt`** 等待 kernel 完成，或者退化成 polling。

而 kernel 自己真正访问 HBM / DDR / PLRAM，并不是 XRT 代劳，而是通过 **`m_axi_<port>`** 主端口向平台内存系统发 AXI4 事务。

可以把整体关系压缩成下面这张图：

```text
+-------------------------+                    +-----------------+
| Host (XRT API)          |                    | Alveo Card      |
|                         |  PCIe MMIO         |                 |
| xrt::kernel(...)     ---+------------------> | s_axi_control   |
| kernel(bo0, bo1, n)     |                    |   RTL Kernel    |
| run.wait()           <--+------ MSI --------+  interrupt       |
|                         |                    |  m_axi_gmem*    |
| xrt::bo / DMA        <--+------------------> | HBM / DDR bank  |
+-------------------------+                    +-----------------+
```

所以，所谓“XRT 契约”，其实至少包含三层：

- **端口契约**：顶层 module 必须暴露固定命名和固定信号集的接口。
- **寄存器契约**：控制寄存器前 16 字节的语义必须符合 `ap_ctrl_hs`。
- **元数据契约**：`kernel.xml` 必须准确描述参数、偏移和端口绑定关系。

这三层只要有一层没对齐，就很容易出现一种特别恼人的情况：

**打包成功，链接成功，主机能打开 kernel，但运行时卡死。**

---

## 三、最重要的一句话：XRT 不是在“调用你的 RTL”，而是在“执行一套固定握手”

很多初学者会把 `xrt::kernel` 理解成类似“远程函数调用”，但对 RTL kernel 来说，真实发生的事情要朴素得多。

假设主机侧代码大致如下：

```cpp
auto device = xrt::device(0);
auto uuid   = device.load_xclbin("acc.xclbin");
auto krnl   = xrt::kernel(device, uuid, "acc_kernel");

auto bo_in  = xrt::bo(device, N * 4, krnl.group_id(0));
auto bo_out = xrt::bo(device, N * 4, krnl.group_id(1));

auto run = krnl(bo_in, bo_out, N);
run.wait();
```

其中真正会落到 RTL 顶层上的关键动作，其实是一串固定的 AXI4-Lite MMIO 操作：

1. 按 `kernel.xml` 里的参数偏移，把每个参数写到 `s_axi_control` 的寄存器空间。
2. 写 `GIER` 和 `IP_IER`，打开中断。
3. 向 `CTRL[0]` 写 `1`，也就是 `ap_start`。
4. kernel 运行，内部通过 `m_axi_*` 访问外部内存。
5. 完成时硬件拉起 `ap_done`，置位中断状态寄存器，并把 `interrupt` 拉高。
6. XRT 收到 MSI 后，读状态、清中断，`run.wait()` 返回。

也就是说，主机与 RTL 的“函数调用关系”最终都会被压缩成一句话：

**XRT 是否能按照预期写对寄存器、看到正确状态、收到正确中断。**

理解这一点后，很多定位思路就会清晰很多。`run.wait()` 卡死时，优先怀疑的通常不是算术逻辑错了，而是：

- `ap_done` 没有按约定置位；
- `interrupt` 没有正确反映 `ISR/IER/GIER`；
- `ap_idle` 一直不对，导致下一次运行前 XRT 认为 kernel 还在忙；
- `kernel.xml` 的参数偏移错了，导致 kernel 读到的是垃圾地址。

---

## 四、顶层端口先别自由发挥，Vitis 对命名是较真的

### 4.1 必须出现的几类端口

一个最典型的 RTL kernel 顶层，至少要包含下面这些逻辑接口：

| 用途 | 名字约定 | 说明 |
| --- | --- | --- |
| 时钟 | `ap_clk` | 所有接口共用的 kernel 时钟 |
| 复位 | `ap_rst_n` | 低有效，同步复位 |
| 控制口 | `s_axi_control_*` | 标准 AXI4-Lite slave 信号集 |
| 数据口 | `m_axi_<name>_*` | 标准 AXI4 full master 信号集 |
| 中断 | `interrupt` | kernel 完成时通知主机 |

这里最容易被忽略的一点是：**Vitis packager 不是“语义理解”，而是大量依赖命名约定识别接口类型。**

例如下面这些看似只差一点点的名字，实际上都可能让工具直接识别失败：

- `ap_reset_n`
- `axi_control`
- `m_axi_data`
- `irq`

对 RTL kernel 来说，接口命名不是风格问题，而是协议的一部分。

### 4.2 AXI 信号必须成套

另一个高频错误，是端口前缀写对了，但信号集不完整。例如：

- AXI4-Lite 漏了 `AWPROT` / `ARPROT`
- AXI4 full 漏了 `AWLEN` / `ARLEN`
- 写通道没有 `WSTRB`
- 读通道没有 `RLAST`

这种情况下，Vivado IP packager 往往不会按你想象中的方式“自动补足”，而是直接把它识别成 generic signal bundle，后面 `v++` 就无法把它自动接进平台互连。

所以更稳妥的策略不是“先写个精简接口试试”，而是：

**按标准 AXI4-Lite / AXI4 full 信号全集把顶层一次性定义完整，再在内部按需要使用。**

### 4.3 顶层最好不要出现这些东西

以下几类内容通常不应该出现在 RTL kernel 顶层：

- `inout`
- 自定义 GPIO 风格外设信号
- 第二套用户时钟
- 异步复位风格写法

原因不复杂。Alveo 平台上的 kernel 不是一个裸 FPGA 工程顶层，而是一个要被装进既定 platform shell 的加速单元。它的边界应该尽量收敛在：

- `ap_clk` / `ap_rst_n`
- `s_axi_control`
- `m_axi_*`
- `interrupt`

---

## 五、真正决定能不能跑起来的核心：`s_axi_control`

如果说顶层端口解决的是“工具能不能识别你”，那么 `s_axi_control` 解决的就是“XRT 能不能驱动你”。

### 5.1 AXI4-Lite 数据宽度固定是 32 位

`s_axi_control` 的 `WDATA` / `RDATA` 必须是 32 位。这一点经常和 `m_axi` 数据通路混淆。

- `m_axi` 可以是 32 / 64 / 128 / 256 / 512 位。
- 但控制寄存器口始终是 **32-bit AXI4-Lite**。

这意味着即使主机传入的是 64 位地址，最终也必须拆成两个 32 位寄存器保存。

### 5.2 前 16 字节不是“建议”，而是固定契约

`s_axi_control` 的起始 16 字节必须保留给控制寄存器，常见布局如下：

| 偏移 | 名字 | 关键位 |
| --- | --- | --- |
| `0x00` | `CTRL` | `ap_start` / `ap_done` / `ap_idle` / `ap_ready` / `auto_restart` |
| `0x04` | `GIER` | 全局中断使能 |
| `0x08` | `IP_IER` | `ap_done` / `ap_ready` 中断使能 |
| `0x0C` | `IP_ISR` | 中断状态，通常 W1C |

这里的重点不是“有这几个寄存器”，而是**每一位的行为语义必须对**。

### 5.3 `ap_start` 最容易写错

`CTRL[0]` 的标准语义不是“写 1 就一直维持高电平”，而是更接近：

- 主机写 `1` 触发启动；
- kernel 内部消费这个启动事件；
- 随后该位自动清零，或者在 `ap_ready` 后回到低电平；
- 如果支持 `auto_restart`，则按 `CTRL[7]` 决定是否继续置高。

如果把 `ap_start` 错写成一个普通的 level 寄存器，经常会导致两类问题：

- kernel 被持续重复启动；
- XRT 无法判断这次启动是否已经被硬件消费。

### 5.4 `ap_done` 必须是“完成事件”，不是“完成状态永远拉高”

`CTRL[1]` 表示一次运行完成。常见实现方式是：

- 内部 `ap_done_pulse` 来一次，就把 `ap_done` sticky 置 1；
- 主机读 `CTRL` 时清零，或者在下一次 `ap_start` 时清零。

博客语境下不必纠结所有变体，但有一条原则非常关键：

**XRT 需要看到“这次运行完成”的可观察事件，而不是一个没有生命周期的常高位。**

`ap_done` 如果从不被清掉，下一次运行的语义会混乱；如果它从来不置位，`run.wait()` 就只会一直等下去。

### 5.5 `ap_idle` 和 `ap_ready` 看起来次要，实际上决定复用行为

`ap_idle` 表示 kernel 当前是否空闲。对简单的非流水 kernel 来说，语义可以理解成：

- 未运行时为 1
- 从启动到完成期间为 0
- 完成后回到 1

`ap_ready` 则表示 kernel 是否可以接受下一次输入。对大多数手写 RTL kernel，如果没有做流水式重叠启动，把它做成与 `ap_done` 同步通常就够了。

很多人第一次只跑一遍测试时看不出问题，但第二次 `run` 就卡住，根源往往是：

- `ap_idle` 一直没回到 1
- `ap_ready` 从未对主机表现出“可再次接收”的状态

### 5.6 中断寄存器别省

即使你准备先用 polling 跑通，也建议把这三组寄存器完整实现：

- `GIER`
- `IP_IER`
- `IP_ISR`

并且让 `interrupt` 的语义与它们保持一致，例如：

```verilog
assign interrupt = int_gie & (|(int_isr & int_ier));
```

这类实现本身并不复杂，但能避免后期从 polling 切到中断模式时再返工。

---

## 六、用户参数从 `0x10` 开始，指针参数一定按 64 位处理

控制寄存器之后，才轮到用户参数区。

### 6.1 布局规则

最常见的规则可以概括成三句：

- 参数区从 **`0x10`** 开始。
- 普通 32 位 scalar 参数占 **1 个寄存器**。
- 全局 buffer 指针参数占 **2 个寄存器**，即低 32 位和高 32 位。

例如 kernel 原型可以抽象成：

```cpp
kernel(int* in0, int* out, int size, int mode)
```

那么比较典型的寄存器布局可能是：

| 偏移 | 内容 |
| --- | --- |
| `0x10` | `in0[31:0]` |
| `0x14` | `in0[63:32]` |
| `0x18` | `out[31:0]` |
| `0x1C` | `out[63:32]` |
| `0x20` | `size` |
| `0x24` | `mode` |

如果中间出现对齐填充，也必须在 `kernel.xml` 和 RTL 里保持一致。

### 6.2 指针参数必须按 64 位看待

即使你当前的板卡实际地址空间远小于 64 位，XRT 传进来的依然是设备侧地址的 64 位表达。

因此下面这种偷懒做法非常危险：

- RTL 内部只接低 32 位地址；
- 高 32 位直接忽略；
- 或者把指针参数当成一个 32 位 scalar。

在某些小实验里它可能“刚好能跑”，但这不是正确契约，一旦设备地址分配跨到更高位，就会立即出现读写错误。

### 6.3 最稳妥的做法是在启动时锁存参数

虽然 XRT 正常情况下不会在 kernel 运行中途改参数，但从硬件设计角度，最好仍然在 `ap_start` 触发时把参数寄存器锁存到内部影子寄存器，然后整个执行期只使用影子值。

这样做有两个好处：

- 内部地址生成与控制口解耦，时序更清晰。
- 以后若要支持更复杂调度，也不容易被寄存器在线变化影响。

---

## 七、`m_axi` 端口决定你怎样访问 HBM / DDR，不只是“把地址线拉出来”

### 7.1 `m_axi` 是标准 AXI4 full，而不是精简自定义总线

`m_axi_<port>` 端口必须提供 AXI4 full 所需的完整读写通道，包括：

- `AW*`
- `W*`
- `B*`
- `AR*`
- `R*`

这里有几个经常被忽视但很关键的点：

- `AWADDR` / `ARADDR` 应按 **64 位** 处理；
- `AWLEN` / `ARLEN` 是 burst 长度控制，不是可有可无；
- `WLAST` / `RLAST` 必须与最后一个 beat 精确对齐；
- `WSTRB` 宽度必须等于 `WDATA / 8`。

### 7.2 数据位宽不要和控制口混淆

控制口是 32 位，不代表 `m_axi` 也该做成 32 位。

对 Alveo 尤其是 HBM 平台来说，过窄的数据位宽会直接把带宽压没。工程上更常见的选择是：

- 128 位
- 256 位
- 512 位

当然，最终还要看平台允许的宽度以及你内部数据通路是否值得扩宽。

### 7.3 AXI burst 的基本约束必须自己守住

平台不会替你的 kernel 修正不合法事务。至少要保证：

- `AWBURST` / `ARBURST` 使用 `INCR`
- burst 不跨 4KB 边界
- 复位后所有 `VALID` 清零
- 不在复位过程中遗留 outstanding transaction

很多“偶现卡死”或者“数据偶尔错一片”的问题，根源并不在算术单元，而在 AXI 主端口状态机上。

---

## 八、`kernel.xml` 不是附属材料，而是 XRT 识别参数的唯一说明书

对于手写 RTL kernel，`kernel.xml` 的作用非常像一张“驱动说明书”。XRT 并不会去理解你的内部 RTL 逻辑，它看到的主要是这份元数据。

一个最小可用的 `kernel.xml` 至少要描述三件事：

1. kernel 名字是什么；
2. 暴露了哪些端口；
3. 每个参数对应哪个端口、什么偏移、什么大小。

例如：

```xml
<kernel name="acc_kernel" hwControlProtocol="ap_ctrl_hs" interrupt="true">
  <ports>
    <port name="s_axi_control" mode="slave" dataWidth="32"/>
    <port name="m_axi_gmem0" mode="master" dataWidth="256"/>
    <port name="m_axi_gmem1" mode="master" dataWidth="256"/>
  </ports>

  <args>
    <arg name="in0"  id="0" port="m_axi_gmem0" offset="0x10" size="0x8" addressQualifier="1" type="int*"/>
    <arg name="out"  id="1" port="m_axi_gmem1" offset="0x18" size="0x8" addressQualifier="1" type="int*"/>
    <arg name="size" id="2" port="s_axi_control" offset="0x20" size="0x4" addressQualifier="0" type="int"/>
  </args>
</kernel>
```

这里最容易出大问题的，不是 XML 语法，而是**数值和 RTL 真值不一致**。

比如：

- XML 里 `offset="0x20"`，RTL 实际在 `0x24` 译码；
- XML 里把指针挂到 `m_axi_gmem0`，RTL 实际用的是另一套端口；
- XML 里声明 `ap_ctrl_hs`，RTL 却没有实现标准 CTRL/GIER/IER/ISR 语义。

这类错误的可怕之处在于：工具链很多时候不会第一时间替你报出本质问题。最常见的表现是：

**主机程序正常跑完调用流程，但硬件行为完全不对。**

---

## 九、把主机侧一次调用，逐项映射回 RTL 行为

如果把调试思路再收紧一点，可以把一次 `kernel(...)` 调用拆成下面这个观察序列：

```text
Host                                    Kernel
  |                                        |
  |-- write args (0x10, 0x14, ...)  ------>|
  |-- write GIER = 0x1               ---->|
  |-- write IP_IER = 0x1             ---->|
  |-- write CTRL = 0x1               ---->|
  |                                        | ap_start latched
  |                                        | ap_idle = 0
  |                                        | ... work ...
  |                                        | ap_done pulse
  |                                        | ISR[0] = 1
  |                                        | interrupt = 1
  |<-- MSI -------------------------------|
  |-- read / clear ISR                --->|
  |-- read CTRL                      --->|
  |-- run.wait() returns                  |
```

这段序列的价值在于，它能直接指导排障。

如果 `run.wait()` 卡住，优先检查：

1. `ap_done` 是否真的产生过；
2. `ISR` 是否被置位；
3. `interrupt` 是否真的拉高；
4. 主机是否成功 W1C 清掉了中断状态；
5. `CTRL` 回读值里 `ap_idle` / `ap_done` 是否符合预期。

如果结果全错，优先检查：

1. `kernel.xml` 的参数偏移；
2. 指针高 32 位是否被正确拼接；
3. `m_axi` 地址生成是否跨 4KB；
4. `WSTRB` / `WLAST` 是否正确；
5. 读写端口和 arg 绑定是否搞反。

---

## 十、一个最小充分实现应该至少满足什么

如果把一切压缩成一份“交付前自检单”，我会优先看下面这些点。

### 10.1 顶层接口

- `ap_clk` 存在，且是 kernel 唯一主时钟
- `ap_rst_n` 存在，低有效，同步复位
- `s_axi_control` 为完整 AXI4-Lite slave
- `m_axi_<name>` 为完整 AXI4 full master
- `interrupt` 存在

### 10.2 控制寄存器语义

- `CTRL[0]` 能正确触发启动，不是永久 level 高
- `CTRL[1]` 能反映一次完成事件，并具备清零机制
- `CTRL[2]` 在运行前后正确表示 idle
- `CTRL[3]` 在简单 kernel 上至少能与 done 协调一致
- `GIER / IP_IER / IP_ISR` 具备标准读写语义

### 10.3 参数与元数据

- 参数从 `0x10` 开始布局
- 指针参数按两个 32 位寄存器处理
- `kernel.xml` 的 `offset` 与 RTL 译码完全一致
- `kernel.xml` 的端口名与顶层前缀完全一致
- `hwControlProtocol="ap_ctrl_hs"` 与 RTL 行为一致

### 10.4 AXI 主端口行为

- 地址位宽、数据位宽、`WSTRB` 匹配
- burst 类型正确
- `WLAST` / `RLAST` 正确
- 复位期间无非法 `VALID`

如果这些都能对齐，哪怕 kernel 内部算法还很简单，也已经具备了“被 XRT 当成正常 kernel 驱动”的最低条件。

---

## 十一、为什么这类问题在 RTL kernel 上特别常见

HLS kernel 在这件事上有一个天然优势：很多控制寄存器语义、AXI-Lite 从接口到 `ap_ctrl_hs` 行为，工具都会自动生成。

而手写 RTL kernel 的自由度更高，但代价就是：

- 接口命名要自己守；
- 寄存器语义要自己守；
- `kernel.xml` 偏移要自己守；
- AXI 主端口协议细节也要自己守。

因此 RTL kernel 的难点往往不在“会不会写状态机”，而在“能不能把一套看似分散的约束，完整地当成一个驱动契约来实现”。

这也是为什么很多设计单看 RTL 仿真没问题，到了 XRT 环境里却表现异常。因为仿真里你验证的是“逻辑会不会算”，而上板后 XRT 真正在验证的是：

**你是不是一个符合平台契约的 kernel。**

---

## 十二、总结

Vitis RTL kernel 能不能在 Alveo 上被 XRT 正常驱动，关键不在于它是不是“能综合、能打包”，而在于它是否严格满足了一整套接口与行为契约。

这套契约里，最关键的四个部分分别是：

- 顶层端口命名与 AXI 信号集完整性；
- `s_axi_control` 前 16 字节控制寄存器的标准语义；
- 用户参数区偏移与 64 位指针处理方式；
- `kernel.xml` 对参数和端口关系的精确描述。

如果你现在正遇到“`package_xo` 成功、`v++` 成功、程序也能打开 kernel，但 `run.wait()` 卡死”这类问题，那么优先不要怀疑算法本身，先沿着本文的握手链路逐段核查：

- 主机到底写了哪些寄存器；
- RTL 到底回了什么状态；
- 中断到底有没有被正确拉起和清掉；
- `kernel.xml` 和 RTL 到底是不是同一份事实。

把这些基础契约先对齐，后面的性能优化、并发访存、多 CU 扩展，才有可靠的地基。
