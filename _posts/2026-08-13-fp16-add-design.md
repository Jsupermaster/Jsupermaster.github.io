---
title: "浮点计算单元详细设计：FP16 加法器"
date: 2026-08-13 12:00:00 +0800
description: 从 IEEE 754 FP16 格式、八级流水线到 CARRY8 与 DSP48E2 资源映射，记录自研 FP16 加法单元的实现、优化和时序结果。
categories:
  - FPGA
home_category: engineering-practice
tags:
  - FPGA
  - Verilog
  - FP16
  - Floating Point
  - DSP48E2
  - CARRY8
article_kicker: FPGA DESIGN NOTE
word_count: 约 13k 字
read_time: 约 45 分钟
---

# 浮点计算单元详细设计文档：FP16_ADD

## 0.前言

浮点计算单元是现代处理器/NPU中常见的基础计算ip。如果考虑在FPGA上的实现情况，Xilinx官方提供了Floating Point IP，其提供了丰富的各类浮点计算。然而，考虑到后续自定义更加丰富的内容实现，以及后续迁移到其他FPGA板卡、改为ASIC实现、使用开源工具链验证等场景，使用自研ip具有一定的必要性。

同时借此机会，详细分析一下xilinx ip的实现细节、如何更好地利用FPGA底层资源优化设计，记录一下从功能的实现到不断优化的全过程。本文档可以用于指导浮点计算单元（FP16 加法单元）实现。

本次共提供了4个版本的ip可供选择，分别是纯逻辑版本、纯逻辑+carry8版本、1dsp版本和2dsp版本，在以下实验环境中与官方ip实现的资源和时序对比结果见下表。自研版本数据为当前RTL重新综合、布局布线后的结果。

> 实验环境：
>
> - 器件：xcvu9p-flga2104-2L-e
> - Vivado：2024.2
> - 时钟约束：3.333 ns，约 300 MHz
> - 接口延迟：8 拍（纯逻辑），11拍（dsp版本）

| 实现              | 延迟 | CLB LUT | 相对官方 | Logic/SRL |   FF | CARRY8 |  DSP |  WNS/WHS(300MHz) | 最差 Data Path Delay |
| ----------------- | ---: | ------: | -------: | --------: | ---: | -----: | ---: | ---------------: | -------------------: |
| 官方纯逻辑        |    8 |     156 |     基线 |    145/11 |  231 |     12 |    0 | +2.136/+0.047 ns |             1.080 ns |
| 自研纯逻辑        |    8 |     175 |      +19 |     166/9 |  213 |      3 |    0 | +1.652/+0.033 ns |             1.559 ns |
| 自研纯逻辑-carry8 |    8 |     150 |       -6 |     141/9 |  214 |      7 |    0 | +1.390/+0.019 ns |             1.723 ns |
| 官方1DSP          |   11 |     115 |     基线 |     113/2 |  289 |     10 |    1 | +2.069/+0.039 ns |             1.148 ns |
| 自研1DSP          |   11 |     131 |      +16 |     122/9 |  236 |      1 |    1 | +1.685/+0.039 ns |             1.518 ns |
| 官方2DSP          |   11 |      84 |     基线 |      83/1 |  227 |      7 |    2 | +2.221/+0.039 ns |             1.062 ns |
| 自研2DSP          |   11 |      92 |       +8 |      83/9 |  188 |      1 |    1 | +1.829/+0.033 ns |             1.233 ns |

由上表可见，纯逻辑版本中，carry使用不足会增加LUT。显式使用carry8后，LUT由175降至150，但CARRY8由3增至7，最差数据路径由1.559 ns增至1.723 ns，WNS由1.652 ns降至1.390 ns。使用DSP后，LUT继续减少，但DSP内部寄存器和数据对齐流水会增加FF。当前1DSP版本比纯逻辑版本少44个LUT、增加23个FF；2DSP版本少83个LUT、减少25个FF。DSP版本的SRL均为9个，说明本次LUT差异主要来自逻辑LUT，而不是SRL。

```verilog
(* shreg_extract = "no" *)
```

以禁用SRL优化。这样会减少SRL、增加FF，但不会减少总LUT，只有在需要用FF换取布局或时序时才有意义。

时序上，carry8版本的资源减少伴随一定时延增加；1DSP和2DSP版本的最差数据路径分别为1.518 ns和1.233 ns，均满足300 MHz目标。当前结果均为同一器件、同一时钟约束下的布局布线结果。

以下是详细设计方法和迭代、修正记录。

## 1.基础知识

为了更好地进行ip设计，我们有几个基础知识需要进行铺垫。

### 1.1 浮点数构成规则

根据IEEE标准，浮点数由符号位、指数位和尾数位构成。NPU中常见的三种浮点数如下：

![FP16、FP32 与 BF16 浮点格式字段划分](/assets/images/fp16_add/FP16_32_BF16.png)

浮点数转换为十进制的规则为：
$$
\text{Value} = (-1)^{\text{sign}} \times (1.\text{fraction}) \times 2^{(\text{exponent} - \text{bias})}
$$
其中，**偏置bias值**为：$$2^{w-1}-1$$。

浮点数中包含一些特殊情况，这在我们的计算过程中需要特别注意。

| 特殊情况                        | 数值表现        | 计算情况              |
| ------------------------------- | --------------- | --------------------- |
| Inf（无穷大，有符号）           | 指数全1,尾数全0 | 同号为无穷，异号为Nan |
| Nan（非数，一般符号无意义）     | 指数全1,尾数非0 | 任何计算均为Nan       |
| Zero（零，严格上有符号）        | 指数全0,尾数全0 | 正常参与计算          |
| Subnormal（非正规数，极小的数） | 指数全0,尾数非0 | 依据不同策略执行      |

尾数的隐含位，由指数位是否全0判断。指数位全0代表非正规数或者0,对非正规数而言，尾数隐含位是0，对于0而言数值上也是0，但没有逻辑意义。

浮点数拥有自己的舍入模式，一般有最近偶数舍入、四舍五入等，一般cpu中的强通用浮点计算单元支持各类不同舍入模式，以满足灵活性和精准性。对于NPU而言，一般固定一种舍入模式就可以，对精度影响较小。

### 1.2 FPGA底层资源

以xilinx ultrascale+为例，在官方文档AMD UG574(https://docs.amd.com/r/en-US/ug574-ultrascale-clb)中已经给出了详细信息，即一个典型 CLB 包含：

  - 2 个 Slice
  - 8 个 6 输入 LUT
  - 16 个触发器
  - 2 个 CARRY8 进位链资源
  - LUT 间的专用宽函数复用结构
  - Slice 内部及相邻 CLB 间的专用连接资源

Slice额外分为SLICEL和SLICEM两类，SLICEL支持LUT、触发器、宽函数组合、进位链；SLICEM额外支持分布式RAM（Distributed RAM，LUTRAM）和移位寄存器（Shift Register，SRL）。CLB是FPGA底层最基本的资源单元，用于实现大部分的组合和时序逻辑电路，是FPGA中的最核心资源。除此之外，还有block ram（BRAM），dsp等用于存储和计算的资源。

在浮点计算单元设计过程中，核心是充分利用CLB资源和dsp资源，以实现使用最少的总资源平衡计算效率、时序与FPGA实现效率。

vivado会根据verilog写法，在可能的情况下会自动推断将一些计算抽象为DSP，CARRY或SRL。为了充分利用这些底层资源，可以通过尽量优化verilog来引导vivado自动推断，也可以显式通过原语等方式调度这些资源。

这些底层资源各自适合的电路结构和优势如下：

| 资源        | 最适合的电路结构                          | 主要优势                       | 主要限制                                 |
| :---------- | :---------------------------------------- | :----------------------------- | :--------------------------------------- |
| **LUT**     | 任意组合逻辑、译码、MUX、小规模运算       | 灵活性最高                     | 逻辑级数和布线会影响时序                 |
| **FF**      | 状态保存、流水线、计数器寄存器            | 提供清晰的时序边界             | 数量和时钟资源有限                       |
| **SRL**     | 移位寄存器、延迟线、小FIFO、FIR延迟链     | 用很少LUT实现较深的延迟        | 通常没有运行时复位，不能替代任意FF       |
| **CARRY8**  | 加法器、减法器、计数器、比较器、地址递增  | 专用进位链，算术速度快         | 只能处理特定进位结构，宽度增加后仍有延迟 |
| **DSP48E2** | 乘法、乘加、MAC、FIR、矩阵/向量乘、宽加法 | 专用乘法器和级联通道，吞吐率高 | 数量固定、位置受限、通常需要流水线       |

其中SRL其实是SLICEM的一部分，使用SRL也会被计算到CLB LUTs中。CARRY8和DSP48E2都不会消耗LUT资源。

SRL的核心作用是作为移位寄存器，为少量数据的延时（如小FIFO）提供深延迟，用少量LUT换取大量FF。

CARRY8和DSP48E2的作用都是进行逻辑运算，以加快计算速度并减少LUT使用，CARRY8适合较简单的小型运算，DSP48E2则可以完成较为负责的运算。

也就是说：如果想省LUT，可以用FF替代SRL，用CARRY8和DSP48E2替代一部分计算逻辑。如果想节省其他资源则反之。

FPGA的资源利用核心就是把不同的资源统筹利用，互相替代交换，以追求更高的利用效率。一般来说在小型FPGA板卡上节约LUT是比较关键的，因为FF数量是LUT的两倍，同时LUT还被用于综合后的布线资源，如果LUT剩余较少会导致布线困难，时间长或时序不佳。

具体的调用方法在后续的资源优化篇章展开。

## 2.浮点加法单元实现（纯逻辑版）

### 2.1 基本实现原理

浮点加法单元主要有以下流程，可以划分为8个流水级：

1. 特殊值检测与输入拆分
  2. 尾数大小比较
  3. 指数相减、选择大数和小数
  4. 小数尾数右移对阶
  5. 尾数加法或减法
  6. 前导零检测
  7. 规格化
  8. 舍入和结果打包

以两个实际的浮点数相加为例，推导计算流程如下：

以1.5 + 0.25为例，根据FP16浮点数构成规则，这两个数据的编码为：

```
FP16数据编码规则：
[15]     sign
[14:10]  exponent，偏置值 bias = 15
[9:0]    fraction

正规数的值：(-1)^sign × 1.fraction × 2^(exponent - 15)
a:1.5  = 0 01111 1000000000 = 16'h3e00
b:0.25 = 0 01101 0000000000 = 16'h3400

a_sign = 0, a_exp = 15, a_frac = 1000000000
b_sign = 0, b_exp = 13, b_frac = 0000000000
```

#### 2.1.1 特殊值检测与输入拆分

先对两个输入数据判断是否为Nan/Inf/0/Subnormal四类特殊值。如果是特殊值，那么需要根据特定规则旁路计算。例如只要有一个输入数据是Nan，那么输出一定为Nan，直接置位特殊标记并随流水线传递，在最后一个流水级旁路输出即可。

由于这两个输入数据并非特殊值，所以正常按照FP16编码构成拆分出符号位、指数位和尾数位。

#### 2.1.2 尾数大小比较

之后，这一步需要对尾数比较大小。然而考虑到下个阶段需要进行指数相减，因此实际应当判断两个数据的绝对值大小。因此可以直接比较指数和尾数大小，从而判断两数的绝对值大小，便于后续处理。

在这一步，比较之后较大的数`big=1.5`，`small=0.25`。

#### 2.1.3 指数相减

较大的数的指数减较小的数的指数，得到`15-13=2`。

同时在这一步进行尾数拓展，补齐高位的隐藏位1,同时在低位拼接3位精度预留：

```
big_frac   = {1, 1000000000, 000}
             = 11000000000000

small_frac = {1, 0000000000, 000}
         = 10000000000000

最低 3 位预留给：

[2] Guard
[1] Round
[0] Sticky
```

这一操作是为了后续的计算舍入做准备。

#### 2.1.4 对阶

这一操作是为了对齐两个数据的指数位，需要将较小数的尾数右移【两数的指数差】位，在这一示例中即需要右移2位。根据保留位的计算原理，G和R都取移位后的低2位，而S位需要取所有被移出位的或。即：

```
假设右移d位：
G = x[d+2]
R = x[d+1]
S = x[d] | x[d-1] | ... | x[0]
```

这一步骤是充分保留浮点数信息、保障精度的关键操作。

#### 2.1.5 尾数加法/减法

根据两个数的符号位，判断进行加法还是减法。如果是同号则相加，异号则相减。同时需要判断是否产生最高位进位。如果产生最高位进位，需要对指数+1。在这一实例中：

```
  11000000000000
+ 00100000000000
  ----------------
  11100000000000

结果为：

1.1100000000 × 2^0
```

没有产生最高位进位，所以指数仍为15。

#### 2.1.6 前导零检测

需要检测结果最高位的1的位置。这一操作是因为将中间计算结果规格化为标准FP16数据，需要保证尾数最高位是1，因此需要通过判断前面0的个数，判断需要左移的位数。

这一实例中，位数最高位是[13]，因此前导0为0，不需要进行左移。

#### 2.1.7 规格化

在这一步，将之前的计算中间结果规格化为：

```
sign = 0
exp  = 15
frac = 1100000000

也就是：

1.1100000000 × 2^(15 - 15)
= 1.75
```

#### 2.1.8 舍入和打包

判断GRS位，并进行舍入。舍入规则如下：

| 舍入模式                                | 是否进位                                                     |
| --------------------------------------- | ------------------------------------------------------------ |
| 向零舍入                                | 永不进位                                                     |
| 向正无穷舍入                            | 正数且被截部分非零时进位                                     |
| 向负无穷舍入                            | 负数且被截部分非零时进位                                     |
| 最近值，平局远离零                      | G=1 时进位                                                   |
| 最近值，平局取偶数（最近偶数舍入，RNE） | G=0不进位；G=1且R或S为1进位；G=1且R,S=0，尾数有效位的最低位为1进位 |

在本实现中使用最近偶数舍入。这一实例中无需进位，最终打包结果与规格化结果相同。

```
1.5 + 0.25 = 1.75
输出 = 16'h3f00
```

### 2.2 纯逻辑版代码实现

在代码编写中，共划分8个流水级。依据调用情况，设计了纯逻辑实现、充分利用carry chain版本、利用1dsp版本和利用2dsp版本。代码实现梳理以纯逻辑版本为例，其他版本在改进中说明。

本模块特性为：不处理subnormal数据（使用ftz，flush_to_zero方式处理，即输入subnormal被处理为0，输出subnormal被规范化为0）、无反压、无阻塞、8周期延时、流水线化计算、无效输入时输出不固定。

#### 2.2.1 模块定义

```verilog
module fp16_add (
    input           clk             ,
    input           rst_n           ,
    input   [15:0]  a_input_data    ,
    input   [15:0]  b_input_data    ,
    input           input_valid     ,

    output  [15:0]  output_data     ,
    output          output_valid

);
```

模块拥有时钟、复位接口，两个16位数据输入端口和输入有效输入端口。一个数据输出端口和数据输出有效端口。当输入有效拉高时，接受a和b两个输入数据，8周期后输出有效拉高，输出有效数据。支持流式输入与任意气泡插入。输出有效为低时，输出数据可能是任意值（不保证是0），请注意这一点对您的设计的影响。

#### 2.2.2 p0:特殊值检测与输入拆分

首先拆分输入数据，并判断特殊情况。

定义四类情况，即无特殊、输出为Nan、输出为+Inf和输出为-Inf。

由于使用ftz规则处理，只需判断指数全0则都统一判断为0。

输出为Nan的有3种情况，即a是Nan，b是Nan，a和b都是inf且符号不一致。

输出为Inf的需要区分符号，正无穷的符号位是0，负无穷的符号位是1。根据a和b是否为inf判断取a和b谁的符号用于判断，由于之前已经排除了a和b均为inf且符号不一致的情况，因此这里使用优先级编码器判断。

```verilog
localparam [1:0] SPECIAL_NONE    = 2'b00;
localparam [1:0] SPECIAL_NAN     = 2'b01;
localparam [1:0] SPECIAL_INF_POS = 2'b10;
localparam [1:0] SPECIAL_INF_NEG = 2'b11;

integer i;
// Separate into sign, exponent, and fraction bits.
wire            a_sign = a_input_data[15];
wire    [4:0]   a_exp  = a_input_data[14:10];
wire    [9:0]   a_frac = a_input_data[9:0];

wire            b_sign = b_input_data[15];
wire    [4:0]   b_exp  = b_input_data[14:10];
wire    [9:0]   b_frac = b_input_data[9:0];

// Special case checking.
wire            a_is_zero_like  = ~|a_exp; // subnormal flush to zero
wire            a_is_inf        = (&a_exp) & (~|a_frac);
wire            a_is_nan        = (&a_exp) & (|a_frac);

wire            b_is_zero_like  = ~|b_exp; // subnormal flush to zero
wire            b_is_inf        = (&b_exp) & (~|b_frac);
wire            b_is_nan        = (&b_exp) & (|b_frac);

// Resolve special results while both original operands are available.
// nan +/- any -> nan
// +inf + -inf -> nan
// +inf + +inf -> +inf
// -inf + -inf -> -inf

wire [1:0] input_special_type =
    (a_is_nan | b_is_nan | (a_is_inf & b_is_inf & (a_sign != b_sign))) ? SPECIAL_NAN :
    (a_is_inf | b_is_inf) ? ((a_is_inf ? a_sign : b_sign) ? SPECIAL_INF_NEG : SPECIAL_INF_POS) : SPECIAL_NONE;
```

接着正式进入p0流水级，首先根据`is_zero_like`判断标志位，将0和subnormal都规范化为0。每一级只对必要控制信号进行复位，数据不复位，以减少复位信号扇出。

其余信号正常打拍，随后将两数exp和frac拼接，进行一次统一的比较大小。

```verilog
// pipeline 0: Special case handling.
reg             p0_ivalid       ;
reg     [1:0]   p0_special_type ;

reg             p0_a_sign       ;
reg     [4:0]   p0_a_exp        ;
reg     [9:0]   p0_a_frac       ;
reg             p0_b_sign       ;
reg     [4:0]   p0_b_exp        ;
reg     [9:0]   p0_b_frac       ;

wire    [14:0]  p0_a_compare    ;
wire    [14:0]  p0_b_compare    ;

wire            p0_a_ge_b       ;
reg             p0_a_is_zero_like;
reg             p0_b_is_zero_like;

always @(posedge clk) begin
    if (!rst_n) begin
        p0_ivalid       <= 1'b0;
        p0_special_type <= SPECIAL_NONE;
    end
    else begin
        p0_ivalid       <=  input_valid;
        p0_special_type <=  input_valid ? input_special_type : SPECIAL_NONE;

        p0_a_is_zero_like <= a_is_zero_like;
        p0_b_is_zero_like <= b_is_zero_like;

        p0_a_sign <= a_sign;
        p0_a_exp  <= a_is_zero_like ? 5'd0  : a_exp;
        p0_a_frac <= a_is_zero_like ? 10'd0 : a_frac;
        p0_b_sign <= b_sign;
        p0_b_exp  <= b_is_zero_like ? 5'd0  : b_exp;
        p0_b_frac <= b_is_zero_like ? 10'd0 : b_frac;
    end
end

assign p0_a_compare = {p0_a_exp, p0_a_frac};
assign p0_b_compare = {p0_b_exp, p0_b_frac};

assign p0_a_ge_b = p0_a_compare >= p0_b_compare;
```

#### 2.2.3 p1:承接大小比较结果并打拍

这一流水级从代码逻辑上来看是纯打拍，但从时序角度来说主要是平衡p0末尾的比较大小组合逻辑，这一设计可以更好地平衡时序。

```verilog
// pipeline 1: Register the canonical operands and magnitude comparison.
reg             p1_ivalid       ;
reg     [1:0]   p1_special_type ;

reg             p1_a_sign       ;
reg     [4:0]   p1_a_exp        ;
reg     [9:0]   p1_a_frac       ;
reg             p1_b_sign       ;
reg     [4:0]   p1_b_exp        ;
reg     [9:0]   p1_b_frac       ;
reg             p1_a_is_zero_like;
reg             p1_b_is_zero_like;
reg             p1_a_ge_b       ;

always @(posedge clk) begin
    if (!rst_n) begin
        p1_ivalid           <= 1'b0;
        p1_special_type     <= SPECIAL_NONE;
        p1_a_ge_b           <= 1'b0;
    end
    else begin
        p1_ivalid           <= p0_ivalid;
        p1_special_type     <= p0_special_type;

        p1_a_sign           <= p0_a_sign;
        p1_a_exp            <= p0_a_exp;
        p1_a_frac           <= p0_a_frac;
        p1_a_is_zero_like   <= p0_a_is_zero_like;
        p1_b_sign           <= p0_b_sign;
        p1_b_exp            <= p0_b_exp;
        p1_b_frac           <= p0_b_frac;
        p1_b_is_zero_like   <= p0_b_is_zero_like;
        p1_a_ge_b           <= p0_a_ge_b;
    end
end
```

#### 2.2.4 p2:指数相减、选择大数和小数

P2流水级主要做指数减法，根据之前的`p1_a_ge_b`信号选择是`a-b`还是`b-a`。

之后，根据`p1_a_ge_b`信号将a和b重新确定为较大数`big`和较小数`small`。同时对尾数进行拓宽，这里注意如果是0或subnormal则隐藏位应该拓展为0，其余情况则为1。末尾拓展3位以满足舍入精度。

```verilog
// pipeline 2: Subtract exponents and select the larger/smaller operand.
reg             p2_ivalid       ;
reg     [1:0]   p2_special_type ;

reg             p2_big_sign     ;
reg     [4:0]   p2_big_exp      ;
reg     [13:0]  p2_big_frac     ;
reg             p2_small_sign   ;
reg     [13:0]  p2_small_frac   ;
reg     [4:0]   p2_exp_sub      ;

always @(posedge clk) begin
    if (!rst_n) begin
        p2_ivalid       <= 1'b0;
        p2_special_type <= SPECIAL_NONE;
    end
    else begin
        p2_ivalid       <= p1_ivalid;
        p2_special_type <= p1_special_type;

        p2_exp_sub <= p1_a_ge_b ? (p1_a_exp - p1_b_exp) : (p1_b_exp - p1_a_exp);

        p2_big_sign   <= p1_a_ge_b ? p1_a_sign : p1_b_sign;
        p2_big_exp    <= p1_a_ge_b ? p1_a_exp  : p1_b_exp;
        p2_big_frac   <= p1_a_ge_b ? {~p1_a_is_zero_like, p1_a_frac, 3'b000} : {~p1_b_is_zero_like, p1_b_frac, 3'b000};
        p2_small_sign <= p1_a_ge_b ? p1_b_sign : p1_a_sign;
        p2_small_frac <= p1_a_ge_b ? {~p1_b_is_zero_like, p1_b_frac, 3'b000} : {~p1_a_is_zero_like, p1_a_frac, 3'b000};
    end
end
```

#### 2.2.5 p3:小数尾数右移对阶

p3阶段的核心运算是按照上一阶段得到的两数指数差，对较小数尾数右移，并计算舍入位。具体实现上，使用一个组合逻辑块完成sticky位的计算和右移结果的计算。随后在时序逻辑中进行打拍和拼接。

`sticky`位的计算是所有被移出的位的或，在移位时先直接计算移位结果，再把额外计算得到的`sticky`位拼接到最低位。

```verilog
// pipeline 3: Align the smaller fraction and generate Sticky.
reg             p3_ivalid       ;
reg     [1:0]   p3_special_type ;

reg             p3_big_sign     ;
reg     [4:0]   p3_big_exp      ;
reg     [13:0]  p3_big_frac     ;
reg             p3_small_sign   ;
reg     [13:0]  p3_small_frac   ;

reg     [13:0]  p3_shift_right_comb;
reg             p3_sticky_comb  ;

always @(*) begin
    p3_sticky_comb = 1'b0;

    for (i = 0; i < 14; i = i + 1) begin
        if (i <= p2_exp_sub)
            p3_sticky_comb = p3_sticky_comb | p2_small_frac[i];
    end

    p3_shift_right_comb = p2_small_frac >> p2_exp_sub;
end

always @(posedge clk) begin
    if (!rst_n) begin
        p3_ivalid       <= 1'b0;
        p3_special_type <= SPECIAL_NONE;
    end
    else begin
        p3_ivalid       <= p2_ivalid;
        p3_special_type <= p2_special_type;

        p3_big_sign     <= p2_big_sign;
        p3_big_exp      <= p2_big_exp;
        p3_big_frac     <= p2_big_frac;
        p3_small_sign   <= p2_small_sign;
        p3_small_frac   <= {p3_shift_right_comb[13:1], p3_sticky_comb};
    end
end
```

#### 2.2.6 p4:尾数加法或减法

p4阶段主要完成的是尾数的加法或减法。同号相加，异号相减。需要注意的是需要额外增加一个进位位，用于保存可能出现的进位。

```verilog
// pipeline 4: Add/subtract fractions by sign.

//   p4_add_sub[14]   : Carry bit
//   p4_add_sub[13:3] : Main fraction
//   p4_add_sub[2]    : Guard bit
//   p4_add_sub[1]    : Round bit
//   p4_add_sub[0]    : Sticky bit
reg             p4_ivalid       ;
reg     [1:0]   p4_special_type ;

reg             p4_big_sign     ;
reg     [4:0]   p4_big_exp      ;

reg     [14:0]  p4_add_sub      ;

always @(posedge clk) begin
    if (!rst_n) begin
        p4_ivalid       <= 1'b0;
        p4_special_type <= SPECIAL_NONE;
    end
    else begin
        p4_ivalid       <=  p3_ivalid;
        p4_special_type <=  p3_special_type;

        p4_big_sign     <= p3_big_sign      ;
        p4_big_exp      <= p3_big_exp       ;

        if (p3_big_sign == p3_small_sign)
            p4_add_sub <= {1'b0, p3_big_frac} + {1'b0, p3_small_frac};
        else
            p4_add_sub <= {1'b0, p3_big_frac} - {1'b0, p3_small_frac};
    end
end
```

#### 2.2.7 p5:前导零检测

p5的核心计算是前导零检测，目的是检测尾数中首个“1”前面有多少个0。前导零检测的实现使用一个简易函数，由于有进位的情况需要调整指数并对尾数右移，而前导零检测是用于左移调整尾数的，所以检测前导零时忽略尾数的最高位，即进位位。

```verilog
// pipeline 5: Compress to result sign, exponent, and add/sub magnitude.
reg             p5_ivalid       ;
reg     [1:0]   p5_special_type ;

reg             p5_sign         ;
reg     [4:0]   p5_exp          ;

reg     [14:0]  p5_add_sub      ;
reg     [3:0]   p5_lzc          ;

function [3:0] leading_zero_count;
    input [13:0] value;
    integer k;
    reg found;
    begin
        leading_zero_count = 4'd14;
        found = 1'b0;

        for (k = 13; k >= 0; k = k - 1) begin
            if (!found && value[k]) begin
                leading_zero_count = 4'd13 - k;
                found = 1'b1;
            end
        end
    end
endfunction

always @(posedge clk) begin
    if (!rst_n) begin
        p5_ivalid       <= 1'b0;
        p5_special_type <= SPECIAL_NONE;
        p5_lzc          <= 4'd0;
    end
    else begin
        p5_ivalid       <=  p4_ivalid;
        p5_special_type <=  p4_special_type;

        p5_sign         <= p4_big_sign;
        p5_exp          <= p4_big_exp;
        p5_add_sub      <= p4_add_sub;
        p5_lzc          <= leading_zero_count(p4_add_sub[13:0]);
    end
end
```

#### 2.2.8 p6:规格化

p6的主要计算是依据前导零检测结果，对尾数进行左移并给指数减去前导零检测结果；如果有进位，则对指数+1,尾数右移1.这里注意最低位的sticky仍然需要保留所有被移出数据的或。如果lzc的值大于指数值，注意保持指数为0。如果尾数相加减结果为全0，则输出也应为全0。

根据最近偶数舍入原则，确定是否进位，如果需要进位则在当前尾数结果基础上再+1。

```verilog
// pipeline 6: Normalize the finite result.
reg             p6_ivalid       ;
reg     [1:0]   p6_special_type ;

reg             p6_sign         ;
reg     [4:0]   p6_exp          ;
reg     [13:0]  p6_frac         ;

wire            p6_round_up        ;
wire    [10:0]  p6_frac_rounded    ;

always @(posedge clk) begin
    if (!rst_n) begin
        p6_ivalid       <= 1'b0;
        p6_special_type <= SPECIAL_NONE;
    end
    else begin
        p6_ivalid       <= p5_ivalid;
        p6_special_type <= p5_special_type;

        p6_sign <= p5_sign;

        if (p5_add_sub[14]) begin
            p6_exp  <= p5_exp + 5'd1;
            p6_frac <= {p5_add_sub[14:2], (p5_add_sub[1] | p5_add_sub[0])};
        end
        else if (~|p5_add_sub[13:0]) begin
            p6_exp  <= 5'd0;
            p6_frac <= 14'd0;
        end
        else if (p5_exp <= p5_lzc) begin
            p6_exp  <= 5'd0;
            p6_frac <= 14'd0;
        end
        else begin
            p6_exp  <= p5_exp - p5_lzc;
            p6_frac <= p5_add_sub[13:0] << p5_lzc;
        end
    end
end

assign p6_round_up = p6_frac[2] && (p6_frac[1] || p6_frac[0] || p6_frac[3]);

assign p6_frac_rounded = {1'b0, p6_frac[12:3]} + (p6_round_up ? 11'd1 : 11'd0);
```

#### 2.2.9 p7:最终舍入和结果打包

p7根据最开始确定的特殊情况，确认是否把输出直接旁路为特殊值。如果在计算过程中指数位变成了全1或者上一阶段舍入进位后是全1，在此处修正为inf。如果需要向指数位进位则给指数+1，如果指数全0则输出0，其余情况下则正常输出结果。

最后把最终结果连接到output。

```verilog
// pipeline 7: Round to nearest, ties to even.
reg             p7_ivalid       ;
reg     [15:0]  p7_data         ;

always @(posedge clk) begin
    if (!rst_n) begin
        p7_ivalid       <= 1'b0;
        p7_data         <= 16'd0;
    end
    else begin
        p7_ivalid       <= p6_ivalid;

        if (p6_special_type == SPECIAL_NAN) begin
            p7_data <= 16'h7e00;
        end
        else if (p6_special_type[1]) begin
            p7_data <= {p6_special_type[0], 5'b11111, 10'b0};
        end
        else if ((&p6_exp) || (p6_exp == 5'd30 && p6_frac_rounded[10])) begin
            p7_data <= {p6_sign, 5'b11111, 10'b0};
        end
        else if (p6_frac_rounded[10]) begin
            p7_data <= {p6_sign, p6_exp + 5'd1, 10'd0};
        end
        else if (~|p6_exp) begin
            p7_data <= {p6_sign, 15'd0};
        end
        else begin
            p7_data <= {p6_sign, p6_exp, p6_frac_rounded[9:0]};
        end
    end
end

// p7 is the final registered output stage.
assign output_data  = p7_data;
assign output_valid = p7_ivalid;

endmodule
```

### 2.3 纯逻辑版模块改进

以上是完整的纯逻辑版本代码，从笔者最开始的初代代码，到形成最终版代码过程中，对于资源使用和时序进行了一定优化。最开始的初代代码中，经vivado综合后总计使用`LUT=256`个，最终版代码则使用`LUT=177`个，经过各种优化减少了`30%`左右的LUT占用，FF占用略有提升（多使用了约20个）。由于FPGA中FF资源是LUT的两倍，因此LUT使用是核心的资源使用。

#### 2.3.1 资源优化——修改指数和尾数比较写法

首先，把综合后的网表交给AI做资源分布分析，确认全局的资源热点。随后从代码中寻找可能的优化空间。以下将笔者实践过的所有确实有效的优化点记录如下。

原代码：

```verilog
assign a_ge_b = (a_exp > b_exp) || (a_exp == b_exp) && (a_frac >= b_frac)
```

修改后代码：

```verilog
assign p0_a_compare = {p0_a_exp, p0_a_frac};
assign p0_b_compare = {p0_b_exp, p0_b_frac};

assign p0_a_ge_b = p0_a_compare >= p0_b_compare;
```

这一简单的改动带来了`22LUT`的资源节省。通过将指数和尾数先拼接为一个完整的数再直接比较更有利于EDA进行优化，而上面的写法是常规思路中很容易写出的代码。着一个简单的改动就可以带来有效的资源减少。当前的优化进度：`LUT=234(SRL=22)`，`FF=196`。

#### 2.3.2 资源优化——移除条件清零

在最开始的版本中，笔者对每一阶段流水线进行了条件清零，即只判断为有效输入才开启这一流水级的计算。从逻辑上这样更加严格，但带来了额外的资源使用。去掉这一部分并不会影响结果的正确性，然而能够提升资源利用效率，无需每个字段都生成MUX，只在最后输出阶段进行条件清零。

原代码：

```verilog
always @(posedge clk) begin
    if (!rst_n) begin
        p0_ivalid       <= 1'b0;
        p0_special_type <= SPECIAL_NONE;
    end
    else begin
        p0_ivalid       <=  input_valid;
        p0_special_type <=  input_valid ? input_special_type : SPECIAL_NONE;

        if (input_valid && (input_special_type == SPECIAL_NONE)) begin

            p0_a_is_zero_like <= a_is_zero_like;
            p0_b_is_zero_like <= b_is_zero_like;

            p0_a_sign_n <= a_sign;
            p0_a_exp_n  <= a_is_zero_like ? 5'd0  : a_exp;
            p0_a_frac_n <= a_is_zero_like ? 10'd0 : a_frac;
            p0_b_sign_n <= b_sign;
            p0_b_exp_n  <= b_is_zero_like ? 5'd0  : b_exp;
            p0_b_frac_n <= b_is_zero_like ? 10'd0 : b_frac;
        end

        else begin
            p0_a_sign_n <=  1'd0;
            p0_a_exp_n  <=  5'd0;
            p0_a_frac_n <=  10'd0;
            p0_b_sign_n <=  1'd0;
            p0_b_exp_n  <=  5'd0;
            p0_b_frac_n <=  10'd0;
        end
    end
end
```

修改后代码：

```verilog
always @(posedge clk) begin
    if (!rst_n) begin
        p0_ivalid       <= 1'b0;
        p0_special_type <= SPECIAL_NONE;
    end
    else begin
        p0_ivalid       <=  input_valid;
        p0_special_type <=  input_valid ? input_special_type : SPECIAL_NONE;

        p0_a_is_zero_like <= a_is_zero_like;
        p0_b_is_zero_like <= b_is_zero_like;

        p0_a_sign_n <= a_sign;
        p0_a_exp_n  <= a_is_zero_like ? 5'd0  : a_exp;
        p0_a_frac_n <= a_is_zero_like ? 10'd0 : a_frac;

        p0_b_sign_n <= b_sign;
        p0_b_exp_n  <= b_is_zero_like ? 5'd0  : b_exp;
        p0_b_frac_n <= b_is_zero_like ? 10'd0 : b_frac;
    end
end
```

实测结果中，综合后LUT节省了25个，其余资源基本不变。当前的优化进度：`LUT=209(SRL=22)`，`FF=195`。

#### 2.3.2 资源优化——移除p8、移除冗余控制信号

初版规划中使用了9级流水实现这一模块，写完，模块后我重新评估了各级流水所承担的实际作用，删除了最后的一个流水级，合并到上一个流水级中。同时优化了判断条件写法。

原代码：

```verilog
// pipeline 7: Round to nearest, ties to even.

reg             p7_ivalid       ;
reg     [15:0]  p7_data         ;

always @(posedge clk) begin
    if (!rst_n) begin
        p7_ivalid       <= 1'b0;
        p7_data         <= 16'd0;
    end
    else begin
        p7_ivalid       <= p6_ivalid;

        if (p6_special_type == SPECIAL_NAN) begin
            p7_data <= 16'h7e00;
        end
        else if (p6_special_type[1]) begin
            p7_data <= {p6_special_type[0], 5'b11111, 10'b0};
        end
        else if ((p6_exp >= 5'd31) || (p6_exp == 5'd30 && p6_frac_rounded[10])) begin
            p7_data <= {p6_sign, 5'b11111, 10'b0};
        end
        else if (p6_frac_rounded[10]) begin
            p7_data <= {p6_sign, p6_exp + 5'd1, 10'd0};
        end
        else if (p6_exp == 5'd0) begin
            p7_data <= {p6_sign, 15'd0};
        end
        else begin
            p7_data <= {p6_sign, p6_exp, p6_frac_rounded[9:0]};
        end
    end
end

// pipeline 8: Pack the final FP16 value.
reg     [15:0]  p8_data     ;
reg             p8_valid    ;

always @(posedge clk) begin
    if (!rst_n) begin
        p8_data  <= 16'd0;
        p8_valid <= 1'b0;
    end
    else begin
        p8_valid <= p7_ivalid;
        p8_data  <= p7_data;
    end
end

assign output_data  = p8_data;
assign output_valid = p8_valid;
```

修改后代码：

```verilog
// pipeline 7: Round to nearest, ties to even.
reg             p7_ivalid       ;
reg     [15:0]  p7_data         ;

always @(posedge clk) begin
    if (!rst_n) begin
        p7_ivalid       <= 1'b0;
        p7_data         <= 16'd0;
    end
    else begin
        p7_ivalid       <= p6_ivalid;

        if (p6_special_type == SPECIAL_NAN) begin
            p7_data <= 16'h7e00;
        end
        else if (p6_special_type[1]) begin
            p7_data <= {p6_special_type[0], 5'b11111, 10'b0};
        end
        else if ((&p6_exp) || (p6_exp == 5'd30 && p6_frac_rounded[10])) begin
            p7_data <= {p6_sign, 5'b11111, 10'b0};
        end
        else if (p6_frac_rounded[10]) begin
            p7_data <= {p6_sign, p6_exp + 5'd1, 10'd0};
        end
        else if (~|p6_exp) begin
            p7_data <= {p6_sign, 15'd0};
        end
        else begin
            p7_data <= {p6_sign, p6_exp, p6_frac_rounded[9:0]};
        end
    end
end

// p7 is the final registered output stage.
assign output_data  = p7_data;
assign output_valid = p7_ivalid;
```

另外，将一些多级比较改成 Vivado 更容易识别的归约逻辑，例如：

```verilog
~|a_exp       // 判断指数为零
&a_exp        // 判断指数全一
|a_frac       // 判断 fraction 非零
~|p1_exp_sub  // 判断指数差为零
```

  同时删除了一些由位宽和前置特殊值判断已经保证的冗余比较，例如：

```verilog
p6_exp >= 5'd31
p7_exp >= 5'd31
```

经过这一优化，节约了18LUT和17FF，目前的优化进度：`LUT=194(SRL=22)`，`FF=178`。

#### 2.3.3 资源与时序优化——p0~p3流水重排

随后根据综合、布局布线后的时序报告，发现初版p0~p3的流水线分配有一些不妥，导致最差路径较长。因此对p0~p3流水级进行了重排，规划任务如下：

1. p0→p1：输入规范化和 {exp, frac} 比较；
  2. p1→p2：指数差和大小数选择；
  3. p2→p3：对阶移位和 Sticky。

这一修改优化了时序，减少了13个SRL，2个逻辑LUT，因此总LUT减少15个，但FF增加了34个。后续优化了一处残留的重复比较，达到了最终`LUT=174(SRL=9)`，`FF=213`的资源使用。

### 2.4 纯逻辑版模块手动例化carry8优化

根据本文最开始的对比表，纯逻辑版本相较官方ip实现主要是少了carry8的使用，因此使用了更多的LUT。我尝试了各种方式引导vivado综合出更多carry8，但都没有效果，因此使用原语直接例化来实现。

首先需要确认的是哪里适合放置carry8进位链。

| 位置                       | 逻辑类型           | 适合程度 | 备注              |
| -------------------------- | ------------------ | -------- | ----------------- |
| p4 尾数加减                | 14 位加法/减法     | 高       | Vivado 已自动推断 |
| p5 Leading-Zero / 全零检测 | 分组全零前缀传播   | 较高     | Carry8版本已采用  |
| p6 舍入加一                | 尾数加一并传入指数 | 很高     | Carry8版本已采用  |
| p0 幅值比较                | 15 位大小比较      | 中       | Vivado 已自动推断 |
| p1 指数差                  | 5 位减法/绝对值    | 中低     | 位宽小，收益不大  |
| p6 指数加减                | 5 位加减           | 中低     | 位宽小，收益不大  |
| p3 对阶移位/Sticky         | 可变移位和 OR 网络 | 很低     | Carry8 不适合     |
| p7 特殊值分类              | 多路条件选择       | 很低     | Carry8 不适合     |

经过分析后，主要在p5 Leading-Zero / 全零检测和p6 舍入加一这两个部分通过手动例化carry8来增加carry8使用，共计使用`4Carry8`，节省`20LUT`。

#### 2.4.1 vivado-carry8使用方法

Xilinx UltraScale 的 CARRY8 是 8 位专用进位链，可用于：

  - 加法/减法；
  - 加一；
  - 无符号比较；
  - 前缀检测，例如连续零检测；
  - 部分计数和条件传播逻辑。

它是组合逻辑，没有时钟和复位。

```verilog
CARRY8 #(
  .CARRY_TYPE("SINGLE_CY8")
) u_carry (
  .CI     (carry_in),
  .CI_TOP (1'b0),
  .DI     (di[7:0]),
  .S      (s[7:0]),
  .O      (o[7:0]),
  .CO     (co[7:0])
);
```

核心关系可以理解为：

```verilog
O[i]  = S[i] ^ carry_in[i]
CO[i] = S[i] ? carry_in[i] : DI[i]
```

其中：

  - O：每一位的结果输出；
  - CO：每一位向高位传播的进位；
  - CI：最低位输入进位；
  - DI：generate 输入；
  - S：propagate 控制输入。

例如用于加法：对于A + B的按位计算实现算法为：`sum[i] = a[i] XOR b[i] XOR carry_in[i]`，其中`carry_in[i]`是低一位的进位标志。由此，我们得到使用CARRY8计算加法的连接方式：

```verilog
wire [7:0] add_o;
wire [7:0] add_co;

CARRY8 #(
  .CARRY_TYPE("SINGLE_CY8")
) u_add (
  .CI     (1'b0),
  .CI_TOP (1'b0),
  .DI     (b[7:0]),
  .S      (a[7:0] ^ b[7:0]),
  .O      (add_o),
  .CO     (add_co)
);
```

这里：假设计算A+B是独立存在的，因此没有外部来源的进位。这里的`a[i]^b[i]`直接连接到CARRY8的`S`端，也就是说底层仍然使用LUT计算出来结果，而CARRY8真正完成的是`S[i]^carry_in[i]`这个进位操作。这也是CARRY8作为快速进位链的核心用途。注意这里：

```verilog
carry_in[0] = CI
carry_in[i] = CO[i-1]   // i>0
```

也就是说`CI`代表的是最低位的进位，而`CO`代表的是`[i+1]`位进位。CARRY8内部自动处理了`[0]~[i]`的进位过程，唯一需要关注的是`carry_in[8]`即`CO[7]`，也就是`CO`的最高位进位，它没有在内部得到处理，属于进位的溢出位，为保证正确性应额外进行处理。

#### 2.4.2 CARRY8用于替代纯逻辑版本的部分LUT

首先是在p5，手动例化两颗CARRY8用于替换先前的lzc模块。代码如下：

```verilog
// pipeline 5: Compress to result sign, exponent, and add/sub magnitude.
reg             p5_ivalid       ;
reg     [1:0]   p5_special_type ;

reg             p5_sign         ;
reg     [4:0]   p5_exp          ;

reg     [14:0]  p5_add_sub      ;
reg     [3:0]   p5_lzc          ;
reg             p5_all_zero     ;

wire    [3:0]   p4_lze_chunk_zero;
wire    [7:0]   p4_lze_hi_o     ;
wire    [7:0]   p4_lze_hi_co    ;
wire    [7:0]   p4_lze_lo_o     ;
wire    [7:0]   p4_lze_lo_co    ;
wire    [1:0]   p4_lze_group    ;
reg     [13:0]  p4_lze_coarse_frac;
wire    [1:0]   p4_lze_fine     ;
wire    [3:0]   p4_lze_count    ;

// Split the 14-bit magnitude into 4/4/4/2 groups.  The carry-prefix
// results are shared by all-zero detection and coarse normalization.
assign p4_lze_chunk_zero[0] = ~|p4_add_sub[13:10];
assign p4_lze_chunk_zero[1] = ~|p4_add_sub[9:6];
assign p4_lze_chunk_zero[2] = ~|p4_add_sub[5:2];
assign p4_lze_chunk_zero[3] = ~|p4_add_sub[1:0];

CARRY8 #(
    .CARRY_TYPE("SINGLE_CY8")
) u_lze_carry_hi (
    .CI     (1'b1),
    .CI_TOP (1'b0),
    .DI     (8'b0),
    .S      ({6'b0, p4_lze_chunk_zero[1:0]}),
    .O      (p4_lze_hi_o),
    .CO     (p4_lze_hi_co)
);

CARRY8 #(
    .CARRY_TYPE("SINGLE_CY8")
) u_lze_carry_lo (
    .CI     (1'b1),
    .CI_TOP (1'b0),
    .DI     (8'b0),
    .S      ({6'b0, p4_lze_chunk_zero[3:2]}),
    .O      (p4_lze_lo_o),
    .CO     (p4_lze_lo_co)
);

// Preserve the unshifted magnitude on the addition-overflow path.
assign p4_lze_group = p4_add_sub[14] ? 2'd0 : {
    p4_lze_hi_co[1],
    p4_lze_hi_co[1] ? p4_lze_lo_co[0] : p4_lze_hi_co[0]
};

always @(*) begin
    case (p4_lze_group)
        2'd0: p4_lze_coarse_frac = p4_add_sub[13:0];
        2'd1: p4_lze_coarse_frac = {p4_add_sub[9:0], 4'b0};
        2'd2: p4_lze_coarse_frac = {p4_add_sub[5:0], 8'b0};
        default: p4_lze_coarse_frac = {p4_add_sub[1:0], 12'b0};
    endcase
end

assign p4_lze_fine = p4_lze_coarse_frac[13] ? 2'd0 :
                     p4_lze_coarse_frac[12] ? 2'd1 :
                     p4_lze_coarse_frac[11] ? 2'd2 : 2'd3;
assign p4_lze_count = {p4_lze_group, p4_lze_fine};

always @(posedge clk) begin
    if (!rst_n) begin
        p5_ivalid       <= 1'b0;
        p5_special_type <= SPECIAL_NONE;
        p5_lzc          <= 4'd0;
        p5_all_zero     <= 1'b0;
    end
    else begin
        p5_ivalid       <=  p4_ivalid;
        p5_special_type <=  p4_special_type;

        p5_sign     <= p4_big_sign;
        p5_exp      <= p4_big_exp;
        p5_add_sub  <= {p4_add_sub[14], p4_lze_coarse_frac};
        p5_lzc      <= p4_lze_count;
        p5_all_zero <= p4_lze_hi_co[1] & p4_lze_lo_co[1];

    end
end
```

纯逻辑版 p5 的核心是：

```verilog
p5_lzc <= leading_zero_count(p4_add_sub[13:0]);
p5_add_sub <= p4_add_sub;
```

其中：

```verilog
p4_add_sub[14]    : 加法溢出进位
p4_add_sub[13:0]  : 需要寻找最高位 1 的尾数结果
```

`leading_zero_count()` 从 bit13 向 bit0 扫描：

```verilog
for (k = 13; k >= 0; k = k - 1) begin
    if (!found && value[k]) begin
      leading_zero_count = 4'd13 - k;
      found = 1'b1;
    end
end
```

例如：

`p4_add_sub[13:0] = 0001xxxxxxxxxx`

最高位 1 在 bit10，因此：

`p5_lzc = 13 - 10 = 3`

p6 再使用这个完整的 p5_lzc：

`p6_frac <= p5_add_sub[13:0] << p5_lzc;`

也就是说，纯逻辑版的工作分布是：

p5：完整 14 位优先编码，生成 LZC
p6：根据 0~13 位 LZC 做完整动态左移

问题在于：

  - p5 的优先编码器较宽；
  - p6 还需要一个 14 位可变左移网络；
  - p5 的全零判断和 LZC 逻辑没有充分复用；
  - 逻辑路径主要由优先判断和大范围选择器构成。

Carry8 版没有让 CARRY8 直接输出完整的 4 位 LZC，而是把 14 位数据分成四个区域：

```verilog
[13:10]   4 bit
[9:6]     4 bit
[5:2]     4 bit
[1:0]     2 bit
```

对应代码：

```verilog
assign p4_lze_chunk_zero[0] = ~|p4_add_sub[13:10];
assign p4_lze_chunk_zero[1] = ~|p4_add_sub[9:6];
assign p4_lze_chunk_zero[2] = ~|p4_add_sub[5:2];
assign p4_lze_chunk_zero[3] = ~|p4_add_sub[1:0];
```

每个 `chunk_zero` 的含义是：

1：这一组全部为 0
0：这一组至少有一个 1

例如：

`p4_add_sub[13:0] = 0000 0000 1010 00`

则：

```
chunk_zero[0] = 1
chunk_zero[1] = 1
chunk_zero[2] = 0
chunk_zero[3] = 1
```

最高位非零组是第三组，也就是 [5:2]。

**使用两颗 CARRY8 分别处理两组：**

高两组使用：

```verilog
CARRY8 #(
  .CARRY_TYPE("SINGLE_CY8")
) u_lze_carry_hi (
  .CI     (1'b1),
  .CI_TOP (1'b0),
  .DI     (8'b0),
  .S      ({6'b0, p4_lze_chunk_zero[1:0]}),
  .O      (p4_lze_hi_o),
  .CO     (p4_lze_hi_co)
);
```

低两组使用：

```verilog
CARRY8 #(
  .CARRY_TYPE("SINGLE_CY8")
) u_lze_carry_lo (
  .CI     (1'b1),
  .CI_TOP (1'b0),
  .DI     (8'b0),
  .S      ({6'b0, p4_lze_chunk_zero[3:2]}),
  .O      (p4_lze_lo_o),
  .CO     (p4_lze_lo_co)
);
```

这里真正有用的是每颗 CARRY8 的低两位 `CO`：

`CO[0]`：当前第一组是否为零
`CO[1]`：前两组是否连续为零

由于：

```verilog
CI  = 1'b1;
DI  = 8'b0;
S   = chunk_zero
```

所以：

  - 当前组为零时，进位继续传播；
  - 当前组非零时，传播被截断；
  - `CO[1]` 就能表示前两组是否都为零。

> 注意：这里实际上carry8进位链只起到了一个“与”门的作用，但在实际测试中，确实可以大幅减少资源使用。经过分析，应该是vivado在识别到carry8之后可以进行更好的资源整合与优化，所以减少的LUT实际上并不是靠carry8取代了复杂的逻辑计算，而是引入carry8让vivado能够更加高效地分配与整合资源，下面的carry8也是如此。实际测试下，如果直接把这两个carry8写成&，逻辑功能没有变化，但LUT消耗增多23.

其次是p6部分，使用carry8进行了舍入+1的计算替换。

```verilog
// pipeline 6: Normalize the finite result.
reg             p6_ivalid       ;
reg     [1:0]   p6_special_type ;

reg             p6_sign         ;
reg     [4:0]   p6_exp          ;
reg     [13:0]  p6_frac         ;

wire            p6_round_up        ;
wire    [10:0]  p6_frac_rounded    ;
wire    [7:0]   p6_round_lo_o      ;
wire    [7:0]   p6_round_lo_co     ;
wire    [7:0]   p6_round_hi_o      ;
wire    [7:0]   p6_round_hi_co     ;
wire    [4:0]   p6_exp_round_o     ;

always @(posedge clk) begin
    if (!rst_n) begin
        p6_ivalid       <= 1'b0;
        p6_special_type <= SPECIAL_NONE;
    end
    else begin
        p6_ivalid       <= p5_ivalid;
        p6_special_type <= p5_special_type;

        p6_sign <= p5_sign;

        if (p5_add_sub[14]) begin
            p6_exp  <= p5_exp + 5'd1;
            p6_frac <= {p5_add_sub[14:2], (p5_add_sub[1] | p5_add_sub[0])};
        end
        else if (p5_all_zero) begin
            p6_exp  <= 5'd0;
            p6_frac <= 14'd0;
        end
        else if (p5_exp <= p5_lzc) begin
            p6_exp  <= 5'd0;
            p6_frac <= 14'd0;
        end
        else begin
            p6_exp  <= p5_exp - p5_lzc;
            p6_frac <= p5_add_sub[13:0] << p5_lzc[1:0];
        end
    end
end

assign p6_round_up = p6_frac[2] &&
                     (p6_frac[1] || p6_frac[0] || p6_frac[3]);
// The low eight fraction bits occupy carry positions 0..7.
CARRY8 #(
    .CARRY_TYPE("SINGLE_CY8")
) u_round_carry_lo (
    .CI     (p6_round_up),
    .CI_TOP (1'b0),
    .DI     (8'b0),
    .S      (p6_frac[10:3]),
    .O      (p6_round_lo_o),
    .CO     (p6_round_lo_co)
);

// Carry positions 0..1 finish the fraction.  Positions 2..6 add the
// fraction overflow into the five-bit exponent in the same chain.
CARRY8 #(
    .CARRY_TYPE("SINGLE_CY8")
) u_round_carry_hi_exp (
    .CI     (p6_round_lo_co[7]),
    .CI_TOP (1'b0),
    .DI     (8'b0),
    .S      ({1'b0, p6_exp, p6_frac[12:11]}),
    .O      (p6_round_hi_o),
    .CO     (p6_round_hi_co)
);

// CO[1] is the overflow out of mantissa bit 9.
assign p6_frac_rounded = {
    p6_round_hi_co[1], p6_round_hi_o[1:0], p6_round_lo_o
};
assign p6_exp_round_o = p6_round_hi_o[6:2];
```

这里的两个carry进行了级联，便于处理更大位数的计算。

注意：p5和p6引入carry8，导致了时序下降。这似乎是资源下降带来的取舍。TODO：探究为什么carry8能够使vivado综合更加高效，以及如何进一步降低延时。

## 3.浮点加法单元实现（DSP版）
### 3.1 DSP48E2基础使用

DSP48E2内部包含乘法器和ALU。本设计使用的主要数据通路为：

```text
P = A × B + C
```

通过`ALUMODE`可以将加法改为减法。因此，DSP不仅可以做乘法，也可以承担移位后的尾数加减和规格化后的舍入加法。

本设计中各端口的作用如下：

- `A`、`D`：输入尾数，`INMODE`用于选择A、D输入；
- `B`：移位因子。将移位量编码为one-hot数后，乘法器可以完成可变移位；
- `C`：另一操作数，或固定位置的指数、隐藏位和舍入控制；
- `OPMODE`：选择乘法器结果和C输入的组合方式；
- `ALUMODE`：选择加法或减法；
- `AREG`、`BREG`、`CREG`、`MREG`、`PREG`：配置内部寄存器级数。

寄存器配置决定A、B、C三路数据何时到达ALU。使用DSP时，不仅要对齐数值，还要对齐符号、指数和控制信号。

### 3.2 DSP适合替换的位置

浮点加法器中，适合使用DSP的部分主要有两处：

1. 对阶后的尾数加减。较小尾数先右移，再与较大尾数相加或相减；
2. 规格化和舍入。尾数根据前导零数左移，再加上舍入位，舍入进位同时影响指数。

前导零检测、特殊值判断、指数差计算和结果打包主要是比较、编码和位拼接，放入DSP的收益较小，保留在LUT中。

因此，1DSP版本只替换第一处，2DSP版本继续替换第二处。

### 3.3 1DSP版本

#### 3.3.1 对阶因子

对阶逻辑先判断指数差是否在尾数宽度内：

```verilog
wire        p1_shift_in_range = ~p1_exp_diff[4];
wire [15:0] p1_shift_factor = p1_shift_in_range ?
    (16'h8000 >> p1_exp_diff[3:0]) : 16'd0;
wire        p1_align_enable = p1_shift_in_range & ~p1_small_zero;
wire        p1_far_sticky   = ~p1_shift_in_range & ~p1_small_zero;
```

`p1_shift_factor`是one-hot右移因子。指数差超出范围时，小数直接变为零，只保留sticky信息。

#### 3.3.2 DSP输入组织

第一个DSP将小数乘以对阶因子，并把大数尾数放到C输入的对应位段：

```verilog
wire [29:0] dsp_a = {19'd0, 1'b1, p1_b_frac};
wire [26:0] dsp_d = {16'd0, 1'b1, p1_a_frac};
wire [17:0] dsp_b = {2'd0, p2_shift_factor};
```

大数尾数通过C输入左移15位：

```verilog
p2_dsp_c <= {22'd0, p1_big_sig, 15'd0};
p3_dsp_c <= p2_dsp_c;
```

DSP的等效计算为：

```text
P = small_sig × shift_factor + (big_sig << 15)
```

同号时执行加法，异号时通过`ALUMODE`执行减法。这样，对阶移位和尾数加减由同一个DSP完成。

#### 3.3.3 加减控制和流水

尾数大小关系、符号关系和操作数有效性仍由LUT逻辑判断。DSP接收已经完成选择的`big_sig`、`small_sig`及加减控制。

DSP内部寄存器使A、B、C三路延迟不同，因此需要分别安排：

- 尾数输入的流水；
- `p2_shift_factor`的流水；
- `p2_dsp_c`的流水；
- 加减模式及异常控制信号的流水。

这些信号必须与DSP输出`dsp_p`同拍到达后级。

#### 3.3.4 DSP输出

DSP输出取出扩展后的尾数和sticky信息：

```verilog
p6_add_sub <= {dsp_p[26:13],
               (~dsp_pattern_detect | p5_far_sticky)};
```

后续的前导零检测、规格化左移和舍入仍由LUT逻辑完成。1DSP版本只替换p3、p4的对阶和尾数加减逻辑。

当前版本综合、布局布线结果为：

```text
LUT = 131（Logic/SRL = 122/9），FF = 236，CARRY8 = 1，DSP = 1，延迟 = 11拍
```

### 3.4 2DSP版本

第一个DSP与1DSP版本相同。第二个DSP用于规格化左移、尾数舍入及舍入进位引起的指数调整。

#### 3.4.1 第一个DSP的输出

第一个DSP输出先整理为15位规格化输入：

```verilog
wire [14:0] norm_raw = {
    dsp_p[26:13],
    (~dsp_pattern_detect | p5_far_sticky)
};
```

前导零检测仍在LUT逻辑中完成：

```verilog
wire [12:0] norm_sig = dsp_p[26:14];
wire [3:0]  norm_lzc = leading_zero_count_13(norm_sig);
wire        norm_zero = ~|norm_sig;
```

#### 3.4.2 规格化因子

将前导零数转换为one-hot左移因子：

```verilog
wire [12:0] p6_shift_factor =
    p6_norm_zero ? 13'd0 : (13'd1 << p6_lzc);
```

第二个DSP用尾数乘以该因子实现可变左移。前导零检测本身没有放入DSP，因为它是优先编码结构，不适合使用DSP乘法器。

#### 3.4.3 指数和舍入控制

第二个DSP的C输入同时携带指数、隐藏位和舍入控制：

```verilog
wire [47:0] norm_dsp_c = {
    29'd0, p8_norm_exp, 1'b1, 10'd0,
    p8_round_up, 2'b00
};
```

其计算形式为：

```text
P = norm_sig × normalize_factor
  + exponent/hidden_bit/round_control
```

尾数左移后，`p8_round_up`在固定位置加一。若尾数舍入产生进位，进位可以直接传入指数路径，避免再使用一组独立的加法器。

#### 3.4.4 第二个DSP

第二个DSP的主要输入为：

```verilog
wire [29:0] norm_dsp_a = {17'd0, norm_sig};
wire [17:0] norm_dsp_b = {5'd0, p7_shift_factor};
wire [47:0] norm_dsp_c = {
    29'd0, p8_norm_exp, 1'b1, 10'd0,
    p8_round_up, 2'b00
};
```

其中A路为待规格化尾数，B路为one-hot规格化因子，C路为指数、隐藏位和舍入控制。DSP的乘法器完成左移，ALU完成舍入加法。

#### 3.4.5 第二个DSP的数据对齐

第二个DSP配置了不同的输入寄存器级数：

- A路`AREG = 2`；
- B路`BREG = 1`；
- C路`CREG = 1`。

因此，`p7_shift_factor`需要额外打一拍，`p8_norm_exp`和`p8_round_up`也要与C路对齐。否则会出现尾数来自当前事务、指数或舍入控制来自前一事务的错误。

#### 3.4.6 第二个DSP输出

第二个DSP输出中，指数和尾数字段位于固定位置：

```text
norm_dsp_p[18:14]：指数
norm_dsp_p[11:2] ：fraction
```

规格化和舍入完成后，后级只需处理异常情况并完成结果打包。

当前版本综合、布局布线结果为：

```text
LUT = 92（Logic/SRL = 83/9），FF = 188，CARRY8 = 1，DSP = 2，延迟 = 11拍
```

### 3.5 1DSP和2DSP版本对比

```text
                LUT    Logic/SRL    FF    CARRY8    DSP    延迟
1DSP版本       131     122/9      236      1       1     11拍
2DSP版本        92      83/9      188      1       2     11拍
```

1DSP版本只把对阶和尾数加减交给DSP，规格化和舍入仍占用较多LUT。相较纯逻辑版本，LUT减少44个，但FF增加23个，CARRY8减少2个，最差数据路径减少0.041 ns。

2DSP版本进一步利用DSP完成可变左移和舍入加法。相较1DSP版本，LUT减少39个，FF减少48个，其他逻辑资源不变，延迟仍为11拍。相较纯逻辑版本，LUT减少83个、FF减少25个，最差数据路径减少0.326 ns。这里的LUT减少不是两个与计算本身带来的，而是DSP改变了综合后的算术结构，使对阶、尾数加减、规格化和舍入不再分别映射为大段LUT逻辑。
