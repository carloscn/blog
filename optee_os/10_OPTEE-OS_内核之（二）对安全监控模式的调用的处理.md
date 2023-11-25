# 10_OPTEE-OS_内核之（二）对安全监控模式的调用的处理
来自正常世界状态（NWS）的安全监控模式调用（smc）最终都会使用OP-TEE提供的处理接口进行处理，而该处理接口中的内容在OP-TEE启动过程中会被初始化。ARM官方将安全监控模式调用的类型分为两个大类：快**速安全监控模式调用（fast smc）和标准安全监控模式调用（std smc）**，使用不同的SMC ID来表示，关于SMC ID的含义和设置，可参阅[09_OPTEE-OS_内核之（一）ARM核安全态和非安全态的切换](https://github.com/carloscn/blog/issues/99)。ARMv7或者ARMv8中都使用smc汇编指令来使ARM核陷入Monitor模式或者EL3阶段，Monitor模式或者EL3判定安全状态位（NS bit）后会设置对应的运行上下文，然后退出Monitor模式或者EL3阶段，再跳转到OP-TEE中使用特定的处理接口作进一步处理。

# 1. OP-TEE的线程向量表
OP-TEE中会定义一个线程向量表 `thread_vector_table`，该线程向量表会被Monitor模式或者EL3使用。
* 在ARMv7架构中，Monitor模式的处理代码将会使用该变量来进行安全监控模式调用的最终处理；
* 在ARMv8架构中，该线程向量表的地址会在OP-TEE启动过程中返回给ATF中的bl31，当EL3接收到FIQ、smc或者其他事件时，将会使用该线程向量表中的具体函数来对事件进行最终的处理。

关于线程向量表和全局处理变量的内容可参阅 [14_OPTEE-OS_内核之（六）线程管理与并发](https://github.com/carloscn/blog/issues/104)。

在OP-TEE中用于处理各种来自外部或者的monitor模式请求的入口函数都存放在OP-TEE的线程向量表thread_vector_table中。该项量的实现在`optee_os/core/arch/arm/kernel/thread_a32.S`文件中。其内容如下：

```
FUNC thread_vector_table , :
UNWIND(	.fnstart)
UNWIND(	.cantunwind)
	b	vector_std_smc_entry	//OP-TEE中处理标准smc请求
	b	vector_fast_smc_entry	//OP-TEE中处理快速smc
	b	vector_cpu_on_entry	//OP-TEE中处理cpu on操作
	b	vector_cpu_off_entry	//OP-TEE中处理cpu off操作
	b	vector_cpu_resume_entry	//OP-TEE中处理resume操作
	b	vector_cpu_suspend_entry	//OP-TEE中处理cpu suspend操作
	b	vector_fiq_entry		//OP-TEE中处理处理快速中断操作
	b	vector_system_off_entry		//OP-TEE中处理系统off操作
	b	vector_system_reset_entry	//OP-TEE中处理系统重启操作
UNWIND(	.fnend)
END_FUNC thread_vector_table
```

**注意**:该`线程向量表`与OP-TEE的中断处理向量表`thread_vect_table`是不一样的。`thread_vector_table` 属于线程级别，会被monitor模式或者其他中断处理函数调用到，而`thread_vect_table`才是OP-TEE存放在VBAR寄存器中的中断向量表。当在secure world状态下产生了FIQ事件时，将会调用中断向量表thread_vect_table中的FIQ中断处理函数，然后才会调用到thread_vector_table中给的vector_fiq_entry来执行FIQ的后续处理。

# 2. ARMv7中Monitor模式对安全监控模式的调用的处理

当在正常世界状态或者安全世界状态中触发了安全监控模式调用后，在ARMv7架构中ARM核会切换到Monitor模式，且从MVBAR寄存器中获取到异常向量表的基地址，然后查找到对安全监控模式调用的处理函数——sm_smc_entry，使用该函数来完成对安全监控模式调用的处理。在处理过程中会判定该安全监控模式调用来自正常世界状态还是安全世界状态，如果触发该安全监控模式调用是正常世界状态，则会调用smc_from_nsec函数进行处理，然后再根据SMC ID判定该安全监控模式调用的类型做进一步处理。在Monitor模式中对安全监控模式调用的处理过程如图所示。

libteec和tee_supplicant调用接口之后最终会调用到OP-TEE驱动来触发对应的SMC操作。在OP-TEE驱动中触发SMC操作的方法是调用`arm_smccc_smc(a0, a1, a2, a3, a4, a5, a6, a7, res)`来实现，其中在REE端需要传递給TEE侧的数据被存放在`a0~a7`中。调用上述函数自后，CPU中的cortex就会切换到monitor模式，进入monitor模式之后首先会去获取`MVBAR寄存器`中存放的monitor模式的`中断向量表地址`，然后查找monitor模式的中断向量表，命中smc的处理函数。

进入到处理函数之后再根据从REE侧带入的参数判定是进行`快速smc处理`还是`标准的smc处理`。关于monitor模式下如何实现normal world到secure world之间的切换过程请参考文章[09_OPTEE-OS_内核之（一）ARM核安全态和非安全态的切换](https://github.com/carloscn/blog/issues/99)。在REE端触发smc请求之后在monitor的处理流程如下图所示：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221005173125.png)

ARMv7架构通过Monitor模式来实现正常世界状态到安全世界状态之间的切换，并根据不同的SMC ID来判定当前安全监控模式调用是快速安全监控模式调用（fast smc）还是标准安全监控模式调用（std smc），然后通过查找线程向量表进入到fast smc和std smc的处理函数，在各自的处理函数中最终会调用OP-TEE中的全局handler变量中对应的函数指针来实现对该安全监控模式调用的具体处理。当程序运行到sm_from_nsec的时候就已经完成了normal world与secure world的切换，从该函数开始就进入到了OP-TEE的处理范畴。

# 3. ARMv8中的EL3处理安全监控模式调用的实现

ARMv8架构使用ATF中的bl31来实现安全世界状态与正常世界状态之间的切换，以及安全监控模式调用的第一步处理，bl31运行于EL3，所有的安全监控模式调用在ARMv8架构中都会在EL3先被处理，然后根据不同的TEE方案使用对应的接口进行安全监控模式调用的分发，在分发之前，bl31会设定好ARM核安全状态，保存当前CPU的运行上下文并恢复将要切换到的ARM核状态对应的运行上下文。关于EL3中如何实现正常世界状态与安全世界状态的切换以及如何跳转到OP-TEE中运行，可参阅文章[09_OPTEE-OS_内核之（一）ARM核安全态和非安全态的切换](https://github.com/carloscn/blog/issues/99)。从EL3进入OP-TEE是通过调用OP-TEE在初始化阶段提供的线程向量表来实现的，即EL3在设定CPU运行上下文时会根据SMC ID来判定是进入到vector_std_smc_entry还是vector_fast_smc_entry，在EL3中对安全监控模式调用（smc）的处理流程如图所示。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221005173532.png)

# 4. OP-TEE对fast SMC的处理
快速安全监控模式调用（fast smc）一般会在驱动挂载过程中，或需要**获取OP-TEE OS版本信息、共享内存配置、Cache信息时被调用**。OP-TEE不会使用建立线程的方式对fast smc进行处理，而是在OP-TEE的内核空间调用tee_entry_fast函数对安全监控模式调用（smc）进行处理，并通过再次产生安全监控模式调用（smc）的方式返回最终的处理结果。在OP-TEE中对fast smc的处理过程：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221005173746.png)

在sm_from_nsec中调用最终会调用到thread_vector_table向量表中的vector_fast_smc_entry函数来对fast smc进行处理。该函数使用汇编实现，定义在`optee_os/core/arch/arm/kernel/thread_a32.S`中，其内容如下：

```C
LOCAL_FUNC vector_fast_smc_entry , :
UNWIND(	.fnstart)
UNWIND(	.cantunwind)
	push	{r0-r7}	//将r0~r7入栈
	mov	r0, sp	//将栈地址赋值给r0
	bl	thread_handle_fast_smc	//调用thread_handle_fast_smc进行处理，参数为r0中的数据
	pop	{r1-r8}	//处理完成之后，执行出栈操作
	ldr	r0, =TEESMC_OPTEED_RETURN_CALL_DONE 	//r0存放fast smc处理操作的结果
	smc	#0	//触发smc请求 切换到monitor模式，返回normal world中
	b	.	/* SMC should not return */
UNWIND(	.fnend)
END_FUNC vector_fast_smc_entry
 
void thread_handle_fast_smc(struct thread_smc_args *args)
{
/* 使用canaries原理检查栈空间是否存在溢出或者被破坏 */
	thread_check_canaries();
 
/* 调用thread_fast_smc_handler_ptr处理smc请求 */
	thread_fast_smc_handler_ptr(args);
	/* Fast handlers must not unmask any exceptions */
	assert(thread_get_exceptions() == THREAD_EXCP_ALL);
}
```

fast smc被处理完成后会重新触发安全监控模式调用：
* 对于ARMv7而言，触发该安全监控模式调用的作用是让ARM核重新进入Monitor模式，最终将结果返回给正常世界状态。
* 对于ARMv8而言，触发该安全监控模式调用的作用是让ARM核重新进入EL3，即bl31中。在bl31中最终会调用opteed_smc_handler函数对该安全监控模式调用进行处理，根据该SMC的ID号进入`TEESMC_OPTEED_RETURN_CALL_DONE`分支，执行保存安全世界状态上下文、恢复正常世界状态上下文，并将返回的数据填充到正常世界状态上下文中，然后调用exit_el3退出EL3返回到正常世界状态中继续执行。tee_entry_fast中的内容如下，用户可以根据实际的需求增加。处理函数源码如下。

在OP-TEE启动的时候会执行init_handlers操作，该函数的主要作用是将真正的处理函数赋值给各种thread函数指针变量。关于init_handlers函数的调用和处理过程请查阅前期文章。thread_fast_smc_handler_ptr会被赋值成`handlers->fast_smc`，而在vxpress板级中`handlers->fast_smc`执行tee_entry_fast函数。该函数内容如下:

```C
void tee_entry_fast(struct thread_smc_args *args)
{
	switch (args->a0) {
 
	/* Generic functions */
/* 获取API被调用的次数，可以根据实际需求实现 */
	case OPTEE_SMC_CALLS_COUNT:
		tee_entry_get_api_call_count(args);
		break;
/* 获取OP-TEE API的UID值 */
	case OPTEE_SMC_CALLS_UID:
		tee_entry_get_api_uuid(args);
		break;
/* 获取OP-TEE中API的版本信息 */
	case OPTEE_SMC_CALLS_REVISION:
		tee_entry_get_api_revision(args);
		break;
/* 获取OP-TEE OS的UID值 */
	case OPTEE_SMC_CALL_GET_OS_UUID:
		tee_entry_get_os_uuid(args);
		break;
/* 获取OS的版本信息 */
	case OPTEE_SMC_CALL_GET_OS_REVISION:
		tee_entry_get_os_revision(args);
		break;
 
	/* OP-TEE specific SMC functions */
/* 获取OP-TEE与驱动之间的共享内存配置信息 */
	case OPTEE_SMC_GET_SHM_CONFIG:
		tee_entry_get_shm_config(args);
		break;
/* 获取I2CC的互斥体信息 */
	case OPTEE_SMC_L2CC_MUTEX:
		tee_entry_fastcall_l2cc_mutex(args);
		break;
/* OP-TEE的capabilities信息 */
	case OPTEE_SMC_EXCHANGE_CAPABILITIES:
		tee_entry_exchange_capabilities(args);
		break;
/* 关闭OP-TEE与驱动的共享内存的cache */
	case OPTEE_SMC_DISABLE_SHM_CACHE:
		tee_entry_disable_shm_cache(args);
		break;
/* 使能OP-TEE与驱动之间共享内存的cache */
	case OPTEE_SMC_ENABLE_SHM_CACHE:
		tee_entry_enable_shm_cache(args);
		break;
/* 启动其他cortex的被使用 */
	case OPTEE_SMC_BOOT_SECONDARY:
		tee_entry_boot_secondary(args);
		break;
 
	default:
		args->a0 = OPTEE_SMC_RETURN_UNKNOWN_FUNCTION;
		break;
	}
}
```

从上面的函数可以看到，tee_entry_fast会根据不同的command ID来执行特定的操作。使用者也可以在此函数中添加自己需要fast smc实现的功能，只要在REE侧和OP-TEE中定义合法fast smc的command ID并实现具体操作就可以。

# 5. OP-TEE对std SMC调用的处理
当OP-TEE驱动中触发标准安全监控模式调用（std smc）时：
* ARMv7架构的ARM核会进入Monitor模式，然后使用线程向量表中的vector_std_smc_entry来处理该请求；
* ARMv8架构的核则进入EL3，处理过程最终同样也会调用OP-TEE中定义的线程向量表中的vector_std_smc_entry来对该请求进行处理。在Monitor模式或EL3都是根据a0参数中的bit[31]来判定是快速安全监控模式调用（fast smc）还是标准安全监控模式调用。如果bit[31]的值是0，则会进入标准安全监控模式调用的处理逻辑。vector_std_smc_entry函数的执行流程如图所示。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221005174427.png)

## 5.1 用于处理标准smc的向量处理

用于处理标准smc的向量处理函数如下：

```C
LOCAL_FUNC vector_std_smc_entry , :
UNWIND(	.fnstart)
UNWIND(	.cantunwind)
	push	{r0-r7}	//将参数入栈
	mov	r0, sp	//将栈指针赋值给r0寄存器
	bl	thread_handle_std_smc	//调用处理函数，参数的地址存放在r0寄存器中
	/*
	 * Normally thread_handle_std_smc() should return via
	 * thread_exit(), thread_rpc(), but if thread_handle_std_smc()
	 * hasn't switched stack (error detected) it will do a normal "C"
	 * return.
	 */
	pop	{r1-r8}	//出栈操作
	ldr	r0, =TEESMC_OPTEED_RETURN_CALL_DONE	//标记OP-TEE处理完成
	smc	#0	//调用smc切回到normal world
	b	.	/* SMC should not return */
UNWIND(	.fnend)
END_FUNC vector_std_smc_entry
```

函数thread_handle_std_smc的内容如下：

```C
void thread_handle_std_smc(struct thread_smc_args *args)
{
/* 检查堆栈 */
	thread_check_canaries();
 
	if (args->a0 == OPTEE_SMC_CALL_RETURN_FROM_RPC)
//处理由tee_supplican回复的RPC请求处理结果
		thread_resume_from_rpc(args);
	else
//处理来自Libteec的请求，主要包括open session, close session, invoke等
		thread_alloc_and_run(args);	
}
```

只有在libteec中触发的smc后，需要OP-TEE作出相应的操作后才可能产生来自RPC请求，故先介绍OP-TEE对来自libteec请求部分，主要是对打开session, 关闭close, 调用特定TA的command，取消command等操作。

## 5.2 对RPC请求返回操作处理
远程处理请求（Remote Procedure Call，RPC）是指OP-TEE需要REE侧协助完成对REE侧资源进行操作的请求。当OP-TEE需要操作REE侧的资源时，OP-TEE会发送RPC类型的安全监控模式调用，REE侧收到来自OP-TEE的RPC请求后，REE侧根据RPC请求的ID进行处理并将处理结果返回给OP-TEE，关于在REE侧如何获取和处理RPC请求可参阅[07_OPTEE-OS_系统集成之（五）REE侧上层软件](https://github.com/carloscn/blog/issues/97)。待REE侧处理完成后，会将处理结果放在OP-TEE驱动设备teepriv0的返回队列中，然后在驱动中触发安全监控模式调用将结果发送到OP-TEE中。OP-TEE驱动产生的安全监控模式调用请求是标准类型的SMC，最终在OP-TEE中会调用thread_resume_from_rpc函数对该请求进行处理。RPC请求的处理过程如图所示：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221005174909.png)

OP-TEE在发送RPC请求时会带入发送该请求的线程的ID，该ID将会在接收RPC结果时被用于恢复该线程继续执行。关于RPC操作在OP-TEE中的处理过程将会在[15_OPTEE-OS_内核之（七）系统调用及IPC机制](https://github.com/carloscn/blog/issues/105)中详细介绍。

## 5.3 处理libteec的smc请求

### 5.3.1 new thread
在[07_OPTEE-OS_系统集成之（五）REE侧上层软件](https://github.com/carloscn/blog/issues/97)一文中介绍了libteec提供给上层使用的所有接口，这些接口调用之后就会就有可能需要OP-TEE进行对应的操作。在monitor模式对这些请求处理之后会进入到OP-TEE中，然后调用thread_alloc_and_run创建一个线程来对请求做专门的处理。而且在处理过成中还有可能TEE与REE侧之间的RPC请求等。thread_alloc_and_run函数的内容如下：
```c
static void thread_alloc_and_run(struct thread_smc_args *args)
{
	size_t n;
/* 获取当前CPU的ID，并返回该CPU core的对应结构体 */
	struct thread_core_local *l = thread_get_core_local();
	bool found_thread = false;
 
/* 判定是否有thread正在占用CPU */
	assert(l->curr_thread == -1);
 
/* 锁定线程状态 */
	lock_global();
 
/* 查找系统中那个线程空间当前可用 */
	for (n = 0; n < CFG_NUM_THREADS; n++) {
		if (threads[n].state == THREAD_STATE_FREE) {
			threads[n].state = THREAD_STATE_ACTIVE;
			found_thread = true;
			break;
		}
	}
 
/* 解锁 */
	unlock_global();
 
/* 初步设定返回给REE侧驱动的结果为OPTEE_SMC_RETURN_ETHREAD_LIMIT，返回的
数据在后续处理中会被更改 */
	if (!found_thread) {
		args->a0 = OPTEE_SMC_RETURN_ETHREAD_LIMIT;
		return;
	}
 
/* 记录当前cortex使用了那个thread空间来执行操作 */
	l->curr_thread = n;
 
/* 设置选中的thread空间的flag为0*/
	threads[n].flags = 0;
 
/*并对该thread中使用的pc，cpsr等相关寄存器进行设置,并且将参数传递到thread context的
reg.ro~reg.r7中 */
	init_regs(threads + n, args);
 
	/* Save Hypervisor Client ID */
/* 保存Hypervisor客户端的ID值 */
	threads[n].hyp_clnt_id = args->a7;
 
/* 保存vfp相关数据 */
	thread_lazy_save_ns_vfp();
 
/* 调用thread_resume函数，开始执行被初始化好的thread */
	thread_resume(&threads[n].regs);
}
```

thread_alloc_and_run会建立一个thread，并通过init_regs函数初始化该thread的运行上下文，指定该thread的入口函数以及运行时的参数，初始化完成之后，调用thread_resume启动该线程。thread的运行上下文的配置和初始化在init_regs函数中实现，内容如下：

```c
static void init_regs(struct thread_ctx *thread,
		struct thread_smc_args *args)
{	
/* 指定该线程上下文中PC指针的地址，当该thread resume回来之后就会开始执行regs.pc执
行的函数 */
	thread->regs.pc = (uint32_t)thread_std_smc_entry;
 
	/*
	 * Stdcalls starts in SVC mode with masked foreign interrupts, masked
	 * Asynchronous abort and unmasked native interrupts.
	 */
/* 设定cpsr寄存器的值，屏蔽外部中断，进入SVC模式 */
	thread->regs.cpsr = read_cpsr() & ARM32_CPSR_E;
	thread->regs.cpsr |= CPSR_MODE_SVC | CPSR_A |
			(THREAD_EXCP_FOREIGN_INTR << ARM32_CPSR_F_SHIFT);
	/* Enable thumb mode if it's a thumb instruction */
	if (thread->regs.pc & 1)
		thread->regs.cpsr |= CPSR_T;
	/* Reinitialize stack pointer */
	thread->regs.svc_sp = thread->stack_va_end; 	//重新定位栈地址
 
	/*
	 * Copy arguments into context. This will make the
	 * arguments appear in r0-r7 when thread is started.
	 */
/* 配置运行时传入的参数 */
	thread->regs.r0 = args->a0;
	thread->regs.r1 = args->a1;
	thread->regs.r2 = args->a2;
	thread->regs.r3 = args->a3;
	thread->regs.r4 = args->a4;
	thread->regs.r5 = args->a5;
	thread->regs.r6 = args->a6;
	thread->regs.r7 = args->a7;
}
```

### 5.3.2 resume thread
通过init_regs配置完thread的运行上下文之后，通过调用thread_resume函数来唤醒该线程，让其进入到执行状态。resume函数使用汇编来实现，主要是保存一些寄存器状态，指定thread运行在什么模式。

```c
FUNC thread_resume , :
UNWIND(	.fnstart)
UNWIND(	.cantunwind)
	add	r12, r0, #(13 * 4)	/* Restore registers r0-r12 later */
 
	cps	#CPSR_MODE_SYS	//进入sys模式
	ldm	r12!, {sp, lr}	
 
	cps	#CPSR_MODE_SVC	//进入到svc模式
	ldm	r12!, {r1, sp, lr}		
	msr	spsr_fsxc, r1
 
	cps	#CPSR_MODE_SVC	//进入到svc模式
	ldm	r12, {r1, r2}
	push	{r1, r2}	//出栈操作
 
	ldm	r0, {r0-r12}	//将参数存放到r0~r12中
 
	/* Restore CPSR and jump to the instruction to resume at */
	rfefd	sp!	//跳转到thread的pc指针出执行并返回
UNWIND(	.fnend)
END_FUNC thread_resume
```

### 5.3.3 线程入口函数
init_regs的regs.pc中已经指定了该线程被恢复回来后pc指针的值为thread_std_smc_entry。当线程被恢复后就会去执行该函数，进入到处理由调用libteec库中的接口引起的安全监控模式调用（smc）的过程，该入口函数使用汇编实现，内容如下：
在init_regs中的regs.pc中已经制定了该thread被resume回来之后的pc指针为thread_std_smc_entry，当thread被resume之后就会去执行该函数

```c
FUNC thread_std_smc_entry , :
UNWIND(	.fnstart)
UNWIND(	.cantunwind)
	/* Pass r0-r7 in a struct thread_smc_args */
	push	{r0-r7}	//入栈操作，将r0~r7的数据入栈
	mov	r0, sp	//将r0执行栈地址作为参数传递給__thread_std_smc_entry
	bl	__thread_std_smc_entry		//正式对标准smc进行处理
	/*
	 * Load the returned r0-r3 into preserved registers and skip the
	 * "returned" r4-r7 since they will not be returned to normal
	 * world.
	 */
	pop	{r4-r7}
	add	sp, #(4 * 4)
 
	/* Disable interrupts before switching to temporary stack */
	cpsid	aif	//关闭中断
	bl	thread_get_tmp_sp	//获取堆栈
	mov	sp, r0	//将r0的值存放到sp中
 
	bl	thread_state_free	//释放thread
 
	ldr	r0, =TEESMC_OPTEED_RETURN_CALL_DONE//设置返回到normal的r0寄存器的值
	mov	r1, r4
	mov	r2, r5
	mov	r3, r6
	mov	r4, r7
	smc	#0	//调用smc，切回到normal world
	b	.	/* SMC should not return */
UNWIND(	.fnend)
END_FUNC thread_std_smc_entry
```

进入线程后会使用__thread_std_smc_entry函数进行处理，在该函数中会调用在OP-TEE启动过程中初始化的全局handler指针函数来处理标准的安全监控模式调用（std smc），处理完成后该线程资源将会被释放，线程编号将会被重新设定成可用状态等待下次调用。

### 5.3.4 标准smc请求的handle
在`__thread_std_smc_entry`函数中最终会调用thread_std_smc_handler_ptr来对请求做正式的处理，而thread_std_smc_handler_ptr是在OP-TEE启动的过程中执行init_handlers函数时被初始化成了`handlers->std_smc`。而`handlers->std_smc`根据不同的板级可能有所不同，在vexpress板级中std_smc的值为tee_entry_std。

```C
void tee_entry_std(struct thread_smc_args *smc_args)
{
	paddr_t parg;
	struct optee_msg_arg *arg = NULL;	/* fix gcc warning */
	uint32_t num_params;
 
/* 判定a0是否合法 */
	if (smc_args->a0 != OPTEE_SMC_CALL_WITH_ARG) {
		EMSG("Unknown SMC 0x%" PRIx64, (uint64_t)smc_args->a0);
		DMSG("Expected 0x%x\n", OPTEE_SMC_CALL_WITH_ARG);
		smc_args->a0 = OPTEE_SMC_RETURN_EBADCMD;
		return;
	}
/* 判定传入参数起始地址否存属于non-secure memory中，因为驱动与OP-TEE之间使用共
享内存来共享数据，而共享内存属于非安全内存 */
	parg = (uint64_t)smc_args->a1 << 32 | smc_args->a2;
	if (!tee_pbuf_is_non_sec(parg, sizeof(struct optee_msg_arg)) ||
	    !ALIGNMENT_IS_OK(parg, struct optee_msg_arg) ||
	    !(arg = phys_to_virt(parg, MEM_AREA_NSEC_SHM))) {
		EMSG("Bad arg address 0x%" PRIxPA, parg);
		smc_args->a0 = OPTEE_SMC_RETURN_EBADADDR;
		return;
	}
 
/* 检查所有参数是否存放在non-secure memory中 */
	num_params = arg->num_params;
	if (!tee_pbuf_is_non_sec(parg, OPTEE_MSG_GET_ARG_SIZE(num_params))) {
		EMSG("Bad arg address 0x%" PRIxPA, parg);
		smc_args->a0 = OPTEE_SMC_RETURN_EBADADDR;
		return;
	}
 
	/* Enable foreign interrupts for STD calls */
	thread_set_foreign_intr(true);	//使能中断
 
/* 根据参数的cmd成员来判定来自libteec的请求是要求OP-TEE做什么操作 */
	switch (arg->cmd) {
/* 执行打开session操作 */
	case OPTEE_MSG_CMD_OPEN_SESSION:
		entry_open_session(smc_args, arg, num_params);
		break;
/* 执行关闭session的操作 */
	case OPTEE_MSG_CMD_CLOSE_SESSION:
		entry_close_session(smc_args, arg, num_params);
		break;
/* 请求特定TA执行特定的command */
	case OPTEE_MSG_CMD_INVOKE_COMMAND:
		entry_invoke_command(smc_args, arg, num_params);
		break;
/* 请求cancel掉某个session的command */
	case OPTEE_MSG_CMD_CANCEL:
		entry_cancel(smc_args, arg, num_params);
		break;
	default:
		EMSG("Unknown cmd 0x%x\n", arg->cmd);
		smc_args->a0 = OPTEE_SMC_RETURN_EBADCMD;
	}
}
```

在tee_entry_std函数中会根据在OP-TEE中填入的cmd值执行不同的分类操作，主要包括打开TA与CA之间的session操作，关闭session操作，CA请求invoke操作，取消invoke操作中特定的cmd操作等，开发者也可以根据实际需求对该部分进行扩展，但是必须保证在REE侧和TEE侧的修改一致。在执行打开session的操作时，根据调用的TA是属于静态TA还是动态TA可能会触发RPC请求，当TA image存放在文件系统中时，在打开session时OP-TEE就会触发RPC请求，请求tee_supplicant从文件系统中读取TA image的内容，并将内容传递给OP-TEE，然后经过对image的校验判定完成TA image的加载操作后才执行open session查找并将该session添加到OP-TEE的全局session的队列中，以便在执行invoke时查询session队列找到对应的session。

# 6. 总结
本章介绍了OP-TEE中处理快速安全监控模式调用（fast smc）和标准安全监控模式调用（std smc）的详细过程以及由RPC请求和调用libteec库中的接口产生的std smc的处理过程，而对于libteec库产生的std smc进入到tee_entry_std处理之后会根据command ID进行不同的操作，即Open session、close session、invoke command等。open session是invoke command操作的前提，在open session操作中会根据TA的UUID来进行运行上下文的配置，并根据是动态TA还是静态TA配置不同的operation，关于各invoke command的操作将会在[12_OPTEE-OS_内核之（四）对TA请求的处理](https://github.com/carloscn/blog/issues/102)中详细介绍。

