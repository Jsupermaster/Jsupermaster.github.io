---
title: "FPGA 8051软核处理器设计实战 - 3.8051中断与keil开发流程"
date: 2025-01-07 20:00:00 +0800
description: "关于keil，大家都并不陌生，它是开发51单片机和ARM架构的32单片机的有力工具。关于下载安装方法这里不多赘述，请参阅其他文章。需要注意的是：keil分为ARM和C51版本，我们需要安装C51的版本。"
categories:
  - FPGA
tags:
  - FPGA
  - 8051
  - Verilog
  - Book Notes
article_kicker: BOOK NOTE
cover_image: /assets/images/fpga_8051_softcore/51KE6P2qUSL._SY445_SX342_.jpg
---

### 3.8051中断与keil开发流程

#### 3.1 keil的下载与概述

关于keil，大家都并不陌生，它是开发51单片机和ARM架构的32单片机的有力工具。关于下载安装方法这里不多赘述，请参阅其他文章。需要注意的是：keil分为ARM和C51版本，我们需要安装C51的版本。

下载完keil后我们可以在安装路径找到这些文件：

![屏幕截图 2024-12-30 143839](/assets/images/fpga_8051_softcore/屏幕截图 2024-12-30 143839.png)

点击进入C51文件夹，找到Examples文件夹，点击进入：

![屏幕截图 2024-12-30 143950](/assets/images/fpga_8051_softcore/屏幕截图 2024-12-30 143950.png)

其中包含很多文件夹，这些文件夹是一些示例程序，有针对特定芯片的工程，也有不针对特定芯片的通用工程。我们点击进入Hello文件夹，来看看最简单的工程：

![屏幕截图 2024-12-30 144340](/assets/images/fpga_8051_softcore/屏幕截图 2024-12-30 144340.png)

双击hello.uvproj即可打开keil的工程，界面如下：

![屏幕截图 2024-12-30 144622](/assets/images/fpga_8051_softcore/屏幕截图 2024-12-30 144622.png)

为了方便说明，把这段程序复制如下：

```c
/*------------------------------------------------------------------------------
HELLO.C

Copyright 1995-2005 Keil Software, Inc.
------------------------------------------------------------------------------*/

#include <REG52.H>                /* special function register declarations   */
                                  /* for the intended 8051 derivative         */

#include <stdio.h>                /* prototype declarations for I/O functions */


#ifdef MONITOR51                         /* Debugging with Monitor-51 needs   */
char code reserve [3] _at_ 0x23;         /* space for serial interrupt if     */
#endif                                   /* Stop Exection with Serial Intr.   */
                                         /* is enabled                        */


/*------------------------------------------------
The main C function.  Program execution starts
here after stack initialization.
------------------------------------------------*/
void main (void) {

/*------------------------------------------------
Setup the serial port for 1200 baud at 16MHz.
------------------------------------------------*/
#ifndef MONITOR51
    SCON  = 0x50;		        /* SCON: mode 1, 8-bit UART, enable rcvr      */
    TMOD |= 0x20;               /* TMOD: timer 1, mode 2, 8-bit reload        */
    TH1   = 221;                /* TH1:  reload value for 1200 baud @ 16MHz   */
    TR1   = 1;                  /* TR1:  timer 1 run                          */
    TI    = 1;                  /* TI:   set TI to send first char of UART    */
#endif

/*------------------------------------------------
Note that an embedded program never exits (because
there is no operating system to return to).  It
must loop and execute forever.
------------------------------------------------*/
  while (1) {
    P1 ^= 0x01;     		    /* Toggle P1.0 each time we print */
    printf ("Hello World\n");   /* Print "Hello World" */
  }
}
```

这段代码的含义是把“Hello World\n”这12个字符（包含空格和换行符）从初始化完毕后的串口按照顺序写入SBUF寄存器上，之后单片机通过串口把SBUF内的字符通过串口输出。当我们打开串口调试助手，可以看到单片机发回的字符。由于写入while(1)循环，将会不断输出这句话。在电脑上使用printf时，字符被打印到屏幕上，但对于单片机而言，这些字符需要一个指定的打印终端，在这里我们指定串口为打印终端。在打印之前，程序先对串口进行了配置，以保证其正常输出。

```c
#ifndef MONITOR51
    SCON  = 0x50;		        /* SCON: mode 1, 8-bit UART, enable rcvr      */
    TMOD |= 0x20;               /* TMOD: timer 1, mode 2, 8-bit reload        */
    TH1   = 221;                /* TH1:  reload value for 1200 baud @ 16MHz   */
    TR1   = 1;                  /* TR1:  timer 1 run                          */
    TI    = 1;                  /* TI:   set TI to send first char of UART    */
#endif
```

这段程序就是配置串口的过程，其中SCON、TMOD、TH1、TR1、TI都是8051的寄存器，这些对于我们使用8051MCU进行编程很重要，但对于我们自己设计的软核处理器不必要过多关注，在这里不多赘述，详细内容可以参阅51单片机神级教程——江协科技视频。

这些寄存器的地址在REG52.H这个头文件中存放，但直接打开工程无法找到这个头文件，点击编译（左上角两个向下箭头的rebuild键），这个文件将被自动编译并添加。

```c
/*  BYTE Registers  */
sfr P0    = 0x80;
sfr P1    = 0x90;
sfr P2    = 0xA0;
sfr P3    = 0xB0;
sfr PSW   = 0xD0;
sfr ACC   = 0xE0;
sfr B     = 0xF0;
sfr SP    = 0x81;
sfr DPL   = 0x82;
sfr DPH   = 0x83;
sfr PCON  = 0x87;
sfr TCON  = 0x88;
sfr TMOD  = 0x89;
sfr TL0   = 0x8A;
sfr TL1   = 0x8B;
sfr TH0   = 0x8C;
sfr TH1   = 0x8D;
sfr IE    = 0xA8;
sfr IP    = 0xB8;
sfr SCON  = 0x98;
sfr SBUF  = 0x99;
```

此处只列举了一小部分，详细内容大家可以自己去查看。这里使用sfr关键字定义，在 C 语言中，`sfr` 关键字通常用于定义一个 8 位特殊功能寄存器的别名。sfr指定特定的寄存器，sbit指定寄存器的特定位：

```c
/*  BIT Registers  */
/*  PSW  */
sbit CY    = PSW^7;
sbit AC    = PSW^6;
sbit F0    = PSW^5;
sbit RS1   = PSW^4;
sbit RS0   = PSW^3;
sbit OV    = PSW^2;
sbit P     = PSW^0; //8052 only
```

这里定义的就是PSW寄存器各个位的名称和地址，我们使用定义的名称可以直接访问寄存器特定的位。

#### 3.2 keil的配置

我们回顾一下，单片机是如何运行起来的。我们在keil中使用C语言编写程序，keil将其转换为汇编代码和机器指令，并生成对应二进制文件，这个二进制文件放入我们设计的8051软核的代码区，便可使其执行对应程序。那么我们需要首先配置keil输出二进制文件。

![屏幕截图 2024-12-30 150228](/assets/images/fpga_8051_softcore/屏幕截图 2024-12-30 150228.png)

选中项目根目录：Simulator，点击魔术棒，弹出配置窗口。

![屏幕截图 2024-12-30 150420](/assets/images/fpga_8051_softcore/屏幕截图 2024-12-30 150420.png)

点击到device页，可以看到目前默认的是8XC52，它拥有2个DPTR，256Byte RAM，说明它是一款8052型单片机。8051的片内RAM是128Byte，8052是256Byte。接下来回到target页面：

![屏幕截图 2024-12-30 150638](/assets/images/fpga_8051_softcore/屏幕截图 2024-12-30 150638.png)

在Memory Model界面下拉有3个选项，表示将变量存储在DATA、PDATA或XDATA中。

- DATA：前面所述的DATA区，大小128字节，使用MOV指令0x00\~0x7F可以直接访问；
- PDATA：XDATA的低256字节，地址空间0x00\~0xFF，使用MOVX@Rn指令访问（Rn寄存器间接寻址），可以直接使用8bit地址访问；
- XDATA：片外的XDATA存储区，使用MOVX指令访问，需要提供16bit地址（MOVX@DPTR），大小为64KB。

![屏幕截图 2024-12-30 151306](/assets/images/fpga_8051_softcore/屏幕截图 2024-12-30 151306.png)

Memory Model设置的是变量存储位置，下一个选项Code Rom Size则设置的是程序大小，三个选项的含义如下：

- Small：整个工程不超过2KB；
- Compact：每个子函数不超过2KB，整个工程最大可达64KB；
- Large：整个工程可以是64KB，子函数大小无限制。

配置完成后，我们编写的C语言程序就有了运行空间，程序编写完成，经过keil编译，就可以得到ROM空间的二进制程序，把这个二进制数据送入真正单片机的ROM中，就可以执行程序了。为了获得二进制文件，我们还需要进行一系列配置：

首先在刚才的设置窗口点到Output页面，勾选Create Hex File选项，这样编译器会在编译完成后输出Hex文件：

![屏幕截图 2024-12-30 151845](/assets/images/fpga_8051_softcore/屏幕截图 2024-12-30 151845.png)

然而我们实际使用的时候，一般不适用hex文件，而是需要使用纯二进制文件（bin），我们需要下载一个插件“HEX2BIN.exe”，将其放入安装目录下：(\Keil_v5\C51\BIN)

![屏幕截图 2024-12-30 152119](/assets/images/fpga_8051_softcore/屏幕截图 2024-12-30 152119.png)

然后在keil中刚才的设置界面的User页面，选中After Build/Rebuild的第一项，输入以下代码：

![屏幕截图 2024-12-30 152310](/assets/images/fpga_8051_softcore/屏幕截图 2024-12-30 152310.png)

输入的代码是这个：

```c
$k\C51\BIN\hex2bin.exe -s 0 -p 0 @H.hex
```

点击OK，回到主界面，点击Rebuild，会出现如下提示：

![屏幕截图 2024-12-30 152457](/assets/images/fpga_8051_softcore/屏幕截图 2024-12-30 152457.png)

打开工程目录，发现已经生成bin文件，说明配置成功。

![屏幕截图 2024-12-30 152625](/assets/images/fpga_8051_softcore/屏幕截图 2024-12-30 152625.png)

#### 3.3 keil工程的创建

了解了大概流程后，我们新建一个工程，点击Project->New uVersion Project，之后选择文件夹，设置文件名，来到这个界面：

![屏幕截图 2024-12-30 152902](/assets/images/fpga_8051_softcore/屏幕截图 2024-12-30 152902.png)

在这里可以选择对应单片机型号，由于我们要自己设计一个软核处理器，所以我们在这里选择Generic，这里可以选择4种型号，我们选择8051：

![屏幕截图 2024-12-30 153028](/assets/images/fpga_8051_softcore/屏幕截图 2024-12-30 153028.png)

点击OK，会弹出一个窗口，询问是否选择汇编程序STARTUP.A51作为起始程序，我们这里不需要，点击否，可进入工程主页面。

点击File->new新建一个文件，输入以下代码：

```c
void main(void) {

}
```

ctrl+s将其另存为.c文件，之后右键点击Source Group1，将其添加到工程中。点击Rebuild，输出如下信息：

![屏幕截图 2024-12-30 153432](/assets/images/fpga_8051_softcore/屏幕截图 2024-12-30 153432.png)

这里的code=16说明这段代码最后合成了16字节的指令组合，我们查看其具体内容，在菜单中选择Debug->Start/stop Debug Session，进入调试界面：

![屏幕截图 2024-12-30 153845](/assets/images/fpga_8051_softcore/屏幕截图 2024-12-30 153845.png)

左侧这部分是之前学过的8051的寄存器，上面是对应的代码序列，下面是我们的程序，这里我们详细逐句解读代码：

```c
C:0x0000    020003   LJMP     C:0003
C:0x0003    787F     MOV      R0,#0x7F
C:0x0005    E4       CLR      A
C:0x0006    F6       MOV      @R0,A
C:0x0007    D8FD     DJNZ     R0,C:0006
C:0x0009    758107   MOV      SP(0x81),#0x07
C:0x000C    02000F   LJMP     main(C:000F)
C:0x000F    22       RET      
```

首先来看第一条指令，我们先从形式上解析一下。前面的“C:0x0000”代表这条指令的地址，“020003”是以十六进制表示的机器码，我们将其转换为16进制：“0000 0010_0000 0000_0000 0011”可见其是3字节的。我们回顾LJMP的指令组成：

&#123;&#123;0000_0010},{address[15:8]},{address[7:0]&#125;&#125;这里的address是十六bit的十六进制数0003，我们发现，这个机器码就是我们前面学习的指令组成，第一字节是指令识别序列，在这里第二三字节组合在一起是跳转地址。

后面的LJMP C:0003就是和机器码一一对应的注记汇编指令。

所以我们明白了，keil生成的一条指令是这样构成的：

```c
C:0x0000    020003    LJMP     C:0003
指令地址     机器码     汇编代码
```

那么它生成的二进制bin文件又是怎么存储的呢？我们使用winhex打开它生成的bin文件：（注意：当你每一次创建新的工程，都需要把刚才的配置步骤重新完成，主要是设置输出Hex文件和在User界面输入代码以生成Bin文件；还有在hex输出界面，新建工程默认会创建一个Object文件夹，把hex输出放在那个文件夹中，这会导致hex2bin程序找不到这个文件，必须重新设置，选择输出路径和当前项目的目录一致，就是uproj的同一个目录下）

![屏幕截图 2024-12-30 160020](/assets/images/fpga_8051_softcore/屏幕截图 2024-12-30 160020.png)



可见02 00 03这就是我们第一条指令的机器码，后面也可完全对应。说明bin文件中写入的就是生成指令序列的机器码。16字节也恰好是机器码的长度。

研究完它的输出，我们继续解析这些指令。刚才已经讲解了第一条指令，它的含义是跳转到0003地址继续执行，那么这个地址其实也就是我们第二条指令的地址。

```c
C:0x0003    787F     MOV      R0,#0x7F	;设置R0值为0x7F(DATA区最高地址)
C:0x0005    E4       CLR      A			;将ACC值清0
C:0x0006    F6       MOV      @R0,A		;将A写入R0保存的地址对应数据区，即将0写入0x7F
C:0x0007    D8FD     DJNZ     R0,C:0006	;循环，R0减一，不等于0就跳转，意为清楚DATA区所有数据
```

这4条指令执行的作用是让DATA区清0，具体的含义见上面的注释。说明：DATA区的地址是00H\~7FH，这里先将最高位地址（7FH）存入R0，使用寄存器间接寻址，清楚这个地址的数据，之后再进行循环，使R0减一，不断执行操作，直到最后地址变为00H。这段内容可对应第二章反复对照学习。

```c
C:0x0009    758107   MOV      SP(0x81),#0x07
C:0x000C    02000F   LJMP     main(C:000F)
C:0x000F    22       RET      
```

接着，设置了堆栈段初始值，SP是栈顶地址，这表明除了寄存器组的第一组（R0\~R7），剩下的DATA区都会用作堆栈区。之后的LJMP作用是跳转到主程序进行执行，由于main函数中没有任何语句，所以直接返回（RET）,但作为主函数，它不能返回，因此我们应该在刚才的程序中加入一条死循环，放置程序跳转回不可控地区。

```c
void main(void) {
	while(1);
}
```

这时我们再重新编译，调试，看看它的汇编代码：

```c
C:0x0000    020003   LJMP     C:0003
C:0x0003    787F     MOV      R0,#0x7F
C:0x0005    E4       CLR      A
C:0x0006    F6       MOV      @R0,A
C:0x0007    D8FD     DJNZ     R0,C:0006
C:0x0009    758107   MOV      SP(0x81),#0x07
C:0x000C    02000F   LJMP     main(C:000F)
     2:         while(1); 
C:0x000F    80FE     SJMP     main(C:000F)
```

发现最后的RET返回指令没有了，取而代之的是一条SJMP指令，其跳转回自己的地址，即反复执行这句话，达到了死循环的效果。

#### 3.4 一个C语言程序实例

现在我们了解到keil是如何把C语言转换为CPU可以执行的指令序列。无论多么复杂的程序，本质上都是这些基本指令序列的组合。使用C语言开发单片机，和我们在电脑上使用C语言有一些不同之处，最大的区别是我们经常需要指定一个特定的寄存器。

假设我们编写这样一段代码：

```c
sfr DISPLAY = 0xc0;

void print(char *str) {
    while(*str!='\0'){
        DISPLAY = *str;
        str++;
    }
}

void main (void) {
    print("Hello World\n");
    while(1);
}
```

其中，DISPLAY是我们自定义的接收打印字符的寄存器，“sfr DISPLAY = 0xc0”，表示它位于SFR区的C0位置，我们前面已经说明过，SFR区中空余的位置我们可以定义自己的寄存器。但是现在这个函数无法打印变量，因为它不是系统函数printf。

下面这个程序解决了这个问题：

```c
#include<stdio.h>
void main (void) {
	char i = 0;
    while(1){
        printf("Hello World %d \n",i);
        i++;
    }
}
```

这个系统函数调用的是‘’C51/LIB‘的“PUTCHAR.c”函数来打印，如下：

```c
char putchar (char c)  {

  if (c == '\n')  {
    if (RI)  {
      if (SBUF == XOFF)  {
        do  {
          RI = 0;
          while (!RI);
        }
        while (SBUF != XON);
        RI = 0; 
      }
    }
    while (!TI);
    TI = 0;
    SBUF = 0x0d;                         /* output CR  */
  }
  if (RI)  {
    if (SBUF == XOFF)  {
      do  {
        RI = 0;
        while (!RI);
      }
      while (SBUF != XON);
      RI = 0; 
    }
  }
  while (!TI);
  TI = 0;
  return (SBUF = c);
}
```

这个函数使用系统定义的串口接收寄存器SBUF，如果想使用自定义的寄存器可以重写putchar函数：

```c
#include<stdio.h>

sfr DISPLAY = 0xc0;

char putchar(char c) {
	return (DISPLAY = c);
}

void main (void) {
	char i = 0;
    while(1){
        printf("Hello World %d \n",i);
        i++;
    }
}
```

#### 3.5 8051中断与中断程序编写

中断是一种“意外”，可能是程序员故意添加的，也可能是由于内部执行错误自动抛出的。我们在这里讨论的是第一种，即我们自己创造的中断。如果没有中断，程序会按照既定的序列不断执行，但有时我们需要人为干预它的执行，当我们给出这个执行中断信号时，处理器应当保存当前指令的下一条指令地址，然后跳转到中断向量处开始执行，然后在执行完中断程序后再次回到之前的程序。

8051最多支持32个中断，这些中断指向的地址都是以0x3或0xB结尾的。这32个中断向量分别是0x03、0x0B、0x13、0x1B......0xF3、0x0B。处理一条中断，相当于处理器执行了一次LCALL指令，但这是强迫发生的，而非程序自然执行发生的。

中断控制器应当具备接收和发出的功能，即接收RETI指令，这条指令除了使程序回到之前的位置继续执行，还通知中断控制器之前的中断执行结束，中断控制器只有接收到这个信号后才能再次发起中断。中断控制器发出的是一个模拟LCALL指令，引导程序从当前执行位置跳转到中断向量开始执行。

在上一段程序中，我们编写中断向量程序：

```c
#include<stdio.h>

sfr DISPLAY = 0xc0;

char putchar(char c) {
	return (DISPLAY = c);
}

void inter(void) interrupt 1 {
    printf("There is 1 interrupt\n");
}

void main (void) {
	char i = 0;
    while(1){
        printf("Hello World %d \n",i);
        i++;
    }
}
```

void inter(void) interrupt 1是中断服务程序的开头。interrupt关键字后面是中断序号，范围是0-31，这里interrupt 1代表中断跳转地址是0x0B。

我们重新编译后，点击调试，将会在汇编程序中找到这条语句：

```c
C:0x000B    0203F8   LJMP     inter(C:03F8)
```

这条长跳转语句放在中断向量处，引导跳转到inter函数，这是因为两个中断向量之间只有8字节位置，不够存放长的程序。

```c
     9: void inter(void) interrupt 1 { 
C:0x03F8    C0E0     PUSH     ACC(0xE0)
C:0x03FA    C0F0     PUSH     B(0xF0)
C:0x03FC    C083     PUSH     DPH(0x83)
C:0x03FE    C082     PUSH     DPL(0x82)
C:0x0400    C0D0     PUSH     PSW(0xD0)
C:0x0402    75D000   MOV      PSW(0xD0),#0x00
C:0x0405    C000     PUSH     0x00
C:0x0407    C001     PUSH     0x01
C:0x0409    C002     PUSH     0x02
C:0x040B    C003     PUSH     0x03
C:0x040D    C004     PUSH     0x04
C:0x040F    C005     PUSH     0x05
C:0x0411    C006     PUSH     0x06
C:0x0413    C007     PUSH     0x07
    10:     printf("There is 1 interrupt\n"); 
C:0x0415    7BFF     MOV      R3,#0xFF
C:0x0417    7A04     MOV      R2,#0x04
C:0x0419    7939     MOV      R1,#0x39
C:0x041B    120070   LCALL    PRINTF(C:0070)
    11: } 
    12:  
C:0x041E    D007     POP      0x07
C:0x0420    D006     POP      0x06
C:0x0422    D005     POP      0x05
C:0x0424    D004     POP      0x04
C:0x0426    D003     POP      0x03
C:0x0428    D002     POP      0x02
C:0x042A    D001     POP      0x01
C:0x042C    D000     POP      0x00
C:0x042E    D0D0     POP      PSW(0xD0)
C:0x0430    D082     POP      DPL(0x82)
C:0x0432    D083     POP      DPH(0x83)
C:0x0434    D0F0     POP      B(0xF0)
C:0x0436    D0E0     POP      ACC(0xE0)
C:0x0438    32       RETI     
```

可以看到，从0x03F8开始执行中断程序，执行中断程序需要保存现场和恢复现场，我们看到程序将ACC、B、DPH、DPL、PSW、R0\~R7全部保存到堆栈中，之后又全部取出。其中保存完PSW后，重新设置了PSW，这是因为之前程序调用的是寄存器组0，而中断程序也要调用寄存器组0，所以需要先保存之前的数据，然后再设置调用寄存器组0，最后再将其复原。如果我们希望让其更加简洁，可以指定中断程序使用寄存器组1，这样它们就不会发生冲突。

```c
#include<stdio.h>

sfr DISPLAY = 0xc0;

char putchar(char c) {
	return (DISPLAY = c);
}

void inter(void) interrupt 1 using 1{
    printf("There is 1 interrupt\n");
}

void main (void) {
	char i = 0;
    while(1){
        printf("Hello World %d \n",i);
        i++;
    }
}
```

这次我们再检查中断程序会发现其更加简洁：

```c
     9: void inter(void) interrupt 1 using 1{ 
C:0x041F    C0E0     PUSH     ACC(0xE0)
C:0x0421    C0F0     PUSH     B(0xF0)
C:0x0423    C083     PUSH     DPH(0x83)
C:0x0425    C082     PUSH     DPL(0x82)
C:0x0427    C0D0     PUSH     PSW(0xD0)
C:0x0429    75D008   MOV      PSW(0xD0),#0x08
    10:     printf("There is 1 interrupt\n"); 
C:0x042C    7BFF     MOV      R3,#0xFF
C:0x042E    7A03     MOV      R2,#0x03
C:0x0430    79F8     MOV      R1,#0xF8
C:0x0432    120070   LCALL    PRINTF(C:0070)
    11: } 
    12:  
C:0x0435    D0D0     POP      PSW(0xD0)
C:0x0437    D082     POP      DPL(0x82)
C:0x0439    D083     POP      DPH(0x83)
C:0x043B    D0F0     POP      B(0xF0)
C:0x043D    D0E0     POP      ACC(0xE0)
C:0x043F    32       RETI     
```

对于市面上MCU的51单片机编程，我们需要仔细阅读它的手册，学习配置寄存器。而对于我们自己设计的软核处理器便可以随意很多，我们可以依据我们的需要创造寄存器，比如设置一个协处理器，将其配置寄存器放在SFR区。本质上来说，处理器只负责按照C语言的调度访问寄存器，再写入寄存器，仅仅是这样简单功能的叠加便可成为我们多彩的程序。使用FPGA设计时，我们可以使其物尽其用，需要什么就添加什么，不需要的可以大刀阔斧地改进。
