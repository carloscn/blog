# DSP-F2812的CMD文件 

我们编写F2812文件的时候必须要添加一个CMD文件嘛！那个CMD文件的作用就是把结构体编写的文件映射到DSP内部的内存中。之前我们通过结构体和共同体的方式定义了很多的寄存器，通过语言就能控制寄存器，然而定义那么多只不过是个规则而已，只是空洞的字母，CMD文件的作用就是把定义那些空洞的字母和内存里面的地址链接起来，这样我们通过控制那些字母就能控制寄存器了。

流程：硬件 --> CMD文件 --> C链接文件 --> 头文件 --> 程序代码

Memory Map：

![image-20221202202957774](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221202202957774.png)

上面是F2812的存储映像图，我们怎么来写CMD文件，首先，我们得弄清楚点儿东西

> 存储器的种类：
>
> 一、补充点儿知识：咱们按照这么分：RAM和ROM和FLASH
> 
> RAM：
> 1. SRAM：静态读写存储器，速度快，容量小，断电后数据丢失。可读可写。
> 2. DRAM：动态读写存储器，速度慢，容量大，断电后数据丢失。需要刷新电路。可读可写。
> 
> 这个就是咱们经常说的电脑里面的内存条，CPU可读可写，断电后里面啥数据都没了。
> 
> ROM：
>  只读存储器，原理不说了，相当于硬盘，CPU不可写。在利用里面数据之前可以写入。里面存放大量的数据和程序。由RAM来读。分为掩膜ROM和可编程ROM。有的ROM只准写一次就不能擦了，所以注意了。
>  
> FLASH：
> FALSH集成了ROM和RAM的优点，比如说U盘就是个FLASH。可读可写，可编程。非常好。
> （来源：计算机组成原理）
> 
> 二、然后咱们看看DSP里面的存储结构：
> 
> 1.    片内SARAM：
> SARAM有RAM关键字，肯定是个RAM了。Single Access RAM. 这是按CPU每个机器周期能对内存进行访问的次数来划分的两种内存。SARAM在一个机器周期内只能被访问一次，而DARAM则在一个机器周期内能被访问两次。均可以作为程序空间也可以作为数据空间。
> 
> 2.    片内OTP
> OTP为ROM，而且还是一次性可编程的空间，可以看到里面是2K × 16的空间。One Time Programmable.
> 
> 3.    Boot ROM
> 引导ROM，装载Ti公司装载的产品的信息。MP/MC = 0，时候CPU置位为这段程序。
> 
> 4.    片内FLASH
> F2812有128K * 16位的片内Flash，Flash空间既可以作为程序空间也可以作为数据空间，FLASH存储器是分区操作的，用户可以单独擦某段区域。

现在，我们弄清楚都是干啥的了，就开始自定义分区，就像是电脑做系统似的，你要安装电脑系统，要分区啊，哪个是C盘，D盘，哪个用来装系统，或者把那个地方用来装自己的文件。当然了装系统用PE分区就行了，简单的点点，这个我们得用命令的方式进行分区，还得按照上面给的地图，进行分，避开重要的部分。现在总结一下，我们有很多的可利用的地方，SARAM(M0，M1，L0，L1，H0)，FLASH，OTP。现在就要用这几个地方分配空间了。

注意几点啊：

1. 文件的大小不能超过这个空间地址大小，这个很好理解吧，我装了个游戏超过这个磁盘是不可能的。

2. 比较大的空间进行分区之后不能有重叠，因为cmd文件这么写的，开始，然后长度，不要有重叠。

之后咱们再说一下分页问题：我理解的分页就是分文件夹。这个目录，创建两个文件夹，因为解释是说，同一页内，定义分区的名字不能相同，不同页内可以相同。不难猜测，就是分文件夹。

还需要了解个知识：COFF格式和段的那些事儿。

C语言会生成一些段：已经初始化的段和还没有初始化的段儿。已经初始化的段儿含有真是的指令和数据，存放在程序存储空间，未初始化的段儿只保留变量的地址空间，未初始化的段儿保存在数据存储空间。

已初始化的段儿：.text(存放汇编指令和代码) .cinit(存放全局和静态变量) .const .econst .pinit .switch

未初始化的段儿：.bbs .ebbs .stack .system .esysmem

CMD文件的编写：

```C
MEMORY{   // 开始分区
	PAGE 0:
		PRAMH0 : origin = 0x3f8000 , length = 0x001000  /* 下面的H0内存8K*16  */
	PAGE 1:
		RAMM0  :  origin = 0x000000 , length = 0x000400  /* SARAM M0区域 */ 
		RAMM1  :  origin = 0x000800 , length = 0x000400  /* SARAM M1区域 */
		DEV_EMU : origin = 0x000880 , length = 0x000180  /* 开始外设帧 */
		FLASH_REGS : origin = 0x000A80 , length = 0x000060 /* FLASH寄存器 */
		CSM        : origin = 0x000AE0, length = 0x000010
   		XINTF      : origin = 0x000B20, length = 0x000020
   		CPU_TIMER0 : origin = 0x000C00, length = 0x000008
  		CPU_TIMER1 : origin = 0x000C08, length = 0x000008		 
   		CPU_TIMER2 : origin = 0x000C10, length = 0x000008		 
  		PIE_CTRL   : origin = 0x000CE0, length = 0x000020
   		PIE_VECT   : origin = 0x000D00, length = 0x000100

   		/* Peripheral Frame 1:   */
   		ECAN_A     : origin = 0x006000, length = 0x000100
   		ECAN_AMBOX : origin = 0x006100, length = 0x000100

   		/* Peripheral Frame 2:   */
   		SYSTEM     : origin = 0x007010, length = 0x000020
   		SPI_A      : origin = 0x007040, length = 0x000010
   		SCI_A      : origin = 0x007050, length = 0x000010
   		XINTRUPT   : origin = 0x007070, length = 0x000010
   		GPIOMUX    : origin = 0x0070C0, length = 0x000020
   		GPIODAT    : origin = 0x0070E0, length = 0x000020
   		ADC        : origin = 0x007100, length = 0x000020
   		EV_A       : origin = 0x007400, length = 0x000040
   		EV_B       : origin = 0x007500, length = 0x000040
   		SPI_B      : origin = 0x007740, length = 0x000010
   		SCI_B      : origin = 0x007750, length = 0x000010
   		MCBSP_A    : origin = 0x007800, length = 0x000040

   		/* CSM Password Locations */
   		CSM_PWL    : origin = 0x3F7FF8, length = 0x000008

   		/* SARAM                    */     
   		DRAMH0     : origin = 0x3f9000, length = 0x001000         
}
 SECTIONS{
 /* Allocate program areas: */
   .reset           : > PRAMH0,      PAGE = 0
   .text            : > PRAMH0,      PAGE = 0
   .cinit           : > PRAMH0,      PAGE = 0

   /* Allocate data areas: */
   .stack           : > RAMM1,       PAGE = 1
   .bss             : > DRAMH0,      PAGE = 1
   .ebss            : > DRAMH0,      PAGE = 1
   .const           : > DRAMH0,      PAGE = 1
   .econst          : > DRAMH0,      PAGE = 1      
   .sysmem          : > DRAMH0,      PAGE = 1
   
   /* Allocate Peripheral Frame 0 Register Structures:   */
   DevEmuRegsFile    : > DEV_EMU,    PAGE = 1
   FlashRegsFile     : > FLASH_REGS, PAGE = 1
   CsmRegsFile       : > CSM,        PAGE = 1
   XintfRegsFile     : > XINTF,      PAGE = 1
   CpuTimer0RegsFile : > CPU_TIMER0, PAGE = 1      
   CpuTimer1RegsFile : > CPU_TIMER1, PAGE = 1      
   CpuTimer2RegsFile : > CPU_TIMER2, PAGE = 1      
   PieCtrlRegsFile   : > PIE_CTRL,   PAGE = 1      
   PieVectTable      : > PIE_VECT,   PAGE = 1

   /* Allocate Peripheral Frame 2 Register Structures:   */
   ECanaRegsFile     : > ECAN_A,      PAGE = 1   
   ECanaMboxesFile   : > ECAN_AMBOX   PAGE = 1

   /* Allocate Peripheral Frame 1 Register Structures:   */
   SysCtrlRegsFile   : > SYSTEM,     PAGE = 1
   SpiaRegsFile      : > SPI_A,      PAGE = 1
   SciaRegsFile      : > SCI_A,      PAGE = 1
   XIntruptRegsFile  : > XINTRUPT,   PAGE = 1
   GpioMuxRegsFile   : > GPIOMUX,    PAGE = 1
   GpioDataRegsFile  : > GPIODAT     PAGE = 1
   AdcRegsFile       : > ADC,        PAGE = 1
   EvaRegsFile       : > EV_A,       PAGE = 1
   EvbRegsFile       : > EV_B,       PAGE = 1
   ScibRegsFile      : > SCI_B,      PAGE = 1
   McbspaRegsFile    : > MCBSP_A,    PAGE = 1

   /* CSM Password Locations */
   CsmPwlFile      : > CSM_PWL,     PAGE = 1

```

下面我们就说说这个文件：DSP28_GlobalVariableDefs.c

```C
#include <DSP28_Device.h>
#pragma DATA_SECTION(AdcRegs,”AdcRegsFile”);
Volatile struct ADC_REGS AdcRegs;
```

这个就是链接C头文件和CMD文件的中介，它首先是一个c文件。这样明白了CMD文件的全过程。

还有一点重要的就是RAM中运行的程序比较慢，我们想要提高速度要放在flash里面。

外部接口XINTF：

![image-20221202203253956](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221202203253956.png)

看见没，上面各种Zone0，zone1，zone2，zone6，zone7。就是能扩充的区域。要这一条不是一个内存条啊，每一个空间就是一个内存条，而且我们往上面按内存的时候一定要按照上面的内存空间进行扩充，上面8K的就不能添加16K的，必须按照规格进行扩充

XINTF接口已经被映射到了空间01267 ，5个固定的存储空前，范围就是如上图所示啦。每个信号线的作用有的我们能猜出来，有的我们当然猜不出来了，但是原理都差不多，当系统使能某个片选信号，相应的设备被选中，就能进行读或者写操作。上面的英文说得很仔细了，有的是单独的片选信号，有的是公用一个片选信号，用了个或门。那我们就知道了XZCS就是片选信号。下面解释解释我们不知道的信号标示！

* XD(15~0):共16根线，外扩数据总线，总共16根
* XA(18~0):共19根线，外扩的地址总线，外部选址寻址空间最大为512K *16 ！
* XR/W:通常为高电平。当为低电平时候表示处于写周期，当为高电平的时候处于读周期。
* XREADY：数据准备输入信号，高电平表示外设已经做好了准备。
* XMP/MC：微处理器/微计算机模式选择信号。（前面介绍过，此电平为0，DSP成为微计算机模式，CPU复位后，Boot ROM中的引导程序自运行一遍。）
* XHOLD : 外部DMA去请求保持信号。
* XHOLDA：外部DMA保持确认信号。
* XCLKOUT：时钟信号。

现在有个问题：0,1空间 6,7空间都各自共用一个片选信号，但是如果我非得想用0和6的组合，我要怎么做呢。对外界的片选，是怎么实现呢？（假设现在我就要用0,6组合，外边有个8K*16 RAM和512K*16 FLASH。）

秘诀就在这里：Zone0的寻址空间和Zone1的遵旨空间分别为0x2000~0x3FFF，而Zone1的寻址空间为0x4000~0x5FFF。看到了没，低12位的变化一样，而高4位却不一样。好了我把XA12~XA15拿出来和共享片选信号进行与门处理，就能达到单选一个空间的目的了。

注意啦：Boot ROM被使能的话，Zone7是不被使用的。这破DSP规则还挺多的呢！记住吧。

XINTF的时钟

驱动一个东西，需要电源，需要接地，还需要的基本的就是时钟，时钟就如同心脏在跳动，为一切的功能提供一个时序。是非常重要的。时钟的原理我就不说了，接入SysCtrl进行分频。

例子：XINTF接口对外扩RAM或者FLASH空间。如果我们手里有256K *16的RAM和256K *16的FLASH。根据内存图，看看那里可以接受256K *16的的空间：

 8K *16的就别想了，不够用。选择zone2和zone6吧因为就他俩是512K的，剩下的就浪费掉不进行扩充。

例1：外部RAM空间数据读/写操作

现在我们怎么算这个长度呢？算法这么算的！ 256 * 1024 = 262144 转化为16进制 0x400，所以长度为0x400

为512K 为 512*1024 = 524288 转化为16进制 0x80000。看到没，差多了。我们十进制看到的256和512在硬件中差的不是2倍的关系，差的是 0x80000- 0x400这么多。

我们的要求是，从起始地址开始写数据，内容从0开始，随着地址增加不断加1，长度为0x4000，即16k。然后从内存空间读取这些数据，最后将外部RAM空间清0.这个程序怎么实现呢？

Ram.c文件：

```C
#include <DSP28_Device.h>

Uint16 * ExRamStart = (Uint16 *)0x100000; // 外扩内存的其实地址，我们可以看的很清楚了。

void RamWrite(Uint16 Start){
	Uint16 i;
	for(i = 0; i < 0x4000; i++){
		*(ExRamStart + Start + i) = i;  // 这个就是地址写数据啦，Start是我们要求开始的位置。
	}

}
void RamClear(Uint16 Start){   // 清零
	Uint16 i;
	for(i = 0; i < 0x400 ; i++){
		*(ExRamStart + Start + i) = 0;
	}
}
void RamRead(Uint16 Start){      // 读数据
	Uint16 i;
	for(i = 0; i < 0x4000;i++){
		*(ExRamStart + Start + i) = * (ExRamStart + i);
	}
}


// Main文件
#include <DSP28_Device.h>

void RamWrite(Uint16);
void RamRead(Uint16);
void RamClear(Uint16);

void main(void) {
	
	InitSysCtrl();
	DINT;
	IER = 0x0000;
	IFR = 0x0000;
	InitPieCtrl();
	InitPieVectTable();
	RamWrite(0);
	RamRead(0x4000);
	RamClear(0x00);
	while(1);

}

```

这是一个非常简单的操作！内存呢，我们就先说这么多，还有个FLASH扩展，等哪天咱们用了就会给补上。内存结束了。怎么回事儿是不是很清晰了。下面开始附参考文献，与时代和现实接轨！