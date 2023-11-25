# 09_OPTEE-OS_内核之（一）ARM核安全态和非安全态的切换

我们根据[02_ARMv8_基本概念](https://github.com/carloscn/blog/issues/1) 和 [10_ARMv8_异常处理（一） - 入口与返回、栈选择、异常向量表](https://github.com/carloscn/blog/issues/47) 中了解到ARMv8有EL0 - EL3四个异常等级，有NS和S两种状态。ARMv8对TZ技术有先天性支持的条件。而对于ARMv7架构，要想使用安全feature就需要做一些手脚。

# 1. ARMv7基础知识（复习）
ARMv7架构的ARM核为支持TrustZone技术，**在ARM核原有七种运行模式的基础上扩展除了Monitor模式**，正常世界状态（NWS）与安全世界状态（SWS）之间的切换就是由**运行于Monitor模式下的程序来完成的**，为方便理解在ARMv7架构中正常世界状态与安全世界状态之间的切换，本节将介绍一些基础知识。

## 1.1 ARMv7运行模式扩展
在未支持TrustZone技术之前，ARM具有**七种运行模式**，分别为：
* **usr模式（用户模式）**：正常程序运行时的模式；
* **fiq模式（快速中断模式）**：当配置有快速中断时，如果产生fiq事件，ARM核将会切换到该模式；
* **irq模式（用户模式）**：中断模式，一般用于通用中断处理，被ROS使用；
* **svc模式（管理模式）**：操作系统使用的保护模式；
* **sys模式（系统模式）**：运行具有特权的操作系统任务；
* **abt模式（数据访问终止模式）**：当数据或者指令预取值时终止则会进入该模式；
* **und模式（未定义指令模式）**：当未定义指令执行时则会进入该模式。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221005201532.png" width="80%" /></div>

这些模式都是一种硬件反应态，AXI总线强烈依赖访问时的模式。

**支持TrustZone技术后，ARM增加了Monitor模式，Monitor模式起到进行安全世界状态与正常世界状态之间切换的桥梁作用**。所以在ARMv7架构的ARM核中具有八种类型的运行模式和两种状态，每种状态下具有自己独立的七种模式，Monitor模式是共享的。

## 1.2 ARMv7安全状态位扩展
在支持TrustZone技术时，ARM在AXI系统总线上增加了一个安全状态位（NS bit）（详细情况可查阅ARM给出的TrustZone白皮书），而安全状态位就是用来标识当前的数据、指令是属于安全世界状态还是正常世界状态，**安全状态位会被保存到scr寄存器的第0位**。当安全状态位等于1时，处理器处于正常世界状态；当安全状态位等于0时，处理器处于安全世界状态。

除了对总线进行扩展之外，ARM对MMU和Cache也同样进行了安全状态位的扩展，用于标记MMU中存放的物理内存映射后的地址是属于安全内存地址还是非安全地址，而对于Cache该位会被用来标记当前的Cache是属于安全态的Cache还是非安全态的Cache。当ARM核访问物理地址时，会对该虚拟地址的安全状态位进行检查，而在访问物理内存时安全扩展组件会对地址进行权限检查，该权限检查操作属于硬件级别的检查，不受软件的控制。关于安全地址的配置则是在IC设计时通过配置安全组件的参数来设定的。

## 1.3 ARMv7安全相关的重要寄存器
执行两个世界之间的切换操作会使用到各种寄存器的操作，这些寄存器的作用说明如下。
* 异常向量基地址寄存器（VBAR）
* Monitor模式的异常向量基地址寄存器 (MVBAR)
* 安全配置寄存器 (SCR)
* 栈指针寄存器 (SP)
* 当前程序状态寄存器 (CPSR)
* 程序保存状态寄存器 (SPSR)
* 链接寄存器 (LR)

### 1.3.1 异常向量基地址寄存器（VBAR）
异常向量基地址寄存器（Vector Base Address Register，VBAR）将保存异常向量表的基地址，在安全世界状态和正常世界状态都具有各自独有的VBAR寄存器用于存放两种状态各自独有的异常向量表的基地址。

异常向量安全世界有一份，正常世界也有一份。

### 1.3.2 Monitor模式的异常向量基地址寄存器
Monitor模式的异常向量基地址寄存器（Monitor Vector Base Address Register，MVBAR）用于保存在Monitor模式下异常向量表的基地址，该寄存器在安全世界状态和正常世界状态之间进行切换时起到关键作用。

### 1.3.3 安全配置寄存器
处理器在运行时，安全配置寄存器（Secure Configuration Register，SCR）中会保存相关的标志，其中用于标记处理器处于安全世界状态还是正常世界状态的安全状态位（NS bit）就被保存在该寄存器中。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221005154918.png)

值得关注的是NS位，用于指示异常是由NS world进来的，还是Secure world进来的。0代表安全，1代表非安全。
![](https://raw.githubusercontent.com/carloscn/images/main/typora20221005154941.png)

### 1.3.4 栈指针寄存器
栈指针寄存器（Stack Pointer，SP）用来存放处理器使用的栈的偏移地址。

### 1.3.5 当前程序状态寄存器
当前程序状态寄存器（Current Program Status Register，CPSR）将保存处理器运行时的各种标志位信息，包括标志域、状态域、扩展域和控制域。

### 1.3.6 程序保存状态寄存器
当特定的异常中断发生时，程序保存状态寄存器（Saved Program Status Register，SPSR）将保存当前程序的cpsr寄存器中的内容，待异常中断退出之后，处理器会使用spsr寄存器中的数据来恢复cpsr寄存器中的数据。

### 1.3.7 链接寄存器
链接寄存器（Link Register，LR）一般用来保存子程序的返回地址。

## 1.4 Monitor指令
注意进入成功之后，这些寄存器都是自动被load和维护的。通过在程序中执行smc汇编指令可以让处理器进入Monitor模式。如果该汇编指令执行成功，**则处理器就切换到了Monitor模式下，并且更新Monitor模式下的重要寄存器，包括CPSR、SPSR、LR、SCR等**。该操作与ARM进入到IRQ、ABT等模式的操作一样，采取的是产生异常来进行模式的切换。当处理器进入到Monitor后，处理器就会去查询该模式下的异常处理向量表的位置，而Monitor模式下具有独立的异常向量表的基地址，该地址被保存在MVBAR寄存器中。在ARMv8架构同样也是使用smc指令切换到EL3阶段。

## 1.5 Monitor下处理过程

在安全世界状态或者正常世界状态中执行smc指令之后，处理器将会触发异常操作进入Monitor模式，并从MVBAR寄存器中获取到Monitor模式的异常中断向量表基地址，进而找到安全监控模式调用操作的异常处理函数。在[11_OPTEE-OS_内核之（三）中断与异常的处理](https://github.com/carloscn/blog/issues/101)中将详细介绍Monitor模式的异常中断向量表基地址是如何保存到MVBAR寄存器中的，此处不赘述。Monitor模式下整个处理逻辑：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221005155347.png)

在OP-TEE中，Monitor模式的异常中断向量表定义在optee_os/core/arch/arm/sm/sm_a32.S文件中，其内容如下：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221005153346.png)

当系统调用smc指令后，处理器将切换到Monitor模式，查找到异常中断向量表，并执行b sm_smc_entry指令来对安全监控模式调用进行处理。该函数定义在optee_os/core/arch/arm/sm/sm_a32.S文件中，其完整内容如下：

```assembly
LOCAL_FUNC sm_smc_entry , :
UNWIND( .fnstart)
UNWIND( .cantunwind)
	//将当前模式的lr和spsr寄存器中的值分别存储在monitor模式的sp中
    srsdb   sp!, #CPSR_MODE_MON    
    push    {r0-r7}         //将r0到r7中的值压入栈(sp)    
    clrex        //独占清除,可以将关系紧密的独占访问监控器返回为开放模式    
    read_scr r1                //获取当前scr寄存器中的值,并将值保存在r1寄存器中    
    //判定scr寄存器中值的NS位是否为1,如果是1,则将会改变CPSR中的条件标志位为0    
    tst r1, #SCR_NS    
    bne .smc_from_nsec  //如果请求来自于Normal World,则跳转到smc_from_nsec进    
    //将当前处于SWS中,Secure World的运行栈存放的运行栈存放在r0中    
    //所以将当前sp的值减去offset就可以得到Secure World的运行栈地址    
    //并将sp的值指向得到的Secure World的运行栈地址    
    sub sp, sp, #(SM_CTX_SEC + SM_SEC_CTX_R0)    //将sp的值加上secure world context的长度保存在r0寄存器中    
    add r0, sp, #SM_CTX_SEC    //保存secure world中八种模式的主要寄存器的值,并将值存放到r0寄存器    
    //而r0寄存器已经指向了CPU栈的位置中以便实现secure context的保存    
    bl  sm_save_modes_regs    //将sp的值加上secure world context中r0存放的位置    
    add r8, sp, #(SM_CTX_SEC + SM_SEC_CTX_R0)    
    ldm r8, {r0-r4}        //将r8寄存器中的值指向地址中的值依次赋给r0到r4    
    //  将FIQ指向完的值保存到r9寄存器中    
    mov_imm r9, TEESMC_OPTEED_RETURN_FIQ_DONE    
    cmp r0, r9                //对比r0寄存器和r9寄存器中的值    
    //如果r0与r9不相等则将sp加上non-secure context中的r0的值保存到r8寄存器中    
    addne   r8, sp, #(SM_CTX_NSEC + SM_NSEC_CTX_R0)    
    //如果r0与r9不相等则将r1到r4寄存器中的值依次加载到r8指定的位置    
    stmne   r8, {r1-r4}    //将sp的值加上non-secure world context的长度保存到r0寄存器中    
    add r0, sp, #SM_CTX_NSEC    
    bl  sm_restore_modes_regs //获取non-secure context的内容//执行返回到Normal World的操作.sm_ret_to_nsec:    
    //将sp的值加上normal world context中从起始位置到r8寄存器的偏移值    
    //然后将结果保存到r0寄存器中    
    add     r0, sp, #(SM_CTX_NSEC + SM_NSEC_CTX_R8)    
    ldm r0, {r8-r12}        //r0寄存器中值指向的地址中的值一次次赋给r8和r12寄存    
    read_scr r0 //获取当前scr寄存器的值,并保存到r0寄存器中    
    orr r0, r0, #(SCR_NS | SCR_FIQ)        //将scr中的NS位和FIQ位置1    
    write_scr r0        //将修改后的r0的值写入scr寄存器    
    //将sp的值加上non-secure world context中从起始位置到r0寄存器的偏移值    
    //然后将结果保存到sp中    
    add sp, sp, #(SM_CTX_NSEC + SM_NSEC_CTX_R0)    
    b   .sm_exit        //跳转到sm_exit函数继续执行//指向切换到Secure World的操作.smc_from_nsec:
    //当前处于Normal world态,栈指针就是
    sp    //所以将当前sp的值减去offset就可以得到Normal World的运行栈地址    
    //并将sp的值指向得到的Normal World的运行栈地址    
    sub sp, sp, #(SM_CTX_NSEC + SM_NSEC_CTX_R0)    
    bic r1, r1, #(SCR_NS | SCR_FIQ)        //清除r1寄存器中的NS位和FIQ位    
    write_scr r1        //将r1寄存器中的值写入scr寄存器    
    //将sp的值加上non-secure world context中r8存放的值    
    //然后将结果保存到r0寄存器中    
    add r0, sp, #(SM_CTX_NSEC + SM_NSEC_CTX_R8)    
    stm r0, {r8-r12}        //将r8到r12寄存器中的值保存到r0指向的地址位置    
    mov r0, sp        //将sp的值赋值给r0寄存器    
    //跳转到secure world中进行处理来自non-secure world的smc请求    
    bl  sm_from_nsec    cmp r0, #0        //对比返回值是否为零,即判断sm_form_nsec函数是否执行成功    
    beq .sm_ret_to_nsec        //如果执行成功,则执行返回到non-secure world的操    
    //如果sm_from_nsec函数并未执行成功,    //则将sp的值加上secure world context中r8存放的位置    
    //然后将结果保存到sp中    
    add sp, sp, #(SM_CTX_SEC + SM_SEC_CTX_R0) 
    //执行退出sm操作.sm_exit:    
    pop {r0-r7}  //将栈中的r0到r7寄存器中的值进行出栈操作    
    rfefd   sp! //使用sp寄存器中的数据执行返回操作
UNWIND( .fnend) 
END_FUNC sm_smc_entry
```

### 1.5.1 正常世界触发Monitor处理
根据上图，判断SCR.ns为1则代表这个call是来来自于正常世界。当在正常世界状态（NWS）触发安全监控模式调用时，SCR寄存器中的安全状态位（NS bit）必定为1，处理器进入Monitor模式后，异常向量表中的sm_smc_entry处理函数会执行smc_from_nsec的分支，正式进入对来自正常世界状态的安全监控模式调用进行具体处理。整个执行过程的流程图：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221005155548.png" width="80%" /></div>

在整个处理过程中，当SCR寄存器中的安全状态位被设定后，即表示处理器的状态已经处于安全世界状态或者是正常世界状态。判定该安全监控模式调用来自于正常世界状态后将会执行到sm_smc_entry函数中的smc_from_nsec代码块。

在该代码段中有重要的两条语句，将从sm_smc_entry开始获取到的scr值保存到r1寄存器中，并清i空r1寄存器中的安全状态位和FIQ位来完成设定处理器状态和使能FIQ，然后再将r1寄存器重新载入到scr寄存器中来完成正常世界状态到安全世界状态的切换。

当正常世界状态中的安全监控模式调用被OPTEE处理完毕后，处理器将调用sm_ret_to_nsec函数重新回到正常世界状态。从安全世界状态切换到正常世界状态是通过读取当前scr寄存器的值到r0寄存器，将r0寄存器中的值的安全状态位和FIQ位设置成1来实现将处理器切换回正常世界状态和屏蔽FIQ的功能。再通过write_scr函数将修改后的r0寄存器的值重新载入到scr寄存器中。

### 1.5.2 正常世界触发Monitor处理
当安全监控模式调用是在安全世界状态中触发时，SCR寄存器中的安全状态位必定为0，处理器会执行smc_ret_to_nsec分支，正式进入对来自正常世界状态的安全监控模式调用的处理过程。整个执行过程的流程图：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221005160032.png)

在上述过程中，从安全世界状态切换到正常世界状态的方法也是通过修改SCR寄存器中的安全状态位来实现的。在执行切换之前需要保存安全世界状态的上下文信息，并将当前处理器的上下文信息恢复成正常世界状态的上下文信息。待正常世界状态上下文信息恢复之后，再修改SCR寄存器的安全状态位来实现切换。保存安全世界状态的上下文信息和恢复正常世界状态的上下文信息的操作分别通过执行sm_save_modes_regs和sm_restore_modes_regs函数来实现。

# 2. ARMv8基础知识（复习）

ARMv7需要自身处理很多程序逻辑。ARMv8使用ATF来完成正常世界状态与安全世界状态之间切换的过程。ARMv8的切换过程与ARMv7大致一样，也是使用smc汇编指令来触发切换动作，关于切换的软件则需要运行在EL3中，且该部分的**具体切换过程是在ATF中的bl31中实现**的。

我们根据[02_ARMv8_基本概念](https://github.com/carloscn/blog/issues/1) 和 [10_ARMv8_异常处理（一） - 入口与返回、栈选择、异常向量表](https://github.com/carloscn/blog/issues/47) 中了解到ARMv8有EL0 - EL3四个异常等级，有NS和S两种状态。

ARMv8中关于总线、MMU、Cache以及其他安全组件的扩展与ARMv7中的完全一样。相关扩展功能的说明可参见[02_OPTEE-OS_基础之（二）TrustZone和ATF功能综述、简要介绍](https://github.com/carloscn/blog/issues/92)

## 2.1 Monitor模式指令
ARMv8中smc指令的作用与ARMv7中完全一样，在ARMv8中，smc指令用来产生目标为EL3的异常（异常类型为：Synchronous），只有EL1或更高的特权等级才能调用smc指令。任何需要交给OPTEE OS完成的任务都需要首先发送相应的安全监控模式调用，ATF再根据该调用的来源、ID号等来决定交给OP-TEE OS中相应的处理函数。触发安全监控模式调用的语法为：

`smc  #imm16             /* imm4 会被处理器忽略,一般设置为#0 */`

安全监控模式调用可以分为SMC32调用规范（参数采用32位寄存器）和SMC64调用规范（参数采用64位寄存器）。这种模式独立于AArch32和AArch64模式。在AArch32模式下，ATF规定只能使用SMC32规范；在AArch64模式下，可以同时使用SMC32/64两种调用规范。该规范的说明如表所示。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221005162630.png" width="90%" /></div>

两种规范中参数的位数不同，但ID号都是使用的32位。为避免不同安全监控模式调用定义的冲突和混乱，ATF通过定义安全监控模式调用格式中不同域的含义来决定安全监控模式调用的类型、服务范围等（参考SMC Calling Convention PDD）

由图可知，在SMC32规范中，针对TEE OS的快速安全监控模式调用（fast smc）的SMC ID范围为0xB2000000～0xBF00FFFF；针对TEE OS的标准安全监控模式调用（std smc）的SMC ID范围为0x02000000～0x1FFFFFFF。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221005163948.png" width="90%" /></div>

## 2.2 EL3处理过程
为了简化不同ARMv8平台Trusted OS的移植，ARM提供了在EL3运行的代码示例，称为ARM Trusted Firmware（以下简称为ATF）。采用的BSD许可证，因此目前各个TEE厂商都是在ATF基础上做相应的定制。在ATF里面提供了各种相应的接口标准，包含：

* Power State Coordination Interface（PSCI）：用于CPU电源管理。·
* Trusted Board Boot Requirement：描述可信任的系统启动/加载镜像的流程。
* SMC Calling Convention：定义Secure Monitor Call的请求格式。

### 2.2.1 ATF的EL3异常向量注册

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221005165638.png)

我们在： [02_Embedded_ARMv8 ATF Secure Boot Flow (BL1/BL2/BL31)](https://github.com/carloscn/blog/issues/65#top) 的 “4. SoC BL31 Booting”中介绍：在ATF的bl31启动过程中，会调用函数`el3_entrypoint_common`来初始化异常向量寄存器（VBAR/MVBAR）。以下是`el3_entrypoint_common`的部分实现，其中设定异常向量的基地址为：

```assembly
#if !RESET_TO_BL31
	/* ---------------------------------------------------------------------
	 * For !RESET_TO_BL31 systems, only the primary CPU ever reaches
	 * bl31_entrypoint() during the cold boot flow, so the cold/warm boot
	 * and primary/secondary CPU logic should not be executed in this case.
	 *
	 * Also, assume that the previous bootloader has already initialised the
	 * SCTLR_EL3, including the endianness, and has initialised the memory.
	 * ---------------------------------------------------------------------
	 */
	el3_entrypoint_common					\
		_init_sctlr=0					\
		_warm_boot_mailbox=0				\
		_secondary_cold_boot=0				\
		_init_memory=0					\
		_init_c_runtime=1				\
		_exception_vectors=runtime_exceptions		\
		_pie_fixup_size=BL31_LIMIT - BL31_BASE
#else

	/* ---------------------------------------------------------------------
	 * For RESET_TO_BL31 systems which have a programmable reset address,
	 * bl31_entrypoint() is executed only on the cold boot path so we can
	 * skip the warm boot mailbox mechanism.
	 * ---------------------------------------------------------------------
	 */
	el3_entrypoint_common					\
		_init_sctlr=1					\
		_warm_boot_mailbox=!PROGRAMMABLE_RESET_ADDRESS	\
		_secondary_cold_boot=!COLD_BOOT_SINGLE_CPU	\
		_init_memory=1					\
		_init_c_runtime=1				\
		_exception_vectors=runtime_exceptions		\
		_pie_fixup_size=BL31_LIMIT - BL31_BASE
```

在runtime_exceptions中会设定不同异常向量的入口函数，其中smc指令产生的异常属于Synchronous异常，分别对应AArch64/32模式下的入口为sync_exception_aarch64/32，两者都调用同一个处理函数handle_sync_exception。

### 2.2.2 EL3处理Monitor调用流程
ARMv8调用smc指令产生安全监控模式调用后，ARM核会切换到EL3中，然后读取MVBAR寄存器中的异常向量表的基地址来获取异常向量表的内容，并命中安全监控模式调用请求处理函数。对于AArch32和AArch64结构，安全监控模式调用的处理函数不同，但最终都会调用`handle_sync_exception`函数来对安全监控模式调用进行处理。进入`handle_sync_exception`函数后会对触发安全监控模式调用的世界进行判定，并设定需要切换到的那个世界的状态并恢复对应的CPU上下文，再根据安全监控模式调用ID进入具体的分支，并将ARM核的运行模式切换成EL1或者EL0，待安全监控模式调用处理完毕后会再次触发安全监控模式调用，触发异常重新进入EL3中继续运行余下流程。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typoraflowchar1_bl31_bl32.svg" width="100%" /></div>

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221005170738.png)

### 2.2.3 opteed_smc_handler
EL3中用于处理OP-TEE的安全监控模式调用是通过调用opteed_smc_handler来实现的，该函数在ATF启动时会被编译到rt_svc_descs段中。该函数的内容如下：

在该函数中上下文会被保存到全局变量opteed_sp_context中，optee初始化完成后返回smc处理的流程如下（services/spd/opteed/opteed_main.c）：

```c
uintptr_t opteed_smc_handler(…)
{
optee_context_t *optee_ctx = &opteed_sp_context[linear_id];
    …
	switch (smc_fid) {
	case TEESMC_OPTEED_RETURN_ENTRY_DONE:                             （1）
		assert(optee_vector_table == NULL);
		optee_vector_table = (optee_vectors_t *) x1;
		…
		opteed_synchronous_sp_exit(optee_ctx, x1);                 （2）
		break;
    …
	}
}
```

opteed_smc_handler函数是ATF用于处理OPTEE产生的安全监控模式调用的处理函数，该函数会对具体的安全监控模式调用类型进行处理。

### 2.2.4 安全世界状态中触发monitor调用过程
在安全世界状态中触发安全监控模式调用后，ARM核会进入EL3中，从MVBAR中获取异常向量表的基地址，并找到安全监控模式调用的处理函数，然后进入handle_sync_exception函数，再调用opteed_smc_handler函数对该安全监控模式调用进行处理，该函数中将判定该安全监控模式调用时SCR寄存器中的安全状态位是否为安全值，然后再根据SMC ID来决定是否需要恢复正常世界状态的运行上下文，整体过程如图：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221005171613.png)

当OP-TEE处理完来自正常世界状态的安全监控模式调用后会再次触发安全监控模式调用重新进入EL3，再次调用EL3的安全监控模式调用处理函数，调用opteed_smc_handler函数，根据SMC ID进入不同的分支。一般情况下会进入TEESMC_OPTEED_RETURN_CALL_DONE的分支，在该分支中会保存安全世界状态的运行上下文并恢复正常世界状态的运行上下文，然后调用SMC_RET4返回到正常世界状态中继续运行。

### 2.2.5 正常世界状态中触发monitor调用过程

在正常世界状态中调用smc指令触发安全监控模式调用后，ARM核会进入EL3，即ATF中的bl31，进入EL3后会从MVBAR寄存器中获取到EL3的异常向量表，然后命中安全监控模式调用的处理函数，最终调用opteed_smc_handler来处理该安全监控模式调用，EL3处理正常世界状态中触发的安全监控模式调用的整体流程如图所示。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221005171707.png)

在opteed_smc_handler函数中会调用is_caller_non_secure来判定当前安全监控模式调用是来自正常世界状态还是安全世界状态。如果异常来自正常世界状态，则会保存正常世界状态的运行上下文并恢复安全世界状态的运行上下文，然后根据SMC ID将快速安全监控模式调用（fast smc）或标准安全监控模式调用（stf smc）的处理函数注册到运行上下文中，然后通过调用SMC_RET4进入OP-TEE中对该安全监控模式调用做进一步处理。

# 3. 总结
ARMv7架构中对安全监控模式调用的处理是在Monitor模式下进行的，Monitor模式具有独立的代码。而在ARMv8架构中，对安全监控模式调用的处理则是在ATF的bl31中实现的，在ATF中ARM为兼容不同厂商的TEE方案，提供了集成接口，只要按照一定规范就可以将TEE方案对安全监控模式调用的最终处理逻辑和接口添加到ATF的bl31中。