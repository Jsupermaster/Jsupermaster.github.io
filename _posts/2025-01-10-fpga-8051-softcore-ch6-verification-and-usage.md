---
title: "FPGA 8051软核处理器设计实战 - 6. 8051软核处理器的验证和使用"
date: 2025-01-10 20:00:00 +0800
description: "为了充分测试8051的性能，我们需要测试每一条指令。在HELLO文件夹中存放了整个测试的C语言工程文件。主函数存放在指令被分为五大类，和上面一样。"
categories:
  - FPGA
home_category: reading-notes
series: fpga-8051-softcore-design
tags:
  - FPGA
  - "8051"
  - Verilog
  - Book Notes
article_kicker: BOOK NOTE
cover_image: /assets/images/fpga_8051_softcore/51KE6P2qUSL._SY445_SX342_.jpg
---

### 6. 8051软核处理器的验证和使用

为了充分测试8051的性能，我们需要测试每一条指令。在HELLO文件夹中存放了整个测试的C语言工程文件。主函数存放在指令被分为五大类，和上面一样。

![屏幕截图 2025-01-14 112132](/assets/images/fpga_8051_softcore/屏幕截图 2025-01-14 112132.png)

![屏幕截图 2025-01-14 112245](/assets/images/fpga_8051_softcore/屏幕截图 2025-01-14 112245.png)

打开后是这样的文件结构。HELLO.c是主文件，这是里面的代码：

```c
/*------------------------------------------------------------------------------
HELLO.C

Copyright 1995-2005 Keil Software, Inc.
------------------------------------------------------------------------------*/

#include <REG52.H>                /* special function register declarations   */
                                  /* for the intended 8051 derivative         */

#include <stdio.h>                /* prototype declarations for I/O functions */

#include "instruction.h"


/*------------------------------------------------
The main C function.  Program execution starts
here after stack initialization.
------------------------------------------------*/
void main (void) {
  test_status = 1;
	
	instruction_test_all();
	
	
	if (test_status) {
		printf("Test success!\n");
	}else{
		printf("Test failed!\n");
	}
  printf("Test finished!\n");
	kill_self = 1;
  while (1);
}
```

这里引用了头文件`instruction.h`，它的实现在Instruction文件夹中，先看一下`instruction.c`的内容：

```c
#include <REG52.H>
#include <stdio.h> 
#include "instruction.h"

void error(void){
	if (test_status==0) {
    printf("ERROR HERE...\n");
		while(1);
	}
}	

void instruction_test_all(void){
#ifdef ARITHMETIC
  arithmetic();  
#endif
#ifdef LOGICAL
	logical();
#endif
#ifdef TRANSFER
	transfer();
#endif
#ifdef BOOLEAN
	boolean();
#endif
#ifdef PROGRAM
	program();
#endif
}

void arithmetic(void){
#ifdef ADD_A_RN
    add_a_rn();
#endif
#ifdef ADD_A_DI
    add_a_di();
#endif	
#ifdef ADD_A_RI
    add_a_ri();
#endif		
#ifdef ADD_A_DA
    add_a_da();
#endif
#ifdef ADDC_A_RN
    addc_a_rn();
#endif	
#ifdef ADDC_A_DI
    addc_a_di();
#endif		
#ifdef ADDC_A_RI
    addc_a_ri();
#endif
#ifdef ADDC_A_DA
    addc_a_da();
#endif
#ifdef SUBB_A_RN
    subb_a_rn();
#endif
#ifdef SUBB_A_DI
    subb_a_di();
#endif
#ifdef SUBB_A_RI
    subb_a_ri();
#endif
#ifdef SUBB_A_DA
    subb_a_da();
#endif
#ifdef INC_A
    inc_a();
#endif
#ifdef INC_RN
    inc_rn();
#endif
#ifdef INC_DI
    inc_di();
#endif
#ifdef INC_RI
    inc_ri();
#endif
#ifdef INC_DP
    inc_dp();
#endif
#ifdef DEC_A
    dec_a();
#endif
#ifdef DEC_RN
    dec_rn();
#endif
#ifdef DEC_DI
    dec_di();
#endif
#ifdef DEC_RI
    dec_ri();
#endif
#ifdef MULT
    mult();
#endif
#ifdef DIVIDE
    divide();
#endif
#ifdef DA_A
    da_a();
#endif
}
```

这里只列出了一部分。可以看出通过预编译指令定义了五类指令及每一类的指令。而每一类中具体的指令通过汇编语言使用预编译指令进行编译。它们分布在剩余的几个c文件中。比如算术运算指令如下：

```c
#include <REG52.H>
#include <stdio.h> 
#include "instruction.h"

void add_a_rn(void) {
	printf("ADD_A_RN\n");
	#pragma ASM  
	push psw
	push acc
  mov  psw,#0H	
  setb rs0     
	setb rs1	
  #pragma ENDASM 
	
	#pragma ASM
	mov acc,#01H
	mov R0,#0fH
	add A,R0
  #pragma ENDASM	
	if (ACC!=0x10) test_status = 0;
	if (AC!=1) test_status = 0;
	if (OV!=0) test_status = 0;
	if (CY!=0) test_status = 0;
	AC = 0;
		
	#pragma ASM
	mov acc,#40H
	mov R1,#40H
	add A,R1
	#pragma ENDASM
	if (ACC!=0x80) test_status = 0;
	if (AC!=0) test_status = 0;
	if (OV!=1) test_status = 0;
	if (CY!=0) test_status = 0;
	OV = 0;

	
	#pragma ASM
  mov acc,#80H
	mov R2,#81H
	add A,R2
  #pragma ENDASM
	if (ACC!=0x01) test_status = 0;
	if (AC!=0) test_status = 0;
	if (OV!=1) test_status = 0;
	if (CY!=1) test_status = 0;
	OV = 0;
	CY = 0;
	
	#pragma ASM
  mov acc,#0C0H
	mov R3,#0C2H
	add A,R3
  #pragma ENDASM
	if (ACC!=0x82) test_status = 0;
	if (AC!=0) test_status = 0;
	if (OV!=0) test_status = 0;
	if (CY!=1) test_status = 0;
	CY = 0;	
	
	#pragma ASM 
	pop acc
  pop psw	
  #pragma ENDASM 	
	error();
}
```

如果每一条指令都通过，最后会打印测试成功字样，如果有任何一条指令执行有误则会导致抛出错误和暂停测试。

将HELLO工程编译后生成了HELLO.bin，这就是我们最后需要使用的文件，将其留存。之后编写一个tb文件，用于对接接口和设置存储空间：

```verilog
`timescale 1 ns/1 ps
`define PERIOD 10
`define HALF_PERIOD (`PERIOD/2)
//`define TYPE8052
`define CODE_FILE "C:/Users/15661/Desktop/R8051_test/HELLO/HELLO.bin"
module tb;

reg     clk = 1'b0;
always #`HALF_PERIOD clk = ~clk;

reg     rst = 1'b1;
initial #`PERIOD rst = 1'b0;

wire            rom_en;
wire [15:0]     rom_addr;
reg  [7:0]      rom_byte;
reg             rom_vld;

wire            ram_rd_en_data;
wire            ram_rd_en_sfr;
wire            ram_rd_en_xdata;
wire [15:0]     ram_rd_addr;

reg  [7:0]      ram_rd_byte;

wire            ram_wr_en_data;
wire            ram_wr_en_sfr;
wire            ram_wr_en_xdata;
wire [15:0]     ram_wr_addr;
wire [7:0]      ram_wr_byte;


r8051 u_cpu (
    .clk                  (    clk              ),
	.rst                  (    rst              ),
	.cpu_en               (    1'b1             ),
	.cpu_restart          (    1'b0             ),
	
	.rom_en               (    rom_en           ),
	.rom_addr             (    rom_addr         ),
	.rom_byte             (    rom_byte         ),
	.rom_vld              (    rom_vld          ),
	
	.ram_rd_en_data       (    ram_rd_en_data   ),
	.ram_rd_en_sfr        (    ram_rd_en_sfr    ),
	.ram_rd_en_xdata      (    ram_rd_en_xdata  ),
	.ram_rd_addr          (    ram_rd_addr      ),
	.ram_rd_byte          (    ram_rd_byte      ),
	.ram_rd_vld           (    1'b1             ),
	
	.ram_wr_en_data       (    ram_wr_en_data   ),
	.ram_wr_en_sfr        (    ram_wr_en_sfr    ),
	.ram_wr_en_xdata      (    ram_wr_en_xdata  ),
	.ram_wr_addr          (    ram_wr_addr      ),
	.ram_wr_byte          (    ram_wr_byte      )

);

reg [7:0] rom[(1'b1<<16)-1:0];

integer fd,fx;
initial begin
  fd = $fopen(`CODE_FILE,"rb");
  fx = $fread(rom,fd);
  $fclose(fd);
end
	
always @ ( posedge clk )
if ( rom_en )
    rom_byte <=  rom[rom_addr];
else;

always @ ( posedge clk )
rom_vld <=  rom_en;


reg [7:0] data [127:0];
reg [7:0] data_rd_byte;
always @ ( posedge clk )
if ( ram_rd_en_data )
    data_rd_byte <=  data[ram_rd_addr[6:0]];
else;

always @ ( posedge clk )
if ( ram_wr_en_data )
    data[ram_wr_addr[6:0]] <=  ram_wr_byte;
else;

reg [7:0] xdata [127:0];
reg [7:0] xdata_rd_byte;
always @ ( posedge clk )
if ( ram_rd_en_xdata )
    xdata_rd_byte <=  xdata[ram_rd_addr[6:0]];
else;

always @ ( posedge clk )
if ( ram_wr_en_xdata )
    if (( ram_wr_addr[6:0]==8'h7f ) & ram_wr_byte[0] ) begin
	    repeat(1000) @ (posedge clk);
		$display("Test over, simulation is OK!");
		$stop(1);
		end
	else
        xdata[ram_wr_addr[6:0]] <=  ram_wr_byte;
else;

reg [7:0] sfr_rd_byte;

always @ ( posedge clk )
if ( ram_wr_en_sfr & ( ram_wr_addr[7:0]==8'h99 ) )
    $write("%s",ram_wr_byte);
else;


always @ ( posedge clk )
if ( ram_rd_en_sfr ) 
    if ( ram_rd_addr[7:0]==8'h98 )
	    sfr_rd_byte <=  8'h3;
	else if ( ram_rd_addr[7:0]==8'h99 )
	    sfr_rd_byte <=  0;
	else
    begin
        $display($time," ns : --- SFR READ: %2h---",ram_rd_addr[7:0]);
        //$stop;
    end
else;	

always @ ( posedge clk )
if ( ram_wr_en_sfr )
    if(( ram_wr_addr[7:0]==8'h98 )|( ram_wr_addr[7:0]==8'h99 ))
	    #0;
    else	
    begin
    $display($time," ns : --- SFR WRITE: %2h -> %2h---",ram_wr_addr[7:0],ram_wr_byte);
    //$stop;
    end
else;	

reg [1:0] read_flag;
always @ ( posedge clk )
if ( ram_rd_en_sfr )
    read_flag <= 2'b10;
else if ( ram_rd_en_xdata )
    read_flag <= 2'b01;	
else if ( ram_rd_en_data )
    read_flag <= 2'b0;
else;

always @*
if ( read_flag[1] )
    ram_rd_byte = sfr_rd_byte;
else if ( read_flag[0] )
    ram_rd_byte = xdata_rd_byte;
else
    ram_rd_byte = data_rd_byte;

endmodule

```

将tb.v和R8051.v一起加入Modelsim工程（注意不要导入instruction.v，否则会报错，需要更改的有两处路径，分别是tb中引用HELLO.bin的路径，和R8051.v中对instruction.v引用的路径，更改后即可复现代码。）之后进行仿真，结果正确。

![屏幕截图 2025-01-14 113417](/assets/images/fpga_8051_softcore/屏幕截图 2025-01-14 113417.png)

![屏幕截图 2025-01-14 113435](/assets/images/fpga_8051_softcore/屏幕截图 2025-01-14 113435.png)

至此，我们的8051软核处理器开发基本完毕。之后可以自行修改一些功能，使用这套测试框架进行测试。现在这个软核处理器中没有添加中断，在书中并没有提供添加中断的说明。后续如果有机会将继续更新添加中断的内容。
