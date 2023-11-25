# 0x22_LinuxKernel_内核活动（二）中断体系结构（中断上文）

基于ARM64体系结构，我们了解了在CPU上面关于异常处理的一些知识储备，并知道中断如何产生，并对中断一些术语进行了精准的校正。可以参考：

*   [10_ARMv8_异常处理（一） - 入口与返回、栈选择、异常向量表](https://github.com/carloscn/blog/issues/47)
*   [11_ARMv8_异常处理（二）- Legacy 中断处理](https://github.com/carloscn/blog/issues/48)
*   [12_ARMv8_异常处理（三）- GICv1/v2中断处理](https://github.com/carloscn/blog/issues/49)

**ARM64和LinuxKernel对中断有着不同的定义和理解**。从ARM64的角度来看，中断是异常的一种，分为同步异常和异步异常，而中断(FIQ/IRQ)属于异步异常中的一个成员（还包含了SERROR）。而在讲异常的时候，通常指示的都是同步异常，比如指令错误、SP/PC对齐检查/调试异常/MMU数据错误/SVC等等。从Linux内核的角度，是以中断划分，中断分为同步中断和异步中断，同步中断对应ARM64的同步异常概念，异步中断对应ARM64的中断概念。我们这篇既然在LinuxKernel的目录下，那么按照LinuxKernel定义的术语进行措辞使用。

系统在执行的时候可以分为两大区域：**内核态和用户态**。系统调用并非是用户态和内核态互相切换的唯一途径，中断也是一种方式。

**同步中断和异步中断有什么共性呢**？第一，当处于用户态的时候，CPU则发起用户态到到内核态的切换，接着在内核程序中执行ISR或者interrupt handler。第二，是否可以禁用中断。内核更倾向于避免禁用中断，因为会损害系统性能。仅少数的必要禁止中断的情境，是为了防止内核遇到一些严重麻烦。而且在允许内核禁用中断的情况下，若过多时间的执行ISR，那么可能就会miss掉很多重要的中断。**在这种情况下就引入了中断下半部**，即可以让一些不重要的中断事务延期执行。

我们把Linux内核体系结构总结为以下图片（**包含了中断上半部和中断下半部**）：

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typora%E4%B8%AD%E6%96%AD%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84.svg" width="100%" />
</div>

# 1. 中断处理

## 1.1 进入和退出任务

书接上文 [10_ARMv8_异常处理（一） - 入口与返回、栈选择、异常向量表 ](https://github.com/carloscn/blog/issues/47) 的 4.5.2 Linux Kernel，linux内核的entry.S文件中书写的异常向量表。我们的ARM64处理器做了一些工作：

>-   S1: PSTATE保存到SPSR_ELx
>-   S2: 返回地址保存到ELR_ELx
>-   S3: PSTATE寄存器里面的DAIF域都设定为1（等同于关闭调试异常、SError，IRQ FIQ中断）
>-   S4: 更新ESR_ELx寄存器（包含了同步异常发生的原因）
>-   S5: SP执行SP_ELx
>-   S6: 切换到对应的EL，接着跳转到异常向量表执行

对于操作系统也是有要求的：

>-   识别异常发生的类型
>-   跳转到合适的异常向量表（包含异常跳转函数）
>-   处理异常
>-   操作系统执行eret指令

Linux kernel把这些异常处理委托给一个handler，并且将每个中断和异常都编为唯一地址。内核会使用一个数组保存处理程序函数的指针。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typora0x21_01x.svg" width="77%" />
</div>

从中断的**执行**的角度来看，把中断处理函数之前的操作称为进入路径（entry path），把中断处理之后的操作称为退出路径（exit path）。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typora0x21_02.drawio.svg" width="100%" />
</div>

在进入路径阶段，中断进来之后一个关键的任务就是从用户态切入内核态，这个还不够，还需要把寄存器的信息备份到寄存器和内核栈里面，待退出路径的时候把信息恢复再切回用户态。值得注意的是退出的路径，我们通常说**中断上下文的切换，完成ISR处理之后就开始恢复现场了，这里并不是很贴切**。在恢复现场之前，还有调度器和信号的工作。

*   调度器会查询是否选择一个的新的进程来替换当前的进程。
*   是否有信号必须投递到原始的进程当中。

## 1.2 中断处理程序ISR

中断处理程序在编写上有一些点需要处理的十分谨慎，特别是需要考虑在执行中断函数期间有其他的中断请求进来，中断的嵌套某种程度可以使内核发生死锁。我们现有的方式masking中断，注意，并不是关闭中断，因为一些非常重要的中断不可以关闭。如果我们使用masking中断的方法，那么为了不错过一些中断，就要求持有屏蔽状态尽可能的短，与此同时，我们还要在中断处理函数中友好的支持其他中断的调度。因此在Linux内核中对**中断进行分类和调度策略支持**：

*   关键操作必须在中断发生之后立即执行，否则，无法维持系统的稳定性。
*   非关键操作也应该尽快执行，但允许启动中断。
*   可延期操作不是特别重要，不必在ISR内执行。内核可以延迟操作，在时间充裕时执行，如tasklet。

# 2. 中断数据结构表述
kernel内部中断程序的组成有两部分：A.**与处理器架构相关的汇编代码**；B.**设备的驱动程序（管理IRQ的数据结构和程序）**。内核更关注设备的驱动程序。

如下图所示，irq_desc结构体用于描述Linux整个系统中断：

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typorastruct%20irq_desc.svg" width="80%" /> </div>

https://elixir.bootlin.com/linux/v2.6.24/source/include/linux/irq.h#L152

```C
struct irq_desc {
	irq_flow_handler_t	handle_irq;
	struct irq_chip		*chip;
	struct msi_desc		*msi_desc;
	void			*handler_data;
	void			*chip_data;
	struct irqaction	*action;	/* IRQ action list */
	unsigned int		status;		/* IRQ status */

	unsigned int		depth;		/* nested irq disables */
	unsigned int		wake_depth;	/* nested wake enables */
	unsigned int		irq_count;	/* For detecting broken IRQs */
	unsigned int		irqs_unhandled;
	unsigned long		last_unhandled;	/* Aging timer for unhandled count */
	spinlock_t		lock;
#ifdef CONFIG_SMP
	cpumask_t		affinity;
	unsigned int		cpu;
#endif
#if defined(CONFIG_GENERIC_PENDING_IRQ) || defined(CONFIG_IRQBALANCE)
	cpumask_t		pending_mask;
#endif
#ifdef CONFIG_PROC_FS
	struct proc_dir_entry	*dir;
#endif
	const char		*name;
} ____cacheline_internodealigned_in_smp;
```
我们可以在 https://elixir.bootlin.com/linux/v2.6.24/source/kernel/irq/handle.c#L50 找到desc的定义，其是一个和体系结构没有任何关系的C语言结构体：
```c
struct irq_desc irq_desc[NR_IRQS] __cacheline_aligned_in_smp = {
	[0 ... NR_IRQS-1] = {
		.status = IRQ_DISABLED,
		.chip = &no_irq_chip,
		.handle_irq = handle_bad_irq,
		.depth = 1,
		.lock = __SPIN_LOCK_UNLOCKED(irq_desc->lock),
#ifdef CONFIG_SMP
		.affinity = CPU_MASK_ALL
#endif
	}
};
```

## NR_IRQS/STATUS/DEPTH/NAME
`NR_IRQS`: 根据定义，我们可以知道，体系结构中定义了多少中断就有多少`NR_IRQS`。这个值是由不同架构平台进行定义的，例如 https://elixir.bootlin.com/linux/v2.6.24/source/include/asm-arm/arch-davinci/irqs.h 在TI的davinci平台就会定义这个数值。

`status`: 用于描述SoC层irq的状态。

`name`: 表述中断的名字，/proc/interrupts 下面可以看到名字，如图所示[^1]

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220728102528.png" width="80%" /> </div>

1、逻辑中断号  
2、中断在各CPU发生的次数
3、中断所属设备类名称 
4、硬件中断号 
5、中断处理函数。

`depth`:  用于确定IRQ在SoC层级是启动还是禁止的，0表示启动，~0表示禁止。另外，这个值相当于一个计数器，内核其余部分代码禁用一次中断 +1， 开启一次中断 -1，等到depth回归0的时候才能开启这个中断。

关于`irq_chip`和`handle_irq`比较重要我们单独设立两个小节来说明。

## irq_chip

irq_chip被称为IRQ控制器抽象，结构体定义为 https://elixir.bootlin.com/linux/v2.6.24/source/include/linux/irq.h#L98 
 
```C
struct irq_chip {
	const char	*name;
	unsigned int	(*startup)(unsigned int irq);
	void		(*shutdown)(unsigned int irq);
	void		(*enable)(unsigned int irq);
	void		(*disable)(unsigned int irq);

	void		(*ack)(unsigned int irq);
	void		(*mask)(unsigned int irq);
	void		(*mask_ack)(unsigned int irq);
	void		(*unmask)(unsigned int irq);
	void		(*eoi)(unsigned int irq);

	void		(*end)(unsigned int irq);
	void		(*set_affinity)(unsigned int irq, cpumask_t dest);
	int		(*retrigger)(unsigned int irq);
	int		(*set_type)(unsigned int irq, unsigned int flow_type);
	int		(*set_wake)(unsigned int irq, unsigned int on);

	/* Currently used only by UML, might disappear one day.*/
#ifdef CONFIG_IRQ_RELEASE_METHOD
	void		(*release)(unsigned int irq, void *dev_id);
#endif
	/*
	 * For compatibility, ->typename is copied into ->name.
	 * Will disappear.
	 */
	const char	*typename;
};
```

start/disable/enable，顾名思义，这些回调函数充当一个IRQ的开始、启动、关闭；**ack与中断控制器硬件密切相关，IRQ请求达到之后必须有个显式的确认，后续请求才能处理进行**。

`eoi`表示end of interrupt，结束中断后的回调。因为现代处理器内部处理中断很丰富，不需要操作系统内核做太多的SoC控制。

`set_affinity`，在多处理器系统中，可以用这个函数指定特定CPU来处理IRQ，这使得IRQ分配给某个CPU（通常，SMP系统上的IRQ是平均发布到处理器上的）。

`set_type`设定IRQ电流的类型。该方法主要使用ARM、PowerPC、SuperH机器。这里包含`set_irq_type`便捷函数来设定:
* IRQ_TYPE_RISING
* IRQ_TYPE_FALLING
* IRQ_TYPE_EDGE_BOTH
* IRQ_TYPE_EDGE_LOW

对于irq_chip我们可以举几个例子：

[arm-irq-chip](https://elixir.bootlin.com/linux/v2.6.24/A/ident/irq_chip)

arm里面每一个涉及中断的模块里面都会定义`irq_chip`，如图这是检索顶一个irq_chip位置：

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220728121324.png" width="80%" /> </div>

随便找一个：https://elixir.bootlin.com/linux/v2.6.24/source/arch/arm/common/it8152.c#L77
```C
static void it8152_mask_irq(unsigned int irq)
{
       if (irq >= IT8152_LD_IRQ(0)) {
	       __raw_writel((__raw_readl(IT8152_INTC_LDCNIMR) |
			    (1 << (irq - IT8152_LD_IRQ(0)))),
			    IT8152_INTC_LDCNIMR);
       } else if (irq >= IT8152_LP_IRQ(0)) {
	       __raw_writel((__raw_readl(IT8152_INTC_LPCNIMR) |
			    (1 << (irq - IT8152_LP_IRQ(0)))),
			    IT8152_INTC_LPCNIMR);
       } else if (irq >= IT8152_PD_IRQ(0)) {
	       __raw_writel((__raw_readl(IT8152_INTC_PDCNIMR) |
			    (1 << (irq - IT8152_PD_IRQ(0)))),
			    IT8152_INTC_PDCNIMR);
       }
}

static void it8152_unmask_irq(unsigned int irq)
{
       if (irq >= IT8152_LD_IRQ(0)) {
	       __raw_writel((__raw_readl(IT8152_INTC_LDCNIMR) &
			     ~(1 << (irq - IT8152_LD_IRQ(0)))),
			    IT8152_INTC_LDCNIMR);
       } else if (irq >= IT8152_LP_IRQ(0)) {
	       __raw_writel((__raw_readl(IT8152_INTC_LPCNIMR) &
			     ~(1 << (irq - IT8152_LP_IRQ(0)))),
			    IT8152_INTC_LPCNIMR);
       } else if (irq >= IT8152_PD_IRQ(0)) {
	       __raw_writel((__raw_readl(IT8152_INTC_PDCNIMR) &
			     ~(1 << (irq - IT8152_PD_IRQ(0)))),
			    IT8152_INTC_PDCNIMR);
       }
}

static inline void it8152_irq(int irq)
{
	struct irq_desc *desc;

	desc = irq_desc + irq;
	desc_handle_irq(irq, desc);
}

static struct irq_chip it8152_irq_chip = {
	.name		= "it8152",
	.ack		= it8152_mask_irq,
	.mask		= it8152_mask_irq,
	.unmask		= it8152_unmask_irq,
};
```

## irq_action
action提供一个链式操作，需要中断发生的时候会从链表中找到相应的handler。`irq_action`表示如下：
```C
struct irqaction {
	irq_handler_t handler; /* （1） */
	unsigned long flags;   /* （2） */
	cpumask_t mask;
	const char *name; 
	void *dev_id;   /* (3) 唯一标识 */
	struct irqaction *next;   /* （4） */
	int irq;
	struct proc_dir_entry *dir;
};
```

我们来解释一下为什么irq_action需要一个链式操作？作为操作系统，我们期望的是IRQ能成为唯一标识符，这样就能很快的找到IRQ handler，但是现实很骨感，通常多个设备需要共享同一个IRQ，因为sharing的情况，所以导致了内核要进行二次查找（先找IRQ，在IRQ下面在找那个设备）。 Linux Kernel采用的是链式方法，next用于实现共享的IRQ处理程序。几个irqaction实例聚合到一个链表中，换句话说，链表的所有元素处理同一个IRQ编号（处理不同的编号在数组中的不同位置，参考1.1 图1）。此类处理程序链表大概能包含5个元素。

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora0x21_03.svg" width="100%" /> </div>

（3）-> `dev_id` 这是唯一标识符，内核查找到IRQ之后遍历链表的时候唯一的标识符。
（1）-> `handler`是一个指针函数`typedef irqreturn_t (*irq_handler_t)(int, void *);`。在中断控制器将请求转发到处理器的时候，内核将调用该处理程序。注意`irq_handler_t`和`irq_flow_handler_t`（在irq_desc->handle_irq）是两个不同的处理流程，名字很近，但是不要搞混。

# 3. 中断电流处理

我们来理解一下电流这个翻译，原文是*interrupt flow handling and chip-specific operations*，翻译给出的电流这个实在是难以表达背后的意思，我理解借鉴《数字电子技术》的*trigger*似乎能更表述这个意思，flow包含两类处理，一种是**电平触发**（高电平触发的、低电平触发的），一种是**边沿触发**（下降沿触发的、上升沿触发的），我认为或许*interrupt triggered handling*更贴一点。

我另外一点疑惑在于，为什么把triggered处理放在irq_desc这么高的位置封装，triggered处理本身就是中断的一种方式，应该是中断的处理其中的一个步骤。我猜测，triggered方式不一样，对于中断处理机制也不一样，甚至影响了中断处理的结构。所以linux kernel把triggered放在一个很高的位置。（废话碎碎念）

## 3.1 注册flow handler到irq_chip

Linux内核提供了接口，可以安全地把flow_handler注册到了irq_chip上面，并且将irq和flow_handler进行绑定。

https://elixir.bootlin.com/linux/v2.6.24/source/include/linux/irq.h#L353

```C
int set_irq_chip(unsigned int irq, struct irq_chip *chip);
void set_irq_handler(unsigned int irq, irq_flow_handler_t handle);
void set_irq_chained_handler(unsigned int irq, irq_flow_handler_t handle)
void set_irq_chip_and_handler(unsigned int irq, struct irq_chip *chip,
irq_flow_handler_t handle);
void set_irq_chip_and_handler_name(unsigned int irq, struct irq_chip *chip,
irq_flow_handler_t handle, const char
*name);
```

需要注意的是`set_irq_chained_handler`为某个给定的IRQ设置flow_handler的同时设定`irq_desc[irq]->status`标志位为IRQ_NOREQUEST和IRQ_NOPROBE，因为chained表示链式处理，则表示共享中断，共享中断不能独占使用，而IRQ_NOPROBE的选择是因为对于共享中断，采用PROBE，是一个不好的选择。

电流处理程序的回调函数原型如下面所示：

```C
typedef void fastcall (*irq_flow_handler_t)(unsigned int irq, struct irq_desc *desc);
```


## flow handler

不同的硬件可能存在不同的触发中断的方式，对于triggerd处理，Linux Kernel分为两类：
* 边沿触发（Edge-Triggered Interrupts）
* 电平触发（Level-triggered interrupts）

这两类差别就是触发条件和处理流程不一样，相同点在于工作结束之后负责调用高层的ISR（通过 handle_IRQ_event来激活高层IRQ）

为什么要分两个类型处理？这个和中断源有关系。试想，在电平触发的中断中，电平会是持续态，则中断一直处于激活状态，同一个中断会源源不断的被感知到，如果在多核的系统中，多个CPU都会远远不断的感知这个中断。而对于边沿触发，中断状态是一个顺态，会被中断控制器锁存，需要kernel向CPU发送ACK解除该中断的锁存状态。

### 边沿触发

在处理边沿触发的时候无需屏蔽（因为只有一次顺态）。我们要考虑SMP场景，顺态激活了中断之后，多个CPU会感知到中断，这就意味着一号CPU正在运行ISR handler的时候，二号CPU也可以运行这个ISR handler。但内核期望的是，只有一个CPU运行就够了，所以就使用设定标志位的方法，使用IRQ_INPROGRESS表示我当前正在运行这个ISR（通过`chip->ack()`设定），使用IRQ_PENDING标识表示还有一个ISR我将要处理。使用`mask_ack_irq`表示屏蔽该IRQ并且向控制器发送了ack（让控制器能在接收中断）

<div align='left'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220728145448.png" width="88%" /> </div>

### 电平触发

电平触发和边沿触发流程不一致，因为电平触发是一个持续态操作。电平触发进入处理的时候必须要屏蔽中断，等handle_IRQ_event处理完之后再去打开中断。这里也需要考虑SMP竞争情况，需要用IRQ_INPRROCESS表示中断已经被一个CPU core处理。

<div align='left'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220728145355.png" width="66.6%" /> </div>

### 其他类型

还有一些不常见的触发类型，内核为他们提供了一种插件式的方法，使用`chip->eoi`来完成，默认处理程序是：
* `handle_fasteoi_irq`：只需要极少的flowhandler处理
* `handle_simple_irq`：根本不需要flowhandler的处理
* `handle_percpu_irq` ： 发送到特定的cpu处理

# 4. IRQ处理

## 4.1 注册/注销IRQ
kernel/irq/manage.c: 给出了注册IRQ的定义：
```C
int request_irq(unsigned int irq, 
		irqreturn_t handler,
		unsigned long irqflags, 
		const char *devname, 
		void *dev_id)
```

下面是注册流程，相当于填充结构体。

<div align='left'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220728150302.png" width="66.6%" /> </div>

* IRQF_SAMPLE_RANDOM： 该中断对内核熵池有所贡献，作用于`/dev/random`
* `register_irq_proc`：在proc系统中/proc/irq/NUM 建立节点。

注销irq的时候，会调用`chip->shutdown`回调函数。

## 4.2 处理IRQ
* 切换到内核态
* IRQ栈初始化
* 调用flow handler
* 调用高层handle_IRQ_event
* 调用用户实现handler

### 4.2.1 切内核态

这部分代码的责任应该在SoC程序上面，并且需要使用汇编实现。`arch/arch/kernel/entry.S`找到该部分。切换到内核态最重要的是完成寄存器配置之后尽快的创建C语言环境（至少栈要初始化），然后切回C语言的逻辑中。在C语言调用函数时候，需要将所需的数据（返回地址和参数）按照一定的顺序放在栈空间上。在用户态和内核态切换的时候，还需要将重要的寄存器保存到当前的栈上（用户切内核的时候保存在内核栈，内核切用户栈时候，保存在用户栈），以便以后恢复。在大多数平台上，控制流接下来调用C函数的`do_IRQ`（AMD64/IA-32的叫法），在ARM架构上叫做`asm_do_IRQ`。

https://elixir.bootlin.com/linux/v2.6.24/source/arch/arm/kernel/irq.c#L111

`asmlinkage void __exception asm_do_IRQ(unsigned int irq, struct pt_regs *regs)`

* `pt_regs`：用于保存内核使用的寄存器信息
	<div align='left'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220728153138.png" width="40%" /> </div>
* `pt_regs`：定义在：https://elixir.bootlin.com/linux/v2.6.24/source/include/asm-arm/ptrace.h （这个是armv7的）
	```C
	struct pt_regs {
		long uregs[18];
	};
	
	#define ARM_cpsr	uregs[16]
	#define ARM_pc		uregs[15]
	#define ARM_lr		uregs[14]
	#define ARM_sp		uregs[13]
	#define ARM_ip		uregs[12]
	#define ARM_fp		uregs[11]
	#define ARM_r10		uregs[10]
	#define ARM_r9		uregs[9]
	#define ARM_r8		uregs[8]
	#define ARM_r7		uregs[7]
	#define ARM_r6		uregs[6]
	#define ARM_r5		uregs[5]
	#define ARM_r4		uregs[4]
	#define ARM_r3		uregs[3]
	#define ARM_r2		uregs[2]
	#define ARM_r1		uregs[1]
	#define ARM_r0		uregs[0]
	#define ARM_ORIG_r0	uregs[17]
	```

### IRQ栈（in kernel stack）[^2]

内核栈不同平台有着不同的处理方法，我们这里只讨论ARM32和ARM64的（IRQ栈）。

#### ARM32的IRQ栈

ARM32的IRQ栈很小，在setup.c文件中定义了这个栈，一个core只有12-bytes，然而IA-32架构中配置了CONFIG_4KSTACK的IRQ栈。这是因为架构不同导致的，ARM体系结构中断的处理需要进入到SVC特权模式，而IRQ只不过是一个中间介质状态。在进入IRQ函数中利用了`MSR`指令切换到了SVC模式，并且放弃了ARM处于的IRQ的模式。从中断返回也是从SVC模式返回，而非IRQ模式。简言之，ARM处理器发生中断的处理器状态(USR -> IRQ -> SVC -> USR)。

linux kernel arm32中定义的irq栈，其实就在一个static struct stack结构体变量中，大小为12bytes. irq_hander使用svc栈。

https://elixir.bootlin.com/linux/v2.6.24/source/arch/arm/kernel/setup.c#L99

```C
struct stack {
	u32 irq[3];
	u32 abt[3];
	u32 und[3];
} ____cacheline_aligned;

static struct stack stacks[NR_CPUS];
```

而SVC模式的堆栈是在CPU初始化的时候进行操作的，在cpu_init函数中： https://elixir.bootlin.com/linux/v2.6.24/source/arch/arm/kernel/setup.c#L387

```C
	/*
	 * setup stacks for re-entrant exception handlers
	 */
	__asm__ (
	"msr	cpsr_c, %1\n\t"
	"add	sp, %0, %2\n\t"
	"msr	cpsr_c, %3\n\t"
	"add	sp, %0, %4\n\t"
	"msr	cpsr_c, %5\n\t"
	"add	sp, %0, %6\n\t"
	"msr	cpsr_c, %7"
	    :
	    : "r" (stk),
	      "I" (PSR_F_BIT | PSR_I_BIT | IRQ_MODE),
	      "I" (offsetof(struct stack, irq[0])),  /*@ init irq stack*/
	      "I" (PSR_F_BIT | PSR_I_BIT | ABT_MODE),
	      "I" (offsetof(struct stack, abt[0])),
	      "I" (PSR_F_BIT | PSR_I_BIT | UND_MODE),
	      "I" (offsetof(struct stack, und[0])),
	      "I" (PSR_F_BIT | PSR_I_BIT | SVC_MODE)
	    : "r14");
```

#### ARM64的IRQ栈
ARMv8体系结构的中irq栈的地址和大小在 irq.h中定义：
```C
// irq.h
#define IRQ_STACK_SIZE			THREAD_SIZE
#define IRQ_STACK_START_SP		THREAD_START_SP

// thread_info.h
#define THREAD_SIZE		16384   
#define THREAD_START_SP		(THREAD_SIZE - 16)
```

以下是armv8的irq_handler的中断函数处理的过程：

```assembly
.macro	irq_handler
	ldr_l	x1, handle_arch_irq @ ------ (1)
	mov	x0, sp
	irq_stack_entry             @ ------ (2)
	blr	x1                  @ ------ (3)
	irq_stack_exit              @ ------ (4)
.endm
```

(1) 将handle地址保存在x1
(2)  切换栈，也就是将svc栈切换程irq栈. 在此之前，SP还是EL1_SP，在此函数中，将EL1_SP保存，再将IRQ栈的地址写入到SP寄存器
(3) 执行中断处理函数
(4) 恢复EL1_SP(svc栈)

对于irq_stack_entry：实际上就是找到SP指针的地址，然后赋值给EL1_SP，在内存"首地址"处，大小16k. irq_hander使用irq栈。

```assembly
	.macro	irq_stack_entry
	//将svc mode下的栈地址(也就是EL1_SP)保存到x19
	mov	x19, sp			// preserve the original sp    
	
	/*
	 * Compare sp with the base of the task stack.
	 * If the top ~(THREAD_SIZE - 1) bits match, we are on a task stack,
	 * and should switch to the irq stack.
	 */
#ifdef CONFIG_THREAD_INFO_IN_TASK
	ldr	x25, [tsk, TSK_STACK]
	eor	x25, x25, x19
	and	x25, x25, #~(THREAD_SIZE - 1)
	cbnz	x25, 9998f
#else
	and	x25, x19, #~(THREAD_SIZE - 1)
	cmp	x25, tsk
	b.ne	9998f
#endif
	adr_this_cpu x25, irq_stack, x26
	//IRQ_STACK_START_SP这是irq mode的栈地址
	mov	x26, #IRQ_STACK_START_SP    
	add	x26, x25, x26

	/* switch to the irq stack */
	//将irq栈地址，写入到sp
	mov	sp, x26     
	/*
	 * Add a dummy stack frame, this non-standard format is fixed up
	 * by unwind_frame()
	 */
	stp     x29, x19, [sp, #-16]!
	mov	x29, sp
9998:
	.endm
```

```assembly
/*
 * x19 should be preserved between irq_stack_entry and
 * irq_stack_exit.
 */
.macro	irq_stack_exit
//x19保存着svc mode下的栈，也就是EL1_SP
	mov	sp, x19     
.endm

```

### 调用flow handler
这部分也是根据架构不同有不一样的定义，对于ARM平台和AMD64的处理很像。

```C
asmlinkage void __exception asm_do_IRQ(unsigned int irq, struct pt_regs *regs)
{
	struct pt_regs *old_regs = set_irq_regs(regs);
	struct irq_desc *desc = irq_desc + irq;

	/*
	 * Some hardware gives randomly wrong interrupts.  Rather
	 * than crashing, do something sensible.
	 */
	if (irq >= NR_IRQS)
		desc = &bad_irq_desc;

	irq_enter();

	desc_handle_irq(irq, desc); /* flow handler：desc[irq]->handle_irq()） */

	/* AT91 specific workaround */
	irq_finish(irq);

	irq_exit();
	set_irq_regs(old_regs);
}
```

<div align='left'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220728163212.png" width="60%" />  <p align='center'><b>Note, AMD64流程和ARM32是一致的</b></p>
</div>

#### 高层ISR - handle_IRQ_event

高层ISR的逻辑是被flow handler函数调用的，当然这个包含了`handle_level_irq`，`handle_edge_irq`等等。在handle_IRQ_event中要遍历irq_action，执行用户实现的中断服务函数。

https://elixir.bootlin.com/linux/v2.6.24/source/kernel/irq/chip.c#L310

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora20220728170305.png" width="100%" />  <p align='center'><b>Note, handle_level_irq处理流程</b></p>
</div>

handle_IRQ_event定义在： https://elixir.bootlin.com/linux/v2.6.24/source/kernel/irq/handle.c#L129

```C
irqreturn_t handle_IRQ_event(unsigned int irq, struct irqaction *action)
{
	irqreturn_t ret, retval = IRQ_NONE;
	unsigned int status = 0;

	handle_dynamic_tick(action);

	if (!(action->flags & IRQF_DISABLED))
		local_irq_enable_in_hardirq(); // ------- （1）

	do {
		ret = action->handler(irq, action->dev_id); // ------ （2）
		if (ret == IRQ_HANDLED)
			status |= action->flags;
		retval |= ret;
		action = action->next;
	} while (action);

	if (status & IRQF_SAMPLE_RANDOM)
		add_interrupt_randomness(irq);              // ------- （3）
	local_irq_disable();          // --------- （4）

	return retval;
}
```

（1）如果之前的处理函数中没有设定`IRQF_DISABLED`，则用`local_irq_enable_in_hardirq` 启动当前CPU的中断。
（2）逐一调用所注册的IRQ处理程序的action函数。（这部分是用户实现的）
（3）如果对IRQ设定了RANDOM，将事件的时间作为熵池的一个熵源（如果中断的发生是随机的，那么他们是理想的熵源）**这个设计真的挺惊叹的**。
（4）`local_irq_disable`禁用中断，因为中断不支持嵌套的。

### user interrupt handler

#### 理论

最终用户实现的中断处理函数，都会被挂到irq_action的链表上。在用户实现的中断函数中，内核是对其有一定要求的，因为用户的一些程序是被内核执行流所执行，这部分配置和操作会陷入到内核之中。因此用户的中断处理函数应该十分小心，这些会极大的影响系统的性能和稳定性。

在实现用户ISR时，主要的问题是他们的所谓的中断上下文(interrupt context)中执行。内核程序有的时候在常规的上下文中执行，有的时候会在中断上下文中执行，这种情况，从用户角度来说是没有办法看见的。为了让用户能够感知到内核的执行状态`in_interrupt`系统调用，用于指明当前内核状态。

中断上下文和常规（进程）上下文有不同之处：
* 中断是异步执行的。在内核处于中断上下文期间，不可以访问用户空间，尤其是copy_to_user；常规上下文是允许访问用户空间的。
* 中断上下文不能调用调度器，因而不可以放弃控制权。
* 中断上下文，ISR程序内禁用睡眠。只有在外部事件导致状态改变唤醒的时候，才能解除睡眠。但中断上下文不允许中断，进程睡眠之后，内核只能永远等待下去。

Linux用户态实现中断响应，通过ISR函数原型定义 ：
`typedef irqreturn_t (*irq_handler_t)(int, void *);`

只能返回两个返回值：`IRQ_NONE`和`IRQ_HANDLED`

#### 示例1[^4]

我们在早些时候也做了一些IRQ申请的示例，参考：[# 基于OMAPL138的字符驱动_GPIO驱动AD9833(三)之中断申请IRQ](https://github.com/carloscn/blog/issues/37)

#### 示例 2 [^3]

这里引用一个例子<Linux内核中断引入用户空间（异步通知机制）>。 这个原理就是当SoC有一个中断透过驱动层使用异步信号机制让用户拿到消息。

1.在驱动中定义一个`static struct fasync_struct *async;`
2.在fasync系统调用中注册`fasync_helper(fd, filp, mode, &async);`
3.在中断服务程序（顶半部、底半部都可以）发出信号`kill_fasync(&async, SIGIO, POLL_IN);`
4.在用户应用程序中用signal注册一个响应SIGIO的回调函数`signal(SIGIO, sig_handler);`
5.通过`fcntl(fd, F_SETOWN, getpid())`将将进程pid传入内核
6.通过`fcntl(fd, F_SETFL, fcntl(fd, F_GETFL) | FASYNC)`设置异步通知

代码参考：https://gist.github.com/carloscn/e7d00a4831f93bade45757d2a509ebef


# 5. Reference 
[^1]:[csdn - /proc/interrupts](https://blog.csdn.net/fybon/article/details/78414794)
[^2]:[# linux kernel中的栈的介绍](https://blog.csdn.net/weixin_42135087/article/details/107931597)
[^3]:[# Linux内核中断引入用户空间（异步通知机制）](https://blog.csdn.net/kingdragonfly120/article/details/10858647)
[^4]:[# 基于OMAPL138的字符驱动_GPIO驱动AD9833(三)之中断申请IRQ](https://github.com/carloscn/blog/issues/37)