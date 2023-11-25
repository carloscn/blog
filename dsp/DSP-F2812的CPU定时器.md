# DSP-F2812的CPU定时器

2014年4月25日，于 沈阳化工大学，5号楼

我一开始的时候把定时器和时钟弄混了，以为是一个东西，这个问题多么的幼稚啊。其实是自己学的太着急了，很多东西都一起看，结果，结果就学的洗了糊涂。学习本来就是一个熟悉的过程，需要时间来将它理解。太急了肯定不行，所以要放慢脚步，仔细读你当前的东西。也许就会发现，我学到的，是这么的多。

## 1. CPU定时器工作原理

> CPU-Timers 0, 1, and 2 are identical 32-bit timers with presettable periods and with 16-bit clock prescaling. The timers have a 32-bit count-down register, which generates an interrupt when the counter reaches zero. The counter is decremented at the CPU clock speed divided by the prescale value setting. When the counter reaches zero, it is automatically reloaded with a 32-bit period value. CPU-Timer 2 is reserved for the DSP/BIOS Real-Time OS, and is connected to INT14 of the CPU. If DSP/BIOS is not being used, CPU-Timer 2 is available for general use. CPU-Timer 1 is for general use and can be connected to INT13 of the CPU. CPU-Timer 0 is also for general use and is connected to the PIE block. 

![image-20221203095636814](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203095636814.png)

我们引入了英文，其实上面的问题应该很清晰了。但是我们应该看看下面的原理图。要设置几个寄存器才能进行正确的使用定时器。我们先仔细看看这个图，我们怎么看呢！先不看中间过程，先看输入是什么，输出又是什么。输入是一个复位信号，TimerReload组成的一组，然后CPU时钟信号和非TCR.4信号组成的与门。输出是INT，一个中断。那么我们就很明白了，我们需要定时器的目的就是产生一个中断信号。但是这个中断信号的产生需要什么呢？或者说，我们怎么利用这个中断信号呢？

从左上角开始，一个复位信号，或Timer Reload信号就能通过此门，Reset信号我们不难猜出，这个肯定可以寄存器设置，但是Timer Reload这个信号怎么产生？答案就在上面的那段英文，（**When the counter reaches zero, it is automatically reloaded with a 32-bit period value.**）当这个计数器到达0，他就能自动的重载…信号肯定就是这里发出的。现在我们再看，什么是counter，这个计数器是什么。是不是我们需要设定计数器的值就能像每天早上起床设闹钟一样，进行定时呢？我们查询counter有关资料

上面说的很明白了，TIMH:TIM就是counter的真身，它是加载在周期寄存器（PRDH:PRD）的值，counter以SYSCLKOUT的速率进行递减。当counter到0的时候一个中断信号就输出了。问题貌似还没有解决，资料给我提供的只告诉我们counter的值来自于周期寄存器，并非我们DIY进行设置的。所以，它不是和我们直接沟通的东西，只是个中间产物。我们顺藤摸瓜，将矛头指向周期寄存器，我们需要截取周期寄存器有关资料:

> CPU-Timer Period Registers (PRDH:PRD): The PRD register holds the low 16 bits of the 32-bit period. The PRDH register holds the high 16 bits of the 32-bit period. When the TIMH:TIM decrements to zero, the TIMH:TIM register is reloaded with the period value contained in the PRDH:PRD registers, at the start of the next timer input clock cycle (the output of the prescaler). The PRDH:PRD contents are also loaded into the TIMH:TIM when you set the timer reload bit (TRB) in the Timer Control Register (TCR).

我们找到了相关资料，当TIMH:TIM也就是counter递减到0的时候，counter就会从周期寄存器中加载周期值，在以下一个定时器输入时钟周期开始！又会在循环一次。无限循环。

那么我们从新整理一下。和我们玩家直接沟通的寄存器就是PRD寄存器了吧。我们往PRD里面写值，然后counter就会来读取，然后以CPU的时钟频率进行递减，减到0了，就产生了个中断，同时重载PRD寄存器里面的值。在下一个时钟信号开始时候，在进行一次。一直进行着，不断发送者中断信号。

![image-20221203095732773](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203095732773.png)

CPUTIMER0通过发送TINT0信号给PIE控制器，然后申请CPU的INT1到INT12的请求。看来定时器的中断权限还是比较高的。要学会恰到好处的利用CPU中断。CPUTIMER1和CPUTIMER2我们不说了。

> The PRDH:PRD contents are also loaded into the TIMH:TIM when you set the timer reload bit (TRB) in the Timer Control Register (TCR).

现在我们已经清晰了最外围的原理，那么还有两个块儿还没有弄清楚。如果我们足够细心的话，看到的主要干路还是放在SYSCLKOUT一条路上面，它的输入端是SYSCLKOUT和非TCR，现在看看TCR，如果我们将TCR写1，它的非就是0，如果是0则SYSCLOCK就进不来了。那么导致的问题就是根本就无法开始计数，不递减。

资料给的是通过TCR也可以重载counter的值。看到了吧。现在我们要清楚一个问题，到底COUNTER经过多长时间递减1呢，笼统的说是以CPU的时钟速率，但是要知道定时器模块有自己的分频器，可以将CPU的时钟进行分频。还有预定标计数器PSC。工作原理就是先给TDDR赋值，然后装载入PSC，每个一个SYSCLKOUT脉冲，PSC中的值减1.当PSC为0的时候，就会输出一个TIMCLK，然后带动COUNTER减1。然后PSC重新载入TDDR的值，周而复始的循环下去。
		
总结一下原理，这东西好比齿轮，一部分带动另一部分，一共有两组循环机制在运作。第一组就是控速组，TDDR的值装入PSC，PSC以CPU速率减，到0输出TIMCLK，然后counter的值减去1，counter的值是从PRD中载入的，counter减到0，发送中断信号。完成使命。

Counter走一步需要的时间为：

`t = (TDDR + 1) * 10^-6 / X;(x is diy);`

## 2. 定时器的寄存器

| 1.计数器低位     | TIMERxTIM  | 就是那个counter的数值；我们能设定的也就是第一次的数值，下一次要从PDR中读取。 |
| ---------------- | ---------- | ------------------------------------------------------------ |
| 2.计数器高位     | TIMERxTIMH | 以上的高位。                                                 |
| 3.周期寄存器低   | TIMERxPRD  | 设定counter的数值，无限载入。                                |
| 4.周期寄存器高   | TIMERxPRDH | 以上的高位。                                                 |
| 5.控制寄存器     | TIMERxTCR  | 有TIF,TIE,FREE,SOFT,TRB,TSS，TIF中断标志，TIE中断使能TRB是重载counter，TSS开始计时。 |
| 6.预定标计数器低 | TIMERxTPR  | PSC位和TDDR位                                                |
| 7.预定标计数器高 | TIMERxTPRH | PSCH位和TDDRH位                                              |

注释：TIF,TIE是有关于产生中断信号的位。我们要的就是产生中断信号，所以这个位还是特别说明下，TIF，TIE是自动置位的，当然可以软件置位。当COUNTER值变为0都是自动设置，TIF,TIE就会使能，产生中断。

配置函数:

```c
#include "DSP28_Device.h"
struct CPUTIMER_VARS CpuTimer0;
struct CPUTIMER_VARS CpuTimer1;
struct CPUTIMER_VARS CpuTimer2;
void InitCpuTimers(void){
	CpuTimer0.RegsAddr = &CpuTimer0Regs;
	CpuTimer0Regs.PRD.all = 0xFFFFFFFF;
	CpuTimer0Regs.TPR.all = 0;
	CpuTimer0Regs.TPRH.all = 0;
	CpuTimer0Regs.TCR.bit.TSS = 1;
	CpuTimer0Regs.TCR.bit.TRB = 1;
	CpuTimer0.InterruptCount = 0;
}
```

```c
void ConfigCpuTimer(struct CPUTIMER_VARS*TIMER,float Freq,float Peroid){
	
	Uint32 temp;
	Timer -> CPUFreqInMHz = Freq;
	Timer -> PeriodInUSec = Period;
	
	temp = (long)(Freq * Period);
	
	Timer -> RegsAddr -> PDR.all = temp;
	Timer -> RegsAddr -> TPR.all = 0;
	Timer -> RegsAddr -> TPRH.all = 0;

	Timer -> RegsAddr -> TCR.bit.TIF = 1;
	Timer -> RegsAddr -> TCR.bit.TSS = 1;
	Timer -> RegsAddr -> TCR.bit.TRB = 1;

	Timer -> RegsAddr -> TCR.bit.SOFT = 1;
	Timer -> RegsAddr -> TCR.bit.FREE = 1;
	Timer -> RegsAddr -> TCR.bit.TIE = 1;

	Timer -> InterruptCount = 0;

}
```

