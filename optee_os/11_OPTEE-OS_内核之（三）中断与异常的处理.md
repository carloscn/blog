# 11_OPTEE-OS_内核之（三）中断与异常的处理
一个完整的系统都会存在中断，ARMv7架构扩展出了Monitor模式，而ARMv8使用EL的方式对ARM异常运行模式进行了重新定义，分为EL0~EL3。在ARMv8架构系统中，**OP-TEE运行于安全侧的EL1，bl31运行于EL3**。系统运行过程中任何阶段都有可能会产生外部中断。本章将主要介绍FIQ事件和IRQ事件在OP-TEE、ARMv7架构中的Monitor模式、ARMv8架构中的EL3的处理过程。

# 1. 系统的中断处理
ARM核处于安全世界状态（SWS）和正常世界状态（NWS）都具有独立的VBAR寄存器和中断向量表。而当ARM核处于Monitor模式或者EL3时，ARM核将具有独立的中断向量表和MVBAR寄存器。想实现各种中断在三种状态下被处理的统一性和正确性，就需要确保各种状态下中断向量表以及GIC的正确配置。**ARM的指导手册中建议在TEE中使用FIQ，在ROS中使用IRQ，即TEE侧会处理由中断引起的FIQ事件，而Linux内核端将会处理中断引起的IRQ事件**。而由于ATF的使用，Monitor状态或者EL3下中断的处理代码将会在ATF中实现。(ATF来处理更高级及跟敏感的中断)

针对ARMv7核，中断与ARM核每种状态的关系图如图所示：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221005204526.png)

系统中的中断主要被分为Native Interrupt和Foreign Interrupt事件，FIQ会被TEE侧处理，IRQ会被REE侧处理，如果在Monitor模式或EL3阶段产生了中断，则处于Monitor模式或者EL3的软件会使用MVBAR寄存器中保存的异常向量表中的处理函数对FIQ或者IRQ事件进行处理。

# 2. 中断控制器
中断控制器（General Interruption Controller，GIC）模块是CPU的外设之一，它的作用是接收来自其他外设的中断引脚输入，然后根据中断触发模式、中断类型优先级等设置来控制发送不同的中断信号到CPU。ARM对GIC的架构也在不断改进，已经从GICv1发展到现在的GICv4版本。目前主要使用的是GICv2和GICv3架构。本书将介绍在支持TEE安全扩展的ARM处理器平台上这两个版本的中断控制器是如何工作的。

参考：[12_ARMv8_异常处理（三）- GICv1/v2中断处理](https://github.com/carloscn/blog/issues/51)

## 2.1 GIC寄存器
GIC模块中的寄存器主要分为中断控制分发寄存器（缩写为GICD）以及CPU接口寄存器（缩写为GICC）两部分。GICD接收所有的中断源，然后根据中断的优先级来判定是否响应中断，以及是否将该中断信号转发到对应的CPU。GICC和各个ARM核相连。当收到来自GICD的中断信号时，由GICC来决定是否将中断请求发送给ARM核。

支持安全扩展的GIC模块将中断分为了两组：Group0中断和Group1中断。

* 对于ARMv7架构，Group0为安全中断，Group1为非安全中断。
* 对于ARMv8架构，Group0为安全中断且有最高的优先级，而Group1又分安全中断（Group1 Secure，G1S）和非安全中断（Group1 NonSecure，G1NS）。

GIC会根据中断所在的Group安全类型及当前ARM核运行模式来决定是发送FIQ还是IRQ信号到ARM核。根据GIC版本的不同其决定方式也不同。关于这点将在接下来的章节分开介绍。另外，当ARM核收到FIQ/IRQ信号后会进入哪种模式是由SCR寄存器来决定的。

ARMv8架构中，OP-TEE根据中断要求触发的模式将中断类型分为三类，其定义如下：
```C
#define INTR_TYPE_S_EL1      0        // 该中断应该由Secure EL1处理
#define INTR_TYPE_EL3        1        // 该中断应该由EL3处理
#define INTR_TYPE_NS         2        // 该中断应该由Normal World 处理
```
不同版本的GIC对于以上三种类型的中断将会产生不同的IRQ或FIQ事件，故需要先根据GIC版本来确定上述三种类型的中断所产生的是IRQ还是FIQ事件，然后再设定SCR寄存器中SCR.FIQ和SCR.IRQ位来决定该中断是否会触发ARM核进入EL3阶段。

## 2.2 ARMv7 SCR寄存器
![](https://raw.githubusercontent.com/carloscn/images/main/typora20221005205109.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221005205601.png)

ARMv7架构中，SCR寄存器中的值是在optee_os/core/arch/arm/sm/sm_a32.S文件被设定，其内容如下：
```assembly
.sm_ret_to_nsec:
	//回到Normal World之前设定SCR.NS位        
	add     r0, sp, #(SM_CTX_NSEC + SM_NSEC_CTX_R8)        
	ldm     r0, {r8-r12} //设定SCR.NS下FIQ为1, FIQ中断会进入Monitor模式        
	read_scr r0        
	orr     r0, r0, #(SCR_NS | SCR_FIQ)        
	write_scr r0        
	add     sp, sp, #(SM_CTX_NSEC + SM_NSEC_CTX_R0)        
	b       .sm_exit 
.smc_from_nsec: //进入Secure World        
	sub     sp, sp, #(SM_CTX_NSEC + SM_NSEC_CTX_R0) /* 设定SCR.FIQ位为0,FIQ中断会直接通过VBAR进入EL1S FIQ异常向量 */        
	bic     r1, r1, #(SCR_NS | SCR_FIQ)        
	write_scr r1        
	add     r0, sp, #(SM_CTX_NSEC + SM_NSEC_CTX_R8)        
	stm     r0, {r8-r12}        
	mov     r0, sp        
	bl      sm_from_nsec        
	cmp     r0, #0        
	beq     .sm_ret_to_nsec        
	add     sp, sp, #(SM_CTX_SEC + SM_SEC_CTX_R0)
```
## 2.3 ARMv8 SCR寄存器的设定
首先ATF在bl31/interrupt_mgmt.h下分别定义了Secure EL1、NonSecure以及EL3模式下Group0和Group1中断的路由模式。
![](https://raw.githubusercontent.com/carloscn/images/main/typora20221005205802.png)
![](https://raw.githubusercontent.com/carloscn/images/main/typora20221005205848.png)

```C
/* 以下分别定义了在EL3/SEL1/NS模式下中断的路由模式,定义名称格式如下
 * RM:   Routing Model
 * SEL1: Secure EL1 Mode(optee os)
 * NS:   Non Secure Mode
 * 0:    Routing Model 0
 * 1:    Routing Model 1
 * 值对应SCR寄存器中的bit[2:0],定义如下
 * bit[0]: SCR.NS (0: Secure, 1: Non Secure)
 * bit[1]: SCR.IRQ (0: enter IRQ mode 1: enter EL3 monitor)
 * bit[2]: SCR.FIQ (0: enter FIQ mode 1: enter EL3 monitor) */
 /* 从NS进入EL3.并在安全态的EL1进行处理 */
#define INTR_SEL1_VALID_RM0        0x2
/* 从NS或者安全态进入EL3 */ 
#define INTR_SEL1_VALID_RM1        0x3
/* 从NS进入EL1/EL2并转切到安全态的EL1 */
#define INTR_NS_VALID_RM0        0x0
/* 从NS到EL1/EL2或从安全态进入EL3
#define INTR_NS_VALID_RM1        0x1
/* 从NS进入EL3，并进入安全态的EL1，最终进入EL3 */
#define INTR_EL3_VALID_RM0        0x2
/* 从NS或安全态进入EL3 */
#define INTR_EL3_VALID_RM1        0x3 
/* 默认模式转移路径 */
#define INTR_DEFAULT_RM        0x0
```

为兼容GICv2和GICv3平台，在初始化CPU时将IRQ和FIQ位同时设置为1，设置相关代码如下：
```c
/*******************************************************************************
 * Handler called when the CPU power domain is about to enter standby.
 ******************************************************************************/
void css_cpu_standby(plat_local_state_t cpu_state)
{
	unsigned int scr;
	assert(cpu_state == ARM_LOCAL_STATE_RET);
	scr = read_scr_el3();
	/*
	 * Enable the Non secure interrupt to wake the CPU.
	 * In GICv3 affinity routing mode, the non secure group1 interrupts use
	 * the PhysicalFIQ at EL3 whereas in GICv2, it uses the PhysicalIRQ.
	 * Enabling both the bits works for both GICv2 mode and GICv3 affinity
	 * routing mode.
	 */
	/*对于非安全中断，如果当前CPU运行在EL3，对于GICv3非安全group*/
	write_scr_el3(scr | SCR_IRQ_BIT | SCR_FIQ_BIT);
	isb();
	dsb();
	// 等待非安全中断触发
	wfi();
	/*
	 * Restore SCR to the original value, synchronisation of scr_el3 is
	 * done by eret while el3_exit to save some execution cycles.
	 */
	// 恢复SCR寄存器的原始值
	write_scr_el3(scr);
}
```
CPU初始化过程中会调用register_interrupt_type_handler来设定Secure EL1下的SCR寄存器，其内容如下：
```C
	case TEESMC_OPTEED_RETURN_ENTRY_DONE:
		/*
		 * Stash the OPTEE entry points information. This is done
		 * only once on the primary cpu
		 */
		assert(optee_vector_table == NULL);
		optee_vector_table = (optee_vectors_t *) x1;

		if (optee_vector_table) {
			set_optee_pstate(optee_ctx->state, OPTEE_PSTATE_ON);

			/*
			 * OPTEE has been successfully initialized.
			 * Register power management hooks with PSCI
			 */
			 // optee初始成功，安装psci处理函数
			psci_register_spd_pm_hook(&opteed_pm);

			/*
			 * Register an interrupt handler for S-EL1 interrupts
			 * when generated during code executing in the
			 * non-secure state.
			 */
			 // 设置flag为ON_SECURE定义为1
			flags = 0;
			set_interrupt_rm_flag(flags, NON_SECURE);
			rc = register_interrupt_type_handler(INTR_TYPE_S_EL1,
						opteed_sel1_interrupt_handler,
						flags);
			if (rc)
				panic();
		}

		/*
		 * OPTEE reports completion. The OPTEED must have initiated
		 * the original request through a synchronous entry into
		 * OPTEE. Jump back to the original C runtime context.
		 */
		opteed_synchronous_sp_exit(optee_ctx, x1);
		break;
```
register_interrypt_type_handler函数会调用set_routing_model来定义三种不同目标的中断在EL3和EL1的SCR寄存器的值，该函数内容如下：
```C
/*******************************************************************************
 * This function validates the routing model specified in the 'flags' and
 * updates internal data structures to reflect the new routing model. It also
 * updates the copy of SCR_EL3 for each security state with the new routing
 * model in the 'cpu_context' structure for this cpu.
 ******************************************************************************/
int32_t set_routing_model(uint32_t type, uint32_t flags)
{
	int32_t rc;
	/* 检查将要设定的SCR的值是否是之前interrupt_mgmt.h中预定义的有效值 */
	rc = validate_interrupt_type(type);
	if (rc != 0)
		return rc;
	/* 结构体变量intr_type_descs用来描述安全/正常模式下SCR的设定 */
	rc = validate_routing_model(type, flags);
	if (rc != 0)
		return rc;

	/* Update the routing model in internal data structures */
	intr_type_descs[type].flags = flags;
	/* 结构体变量intr_type_descs用来描述安全/正常模式下SCR的设定 */
	set_scr_el3_from_rm(type, flags, SECURE);
	/* 设置在CPU正常模式下(SCR.NS=1)的SCR.IRQ、SCR.FIQ位 */
	set_scr_el3_from_rm(type, flags, NON_SECURE);

	return 0;
}

```
set_routing_model来定义三种不同目标的中断在EL3和EL1的SCR寄存器的值，该函数内容如下：
```C
/*******************************************************************************
 * This function uses the 'interrupt_type_flags' parameter to obtain the value
 * of the trap bit (IRQ/FIQ) in the SCR_EL3 for a security state for this
 * interrupt type. It uses it to update the SCR_EL3 in the cpu context and the
 * 'intr_type_desc' for that security state.
 ******************************************************************************/
static void set_scr_el3_from_rm(uint32_t type,
				uint32_t interrupt_type_flags,
				uint32_t security_state)
{
	uint32_t flag, bit_pos;
	/*   
	 * 这里根据security_state状态来获取对应SCR要设定的值   
	 * 如果之前调用的是set_interrupt_rm_flag(flags, NON_SECURE)   
	 * 1. security_state == SECUREflag = (0xb10 >> SECURE) & 0xb1  = 0  
	 * 2. security_state == NONSECURE: flag = (0xb10 >> NONSECURE) & 0xb1    
	 * 如果之前调用的是set_interrupt_rm_flag(flags, SECURE) 同理   
	 * 1. security_state == SECURE: flag = 1   
	 * 2. security_state == NONSECURE: flag = 0    */
	flag = get_interrupt_rm_flag(interrupt_type_flags, security_state);
	/* 这个函数根据GIC的版本决定设定SCR寄存器中FIQ/IRQ位 */
	bit_pos = plat_interrupt_type_to_line(type, security_state);
	intr_type_descs[type].scr_el3[security_state] = (u_register_t)flag << bit_pos;

	/*
	 * Update scr_el3 only if there is a context available. If not, it
	 * will be updated later during context initialization which will obtain
	 * the scr_el3 value to be used via get_scr_el3_from_routing_model()
	 */
/* 如果当前上下文有效则可以在这里直接更新scr_el3否则将要设定的SCR的值保存在
 * intr_type_descs中,之后通过get_scr_els3_from_routing_model()函数来获取并
 * 写入SCR寄存器中*/
	if (cm_get_context(security_state) != NULL)
		cm_write_scr_el3_bit(security_state, bit_pos, flag);
}
```

## 2.4 GICv2架构
GICv2设定**Group0为安全中断，Group1为非安全中断**。中断号属于哪个Group是由其在GICD_IGROUPRn寄存器中的值来决定的。当GIC接收到中断信号后，如果中断属于Group0则发送IRQ信号到目标CPU，中断属于Group1则发送FIQ信号到目标CPU。

plat_interrupt_type_to_line（type，security_state）在GICv2下的实现如下：
```C 
uint32_t plat_interrupt_type_to_line(uint32_t type,
                                     uint32_t security_state) {
    assert(type == INTR_TYPE_S_EL1 ||    type == INTR_TYPE_EL3 ||    type == INTR_TYPE_NS);
    /* NonSecure中断发IRQ信号,设置SCR.IRQ = 1*/
    if (type == INTR_TYPE_NS)
        return __builtin_ctz(SCR_IRQ_BIT);
    /*    * 两种情况
     * (1) FIQ disabled: 安全中断(Group0)会产生IRQ中断信号设置SCR.IRQ=1
     * (2) FIQ enabled:  安全中断(Group1)会产生FIQ中断信号设置SCR.FIQ=1    */
    return ((gicv2_is_fiq_enabled()) ? __builtin_ctz(SCR_FIQ_BIT) :
                                       __builtin_ctz(SCR_IRQ_BIT));
}
```

## 2.5 GICv3架构
与GICv2相比GICv3的主要改进有以下几点：

* 在软件中断（SGI）方面新增了中断目标路由模式（affinity routing），SGI中断能支持更大范围的CPU ID。
* GICv3对Group1的中断类型进行了进一步的细分。Group0中断和GICv2一样为安全中断（以下用G0S表示）且拥有最高的优先级，而Group1中断又分为Group1非安全中断（以下用G1NS表示）和Group1安全中断（以下用G1S表示）。
* GIC的CPU接口寄存器（GICC）不再需要地址映射，可以直接通过系统寄存器访问。
* 在IRQ/FIQ都使能的情况下，属于Group0的中断始终会触发FIQ信号，而属于Group1的中断则根据CPU当前工作模式和中断类型（secure/nonsecure）分别触发FIQ或者IRQ信号。

为EL3在AArch64模式下GICv3对不同中断的处理：

| 当前处理器模式      | Group0 | Group1 | Group1 |
| ------------------- | ------ | ------ | ---- |
|                     |        | G1S    | G1NS |
| Secure EL1/EL0      | FIQ    | **IRQ**    | FIQ  |
| Non Secure  EL1/EL0 | FIQ    | FIQ    | **IRQ**  |
| Secure EL3          | FIQ    | FIQ    | FIQ  |

当处理器接收的中断类型（secure/non secure）和当前处理器工作模式（secure/non-secure）不一致时，**GIC会发送FIQ中断信号否则会发出IRQ**中断信号。

plat_interrupt_type_to_line（type，security_state）在GICv3下的内容如下：

```C
uint32_t plat_interrupt_type_to_line(uint32_t type,
                                     uint32_t security_state) {
    assert(type == INTR_TYPE_S_EL1 ||    type == INTR_TYPE_EL3 ||    type == INTR_TYPE_NS);
    assert(sec_state_is_valid(security_state));
    assert(IS_IN_EL3());
    switch (type) {
        case INTR_TYPE_S_EL1:
            /*            
             * 当安全中断G1S在S-EL1发生IRQ中断,设置SCR.IRQ=1
             * 当安全中断G1S在NS发生FIQ中断,设置SCR.FIQ=1
             */
            if (security_state == SECURE)
                return __builtin_ctz(SCR_IRQ_BIT);
            else
                return __builtin_ctz(SCR_FIQ_BIT);
        case INTR_TYPE_NS:
            /*
             * 当非安全中断G1NS在NS发生IRQ中断,设置SCR.IRQ=1
             * 当非安全中断在S-EL1发生FIQ中断,设置SCR.FIQ=1
             */
            if (security_state == SECURE)
                return __builtin_ctz(SCR_FIQ_BIT);
            else
                return __builtin_ctz(SCR_IRQ_BIT);
        default:
            assert(0);
        case INTR_TYPE_EL3:
            /*无论在S-EL1还是在NS-EL1,目标为EL3的中断都是FIQ*/
            return __builtin_ctz(SCR_FIQ_BIT);
    }
}
```

# 3. 异常配置
REE侧、TEE侧以及Monitor模式或EL3都可接收中断信号。在系统中存在两个VBAR寄存器和一个MVBAR寄存器，
* **REE侧的VBAR寄存器**中存放的是Linux内核的异常向量表基地址；
* **OP-TEE中的VBAR**寄存器存放的是OP-TEE系统的中断向量表基地址；
* **Monitor或者EL3的MVBAR**存放的是Monitor模式或EL3运行时的中断向量表基地址，即在Monitor或者EL3阶段是可以接收外部中断信号的。

本节将介绍**OP-TEE中断的配置和Monitor或EL3阶段中断的配置**。

EE侧的VBAR寄存器中存放的是Linux内核的异常向量表基地址，可以参考：[10_ARMv8_异常处理（一） - 入口与返回、栈选择、异常向量表](https://github.com/carloscn/blog/issues/47)

## 3.1 ARM中的异常向量表
在介绍OP-TEE的异常向量表之前，先来介绍一下ARM中的异常向量表如何配置的。ARMv7是单独的Monitor的异常向量表，ARMv8是在EL3模式下进行配置的异常向量表。

### 3.1.1 ARMv7中Monitor模式的异常向量表
ARMv7架构在ARM扩展出了Monitor模式，Monitor模式属于安全世界状态，用于实现ARM核安全世界状态与正常世界状态之间的切换，且**该模式具有独立的中断向量表**。使用MVBAR寄存器来保存该运行模式的中断向量表的基地址。在OPTEE初始化过程中会调用sm_init函数来初始化Monitor模式的配置，并将Monitor模式的中断向量基地址写入到MVBAR寄存器中，该函数内容如下：
```assembly
FUNC sm_init , :
UNWIND( .fnstart)
    mrs r1, cpsr //设置Monitor模式使用的栈
    cps #CPSR_MODE_MON
    sub sp, r0, #(SM_CTX_SIZE - SM_CTX_NSEC)
    msr cpsr, r1
    //将Monitor模式的异常向量表地址保存到r0寄存器
    ldr r0, =sm_vect_table     
    //将Monitor模式的异常向量表基地址写入MVBAR寄存器中
    write_mvbar r0
    bx  lr // 返回
END_FUNC sm_init
```
sm_init函数中写入MVBAR寄存器中的值即是Monitor模式下的异常向量表的基地址—— sm_vect_table，该向量表的内容如下：
```assembly
UNWIND( .fnstart)
UNWIND( .cantunwind)
    b   .		/* 重启操作 */
    b   .		/* 未定义指令操作 */
    b   sm_smc_entry	/* smc异常处理函数 */
    b   .	/* 执行时的abort操作 */
    b   .	/* 数据abort操作 */
    b   .	/* 预留 */
    b   .	/* IRQ事件 */
    b   sm_fiq_entry /* FIQ中断处理入口函数 */
UNWIND( .fnend)
END_FUNC sm_vect_table        
```

从上述异常向量表中可知，当在Monitor模式下接收到FIQ中断时，系统将会调用sm_fiq_entry函数对该FIQ中断进行处理。

### 3.2.1 ARMv8中EL3阶段的异常向量表
ARMv8使用ATF中的bl31作为EL3阶段的代码，其作用与ARMv7中Monitor模式下运行的代码作用一致。在ATF的启动过程中，bl31通过调用el3_entrypoint_common函数来进行EL3运行环境的初始化，在初始化过程中会执行EL3阶段异常向量表的初始化，EL3的异常向量表的基地址为runtime_exception_vectors。EL3异常向量表的内容如下：
```assembly
vector_base runtime_exceptions

	/* ---------------------------------------------------------------------
	 * Current EL with SP_EL0 : 0x0 - 0x200
	 * ---------------------------------------------------------------------
	 */
vector_entry sync_exception_sp_el0
#ifdef MONITOR_TRAPS
	stp x29, x30, [sp, #-16]!

	mrs	x30, esr_el3
	ubfx	x30, x30, #ESR_EC_SHIFT, #ESR_EC_LENGTH

	/* Check for BRK */
	cmp	x30, #EC_BRK
	b.eq	brk_handler

	ldp x29, x30, [sp], #16
#endif /* MONITOR_TRAPS */

	/* We don't expect any synchronous exceptions from EL3 */
	b	report_unhandled_exception
end_vector_entry sync_exception_sp_el0

vector_entry irq_sp_el0
	/*
	 * EL3 code is non-reentrant. Any asynchronous exception is a serious
	 * error. Loop infinitely.
	 */
	b	report_unhandled_interrupt
end_vector_entry irq_sp_el0


vector_entry fiq_sp_el0
	b	report_unhandled_interrupt
end_vector_entry fiq_sp_el0


vector_entry serror_sp_el0
	no_ret	plat_handle_el3_ea
end_vector_entry serror_sp_el0

	/* ---------------------------------------------------------------------
	 * Current EL with SP_ELx: 0x200 - 0x400
	 * ---------------------------------------------------------------------
	 */
vector_entry sync_exception_sp_elx
	/*
	 * This exception will trigger if anything went wrong during a previous
	 * exception entry or exit or while handling an earlier unexpected
	 * synchronous exception. There is a high probability that SP_EL3 is
	 * corrupted.
	 */
	b	report_unhandled_exception
end_vector_entry sync_exception_sp_elx

vector_entry irq_sp_elx
	b	report_unhandled_interrupt
end_vector_entry irq_sp_elx

vector_entry fiq_sp_elx
	b	report_unhandled_interrupt
end_vector_entry fiq_sp_elx

vector_entry serror_sp_elx
#if !RAS_EXTENSION
	check_if_serror_from_EL3
#endif
	no_ret	plat_handle_el3_ea
end_vector_entry serror_sp_elx

	/* ---------------------------------------------------------------------
	 * Lower EL using AArch64 : 0x400 - 0x600
	 * ---------------------------------------------------------------------
	 */
vector_entry sync_exception_aarch64
	/*
	 * This exception vector will be the entry point for SMCs and traps
	 * that are unhandled at lower ELs most commonly. SP_EL3 should point
	 * to a valid cpu context where the general purpose and system register
	 * state can be saved.
	 */
	apply_at_speculative_wa
	check_and_unmask_ea
	handle_sync_exception
end_vector_entry sync_exception_aarch64

vector_entry irq_aarch64
	apply_at_speculative_wa
	check_and_unmask_ea
	handle_interrupt_exception irq_aarch64
end_vector_entry irq_aarch64

vector_entry fiq_aarch64
	apply_at_speculative_wa
	check_and_unmask_ea
	handle_interrupt_exception fiq_aarch64
end_vector_entry fiq_aarch64

vector_entry serror_aarch64
	apply_at_speculative_wa
#if RAS_EXTENSION
	msr	daifclr, #DAIF_ABT_BIT
	b	enter_lower_el_async_ea
#else
	handle_async_ea
#endif
end_vector_entry serror_aarch64

	/* ---------------------------------------------------------------------
	 * Lower EL using AArch32 : 0x600 - 0x800
	 * ---------------------------------------------------------------------
	 */
vector_entry sync_exception_aarch32
	/*
	 * This exception vector will be the entry point for SMCs and traps
	 * that are unhandled at lower ELs most commonly. SP_EL3 should point
	 * to a valid cpu context where the general purpose and system register
	 * state can be saved.
	 */
	apply_at_speculative_wa
	check_and_unmask_ea
	handle_sync_exception
end_vector_entry sync_exception_aarch32

vector_entry irq_aarch32
	apply_at_speculative_wa
	check_and_unmask_ea
	handle_interrupt_exception irq_aarch32
end_vector_entry irq_aarch32

vector_entry fiq_aarch32
	apply_at_speculative_wa
	check_and_unmask_ea
	handle_interrupt_exception fiq_aarch32
end_vector_entry fiq_aarch32

vector_entry serror_aarch32
	apply_at_speculative_wa
#if RAS_EXTENSION
	msr	daifclr, #DAIF_ABT_BIT
	b	enter_lower_el_async_ea
#else
	handle_async_ea
#endif
end_vector_entry serror_aarch32      
```

从异常向量表来看，ARMv8架构中不管是AArch32还是AArch64，当在EL3阶段产生了FIQ事件或者IRQ事件后，bl31将会调用handle_interrupt_exception宏来处理，该宏使用的参数就是产生的异常的标签。

## 3.2 OP-TEE异常向量的配置
在初始化阶段，OP-TEE异常向量的加载和配置会通过执行`thread_init_vbar`函数来实现，从初始化起始到配置异常向量表的整个调用过程如图所示：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221006102613.png" width="80%" /></div>

thread_init_vbar函数在AArch32位系统中的定义如下：
```assembly
FUNC thread_init_vbar , :
	/* Set vector (VBAR) */
	write_vbar r0
	bx	lr
END_FUNC thread_init_vbar
DECLARE_KEEP_PAGER thread_init_vbar
```

thread_init_vbar函数在AArch64位系统中的定义如下：
```assembly
FUNC thread_init_vbar , :
	msr	vbar_el1, x0
	ret
END_FUNC thread_init_vbar
DECLARE_KEEP_PAGER thread_init_vbar
```

OP-TEE的AArch32异常向量表：
```assembly
FUNC thread_excp_vect , :, align=32
UNWIND(	.cantunwind)
	b	.			/* Reset			*/
	b	__thread_und_handler	/* Undefined instruction	*/
	b	__thread_svc_handler	/* System call			*/
	b	__thread_pabort_handler	/* Prefetch abort		*/
	b	__thread_dabort_handler	/* Data abort			*/
	b	.			/* Reserved			*/
	b	__thread_irq_handler	/* IRQ				*/
	b	__thread_fiq_handler	/* FIQ				*/
#ifdef CFG_CORE_WORKAROUND_SPECTRE_BP_SEC
```

AArch64的异常向量表：
`core/arch/arm/kernel/thread_a64.S:266`
```assmebly
FUNC thread_excp_vect , : , default, 2048, nobti
	/* -----------------------------------------------------
	 * EL1 with SP0 : 0x0 - 0x180
	 * -----------------------------------------------------
	 */
	.balign	128, INV_INSN
el1_sync_sp0:
	store_xregs sp, THREAD_CORE_LOCAL_X0, 0, 3
	b	el1_sync_abort
	check_vector_size el1_sync_sp0

	.balign	128, INV_INSN
el1_irq_sp0:
	store_xregs sp, THREAD_CORE_LOCAL_X0, 0, 3
	b	elx_irq
	check_vector_size el1_irq_sp0

	.balign	128, INV_INSN
el1_fiq_sp0:
	store_xregs sp, THREAD_CORE_LOCAL_X0, 0, 3
	b	elx_fiq
	check_vector_size el1_fiq_sp0

	.balign	128, INV_INSN
el1_serror_sp0:
	b	el1_serror_sp0
	check_vector_size el1_serror_sp0

	/* -----------------------------------------------------
	 * Current EL with SP1: 0x200 - 0x380
	 * -----------------------------------------------------
	 */
	.balign	128, INV_INSN
el1_sync_sp1:
	b	el1_sync_sp1
	check_vector_size el1_sync_sp1

	.balign	128, INV_INSN
el1_irq_sp1:
	b	el1_irq_sp1
	check_vector_size el1_irq_sp1

	.balign	128, INV_INSN
el1_fiq_sp1:
	b	el1_fiq_sp1
	check_vector_size el1_fiq_sp1

	.balign	128, INV_INSN
el1_serror_sp1:
	b	el1_serror_sp1
	check_vector_size el1_serror_sp1

	/* -----------------------------------------------------
	 * Lower EL using AArch64 : 0x400 - 0x580
	 * -----------------------------------------------------
	 */
	.balign	128, INV_INSN
el0_sync_a64:
	restore_mapping
	/* PAuth will be disabled later else check_vector_size will fail */

	b	el0_sync_a64_finish
	check_vector_size el0_sync_a64

	.balign	128, INV_INSN
el0_irq_a64:
	restore_mapping
	disable_pauth x1

	b	elx_irq
	check_vector_size el0_irq_a64

	.balign	128, INV_INSN
el0_fiq_a64:
	restore_mapping
	disable_pauth x1

	b	elx_fiq
	check_vector_size el0_fiq_a64

	.balign	128, INV_INSN
el0_serror_a64:
	b   	el0_serror_a64
	check_vector_size el0_serror_a64

	/* -----------------------------------------------------
	 * Lower EL using AArch32 : 0x0 - 0x180
	 * -----------------------------------------------------
	 */
	.balign	128, INV_INSN
el0_sync_a32:
	restore_mapping

	b 	el0_sync_a32_finish
	check_vector_size el0_sync_a32

	.balign	128, INV_INSN
el0_irq_a32:
	restore_mapping

	b	elx_irq
	check_vector_size el0_irq_a32

	.balign	128, INV_INSN
el0_fiq_a32:
	restore_mapping

	b	elx_fiq
	check_vector_size el0_fiq_a32

	.balign	128, INV_INSN
el0_serror_a32:
	b	el0_serror_a32
	check_vector_size el0_serror_a32

#if defined(CFG_CORE_WORKAROUND_SPECTRE_BP_SEC)
	.macro invalidate_branch_predictor
		store_xregs sp, THREAD_CORE_LOCAL_X0, 0, 3
		mov_imm	x0, SMCCC_ARCH_WORKAROUND_1
		smc	#0
		load_xregs sp, THREAD_CORE_LOCAL_X0, 0, 3
	.endm

	.balign	2048, INV_INSN
	.global thread_excp_vect_wa_spectre_v2
thread_excp_vect_wa_spectre_v2:
	/* -----------------------------------------------------
	 * EL1 with SP0 : 0x0 - 0x180
	 * -----------------------------------------------------
	 */
	.balign	128, INV_INSN
wa_spectre_v2_el1_sync_sp0:
	b	el1_sync_sp0
	check_vector_size wa_spectre_v2_el1_sync_sp0

	.balign	128, INV_INSN
wa_spectre_v2_el1_irq_sp0:
	b	el1_irq_sp0
	check_vector_size wa_spectre_v2_el1_irq_sp0

	.balign	128, INV_INSN
wa_spectre_v2_el1_fiq_sp0:
	b	el1_fiq_sp0
	check_vector_size wa_spectre_v2_el1_fiq_sp0

	.balign	128, INV_INSN
wa_spectre_v2_el1_serror_sp0:
	b	el1_serror_sp0
	check_vector_size wa_spectre_v2_el1_serror_sp0

	/* -----------------------------------------------------
	 * Current EL with SP1: 0x200 - 0x380
	 * -----------------------------------------------------
	 */
	.balign	128, INV_INSN
wa_spectre_v2_el1_sync_sp1:
	b	wa_spectre_v2_el1_sync_sp1
	check_vector_size wa_spectre_v2_el1_sync_sp1

	.balign	128, INV_INSN
wa_spectre_v2_el1_irq_sp1:
	b	wa_spectre_v2_el1_irq_sp1
	check_vector_size wa_spectre_v2_el1_irq_sp1

	.balign	128, INV_INSN
wa_spectre_v2_el1_fiq_sp1:
	b	wa_spectre_v2_el1_fiq_sp1
	check_vector_size wa_spectre_v2_el1_fiq_sp1

	.balign	128, INV_INSN
wa_spectre_v2_el1_serror_sp1:
	b	wa_spectre_v2_el1_serror_sp1
	check_vector_size wa_spectre_v2_el1_serror_sp1
```
当系统处于OP-TEE中时，系统会到VBAR寄存器中获取OP-TEE的异常向量表基地址，然后根据异常类型获取到FIQ或IRQ事件的处理函数，并对不同的事件进行处理。针对不同的事件会调用线程向量表thread_vector_table变量中对应的处理函数来完成对该异常事件的处理。

# 4. 线程向量表
在OP-TEE中会定义一个用于保存各种事件处理函数的线程向量表，**该线程向量表中的成员是OP-TEE对fast smc、std smc、FIQ事件、CPU关闭和打开以及系统关机和重启事件的处理函数**。

OP-TEE的AArch32线程向量表：
```assembly
/*
 * Vector table supplied to ARM Trusted Firmware (ARM-TF) at
 * initialization.  Also used when compiled with the internal monitor, but
 * the cpu_*_entry and system_*_entry are not used then.
 *
 * Note that ARM-TF depends on the layout of this vector table, any change
 * in layout has to be synced with ARM-TF.
 */
FUNC thread_vector_table , : , .identity_map
UNWIND(	.cantunwind)
	b	vector_std_smc_entry
	b	vector_fast_smc_entry
	b	vector_cpu_on_entry
	b	vector_cpu_off_entry
	b	vector_cpu_resume_entry
	b	vector_cpu_suspend_entry
	b	vector_fiq_entry
	b	vector_system_off_entry
	b	vector_system_reset_entry
END_FUNC thread_vector_table
DECLARE_KEEP_PAGER thread_vector_table
#endif /*if defined(CFG_WITH_ARM_TRUSTED_FW)*/
```

AArch64的线程向量表：
```assmebly
/*
 * Vector table supplied to ARM Trusted Firmware (ARM-TF) at
 * initialization.
 *
 * Note that ARM-TF depends on the layout of this vector table, any change
 * in layout has to be synced with ARM-TF.
 */
FUNC thread_vector_table , : , .identity_map, , nobti
	b	vector_std_smc_entry
	b	vector_fast_smc_entry
	b	vector_cpu_on_entry
	b	vector_cpu_off_entry
	b	vector_cpu_resume_entry
	b	vector_cpu_suspend_entry
	b	vector_fiq_entry
	b	vector_system_off_entry
	b	vector_system_reset_entry
END_FUNC thread_vector_table
DECLARE_KEEP_PAGER thread_vector_table
```
每一个函数中都定义了异常处理的程序：
![](https://raw.githubusercontent.com/carloscn/images/main/typora20221006103329.png)

ARMv8架构中，该线程向量表的地址会被返回给bl31，以备EL3接收到安全监控模式调用或FIQ事件时可使用该变量中的处理函数对请求和异常事件进行进一步的处理。在ARMv7架构中，该变量会被Monitor模式下运行的程序使用，用于处理安全监控模式调用和FIQ事件。

## 5. 全局handle变量的初始化

ARMv8架构中会将thread_vector_table的地址返回给ATF的bl31，用于处理安全监控模式调用、FIQ事件以及CPU和系统的相关操作，而在ARMv7中则会被Monitor模式的异常向量表使用。通过对该thread_vector_table的基地址进行偏移计算来获得安全监控模式调用、FIQ事件以及CPU和系统的相关处理函数的实际地址，然后调用获得的地址指向的函数来处理上述事件。

thread_vector_table变量中的函数都是使用汇编来实现的，当异常事件发生时会调用各自对应的处理函数对事件进行处理，处理函数的名字类似于thread_xxx_xxx_handler_ptr。这些变量都是函数指针，在OP-TEE启动时，通过调用init_handlers函数来实现对这些全局函数指针变量进行赋值。

当在ARMv7或ARMv8中产生了FIQ事件后，将会调用main_fiq函数来处理FIQ事件。

# 6. FIQ处理

## 6.1 ARMv7 Monitor对FIQ事件的处理
当在Monitor模式下出现了FIQ事件时，系统会从MVBAR寄存器中获取到异常向量表的基地址，并查找到FIQ事件的处理函数——sm_fiq_entry。该函数即为Monitor模式下对FIQ事件的处理函数，Monitor模式下对FIQ的处理过程如图所示[^1]。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221006112709.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221006114054.png)

## 6.2 ARMv8 EL3阶段对FIQ事件的处理
ARMv8架构中通过查看EL3的异常向量可知，在EL3阶段是通过调用handle_interrupt_exception宏对FIQ事件进行处理的，最终该宏会将FIQ事件转发给OP-TEE，由OP-TEE来完成对FIQ事件的处理，并指定OP-TEE提供的线程向量表中的fiq_entry作为处理该事件的入口函数。在EL3中对FIQ事件的处理过程如图所示。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221006114317.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221006114426.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221006114443.png)

## 6.3 OP-TEE对FIQ事件的处理
OP-TEE启动时会调用thread_init_vbar函数来完成安全世界状态（SWS）的中断向量表的初始化，且在GIC中配置FIQ在安全世界状态时才有效。所以在安全世界状态中产生了FIQ事件时，CPU将直接通过VBAR寄存器查找到中断向量表的基地址，并命中FIQ的处理函数。整个处理过程如图所示。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221006114705.png)

## 6.4 OP-TEE对IRQ事件的处理
IRQ事件的处理一般会用在REE侧。但当ARM核处于安全世界状态时，系统产生了IRQ事件，而该事件又不能被暴力的作为无用事件而轻易丢弃，系统还是需要响应并执行相关操作的。针对该情况的处理方式和逻辑如图所示。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221006114824.png)

在系统初始化时，系统会调用thread_init_vbar函数来初始化安全世界状态的中断向量表并将中断向量的基地址保存到VBAR寄存器中。当系统在ARM核处于安全世界状态中产生IRQ事件时，系统通过VBAR寄存器获取到中断向量表的基地址，然后查找到IRQ对应的中断处理函数—— thread_irq_handler，使用该函数处理IRQ事件，整个处理过程的流程如图所示。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221006114913.png)

OP-TEE接收到IRQ事件后，ARMv7架构中会通过切换到Monitor模式将该IRQ事件发送到REE侧进行处理，ARMv8架构中IRQ中断事件会通过切换到EL3将该IRQ事件发送到REE侧进行处理。

OP-TEE接收IRQ事件后会触发安全监控模式调用（smc），在ARMv7中将会进入安全监控模式调用（smc）的处理过程，即进入sm_smc_entry函数中进行处理。

当Monitor模式将IRQ事件传递到正常世界状态后，Linux将根据具体得到的参数执行对该IRQ事件的具体处理。完成对IRQ事件的处理后，会触发安全监控模式调用重新切回到Monitor态，然后恢复安全世界状态中被中断的线程的状态继续执行。对于ARMv8，该部分的处理逻辑类似，在此不再赘述，详细部分可查看ATF中bl31部分的代码。

# 7. 总结
ARMv7架构中安全世界状态包含Monitor模式和OP-TEE，而在ARMv8架构中安全世界状态则包含EL3阶段和OP-TEE。Monitor模式或EL3阶段对FIQ的处理都是通过调用OP-TEE在初始化时赋值的处理函数来实现的。对于ARMv7，该处理函数的指针最终会被Monitor模式下运行的代码用来处理FIQ中断事件，而对于ARMv8，该处理函数的地址会被返回给ATF的bl31，当在EL3中接收到FIQ事件时，EL3会使用该处理函数来处理该FIQ事件。FIQ的处理都是通过调用OP-TEE中的main_fiq函数来完成的，由于CPU和板级配置不同，该函数的实现也各不相同。在OP-TEE中接收到IRQ事件时，OP-TEE会将IRQ中断事件转发给Monitor模式或EL3进行处理，Monitor模式或者EL3最终会将IRQ事件发送到REE侧，REE侧处理完该IRQ事件后会触发安全监控模式调用恢复到安全世界状态中继续执行。

[^1][^2]
# Ref

[^1]:[Entry and exit of secure world](https://review.trustedfirmware.org/plugins/gitiles/OP-TEE/optee_os/+/bc62d278d25b4b6386305b72bc6ce4fc9d065067/documentation/interrupt_handling.md)
[^2]:[Interrupt Management Framework](https://trustedfirmware-a.readthedocs.io/en/latest/design/interrupt-framework-design.html#interrupt-management-framework "Permalink to this headline")
