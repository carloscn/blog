# DSP-F2812的事件管理器EV

2014年5月2日， 于沈阳化工大学 5号楼

明天就是五一了，本想在五一前夕把F2812有关的基础知识和原理总结完毕，没想到事儿太多，四月的最后一天还在整理事件管理器EV，进度差了，得抓紧时间了。时间管理器的页码很多，有关于它的资料也是很丰富的。第二次的学习，相信你与我一样，都有收获。

事件管理器，名字是事件管理，我们猜测就是跟发什么事儿有关，然后这个事儿有特定的管理机制进行管理。那些事儿是什么事儿呢？我们先放一放。首先呢，我先说个东西PWM波，这个东西就是个方波，但是高低电平的时间并不是一致的，是按照占空比来算的。这个东西有啥用呢？PWM波据我所知能进行的就是电机的控制，电力调速的，它的发现肯定节约能源。事件管理器的功能之一就是PWM波的产生，它可以有好几种方式产生PWM波呢。

我先说说EV要说些什么。首先本模块包括定时器（CPU也有，它也有），全比较/PWM单元，捕获单元，正交编码脉冲电路。共四大天王，组成EV家族。功能是相当强大。而最重要的EV的功能就是产生PWM波，四大天王各有各的功能，以自己的方式产生PWM波。

## 1. 事件管理器的功能

DSP有两个EV，EVA,EVB，两个EV除了名字不同之外其他的功能是完全一样的。所以我们只说EVA，然后EVB我们就可以把EVA的原理对称过去。

EVA包含2个16位的通用定时器，3个比较单元，3个捕获单元，1个正交编码电路。我分别解释下四大天王：

* 通用定时器：CPUTIMER0,1,2也同样是定时器，这个通用定时器和CPU定时器是两个东西，但是它俩的原理相似，区别在于CPU定时器为32位的，通用定时器是16位的，然后时钟不一样，工作方式也不一样，而且EV的通用定时器除了能计时之外，每一路还能产生单独的PWM波形。（高级了是吧！）
* 比较单元：他就是上面说的【全比较/PWM单元】，名字中携带PWM单元，看来是产生PWM的主力，每个比较单元可以产生一对（两路）互补的PWM波形。3个比较单元就可以产生6路。正好可以驱动一个三项全桥电路。（一会儿讲解）
* 捕获单元：功能就是捕获外部输入脉冲波形的的上升沿或者下降沿，可以统计脉冲的间隔，也可以统计脉冲的个数。通常用来对外部硬件信号的时间间隔进行测量，利用6个边沿检测单元测量外部的时间差，从而可以确定点击转子的转速。看来它的功能就是用来测量的。
* 正交编码电路：可以对输入的正交脉冲进行编码和计数，它与广电编码器项链可以获得旋转机械部位的位置，速率等信息，也用来电机控制。

看来四大天王各有千秋，通用定时器是用来计时而且产生PWM波也可以顶一顶。最多能产生4路。EVA两路，EVB两路。比较单元是PWM的主力，一个比较单元就能产生两路互补的波形，3个比较单元就是6路，再加上定时器的，一共可以产生16路。捕获单元捕获上升沿或者下降沿来捕获点点击的转速，是对PWM的测量。然而正交编码电路的也是测量功能，他的方式是对电机本身的，获取物理位置速率等信息。有可能是设置在反馈身上。

EV占用的管脚真的很多，四个通用定时器就能单独产生4路PWM波形，肯定就占用了4个引脚，比较单元更多了能产生12路呢，至少得有6个引脚啊。捕获单元同样6个，正交编码电路也得有6个。除此之外这只是功能引脚还得有其他的，包含一些，终止引脚。

![image-20221203101502703](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203101502703.png)

### 1)	External Cmpare-output Trip Inputs

![image-20221203101528421](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203101528421.png)

切断比较输出的外部控制输出。引脚已经在上面尚亮。C1 C2 C3分别控制比较单元的1,2,3。现在以PWM1,2为例，PWM1，2波形来自于比较单元1，如果PWM1，2正在输出互补的波形呢，我现在对C1TPIP写0。知识后PWM1，PWM2立即变为高阻状态，就不会有PWM波形的输出。我们可以联想到什么呢，如同JAVA里面的异常处理，如果捕获到异常，进入异常函数，我们不妨将这个功能写入异常处理的功能函数中，这样就不让PWM进行工作，使得电机也好其他设备不再工作，进行检修。

![image-20221203101543963](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203101543963.png) ![image-20221203101549978](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203101549978.png)

左面就是EVA的3个比较单元产生的PWM1----PWM6在DSP上面的引脚。如果我们写好程序，正常工作的话，将在这6个引脚产生PWM波形。

右面的是EVB的3个比较单元产生的PWM7----PWM12在DSP上面的引脚，我们不妨注意一下。和实际联合一下。

### 2)	External Timer-compare Trip Inputs

对应上个这个是切断定时器比较输出的外部控制输入。引脚如下：

![image-20221203101702341](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203101702341.png)

### 3)	External Trip Inputs

PDPINTA和PDPINTB其实是一个功率保护信号，它为系统的安全提供了保护。例如电机控制对的时候，有时候电路产生过电压，或者温度急剧上升的情况。此时如果PDPINTx的中断没有被屏蔽，当PDPINTx的引脚变为低电平，所有的PWM输出引脚都会立刻变为高阻状态，同时产生一个中断，从而阻止过高的电压，电流或者温度损坏电路，达到系统保护的目的。PDPINTA对应的是PWM1----PWM6,PDPINTB对应的是PWM7----PWM12。在使用的时候我们得驱动它，所以就要设计一个电路配备一个监视电路状态的信号。

下面呢，就该详细的说说四大天王是怎么工作的了。

### 2. EV工作原理

#### 2.1 通用定时器

通用定时器的个数我们已经很清楚了，EVA有两个EVB有两个分别是T1,T2，T3，T4。每个定时器呢，可以独立使用，也可以两两同步使用，T1和T2可以进行同步，T3和T4可以进行同步。定时器的任务主要有三个：第一，计时；第二，使用定时器的比较产生PWM波形；第三，给事件管理器的其他天王，提供基准时钟

![image-20221203101752456](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203101752456.png)

我们也看到了，定时器的核心就是比较逻辑，左有比较寄存器，右面是输出和中断的使能，上面是两个定时器的周期寄存器，下面是T1CNT定时器计数寄存器。既然叫比较逻辑，肯定是拿什么和什么进行对比，以上可以看出TCNT是计数功能，是来回跳变的，而TCMPR和TPR是比较寄存器和周期寄存器，这个值应该是固定的，所以比较应该是发生在T1CNT和TCMPR之间或者是T1CNT和T1PR之间的。前者是比较匹配事件，后者叫做周期匹配事件。如果这些匹配时间发生了的话，就会触发相应的中断或者产生PWM波形。我们这里直说原理，后面的工作方式我们慢慢的进行。

如果细心的人可以发现上面的图很多模块都有shadow的字样，阴影为何要写在上面？我们不得不百度一下shadow是干什么用的！

![image-20221203101811923](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203101811923.png)

> 阴影寄存器：值赋给影子寄存器，然后影子寄存器在赋给工作寄存器。工作寄存器里面的值可以随意改变，如果想恢复，就从影子寄存器中重载。影子寄存器帮助锁定初始值，实现了随时赋值，记忆初始值的功能。

你看到了，只有TxCMPR和TxPR存在影子寄存器，T1CNT是个变化的值，而作为参考的周期寄存器和比较寄存器存在影子寄存器来进行指定标准。那我们怎么控制从影子寄存器向工作寄存器射入值呢？答案就在T1CON寄存器中，控制寄存器。其实我们在DSP的世界上已经沐浴了很久了，似乎摸准他们的套路了，配合响应设置就要有控制寄存器的存在。影子寄存器也不例外，T1CON第3位和第2位TCLD1和TCLD0控制影子寄存器。

![image-20221203101845837](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203101845837.png)

实际应用中，我们很可能需要在程序执行的过程中不断改变这两个寄存器的数值，例如改变PWM波形的频率或者脉宽，所以其重载的条件我们是需要关注的。

#### 2.1.1 通用定时器的时钟

前面说了时钟的重要性了，这个外设也同样。通用定时器的工作离不开时钟，犹如人离不开心脏一样，而时钟的提供只能由系统SYSCLK提供。我们要弄清楚送SYSCLKOUT过来的时钟经过分频到底进入到通用定时器模块到底能分多少。系统的外部晶振是150MHZ的话，我们从下面的图计算一下！

![image-20221203101923506](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203101923506.png)

外部晶振OSCCLK经过PLL分频，再到高速预定标分频，再到T1CON.TPS分频，三分频进入到本模块。然后就知道了吧。

#### 2.1.2 通用定时器的计数模式

我们回忆一下定时器的计数模式，计数器载入周期寄存器的值然后根据时钟不断减1减到0后，发生中断事件。比起cpu定时器，我们的通用定时器比CPU定时器的方式丰富的多。有停止保持模式，连续增模式，连续增减和定向增减计数模式共4中计数模式。定时器到底工作于哪个模式取决于T1控制寄存器的第12位TMODE1和第11位TMODE0，具体某种模式我们看表。

| **TMODE1** | **TMODE0** | **描述**          |
| ---------- | ---------- | ----------------- |
| 0          | 0          | 停止/保持模式     |
| 0          | 1          | 连续增/减计数模式 |
| 1          | 0          | 连续增计数模式    |
| 1          | 1          | 定向增/减计数模式 |

我们这里只说两种常用的方式，如果选用其他的方式我们在查询就好。连续增减计数模式，连续递增计数模式最常用我们分析一下。

###### 1）连续递增/减计数模式

当TMODE = 1时候，定时器工作于连续增减计数模式。工作方式如下：定时器计数器寄存器T1CNT先从初始值开始递增至周期寄存器的值，再递减至0，然后从0开始递增至周期寄存器的值，接着再从周期寄存器的值递减至0，就这样不断循环下去。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221203125205.png)

###### 2)  连续增计数模式

当TMODE = 2时候，定时器工作于连续增模式。工作方式如下如图，我们就不难理解了。

![image-20221203102053095](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203102053095.png)

#### 2.1.2 通用定时器的中断事件

记得我们当初学习过的是CPU定时器的中断产生，源于CPU定时器计数器递减至0。而通用定时器的中断时间也不在少数。上溢中断，下溢中断，比较中断，周期中断。后两个不难理解，呼应上面的工作方式，当T1CNT的数值和T1PR或者T1CMPR的数值相同则产生中断。而上溢中断和下溢中断，我们要研究一下。

1） 上溢中断T1OFINT

当T1CNT的值为0xFFFF时，发生定时器T1的上溢中断。当上溢中断发生之后，再过一个定时器时钟周期，则上溢中断的标志位被置位。值得注意的是，只要T1CNT超过0xFFFF，就会产生上溢中断，假如刚开始赋值的时候超过了那个数，则产生了上溢中断。

2） 下溢中断T1UFINT

当T1CNT的值为0x0000时，发生定时器T1的下溢中断。下溢中断发生的时候下溢中断标志位被置位，原理如同上面的上溢中断。

3） 比较中断T1CMP

当T1CNT都和T1比较寄存器T1CMPR的值相同的时候，发生定时器T1的比较中断。当发生比较匹配之后，再过一个始终周期，则比较中断标志位就会被置位。

4） 周期中断T1PR

当T1CNT的值和T1周期寄存器T1PR相同的时候，发生定时器T1的周期中断。当发生周期中断之后，再过一个时钟周期，则周期中断的标志位就会被置位。当上面的中断标志位被置位后，如果该中断经被使能了，就会向PIE申请中断。一定要记住就是推出中断一定要通过程序手动清除中断标志位。以上在通用定时器中EVA中断标志寄存器EVAIFRA和EVAIFRB，EVA中断屏蔽寄存器是EVAIMRA和EVAIMRB。一个是中断使能寄存器，一个是中断标记寄存器，这个还是老套路了

5）ADSOC信号产生

![image-20221203102217276](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203102217276.png)

ADSOC信号虽然发出，但是到底启动ADC否，还得在ADC那边进行相应的设置。我们只是说ADC的启动，可以通过EV来进行。

#### 2.1.3 通用定时器的同步

现在有个问题，就是EVA的T1和T2如何同步进行，我们可以先开始T1CON的开始，然后再T2CON的开始，但是两句话，在执行语句的过程中时间已经发生了延迟，所以这个不是严格意义上的同步。所以DSP的设定早就考虑好这个问题了。我们可以通过设置，让T2忽略自身的周期寄存器并且使用T1的周期寄存器，也可以使用T1的使能位来启动T2计数器，这样就保证了同步。是不是很聪明啊！

怎么设定呢？

1. 将T2CON的T2SWT1位设成1，实现T1CON的TENABLE位来启动定时器T1和T2。这样就公用了一个开关。
2. 对T1CNT和T2CNT赋不一样的初始值。
3. 将T2CON的SELT1PR位设成1，指定定时器T2使用T1的周期寄存器，忽略自己的周期寄存器。

#### 2.1.4 **通用定时器的比较操作和PWM波**

现在我们知道了，每个定时器都有一个比较寄存器TxCMPR和一个PWM输出引脚TxPWM。通过给TxCNT的值和比较寄存器，周期寄存器值相同就会产生一个中断信号，或者产生一个ADC转换的启动信号。但是还有一个功能没有说，那么就是同样会产生一个PWM波形在TxPWM上。通用定时器的3个功能是需要设定TxCON来决定是选择哪个功能的。

我们选择T1CON中的产生PWM波形的模式。

1.	当T1CNT工作于连续增计数模式时，T1PWM_T1CMP引脚输出不对称的PWM波形。

```C
TMODE = 2; 
T1CON.bit.TECMPR = 1;
GPTCONA.bit.TCMPOE = 1;
GPTCONA.bit.T1PIN = 1; // 输出极性
```

![image-20221203102504614](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203102504614.png)

2. 当T1CNT工作于连续增减计数模式，T1PWM_T1CMP引脚输出对称的PWM波形。

```C
TMODE = 1;
T1CON.bit.TWCMPR = 1;
GPTCONA.bit.TCMPOE = 1;
GPTCONA.bit.T1PIN = 1;
```

![image-20221203102508518](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203102508518.png)

#### 2.1.5 **通用定时器的寄存器**

**只说明两个寄存器：**

1) GPTCONA通用定时器全局控制寄存器A

   上面说通用定时器有比较输出PWM，启动ADC信号，PWM输出的极性，都归它管理。负责T1和T2 EVA的嘛。

2) TxCON定时器控制寄存器

   定时器使能，仿真控制，计数模式选择，时钟选择（T1，T2同步），比较寄存器重载条件，比较使能，选择周期寄存器都归TxCON管理。

![image-20221203102553165](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203102553165.png)

## 3. 全比较单元与PWM电路

### 3.1 全比较单元

通用定时器产生PWM充其量是个业余的选手。专业的还得找全比较单元。EV中共有6个全比较单元，而且每个全比较单元都可以产生2路互补的PWM波形。在点击控制，开关电源，变频器，逆变器等电力电子电路中经常卡看到三相全桥控制电路。

![image-20221203102640023](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203102640023.png)

看到上面的Pha了吗？那个就是PWM波输入的地方，一共需要3对互补的波形进行驱动，刚好EVA或者EVB的驱动的6路互补的PWM可以进行对其驱动。右面的图是理想的情况，可实际上，在Q1，Q2导通关闭切换的过程中，会发生暂缓，总有那么一瞬间1和2的位置一起导通，或者一起关闭。这样是非常危险的。所以我们设定的时候在理想的情况下将PWM进行改进，表示称带有死区时间的PWM，也就是给Q1,Q2充足的时间进行跳变。DSP已经设计好了，我们只需要设定死区时间就行了。

![image-20221203102651639](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203102651639.png)

DSP提供了相应的死区寄存器。EVA和EVB都有能力驱动一个三相全桥电路。

全比较单元和通用定时器比较功能比较类似，也是设定相应的比较寄存器，然后进行比较产生某些事件。只不过这个寄存器的名称为CMPR1。

涉及的资源有：EVA的CMPR1，CMPR2，CMPR3，和EVB的CMPR5 ----CMPR6。比较控制寄存器COMCONA，COMCONB。一个行为控制寄存器ACTRA和ACTRB，每个比较单元对应的引脚PWM1，2，3，4，5，6,------12

EVA的比较单元所有的时基是由T1提供的，也就是在使用CMPR1产生的PWM波形时，用到的是T1PR和T1CNT，与T2没有关系。当定时器计数寄存器T1CNT和比较寄存器CMPR1相等的时候就发生了匹配事件。如果比较控制寄存器COMCONA的CENABLE为1，比较操作被使能，位FCMPOE为1，比较输出时各路PWM波形由比较逻辑驱动，同时行为控制寄存器ACTRA中的位CMP1和CMP2的极性为低电平或者高电平，就会产生两路互补的波形。

这个貌似很费劲啊，要是定CMPR1还有设定比较控制寄存器，还要设定行为控制寄存器。记住了吧。

>**说下个事情：死区！**
>
>死区的目的我们已经很清楚了。死区有个专门管理它的地方，叫做死区定时器控制寄存器,还有死区周期寄存器。死区时间为tbd = mx*p*xt。m是死区定时器周期m。死区定时器预定标因子为x/p，高速时钟HSPCLK的时钟周期为t。只要我们使能死区控制寄存器的开始位，则死区就生效了，在正常的PWM输出中就会掺入带有死区的信号，合成之后就成了带有死区的PWM，就可以进行工业上的应用了。

### 3.2 **比较单元的中断事件**

![image-20221203102750168](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203102750168.png)

我们只看上面的EVA的寄存器组，和他有关的比较重要的包括COMCONA比较控制寄存器，还有ACTRA比较行为控制寄存器，DBTCONA死区定时器控制寄存器。由于EVA中的比较单元有3个，所以比较寄存器也有三个。我们来解释解释这些寄存器。

**比较寄存器CMPR**：这个主要就是设定一些值T1CON与之比较的了。也就是说PWM的占空比主要由他来控制。10%的占空比的数为938。加10%的占空比就往CMPR中+938。

例子：

```C
EvaRegs.CMPR1 = 0x0000;
EvaRegs.CMPR2 = 0X0000;
EvaRegs.CMPR3 = 0x0000;
```

**比较控制寄存器COMCONA**：存在CENABLE位，比较操作的使能；CLD1~CLD0，CMPRx重载条件；FCMPOE全比较器输出使能；FCMP3OE，全比较器3使能，对应还有FCMP2OE,FCMP1OE。

例子：

```C
EvaRegs.COMCONA.bit.CENABLE = 1;
EvaRegs.COMCONA.bit.FCOMPOE = 1;
EvaRegs.COMCONA.bit.CLD = 2;
```

这是对其初始化的用到的，开始使能比较操作，然后全比较器的输出使能，然后选择CMPR1的重载条件为立即装入。**比较行为控制寄存器ACTRA**：主要就是控制哪个引脚输出什么极性的。不做过多解释。

`EvaRegs.ACTR.all = 0x0666;  // 设定动作属性；`

**死区定时器控制寄存器DBTCONA**：有EDBT3~死区定时器周期。然后还有EDBT1是控制PWM1和PWM2引脚的死区功能使能。EDBT2是设置PWM3，PWM4的使能。EDBT3是控制PWM5，PWM6的使能。是定时器就要存在预定标因子功能，占两位。

```C
EvaRegs.DBTCONA.bit.DBT = 10;  // 设定死区定时器的周期
EvaRegs.DBTCONA.bit.EDBT1 = 1;   // 死区遇定时器1使能，也就是让PWM1和PWM2输出带有死区。
EvaRegs.DBTCONA.bit.DBTPS = 4;   // 死区预定标因子。
```

### 3.2 捕获单元

捕获单元的作用我们前面已经说了，对于PWM波，它能检测到上升沿，下降沿，然后能计算他们之间的时间间隔，因此用于测速，测量。是基于信号的一种测量。捕获单元的原理非常简单，一面连着时钟，一面就是计数器，中间的开关由信号控制，信号如果满足条件，就会立即开启，然后计数器和时钟导通，就能计数。

![image-20221203102941195](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203102941195.png)

捕获单元可以检测出脉冲或者数信信号的宽度。当电机旋转的时候，当转轴转到某个特定的位置时，通过**光电码盘**，看下面的解释。

![image-20221203102954042](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203102954042.png)

F2812存在6个比较单元，6个捕获单元。EVA就占了3个捕获单元，分别是CAP1，CAP2，CAP3。每一个捕获单元都能一个捕获引脚。

![image-20221203103004194](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203103004194.png)

这是个例子，这是EVB的捕获单元。但是EVA名字差不多，都含有CAP字样。当输入引脚状态变化的时候，所选用的定时器的值就会被捕获并锁存到相应的2级FIFO堆栈中。啥意思呢，比如我上面使用的光电码盘的传感器，从上面接上传来的脉冲信号，高低高低的变换就被CAP单元捕获，在DSP上如果我们设置了，捕获低电平，和定时器配合起来就能捕获响应的数值然后存在FIFO栈里面。捕获单元同样有控制寄存器，状态寄存器，FIFO寄存器。我们呢一会儿说。

#### 3.2.1 捕获单元的结构

![image-20221203103024353](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203103024353.png)

EVA的每个捕获单元能够选择定时器1或者定时器2作为自己的时基，但是CAP1和CAP2必须选择相同的实际。而CAP3就可以进行随意选择。还应该注意的是CAP3的输入引脚检测到电平变换的时候，除了可以记录下定时器计数的值意外还能发出一个信号去启动ADC转换。注意FIFO只能存2个数，第3个数进来的时候第一个数就已经消失了。

#### 3.2.2 捕获单元的中断事件

捕获单元只有捕获中断。事件管理器每个捕获单元都对应一个捕获中断，例如EVA的捕获单元1有CAP1INT，捕获单元2有捕获中断CAP2INT….要发生中断必须满足两个事儿。1.捕获单元收到变化的信号。2.FIFO堆栈中有了有效数据。CAPxFIFO不为0。记住退出中断时候一定要清0.在EV中，和捕获单元的捕获中断相关的寄存器有：EVA中断标志寄存器EVAIFRC，EVA中断屏蔽寄存器EVAIMRC，EVB的中断标志寄存器EVBIFRC，EVB的中断屏蔽寄存器EVBIMRC。

#### 3.2.3 捕获单元的寄存器

![image-20221203103124296](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203103124296.png)

**捕获单元控制寄存器CAPCONA**：捕获单元复位，捕获单元1和2的使能，捕获3的使能，捕获单元的定时器选择。还有捕获3启动ADC信号，捕获单元1边缘检测控制位。都由他们控制。

**捕获单元FIFO**状态寄存器：CAP3FIOFO,CAP2FIFO,CAP1FIFO分别为他们的状态位。

### 4.正交编码电路

我们先不说这块，因为我们用不到。我们说说原理，上面的光电码盘，怎么确定他的位置呢？我么设定了两个传感器，在相位上相差90度，也就是他们是正交的。这样就能确定他们的位置了。我们用他们来检测位置的。这块我们不说了。我们有一天用到的时候再说吧。

### 5. 总结

总结一下：其实我们用这个就是为了产生PWM波形，大多数的功能都是这个。EV的模块我们结束了，怎么产生PWM波形呢，或多或少的有些了解了。比如说啊，我们利用定时器的比较功能产生输出4路的PWM波形，能够用专门的比较单元输出12路PWM波形。比较的原理都是定时器的T1CON与各自的比较寄存器CMPR进行比较，然后出现匹配。匹配之后可以通过控制寄存器，选择匹配后发生的时间，是唤醒中断啊，还是发出PWM波形啊，还是启动ADC啊。就这些功能。下面是手把手教你PWM波形的程序。

### 6. 示例

输出占空比可变的PWM波形，定义为1Khz，周期为1ms。1s占空比变10%。怎么实现呢。原理就是让比较单元输出PWM1和PWM2，然后利用定时器计时，然后到1ms就进入中断修改CMPR的值，这样就能实现可变了。这里的T1CON有两个作用，第一个就是和CMPR比较输出PWM波形，第二个作用就是和T1CMP比较触发中断，进入中断后修改CMPR的值，这样就可以达到值变化的目的了。这个例子非常的好！

#### 系统初始化

```C
#include <DSP28_Device.h>

void InitSysCtrl(void){
	Uint16 i;
	SysCtrlRegs.WDCR = 0x0068;
	SysCtrlRegs.PLLCR = 0xA;
	for(i = 0; i < 5000; i++){ }
	SysCtrlRegs.HISPCP.all = 0x0001;
	SysCtrlRegs.LOSPCP.all = 0x0002;
	SysCtrlRegs.PCLKCR.bit.EVAENCLK = 1;
}
```

#### 初始化GPIO

```C
#include <DSP28_Device.h>

void InitGpio(void){
	EALLOW;
	GpioMuxRegs.GPAMUX.bit.PWM1_GPIOA0 = 1;
	GpioMuxRegs.GPAMUX.bit.PWM2_GPIOA1 = 1;
	EDIS;
}
```
#### 初始化EV

```C
#include <DSP28_Device.h>

void InitEv(void){
	EvaRegs.T1CON.bit.TMODE = 1;
	EvaRegs.T1CON.bit.TPS = 1;
	EvaRegs.T1CON.bit.TENABLE = 0;
	EvaRegs.T1CON.bit.TCLKS10 = 0;
	EvaRegs.T1CON.bit.TECMPR = 1;
	EvaRegs.T1PR = 0x493E;
	EvaRegs.T1CNT = 0;
	EvaRegs.COMCONA.bit.CENABLE = 1;
	EvaRegs.COMCONA.bit.FCOMPOE = 1;
	EvaRegs.COMCONA.bit.CLD = 2;
	
	EvaRegs.DBTCONA.bit.DBT = 10;
	EvaRegs.DBTCONA.bit.EDBT1 = 1;
	EvaRegs.DBTCONA.bit.DBTPS = 4;
	EvaRegs.ACTR.all = 0x0666;
	EvaRegs.EVAIMRA.bit.T1PINT = 1;
	EvaRegs.EVAIFRA.bit.T1PINT = 1;
}
```

#### 主函数

```C
#include <DSP28_Device.h>

Uint32 intcount;
int increase;   
int decrease;   
void main(void){
	InitSysCtrl();
	DINT;
	IER = 0x0000;
	IFR = 0x0000;
	InitPieCtrl();
	InitPieVectTable();
	EALLOW;
	PieVectTable.T1PINT = & T1PINT_ISR;
	EDIS;
	InitGpio();
	InitEv();
	intcount = 0;
	increase = 0;
	decrease = 1;
	PieCtrl.PIEIER2.bit.INTx4 = 1;
	IER |= M_INT2;
	EINT;
	DRTM;
	EvaRegs.T1CON.bit.TENABLE = 1;
}
```

#### 中断

```C
interrupt void T1PINT_ISR(void){
	intcount++;
	if(intcount >= 1000){  
			if((increase == 1) && (decrease == 0)){		
EvaRegs.CMPR1 = EvaRegs.CMPR1 + 938;
			if(EvaRegs.CMPR1 >= 0x41eb){
				EvaRegs.CMPR1 = 0x41eb;
				increase = 0;
				decrease = 1;
			}
		}
		if((increase == 0)&&(decrease == 1)){
			EvaRegs.CMPR1 = EvaRegs.CMPR1 - 938;
			if(EvaRegs.CMPR1 <= 0x0753 ){
				EvaRegs.CMPR1 = 0x0753;
				increase = 1;
				decrease = 0;
			}
		}
		intcount = 0;

	}
	PieCtrl.PIEACK.all = 0x0001;
	EvaRegs.EVAIFRA.bit.T1PINT = 1;
	EINT;
}
```