# DSP-F2812的模数转换器ADC

2014年5月3日，于沈阳化工大学，5号楼

ADC是很重要的外设，今天我们得重视，这个东西嘛。本身还是很好学的，只是我们要有耐心就好了。会了基础，剩下的就是做工程的时候灵活应用，基础是死的，可人的思维是活得。我们充分利用好每个功能哦。
	
工程，总得感受外界的东西。只有感受到外界的东西，我们才能进行处理，数字化是种进步，当我们发现了某种规律和数学有着各种各样的联系才能为我们造福。数字信号处理，数字电子技术中，都存在着一种模数转换，数模转换的概念。我们已经很清晰了，外界的信号是模拟量，也就是连续的信号，声音，温度等。在数学上这些模拟量如果转化到数字的世界，他们之间的间距是无穷小的，根本就无法运算。可是奈奎斯特发现了采样定理，使我们进入了一个数字化的世界。采样定理就是我们采样的频率不能少于二倍的信号本身的固有频率，这样我们可以轻易的从模拟转换成数字，又从数字还原到模拟。我们能做的，就是利用各种数学关系去计算中间的数字化的信号。这个过程就是数字信号处理。而现在，我们要通过DSP的ADC功能将外部的模拟信号采集进来，然后再利用我们的数学算法进行信号处理，然后再数模转换输出应用于世界的模拟信号，达到预期的效果。今天本节就是说一说模数转换器ADC。怎么外来世界采入信号到DSP呢？我们一步步的说。

## 1. X2812x内部的ADC模块

X2812x内部的ADC模块是一个12位分辨率，具有流水线结构的模数转换器。（这里面有两个概念，第一个就是12位分辨率，和流水线结构）。

>有关于分辨率的参考文献：
举个例子，假设一个模拟模块的输入电压范围是0~10V，而该模块的分辨率为10位，2的10次方=1024，则该模块所能识别的最小电压等级为：10/1024=0.009765V，即电压每增加0.009765V，数字量增加1。所以位数越大精度越高，运算量就越大，运算时间越长，根据自己的需要选位数，不是位数越多越好。
>
>模拟量输入模块有两个参数容易混淆：
>	1.模拟量转换的分辨率 
>	2.模拟量转换的精度（误差） 
>
>分辨率是A/D模拟量转换芯片的转换精度，即用多少位的数值来表示模拟量。S7-200模拟量模块的转换分辨率是12位，能够反映模拟量变化的最小单位是满量程的1/4096。模拟量转换的精度除了取决于A/D转换的分辨率，还受到转换芯片的外围电路的影响。在实际应用中，输入的模拟量信号会有波动、噪声和干扰内部模拟电路也会产生噪声、漂移，这些都会对转换的最后精度造成影响。这些因素造成的误差要大于A/D芯片的转换误差。（百度空间）

> 流水线结构：
> 系统在处理数据的时候，一个指令周期含有4～6个时钟脉冲，每个脉冲周期由不同的部件完成不同的操作。非流水线结构是指一个指令周期完成以后再接受下一条处理数据的指令；而流水线结构，每个时钟脉冲都接受下一条处理数据的指令，只是不同的部件做不同的事情，就象生产线流水操作一样，并不是等一个或一批产品做完，再接受下一批生产命令，而是每个工序完成以后，立即接受下一批生产任务。这样提高了系统处理数据的速度
>
> ![image-20221203103743651](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203103743651.png)

明白了分辨率和流水线结构。我们就来说说这个ADC模块。

![image-20221203103753988](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203103753988.png)

最左侧的是ADCINA0…ADCINA7，和ADCINB0…ADCINB7，共16路采样，看来这个就是个引脚了，也就是要采什么，就把什么东西放在这个引脚上。我们看看在DSP上这几个引脚都在哪里！

![image-20221203103808377](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203103808377.png)

再往右是个Analog Mux模拟多路复用器。他的作用很简单，就是这不有8根儿线连接在Mux上吗？ADC只能接受1路信号，他的作用就是，当有多个引脚发送采样的时候，只对1路进行数据转换，然后进入到ADC中。下面就涉及一个问题了。ADC转换的顺序为何？

我们说了AM的作用就是只转换1路的数据，但是这个数据的顺序怎么规定的呢？假设我只用0,1,2,3引脚进行AD采样，我想的顺序是1,0,3,2也就是1引脚的数据先转换，然后存到结果寄存器0里面，然后是0引脚的数据进行采样，然后把数据存到结果寄存器1里面……如何设置呢？当然ADC有个机构叫做序列发生器，它的作用就是预先设定先对哪路进行数据转换。我们可以通过设定序列发生器的顺序对AM进行设置。

S/H的作用我们可以理解为是量化。数据存到结果寄存器，我们就对结果寄存器里面的数值进行应用了。

### 1.1 **ADC**模块的特点

1. 16个模拟输入引脚分为了2组，A组合B组命名为 ADCINA0 ADCINB3等。

2. ADC的时钟频率最高可为25Mhz，采样频率最高位12.5MSPS。(MSPS-转换速率)，也就是每秒完成12.5百万次的采样。

3. ADC模块的序列发生器，可以按照2个独立的8为状态SEQ1和SEQ2来运行，也可以按一个16位的序列发生器来运行。这个通过寄存器可以设定。

4. ADC的模拟输入的范围为0~3V，为了保险起见ADC端口前要接入钳位电路。
5. ADC模块对一个序列的通道开始转换必须有个启动信号，或者说是一个出发信号。SEQ的方式组成不同，启ADC的方式也是不同的。

![image-20221203103857370](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203103857370.png)

| 序列发生器 | 启动方式                                                     |
| ---------- | ------------------------------------------------------------ |
| 单独SEQ1   | 1.软件立即启动 2.EVA的多种事件 3.外部引脚(GPIO/XINT2_ADCSOC) |
| 单独SEQ2   | 1.软件立即启动 2.EVB的多种事件                               |
| 级联SEQ    | 1.软件立即启动 2.EVA的多种事件 3.外部引脚(GPIO/XINT2_ADCSOC) |

​     这里的立即启动ADCTRL控制寄存器发出的启动信号。ADC可以多方启动，我们也看到了。

5. ADC模块共有16个结果寄存器，ADCRESULT0-----ADCRESULT15用来保存采样的记过。但是数值只有12位，放在高12位上。最高是65520。

### 1.2 ADC采样方式

#### 级联模式下的顺序采样方式

AD的采样方式有很多，我们这里呢使用最常用的方式，我们要非常熟练的利用这样的方式足以了。我们现在用了级联模式了，也就是SEQ1和SEQ2级联成一个16位的SEQ。采用的方式是顺序采样，所以必须对16个通道中的每一个通道都进行排序，SEQ将用到通道选择控制寄存器ADCCHSELSEQ1,2,3,4.

![image-20221203104007218](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203104007218.png)

代码如下：

```C
AdcRegs.ADCTRL1.bit.SEQ_CASC = 1; // 选择级联模式
	AdcRegs.ADCTRL3.bit.SMODE_SEL = 0; // 选择顺序采样模式
	AdcRegs.MAX_CONV.all = 0x000f;  // 序列发生器最大采样通道数16个，一次采样一个通道，总共是16个通道
	// 采样ADCINA0通道赋予状态指针，指向0x00
	AdcRegs.CHSELSEQ1.bit.CONV00 = 0x00; 
	AdcRegs.CHSELSEQ1.bit.CONV01 = 0x01; 
	AdcRegs.CHSELSEQ1.bit.CONV02 = 0x02;
	AdcRegs.CHSELSEQ1.bit.CONV03 = 0x03;
	// 采样ADCINA4通道赋予状态指针，指向0x04
	AdcRegs.CHSELSEQ2.bit.CONV04 = 0x04;
	AdcRegs.CHSELSEQ2.bit.CONV05 = 0x05;
	AdcRegs.CHSELSEQ2.bit.CONV06 = 0x06;
	AdcRegs.CHSELSEQ2.bit.CONV07 = 0x07;
	// 采样ADCINB0通道赋予状态指针，指向0x08
	AdcRegs.CHSELSEQ3.bit.CONV08 = 0x08;
	AdcRegs.CHSELSEQ3.bit.CONV09 = 0x09;
	AdcRegs.CHSELSEQ3.bit.CONV10 = 0x0a;
	AdcRegs.CHSELSEQ3.bit.CONV11 = 0x0b;
	// 采样ADCINB4通道赋予状态指针，指向0x0c
	AdcRegs.CHSELSEQ4.bit.CONV12 = 0x0c;
	AdcRegs.CHSELSEQ4.bit.CONV13 = 0x0d;
	AdcRegs.CHSELSEQ4.bit.CONV14 = 0x0e;
	AdcRegs.CHSELSEQ4.bit.CONV15 = 0x0f;

```

如果序列发生器完成了转换，就会自动存入结果寄存器。

##### ADC模块的中断

当序列发生器完成一个序列的转换时，就会产生对该序列发生器的中断标志位进行置位，如果这个中断已经使能了，则ADC模块会向PIE发送请求。当ADC工作于双序列发生器模式的时候，序列发生器SEQ1和SEQ2可以分单独设置中断标志位和中断使能位。也级联模式的时候自然你懂得。

中断的方式有两种，第一种是发生在每个序列转换结束的时候。另一种是出现在每隔一个序列转换结束的时候。（第一次不产生，第二次产生，第三次不产生，第四次产生），可以通过ADCTRL2的中断方式控制。

#### **寄存器**

ADC控制寄存器1：ADCTRL1：有复位，仿真，设定采集窗口大小，内核时钟标记因子，继续运行，序列发生器忽略，级联序列发生器工作方式的控制。

ADC控制寄存器2：ADCTRL2：EVB转换使能，复位SEQ1，启动SEQ1，SEQ1中断使能，SEQ1中断方式。和对应SEQ2的相关设置。

ADC控制寄存器3：ADCTRL3：ADC电源控制，时钟分频器，和采样方式选择。

最大转换通道寄存器：MAXCONV位

这些就是主要的了。我们看程序。



## 2.**程序相关：**

现在假设ADC模块工作于级联模式，利用并发采样的方式进行。关于启动方式我们采用EV管理器启动的方式

1. 先初始化时钟，系统，看门狗

2. 初始化ADC模块，设定ADC的采样相关方式，决定通道数，顺序等

3. 初始化EV，利用通用定时器T1开始计时，然后设定T1CMPR等待周期中断，然后选择启动ADC。

初始化时钟：

```C
void InitSysCtrl(void){

	Uint16 i;
	EALLOW;
	SysCtrlRegs.WDCR = 0x0068;
	SysCtrlRegs.PLLCR = 0x0A;
	for(i = 0; i <= 5000; i++){
	}
	SysCtrlRegs.HISPCP.all = 0x0001;
	SysCtrlRegs.LOSPCP.all = 0x0002;
	SysCtrlRegs.PCLKCR.bit.EVAENCLK = 1;
	SysCtrlRegs.PCLKCR.bit.ADCENCLK = 1;

}
```

初始化ADC模块：

```C
void InitAdc(void){
	
	Uint16 i;
	AdcRegs.ADCTRL1.bit.RESET = 1;
	NOP;
	AdcRegs.ADCTRL1.bit.RESET = 0;
	AdcRegs.ADCTRL1.bit.SEQ_CASC = 1; // SEQ1和SEQ2级联成16位的SEQ
	AdcRegs.ADCTRL1.bit.SUSMOD = 3; // 仿真暂停
	AdcRegs.ADCTRL1.bit.CONT_RUN = 0; // 运行于启动/停止模式
	AdcRegs.ADCTRL1.bit.CPS = 0; // 预定时钟标记因子
	AdcRegs.ADCTRL3.bit.ADCBGRFDN = 3; //参考电源控制
	for(i = 0;i < 10000;i++){NOP;}
	AdcRegs.ADCTRL3.bit.ADCPWDN = 1;
	for(i = 0;i < 5000;i++){NOP;}
	AdcRegs.ADCTRL3.bit.ADCCLKPS = 15; // ADCLK = HSPCLK /30
	AdcRegs.ADCTRL3.bit.SMODE_SEL = 1; // 并发采样方式
	AdcRegs.MAX_CONV.bit.MAX_CONV = 0x07;  //  8个并发采样，共采样16路
	// 采样ADCINA0通道赋予状态指针，指向0x00
	AdcRegs.CHSELSEQ1.bit.CONV00 = 0x00; 
	AdcRegs.CHSELSEQ1.bit.CONV01 = 0x01; 
	AdcRegs.CHSELSEQ1.bit.CONV02 = 0x02;
	AdcRegs.CHSELSEQ1.bit.CONV03 = 0x03;
	// 采样ADCINA4通道赋予状态指针，指向0x04
	AdcRegs.CHSELSEQ2.bit.CONV04 = 0x04;
	AdcRegs.CHSELSEQ2.bit.CONV05 = 0x05;
	AdcRegs.CHSELSEQ2.bit.CONV06 = 0x06;
	AdcRegs.CHSELSEQ2.bit.CONV07 = 0x07;
	
	AdcRegs.ADC_ST_FLAG.bit.INT_SEQ1_CLR = 1;
	AdcRegs.ADC_ST_FLAG.bit.INT_SEQ2_CLR = 1;
	
	AdcRegs.ADCTRL2.all = 0x0828;
	
}
```

初始化EV：

```C
void InitEv(void){
	EvaRegs.T1CON.all = 0x4013;
	EvaRegs.GPTCONA.bit.T1TOADC = 2;  // 周期中断启动ADC
	EvaRegs.EVAIMRA.bit.T1PINT = 1;
	EvaRegs.EVAIFRA.bit.T1PINT = 1;
	EvaRegs.T1PR = 0x0EA5;
	EvaRegs.T1CNT = 0;
}
```

Main函数：

```C
float adc[16],adclo;
interrupt void ADCINT_ISR(void);
void InitADC(void);

void main(void) {
	Uint16 i;
	InitSysCtrl();
	DINT;
	IER = 0x0000;
	IER = 0x0000;
	InitPieCtrl();
	//InitPieVectTable();
	EALLOW;
	PieVectTable.ADCINT = &ADCINT_ISR;
	adclo = 0; 
	PieCtrl.PIEIER1.bit.INTx6 = 1;
	IER |= M_INT1;
	EINT;
	ERTM;
	EvaRegs.T1CON.bit.TENABLE = 1;
	for(i = 0; i < 15; i++ ){
		adc[i] = 0;
	}
	while(1){

	}

}
interrupt void ADCINT_ISR(void){
	
	adc[0] = ((float)AdcRegs.RESULT0) * 3.0/65520.0 + adclo;
	adc[1] = ((float)AdcRegs.RESULT1) * 3.0 / 65520.0 + adclo;
	adc[2] = ((float)AdcRegs.RESULT2) * 3.0 / 65520.0 + adclo;
	adc[3] = ((float)AdcRegs.RESULT3) * 3.0 / 65520.0 + adclo;
	adc[4] = ((float)AdcRegs.RESULT4) * 3.0 / 65520.0 + adclo;
	adc[5] = ((float)AdcRegs.RESULT5) * 3.0 / 65520.0 + adclo;
	adc[6] = ((float)AdcRegs.RESULT6) * 3.0 / 65520.0 + adclo;
	adc[7] = ((float)AdcRegs.RESULT7) * 3.0 / 65520.0 + adclo;
	adc[8] = ((float)AdcRegs.RESULT8) * 3.0 / 65520.0 + adclo;
	adc[9] = ((float)AdcRegs.RESULT9) * 3.0 / 65520.0 + adclo;
	adc[10] = ((float)AdcRegs.RESULT10) * 3.0 / 65520.0 + adclo;
	adc[11] = ((float)AdcRegs.RESULT11) * 3.0 / 65520.0 + adclo;
	adc[12] = ((float)AdcRegs.RESULT12) * 3.0 / 65520.0 + adclo;
	adc[13] = ((float)AdcRegs.RESULT13) * 3.0 / 65520.0 + adclo;
	adc[14] = ((float)AdcRegs.RESULT14) * 3.0 / 65520.0 + adclo;
	adc[15] = ((float)AdcRegs.RESULT15) * 3.0 / 65520.0 + adclo;

	PieCtrl.PIEACK.all  = 0x0001;
	AdcRegs.ADC_ST_FLAG.bit.INT_SEQ1_CLR = 1;
	AdcRegs.ADCTRL2.bit.RST_SEQ1 = 1;
	EINT;
}
```

