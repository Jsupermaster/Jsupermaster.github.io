---
title: "FPGA 8051软核处理器设计实战 - 4.verilog硬件描述语言基础"
date: 2025-01-08 20:00:00 +0800
description: "这部分主要讲述verilog基础语法和开发指南，笔者认为学习这个项目之前都应该已经有verilog基础，所以关于基础语法内容不多赘述，大家可以参见原书，只在这里记录一下笔者觉得看完仍有收获，值得注意的点。"
categories:
  - FPGA
home_category: engineering-practice
collection: fpga-8051-softcore-design
tags:
  - FPGA
  - "8051"
  - Verilog
  - Book Notes
article_kicker: BOOK NOTE
cover_image: /assets/images/fpga_8051_softcore/51KE6P2qUSL._SY445_SX342_.jpg
---

### 4.verilog硬件描述语言基础

这部分主要讲述verilog基础语法和开发指南，笔者认为学习这个项目之前都应该已经有verilog基础，所以关于基础语法内容不多赘述，大家可以参见原书，只在这里记录一下笔者觉得看完仍有收获，值得注意的点。

#### 4.1 net数据类型

| wire     | tri        | tri0       | supply0     |
| -------- | ---------- | ---------- | ----------- |
| **wand** | **triand** | **tri1**   | **supply1** |
| **wor**  | **trior**  | **trireg** | **uwire**   |

wire是我们最常用的线网型数据类型声明，但剩下的11个也是合法的数据类型声明符。

- tri和wire的语法和功能近似，但tri可以接受多个驱动源。
- wor,wand,trior,triand表示用于“线”逻辑操作的net，wor和trior是或操作的net，如果用它把两个驱动源连接起来，结果就是两个驱动源或的结果。
- wand和triand是与逻辑的net。
- trireg是一种特殊的net，用于模拟某种电容，具有verible的特性。
- tri0和tri1是模拟带上拉电阻或下拉电阻的连续，如果没有其他驱动源，其会有一个默认值0或1。
- uwire表示只允许一个驱动源的连线。
- supply0和supply1用于模拟电路中的电源线。

在定义一个net型数据时，有3种方式，第一种是最常见的：

```verilog
wire one_net;
```

第二种方法引入了signed vectored scalared三个关键词，signed指示这是有符号数，会以补码方式存储数据；当net和reg型变量位宽大于1时被认为是一个向量，vectored和scalared是两个可选关键词，vectored不允许位索引和拓展它的位宽，scalared允许位索引和拓展位宽。

```verilog
wire signed [3:0] net0;
wire vectored [2:0] net1;
wire scalared [2:0] net2;
wire vectored signed [11:0] #3 net3;
```

其中#3表示延时3个单位时间，所有对这个net的赋值都会延迟3个单位再生效。

第三种方法引入了driver strength，它是为了区分对同一个net赋值时，应该赋值给哪个驱动源，看net的驱动能力。driver strength分为四档，分别是从强到弱supply,strong,pull,weak，后面还要跟一个0或1，表示驱动0或1的能力。如(supply1,pull0)，表示驱动1的能力是supply，驱动0的能力是pull。不定义时，一般是(strong1,strong0)。如果一个net带有上拉电阻，pullup(net0)，则net0的1驱动时pull级别，如果外界使用strong或supply级别的驱动，net0将会由更强的驱动源决定。再引入highz0和highz1，表示输出0或1时用'bz高阻态代替。不可使用(highz0,highz1)定义一个net，因为它不能输出0或1都是高阻态。

```verilog
wire (highz1,pull0) net0;
```

在下面这个语句中，它的驱动源是a&b：

```verilog
wire net0 = a & b
```

#### 4.2 variable数据类型

variable数据类型可以分为2种，一种是整型的，包括reg,integer和time，初始值是x；一种是实数型，包括real和realtime，初始值是0.0。

integer一般用于仿真，一般没有硬件对应关系，非要对应的话它可以对应与32位宽的寄存器，time用于表示系统时间，一般至少是64位的。

刚才对于net型的叙述，同样适用于reg。

#### 4.3 操作数位选择

当我们要选中一个net、variable、parameter的一部分位时，有以下几种方式：

方式一，也是最简单的方法，需要注意的是这样的方法必须使用常数，包含确定的数字或用parameter定义的常量。

```verilog
vect[msb_expr : lsb_expr];//msb_expr和lsb_expr必须是常数
```

方式二，可以用一种特殊的方法实现用变量进行位选择，如下：

```verilog
reg [15:0] big_vect;//注意，[15:0]的定义方式是高位在前，如big_vect=1110,则big_vect[4]=1
reg [0:15] little_vect;//[0:15]的定义方式是低位在前，如little_vect=1110,则little_vect[0]=1
big_vect[lsb_base_expr +: width_expr]
little_vect[msb_base_expr +: width_expr]
//其中lsb_base_expr和msb_base_expr是变量，width_expr是常量。
```

用方式二定义，举例如下：

```verilog
reg [15:0] big_vect;
reg [0:15] little_vect;
reg [63:0] dword;
integer sel;

big_vect[0 +: 8] // == big_vect[7 : 0]
big_vect[15 -: 8] // == big_vect[15 : 8]
little_vect[0 +: 8] // == little_vect[0 : 7]
little_vect[15 -: 8] // == libble_vect[15 : 8]
dword[8*sel += 8] // 长度固定，基础位可以变化
//也就是说，后面加或减的数相当于位宽，而前面的变量相当于基地址，选取从哪里开始的8位宽数据。位宽不能变，但从哪里开始选择这一段位宽是可以改变的。
```

#### 4.4 一些操作符

关系操作符（>,<,>=,<=）只有两边都是有符号数，才进行有符号数比较，一方是无符号数，都按照无符号数比较。两边位宽不一致，按照高位宽补0后比较。

等式操作符分为两类，如下表：

| 格式    | 说明                                                         |
| ------- | ------------------------------------------------------------ |
| a == =b | case型，将x和z纳入比较，一边的某一位是x，另一边对应位也必须是x才相等，结果只有0或1 |
| a !== b | case型，将x和z纳入比较，一边的某一位是x，另一边对应位也必须是x才相等，结果只有0或1 |
| a == b  | logical型，如果某个位是x，比较结果也是x，结果包括1，0，x     |
| a != b  | logical型，如果某个位是x，比较结果也是x，结果包括1，0，x     |

#### 4.5 赋值语句

连续赋值语句：assign语句，等式两侧连接效果是持续性的，在这个过程中只要右侧值发生改变，左侧值也会立刻改变，适用于net型变量；

过程赋值语句：分为阻塞和非阻塞，赋值后左侧保存上一次赋值不变，直到下一次进行赋值，这种连接是激活性的，只在always或initial作用下被激活时才进行一次赋值，而不是始终连接的，适用于variable型变量。

过程连续赋值语句：包含两种形式，一种是assign，一种是force。assign的赋值对象只能是variable型变量，并且不能是它的位选择或部分位选择。这种连接关系可以通过deassign关键字取消。示例如下：

```verilog
module dff {q, d, clear, preset, clock};
output q;
input d,clear,preset,clock;
reg q;
always @ (clear or preset)begin
	if (!clear)
		assign q = 0;
	else if (!preset)
		assign q = 1;
	else
		deassign q;
end

always @ (posedge clock)
q = d;
endmodule
```

另一种时force语句，其对象可以是net型或vairable型，也可以对位选择或部分位选择赋值。它产生的连接效果可以用release来取消。

```verilog
module test
reg a, b, c, d;
wire e;

assign e = a & b & c;

initial begin
$monitor("%d d=%b,e=%b",$stime, d, e);
assign d = a & b & c;
a = 1;
b = 0;
c = 1;
#10;
force d = {a | b | c};
force e = {a | b | c};
#10;
release d;
release e;
#10 $finish;
end
endmodule
```

注意到这里先用assign对e和d进行赋值，但后面又使用force赋值，这时按照force的赋值规则计算，当用release取消force的赋值后，继续使用先前assign的赋值规则。对wire型的赋值无法取消，只有在always块中对variable赋值的assign赋值可以通过deassign取消。

#### 4.6 case语句

case语句可以比较x、z的状态，如下所示：

```verilog
case(select[1:2])
2'b00: result = 0;
2'b01: result = flaga;
2'b0x,2'b0z:result = flaga ? 'bx : 0;
2'b10: result = flagb;
2'bx0,2'bz0: result = flagb ? 'bx : 0;
default result = 'bx;
endcase
```

case有两个变种，即casez和casex，casez表示z不作为比较项，casex表示x和z都不作为比较项。不作为比较项的含义是，比如输入是100x，则x不作为比较项，只使用100与case中的可能项进行比较。示例如下：

```verilog
reg [7:0] ir;
casez (ir)
8'b1??????? : instruction1(ir);
8'b01?????? : instruction2(ir);
8'b00010??? : instruction3(ir);
8'b000001?? : instruction4(ir);
endcase
```

casex的示例如下：

```verilog
reg [7:0] r, mask;

mask = 8'bx0x0x0x0;
casex (r ^ mask)
8'b001100xx: stat1;
8'b1100xx00: stat2;
8'b00xx0011: stat3;
8'bxx010100: stat4;
endcase
```
