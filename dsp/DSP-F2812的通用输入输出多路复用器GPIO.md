# DSP-F2812的通用输入/输出多路复用器GPIO
## 1. GPIO多路复用器

多路复用的意思很明白吧，就是一个引脚有好几个功能。我又一次惊叹了，本来DSP引脚就跟蜈蚣似的，再复用，那岂不是更多功能了。说也是，但是人家TI就是这么设计的，我们必须要遵从这个规则，而且，还要明白，这个是怎么用的。废话不多说了，进入正题。

GPIO是多路复用的意思，也可以做功能引脚，也可以做I/O口，I/O口的意思就是输入输出口，那就说明DSP可以输入可以输出，可以吞吐。那么功能引脚是什么呢？功能引脚很简单了，就是有一定功能的引脚呗。比如说，咱们的EV,SCI,SPI,ECAN，这些丰富的外设，我们想要用这些这些功能的时候，相应地通过外部的一定控制来进行使能，所以就是功能引脚。比如举个简单的栗子，SCI，异步通信接口，这个很通用是吧，这个引脚的名字是：右图所示，一个SCI发送使能引脚，一个SCI接收使能引脚。我们用它来实现SCI的发送和接收的功能。同样滴，这两个引脚可以做通用的I/O口。是不是很神奇。背后的操纵一会儿再说，我们知道了，这个引脚是有两个用法的，一个是功能引脚，一个是通用I/O口。

一下图片是从DSP上面截下来的，是为了让我们更清晰的看见什么是GPIO。这里有一些是GPIO，

![image-20221203094240747](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203094240747.png) ![img](https://raw.githubusercontent.com/carloscn/images/main/typoraclip_image001.png) ![img](https://raw.githubusercontent.com/carloscn/images/main/typoraclip_image002.png)

有以下不是GPIO，但多数都是，PWM肯定是啦，CAP那个也是，SPI也是。

## 2. GPIO的寄存器

GPIO就是I/O引脚的管理机构，它能够管理I/O引脚，召唤出I/O的功能。它的管理机制和理念就是，将56个引脚分成6组进行管理，分为GPIOA，GPIOB,GPIOC,GPIOD,GPIOE,GPIOF。也就是将这56个属于这个I/O的引脚分为了6组，而且这个6组的I/O引脚不是平均分配的，不一定是谁多谁少。他们分别是，16,16,4,3,15,2个引脚。

咱们首先想想，就跟设置东西似的，GPIO能设置什么东西，首先第一步，肯定设置什么引脚吧，那我的选择就是功能引脚或者I/O口呗。那么设置为功能引脚就没什么说的了，功能就固定呗，你是SCI就是SCI，你是EV就是EV。但是如果我选择I/O引脚，又出现两个分路口了吧，是输入？还是输出？对吧，要不然你怎么指定啥是啥功能呢？你说对吧。现在设置什么搞明白了，就开始怎么设置了。DSP的设置通常都是对寄存器进行设置，那么很简单了，我需要了解几个寄存器我就能对GPIO进行相应的设置了。

GPIO的寄存器分为了两种：第一种就是控制类的，包括功能选择寄存器，GPxMUX，方向控制寄存器GPxDIR，输入限定控制寄存器GPxQUAL。第二种就是数据寄存器，主要是数据寄存器GPxDAT，置位寄存器GPxSET，清除寄存器GPxCLEAR和取反寄存器GPxTOGGLE组成。

我先说说怎么记忆，然后再说说都是干啥的。

功能选择，顾名思义就是个选择功能呗，很简单，GPxMUX进行的设置就是这个引脚是IO口，还是功能引脚，举个例子，CANRXA引脚。是CAN总线的接收使能引脚，我查询一下CNRXA是那个GPIO组管辖之内的，现在去找那个表…..原来是GPIOF管辖的脚，现在知道它在GPIOF省了，那么它在哪个市是不是还要知道。哦，原来它在第7市，好了，我需要的信息就是这些了。然后我怎么设置呢？如下：

```c
EALLOW;
GpioMuxRegs.GPFMUX.bit.CANRXA_GPIOF7 = 1; // 0为I/O口，1为CAN总线的功能引脚
EDIS;
```

你看先得允许喽，EALLOW，才能修改，看来是硬件级的。而且命名的规则就是前面是引脚标示+下划线+“省市”。

上面是设置功能引脚，那么设置成I/O口有啥不同呢？

```C
EALLOW;
GpioMuxRegs.GPFMUX.bit.CANRXA_GPIOF7 = 0;
GpioMuxRegs.GPADIR.bit.GPIOF7 = 0;  // 等于1为输出引脚，等于0为输入引脚。
EDIS;
```

上面选择为I/O口了，对应的要设置方向啊，就是输入还是输出，为1就是输出，为0就是输入，这个记忆不难，但是你不记肯定会蒙圈。

好了，以上问题都解决了，进入下一话题。

**GPxQUAL寄存器。**这是个什么寄存器呢，咱们猜测一下，看字面意思，GPx不用说好了，QUAL貌似是序列的意思。GPxQUAL，跟序列有关系。

下面上图再解释。

![image-20221203094552241](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203094552241.png)

能看懂以上的图吗？看一个图，当然先看题目喽。GPIO Signal，GPIO的信号，上面两个高电平然后连续的低电平，之后突然又一个高电平，之后又是低电平，然后连续高电平。这就是对于GPIO信号的描述。我们很容易，也有很大的概率判定那个突变的1是干扰，可能是噪声之类的。或许GPIO这个机制命运颠沛流离吧，各种干扰会来，所以DSP设计GPIO的时候也同样进行了一个抗干扰设计。这个机制就是GPxQUAL寄存器主要的内容。我翻译一下下面的NOTE。**A：这个小故障被输入限定器忽略掉了，这个QUALPRD位（GPxQUAL寄存器）这个地方规定了限定采样周期，它能够从00变化到0xFF。当QUALPRD = 00的时候输入限定没有被应用。对于任何其他的值，这个限定采样周期是2倍的你设定那个任意值。连续6个采样到相同的值将会被认同。-**

我翻译的不是很好，不过那真的是原汁原味的英语啊。用我们的话说，就是存在GPxQUAL寄存器中的一个QUALPRD位，如果将这个位设定为0，那么那种抗干扰的能力不会被使能。如果是其他的值，比如我写了个5，他将会以2*5的CPU时钟周期计时。就像是如图的下面的SYSCLKOUT。如果连续的6个时钟周期采样的值都是一样的，那么久认同这个值是有效的。那么如果有一个像上面的值突变的情况，肯定是这个值被忽略掉了呗。这就是这个什么序列寄存器的作用，就是为了抗干扰的。

### **GPIO**数据寄存器

这个也是个神仙一样的寄存器存在于GPIO中，这个设置可能有两种方法进行设置，我们也要弄清楚，并且看清楚，是置位还是置数。置位的意思就是清零，置数的意思就是写数，当然啦，当我置数的时候置了个0那么就等同于置位了呗。我说的功能上重叠就是这个意思。下面我们看看怎么用这个数据寄存器。

（我还要声明一下：以前设计的内容是一些控制类的寄存器，对于他们呢，我们需要声明EALLOW的保护，而对于现在的数据寄存器，我们只是为了写数而已，所以不用进行EALLOW的声明。）

数据寄存器是，GpioDataRegs，我什么都不说了，上程序，拿程序说话什么都明白了，

```c
if(GpioDataRegs.GPADAT.bit.GPIOA == 1){   // 可读性
	GpioDataRegs.GPADAT.bit.GPIOA = 0;   // 可写性
}
```

以上是对GPxDAT寄存器进行操作，我上面说的，还存在一个功能重叠的寄存器，那就是置位寄存器，我们的置位寄存器是怎么应用的呢？置位寄存器只能对该位进行清零操作，其他的数是没有办法的。还需要注意一点，就是置数寄存器对一个位进行写0操作，直接写0就好使。但是对于置位寄存器，它的功能是置位，如果使能该寄存器的功能就是写1，所以，对其写1操作就是对该位清0，一定要注意这个问题。

```c
GpioDataRegs.GPASET.bit.GPIOA0 = 1;     // 置数寄存器，置1
GpioDataRegs.GPACLEAR.bit.GPIOA0 = 1;   // 清零寄存器，置0；
```

最后说一个寄存器，我都没注意到，第二次看才看到，真是温故而知新，书读百遍其义自现啊。或许也应该注意这个问题，说不定某年某月看到这个了知道这个不一定会用上的小痞子，就获得成功了。其实每一步啊，即便是不起眼的东西，说不定有一天，它能成为一个问题的关键。不论是一个事儿，还是一个人！

**GPxTOGGLE****寄存器**，他的作用就是翻转电平。原来是高的变低，以前是低的现在变高。

> 声明一下：对于GPxDATA中的CLEAR功能，和GPXTOGGLE翻转电平的功能，如果我们写0的话，肯定是没用，一点儿作用都没有。所以对于他们，有意义的是，写1。我们要想起这些GPIO的寄存器，方便我们的设置。GPIO很重要的。

### 示例：

例子：GPIO引脚控制LED的闪烁。原理图没有，很简单，直接接到GPIO上面，我们也可以想象到，只是我想在这里面说说用法。

```C
#include <DSP28_Device.h>

void delay_loop(void);

void InitSysCtrl(void){
	Uint16 i;
	EALLOW;
	SysCtrlRegs.WDCR = 0x0068;
	SysCtrlRegs.PLLCR  = 0xA;
	for(i = 0; i <= 5000; i++){}
	SysCtrlRegs.HISPCP.all = 0x001;
	SysCtrlRegs.LOSPCP.all = 0x002;
	EDIS;
}

void InitGpio(void){

	EALLOW;
	GpioMuxRegs.GPFMUX.bit.XF_GPIOF14 = 0;
	GpioMuxRegs.GPFDIR.bit.GPIOF14 = 1;
}
void main(void){
	int kk = 0;
	InitSysCtrl();
	DINT;
	IER = 0x0000;
	IFR = 0x0000;
	InitPieCtrl();
	InitPieVectTable();
	InitGpio();
	while(1){
		GpioDataRegs.GPFCLEAR.bit.GPIOF14 = 1;
		for(kk = 0; kk < 100; kk++){
			delay_loop();
		}
		GpioDataRegs.GPFSET.bit.GPIOF14 = 1;
		for(kk = 0; kk < 100; kk++){
			delay_loop();
		}
	}
}
void delay_loop(void){
	short i ;
	for(i = 0; i < 30000; i++){ }
}
```
