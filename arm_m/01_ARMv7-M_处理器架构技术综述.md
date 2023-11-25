# 1.  Overview

Cortex-M3（2006年）和Cortex-M4（2010）都是32位架构，寄存器和数据通路、总线接口都是32位。M系列使用指令集架构（ISA，Instruction Set Architecture）叫做Thumb ISA，它是基于Thumb-2 Technology（支持16位/32位指令）。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202305081034113.png)

M系列的处理器从**架构层面**有以下特点：
*  三级流水线设计；
* **哈佛总线架构**，且具有统一的存储器空间:指令和地址总线使用相同的地址空间；
* 32位寻址，支持4GB存储空间；
* 基于ARM **AMBA**（Advanced Microcontroller Bus Architecture）支持高吞吐的流水线操作。 关于流水线操作，可以参考：https://github.com/carloscn/blog/issues/62
* 名为**NVIC**（Nested Vectored Interrupt Controller，嵌套向量中断控制器）的中断控制器，支持最多240 个中断请求和8 ~256 个 中断优先级 ( 取 决 于 实 际 的 芯 片 设 计 )
* 支持多种OS特性，如节拍定时器以及影子栈指针等。
* 休眼模式和多种低功耗特性。
* 支持可选的**MPU( 存储器保护单元)**,提供了可编程存储器或访问权限控制等存储器 保护特性。
* 通过位段特性支持两个特定存储器区域中的位数据访问。
* 可以选择使用单个或多个处理器。

Cortex-M3 和Cortex-M4处理器提供了多种指令:
* 普通数据处理，包括硬件除法指令。
* 存储器访问指令,支持8位、16位、32位和**64位数据**，以及其他可以传输多个32位数据的指令。
* 位域处理指令。
* 乘累加 (MAC, Multiply Accumulate)以及饱和指令(saturate instructions )。
* 用于跳转、条件跳转以及两数调用的指令。
* 用于系统控制、支持0S等的指令。
* 单指令多数据(SIMD)操作。(M4 - only)
* 其他快速MAC 和乘法指令。(M4 - only)
* 饱和运算指令。
* 可选的浮点指令(单精度)。

一般来说，M系列属于RISC处理器，有些人可能会认为M3和M4的某些特性和CISC相近，入丰富的指令集和多种指令宽度。可能随着处理器技术的发展，RISC处理器指令同样也越来越复杂，因此RISC和CISC的处理器定义间越来越模糊。

M3和M4处理器有很多类似的地方，两个处理器指令一样，都属于ARMv7-M架构，而且NVIC和MPU等编程模型也相同。不过他们的内部设计不太同，例如M4在DSP应用方面具有更高的性能而且支持浮点运算。

M0和M0+以及M1基于ARMv6-M，他们指令集较小。M1专门为FPGA设计，具有TCM[^1]（Tightly Coupled Memory）有助于FPGA内的存储器使用。对于普通的I/O控制任务，由于M0和M0+有着低门数，他们具有良好的能耗效率（这个也是为什么M系列适合控制的原因），然而应用要进行一些复杂的数据处理，则可能需要花费更多的指令周期。

> @carloscn引申：这个其实也是为什么DSP在控制领域还会有市场，即便是ARM控制能力已经很强了。在控制行业，要求高频响较高，所以对于器件低的逻辑门数能够在控制频响层面提供较好的支持，能耗也低。DSP在运算上补足了类似于PID控制算法中涉及的运算，而且在正弦余弦乘法上有着得天独厚的优势。过强的控制能力并不是一个很好的选择，这会导致更低的频响和更高的能耗。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202305081034383.png)

# 2. MPU和MCU区别

在一个典型的微控制器MCU设计中，处理器只会占芯片中的一小块区域。其他部分则为存储器、时钟生成(如PLL)和分配逻辑、系统总线以及外设等(I/O接口单元、通信接口、定时器、ADC 、DAC 等硬件单元 )，如图所示：

<img src="https://raw.githubusercontent.com/carloscn/images/main/typora20230401131733.png" width="60%" align="center" />

而MPU属于MCU中的CPU的一部分。MCU产商会选择ARM的M系列处理器作为他们的CPU，然后他们根据自己的产品，增加存储器系统、存储映射、外设以及操作特性。

在一家公司获得Cortex-M处理器设计之后,ARM则会以Verilog-HDL语言的形式提供设计源代码。这些公司设计工程师会将外设和存储器等他们自己的HDL源代码添加进来，并使用各种EDA工具将整个设计从Verilog-HDL和其他多种形式转换为晶体管的层级芯片设计。

除此之外ARM还提供了其他的IP产品，有些可以被这些公司买走用于他们自己的产品中，例如：
* 逻辑门和存储器等单元设计（ARM物理IP等）
* 外设和AMBA基础部件（Cortex-M系统设计套件CMSDK和ARM CoreLink IP）
* 用于连接多个处理器设计中的调试系统（CoreSight IP）

ARM提供了一种名为Cortex-M系统设计套件(CMSDK)的产品,该设计套件包括 Cortex M处理器中的 AMBA基础部件、基本外设、示例系统以及示例软件。这样芯片设计者可以更快地了解 Cortex-M处理器，并且可重用的 IP 也降低了芯片开发的难度。

 当然，微控制器芯片设计者仍然有很多工作要做。所有的微控制器芯片公司都在努力开发更好的外设，并在自己的产品中加入独特的东西，比如苹果、高通，使用了ARM的架构，但是基于ARM，工作量依旧是有。

# 3. Going through Cortex-M

优势总结为：

* 低功耗优化，M控制器功耗低于200uA/MHz，还有的低于100uA/MHz。另外M处理器还支持休眠特性，可以同许多先进的超低功耗设计技术配合使用。
* 性能，M3和M4处理器性能可达到3 CoreMark/MHz、1.25 DMIPS/MHz，这样M3和M4就可以处理许多复杂的应用。
* 能耗效率。在有限的能量下，仍可以大量的处理工作。区别于低功耗，低功耗只的是进入低功耗的一种状态。而能耗效率是指低能源高效率工作。
* 代码密度。Thumb ISA提供良好的代码密度，意味着完成相同的工作，所需的程序代码更少。因此节约了FLASH存储空间。
* 中断。具有可配置的中断控制器设计，多达240个向量中断和多个中断优先级。中断嵌套由硬件自动处理，因此这给M处理器提供了跑实时操作系统的可能。
* 易于使用。M处理器具有简单的、线性存储器映射，它们比许多8位处理器还容易使用。架构上没有8位存储器所具有的限制（存储器分组，有限的栈空间，以及不可重入）几乎所有的代码可以用C实现，包括中断！
* 调试特性。支持单步调试，还可以生成捕获程序流，数据变动和跟踪数据。
* OS支持。M系列在设计支出就考虑了OS应用。
* 多种系统特性。M3和M4处理器支持多种系统特性，例如可位寻址存储器区域和MPU。
* 软件可移植性和可重用行。C的支持。

## 3.1 处理器架构

谈及处理器架构，比较重要的原语是，指令集架构（ISA），编程模型，调试方法，接口信号，流水线等。要了解ARMv7架构的细节，可以从ARM官方手册[^2]描述的角度来展开讨论：
* 指令集细节
* 编程模型
* 异常模型
* 存储模型
* 调试架构

## 3.2 指令集

Cortex-M 处理器使用的指令集名为 Thumb（其中包括16位的Thumb指令和更新的32位Thumb指令）。M3和M4处理器用到了Thumb-2技术，它允许16位和32位指令混合使用，以获取更高的代码密度和效率。

ARM7TDMI等经典的ARM处理器具有两种操作状态：32位的ARM状态和16位的Thumb状态。在ARM状态中，指令是32位的，内核能够以很高的性能执行所有支持的指令；而Thumb状态，指令是16位的，这样可以得到很好的代码密度，不过Thumb指令不具备ARM指令的所有功能。

要同时得到两者的优势， 许多用 于经典 ARM 处理器的应用程序混合使用了ARM 和Thumb 代码。不过这种混合编码的方式并不是非常理想 ，它会带来状态间切换的开销(执行时间和指令数  ) , 而且两个状态的分离还增加了软件编译过程的复杂度 , 对于不是很熟练的开发人员来说，优化代码更加困难。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20230401141752.png)

随着Thumb-2技术的引入，Thumb指令被扩展为16位和32位两种解码方式，现在，无需要在两个不同的状态切换就可以满足所有的处理要求。事实上，M处理器根本不支持32位的ARM指令。即使中断完全可以在Thumb状态处理，然而在经典的ARM处理器中，中断应该是在ARM状态。Thumb-2技术则提供了更多的可能性：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20230401142430.png)

* 无需状态开销，节约时间和指令空间；
* 无需制定源文件中ARM状态或者thumb状态；
* 代码密度和执行效率之间的平衡；

## 3.3 模块框图

![](https://raw.githubusercontent.com/carloscn/images/main/typora20230401144521.png)

## 3.4 存储系统

M3和M4处理器本身并不包含存储器 (没有程序存储器 、SRAM或缓存)，它们具有通用的片上总线接口，因此微控制器供应商可以将它们自己的存储器系统添加 到系统中。一般来说，微控制器供应商需要将下面的部件添加到存储器系统中。

* 程序存储器，一般是NVM （如FLASH，EMMC）
* 数据存储器，一般是SRAM
* 外设

M处理器总线接口位32位宽度，且基于高级微控制器总线架构AMBA。使用的总线协议AHB Lite。

## 3.5 中断和异常

Cortex-M3 和Cortex-M4处理器中存在一个名为嵌套向量中断控制器(NVIC)的中断控制器，它是可编程的且其奇存器经过了存储器映射。NVIC的地址固定，而且NVIC的编程模 型对于所有的Cortex-M 处理器都是一致的。除了外设和其他外部输人的中断外，NVIC 还支持多个系统异常。其中，包括不可屏蔽中断(NMI) 和处理器内部的其他异常源。

M3/4处理器是可以配置的，微控制器供应商能够决定NVIC设计实际支持的可编程中断数量。尽管NVIC的一些细节在不同的Cortex-M3/ M4处理器间可能存在差异， 中断/异常的处理和NVIC的异常模型却是相同的，它们定义在架构参考手册中。

## 3.6 MPU

PU 为Cortex-M3 和Cortex-M4处理器中的可选特性，微控制器供应商可以决定是否使用MPU。MPU为监控总线传输的可编程设备，需要通过软件(一般是嵌人式OS)配置。 若MPU 存在， 应用程序可以将存储器空间分为多个部分，并为每个部分定义访问权限。 当违反访问规则时，错误异常就会产生，错误异常处理则会分析问题，而且如果可能，将错误加以修复。

MPU可以有很多使用方式。一般情况，OS会设定MPU以保护OS内核和其他特权的任务使用数据，防止恶意用户的程序破坏。而且OS也可以选择不同用户任务使用的存储器隔离开来。这些有助于检测系统错误，并且提高了系统处理错误的健壮性。MPU 也可以将系统配置为只读的，防止意外擦除SRAM中的数据或覆盖指令代码。MPU 默认禁止，若应用不需要存储器保护特性，就无须将其初始化。

## 3.7 OS支持

**从定时器周期中断角度**：Cortex-M3和Cortex-M4处理器在设计时就考虑了对嵌人式OS的高效支持 。它们具有一个内置的系统节拍定时器SysTick。可以为OS定时提供周期性定时中断。

**从stack指针角度**：OS内核和中断的主栈指针（MSP）以及应用任务用的进程栈指针（PSP）。这样，OS内核用的栈就和应用任务分开。可靠性的提升，栈空间也得到了优化。如果没有OS的话可以只使用MSP就好了。

**从执行状态角度**：M3/4支持独立的特权和非特权模式，处理器在启动后默认处于特权模式。当使用OS且执行用户任务时，用户任务可以在非特权模式下执行。特权模式和非特权模式可以和MPU一起使用，防止非特权任务访问某些存储器。这样用户就无法破坏OS内核和其他任务数据。

**异常角度**：提供了错误处理的机制。当检测到一个错误的时候，异常就会被触发。


# 4. ARM 软件开发

微控制器中有很多部分，在许多微控制器中，处理器占的硅片面积小于10%，剩余部分被其他部件占用。对于一个MCU的基础硬件包含：

* FLASH （NVM存储器）
* SRAM 
* 外设
* 内部总线
* 时钟生成逻辑
* 电压调节和电源控制
* 其他模拟器件（ADC/DAC）
* I/O部分

## 4.1 软件开发流程

### 4.1.1 软件开发流程

![](https://raw.githubusercontent.com/carloscn/images/main/typora20230401135129.png)

### 4.1.2 编译流程

嵌人式程序的编译取决于你所使用的开发工具，

![](https://raw.githubusercontent.com/carloscn/images/main/typora20230401135227.png)

在使用GNU的gcc工具链时，一般可以一次性地编译整个应用程序，而不是将编译和链 接阶段拆开。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20230401135503.png)

## 4.2 软件模式

### 4.2.1 轮询（Polling）

![](https://raw.githubusercontent.com/carloscn/images/main/typora20230401135808.png)

### 4.2.2 中断驱动（Interrupt driven）

轮询的另外一个缺点在于能耗效率差，在不需要服务时也会浪费很多能量。为了解诀这
个问题，几乎所有的微控制器都会提供某种休眠模式以降低功耗，在休眠模式下, 外设在需要 服务时可以将处理器唤醒。

在中断驱动的应用中，不同外设的中断可以被指定为不同的中桥优先级。例如，重要/ 关 键的外设可以被指定为较高的优先级，这样, 若中断产生时处理器正在处理更低优先级的中断，低优先级中断就会被暂停，而更高优先级的中断服务就会立即执行。这种设计的响应较快 。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20230401140022.png)

### 4.2.3 多任务系统 （Multi-tasking systems）

当应用更加复杂时，轮询和中断驱动的程序架构末必能够满足处理需求。例如，有些执行 时间长的任务可能会需要同步处理。要实现这一操作，可以将处理器时间划分为多个时间片并且将时间片分给这些任务。在这些应用中，实时操作系统(RTOS)可用于处理任务调度：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20230401140143.png)

## 4.3 C语言和CMSIS库

编程语言支持多种“标准”数据类型，不过数据在硬件中的表示方式要取决于处理器架 构 和C编译器 。 

![](https://raw.githubusercontent.com/carloscn/images/main/typora20230401140237.png)

CMSIS由ARM开发，它使得微控制器和软件供应商可以使用一致的软件结构来开发Cortex 微控制器的软件，许多Cortex-M 微控制器的软件产品都是符合CMSIS的。CMSIS的组织结构：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20230401140506.png)

工程layout如图：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20230401140527.png)





# Reference
[^1]:[Cortex-M0 Devices Generic User Guide Version 1.0 - Tightly-Coupled-Memory](https://developer.arm.com/documentation/den0042/a/Tightly-Coupled-Memory)
[^2]:[Arm Cortex-M4 Processor Technical Reference Manual Revision r0p1](https://developer.arm.com/documentation/100166/0001/Functional-Description?lang=en) 

