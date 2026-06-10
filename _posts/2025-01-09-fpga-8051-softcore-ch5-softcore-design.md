---
title: "FPGA 8051软核处理器设计实战 - 5.8051软核处理器设计"
date: 2025-01-09 20:00:00 +0800
description: "在前面进行了一系列的铺垫之后，我们终于正式进入到了设计8051处理器的步骤。我们需要设计一个8051的ip核，方便我们之后进行调用。"
categories:
  - FPGA
tags:
  - FPGA
  - "8051"
  - Verilog
  - Book Notes
article_kicker: BOOK NOTE
cover_image: /assets/images/fpga_8051_softcore/51KE6P2qUSL._SY445_SX342_.jpg
---

### 5.8051软核处理器设计

在前面进行了一系列的铺垫之后，我们终于正式进入到了设计8051处理器的步骤。我们需要设计一个8051的ip核，方便我们之后进行调用。

李新兵老师官方代码链接：[GitHub - risclite/R8051: 8051 soft CPU core. 700-lines statements for 111 instructions . Fully synthesizable Verilog-2001 core.](https://github.com/risclite/R8051)这个项目也称为R8051，它的特点是只使用了700行verilog代码就实现了一个支持全部111条指令的8051软核处理器。

#### 5.1 8051软核处理器接口

首先我们先来看看8051的接口部分，我们设计的软核处理器是对原有处理器的再创造，首先我们回忆一下8051的整体架构：

![文档扫描_20241227215542908](/assets/images/fpga_8051_softcore/文档扫描_20241227215542908.jpg)

左边是CODE区，我们之前已经分析过C语言程序转换为汇编程序，再按照机器码排列起来，就组成了我们的程序序列，我们的软核处理器必须能够从CODE区读出指令，不断解析和执行下去。其他的存储器分为两个部分，XDATA区是独立的内存地址，使用16bit内存地址，从0000H到FFFFH。另外一部分也是独立的内存地址，使用8bit内存地址，从00H到FFH，对于8051来说，DATA区占据低地址空间（到7FH），SFR区占据高地址空间（从80H开始）；对于8052处理器而言，低地址也是DATA区，但高地址在寄存器间接寻址时使用IDATA区，其他时候使用SFR区。因此，我们使用3套接口，实现从CODE区读取指令、从两个数据区读写指令的功能。

在实现中，我们可以使用ROM存放CODE区数据，使用RAM存放数据区数据。故我们可以使用ROM和RAM的通用接口定义来构建我们的接口。首先把ROM的时序图绘制如下：

![屏幕截图 2025-01-01 131521](/assets/images/fpga_8051_softcore/屏幕截图 2025-01-01 131521.png)

在这里我们假定rom可以在给出读使能和读地址后的1个周期给出数据，但有时这样的假设并不成立，因此我们可以再加入一个读数据有效信号，以适配更多的存储器特性：

![屏幕截图 2025-01-01 132120](/assets/images/fpga_8051_softcore/屏幕截图 2025-01-01 132120.png)

通过加入vld信号，我们的处理器可以适配更广泛的场景，不必需在1个周期后给出读数据。

8051对于不同存储区的访问不会同时进行，因此它们可以公用一组地址线和数据线。当然，我们的顶层接口必须包含clk，rst等信号。综上所述，我们顶层接口信号定义如下：

![屏幕截图 2025-01-01 134615](/assets/images/fpga_8051_softcore/屏幕截图 2025-01-01 134615.png)

四个使能信号，分别对应于DATA区，SFR区，IDATA区和XDATA区。它们公用一个16位的地址信号，DATA、IDATA、SFR区只使用低8位，XDATA区使用16位。系统接口除了clk和rst以外，还有一个cpu_en和cpu_restart信号，cpu_en为1时，cpu向前工作一个周期，如果它是0，则保持本周期状态不变，这相当于增加了一个暂停键。cpu_restart是复位功能，等于1时软核处理器复位，变成0的时候，不管以前处于任何状态都返回0开始重新运行。为了便于说明，我们将作者的源码从github上下载到本地，可以看到有如下几个文件夹，其中rtl文件夹中存放的就是8051的verilog源码。接下来我们仍旧按照书本的内容介绍这份代码。

![屏幕截图 2025-01-01 154922](/assets/images/fpga_8051_softcore/屏幕截图 2025-01-01 154922.png)

#### 5.2 8051软核处理器的基本架构

结合上一节的内容，我们编写的8051软核处理器ip的接口部分：

```verilog
//`define TYPE8052
module r8051 (

input  wire        clk,
input  wire        rst,
input  wire        cpu_en,
input  wire        cpu_restart,
	
output reg         rom_en,
output reg  [15:0] rom_addr,
input  wire [7:0]  rom_byte,
input  wire        rom_vld,
	
output wire        ram_rd_en_data,
output wire        ram_rd_en_sfr,
`ifdef TYPE8052
output wire        ram_rd_en_idata,
`endif
output wire        ram_rd_en_xdata,
output wire [15:0] ram_rd_addr,
input  wire [7:0]  ram_rd_byte,
input  wire        ram_rd_vld,
	
output wire        ram_wr_en_data,
output wire        ram_wr_en_sfr,
`ifdef TYPE8052
output wire        ram_wr_en_idata,
`endif
output wire        ram_wr_en_xdata,
output wire [15:0] ram_wr_addr,
output wire [7:0]  ram_wr_byte

);
endmodule
```

这里针对IDATA的接口只有在定义是8052型处理器时才被启用，实现了对于8051和8052型处理器的兼容。

这里我们针对CODE区的数据接口rom_byte只有8bit，但我们的指令实际是有1字节，2字节和3字节的。这是因为我们设计的系统需要适应变长指令，单字节的指令能在1个周期内完成，双字节的则在2个周期内完成。这样我们按照字节送入指令，不管指令的长度是多少，当指令送入完毕，他也就执行完毕了。

单字节的指令可以在指令流水第一级读取数据，在第二级流水写上处理完毕的数据，双字节可以在第一个字节取出时完成度，在第二个字节取出时完成写，三字节可以在前2个字节读出时完成读数据，第三个字节完成写数据。

我们的指令流需要根据指令的不同指挥RAM相应的读接口启动，接着对RAM中的数据进行改写，改写完毕后从RAM的写接口将结果写入某处。

以下是定义的指令流，先给出一个地址rom_addr[15:0]，并拉高rom_en，那么下一个周期，rom_byte等于cmda，下一个周期，cmda向下流动，cmda中的指令进入cmdb中。

![微信图片_20250101162434](/assets/images/fpga_8051_softcore/微信图片_20250101162434.jpg)

**单字节指令**

单字节指令在cmda时读RAM，在cmdb时写RAM，在cmdc时禁止写RAM。因为当单字节指令处于cmdc时，前面的cmda和cmdb可能是双字节指令，这时它也需要写RAM，这导致写入冲突，所以单字节指令禁止在cmdc写入RAM。

**双字节指令**

回顾一下之前对于指令集的讲解，双字节指令中第一个字节是指令识别序列，第二个字节是读RAM地址，也就是说，当双字节指令的头字节进入cmdb时，读RAM地址进入cmda，这时进行读RAM操作，双字节指令的头字节进入cmdc时进行写操作，这时禁止双字节指令进行读操作，因为此时cmda中可能有一个单字节指令正在发起读请求。

**三字节指令**

三字节指令的头字节位于cmda和cmdb时可以进行读操作，在cmdc时可以进行写操作，它不会和别的指令发生冲突。

我们把上面的过程画一个图来表示一下：

![屏幕截图 2025-01-01 174059](/assets/images/fpga_8051_softcore/屏幕截图 2025-01-01 174059.png)

我们通过编写一个类似C语言中的头文件一样的文件来把指令的识别函数与主体函数做一个分离，这样的优势是框架更清晰。整体程序如下：

```verilog
//ARITHMETIC OPERATIONS
//ADD
function add_a_rn(input [7:0] i);  add_a_rn = (i[7:3]==5'b00101); endfunction
function add_a_di(input [7:0] i);  add_a_di=(i==8'b00100101); endfunction
function add_a_ri(input [7:0] i);  add_a_ri=(i[7:1]==7'b0010011); endfunction
function add_a_da(input [7:0] i);  add_a_da = (i==8'b00100100);  endfunction
//ADDC
function addc_a_rn(input [7:0] i); addc_a_rn = (i[7:3]==5'b00111); endfunction
function addc_a_di(input [7:0] i); addc_a_di = (i==8'b00110101); endfunction
function addc_a_ri(input [7:0] i); addc_a_ri = (i[7:1]==7'b0011011); endfunction
function addc_a_da(input [7:0] i); addc_a_da = (i==8'b00110100); endfunction
//SUBB
function subb_a_rn(input [7:0] i); subb_a_rn=(i[7:3]==5'b10011); endfunction
function subb_a_di(input [7:0] i); subb_a_di=(i==8'b10010101); endfunction
function subb_a_ri(input [7:0] i); subb_a_ri=(i[7:1]==7'b1001011); endfunction
function subb_a_da(input [7:0] i); subb_a_da = (i==8'b10010100); endfunction
//INC
function inc_a (input [7:0] i); inc_a = (i==8'b00000100); endfunction
function inc_rn(input [7:0] i); inc_rn = (i[7:3]==5'b00001); endfunction
function inc_di(input [7:0] i); inc_di = (i==8'b00000101); endfunction
function inc_ri(input [7:0] i); inc_ri = (i[7:1]==7'b0000011); endfunction
function inc_dp(input [7:0] i); inc_dp =(i==8'b10100011); endfunction
//DEC
function dec_a(input [7:0] i); dec_a = (i==8'b00010100); endfunction
function dec_rn(input [7:0] i); dec_rn = (i[7:3]==5'b00011); endfunction
function dec_di(input [7:0] i); dec_di = (i==8'b00010101); endfunction
function dec_ri(input [7:0] i); dec_ri = (i[7:1]==7'b0001011); endfunction
//MUL
function mul(input [7:0] i); mul=(i==8'b10100100); endfunction
//DIV
function div(input [7:0] i); div=(i==8'b10000100); endfunction
//DA
function da(input [7:0] i); da = (i==8'b11010100); endfunction

//LOGICAL OPERATIONS
//ANL
function anl_a_rn(input [7:0] i); anl_a_rn = (i[7:3]==5'b01011); endfunction
function anl_a_di(input [7:0] i); anl_a_di = (i==8'b01010101); endfunction
function anl_a_ri(input [7:0] i); anl_a_ri = (i[7:1]==7'b0101011); endfunction
function anl_a_da(input [7:0] i); anl_a_da = (i==8'b01010100); endfunction
function anl_di_a(input [7:0] i); anl_di_a = (i==8'b01010010); endfunction
function anl_di_da(input [7:0] i); anl_di_da = (i==8'b01010011); endfunction
//ORL
function orl_a_rn(input [7:0] i); orl_a_rn = (i[7:3]==5'b01001); endfunction
function orl_a_di(input [7:0] i); orl_a_di = (i==8'b01000101); endfunction
function orl_a_ri(input [7:0] i); orl_a_ri = (i[7:1]==7'b0100011); endfunction
function orl_a_da(input [7:0] i); orl_a_da = (i==8'b01000100); endfunction
function orl_di_a(input [7:0] i); orl_di_a=(i==8'b01000010); endfunction
function orl_di_da(input [7:0] i); orl_di_da=(i==8'b01000011); endfunction
//XRL
function xrl_a_rn(input [7:0] i); xrl_a_rn = (i[7:3]==5'b01101); endfunction
function xrl_a_di(input [7:0] i); xrl_a_di = (i==8'b01100101); endfunction
function xrl_a_ri(input [7:0] i); xrl_a_ri = (i[7:1]==7'b0110011); endfunction
function xrl_a_da(input [7:0] i); xrl_a_da = (i==8'b01100100); endfunction
function xrl_di_a(input [7:0] i); xrl_di_a = (i==8'b01100010); endfunction
function xrl_di_da(input [7:0] i); xrl_di_da = (i==8'b01100011); endfunction
//CLR
function clr_a(input [7:0] i); clr_a = (i==8'b11100100); endfunction
//CPL
function cpl_a(input [7:0] i); cpl_a = (i==8'b11110100); endfunction
//RL
function rl(input [7:0] i); rl = (i==8'b00100011); endfunction
//RLC
function rlc(input [7:0] i); rlc = (i==8'b00110011); endfunction
//RR
function rr(input [7:0] i); rr = (i==8'b00000011); endfunction
//RRC
function rrc(input [7:0] i); rrc = (i==8'b00010011); endfunction
//SWAP
function swap(input [7:0] i); swap = (i==8'b11000100); endfunction

//DATA TRANSFER
//MOV
function mov_a_rn(input [7:0] i); mov_a_rn = (i[7:3]==5'b11101); endfunction
function mov_a_di(input [7:0] i); mov_a_di = (i==8'b11100101); endfunction
function mov_a_ri(input [7:0] i); mov_a_ri = (i[7:1]==7'b1110011); endfunction
function mov_a_da(input [7:0] i); mov_a_da = (i==8'b01110100); endfunction
function mov_rn_a(input [7:0] i); mov_rn_a = (i[7:3]==5'b11111); endfunction
function mov_rn_di(input [7:0] i); mov_rn_di = (i[7:3]==5'b10101); endfunction
function mov_rn_da(input [7:0] i); mov_rn_da = (i[7:3]==5'b01111); endfunction
function mov_di_a(input [7:0] i); mov_di_a = (i==8'b11110101); endfunction
function mov_di_rn(input [7:0] i); mov_di_rn = (i[7:3]==5'b10001); endfunction
function mov_di_di(input [7:0] i); mov_di_di = (i==8'b10000101); endfunction
function mov_di_ri(input [7:0] i); mov_di_ri = (i[7:1]==7'b1000011); endfunction 
function mov_di_da(input [7:0] i); mov_di_da = (i==8'b01110101); endfunction
function mov_ri_a(input [7:0] i);  mov_ri_a = (i[7:1]==7'b1111011); endfunction
function mov_ri_di(input [7:0] i); mov_ri_di = (i[7:1]==7'b1010011); endfunction
function mov_ri_da(input [7:0] i); mov_ri_da=(i[7:1]==7'b0111011); endfunction
function mov_dp_da(input [7:0] i); mov_dp_da=(i==8'b10010000); endfunction
//MOVC
function movc_a_dp(input [7:0] i); movc_a_dp = (i==8'b10010011); endfunction
function movc_a_pc(input [7:0] i); movc_a_pc = (i==8'b10000011); endfunction
//MOVX
function movx_a_ri(input [7:0] i); movx_a_ri = (i[7:1]==7'b1110001); endfunction
function movx_a_dp(input [7:0] i); movx_a_dp = (i==8'b11100000); endfunction
function movx_ri_a(input [7:0] i); movx_ri_a = (i[7:1]==7'b1111001); endfunction
function movx_dp_a(input [7:0] i); movx_dp_a = (i==8'b11110000); endfunction
//PUSH
function push(input [7:0] i); push = (i==8'b11000000); endfunction
//POP
function pop(input [7:0] i); pop = (i==8'b11010000); endfunction
//XCH
function xch_a_rn(input [7:0] i); xch_a_rn = (i[7:3]==5'b11001); endfunction
function xch_a_di(input [7:0] i); xch_a_di = (i==8'b11000101); endfunction
function xch_a_ri(input [7:0] i); xch_a_ri = (i[7:1]==7'b1100011); endfunction
//XCHD
function xchd(input [7:0] i); xchd = (i[7:1]==7'b1101011); endfunction

//BOOLEAN VARIABLE MANIPULATION
//CLR
function clr_c(input [7:0] i); clr_c = (i==8'b11000011); endfunction
function clr_bit(input [7:0] i); clr_bit = (i==8'b11000010); endfunction
//SETB
function setb_c(input [7:0] i); setb_c = (i==8'b11010011); endfunction
function setb_bit(input [7:0] i); setb_bit = (i==8'b11010010); endfunction
//CPL
function cpl_c(input [7:0] i); cpl_c = (i==8'b10110011); endfunction
function cpl_bit(input [7:0] i); cpl_bit=(i==8'b10110010); endfunction
//ANL
function anl_c_bit(input [7:0] i); anl_c_bit = (i==8'b10000010); endfunction
function anl_c_nbit(input [7:0] i); anl_c_nbit = (i==8'b10110000); endfunction
//ORL
function orl_c_bit(input [7:0] i); orl_c_bit = (i==8'b01110010); endfunction
function orl_c_nbit(input [7:0] i); orl_c_nbit = (i==8'b10100000); endfunction
//MOV
function mov_c_bit(input [7:0] i); mov_c_bit = (i==8'b10100010); endfunction
function mov_bit_c(input [7:0] i); mov_bit_c = (i==8'b10010010); endfunction
//JC
function jc(input [7:0] i); jc = (i==8'b01000000); endfunction
//JNC
function jnc(input [7:0] i); jnc =(i==8'b01010000); endfunction
//JB
function jb(input [7:0] i); jb = (i==8'b00100000); endfunction
//JNB
function jnb(input [7:0] i); jnb = (i==8'b00110000); endfunction
//JBC
function jbc(input [7:0] i); jbc = (i==8'b00010000); endfunction

//PROGRAM BRANCHING
//ACALL
function acall(input [7:0] i); acall = (i[4:0]==5'b10001); endfunction
//LCALL
function lcall(input [7:0] i); lcall = (i==8'b00010010); endfunction
//RET
function ret(input [7:0] i); ret = (i==8'b00100010); endfunction
//RETI
function reti(input [7:0] i); reti = (i==8'b00110010); endfunction
//AJMP
function ajmp(input [7:0] i); ajmp = (i[4:0]==5'b00001); endfunction
//LJMP
function ljmp(input [7:0] i); ljmp = (i==8'b00000010); endfunction
//SJMP
function sjmp(input [7:0] i); sjmp = (i==8'b10000000); endfunction
//JMP
function jmp(input [7:0] i); jmp = (i==8'b01110011); endfunction
//JZ
function jz(input [7:0] i); jz = (i==8'b01100000); endfunction
//JNZ
function jnz(input [7:0] i); jnz = (i==8'b01110000); endfunction
//CJNE
function cjne_a_di_rel(input [7:0] i); cjne_a_di_rel = (i==8'b10110101); endfunction
function cjne_a_da_rel(input [7:0] i); cjne_a_da_rel = (i==8'b10110100); endfunction
function cjne_rn_da_rel(input [7:0] i); cjne_rn_da_rel = (i[7:3]==5'b10111); endfunction
function cjne_ri_da_rel(input [7:0] i); cjne_ri_da_rel=(i[7:1]==7'b1011011); endfunction
//DJNZ
function djnz_rn_rel(input [7:0] i); djnz_rn_rel = (i[7:3]==5'b11011); endfunction
function djnz_di_rel(input [7:0] i); djnz_di_rel =(i==8'b11010101); endfunction
//NOP
function nop(input [7:0] i); nop = (i==8'b00000000); endfunction
```

这些“一句话”函数帮我们识别指令。根据上面的分析我们得知，我们识别指令都应该通过cmda区域进行识别，我们将3字节，2字节和1字节长度的指令找到，用以下的代码实现：

```verilog
//command length

assign  length1 = add_a_rn(cmda)|addc_a_rn(cmda)|subb_a_rn(cmda)|inc_a(cmda)|inc_rn(cmda)|inc_dp(cmda)|dec_a(cmda)|dec_rn(cmda)|mul(cmda)|div(cmda)|da(cmda)|anl_a_rn(cmda)|orl_a_rn(cmda)|xrl_a_rn(cmda)|clr_a(cmda)|cpl_a(cmda)|rl(cmda)|rlc(cmda)|rr(cmda)|rrc(cmda)|swap(cmda)|mov_a_rn(cmda)|mov_rn_a(cmda)|mov_ri_a(cmda)|movx_a_dp(cmda)|movx_ri_a(cmda)|movx_dp_a(cmda)|xch_a_rn(cmda)|clr_c(cmda)|setb_c(cmda)|cpl_c(cmda)|jmp(cmda);

assign  length2r1 = add_a_ri(cmda)|addc_a_ri(cmda)|subb_a_ri(cmda)|inc_ri(cmda)|dec_ri(cmda)|anl_a_ri(cmda)|orl_a_ri(cmda)|xrl_a_ri(cmda)|mov_a_ri(cmda)|movc_a_dp(cmda)|movc_a_pc(cmda)|movx_a_ri(cmda)|xch_a_ri(cmda)|xchd(cmda);

assign  length2 = add_a_di(cmda)|add_a_da(cmda)|addc_a_di(cmda)|addc_a_da(cmda)|subb_a_di(cmda)|subb_a_da(cmda)|inc_di(cmda)|dec_di(cmda)|anl_a_di(cmda)|anl_a_da(cmda)|anl_di_a(cmda)|orl_a_di(cmda)|orl_a_da(cmda)|orl_di_a(cmda)|xrl_a_di(cmda)|xrl_a_da(cmda)|xrl_di_a(cmda)|mov_a_di(cmda)|mov_a_da(cmda)|mov_rn_di(cmda)|mov_rn_da(cmda)|mov_di_a(cmda)|mov_di_rn(cmda)|mov_di_ri(cmda)|mov_ri_di(cmda)|mov_ri_da(cmda)|push(cmda)|pop(cmda)|xch_a_di(cmda)|clr_bit(cmda)|setb_bit(cmda)|cpl_bit(cmda)|anl_c_bit(cmda)|anl_c_nbit(cmda)|orl_c_bit(cmda)|orl_c_nbit(cmda)|mov_c_bit(cmda)|mov_bit_c(cmda)|jc(cmda)|jnc(cmda)|ajmp(cmda)|sjmp(cmda)|jz(cmda)|jnz(cmda)|djnz_rn_rel(cmda);

assign  length3 = anl_di_da(cmda)|orl_di_da(cmda)|xrl_di_da(cmda)|mov_di_di(cmda)|mov_di_da(cmda)|mov_dp_da(cmda)|jb(cmda)|jnb(cmda)|jbc(cmda)|acall(cmda)|lcall(cmda)|ret(cmda)|reti(cmda)|ljmp(cmda)|cjne_a_di_rel(cmda)|cjne_a_da_rel(cmda)|cjne_rn_da_rel(cmda)|cjne_ri_da_rel(cmda)|djnz_di_rel(cmda);
```

其中的2r1表示这条指令虽然是单字节，但由于其操作复杂，所以难以在1个周期内处理完，所以我们将其当作2字节指令，后面可以跟一个空字节。比如寄存器间接寻址指令，我们完成它需要两个读出操作，第一次从Ri中读出地址，第二次才能读出真正的数据。这样的指令难以在1个周期内执行完毕。

作者在引用文件中还写了一个除法器，它让我们可以在1个周期内获得计算结果。

```verilog
function [15:0] divide ( input [7:0] a, input [7:0] b);
reg  [7:0]     ans;
reg  [7:0]     rem;
reg  [7:0]     x7;
reg  [7:0]     x6;
reg  [7:0]     x5;
reg  [7:0]     x4;
reg  [7:0]     x3;
reg  [7:0]     x2;
reg  [7:0]     x1;
reg  [7:0]     x0;

reg  [1:0]     y5;
reg  [2:0]     y4; 
reg  [3:0]     y3;
reg  [4:0]     y2;
reg  [5:0]     y1;
reg  [6:0]     y0;
begin

x7 = a;
ans[7] = (|b[7:1])? 1'b0 : x7[7];

x6 = {(~ans[7])&a[7],a[6:0]};
ans[6] = (|b[7:2])? 1'b0 : (x6[7:6]>=b[1:0]);

y5 = ans[6] ? (x6[7:6]-b[1:0]) : x6[7:6];
x5 = { y5, a[5:0] };
ans[5] = (|b[7:3])? 1'b0 : ( x5[7:5]>=b[2:0] );

y4 = ans[5] ? (x5[7:5]-b[2:0]) : x5[7:5];
x4 = { y4, a[4:0]};
ans[4] = (|b[7:4])? 1'b0 : ( x4[7:4]>=b[3:0] );

y3 = ans[4] ? (x4[7:4]-b[3:0]) : x4[7:4];
x3 = {y3, a[3:0]};
ans[3] = (|b[7:5])? 1'b0 : ( x3[7:3]>=b[4:0] );

y2 = ans[3] ? (x3[7:3]-b[4:0]) : x3[7:3];
x2 = {y2,a[2:0]};
ans[2] = (|b[7:6])? 1'b0 : ( x2[7:2]>=b[5:0] );

y1 = ans[2] ? (x2[7:2]-b[5:0]) : x2[7:2];
x1 = {y1,a[1:0]};
ans[1] = (|b[7]) ? 1'b0 : ( x1[7:1]>=b[6:0] );

y0 = ans[1] ? (x1[7:1]-b[6:0]) : x1[7:1];
x0 = {y0,a[0]};
ans[0] = (x0>=b);

rem = ans[0] ? (x0-b) : x0;

divide = {rem,ans};
end

endfunction
```

其中a是被除数，b是除数，最后的结果是8位商和余数拼成16位输出，高8位是余数，低8位是商。

#### 5.3 主体程序详细解读

##### 5.3.1 使能信号

8051有两种方式暂停，一个是输入端口cpu_en，当cpu_en处于低电平，系统寄存器全部维持现状不变，当其恢复高电平，系统开始继续工作；另一个是rom_vld和ram_rd_vld，若连接的存储器速度较慢，在读使能的一个周期以上才能给出读数据，那么一旦rom_en或ram_rd_en拉高，系统暂停工作，当rom_vld和rom_rd_vld拉高时自动解除暂停恢复工作。

```verilog
//work_en

always @ ( posedge clk or posedge rst )
if ( rst )
    rd_wait <= 1'b0;
else if ( cpu_restart )
    rd_wait <= 1'b0;	
`ifdef TYPE8052
else if ( ram_rd_en_data|ram_rd_en_sfr|ram_rd_en_idata|ram_rd_en_xdata )
`else
else if ( ram_rd_en_data|ram_rd_en_sfr|ram_rd_en_xdata ) 
`endif	
    rd_wait <= 1'b1;
else if ( ram_rd_vld )
    rd_wait <= 1'b0;	
else;

reg rom_wait;
always @ ( posedge clk or posedge rst )
if ( rst )
    rom_wait <= 1'b0;
else if ( cpu_restart )
    rom_wait <= 1'b0;	
else if ( rom_en )
	rom_wait <= 1'b1;
else if ( rom_vld )
    rom_wait <= 1'b0;
else;


assign work_en = ( ( rd_wait & ~ram_rd_vld )|( rom_wait & ~rom_vld ) ) ? 1'b0 : cpu_en;
```

##### 5.3.2 指令的流水序列

由于8051的指令集并非等长的，所以当不同指令交替出现时容易产生干扰。比如这种情况：8'h24，8'h04是一个双字节指令，8'h04是它的立即数，但也是单字节指令INC ACC的机器码。因此我们需要给指令准备一个标志，用来识别这个8bit数据到底是指令识别序列还是数据或者地址。这个标志需要判断当前数据是否是一条指令的首字节，如果是首字节，标志置1，如果不是则置0.只有首字节我们进行识别判断。具体来说，当我们识别到它是单字节指令，下一个数据标志写为1，如果是双字节指令，下一个数据的标志写为0，如果是三字节，下面两个字节的数据都写为0.

```verilog
assign cmd0 = rom_byte;

always @ ( posedge clk or posedge rst )
if ( rst )
    cmd1 <= 8'b0;
else if ( work_en )
    cmd1 <= cmd0;
else;

always @ ( posedge clk or posedge rst )
if ( rst )
    cmd2 <= 8'b0;
else if ( work_en )
    cmd2 <= cmdb;
else;

always @ ( posedge clk or posedge rst )
if ( rst )
    cmd_flag <= 3'b1;
else if ( cpu_restart )
    cmd_flag <= 3'b1;
else if ( work_en )
    if ( wait_en )
	    cmd_flag <= {1'b0,cmd_flag[1:0]};
    else if ( length2|length2r1 )
	    cmd_flag <= 3'b101;
	else if ( length3 )
	    cmd_flag <= 3'b100;
	else
        cmd_flag <= { cmd_flag[1:0],1'b1};
else;

assign cmda = cmd_flag[1] ? cmd0 : 8'b0;

assign cmdb = cmd_flag[2] ? cmd1 : 8'b0;

assign cmdc = cmd2;
```

![屏幕截图 2025-01-01 231653](/assets/images/fpga_8051_softcore/屏幕截图 2025-01-01 231653.png)

我们来详细解读一下这部分，cmd0，cmd1，cmd2是原始的程序序列，cmd1比cmd0延时一个周期，cmd2比cmd1延时一个周期。cmda，cmdb和cmdc存放的是指令识别序列。如当判断cmd0是一个指令的识别序列而非后面的地址或数据时，将其赋值给cmda。图中第二行的数据是指令标志（flag），当处于time0时，cmd0是C1，这表示某指令C的第一个字节，也即指令C的识别序列。cmd1是B，cmd2是A，它们都是单字节指令。这时flag位是111，对应于flag[1]是cmd0的判断位，而flag[2]是cmd1的判断位，cmd2没有判断位，flag[0]没有使用。这时flag[1]=1，说明cmd0是指令的识别序列，将其赋值给cmda。flag[2]=1，说明cmd1是指令的识别序列，将其赋值给cmdb。cmd2直接赋值给cmdc。

这时，对flag进行赋值。此时cmda=cmd0，cmdb=cmd1，cmdc=cmd2。使用cmda判断指令长度，判断出为2字节指令。这时给flag赋值为101，将在下一个周期生效。

time1周期，flag=101，cmd0,1,2分别是C2（指令C的第二个字节），C1，B，对应于flag[1]=0，说明此时cmd0中的指令C2不是指令的第一个字节（指令识别序列），flag[2]=1，cmd1中的指令C1是指令识别序列，进行cmda,b,c的赋值。cmda=0（因为flag[1]=0），cmdb=cmd1，cmdc=cmd2。这时，对flag进行赋值。由于此时cmda=0，所以根据上面的判断语句，被判断为单周期指令，执行else中的语句，即flag向高位移位1位。（实际上应该是左移，使得最高位溢出，同时最低位补1，但这里图中画的flag顺序是[0]\[1]\[2]，故看上去像是右移。总归的意思是向高位移动一位，低位补1）所以得到的flag是110。

time2周期，flag=110，cmd0,1,2分别是D，C2，C1，对应于flag[1]=1，cmda=cmd0，flag[2]=0，cmdb=0（因为这时cmd1是C2，并非指令识别序列，所以不把它赋值给cmdb）

以此类推，三字节的也是一样，只不过它识别到三字节的识别序列后将flag赋值为100（再次提醒，图中看起来是001，原因是图中的字节序是反过来的）。

其中对于cpu_restart拉高时，flag直接赋值为001（3‘b1=3’b001），这是因为本周期发起读指令请求后，读出的第一条指令在下一个时钟周期才会出现在cmda。

其中的wait_en处理的是两条指令的写后读（前一条指令进行了写操作，后一条指令执行了读操作，但此时读到的仍然是之前未被修改的数据，而非前一条指令执行后的新数据）解决这个问题的方法有两个思路，一是直接把前一条指令写入的数据传给后一条指令，让后一条指令直接获得修改后的数据，而不需要从寄存器中读取。这样的缺点是需要使用组合逻辑，导致形成了延时。第二种思路是暂停流水线，中间增加一个周期的空闲。这里检测到wait_en的停顿正是如此。

##### 5.3.3 取指令操作

取指令模块的作用是发出读使能，且给出读地址。这些指令被分为两种，一种是pc和pc_en，一种是rom_en和rom_addr。pc是指令地址指针，在以下几种情况下pc不递增：

1. work_en=0，处理器处于暂停状态；
2. wait_en=1，发生了写后读冲突；
3. length2r1=1，这表示某些1字节指令需要2字节拓展，但这只是因为它难以在1周期内执行，是虚拟的2周期，并不需要从rom中读取指令；

rom_addr和rom_en目前在没有添加任何指令实现的时候等于pc，但后面会讲解例外情况。

```verilog
/*********************************************************/
//rom_en rom_addr

always @*
if ( ~work_en )
    pc_en = 1'b0;
else if ( wait_en|length2r1 )
    pc_en = 1'b0;
else
    pc_en = 1'b1;

always @ ( posedge clk or posedge rst )
if ( rst )
    pc <= 16'd0;
else if ( cpu_restart )
    pc <= 16'd0;
else if ( work_en )
    if ( pc_en )
        pc <= pc_add1;
	else;
else;

assign pc_add1 = rom_addr + 1'b1;

always @*
if ( ~work_en )
    rom_en = 1'b0;
else if ( wait_en )
    rom_en = 1'b0;
else if ( length2r1 )
    rom_en = 1‘b0;
else
    rom_en = 1'b1;

assign code_base = 1'b0 ? dp : pc;

assign code_rel = 1'b0 ? &#123;&#123;8{cmd0[7]&#125;&#125;,cmd0} : &#123;&#123;8{acc[7]&#125;&#125;,acc};	
	
assign code_addr = code_base + code_rel;	

always @*
    rom_addr = 1'b0 ? code_addr : pc;
/*********************************************************/
```

使用pc和rom两类信号可以灵活控制，因为有些指令需要控制是否从存放指令的rom中取指令，有些决定是否对指令寄存器pc改变。另外，跳转指令可能需要通过PC或DPTR获得新地址，因此使用code_addr可以满足这种需求，rom_addr可以从通用的PC切换到特殊的“一次性”code_addr上。

##### 5.3.4 指令的长度

```verilog
//command length

assign  length1 = add_a_rn(cmda)|addc_a_rn(cmda)|subb_a_rn(cmda)|inc_a(cmda)|inc_rn(cmda)|inc_dp(cmda)|dec_a(cmda)|dec_rn(cmda)|mul(cmda)|div(cmda)|da(cmda)|anl_a_rn(cmda)|orl_a_rn(cmda)|xrl_a_rn(cmda)|clr_a(cmda)|cpl_a(cmda)|rl(cmda)|rlc(cmda)|rr(cmda)|rrc(cmda)|swap(cmda)|mov_a_rn(cmda)|mov_rn_a(cmda)|mov_ri_a(cmda)|movx_a_dp(cmda)|movx_ri_a(cmda)|movx_dp_a(cmda)|xch_a_rn(cmda)|clr_c(cmda)|setb_c(cmda)|cpl_c(cmda)|jmp(cmda);

assign  length2r1 = add_a_ri(cmda)|addc_a_ri(cmda)|subb_a_ri(cmda)|inc_ri(cmda)|dec_ri(cmda)|anl_a_ri(cmda)|orl_a_ri(cmda)|xrl_a_ri(cmda)|mov_a_ri(cmda)|movc_a_dp(cmda)|movc_a_pc(cmda)|movx_a_ri(cmda)|xch_a_ri(cmda)|xchd(cmda);

assign  length2 = add_a_di(cmda)|add_a_da(cmda)|addc_a_di(cmda)|addc_a_da(cmda)|subb_a_di(cmda)|subb_a_da(cmda)|inc_di(cmda)|dec_di(cmda)|anl_a_di(cmda)|anl_a_da(cmda)|anl_di_a(cmda)|orl_a_di(cmda)|orl_a_da(cmda)|orl_di_a(cmda)|xrl_a_di(cmda)|xrl_a_da(cmda)|xrl_di_a(cmda)|mov_a_di(cmda)|mov_a_da(cmda)|mov_rn_di(cmda)|mov_rn_da(cmda)|mov_di_a(cmda)|mov_di_rn(cmda)|mov_di_ri(cmda)|mov_ri_di(cmda)|mov_ri_da(cmda)|push(cmda)|pop(cmda)|xch_a_di(cmda)|clr_bit(cmda)|setb_bit(cmda)|cpl_bit(cmda)|anl_c_bit(cmda)|anl_c_nbit(cmda)|orl_c_bit(cmda)|orl_c_nbit(cmda)|mov_c_bit(cmda)|mov_bit_c(cmda)|jc(cmda)|jnc(cmda)|ajmp(cmda)|sjmp(cmda)|jz(cmda)|jnz(cmda)|djnz_rn_rel(cmda);

assign  length3 = anl_di_da(cmda)|orl_di_da(cmda)|xrl_di_da(cmda)|mov_di_di(cmda)|mov_di_da(cmda)|mov_dp_da(cmda)|jb(cmda)|jnb(cmda)|jbc(cmda)|acall(cmda)|lcall(cmda)|ret(cmda)|reti(cmda)|ljmp(cmda)|cjne_a_di_rel(cmda)|cjne_a_da_rel(cmda)|cjne_rn_da_rel(cmda)|cjne_ri_da_rel(cmda)|djnz_di_rel(cmda);
```

指令的长度都是以cmda为检验基础的，cmda通过上面的赋值，已经被确认为每一条指令的第一字节。这里不再赘述。

##### 5.3.5 软核处理器的读操作

处理器对于数据的读取，可以分为两个部分，一是为使能信号赋值，二是为地址信号赋值。

先来看使能信号。使能信号分为三种，一种是对DATA区和SFR区进行读使能，具体是哪个区取决于地址的数值。第二种是DATA、IDATA区，这只有在8052类型中才会用到。第三种是对XDATA进行读取。

```verilog
//ram_rd_en ram_rd_addr 

assign rd_en_data_sfr = add_a_rn(cmda)|add_a_di(cmdb)|add_a_ri(cmda)|addc_a_rn(cmda)|addc_a_di(cmdb)|addc_a_ri(cmda)|subb_a_rn(cmda)|subb_a_di(cmdb)|subb_a_ri(cmda)|inc_rn(cmda)|inc_di(cmdb)|inc_ri(cmda)|dec_rn(cmda)|dec_di(cmdb)|dec_ri(cmda)|anl_a_rn(cmda)|anl_a_di(cmdb)|anl_a_ri(cmda)|anl_di_a(cmdb)|anl_di_da(cmdb)|orl_a_rn(cmda)|orl_a_di(cmdb)|orl_a_ri(cmda)|orl_di_a(cmdb)|orl_di_da(cmdb)|xrl_a_rn(cmda)|xrl_a_di(cmdb)|xrl_a_ri(cmda)|xrl_di_a(cmdb)|xrl_di_da(cmdb)|mov_a_rn(cmda)|mov_a_di(cmdb)|mov_a_ri(cmda)|mov_rn_di(cmdb)|mov_di_rn(cmda)|mov_di_di(cmdb)|mov_di_ri(cmda)|mov_ri_a(cmda)|mov_ri_di(cmda)|mov_ri_di(cmdb)|mov_ri_da(cmda)|movx_a_ri(cmda)|movx_ri_a(cmda)|push(cmdb)|xch_a_rn(cmda)|xch_a_di(cmdb)|xch_a_ri(cmda)|xchd(cmda)|clr_bit(cmdb)|setb_bit(cmdb)|cpl_bit(cmdb)|anl_c_bit(cmdb)|anl_c_nbit(cmdb)|orl_c_bit(cmdb)|orl_c_nbit(cmdb)|mov_c_bit(cmdb)|mov_bit_c(cmdb)|jb(cmdb)|jnb(cmdb)|jbc(cmdb)|cjne_a_di_rel(cmdb)|cjne_rn_da_rel(cmda)|cjne_ri_da_rel(cmda)|djnz_rn_rel(cmda)|djnz_di_rel(cmdb);

assign rd_en_data_idata = add_a_ri(cmdb)|addc_a_ri(cmdb)|subb_a_ri(cmdb)|inc_ri(cmdb)|dec_ri(cmdb)|anl_a_ri(cmdb)|orl_a_ri(cmdb)|xrl_a_ri(cmdb)|mov_a_ri(cmdb)|mov_di_ri(cmdb)|xch_a_ri(cmdb)|xchd(cmdb)|cjne_ri_da_rel(cmdb)|ret(cmda)|ret(cmdb)|reti(cmda)|reti(cmdb)|pop(cmda);

assign rd_en_xdata = movx_a_ri(cmdb)|movx_a_dp(cmda);

assign rd_en_data = (rd_en_data_sfr|rd_en_data_idata) & ~rd_addr[7];
`ifdef TYPE8052
assign rd_en_sfr = rd_en_data_sfr & rd_addr[7];
assign rd_en_idata = rd_en_data_idata & rd_addr[7];
`else
assign rd_en_sfr = (rd_en_data_sfr|rd_en_data_idata) & rd_addr[7];
`endif
assign same_flag_data = rd_en_data & wr_en_data & (rd_addr[7:0]==wr_addr[7:0]);
assign same_flag_sfr = rd_en_sfr & wr_en_sfr & (rd_addr[7:0]==wr_addr[7:0]);
`ifdef TYPE8052
assign same_flag_idata = rd_en_idata & wr_en_idata & (rd_addr[7:0]==wr_addr[7:0]);
`endif
assign same_flag_xdata = rd_en_xdata & wr_en_xdata & (rd_addr[15:0]==wr_addr[15:0]);
assign read_internal = rd_en_sfr & ( (rd_addr[7:0]==8'he0)|(rd_addr[7:0]==8'hd0)|(rd_addr[7:0]==8'h83)|(rd_addr[7:0]==8'h82)|(rd_addr[7:0]==8'h81)|(rd_addr[7:0]==8'hf0) );
assign ram_rd_en_data = work_en & rd_en_data & ~same_flag_data & ~wait_en;
assign ram_rd_en_sfr = work_en & rd_en_sfr & ~same_flag_sfr & ~read_internal & ~wait_en;
`ifdef TYPE8052
assign ram_rd_en_idata = work_en & rd_en_idata & ~same_flag_idata & ~wait_en;
`endif 
assign ram_rd_en_xdata = work_en & rd_en_xdata & ~same_flag_xdata & ~wait_en;


always @*
    rd_addr = 16'd0;
	
assign ram_rd_addr = rd_addr;	

assign use_psw_rs = 1'b0;

assign use_dp = 1'b0;

assign use_acc = 1'b0;

assign use_sp = 1'b0;

assign wait_en = (use_psw_rs&wr_psw_rs)|(use_dp&wr_dp)|(use_acc&wr_acc)|(use_sp&wr_sp);
```

接着我们详细讲解这段代码。先来看rd_en_data_idata信号和rd_en_xdata信号。

```
assign rd_en_data_idata = add_a_ri(cmdb)|addc_a_ri(cmdb)|subb_a_ri(cmdb)|inc_ri(cmdb)|dec_ri(cmdb)|anl_a_ri(cmdb)|orl_a_ri(cmdb)|xrl_a_ri(cmdb)|mov_a_ri(cmdb)|mov_di_ri(cmdb)|xch_a_ri(cmdb)|xchd(cmdb)|cjne_ri_da_rel(cmdb)|ret(cmda)|ret(cmdb)|reti(cmda)|reti(cmdb)|pop(cmda);

assign rd_en_xdata = movx_a_ri(cmdb)|movx_a_dp(cmda);
```

首先需要知道哪些指令要读取IDATA区域。对于寄存器间接寻址型指令而言，如果是8051型，则从DATA区或SFR区取出数据，如果是8052型，则在IDATA区取出数据。也就是说，8051的寄存器Ri中保存的地址是指向SFR的，而8052则是指向IDATA的，尽管它们的地址可能是相同的。这需要我们对指令和型号进行识别。除了明显的寄存器寻址指令，值得注意的是XCHD，使用Ri的CJNE，以及需要使用堆栈读取的RET，RETI和POP指令都需要访问IDATA区。其中大部分指令的读取请求在cmdb发出，但一部分不需要使用第二字节地址的指令可以在cmda发出读请求。

对于XDATA区数据，只有2种指令需要读取，MOVX A,@Ri在cmdb发出读XDATA请求，因为它首先要从Ri中读取真正数据的地址，因此第二个周期才能得到XDATA区的地址。MOVX A,@DPTR在cmda就可以发出读XDATA请求，因为这个地址直接由DPTR提供。

对于SFR区的读取与上面类似，但注意ACC寄存器也存放在这个区域，所以绝大多数指令都需要读取SFR区，

```verilog
assign rd_en_data_sfr = add_a_rn(cmda)|add_a_di(cmdb)|add_a_ri(cmda)|addc_a_rn(cmda)|addc_a_di(cmdb)|addc_a_ri(cmda)|subb_a_rn(cmda)|subb_a_di(cmdb)|subb_a_ri(cmda)|inc_rn(cmda)|inc_di(cmdb)|inc_ri(cmda)|dec_rn(cmda)|dec_di(cmdb)|dec_ri(cmda)|anl_a_rn(cmda)|anl_a_di(cmdb)|anl_a_ri(cmda)|anl_di_a(cmdb)|anl_di_da(cmdb)|orl_a_rn(cmda)|orl_a_di(cmdb)|orl_a_ri(cmda)|orl_di_a(cmdb)|orl_di_da(cmdb)|xrl_a_rn(cmda)|xrl_a_di(cmdb)|xrl_a_ri(cmda)|xrl_di_a(cmdb)|xrl_di_da(cmdb)|mov_a_rn(cmda)|mov_a_di(cmdb)|mov_a_ri(cmda)|mov_rn_di(cmdb)|mov_di_rn(cmda)|mov_di_di(cmdb)|mov_di_ri(cmda)|mov_ri_a(cmda)|mov_ri_di(cmda)|mov_ri_di(cmdb)|mov_ri_da(cmda)|movx_a_ri(cmda)|movx_ri_a(cmda)|push(cmdb)|xch_a_rn(cmda)|xch_a_di(cmdb)|xch_a_ri(cmda)|xchd(cmda)|clr_bit(cmdb)|setb_bit(cmdb)|cpl_bit(cmdb)|anl_c_bit(cmdb)|anl_c_nbit(cmdb)|orl_c_bit(cmdb)|orl_c_nbit(cmdb)|mov_c_bit(cmdb)|mov_bit_c(cmdb)|jb(cmdb)|jnb(cmdb)|jbc(cmdb)|cjne_a_di_rel(cmdb)|cjne_rn_da_rel(cmda)|cjne_ri_da_rel(cmda)|djnz_rn_rel(cmda)|djnz_di_rel(cmdb);
```

使用直接寻址的指令，在cmdb读取SFR区，采用位寻址的指令也在cmdb发出读SFR请求。其中寄存器间接寻址指令在cmda读取SFR，cmdb读取IDATA，并不会发生冲突。

接着按照区域来划分，观察下面的代码：

```verilog
assign rd_en_data = (rd_en_data_sfr|rd_en_data_idata) & ~rd_addr[7];
`ifdef TYPE8052
assign rd_en_sfr = rd_en_data_sfr & rd_addr[7];
assign rd_en_idata = rd_en_data_idata & rd_addr[7];
`else
assign rd_en_sfr = (rd_en_data_sfr|rd_en_data_idata) & rd_addr[7];
`endif
```

rd_en_data表示对DATA区的读操作，包含两类指令，SFR和IDATA，它们都需要对DATA区读取。只有在读地址第8位为0时，读取的是DATA区。（注意，DATA区地址是00H-7FH，需要读取SFR和IDATA的指令都需要读取DATA区，因此通过地址来判断读取落在DATA区（地址低位）还是SFR或IDATA区（地址高位））

接着判断是否是8052型，如果是8052型，SFR和IDATA区单独区分，如果不是8052型，则在读地址第8位为1时全部都是读取SFR区。

这里的rd_en_data是内部发起的DATA区读信号，它只是通过当前指令进行的判断。但实际上发给ram的读取使能是ram_rd_en_data。在以下两种情况下，是不需要真正发起读使能的。一种情况是我们前面说过的写后读冲突，发生写后读冲突时不能进行读操作：

```verilog
assign same_flag_data = rd_en_data & wr_en_data & (rd_addr[7:0]==wr_addr[7:0]);
assign same_flag_sfr = rd_en_sfr & wr_en_sfr & (rd_addr[7:0]==wr_addr[7:0]);
`ifdef TYPE8052
assign same_flag_idata = rd_en_idata & wr_en_idata & (rd_addr[7:0]==wr_addr[7:0]);
`endif
assign same_flag_xdata = rd_en_xdata & wr_en_xdata & (rd_addr[15:0]==wr_addr[15:0]);
```

第二种情况是读取到内部寄存器，包括ACC、B、SP、PSW、DPL和DPH，这些寄存器经常用到，我们直接写成reg形式，而不从ram中读取：

```verilog
assign read_internal = rd_en_sfr & ( (rd_addr[7:0]==8'he0)|(rd_addr[7:0]==8'hd0)|(rd_addr[7:0]==8'h83)|(rd_addr[7:0]==8'h82)|(rd_addr[7:0]==8'h81)|(rd_addr[7:0]==8'hf0) );
```

这样处理后，真正发给ram的读使能是以下代码：

```verilog
assign ram_rd_en_data = work_en & rd_en_data & ~same_flag_data & ~wait_en;
assign ram_rd_en_sfr = work_en & rd_en_sfr & ~same_flag_sfr & ~read_internal & ~wait_en;
`ifdef TYPE8052
assign ram_rd_en_idata = work_en & rd_en_idata & ~same_flag_idata & ~wait_en;
`endif 
assign ram_rd_en_xdata = work_en & rd_en_xdata & ~same_flag_xdata & ~wait_en;
```

接着计算读地址，同时标记出需要读内部寄存器的情况：

```verilog
always @*
    rd_addr = 16'd0;
	
assign ram_rd_addr = rd_addr;	

assign use_psw_rs = 1'b0;

assign use_dp = 1'b0;

assign use_acc = 1'b0;

assign use_sp = 1'b0;

assign wait_en = (use_psw_rs&wr_psw_rs)|(use_dp&wr_dp)|(use_acc&wr_acc)|(use_sp&wr_sp);
```

我们先搭建了一个整体框架，之后针对指令进行补充。这里的use_xxx主要用于标记寄存器被占用，防止出现写后读冲突。

##### 5.3.6 软核处理器的写操作

```verilog
//ram_wr_en ram_wr_addr  

assign wr_en_data_sfr = inc_rn(cmdb)|inc_di(cmdc)|dec_rn(cmdb)|dec_di(cmdc)|anl_di_a(cmdc)|anl_di_da(cmdc)|orl_di_a(cmdc)|orl_di_da(cmdc)|xrl_di_a(cmdc)|xrl_di_da(cmdc)|mov_rn_a(cmdb)|mov_rn_di(cmdc)|mov_rn_da(cmdb)|mov_di_a(cmdb)|mov_di_rn(cmdb)|mov_di_di(cmdc)|mov_di_ri(cmdc)|mov_di_da(cmdc)|pop(cmdb)|xch_a_rn(cmdb)|xch_a_di(cmdc)|clr_bit(cmdc)|setb_bit(cmdc)|cpl_bit(cmdc)|mov_bit_c(cmdc)|(jbc(cmdc)&data0[cmd1[2:0]])|djnz_rn_rel(cmdb)|djnz_di_rel(cmdc);

assign wr_en_data_idata = inc_ri(cmdc)|dec_ri(cmdc)|mov_ri_a(cmdb)|mov_ri_di(cmdc)|mov_ri_da(cmdb)|xch_a_ri(cmdc)|xchd(cmdc)|acall(cmdb)|acall(cmdc)|lcall(cmdb)|lcall(cmdc)|push(cmdc);

assign wr_en_xdata = movx_ri_a(cmdb)|movx_dp_a(cmdb);

assign wr_en_data = (wr_en_data_sfr|wr_en_data_idata) & ~wr_addr[7];
`ifdef TYPE8052
assign wr_en_sfr = wr_en_data_sfr & wr_addr[7];
assign wr_en_idata = wr_en_data_idata & wr_addr[7];
`else
assign wr_en_sfr = (wr_en_data_sfr|wr_en_data_idata) & wr_addr[7];
`endif
assign write_internal = wr_en_sfr & ( (wr_addr[7:0]==8'he0)|(wr_addr[7:0]==8'hd0)|(wr_addr[7:0]==8'h83)|(wr_addr[7:0]==8'h82)|(wr_addr[7:0]==8'h81)|(wr_addr[7:0]==8'hf0) );
assign ram_wr_en_data = work_en & wr_en_data;
assign ram_wr_en_sfr = work_en & wr_en_sfr & ~write_internal;
`ifdef TYPE8052
assign ram_wr_en_idata = work_en & wr_en_idata;
`endif
assign ram_wr_en_xdata = work_en & wr_en_xdata;

always @*
    wr_addr = 16'd0;

assign ram_wr_addr = wr_addr;	
```

写操作的写使能和写地址的实现思路几乎与读操作完全一致，因此不再赘述。

写操作额外需要生成写数据：

```verilog
//ram_wr_byte

always @* begin
	wr_bit_byte = data0;
end

always @*
    wr_byte = 8'd0;

assign ram_wr_byte = wr_byte;
```

详细的，关于特定指令的结果操作，会在后面叙述。

##### 5.3.7 ACC寄存器

ACC寄存器是使用最频繁的寄存器，加减乘除，地址或数据存放都要使用ACC寄存器。

首先来看加减法器，它要输出8位的结果，还要输出PSW中的状态位：（在此处，我们先拿到一个整体框架，之后再填充）

```verilog
always @*
add_a = 8'b0;

always @*
add_b = 8'b0;

always @*
add_c = 8'b0;

always @*
sub_flag = 8'b0;

assign {bit_ac,low} = sub_flag ? (add_a[3:0]-add_b[3:0]-add_c) : (add_a[3:0]+add_b[3:0]+add_c);
assign high = sub_flag ? (add_a[6:4]-add_b[6:4]-bit_ac) : (add_a[6:4]+add_b[6:4]+bit_ac);
assign {bit_c,bit_high} = sub_flag ? (add_a[7]-add_b[7]-high[3]) : (add_a[7]+add_b[7]+high[3]);
assign bit_ov = bit_c ^ high[3];
assign add_byte = {bit_high,high[2:0],low};
```

在进行加减运算之前，需要准备好加数或被减数，加数或减数，用于加减法计算的额外一个位和减法使能标志，add_byte是结果输出，bit_ac，bit_c，bit_ov是PSW中AC,CY,OV标志的结果。

乘除法使用的是ACC寄存器和B寄存器进行运算：

```verilog
assign mult = acc * b;
assign {div_rem,div_ans} = divide(acc,b);
```

逻辑运算也需要用到ACC：

```verilog
assign and_out = acc & data0;
assign or_out  = acc | data0;
assign xor_out = acc ^ data0;
```

对于ACC寄存器的RTL描述如下：

```verilog
always @ ( posedge clk or posedge rst )
if ( rst )
    acc <= 8'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'he0 ) )
	    acc <= wr_byte;
	else;
else;

assign wr_acc = (wr_en_sfr & (wr_addr[7:0]==8'he0));
```

ACC寄存器的地址是8'hE0，当需要读取这个地址时，可以直接读取寄存器而不需要读取ram。

##### 5.3.8 PSW寄存器

PSW寄存器是状态位寄存器，我们同样还是先搭起来框架，内容在之后补充：

```verilog
//psw register

assign psw_p = ^acc;

always @ ( posedge clk or posedge rst )
if ( rst )
    psw_ov <= 1'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'hd0 ) )
	    psw_ov <= wr_byte[2];
	else;
else;

always @ ( posedge clk or posedge rst )
if ( rst )
    psw_ac <= 1'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'hd0 ) )
	    psw_ac <= wr_byte[6];
	else;
else;

always @ ( posedge clk or posedge rst )
if ( rst )
    psw_c <= 1'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'hd0 ) )
	    psw_c <= wr_byte[7];
	else;
else;

always @ ( posedge clk or posedge rst )
if ( rst )
    psw_other <= 4'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'hd0 ) )
	    psw_other <= {wr_byte[5:3],wr_byte[1]};
	else;
else;

assign psw_rs = psw_other[2:1];

assign wr_psw_rs = wr_en_sfr & (wr_addr[7:0]==8'hd0);

assign psw = {psw_c,psw_ac,psw_other[3:1],psw_ov,psw_other[0],psw_p};
```

这里我们只实现了当外部指令需要写PSW寄存器的时候PSW寄存器的变换，而没有添加不同指令执行导致的PSW寄存器变化，这些我们在添加指令时再完成。

##### 5.3.9 DPTR、SP和B寄存器

同样，我们只实现了指令需要写入该寄存器时，寄存器的变化。由于指令导致的寄存器自动改变将会在后面补充。

```verilog
//dp  sp registers

always @ ( posedge clk or posedge rst )
if ( rst )
    dp <= 16'b0;
else if ( work_en )
    if  ( wr_en_sfr & (wr_addr[7:0]==8'h82 ) )
	    dp[7:0] <= wr_byte;
	else;
else;

assign wr_dp = (wr_en_sfr & ((wr_addr[7:0]==8'h82)|(wr_addr[7:0]==8'h83)))|inc_dp(cmdb);

always @ ( posedge clk or posedge rst )
if ( rst )
    sp <= 8'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'h81) )
	    sp <= wr_byte;
	else;
else;

assign sp_sub1 = sp - 1'b1;

assign sp_add1 = sp + 1'b1;

assign wr_sp = (wr_en_sfr & (wr_addr[7:0]==8'h81));

always @ ( posedge clk or posedge rst )
if ( rst )
    b <= 8'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'hf0) )
	    b <= wr_byte;
	else;
else;
```

##### 5.3.10 RAM的输出

当读寄存器时发现地址属于内部寄存器，则不需要从外部RAM读取数据，此时直接将内部寄存器的数值作为存储区输出；否则通过读取RAM来获得数据输出：

```verilog
//ram output

always @*
if ( same_flag )
    data0 = same_byte;
else if ( same_byte[7] )
    data0 = acc;
else if ( same_byte[6] )
    data0 = psw;
else if ( same_byte[5] )
    data0 = dp[15:8];
else if ( same_byte[4] )
    data0 = dp[7:0];
else if ( same_byte[3] )
    data0 = sp;
else if ( same_byte[2] )
    data0 = b;
else
    data0 = ram_rd_byte;

always @ ( posedge clk or posedge rst )
if ( rst )
    data1 <= 8'b0;
else if ( work_en )
    data1 <= data0;
else;
    
always @ ( posedge clk or posedge rst )
if ( rst )
    same_flag <= 1'b0;
else if ( work_en )
`ifdef TYPE8052
    if ( same_flag_data|same_flag_sfr|same_flag_idata|same_flag_xdata )
`else 
    if ( same_flag_data|same_flag_sfr|same_flag_xdata )
`endif	
	    same_flag <= 1'b1;
	else
	    same_flag <= 1'b0;
else;

always @ ( posedge clk or posedge rst )
if ( rst )
    same_byte <= 8'd0;
else if ( work_en )
`ifdef TYPE8052
    if ( same_flag_data|same_flag_sfr|same_flag_idata|same_flag_xdata )
`else
    if ( same_flag_data|same_flag_sfr|same_flag_xdata )
`endif	
	    same_byte <= wr_byte;
	else if ( rd_en_sfr & (rd_addr[7:0]==8'he0) ) //acc
	    same_byte <= 1'b1<<7;
	else if ( rd_en_sfr & (rd_addr[7:0]==8'hd0) ) //psw
	    same_byte <= 1'b1<<6;
	else if ( rd_en_sfr & (rd_addr[7:0]==8'h83) ) //dph
	    same_byte <= 1'b1<<5;	
	else if ( rd_en_sfr & (rd_addr[7:0]==8'h82) ) //dpl
	    same_byte <= 1'b1<<4;	
	else if ( rd_en_sfr & (rd_addr[7:0]==8'h81) ) //sp
	    same_byte <= 1'b1<<3;
	else if ( rd_en_sfr & (rd_addr[7:0]==8'hf0) ) //b
	    same_byte <= 1'b1<<2;		
	else
	    same_byte <= 8'b0;
else;
```

data0是存储区的输出数据，data1是它延时一拍的输出数据。data0在读取外界存储器的时候dengyram_rd_byte，但在读取内部寄存器时直接被赋值为内部寄存器的值，当发生写后读冲突时不能发起读取。使用same_flag和same_data标记这两种情况，same_flag表示发生写后读冲突，此时把写数据送入same_data，并把它传回给data0；当读取到内部寄存器时，使用same_data的位作为标志，表示读取到哪个内部寄存器。此时same_flag=0，故不会发生冲突。

到这里，我们软核处理器的主要框架搭建完毕，之后只需要逐步添加指令的支持，便可以成为一个完善的处理器。

#### 5.4 为主体程序添加指令

接下来我们来为主体程序添加指令。在开始之前，我们有必要先把目前的代码展示出来：

```verilog
//`define TYPE8052
module r8051 (

input  wire        clk,
input  wire        rst,
input  wire        cpu_en,
input  wire        cpu_restart,
	
output reg         rom_en,
output reg  [15:0] rom_addr,
input  wire [7:0]  rom_byte,
input  wire        rom_vld,
	
output wire        ram_rd_en_data,
output wire        ram_rd_en_sfr,
`ifdef TYPE8052
output wire        ram_rd_en_idata,
`endif
output wire        ram_rd_en_xdata,
output wire [15:0] ram_rd_addr,
input  wire [7:0]  ram_rd_byte,
input  wire        ram_rd_vld,
	
output wire        ram_wr_en_data,
output wire        ram_wr_en_sfr,
`ifdef TYPE8052
output wire        ram_wr_en_idata,
`endif
output wire        ram_wr_en_xdata,
output wire [15:0] ram_wr_addr,
output wire [7:0]  ram_wr_byte

);

`include "instruction.v"

/*********************************************************/
//register definition

reg                rd_wait;
reg      [7:0]     cmd1;
reg      [7:0]     cmd2;
reg      [2:0]     cmd_flag;
reg      [15:0]    pc;
reg      [7:0]     acc;
reg                psw_ov;
reg                psw_ac;
reg                psw_c;
reg      [3:0]     psw_other;
reg      [15:0]    dp;
reg      [7:0]     sp;
reg      [7:0]     b;
reg                same_flag;
reg      [7:0]     same_byte;
reg      [7:0]     data1;


/*********************************************************/
//wire definition

wire               work_en;
wire     [7:0]     cmd0;
wire     [7:0]     cmda;
wire     [7:0]     cmdb;
wire     [7:0]     cmdc;
reg                pc_en;
wire     [15:0]    pc_add1;
wire     [15:0]    code_base;
wire     [15:0]    code_rel;
wire     [15:0]    code_addr;
wire               length1;
wire               length2r1;
wire               length2;
wire               length3;
wire               rd_en_data_sfr;
wire               rd_en_data_idata;
wire               rd_en_xdata;
wire               rd_en_data;
wire               rd_en_sfr;
`ifdef TYPE8052
wire               rd_en_idata;
`endif
wire               same_flag_data;
wire               same_flag_sfr;
`ifdef TYPE8052
wire               same_flag_idata;
`endif
wire               same_flag_xdata;
wire               read_internal;
reg      [15:0]    rd_addr;
wire               use_psw_rs;
wire               use_dp;
wire               use_acc;
wire               use_sp;
wire               wait_en;
wire               wr_en_data_sfr;
wire               wr_en_data_idata;
wire               wr_en_xdata;
wire               wr_en_data;
wire               wr_en_sfr;
`ifdef TYPE8052
wire               wr_en_idata;
`endif 
wire               write_internal;
reg      [15:0]    wr_addr;
reg      [7:0]     wr_bit_byte;
reg      [7:0]     wr_byte;
reg      [7:0]     add_a;
reg      [7:0]     add_b;
reg                add_c;
reg                sub_flag;
wire               bit_ac;
wire     [3:0]     low;
wire     [3:0]     high;
wire               bit_c;
wire               bit_high; 
wire               bit_ov; 
wire     [7:0]     add_byte;
wire     [15:0]    mult;
wire     [7:0]     and_out;
wire     [7:0]     or_out; 
wire     [7:0]     xor_out;
wire               wr_acc;
wire               psw_p;
wire     [1:0]     psw_rs;
wire               wr_psw_rs; 
wire     [7:0]     psw;
wire               wr_dp;
wire     [7:0]     sp_sub1;
wire     [7:0]     sp_add1;
wire               wr_sp;
wire     [7:0]     div_ans;
wire     [7:0]     div_rem;
reg      [7:0]     data0;


/*********************************************************/


/*********************************************************/
//work_en

always @ ( posedge clk or posedge rst )
if ( rst )
    rd_wait <= 1'b0;
else if ( cpu_restart )
    rd_wait <= 1'b0;	
`ifdef TYPE8052
else if ( ram_rd_en_data|ram_rd_en_sfr|ram_rd_en_idata|ram_rd_en_xdata )
`else
else if ( ram_rd_en_data|ram_rd_en_sfr|ram_rd_en_xdata ) 
`endif	
    rd_wait <= 1'b1;
else if ( ram_rd_vld )
    rd_wait <= 1'b0;	
else;

reg rom_wait;
always @ ( posedge clk or posedge rst )
if ( rst )
    rom_wait <= 1'b0;
else if ( cpu_restart )
    rom_wait <= 1'b0;	
else if ( rom_en )
	rom_wait <= 1'b1;
else if ( rom_vld )
    rom_wait <= 1'b0;
else;


assign work_en = ( ( rd_wait & ~ram_rd_vld )|( rom_wait & ~rom_vld ) ) ? 1'b0 : cpu_en;

/*********************************************************/

/*********************************************************/
//cmd0/cmd1/cmd2  cmda/cmdb/cmdc

assign cmd0 = rom_byte;

always @ ( posedge clk or posedge rst )
if ( rst )
    cmd1 <= 8'b0;
else if ( work_en )
    cmd1 <= cmd0;
else;

always @ ( posedge clk or posedge rst )
if ( rst )
    cmd2 <= 8'b0;
else if ( work_en )
    cmd2 <= cmdb;
else;

always @ ( posedge clk or posedge rst )
if ( rst )
    cmd_flag <= 3'b1;
else if ( cpu_restart )
    cmd_flag <= 3'b1;
else if ( work_en )
    if ( wait_en )
	    cmd_flag <= {1'b0,cmd_flag[1:0]};
    else if ( length2|length2r1 )
	    cmd_flag <= 3'b101;
	else if ( length3 )
	    cmd_flag <= 3'b100;
	else
        cmd_flag <= { cmd_flag[1:0],1'b1};
else;

assign cmda = cmd_flag[1] ? cmd0 : 8'b0;

assign cmdb = cmd_flag[2] ? cmd1 : 8'b0;

assign cmdc = cmd2;


/*********************************************************/


/*********************************************************/
//rom_en rom_addr

always @*
if ( ~work_en )
    pc_en = 1'b0;
else if ( wait_en|length2r1 )
    pc_en = 1'b0;
else
    pc_en = 1'b1;

always @ ( posedge clk or posedge rst )
if ( rst )
    pc <= 16'd0;
else if ( cpu_restart )
    pc <= 16'd0;
else if ( work_en )
    if ( pc_en )
        pc <= pc_add1;
	else;
else;

assign pc_add1 = rom_addr + 1'b1;

always @*
if ( ~work_en )
    rom_en = 1'b0;
else if ( wait_en )
    rom_en = 1'b0;
else if ( length2r1 )
    rom_en = 1‘b0;
else
    rom_en = 1'b1;

assign code_base = 1'b0 ? dp : pc;

assign code_rel = 1'b0 ? &#123;&#123;8{cmd0[7]&#125;&#125;,cmd0} : &#123;&#123;8{acc[7]&#125;&#125;,acc};	
	
assign code_addr = code_base + code_rel;	

always @*
    rom_addr = 1'b0 ? code_addr : pc;
/*********************************************************/

/*********************************************************/
//command length

assign  length1 = 1'b0;

assign  length2r1 = 1'b0;

assign  length2 = 1'b0;

assign  length3 = 1'b0;


/*********************************************************/


/*********************************************************/
//ram_rd_en ram_rd_addr 

assign rd_en_data_sfr = 1'b0;

assign rd_en_data_idata = 1'b0;

assign rd_en_xdata = 1'b0;

assign rd_en_data = (rd_en_data_sfr|rd_en_data_idata) & ~rd_addr[7];
`ifdef TYPE8052
assign rd_en_sfr = rd_en_data_sfr & rd_addr[7];
assign rd_en_idata = rd_en_data_idata & rd_addr[7];
`else
assign rd_en_sfr = (rd_en_data_sfr|rd_en_data_idata) & rd_addr[7];
`endif
assign same_flag_data = rd_en_data & wr_en_data & (rd_addr[7:0]==wr_addr[7:0]);
assign same_flag_sfr = rd_en_sfr & wr_en_sfr & (rd_addr[7:0]==wr_addr[7:0]);
`ifdef TYPE8052
assign same_flag_idata = rd_en_idata & wr_en_idata & (rd_addr[7:0]==wr_addr[7:0]);
`endif
assign same_flag_xdata = rd_en_xdata & wr_en_xdata & (rd_addr[15:0]==wr_addr[15:0]);
assign read_internal = rd_en_sfr & ( (rd_addr[7:0]==8'he0)|(rd_addr[7:0]==8'hd0)|(rd_addr[7:0]==8'h83)|(rd_addr[7:0]==8'h82)|(rd_addr[7:0]==8'h81)|(rd_addr[7:0]==8'hf0) );
assign ram_rd_en_data = work_en & rd_en_data & ~same_flag_data & ~wait_en;
assign ram_rd_en_sfr = work_en & rd_en_sfr & ~same_flag_sfr & ~read_internal & ~wait_en;
`ifdef TYPE8052
assign ram_rd_en_idata = work_en & rd_en_idata & ~same_flag_idata & ~wait_en;
`endif 
assign ram_rd_en_xdata = work_en & rd_en_xdata & ~same_flag_xdata & ~wait_en;


always @*
    rd_addr = 16'd0;
	
assign ram_rd_addr = rd_addr;	

assign use_psw_rs = 1'b0;

assign use_dp = 1'b0;

assign use_acc = 1'b0;

assign use_sp = 1'b0;

assign wait_en = 1'b0;

/*********************************************************/

/*********************************************************/
//ram_wr_en ram_wr_addr  

assign wr_en_data_sfr = 1'b0;

assign wr_en_data_idata = 1'b0;

assign wr_en_xdata = 1'b0;

assign wr_en_data = (wr_en_data_sfr|wr_en_data_idata) & ~wr_addr[7];
`ifdef TYPE8052
assign wr_en_sfr = wr_en_data_sfr & wr_addr[7];
assign wr_en_idata = wr_en_data_idata & wr_addr[7];
`else
assign wr_en_sfr = (wr_en_data_sfr|wr_en_data_idata) & wr_addr[7];
`endif
assign write_internal = wr_en_sfr & ( (wr_addr[7:0]==8'he0)|(wr_addr[7:0]==8'hd0)|(wr_addr[7:0]==8'h83)|(wr_addr[7:0]==8'h82)|(wr_addr[7:0]==8'h81)|(wr_addr[7:0]==8'hf0) );
assign ram_wr_en_data = work_en & wr_en_data;
assign ram_wr_en_sfr = work_en & wr_en_sfr & ~write_internal;
`ifdef TYPE8052
assign ram_wr_en_idata = work_en & wr_en_idata;
`endif
assign ram_wr_en_xdata = work_en & wr_en_xdata;

always @*
    wr_addr = 16'd0;

assign ram_wr_addr = wr_addr;	

/*********************************************************/


/*********************************************************/
//ram_wr_byte

always @* begin
wr_bit_byte = data0;
end

always @*
    wr_byte = 8'd0;

assign ram_wr_byte = wr_byte;

/*********************************************************/


/*********************************************************/
//acc register

always @*
    add_a = 8'b0;


always @*
    add_b = 8'b0;


always @*
    add_c = 1'b0;	


always @*
    sub_flag = 1'b0;


assign {bit_ac,low} = sub_flag ? (add_a[3:0]-add_b[3:0]-add_c) : (add_a[3:0]+add_b[3:0]+add_c);
assign high = sub_flag ? (add_a[6:4]-add_b[6:4]-bit_ac) : (add_a[6:4]+add_b[6:4]+bit_ac);
assign {bit_c,bit_high} = sub_flag ? (add_a[7]-add_b[7]-high[3]) : (add_a[7]+add_b[7]+high[3]);
assign bit_ov = bit_c ^ high[3];
assign add_byte = {bit_high,high[2:0],low};

assign mult = acc * b;
assign {div_rem,div_ans} = divide(acc,b);
assign and_out = acc & data0;
assign or_out  = acc | data0;
assign xor_out = acc ^ data0;

wire [3:0] da_low  = ( psw_ac|(acc[3:0]>4'h9) ) ? (acc[3:0]+4'd6) : acc[3:0];
wire [3:0] da_high = ( ( psw_c|(acc[7:4]>4'h9)|((acc[7:4]==4'h9)&(psw_ac|(acc[3:0]>4'h9))) ) ? ( acc[7:4]+4'h6 ) : acc[7:4] ) + ( psw_ac|(acc[3:0]>4'h9) );
	
always @ ( posedge clk or posedge rst )
if ( rst )
    acc <= 8'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'he0 ) )
	    acc <= wr_byte;
	else;
else;

assign wr_acc = wr_en_sfr & (wr_addr[7:0]==8'he0);

/*********************************************************/


/*********************************************************/
//psw register

assign psw_p = ^acc;

always @ ( posedge clk or posedge rst )
if ( rst )
    psw_ov <= 1'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'hd0 ) )
	    psw_ov <= wr_byte[2];
	else;
else;

always @ ( posedge clk or posedge rst )
if ( rst )
    psw_ac <= 1'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'hd0 ) )
	    psw_ac <= wr_byte[6];
	else;
else;

always @ ( posedge clk or posedge rst )
if ( rst )
    psw_c <= 1'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'hd0 ) )
	    psw_c <= wr_byte[7];
	else;
else;

always @ ( posedge clk or posedge rst )
if ( rst )
    psw_other <= 4'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'hd0 ) )
	    psw_other <= {wr_byte[5:3],wr_byte[1]};
	else;
else;

assign psw_rs = psw_other[2:1];

assign wr_psw_rs = wr_en_sfr & (wr_addr[7:0]==8'hd0);

assign psw = {psw_c,psw_ac,psw_other[3:1],psw_ov,psw_other[0],psw_p};

/*********************************************************/

/*********************************************************/
//dp  sp registers

always @ ( posedge clk or posedge rst )
if ( rst )
    dp <= 16'b0;
else if ( work_en )
    if  ( wr_en_sfr & (wr_addr[7:0]==8'h82 ) )
	    dp[7:0] <= wr_byte;
	else if  ( wr_en_sfr & (wr_addr[7:0]==8'h83 ) )
	    dp[15:8] <= wr_byte;
	else;
else;

assign wr_dp = (wr_en_sfr & ((wr_addr[7:0]==8'h82)|(wr_addr[7:0]==8'h83)))|inc_dp(cmdb);

always @ ( posedge clk or posedge rst )
if ( rst )
    sp <= 8'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'h81) )
	    sp <= wr_byte;
	else;
else;

assign sp_sub1 = sp - 1'b1;

assign sp_add1 = sp + 1'b1;

assign wr_sp = (wr_en_sfr & (wr_addr[7:0]==8'h81));

always @ ( posedge clk or posedge rst )
if ( rst )
    b <= 8'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'hf0) )
	    b <= wr_byte;
	else;
else;

/*********************************************************/


/*********************************************************/
//ram output

always @*
if ( same_flag )
    data0 = same_byte;
else if ( same_byte[7] )
    data0 = acc;
else if ( same_byte[6] )
    data0 = psw;
else if ( same_byte[5] )
    data0 = dp[15:8];
else if ( same_byte[4] )
    data0 = dp[7:0];
else if ( same_byte[3] )
    data0 = sp;
else if ( same_byte[2] )
    data0 = b;
else
    data0 = ram_rd_byte;

always @ ( posedge clk or posedge rst )
if ( rst )
    data1 <= 8'b0;
else if ( work_en )
    data1 <= data0;
else;
    
always @ ( posedge clk or posedge rst )
if ( rst )
    same_flag <= 1'b0;
else if ( work_en )
`ifdef TYPE8052
    if ( same_flag_data|same_flag_sfr|same_flag_idata|same_flag_xdata )
`else 
    if ( same_flag_data|same_flag_sfr|same_flag_xdata )
`endif	
	    same_flag <= 1'b1;
	else
	    same_flag <= 1'b0;
else;

always @ ( posedge clk or posedge rst )
if ( rst )
    same_byte <= 8'd0;
else if ( work_en )
`ifdef TYPE8052
    if ( same_flag_data|same_flag_sfr|same_flag_idata|same_flag_xdata )
`else
    if ( same_flag_data|same_flag_sfr|same_flag_xdata )
`endif	
	    same_byte <= wr_byte;
	else if ( rd_en_sfr & (rd_addr[7:0]==8'he0) ) //acc
	    same_byte <= 1'b1<<7;
	else if ( rd_en_sfr & (rd_addr[7:0]==8'hd0) ) //psw
	    same_byte <= 1'b1<<6;
	else if ( rd_en_sfr & (rd_addr[7:0]==8'h83) ) //dph
	    same_byte <= 1'b1<<5;	
	else if ( rd_en_sfr & (rd_addr[7:0]==8'h82) ) //dpl
	    same_byte <= 1'b1<<4;	
	else if ( rd_en_sfr & (rd_addr[7:0]==8'h81) ) //sp
	    same_byte <= 1'b1<<3;
	else if ( rd_en_sfr & (rd_addr[7:0]==8'hf0) ) //b
	    same_byte <= 1'b1<<2;		
	else
	    same_byte <= 8'b0;
else;
	
/*********************************************************/

endmodule

```

##### 5.4.1 算数操作指令的添加

指令添加最方便的方法是一条一条添加，由于每一条单独的指令都很简单，只需要对某些地方修改，并让主体程序的读写过程满足该指令的要求，那么这条指令就完成了。

算术操作指令有24条，主要是完成数据的加减乘除等算术运算，一般流程是从数据存储区取得某个数据，让它和其他数据进行算术运算，将结果写入数据存储区的某个地址。

第一步，我们完善指令的字节长度判断，顺便把这些指令罗列出来：

```verilog
//command length

assign  length1 = add_a_rn(cmda)|addc_a_rn(cmda)|subb_a_rn(cmda)|inc_a(cmda)|inc_rn(cmda)|inc_dp(cmda)|dec_a(cmda)|dec_rn(cmda)|mul(cmda)|div(cmda)|da(cmda);

assign  length2r1 = add_a_ri(cmda)|addc_a_ri(cmda)|subb_a_ri(cmda)|inc_ri(cmda)|dec_ri(cmda);

assign  length2 = add_a_di(cmda)|add_a_da(cmda)|addc_a_di(cmda)|addc_a_da(cmda)|subb_a_di(cmda)|subb_a_da(cmda)|inc_di(cmda)|dec_di(cmda);

assign  length3 = 1'b0;
```

算术操作指令只有单字节和双字节，没有三字节。

单字节指令分为`length1`和`length2r1`，`length2r1`表示它虽然是单字节指令，但要当成两字节处理，但指令存储区没有存放单字节的第二虚拟字节，在处理器读到`length2r1`的任何一条指令后，程序将停止一个周期再读下一条指令。

观察后可以发现，`length2r1`的指令都带有`_ri`字样，这表示寄存器间接寻址指令需要花费两个周期才能完成操作，在前面已经解释过了。

接着来看读使能和读地址的赋值部分，需要从外部RAM读取的指令，将读使能拉高。首先分析一下单字节的指令：

```verilog
add_a_rn(cmda)|addc_a_rn(cmda)|subb_a_rn(cmda)|inc_a(cmda)|inc_rn(cmda)|inc_dp(cmda)|dec_a(cmda)|dec_rn(cmda)|mul(cmda)|div(cmda)|da(cmda)
```

单字节的指令在`cmda`执行读操作。`rd_en`信号分为三类，第一类`rd_en_data_sfr`是读取DATA区和SFR区，具体是哪个区由地址的位区分（低地址是DATA区，高地址是SFR区）；第二类`rd_en_data_idata`是读取DATA区，IDATA区，具体是哪个区取决于处理器是否是8052类型，如果是那么取决于地址位（低地址是DATA区，高地址是SFR区），如果不是则全部是读取DATA区；第三类`rd_en_xdata`是对XDATA区读取。

以加法指令为例详细分析一下：

**第一条加法指令**是`ADD A,Rn`，是单字节指令，使用寄存器寻址，指令标识是`add_a_rn`。它先从Rn中读取数据，将其与ACC中的数据相加，最后将结果写入ACC中。因此，它需要读取SFR区（读取ACC）和DATA区（读取Rn）。由于我们将ACC寄存器单独设置，所以这里只需要读取DATA区以取出Rn中的操作数。DATA区的寄存器组区中的寄存器分为4组，特殊寄存器PSW的RS[1:0]决定了是这四组中的哪一组，而指令中的后三位决定了是这一组中的R0到R7的哪一个。因此究竟读取的地址是什么，要通过拼接算出来，实际上由于DATA区是地址的低部分，所以只需要低5位使用`{RS[1:0],cmd0[2:0]}`即可。

因此，体现在读使能上是`rd_en_data_sfr`拉高，并且将读地址`rd_addr`赋值。如下所示：

```verilog
assign rd_en_data_sfr = add_a_rn(cmda);

always @*
	if(add_a_rn(cmda))
		rd_addr = {psw_rs, cmd0[2:0]};
	else
    	rd_addr = 16'd0;
```

**第二条加法指令**是`ADD A, Direct`，是双字节指令，采用直接寻址，指令标识是`add_a_di`，指令的第二个字节存放的是操作数的地址，根据它的地址可以知道是从SFR区还是DATA区取出数据。体现在读使能上是`rd_en_data_sfr`拉高（添加后面的指令都应使用或逻辑连接多条指令），并且将读地址`rd_addr`直接赋值为双字节指令的第二个字节。注意这里双字节指令在`cmdb`发起读操作，这时cmd0存放了这条指令的第二个字节，直接把它赋值给`rd_addr`即可。

```verilog
assign rd_en_data_sfr = add_a_rn(cmda)|add_a_di(cmdb);

always @*
	if(add_a_rn(cmda))
		rd_addr = {psw_rs, cmd0[2:0]};
	else if(add_a_di(cmdb))
		rd_addr = cmd0;
	else
    	rd_addr = 16'd0;
```

**第三条加法指令**是`ADD A,@Ri`，是单字节指令，指令标识是`add_a_ri`，由于使用寄存器间接寻址，所以将其扩展为虚拟双字节指令。通过指令的[0]位可以知道该地址是放在R0还是R1中，从寄存器R0或R1内获取地址，并按照这个地址再从DATA区或SFR区（对于8052来说是从DATA区或IDATA区）取出数据，将其和ACC中的数据相加，最后将结果写入寄存器ACC中。虚拟双字节指令将会使能两次读操作，第一次在`cmda`时读取Ri中的地址，第二次使用Ri中的地址读出真正的数据。

```verilog
assign rd_en_data_sfr = add_a_rn(cmda)|add_a_di(cmdb)|add_a_ri(cmda);
assign rd_en_data_idata = add_a_ri(cmdb);

always @*
	if(add_a_rn(cmda))//使用寄存器寻址，读地址使用PSW_RS[1:0]和cmd0[2:0]拼接
        rd_addr = {psw_rs, cmd0[2:0]};
	else if(add_a_di(cmdb))//使用直接寻址，读地址使用cmd0
        rd_addr = cmd0;
	else if(add_a_ri(cmda))//使用寄存器间接寻址，读取Ri的地址使用PSW_RS[1:0]和cmd0[0]拼接
        rd_addr = {psw_rs,2'b0,cmd0[0]}
	else if(add_a_ri(cmdb))//使用寄存器间接寻址，读取到Ri的数据作为地址
        rd_addr = data0;
	else
        rd_addr = 16'd0;
```

**第四条加法指令**是`ADD A,#DATA`，双字节指令，指令标识`add_a_da`，采用立即寻址，第二个字节保存的是一个立即数，即直接参与运算的数据。直接将其与ACC中的数据相加，最后将结果写入寄存器ACC中即可。由于它不涉及到对于RAM的读取，因此它不占用读使能和读地址。

其他的算术指令与这四条加法指令类似，在此不作赘述，直接给出添加算术指令后的完整读使能和读地址：

```verilog
assign rd_en_data_sfr = add_a_rn(cmda)|add_a_di(cmdb)|add_a_ri(cmda)|addc_a_rn(cmda)|addc_a_di(cmdb)|addc_a_ri(cmda)|subb_a_rn(cmda)|subb_a_di(cmdb)|subb_a_ri(cmda)|inc_rn(cmda)|inc_di(cmdb)|inc_ri(cmda)|dec_rn(cmda)|dec_di(cmdb)|dec_ri(cmda);

assign rd_en_data_idata = add_a_ri(cmdb)|addc_a_ri(cmdb)|subb_a_ri(cmdb)|inc_ri(cmdb)|dec_ri(cmdb);

assign rd_en_xdata = 1'b0;

assign rd_en_data = (rd_en_data_sfr|rd_en_data_idata) & ~rd_addr[7];

always @*
if ( add_a_rn(cmda)|addc_a_rn(cmda)|subb_a_rn(cmda)|inc_rn(cmda)|dec_rn(cmda))//使用寄存器寻址，读地址使用PSW_RS[1:0]和cmd0[2:0]拼接
    rd_addr = { psw_rs,cmd0[2:0] };
else if ( add_a_di(cmdb)|addc_a_di(cmdb)|subb_a_di(cmdb)|inc_di(cmdb)|dec_di(cmdb))//使用直接寻址，读地址使用cmd0
    rd_addr = cmd0;
else if ( add_a_ri(cmda)|addc_a_ri(cmda)|subb_a_ri(cmda)|inc_ri(cmda)|dec_ri(cmda))//使用寄存器间接寻址，读取Ri的地址使用PSW_RS[1:0]和cmd0[0]拼接
    rd_addr = { psw_rs,2'b0,cmd0[0] };
else if ( add_a_ri(cmdb)|addc_a_ri(cmdb)|subb_a_ri(cmdb)|inc_ri(cmdb)|dec_ri(cmdb))//使用寄存器间接寻址，读取到Ri的数据作为地址
    rd_addr = data0;
else
    rd_addr = 16'd0;
```

分析读地址时，发现进行读操作需要用到PSW寄存器的RS段作为读地址的一部分，这会引起读地址与上一级的写数据冲突，因此必须用下面描述声明：凡用到psw_rs的读地址都要列出。如果上一条指令正好是写PSW寄存器的话，我们的主体程序会进行判别，并主动避开这种情况。

大部分指令将结果写入ACC寄存器，但也有部分指令需要写入存储器，它们需要进行写操作。对写使能和写地址赋值，同时需要确定写数据：

```verilog
assign use_psw_rs = add_a_rn(cmda)|add_a_ri(cmda)|addc_a_rn(cmda)|addc_a_ri(cmda)|subb_a_rn(cmda)|subb_a_ri(cmda)|inc_rn(cmda)|inc_ri(cmda)|dec_rn(cmda)|dec_ri(cmda);

//ram_wr_en ram_wr_addr  

assign wr_en_data_sfr = inc_rn(cmdb)|inc_di(cmdc)|dec_rn(cmdb)|dec_di(cmdc)

assign wr_en_data_idata = inc_ri(cmdc)|dec_ri(cmdc)

assign wr_en_xdata = 1'b0;

always @*
    if ( inc_rn(cmdb)|dec_rn(cmdb) )//写入Rn
    wr_addr = { psw_rs,cmd1[2:0] };
else if ( inc_di(cmdc)|dec_di(cmdc) )//直接地址作为写地址
    wr_addr = cmd1;
else if ( inc_ri(cmdc)|dec_ri(cmdc) )//以Ri作为写地址
    wr_addr = data1;
else
    wr_addr = 16'd0;

//wr_byte

always @*
else if ( mov_rn_a(cmdb)|mov_di_a(cmdb)|mov_ri_a(cmdb)|movx_ri_a(cmdb)|movx_dp_a(cmdb)|xch_a_rn(cmdb)|xch_a_di(cmdc)|xch_a_ri(cmdc) )
    wr_byte = acc;
else if ( mov_rn_di(cmdc)|mov_di_rn(cmdb)|mov_di_di(cmdc)|mov_di_ri(cmdc)|mov_ri_di(cmdc)|push(cmdc)|pop(cmdb) )
    wr_byte = data0;
else
    wr_byte = 8'd0;
```

接下来需要控制通用加减器。通用加减器输入有4项，分别是add_a，add_b，add_c和sub_flag。它们分别控制加数（被减数），加数（减数），额外的一个比特位和加减选择。上面的指令只需改变这四项即可实现不同的指令选择，如ADD，ADDC，SUBB，DEC，INC等。

```verilog
always @*
if ( add_a_rn(cmdb)|add_a_di(cmdc)|add_a_ri(cmdc)|add_a_da(cmdb)|addc_a_rn(cmdb)|addc_a_di(cmdc)|addc_a_ri(cmdc)|addc_a_da(cmdb)|subb_a_rn(cmdb)|subb_a_di(cmdc)|subb_a_ri(cmdc)|subb_a_da(cmdb)|inc_a(cmdb)|dec_a(cmdb) )
    add_a = acc;
else if ( inc_rn(cmdb)|inc_di(cmdc)|inc_ri(cmdc)|dec_rn(cmdb)|dec_di(cmdc)|dec_ri(cmdc))
    add_a = data0;
else
    add_a = 8'b0;


always @*
if ( add_a_rn(cmdb)|add_a_di(cmdc)|add_a_ri(cmdc)|addc_a_rn(cmdb)|addc_a_di(cmdc)|addc_a_ri(cmdc)|subb_a_rn(cmdb)|subb_a_di(cmdc)|subb_a_ri(cmdc) )
    add_b = data0;
else if ( add_a_da(cmdb)|addc_a_da(cmdb)|subb_a_da(cmdb) )
    add_b = cmd0;
else if ( inc_a(cmdb)|dec_a(cmdb)|inc_rn(cmdb)|inc_di(cmdc)|inc_ri(cmdc)|dec_rn(cmdb)|dec_di(cmdc)|dec_ri(cmdc))
    add_b = 8'b0;
else
    add_b = 8'b0;


always @*
if ( add_a_rn(cmdb)|add_a_di(cmdc)|add_a_ri(cmdc)|add_a_da(cmdb) )
    add_c = 1'b0;
else if (addc_a_rn(cmdb)|addc_a_di(cmdc)|addc_a_ri(cmdc)|addc_a_da(cmdb)|subb_a_rn(cmdb)|subb_a_di(cmdc)|subb_a_ri(cmdc)|subb_a_da(cmdb) )
    add_c = psw_c;
else if (inc_a(cmdb)|inc_rn(cmdb)|inc_di(cmdc)|inc_ri(cmdc)|dec_a(cmdb)|dec_rn(cmdb)|dec_di(cmdc)|dec_ri(cmdc))
    add_c = 1'b1;	
else
    add_c = 1'b0;	


always @*
if (add_a_rn(cmdb)|add_a_di(cmdc)|add_a_ri(cmdc)|add_a_da(cmdb)|addc_a_rn(cmdb)|addc_a_di(cmdc)|addc_a_ri(cmdc)|addc_a_da(cmdb)|inc_a(cmdb)|inc_rn(cmdb)|inc_di(cmdc)|inc_ri(cmdc) )
    sub_flag = 1'b0;
else if (subb_a_rn(cmdb)|subb_a_di(cmdc)|subb_a_ri(cmdc)|subb_a_da(cmdb)|dec_a(cmdb)|dec_rn(cmdb)|dec_di(cmdc)|dec_ri(cmdc))
    sub_flag = 1'b1;
else
    sub_flag = 1'b0;
```

之后需要写入寄存器，主要包括ACC寄存器和状态寄存器。

ACC寄存器的修改逻辑：

```verilog
wire [3:0] da_low  = ( psw_ac|(acc[3:0]>4'h9) ) ? (acc[3:0]+4'd6) : acc[3:0];
wire [3:0] da_high = ( ( psw_c|(acc[7:4]>4'h9)|((acc[7:4]==4'h9)&(psw_ac|(acc[3:0]>4'h9))) ) ? ( acc[7:4]+4'h6 ) : acc[7:4] ) + ( psw_ac|(acc[3:0]>4'h9) );
	
always @ ( posedge clk or posedge rst )
if ( rst )
    acc <= 8'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'he0 ) )
	    acc <= wr_byte;
	else if ( add_a_rn(cmdb)|add_a_di(cmdc)|add_a_ri(cmdc)|add_a_da(cmdb)|addc_a_rn(cmdb)|addc_a_di(cmdc)|addc_a_ri(cmdc)|addc_a_da(cmdb)|subb_a_rn(cmdb)|subb_a_di(cmdc)|subb_a_ri(cmdc)|subb_a_da(cmdb)|inc_a(cmdb)|dec_a(cmdb) )
	    acc <= add_byte;
	else if ( mul(cmdb) )
	    acc <= mult[7:0];
	else if ( div(cmdb) )
	    acc <= div_ans;
	else if ( da(cmdb) )
	    acc <= {da_high,da_low};	
	else;
else;
```

由于ACC寄存器也用作读地址，为了组织wait_en，我们也需要声明是否要读ACC寄存器的写操作：

```verilog
assign wr_acc = (wr_en_sfr & (wr_addr[7:0]==8'he0))|add_a_rn(cmdb)|add_a_di(cmdc)|add_a_ri(cmdc)|add_a_da(cmdb)|addc_a_rn(cmdb)|addc_a_di(cmdc)|addc_a_ri(cmdc)|addc_a_da(cmdb)|subb_a_rn(cmdb)|subb_a_di(cmdc)|subb_a_ri(cmdc)|subb_a_da(cmdb)|inc_a(cmdb)|dec_a(cmdb)|mul(cmdb)|div(cmdb)|da(cmdb);
```

之后需要修改PSW寄存器的特殊位：

```verilog
always @ ( posedge clk or posedge rst )
if ( rst )
    psw_ov <= 1'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'hd0 ) )
	    psw_ov <= wr_byte[2];
	else if ( add_a_rn(cmdb)|add_a_di(cmdc)|add_a_ri(cmdc)|add_a_da(cmdb)|addc_a_rn(cmdb)|addc_a_di(cmdc)|addc_a_ri(cmdc)|addc_a_da(cmdb)|subb_a_rn(cmdb)|subb_a_di(cmdc)|subb_a_ri(cmdc)|subb_a_da(cmdb) )
	    psw_ov <= bit_ov;
	else if ( mul(cmdb) )
	    psw_ov <= (mult[15:8]!=8'b0);
	else if ( div(cmdb) )
	    psw_ov <= (b==8'b0);
	else;
else;

always @ ( posedge clk or posedge rst )
if ( rst )
    psw_ac <= 1'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'hd0 ) )
	    psw_ac <= wr_byte[6];
	else if ( add_a_rn(cmdb)|add_a_di(cmdc)|add_a_ri(cmdc)|add_a_da(cmdb)|addc_a_rn(cmdb)|addc_a_di(cmdc)|addc_a_ri(cmdc)|addc_a_da(cmdb)|subb_a_rn(cmdb)|subb_a_di(cmdc)|subb_a_ri(cmdc)|subb_a_da(cmdb) )
	    psw_ac <= bit_ac;
	else;
else;

always @ ( posedge clk or posedge rst )
if ( rst )
    psw_c <= 1'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'hd0 ) )
	    psw_c <= wr_byte[7];
	else if ( add_a_rn(cmdb)|add_a_di(cmdc)|add_a_ri(cmdc)|add_a_da(cmdb)|addc_a_rn(cmdb)|addc_a_di(cmdc)|addc_a_ri(cmdc)|addc_a_da(cmdb)|subb_a_rn(cmdb)|subb_a_di(cmdc)|subb_a_ri(cmdc)|subb_a_da(cmdb) )
	    psw_c <= bit_c;
	else if ( mul(cmdb)|div(cmdb) )
	    psw_c <= 1'b0;
	else if ( da(cmdb) )
	    psw_c <= ( psw_c|(acc[7:4]>4'h9)|((acc[7:4]==4'h9)&(psw_ac|(acc[3:0]>4'h9))) ) ? 1'b1 : psw_c;
	else;
else;

always @ ( posedge clk or posedge rst )
if ( rst )
    psw_other <= 4'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'hd0 ) )
	    psw_other <= {wr_byte[5:3],wr_byte[1]};
	else;
else;
```

最后在进行乘除运算时需要修改B寄存器：

```verilog
always @ ( posedge clk or posedge rst )
if ( rst )
    b <= 8'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'hf0) )
	    b <= wr_byte;
	else if ( mul(cmdb) )
	    b <= mult[15:8];
	else if ( div(cmdb) )
	    b <= div_rem;
	else;
else;
```

这样，我们就完全实现了算术运算指令。之后的其他指令我们不再详细叙述，其修改的方式和添加算术指令类似，我们只针对一些特殊的地方进行说明。

##### 5.4.2 逻辑操作指令的添加

逻辑操作指令几乎与算术操作指令完全类似，除了计算结果的地方有所区别。结果的计算如下：

```verilog
assign and_out = ( anl_di_da(cmdc) ? cmd0 : acc ) & ( anl_a_da(cmdb) ? cmd0 : data0 );
assign or_out  = ( orl_di_da(cmdc) ? cmd0 : acc ) | ( orl_a_da(cmdb) ? cmd0 : data0 );
assign xor_out = ( xrl_di_da(cmdc) ? cmd0 : acc ) ^ ( xrl_a_da(cmdb) ? cmd0 : data0 );
```

只需要将这些结果按照对应的指令要求写入ACC寄存器即可。

##### 5.4.3 数据转移指令的添加

这里有两条特殊指令`movc_a_dp`和`movc_a_pc`，它们涉及到把指令存储区CODE区某地址的数据搬移到ACC寄存器上，对CODE区的访问是通过信号rom_en和rom_addr[15:0]完成的。

```verilog
always @*
if ( ~work_en )
    rom_en = 1'b0;
else if ( wait_en )
    rom_en = 1'b0;
else if ( length2r1 )
    rom_en = movc_a_dp(cmda)|movc_a_pc(cmda);
else if ( acall(cmdb)|ret(cmda)|ret(cmdb)|reti(cmda)|reti(cmdb) )
    rom_en = 1'b0;	
else
    rom_en = 1'b1;

assign code_base = ( movc_a_dp(cmda)|jmp(cmda) ) ? dp : pc;

assign code_rel = ( jc(cmdb)||jnc(cmdb)|jb(cmdc)|jnb(cmdc)|jbc(cmdc)|sjmp(cmdb)|jz(cmdb)|jnz(cmdb)|cjne_a_di_rel(cmdc)|cjne_a_da_rel(cmdc)|cjne_rn_da_rel(cmdc)|cjne_ri_da_rel(cmdc)|djnz_rn_rel(cmdb)|djnz_di_rel(cmdc) ) ? &#123;&#123;8{cmd0[7]&#125;&#125;,cmd0} : &#123;&#123;8{acc[7]&#125;&#125;,acc};	
	
assign code_addr = code_base + code_rel;	

rom_addr = (movc_a_dp(cmda) | movc_a_pc(cmda)) ? code_addr : pc;
```

`code_addr`由两部分组成，一是基地址，要么是DP，要么是PC；二是变地址，由ACC寄存器或cmd0直接寻址的第二个字节来表示。

在读地址的赋值部分和前面两类差不多，需要注意的是这里使用到了DPTR和SP作为读地址，在内部需要标注内部寄存器的占用信号：

```verilog
assign use_psw_rs = add_a_rn(cmda)|add_a_ri(cmda)|addc_a_rn(cmda)|addc_a_ri(cmda)|subb_a_rn(cmda)|subb_a_ri(cmda)|inc_rn(cmda)|inc_ri(cmda)|dec_rn(cmda)|dec_ri(cmda)|anl_a_rn(cmda)|anl_a_ri(cmda)|orl_a_rn(cmda)|orl_a_ri(cmda)|xrl_a_rn(cmda)|xrl_a_ri(cmda)|mov_a_rn(cmda)|mov_a_ri(cmda)|mov_di_rn(cmda)|mov_di_ri(cmda)|mov_ri_a(cmda)|mov_ri_di(cmda)|mov_ri_da(cmda)|movx_a_ri(cmda)|movx_ri_a(cmda)|xch_a_rn(cmda)|xch_a_ri(cmda);

assign use_dp = movc_a_dp(cmda)|movx_a_dp(cmda);

assign use_acc = movc_a_dp(cmda)|movc_a_pc(cmda);

assign use_sp = pop(cmda)|ret(cmda)|reti(cmda);
```

对SP寄存器的修改：

```verilog
always @ ( posedge clk or posedge rst )
if ( rst )
    sp <= 8'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'h81) )
	    sp <= wr_byte;
	else if ( push(cmdb) )
	    sp <= sp_add1;
    else if ( pop(cmdb) )
        sp <= sp_sub1;	
	else;
else;

assign sp_sub1 = sp - 1'b1;

assign sp_add1 = sp + 1'b1;
```

##### 5.4.4 布尔变量操作指令的添加

这类指令主要是涉及到位操作，读地址只有一种，就是读可位操作区。可位操作区分为两类，一类是以8'h20为开始地址的DATA区的专门位操作区，另一类是SFR区的某些特殊寄存器。它们不会只读出需要的一个位，而还是读出了整个字节。在写入时，也只需要操作一个位，因此使用`wr_bit_byte`，只需修改需要修改的位，把它和其他无需修改的位一起写回即可。

```verilog
always @* begin
wr_bit_byte = data0;
if ( clr_bit(cmdc)|jbc(cmdc) )
    wr_bit_byte[cmd1[2:0]] = 1'b0;
else if ( setb_bit(cmdc) )
    wr_bit_byte[cmd1[2:0]] = 1'b1;
else if ( cpl_bit(cmdc)	)
    wr_bit_byte[cmd1[2:0]] = ~wr_bit_byte[cmd1[2:0]];
else if ( mov_bit_c(cmdc) )	
    wr_bit_byte[cmd1[2:0]] = psw_c;
else;
end
```

##### 5.5.5 程序跳转指令的添加

这类指令需要更改它们取出的地址，需要对一些特殊的指令进行特殊化处理。第一个特殊指令是ACALL指令，它有两次写操作，并且必须写入PC寄存器在取出第二个字节后的地址。所以我们把它扩展成三字节指令，取出第二个指令后，PC不再增加，那么在第二个字节和虚拟的第三个字节出现时把PC压入栈内。

因此增加了PC_en信号，用于控制PC在`acall(cmdb)`时不能增加。

```verilog
always @*
if ( ~work_en )
    pc_en = 1'b0;
else if ( wait_en|length2r1 )
    pc_en = 1'b0;
else if ( acall(cmdb)|ret(cmda)|ret(cmdb)|reti(cmda)|reti(cmdb) )
    pc_en = 1'b0;
else
    pc_en = 1'b1;
```

其中的ret和reti都是单字节指令，需要扩展成三字节，使其不再增加。

rom_en和rom_addr都需要进行相关的修改，还有一些压栈，出栈涉及到的数据操作，还有DJNZ类指令需要进行递减操作，通用加减器需要打开。片内的PSW的CY位也需要对应修改，压栈出栈的SP寄存器需要递增或递减。具体的修改不再赘述，在此处贴出完整的全部代码，供进行详细研究：

```verilog
/////////////////////////////////////////////////////////////////////////////////////
//
//Copyright 2018  Li Xinbing
//
//Licensed under the Apache License, Version 2.0 (the "License");
//you may not use this file except in compliance with the License.
//You may obtain a copy of the License at
//
//    http://www.apache.org/licenses/LICENSE-2.0
//
//Unless required by applicable law or agreed to in writing, software
//distributed under the License is distributed on an "AS IS" BASIS,
//WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
//See the License for the specific language governing permissions and
//limitations under the License.
//
/////////////////////////////////////////////////////////////////////////////////////

//`define TYPE8052
module r8051 (

input  wire        clk,
input  wire        rst,
input  wire        cpu_en,
input  wire        cpu_restart,
	
output reg         rom_en,
output reg  [15:0] rom_addr,
input  wire [7:0]  rom_byte,
input  wire        rom_vld,
	
output wire        ram_rd_en_data,
output wire        ram_rd_en_sfr,
`ifdef TYPE8052
output wire        ram_rd_en_idata,
`endif
output wire        ram_rd_en_xdata,
output wire [15:0] ram_rd_addr,
input  wire [7:0]  ram_rd_byte,
input  wire        ram_rd_vld,
	
output wire        ram_wr_en_data,
output wire        ram_wr_en_sfr,
`ifdef TYPE8052
output wire        ram_wr_en_idata,
`endif
output wire        ram_wr_en_xdata,
output wire [15:0] ram_wr_addr,
output wire [7:0]  ram_wr_byte

);

`include "instruction.v"

/*********************************************************/
//register definition

reg                rd_wait;
reg      [7:0]     cmd1;
reg      [7:0]     cmd2;
reg      [2:0]     cmd_flag;
reg      [15:0]    pc;
reg      [7:0]     acc;
reg                psw_ov;
reg                psw_ac;
reg                psw_c;
reg      [3:0]     psw_other;
reg      [15:0]    dp;
reg      [7:0]     sp;
reg      [7:0]     b;
reg                same_flag;
reg      [7:0]     same_byte;
reg      [7:0]     data1;


/*********************************************************/
//wire definition

wire               work_en;
wire     [7:0]     cmd0;
wire     [7:0]     cmda;
wire     [7:0]     cmdb;
wire     [7:0]     cmdc;
reg                pc_en;
wire     [15:0]    pc_add1;
wire     [15:0]    code_base;
wire     [15:0]    code_rel;
wire     [15:0]    code_addr;
wire               length1;
wire               length2r1;
wire               length2;
wire               length3;
wire               rd_en_data_sfr;
wire               rd_en_data_idata;
wire               rd_en_xdata;
wire               rd_en_data;
wire               rd_en_sfr;
`ifdef TYPE8052
wire               rd_en_idata;
`endif
wire               same_flag_data;
wire               same_flag_sfr;
`ifdef TYPE8052
wire               same_flag_idata;
`endif
wire               same_flag_xdata;
wire               read_internal;
reg      [15:0]    rd_addr;
wire               use_psw_rs;
wire               use_dp;
wire               use_acc;
wire               use_sp;
wire               wait_en;
wire               wr_en_data_sfr;
wire               wr_en_data_idata;
wire               wr_en_xdata;
wire               wr_en_data;
wire               wr_en_sfr;
`ifdef TYPE8052
wire               wr_en_idata;
`endif 
wire               write_internal;
reg      [15:0]    wr_addr;
reg      [7:0]     wr_bit_byte;
reg      [7:0]     wr_byte;
reg      [7:0]     add_a;
reg      [7:0]     add_b;
reg                add_c;
reg                sub_flag;
wire               bit_ac;
wire     [3:0]     low;
wire     [3:0]     high;
wire               bit_c;
wire               bit_high; 
wire               bit_ov; 
wire     [7:0]     add_byte;
wire     [15:0]    mult;
wire     [7:0]     and_out;
wire     [7:0]     or_out; 
wire     [7:0]     xor_out;
wire               wr_acc;
wire               psw_p;
wire     [1:0]     psw_rs;
wire               wr_psw_rs; 
wire     [7:0]     psw;
wire               wr_dp;
wire     [7:0]     sp_sub1;
wire     [7:0]     sp_add1;
wire               wr_sp;
wire     [7:0]     div_ans;
wire     [7:0]     div_rem;
reg      [7:0]     data0;


/*********************************************************/


/*********************************************************/
//work_en

always @ ( posedge clk or posedge rst )
if ( rst )
    rd_wait <= 1'b0;
else if ( cpu_restart )
    rd_wait <= 1'b0;	
`ifdef TYPE8052
else if ( ram_rd_en_data|ram_rd_en_sfr|ram_rd_en_idata|ram_rd_en_xdata )
`else
else if ( ram_rd_en_data|ram_rd_en_sfr|ram_rd_en_xdata ) 
`endif	
    rd_wait <= 1'b1;
else if ( ram_rd_vld )
    rd_wait <= 1'b0;	
else;

reg rom_wait;
always @ ( posedge clk or posedge rst )
if ( rst )
    rom_wait <= 1'b0;
else if ( cpu_restart )
    rom_wait <= 1'b0;	
else if ( rom_en )
	rom_wait <= 1'b1;
else if ( rom_vld )
    rom_wait <= 1'b0;
else;


assign work_en = ( ( rd_wait & ~ram_rd_vld )|( rom_wait & ~rom_vld ) ) ? 1'b0 : cpu_en;

/*********************************************************/

/*********************************************************/
//cmd0/cmd1/cmd2  cmda/cmdb/cmdc

assign cmd0 = rom_byte;

always @ ( posedge clk or posedge rst )
if ( rst )
    cmd1 <= 8'b0;
else if ( work_en )
    cmd1 <= cmd0;
else;

always @ ( posedge clk or posedge rst )
if ( rst )
    cmd2 <= 8'b0;
else if ( work_en )
    cmd2 <= cmdb;
else;

always @ ( posedge clk or posedge rst )
if ( rst )
    cmd_flag <= 3'b1;
else if ( cpu_restart )
    cmd_flag <= 3'b1;
else if ( work_en )
    if ( wait_en )
	    cmd_flag <= {1'b0,cmd_flag[1:0]};
    else if ( length2|length2r1 )
	    cmd_flag <= 3'b101;
	else if ( length3 )
	    cmd_flag <= 3'b100;
	else
        cmd_flag <= { cmd_flag[1:0],1'b1};
else;

assign cmda = cmd_flag[1] ? cmd0 : 8'b0;

assign cmdb = cmd_flag[2] ? cmd1 : 8'b0;

assign cmdc = cmd2;


/*********************************************************/


/*********************************************************/
//rom_en rom_addr

always @*
if ( ~work_en )
    pc_en = 1'b0;
else if ( wait_en|length2r1 )
    pc_en = 1'b0;
else if ( acall(cmdb)|ret(cmda)|ret(cmdb)|reti(cmda)|reti(cmdb) )
    pc_en = 1'b0;
else
    pc_en = 1'b1;

always @ ( posedge clk or posedge rst )
if ( rst )
    pc <= 16'd0;
else if ( cpu_restart )
    pc <= 16'd0;
else if ( work_en )
    if ( pc_en )
        pc <= pc_add1;
	else;
else;

assign pc_add1 = rom_addr + 1'b1;

always @*
if ( ~work_en )
    rom_en = 1'b0;
else if ( wait_en )
    rom_en = 1'b0;
else if ( length2r1 )
    rom_en = movc_a_dp(cmda)|movc_a_pc(cmda);
else if ( acall(cmdb)|ret(cmda)|ret(cmdb)|reti(cmda)|reti(cmdb) )
    rom_en = 1'b0;	
else
    rom_en = 1'b1;

assign code_base = ( movc_a_dp(cmda)|jmp(cmda) ) ? dp : pc;

assign code_rel = ( jc(cmdb)||jnc(cmdb)|jb(cmdc)|jnb(cmdc)|jbc(cmdc)|sjmp(cmdb)|jz(cmdb)|jnz(cmdb)|cjne_a_di_rel(cmdc)|cjne_a_da_rel(cmdc)|cjne_rn_da_rel(cmdc)|cjne_ri_da_rel(cmdc)|djnz_rn_rel(cmdb)|djnz_di_rel(cmdc) ) ? &#123;&#123;8{cmd0[7]&#125;&#125;,cmd0} : &#123;&#123;8{acc[7]&#125;&#125;,acc};	
	
assign code_addr = code_base + code_rel;	

always @*
if ( acall(cmdc) )
    rom_addr = {pc[15:11],cmd2[7:5],cmd1};
else if ( lcall(cmdc)|ljmp(cmdc) )
    rom_addr = {cmd1,cmd0};	
else if ( ret(cmdc)|reti(cmdc) )
    rom_addr = {data1,data0};
else if ( ajmp(cmdb) )
    rom_addr = {pc[15:11],cmd1[7:5],cmd0};	
else if ( movc_a_dp(cmda)|movc_a_pc(cmda)|(jc(cmdb) & psw_c)|(jnc(cmdb) & ~psw_c)|((jb(cmdc)|jbc(cmdc)) & data0[cmd1[2:0]])|(jnb(cmdc) & ~data0[cmd1[2:0]])|sjmp(cmdb)|jmp(cmda)|(jz(cmdb) & (acc==8'b0))|(jnz(cmdb) & (acc!=8'b0))|(cjne_a_di_rel(cmdc) & (acc!=data0))|(cjne_a_da_rel(cmdc) & (acc!=cmd1))|(cjne_rn_da_rel(cmdc) & (data1!=cmd1))|(cjne_ri_da_rel(cmdc) & (data0!=cmd1))|((djnz_rn_rel(cmdb)||djnz_di_rel(cmdc)) & (data0!=8'h1)) ) 
    rom_addr = code_addr; 
else
    rom_addr = pc;


/*********************************************************/

/*********************************************************/
//command length

assign  length1 = add_a_rn(cmda)|addc_a_rn(cmda)|subb_a_rn(cmda)|inc_a(cmda)|inc_rn(cmda)|inc_dp(cmda)|dec_a(cmda)|dec_rn(cmda)|mul(cmda)|div(cmda)|da(cmda)|anl_a_rn(cmda)|orl_a_rn(cmda)|xrl_a_rn(cmda)|clr_a(cmda)|cpl_a(cmda)|rl(cmda)|rlc(cmda)|rr(cmda)|rrc(cmda)|swap(cmda)|mov_a_rn(cmda)|mov_rn_a(cmda)|mov_ri_a(cmda)|movx_a_dp(cmda)|movx_ri_a(cmda)|movx_dp_a(cmda)|xch_a_rn(cmda)|clr_c(cmda)|setb_c(cmda)|cpl_c(cmda)|jmp(cmda);

assign  length2r1 = add_a_ri(cmda)|addc_a_ri(cmda)|subb_a_ri(cmda)|inc_ri(cmda)|dec_ri(cmda)|anl_a_ri(cmda)|orl_a_ri(cmda)|xrl_a_ri(cmda)|mov_a_ri(cmda)|movc_a_dp(cmda)|movc_a_pc(cmda)|movx_a_ri(cmda)|xch_a_ri(cmda)|xchd(cmda);

assign  length2 = add_a_di(cmda)|add_a_da(cmda)|addc_a_di(cmda)|addc_a_da(cmda)|subb_a_di(cmda)|subb_a_da(cmda)|inc_di(cmda)|dec_di(cmda)|anl_a_di(cmda)|anl_a_da(cmda)|anl_di_a(cmda)|orl_a_di(cmda)|orl_a_da(cmda)|orl_di_a(cmda)|xrl_a_di(cmda)|xrl_a_da(cmda)|xrl_di_a(cmda)|mov_a_di(cmda)|mov_a_da(cmda)|mov_rn_di(cmda)|mov_rn_da(cmda)|mov_di_a(cmda)|mov_di_rn(cmda)|mov_di_ri(cmda)|mov_ri_di(cmda)|mov_ri_da(cmda)|push(cmda)|pop(cmda)|xch_a_di(cmda)|clr_bit(cmda)|setb_bit(cmda)|cpl_bit(cmda)|anl_c_bit(cmda)|anl_c_nbit(cmda)|orl_c_bit(cmda)|orl_c_nbit(cmda)|mov_c_bit(cmda)|mov_bit_c(cmda)|jc(cmda)|jnc(cmda)|ajmp(cmda)|sjmp(cmda)|jz(cmda)|jnz(cmda)|djnz_rn_rel(cmda);

assign  length3 = anl_di_da(cmda)|orl_di_da(cmda)|xrl_di_da(cmda)|mov_di_di(cmda)|mov_di_da(cmda)|mov_dp_da(cmda)|jb(cmda)|jnb(cmda)|jbc(cmda)|acall(cmda)|lcall(cmda)|ret(cmda)|reti(cmda)|ljmp(cmda)|cjne_a_di_rel(cmda)|cjne_a_da_rel(cmda)|cjne_rn_da_rel(cmda)|cjne_ri_da_rel(cmda)|djnz_di_rel(cmda);


/*********************************************************/


/*********************************************************/
//ram_rd_en ram_rd_addr 

assign rd_en_data_sfr = add_a_rn(cmda)|add_a_di(cmdb)|add_a_ri(cmda)|addc_a_rn(cmda)|addc_a_di(cmdb)|addc_a_ri(cmda)|subb_a_rn(cmda)|subb_a_di(cmdb)|subb_a_ri(cmda)|inc_rn(cmda)|inc_di(cmdb)|inc_ri(cmda)|dec_rn(cmda)|dec_di(cmdb)|dec_ri(cmda)|anl_a_rn(cmda)|anl_a_di(cmdb)|anl_a_ri(cmda)|anl_di_a(cmdb)|anl_di_da(cmdb)|orl_a_rn(cmda)|orl_a_di(cmdb)|orl_a_ri(cmda)|orl_di_a(cmdb)|orl_di_da(cmdb)|xrl_a_rn(cmda)|xrl_a_di(cmdb)|xrl_a_ri(cmda)|xrl_di_a(cmdb)|xrl_di_da(cmdb)|mov_a_rn(cmda)|mov_a_di(cmdb)|mov_a_ri(cmda)|mov_rn_di(cmdb)|mov_di_rn(cmda)|mov_di_di(cmdb)|mov_di_ri(cmda)|mov_ri_a(cmda)|mov_ri_di(cmda)|mov_ri_di(cmdb)|mov_ri_da(cmda)|movx_a_ri(cmda)|movx_ri_a(cmda)|push(cmdb)|xch_a_rn(cmda)|xch_a_di(cmdb)|xch_a_ri(cmda)|xchd(cmda)|clr_bit(cmdb)|setb_bit(cmdb)|cpl_bit(cmdb)|anl_c_bit(cmdb)|anl_c_nbit(cmdb)|orl_c_bit(cmdb)|orl_c_nbit(cmdb)|mov_c_bit(cmdb)|mov_bit_c(cmdb)|jb(cmdb)|jnb(cmdb)|jbc(cmdb)|cjne_a_di_rel(cmdb)|cjne_rn_da_rel(cmda)|cjne_ri_da_rel(cmda)|djnz_rn_rel(cmda)|djnz_di_rel(cmdb);

assign rd_en_data_idata = add_a_ri(cmdb)|addc_a_ri(cmdb)|subb_a_ri(cmdb)|inc_ri(cmdb)|dec_ri(cmdb)|anl_a_ri(cmdb)|orl_a_ri(cmdb)|xrl_a_ri(cmdb)|mov_a_ri(cmdb)|mov_di_ri(cmdb)|xch_a_ri(cmdb)|xchd(cmdb)|cjne_ri_da_rel(cmdb)|ret(cmda)|ret(cmdb)|reti(cmda)|reti(cmdb)|pop(cmda);

assign rd_en_xdata = movx_a_ri(cmdb)|movx_a_dp(cmda);

assign rd_en_data = (rd_en_data_sfr|rd_en_data_idata) & ~rd_addr[7];
`ifdef TYPE8052
assign rd_en_sfr = rd_en_data_sfr & rd_addr[7];
assign rd_en_idata = rd_en_data_idata & rd_addr[7];
`else
assign rd_en_sfr = (rd_en_data_sfr|rd_en_data_idata) & rd_addr[7];
`endif
assign same_flag_data = rd_en_data & wr_en_data & (rd_addr[7:0]==wr_addr[7:0]);
assign same_flag_sfr = rd_en_sfr & wr_en_sfr & (rd_addr[7:0]==wr_addr[7:0]);
`ifdef TYPE8052
assign same_flag_idata = rd_en_idata & wr_en_idata & (rd_addr[7:0]==wr_addr[7:0]);
`endif
assign same_flag_xdata = rd_en_xdata & wr_en_xdata & (rd_addr[15:0]==wr_addr[15:0]);
assign read_internal = rd_en_sfr & ( (rd_addr[7:0]==8'he0)|(rd_addr[7:0]==8'hd0)|(rd_addr[7:0]==8'h83)|(rd_addr[7:0]==8'h82)|(rd_addr[7:0]==8'h81)|(rd_addr[7:0]==8'hf0) );
assign ram_rd_en_data = work_en & rd_en_data & ~same_flag_data & ~wait_en;
assign ram_rd_en_sfr = work_en & rd_en_sfr & ~same_flag_sfr & ~read_internal & ~wait_en;
`ifdef TYPE8052
assign ram_rd_en_idata = work_en & rd_en_idata & ~same_flag_idata & ~wait_en;
`endif 
assign ram_rd_en_xdata = work_en & rd_en_xdata & ~same_flag_xdata & ~wait_en;


always @*
if ( add_a_rn(cmda)|addc_a_rn(cmda)|subb_a_rn(cmda)|inc_rn(cmda)|dec_rn(cmda)|anl_a_rn(cmda)|orl_a_rn(cmda)|xrl_a_rn(cmda)|mov_a_rn(cmda)|mov_di_rn(cmda)|xch_a_rn(cmda)|cjne_rn_da_rel(cmda)|djnz_rn_rel(cmda) )
    rd_addr = { psw_rs,cmd0[2:0] };
else if ( add_a_di(cmdb)|addc_a_di(cmdb)|subb_a_di(cmdb)|inc_di(cmdb)|dec_di(cmdb)|anl_a_di(cmdb)|anl_di_a(cmdb)|anl_di_da(cmdb)|orl_a_di(cmdb)|orl_di_a(cmdb)|orl_di_da(cmdb)|xrl_a_di(cmdb)|xrl_di_a(cmdb)|xrl_di_da(cmdb)|mov_a_di(cmdb)|mov_rn_di(cmdb)|mov_di_di(cmdb)|mov_ri_di(cmdb)|push(cmdb)|xch_a_di(cmdb)|cjne_a_di_rel(cmdb)|djnz_di_rel(cmdb) )
    rd_addr = cmd0;
else if ( add_a_ri(cmda)|addc_a_ri(cmda)|subb_a_ri(cmda)|inc_ri(cmda)|dec_ri(cmda)|anl_a_ri(cmda)|orl_a_ri(cmda)|xrl_a_ri(cmda)|mov_a_ri(cmda)|mov_di_ri(cmda)|mov_ri_a(cmda)|mov_ri_di(cmda)|mov_ri_da(cmda)|movx_a_ri(cmda)|movx_ri_a(cmda)|xch_a_ri(cmda)|xchd(cmda)|cjne_ri_da_rel(cmda) )
    rd_addr = { psw_rs,2'b0,cmd0[0] };
else if ( add_a_ri(cmdb)|addc_a_ri(cmdb)|subb_a_ri(cmdb)|inc_ri(cmdb)|dec_ri(cmdb)|anl_a_ri(cmdb)|orl_a_ri(cmdb)|xrl_a_ri(cmdb)|mov_a_ri(cmdb)|mov_di_ri(cmdb)|movx_a_ri(cmdb)|xch_a_ri(cmdb)|xchd(cmdb)|cjne_ri_da_rel(cmdb) )
    rd_addr = data0;
else if ( movx_a_dp(cmda) )
    rd_addr = dp;
else if ( pop(cmda)|ret(cmda)|reti(cmda) )
    rd_addr = sp;
else if ( clr_bit(cmdb)|setb_bit(cmdb)|cpl_bit(cmdb)|anl_c_bit(cmdb)|anl_c_nbit(cmdb)|orl_c_bit(cmdb)|orl_c_nbit(cmdb)|mov_c_bit(cmdb)|mov_bit_c(cmdb)|jb(cmdb)|jnb(cmdb)|jbc(cmdb) )
    rd_addr = cmd0[7] ? {cmd0[7:3],3'b0} : {3'b001,cmd0[7:3]};    
else if ( ret(cmdb)|reti(cmdb) )
    rd_addr = sp_sub1;	
else
    rd_addr = 16'd0;
	
assign ram_rd_addr = rd_addr;	

assign use_psw_rs = add_a_rn(cmda)|add_a_ri(cmda)|addc_a_rn(cmda)|addc_a_ri(cmda)|subb_a_rn(cmda)|subb_a_ri(cmda)|inc_rn(cmda)|inc_ri(cmda)|dec_rn(cmda)|dec_ri(cmda)|anl_a_rn(cmda)|anl_a_ri(cmda)|orl_a_rn(cmda)|orl_a_ri(cmda)|xrl_a_rn(cmda)|xrl_a_ri(cmda)|mov_a_rn(cmda)|mov_a_ri(cmda)|mov_di_rn(cmda)|mov_di_ri(cmda)|mov_ri_a(cmda)|mov_ri_di(cmda)|mov_ri_da(cmda)|movx_a_ri(cmda)|movx_ri_a(cmda)|xch_a_rn(cmda)|xch_a_ri(cmda)|xchd(cmda)|cjne_rn_da_rel(cmda)|cjne_ri_da_rel(cmda)|djnz_rn_rel(cmda);

assign use_dp = movc_a_dp(cmda)|movx_a_dp(cmda)|jmp(cmda);

assign use_acc = movc_a_dp(cmda)|movc_a_pc(cmda)|jmp(cmda);

assign use_sp = pop(cmda)|ret(cmda)|reti(cmda);

assign wait_en = (use_psw_rs&wr_psw_rs)|(use_dp&wr_dp)|(use_acc&wr_acc)|(use_sp&wr_sp);

/*********************************************************/

/*********************************************************/
//ram_wr_en ram_wr_addr  

assign wr_en_data_sfr = inc_rn(cmdb)|inc_di(cmdc)|dec_rn(cmdb)|dec_di(cmdc)|anl_di_a(cmdc)|anl_di_da(cmdc)|orl_di_a(cmdc)|orl_di_da(cmdc)|xrl_di_a(cmdc)|xrl_di_da(cmdc)|mov_rn_a(cmdb)|mov_rn_di(cmdc)|mov_rn_da(cmdb)|mov_di_a(cmdb)|mov_di_rn(cmdb)|mov_di_di(cmdc)|mov_di_ri(cmdc)|mov_di_da(cmdc)|pop(cmdb)|xch_a_rn(cmdb)|xch_a_di(cmdc)|clr_bit(cmdc)|setb_bit(cmdc)|cpl_bit(cmdc)|mov_bit_c(cmdc)|(jbc(cmdc)&data0[cmd1[2:0]])|djnz_rn_rel(cmdb)|djnz_di_rel(cmdc);

assign wr_en_data_idata = inc_ri(cmdc)|dec_ri(cmdc)|mov_ri_a(cmdb)|mov_ri_di(cmdc)|mov_ri_da(cmdb)|xch_a_ri(cmdc)|xchd(cmdc)|acall(cmdb)|acall(cmdc)|lcall(cmdb)|lcall(cmdc)|push(cmdc);

assign wr_en_xdata = movx_ri_a(cmdb)|movx_dp_a(cmdb);

assign wr_en_data = (wr_en_data_sfr|wr_en_data_idata) & ~wr_addr[7];
`ifdef TYPE8052
assign wr_en_sfr = wr_en_data_sfr & wr_addr[7];
assign wr_en_idata = wr_en_data_idata & wr_addr[7];
`else
assign wr_en_sfr = (wr_en_data_sfr|wr_en_data_idata) & wr_addr[7];
`endif
assign write_internal = wr_en_sfr & ( (wr_addr[7:0]==8'he0)|(wr_addr[7:0]==8'hd0)|(wr_addr[7:0]==8'h83)|(wr_addr[7:0]==8'h82)|(wr_addr[7:0]==8'h81)|(wr_addr[7:0]==8'hf0) );
assign ram_wr_en_data = work_en & wr_en_data;
assign ram_wr_en_sfr = work_en & wr_en_sfr & ~write_internal;
`ifdef TYPE8052
assign ram_wr_en_idata = work_en & wr_en_idata;
`endif
assign ram_wr_en_xdata = work_en & wr_en_xdata;

always @*
if ( inc_rn(cmdb)|dec_rn(cmdb)|mov_rn_a(cmdb)|mov_rn_da(cmdb)|xch_a_rn(cmdb)|djnz_rn_rel(cmdb) )
    wr_addr = { psw_rs,cmd1[2:0] };
else if ( inc_di(cmdc)|dec_di(cmdc)|anl_di_a(cmdc)|anl_di_da(cmdc)|orl_di_a(cmdc)|orl_di_da(cmdc)|xrl_di_a(cmdc)|xrl_di_da(cmdc)|mov_di_ri(cmdc)|mov_di_da(cmdc)|xch_a_di(cmdc)|djnz_di_rel(cmdc) )
    wr_addr = cmd1;
else if ( inc_ri(cmdc)|dec_ri(cmdc)|mov_ri_di(cmdc)|xch_a_ri(cmdc)|xchd(cmdc) )
    wr_addr = data1;
else if ( mov_rn_di(cmdc) )
    wr_addr = { psw_rs,cmd2[2:0] };
else if ( mov_di_a(cmdb)|mov_di_rn(cmdb)|mov_di_di(cmdc)|pop(cmdb) )
    wr_addr = cmd0;
else if ( mov_ri_a(cmdb)|mov_ri_da(cmdb)|movx_ri_a(cmdb) )
    wr_addr = data0;
else if ( movx_dp_a(cmdb) )
    wr_addr = dp;
else if ( push(cmdc) )
    wr_addr = sp;
else if ( clr_bit(cmdc)|setb_bit(cmdc)|cpl_bit(cmdc)|mov_bit_c(cmdc)|jbc(cmdc) )
    wr_addr = cmd1[7] ? {cmd1[7:3],3'b0} : {3'b001,cmd1[7:3]};
else if ( acall(cmdb)|acall(cmdc)|lcall(cmdb)|lcall(cmdc) )
    wr_addr = sp_add1;	
else
    wr_addr = 16'd0;

assign ram_wr_addr = wr_addr;	

/*********************************************************/


/*********************************************************/
//ram_wr_byte

always @* begin
wr_bit_byte = data0;
if ( clr_bit(cmdc)|jbc(cmdc) )
    wr_bit_byte[cmd1[2:0]] = 1'b0;
else if ( setb_bit(cmdc) )
    wr_bit_byte[cmd1[2:0]] = 1'b1;
else if ( cpl_bit(cmdc)	)
    wr_bit_byte[cmd1[2:0]] = ~wr_bit_byte[cmd1[2:0]];
else if ( mov_bit_c(cmdc) )	
    wr_bit_byte[cmd1[2:0]] = psw_c;
else;
end

always @*
if ( inc_rn(cmdb)|inc_di(cmdc)|inc_ri(cmdc)|dec_rn(cmdb)|dec_di(cmdc)|dec_ri(cmdc)|djnz_rn_rel(cmdb)|djnz_di_rel(cmdc) )
    wr_byte = add_byte;
else if ( anl_di_a(cmdc)|anl_di_da(cmdc) )
    wr_byte = and_out;
else if ( orl_di_a(cmdc)|orl_di_da(cmdc) )
    wr_byte = or_out;
else if ( xrl_di_a(cmdc)|xrl_di_da(cmdc) )
    wr_byte = xor_out;
else if ( mov_rn_a(cmdb)|mov_di_a(cmdb)|mov_ri_a(cmdb)|movx_ri_a(cmdb)|movx_dp_a(cmdb)|xch_a_rn(cmdb)|xch_a_di(cmdc)|xch_a_ri(cmdc) )
    wr_byte = acc;
else if ( mov_rn_di(cmdc)|mov_di_rn(cmdb)|mov_di_di(cmdc)|mov_di_ri(cmdc)|mov_ri_di(cmdc)|push(cmdc)|pop(cmdb) )
    wr_byte = data0;
else if ( mov_rn_da(cmdb)|mov_di_da(cmdc)|mov_ri_da(cmdb) )
    wr_byte = cmd0;
else if ( xchd(cmdc) )
    wr_byte = {data0[7:4],acc[3:0]};
else if ( clr_bit(cmdc)|setb_bit(cmdc)|cpl_bit(cmdc)|mov_bit_c(cmdc)|jbc(cmdc) )
    wr_byte = wr_bit_byte;
else if ( acall(cmdb) )
    wr_byte = pc[7:0];
else if ( acall(cmdc)|lcall(cmdc) )
    wr_byte = pc[15:8];	
else if ( lcall(cmdb) )
    wr_byte = pc_add1[7:0];
else
    wr_byte = 8'd0;

assign ram_wr_byte = wr_byte;

/*********************************************************/


/*********************************************************/
//acc register

always @*
if ( add_a_rn(cmdb)|add_a_di(cmdc)|add_a_ri(cmdc)|add_a_da(cmdb)|addc_a_rn(cmdb)|addc_a_di(cmdc)|addc_a_ri(cmdc)|addc_a_da(cmdb)|subb_a_rn(cmdb)|subb_a_di(cmdc)|subb_a_ri(cmdc)|subb_a_da(cmdb)|inc_a(cmdb)|dec_a(cmdb) )
    add_a = acc;
else if ( inc_rn(cmdb)|inc_di(cmdc)|inc_ri(cmdc)|dec_rn(cmdb)|dec_di(cmdc)|dec_ri(cmdc)|djnz_rn_rel(cmdb)|djnz_di_rel(cmdc) )
    add_a = data0;
else
    add_a = 8'b0;


always @*
if ( add_a_rn(cmdb)|add_a_di(cmdc)|add_a_ri(cmdc)|addc_a_rn(cmdb)|addc_a_di(cmdc)|addc_a_ri(cmdc)|subb_a_rn(cmdb)|subb_a_di(cmdc)|subb_a_ri(cmdc) )
    add_b = data0;
else if ( add_a_da(cmdb)|addc_a_da(cmdb)|subb_a_da(cmdb) )
    add_b = cmd0;
else if ( inc_a(cmdb)|dec_a(cmdb)|inc_rn(cmdb)|inc_di(cmdc)|inc_ri(cmdc)|dec_rn(cmdb)|dec_di(cmdc)|dec_ri(cmdc)|djnz_rn_rel(cmdb)|djnz_di_rel(cmdc) )
    add_b = 8'b0;
else
    add_b = 8'b0;


always @*
if ( add_a_rn(cmdb)|add_a_di(cmdc)|add_a_ri(cmdc)|add_a_da(cmdb) )
    add_c = 1'b0;
else if ( addc_a_rn(cmdb)|addc_a_di(cmdc)|addc_a_ri(cmdc)|addc_a_da(cmdb)|subb_a_rn(cmdb)|subb_a_di(cmdc)|subb_a_ri(cmdc)|subb_a_da(cmdb) )
    add_c = psw_c;
else if ( inc_a(cmdb)|inc_rn(cmdb)|inc_di(cmdc)|inc_ri(cmdc)|dec_a(cmdb)|dec_rn(cmdb)|dec_di(cmdc)|dec_ri(cmdc)|djnz_rn_rel(cmdb)|djnz_di_rel(cmdc) )
    add_c = 1'b1;	
else
    add_c = 1'b0;	


always @*
if ( add_a_rn(cmdb)|add_a_di(cmdc)|add_a_ri(cmdc)|add_a_da(cmdb)|addc_a_rn(cmdb)|addc_a_di(cmdc)|addc_a_ri(cmdc)|addc_a_da(cmdb)|inc_a(cmdb)|inc_rn(cmdb)|inc_di(cmdc)|inc_ri(cmdc) )
    sub_flag = 1'b0;
else if ( subb_a_rn(cmdb)|subb_a_di(cmdc)|subb_a_ri(cmdc)|subb_a_da(cmdb)|dec_a(cmdb)|dec_rn(cmdb)|dec_di(cmdc)|dec_ri(cmdc)|djnz_rn_rel(cmdb)|djnz_di_rel(cmdc) )
    sub_flag = 1'b1;
else
    sub_flag = 1'b0;


assign {bit_ac,low} = sub_flag ? (add_a[3:0]-add_b[3:0]-add_c) : (add_a[3:0]+add_b[3:0]+add_c);
assign high = sub_flag ? (add_a[6:4]-add_b[6:4]-bit_ac) : (add_a[6:4]+add_b[6:4]+bit_ac);
assign {bit_c,bit_high} = sub_flag ? (add_a[7]-add_b[7]-high[3]) : (add_a[7]+add_b[7]+high[3]);
assign bit_ov = bit_c ^ high[3];
assign add_byte = {bit_high,high[2:0],low};

assign mult = acc * b;
assign {div_rem,div_ans} = divide(acc,b);
assign and_out = ( anl_di_da(cmdc) ? cmd0 : acc ) & ( anl_a_da(cmdb) ? cmd0 : data0 );
assign or_out  = ( orl_di_da(cmdc) ? cmd0 : acc ) | ( orl_a_da(cmdb) ? cmd0 : data0 );
assign xor_out = ( xrl_di_da(cmdc) ? cmd0 : acc ) ^ ( xrl_a_da(cmdb) ? cmd0 : data0 );

wire [3:0] da_low  = ( psw_ac|(acc[3:0]>4'h9) ) ? (acc[3:0]+4'd6) : acc[3:0];
wire [3:0] da_high = ( ( psw_c|(acc[7:4]>4'h9)|((acc[7:4]==4'h9)&(psw_ac|(acc[3:0]>4'h9))) ) ? ( acc[7:4]+4'h6 ) : acc[7:4] ) + ( psw_ac|(acc[3:0]>4'h9) );
	
always @ ( posedge clk or posedge rst )
if ( rst )
    acc <= 8'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'he0 ) )
	    acc <= wr_byte;
	else if ( add_a_rn(cmdb)|add_a_di(cmdc)|add_a_ri(cmdc)|add_a_da(cmdb)|addc_a_rn(cmdb)|addc_a_di(cmdc)|addc_a_ri(cmdc)|addc_a_da(cmdb)|subb_a_rn(cmdb)|subb_a_di(cmdc)|subb_a_ri(cmdc)|subb_a_da(cmdb)|inc_a(cmdb)|dec_a(cmdb) )
	    acc <= add_byte;
	else if ( mul(cmdb) )
	    acc <= mult[7:0];
	else if ( div(cmdb) )
	    acc <= div_ans;
	else if ( da(cmdb) )
	    acc <= {da_high,da_low};	
    else if ( anl_a_rn(cmdb)|anl_a_di(cmdc)|anl_a_ri(cmdc)|anl_a_da(cmdb) )
        acc <= and_out;	
	else if ( orl_a_rn(cmdb)|orl_a_di(cmdc)|orl_a_ri(cmdc)|orl_a_da(cmdb) )
	    acc <= or_out;
	else if ( xrl_a_rn(cmdb)|xrl_a_di(cmdc)|xrl_a_ri(cmdc)|xrl_a_da(cmdb) )
	    acc <= xor_out;
	else if ( clr_a(cmdb) )
	    acc <= 8'b0;
	else if ( cpl_a(cmdb) )
	    acc <= ~acc;
	else if ( rl(cmdb) )
	    acc <= {acc[6:0],acc[7]};
	else if ( rlc(cmdb) )
	    acc <= {acc[6:0],psw_c};	
	else if ( rr(cmdb) )
	    acc <= {acc[0],acc[7:1]};	
	else if ( rrc(cmdb) )
	    acc <= {psw_c,acc[7:1]};	
	else if ( swap(cmdb) )
	    acc <= {acc[3:0],acc[7:4]};	
    else if ( mov_a_rn(cmdb)|mov_a_di(cmdc)|mov_a_ri(cmdc)|movx_a_ri(cmdc)|movx_a_dp(cmdb)|xch_a_rn(cmdb)|xch_a_di(cmdc)|xch_a_ri(cmdc) )
        acc <= data0;	
	else if ( mov_a_da(cmdb)|movc_a_dp(cmdb)|movc_a_pc(cmdb) )
	    acc <= cmd0;
	else if ( xchd(cmdc) )
	    acc[3:0] <= data0[3:0];
	else;
else;

assign wr_acc = (wr_en_sfr & (wr_addr[7:0]==8'he0))|add_a_rn(cmdb)|add_a_di(cmdc)|add_a_ri(cmdc)|add_a_da(cmdb)|addc_a_rn(cmdb)|addc_a_di(cmdc)|addc_a_ri(cmdc)|addc_a_da(cmdb)|subb_a_rn(cmdb)|subb_a_di(cmdc)|subb_a_ri(cmdc)|subb_a_da(cmdb)|inc_a(cmdb)|dec_a(cmdb)|mul(cmdb)|div(cmdb)|da(cmdb)|anl_a_rn(cmdb)|anl_a_di(cmdc)|anl_a_ri(cmdc)|anl_a_da(cmdb)|orl_a_rn(cmdb)|orl_a_di(cmdc)|orl_a_ri(cmdc)|orl_a_da(cmdb)|xrl_a_rn(cmdb)|xrl_a_di(cmdc)|xrl_a_ri(cmdc)|xrl_a_da(cmdb)|clr_a(cmdb)|cpl_a(cmdb)|rl(cmdb)|rlc(cmdb)|rr(cmdb)|rrc(cmdb)|swap(cmdb)|mov_a_rn(cmdb)|mov_a_di(cmdc)|mov_a_ri(cmdc)|mov_a_da(cmdb)|movx_a_ri(cmdc)|movx_a_dp(cmdb)|xch_a_rn(cmdb)|xch_a_di(cmdc)|xch_a_ri(cmdc)|xchd(cmdc);

/*********************************************************/


/*********************************************************/
//psw register

assign psw_p = ^acc;

always @ ( posedge clk or posedge rst )
if ( rst )
    psw_ov <= 1'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'hd0 ) )
	    psw_ov <= wr_byte[2];
	else if ( add_a_rn(cmdb)|add_a_di(cmdc)|add_a_ri(cmdc)|add_a_da(cmdb)|addc_a_rn(cmdb)|addc_a_di(cmdc)|addc_a_ri(cmdc)|addc_a_da(cmdb)|subb_a_rn(cmdb)|subb_a_di(cmdc)|subb_a_ri(cmdc)|subb_a_da(cmdb) )
	    psw_ov <= bit_ov;
	else if ( mul(cmdb) )
	    psw_ov <= (mult[15:8]!=8'b0);
	else if ( div(cmdb) )
	    psw_ov <= (b==8'b0);
	else;
else;

always @ ( posedge clk or posedge rst )
if ( rst )
    psw_ac <= 1'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'hd0 ) )
	    psw_ac <= wr_byte[6];
	else if ( add_a_rn(cmdb)|add_a_di(cmdc)|add_a_ri(cmdc)|add_a_da(cmdb)|addc_a_rn(cmdb)|addc_a_di(cmdc)|addc_a_ri(cmdc)|addc_a_da(cmdb)|subb_a_rn(cmdb)|subb_a_di(cmdc)|subb_a_ri(cmdc)|subb_a_da(cmdb) )
	    psw_ac <= bit_ac;
	else;
else;

always @ ( posedge clk or posedge rst )
if ( rst )
    psw_c <= 1'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'hd0 ) )
	    psw_c <= wr_byte[7];
	else if ( add_a_rn(cmdb)|add_a_di(cmdc)|add_a_ri(cmdc)|add_a_da(cmdb)|addc_a_rn(cmdb)|addc_a_di(cmdc)|addc_a_ri(cmdc)|addc_a_da(cmdb)|subb_a_rn(cmdb)|subb_a_di(cmdc)|subb_a_ri(cmdc)|subb_a_da(cmdb) )
	    psw_c <= bit_c;
	else if ( mul(cmdb)|div(cmdb) )
	    psw_c <= 1'b0;
	else if ( da(cmdb) )
	    psw_c <= ( psw_c|(acc[7:4]>4'h9)|((acc[7:4]==4'h9)&(psw_ac|(acc[3:0]>4'h9))) ) ? 1'b1 : psw_c;
	else if ( rlc(cmdb) )
	    psw_c <= acc[7];	
	else if ( rrc(cmdb) )
	    psw_c <= acc[0];	
    else if ( clr_c(cmdb) )
        psw_c <= 1'b0;	
	else if ( setb_c(cmdb) )
        psw_c <= 1'b1;	
	else if ( cpl_c(cmdb) )
        psw_c <= ~psw_c;	
	else if ( anl_c_bit(cmdc) )
        psw_c <= psw_c & data0[cmd1[2:0]];	
	else if ( anl_c_nbit(cmdc) )
        psw_c <= psw_c & ~data0[cmd1[2:0]];
	else if ( orl_c_bit(cmdc) )
        psw_c <= psw_c | data0[cmd1[2:0]];	
	else if ( orl_c_nbit(cmdc) )
        psw_c <= psw_c | ~data0[cmd1[2:0]];	
	else if ( mov_c_bit(cmdc) )
        psw_c <=data0[cmd1[2:0]];
	else if ( cjne_a_di_rel(cmdc) )
        psw_c <= acc<data0;
    else if ( cjne_a_da_rel(cmdc) )
        psw_c <= acc<cmd1;
	else if ( cjne_rn_da_rel(cmdc) )
        psw_c <= data1<cmd1;	
	else if ( cjne_ri_da_rel(cmdc) )
	    psw_c <= data0<cmd1;
	else;
else;

always @ ( posedge clk or posedge rst )
if ( rst )
    psw_other <= 4'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'hd0 ) )
	    psw_other <= {wr_byte[5:3],wr_byte[1]};
	else;
else;

assign psw_rs = psw_other[2:1];

assign wr_psw_rs = wr_en_sfr & (wr_addr[7:0]==8'hd0);

assign psw = {psw_c,psw_ac,psw_other[3:1],psw_ov,psw_other[0],psw_p};

/*********************************************************/

/*********************************************************/
//dp  sp registers

always @ ( posedge clk or posedge rst )
if ( rst )
    dp <= 16'b0;
else if ( work_en )
    if  ( wr_en_sfr & (wr_addr[7:0]==8'h82 ) )
	    dp[7:0] <= wr_byte;
	else if  ( wr_en_sfr & (wr_addr[7:0]==8'h83 ) )
	    dp[15:8] <= wr_byte;
	else if ( inc_dp(cmdb) )
	    dp <= dp + 1'b1;
	else if ( mov_dp_da(cmdc) )
	    dp <= {cmd1,cmd0};
	else;
else;

assign wr_dp = (wr_en_sfr & ((wr_addr[7:0]==8'h82)|(wr_addr[7:0]==8'h83)))|inc_dp(cmdb);

always @ ( posedge clk or posedge rst )
if ( rst )
    sp <= 8'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'h81) )
	    sp <= wr_byte;
	else if ( push(cmdb)|acall(cmdb)|acall(cmdc)|lcall(cmdb)|lcall(cmdc) )
	    sp <= sp_add1;
    else if ( pop(cmdb)|ret(cmdb)|ret(cmdc)|reti(cmdb)|reti(cmdc) )
        sp <= sp_sub1;	
	else;
else;

assign sp_sub1 = sp - 1'b1;

assign sp_add1 = sp + 1'b1;

assign wr_sp = (wr_en_sfr & (wr_addr[7:0]==8'h81));

always @ ( posedge clk or posedge rst )
if ( rst )
    b <= 8'b0;
else if ( work_en )
    if ( wr_en_sfr & (wr_addr[7:0]==8'hf0) )
	    b <= wr_byte;
	else if ( mul(cmdb) )
	    b <= mult[15:8];
	else if ( div(cmdb) )
	    b <= div_rem;
	else;
else;

/*********************************************************/


/*********************************************************/
//ram output

always @*
if ( same_flag )
    data0 = same_byte;
else if ( same_byte[7] )
    data0 = acc;
else if ( same_byte[6] )
    data0 = psw;
else if ( same_byte[5] )
    data0 = dp[15:8];
else if ( same_byte[4] )
    data0 = dp[7:0];
else if ( same_byte[3] )
    data0 = sp;
else if ( same_byte[2] )
    data0 = b;
else
    data0 = ram_rd_byte;

always @ ( posedge clk or posedge rst )
if ( rst )
    data1 <= 8'b0;
else if ( work_en )
    data1 <= data0;
else;
    
always @ ( posedge clk or posedge rst )
if ( rst )
    same_flag <= 1'b0;
else if ( work_en )
`ifdef TYPE8052
    if ( same_flag_data|same_flag_sfr|same_flag_idata|same_flag_xdata )
`else 
    if ( same_flag_data|same_flag_sfr|same_flag_xdata )
`endif	
	    same_flag <= 1'b1;
	else
	    same_flag <= 1'b0;
else;

always @ ( posedge clk or posedge rst )
if ( rst )
    same_byte <= 8'd0;
else if ( work_en )
`ifdef TYPE8052
    if ( same_flag_data|same_flag_sfr|same_flag_idata|same_flag_xdata )
`else
    if ( same_flag_data|same_flag_sfr|same_flag_xdata )
`endif	
	    same_byte <= wr_byte;
	else if ( rd_en_sfr & (rd_addr[7:0]==8'he0) ) //acc
	    same_byte <= 1'b1<<7;
	else if ( rd_en_sfr & (rd_addr[7:0]==8'hd0) ) //psw
	    same_byte <= 1'b1<<6;
	else if ( rd_en_sfr & (rd_addr[7:0]==8'h83) ) //dph
	    same_byte <= 1'b1<<5;	
	else if ( rd_en_sfr & (rd_addr[7:0]==8'h82) ) //dpl
	    same_byte <= 1'b1<<4;	
	else if ( rd_en_sfr & (rd_addr[7:0]==8'h81) ) //sp
	    same_byte <= 1'b1<<3;
	else if ( rd_en_sfr & (rd_addr[7:0]==8'hf0) ) //b
	    same_byte <= 1'b1<<2;		
	else
	    same_byte <= 8'b0;
else;
	
/*********************************************************/

endmodule

```
