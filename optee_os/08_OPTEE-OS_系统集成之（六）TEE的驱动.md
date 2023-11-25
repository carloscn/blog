# 08_OPTEE-OS_系统集成之（六）TEE的驱动
O**P-TEE驱动是REE侧与TEE侧之间进行交互的重要通道**，在REE侧的CA接口以及RPC请求的接收和结果的返回最终都会被发送到驱动中，由驱动对数据做进一步的处理。OP-TEE驱动通过解析传入的参数，重新组合数据，**将需要被传入到TEE侧的数据载入到共享内存中**，**触发安全监控模式调用（smc）进入到Monitor模式或EL3中将数据发送给TEE**。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typoratypora20221005185005.png" width="80%" /></div>

# 1. 编译

OP-TEE的驱动通过**subsys_initcall和module_init**宏来告知系统在初始化阶段的什么时候去加载OPTEE驱动。subsys_initcall定义在linux/include/init.h文件中。这是Linux内核提供的一系列init的操作集合中的一个。该方法利用Linux内核段属性机制。

>subsys_initcall是一个宏，定义在`linux/init.h`中。经过对这个宏进行展开，发现这个宏的功能是：将其声明的函数放到一个特定的段：`.initcall4.init`
>
>```C
>subsys_initcall
>    __define_initcall("4",fn,4)
>```
>
>以下文件在/include/linux/init.h：
>
><div align='left'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora%E5%8D%9A%E5%AE%A2%E5%9B%BE%E7%89%87008.png" width="80%" /></div>
>
>分析module_init宏，可以看出它将函数放到`.initcall6.init`段
>
>```C
>module_init
>    __initcall
>        device_initcall
>            __define_initcall("6",fn,6)
>```
>
>打开编译过的内核源码树中的的/arch/arm/kernel/vmlinux.lds文件(没编译没有这个文件):
>
>```bash
>SECTIONS
>{
> . = 0xC0000000 + 0x00008000;
> .init : { /* Init code and data                */
>  _stext = .;
>  _sinittext = .;
>   *(.head.text)
>   *(.init.text) *(.cpuinit.text) *(.meminit.text)
>  ......
>  . = ALIGN(16); __setup_start = .; *(.init.setup) __setup_end = .;
>  __initcall_start = .; *(.initcallearly.init) __early_initcall_end = .;
> *(.initcall0.init) *(.initcall0s.init)  *(.initcall1.init) *(.initcall1s.init) 
> *(.initcall2.init) ... __initcall_end = .;
>  ...... 
>```
>
>内核在启动过程中需要顺序的做很多事，内核如何实现按照先后顺序去做很多初始化操作。内核的解决方案就是给内核启动时要调用的所有函数归类，执行内核某一个函数然后每个类就会按照一定的次序被调用执行。这些分类名就叫.initcallx.init。x的值从1到8。**内核开发者在编写内核代码时只要将函数设置合适的级别，这些函数就会被链接的时候放入特定的段，内核启动时再按照段顺序去依次执行各个段即可**（通过某一个函数，链接脚本只是规定了某一程序段在内存中的存放位置）。
>
>**内核源代码**：
>
>以下文件在/init/main.c
>
>```C
>extern initcall_t __initcall_start[], __initcall_end[], __early_initcall_end[];
>
>static void __init do_initcalls(void)
>{
>    initcall_t *fn;
>
>    for (fn = __early_initcall_end; fn < __initcall_end; fn++)
>        do_one_initcall(*fn);
>
>    /* Make sure there is no pending stuff from the initcall sequence */
>    flush_scheduled_work();
>}
>```
>
>执行do_initcalls就会按照设定好的顺序去执行，通过函数的内容可以猜测出其原理就是链接脚本设置好的顺序，然后do_initcalls执行就会去按照链接脚本设置好的顺序一个个遍历。
>

OP-TEE驱动通过subsys_initcall和module_init宏来告知系统在初始化的什么时候去加载OP-TEE驱动，subsys_initcall定义在`linux/include/init.h`文件中，内容如下：

```C
#define __define_initcall(fn, id) \
	static initcall_t __initcall_##fn##id __used \
	__attribute__((__section__(".initcall" #id ".init"))) = fn;
 
#define core_initcall(fn)		__define_initcall(fn, 1)
#define core_initcall_sync(fn)		__define_initcall(fn, 1s)
#define postcore_initcall(fn)		__define_initcall(fn, 2)
#define postcore_initcall_sync(fn)	__define_initcall(fn, 2s)
#define arch_initcall(fn)		__define_initcall(fn, 3)
#define arch_initcall_sync(fn)		__define_initcall(fn, 3s)
#define subsys_initcall(fn)		__define_initcall(fn, 4)
#define subsys_initcall_sync(fn)	__define_initcall(fn, 4s)
#define fs_initcall(fn)			__define_initcall(fn, 5)
#define fs_initcall_sync(fn)		__define_initcall(fn, 5s)
#define rootfs_initcall(fn)		__define_initcall(fn, rootfs)
#define device_initcall(fn)		__define_initcall(fn, 6)
#define device_initcall_sync(fn)	__define_initcall(fn, 6s)
#define late_initcall(fn)		__define_initcall(fn, 7)
#define late_initcall_sync(fn)		__define_initcall(fn, 7s)
```

使用subsys_initcall宏定义的函数最终会被编译到`.initcall4.init`段中，linux系统在启动的时候会执行`initcallx.init`段中的所有内容，而使用`subsys_initcall`宏定义段的执行优先级为4.

`module_init`的定义和相关扩展在`linux/include/linux/module.h`文件和`linux/include/linux/init.h`中，内容如下：

```C
#define device_initcall(fn)		__define_initcall(fn, 6)
#define __initcall(fn) device_initcall(fn)
#define module_init(x)  __initcall(x);
```

由此可见，使用module_init宏构造的函数将会在编译的时候被编译到`initcall6.init`段中，该段在linux系统启动的过程中的优先等级为6.

结合上述两点看，在系统加载OP-TEE驱动的时候，首先**会执行OP-TEE驱动中使用subsys_init定义的函数**，然后再**执行使用module_init定义的函数**。在OP-TEE驱动源代码中使用`subsys_init`定义的函数为`tee_init`，使用`module_init`定义的函数为`optee_driver_init`。

# 2. REE侧OP-TEE驱动的加载
OP-TEE驱动主要作用是REE与TEE端进行数据交互的桥梁作用。**`tee_supplicant`和`libteec`调用接口之后几乎都会首先通过系统调用陷入到kernel space，然后kernel根据传递的参数找到OP-TEE驱动，并命中驱动的operation结构体中的具体处理函数来完成实际的操作，对于OP-TEE驱动，一般都会触发`SMC`调用，并带参数进入到ARM的monitor模式，在monitor模式中对执行normal world和secure world的切换，待状态切换完成之后，会将驱动端带入的参数传递給OP-TEE中的thread进行进一步的处理**。OP-TEE驱动的源代码存放在`linux/drivers/tee`目录中，其内容如下：

```bash
libteec(tee_supplicant)
    -> linux kernel_space
        -> find operation structure 
            -> call `SMC`
	        -> pass params to thread of opteeos. 
```

https://github.com/carloscn/raspi-linux/tree/master/drivers/tee

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221005092357.png" width="80%" /></div>

OP-TEE驱动的加载过程分为两部分：
* 第一部分是创建class和分配设备号；
* 第二部分是probe过程。

首先需要明白两个Linux内核中加载驱动的宏：subsys_initcall和module_init。OP-TEE驱动的第一部分是调用subsys_initcall宏来实现tee_init，而第二部分则是调用module_init宏来实现tee_driver_init。

OP-TEE驱动主要是被期望能够功能实现和数据交换。
* **用于功能**：OPTEE驱动会创建两个设备，分别为/dev/tee0和/dev/teepriv0，这两个设备分别被**libteec库和tee_supplicant使用**，用于实现各自的功能。
* **用于数据交换**：驱动与TEE侧之间的数据传递是通过**共享内存**的方式来完成的，即**在OP-TEE驱动挂载过程中会创建OPTEE与TEE之间的专用共享内存空间**，在Linux的用户空间需要发送到TEE的数据最终都会被保存在该共享内存中，然后再切换ARM核的状态后，OPTEE从该共享内存中去获取数据。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221005100336.png" width="100%" /></div>

## 2.1 tee_init
该函数定义在`linux/drivers/tee/tee_core.c`文件中，主要完成class的创建和设备号的分配，其内容如下：
```C
static int __init tee_init(void){
	int rc;
	//分配OP-TEE驱动的class
	tee_class=class_create(THIS_MODULE,"tee");
	if(IS_ERR(tee_class)){
		pr_err("couldn't create class \n");
		return PTR_ERR(tee_class);
	}
	//分配OP-TEE的设备号
	rc=alloc_chrdev_region(&tee_devt,0,TEE_NUM_DEVICES,"tee");
	if(rc){
		pr_err("failed to allocate char dev region\n");
		class_destory(tee_class);
		tee_class = NULL;
	}
	retrun rc;
}
```
`设备号`和`class`将会在驱动执行`probe`的时候被使用到。

## 2.2 optee_driver_init
linux启动过程中会执行moudule_init宏定义的函数，在OP-TEE驱动的挂载过程中将会执行`optee_driver_ini`t函数，该函数定义在`linux/drivers/tee/optee/core.c`文件中，其内容如下：

```C
static int __init optee_driver_init(void){
	struct device_node* fw_np;
	struct device_node* np;
	struct optee* optee;
	
	//node is supposed to be below /firmware
	//从device tree查找到firmware的节点
	fw_np = of_find_node_by_name(NULL,"firmware");
	if(!fw_np)
		return -ENODEV;
	//匹配device tree 中firmware节点下为linaro，optee-tz内容的节点
	np = of_find_match_node(fw_np, optee_match);
	of_node_put(fw_np);
	if(!np)
		return -ENODEV;
	//使用查找到的节点执行OP-TEE驱动的probe操作
	optee= optee_probe(np);
	of_node_put(np);
	if(IS_ERR(optee))
		return PTR_ERR(optee);
	//保存初始化完成之后OP-TEE设备信息到optee_svc中，以备在卸载时使用
	optee_svc=optee;
	return 0;
}
```

## 2.3 挂载probe操作
OP-TEE驱动在optee_driver_init函数中完成probe操作。该函数首先会通过设备树找到OP-TEE驱动的设备信息，然后将获取到的信息传递给optee_probe函数执行probe操作。probe操作主要完成版本的校验、获取OP-TEE驱动与TEE侧共享内存的配置、建立共享内存的地址映射、添加安全监控模式调用（smc）接口、分配/dev/tee0和/dev/teepriv0设备、建立RPC请求队列等操作。optee_probe函数内容如下：
```C
static struct optee *optee_probe(struct device_node *np){
	optee_invoke_fn *invoke_fn;
	struct tee_shm_pool* pool;
	struct optee* optee=NULL;
	void *memremaped_shm=NULL;
	struct tee_device* teedev;
	u32 sec_caps;
	int rc;
	
	//获取OP-TEE驱动在device tree中及节点描述内容中的定义的执行切换到monitor模式的接口
	invoke_fn = get_invoke_func(np);
	if(IS_ERR(invoke_fn))
		return (void* )invoke_fn;
	//调用到secure world中获取API版本信息是否匹配
	if(!optee_msg_api_uid_is_optee_api(invoke_fn)){
		pr_warn("api uid mismatch \n");
		return ERR_PTR(-EINVAL);
	}
	//调用到secure world中获取版本信息是否匹配
	if(!optee_msg_api_revision_is_compatible(invoke_fn)){
		pr_warn("api revision mismatch \n");
		return ERR_PTR(-EINVAL);
	}
	//调用到secure world 中获取secure world是否reserved shared memory
	if(!optee_msg_exchange_capbilities(invoke_fn,&sec_caps)){
		pr_warn("capabilities mismatch\n");
		return ERR_PTR(-EINVAL);
	}
	//we have no other option for shared memory,if secure world 
	//doesn't have any reserved memory we can use we can't continue
	//判断secure world中是否reserve了share memory，如果没有则报错
	if(!(sec_caps & OPTEE_SMC_CAP_HAVE_RESERVED_SHM)){
		return ERR_PTR(-EINVAL);
	}
//配置secure world 与驱动之间的shared memory,并进行地质映射到建立共享内存池
	pool= optee_config_shm_memremap(invoke_fn,&memremaped_shm);
	if(IS_ERR(pool)){
		return (void* )pool;
	}
//在kernel space内存空间中分配一块内存用于存放OP-TEE驱动的结构体变量
	optee = kzalloc(sizeof(*optee),GFP_KERNEL);
	if(!optee){
		rc=-ENOMEM;
		goto err;
	}
//将驱动用于实现进入monitor模式的接口赋值到optee结构体中的invoke_fn成员中
	optee->invoke_fn= invoke_fn;
//分配设备信息，填充libteec使用的驱动文件信息和operation结构体
//并创建/dev/tee0文件，libteec将会使用该文件来实现op-tee驱动
	teedev=tee_device_alloc(&optee_desc,NULL,pool,optee);
	if(IS_ERR(teedev)){
		rc=PTR_ERR(teedev);
		goto err;
	}
	
	//libteec使用的驱动文件信息填充到optee中的teedev的成员中
	//分配设备信息，填充被tee_supplicant使用的驱动文件信息和operation结构体并创建
	//到dev/teepri0文件，tee_supplicant将会使用该文件使用op-tee驱动
	teedev=tee_device_alloc(&optee_supp_desc,NULL,pool,optee);
	if(IS_ERR(teedev)){
		rc=PTR_ERR(teedev);
		goto err;
	}
//将tee_supplicant使用的驱动文件信息填充到optee中的supp_teedev成员中
	optee->supp_teedev=teedev;
//将被libteec使用的设备信息注册到系统设备中
	rc=tee_device_register(optee->teedev);
	if(rc)
		goto err;
//将被tee_supplicant 使用的设备信息注册到系统设备中
	tc=tee_device_register(optee->supp_teedev);
	if(rc)
		goto err;
	mutex_init(&optee->call_queue.mutex);
	INIT_LIST_HEAD(&optee->call_queue.waiters);
//初始化RPC操作队列
	optee_wait_queue_init(&optee->wait_queue);
//初始化被tee_supplicant用到的用于存放来自TA的请求队列
	optee_supp_init(&optee->supp);
//填充optee中的共享内存信息和共享内存池信息成员
	optee->memremaped_shm=memremaped_shm;
	optee->pool=pool;
//使能共享内存的cache
	optee_enable_shm_cache(optee);
	ptr_info("initialized driver\n");
	return optee;
err:
	if(optee){
		//tee_device_unregister() is safe to call even if the
		//devices hasn't been registered with tee_device_register() yet
		tee_device_unregister(optee->supp_teedev);
		tee_device_unregister(optee->teedev);
		kfree(optee);
	}
	if(pool)
		tee_shm_pool_free(pool);
	if(memremaped_shm)
		memunmap(memremaped_shm);
	return ERR_PTR(rc);
}
```

## 2.4 获取切换到Monitor模式或EL3的接口

正常世界状态与安全世界状态之间的切换是通过在Monitor模式或EL3下设定SCR寄存器中的安全状态位（NS bit）来实现的，OP-TEE驱动被上层调用时，最终会通过触发安全监控模式调用（smc）切换到Monitor模式或EL3，并通过共享内存的方式将数据发送给安全世界状态来进行处理。而用户触发安全监控模式调用的接口函数将在OP-TEE驱动初始化时被填充到OP-TEE驱动的device info中，在OP-TEE驱动中通过调用get_invoke_func函数来获取该接口的指针。该函数的内容如下：

```C
static optee_invoke_fn *get_invoke_func(struct device_node*np){
	const char*method;
	pr_info("probing for conduit method from DT.\n");
//获取op-tee驱动在device tree中的节中method属性的值
	if(of_property_read_string(np,"method",&methodd)){
		pr_warn("missing \"method\" property\n");
		return ERR_PTR("-ENXIO");
	}
//判定op-tee驱动是使用smc的方式还是使用hvc的方式来实现进入monitor模式的操作，
//根据method的值与hvc还是smc匹配来决定那种切换方法，并将用于切换到monitor的接口
	if(!strcmp("hvc",method))
		return optee_smccc_hvc;
	else if(!strcmp("smc",method))
		return optee_smccc_smc;
	
	pr_warn("invalid \"method\" property:%s\n",method);
	return ERR_PTR(-EINVAL);
}
```

执行安全监控模式调用指令会使ARM核进入EL3或Monitor模式。如果使用hvc，会是ARM核进入到EL2或者hyp模式，该模式主要用在使能虚拟机的系统上。这里以安全监控模式调用为例，实现系统状态切换的函数就是optee_smccc_smc，该函数内容如下：

```C
static void optee_smcc_smc(unsigned long a0,unsigned long a1,
	unsigned long a2,unsinged long a3,
	unsinged long a4,unsinged long a4,
	unsinged long a4,unsinged long a5,
	unsinged long a6,unsinged long a7,
	struct arm_smccc_res* res){
	arm_smccc_smc(a0,a1,a2,a3,a4,a5,a6,a7,res);
}
```

即是函数get_invoke_func执行完成之后会返回arm_smccc_smc函数的地址。arm_smccc_smc函数就是驱动用来将cortex切换到monitor模式的函数，该函数是以汇编的方式编写，定义在`linux/arch/arm/kernel/smccc-call.S`文件中。如果是64位系统，则该函数定义在`linux/arch/arm64/kernel/smccc-call.S`目录中，本文以32位系统为例，该函数内容如下：

```assembly
//wrap cmacros in asm macros to delay expansion until after the 
//SMCCC asm macro is expanded
/*SMC_SMC宏，触发smc*/
	.macro SMCCC_SMC
	__SMC(0)
	.endm

/*SMCCC_HVC宏，触发hvc*/
	.macro SMCCC_HVC
	__HVC(0)
	.endm

/*定义SMCCC宏，其参数为instr*/
	.mac SMCCC instr
UNWIND( .fnstart) //将normal world中的寄存器入栈，保存现场
	mov r12,sp
	push {r4-r7}
UNWIND( .save {r4-r7})
	ldm r12,{r4-r7}
	\instr   //执行instr参数的内容，即执行smc切换
	pop {r4-r7}   //出栈操作，恢复现场
	ldr r12,[sp,#(4 * 4)]
	stm r12,{r0-r3}
	bx lr
UNWIND( .fnend)
	.endm

ENTRY(arm_smccc_smc)
	SMCCC SMCCC_SMC
ENDPROC(arm_smccc_smc)
```

## 2.5 驱动版本和API版本校验
OP-TEE驱动挂载过程中会校验驱动的版本以及提供的API版本是否一致，该检查是通过触发快速安全监控模式调用（fast smc）从OP-TEE中获取到版本信息来实现的。快速安全监控模式调用与标准安全监控模式调用（std smc）的不同之处就在于第一个参数的BIT31的值不一样。

驱动加载过程中获取到REE侧与TEE侧之间进行交互的接口函数（调用`get_invoke_func`函数返回的函数地址）之后，OP-TEE驱动会对API的UID和版本信息进行校验。上述操作是通过调用`optee_msg_api_uid_is_optee_api`函数和`optee_msg_api_revision_is_compatible`函数来实现的。这两个函数的内容如下：
```C
static bool optee_msg_api_uid_is_optee_api(optee_invoke_fn *invoke_fn) {
	struct arm_smccc_res res; 
	/* 调用执行smc操作的接口函数,带入的command ID为OPTEE_SMC_CALLS_UID */
	invoke_fn(OPTEE_SMC_CALLS_UID, 0, 0, 0, 0, 0, 0, 0, &res); 
	/* 比较返回的UID的值与在驱动中定义的UID的值是否匹配 */ 
	if (res.a0 == OPTEE_MSG_UID_0 && res.a1 == OPTEE_MSG_UID_1 && 			res.a2 == OPTEE_MSG_UID_2 && res.a3 == OPTEE_MSG_UID_3) 
		return true; 
	return false; 
}
```

```C
static bool optee_msg_api_revision_is_compatible(optee_invoke_fn *invoke) { 
	union { 
            struct arm_smccc_res smccc; 
            struct optee_smc_calls_revision_result result; 
        } res;
/* 调用执行smc操作的接口函数,带入的command ID为OPTEE_SMC_CALLS_REVISION*/ 
	invoke_fn(OPTEE_SMC_CALLS_REVISION, 0, 0, 0, 0, 0, 0, 0, &res.smccc); 
/* 比较返回的版本信息的值与驱动中定义的版本值是否匹配 */ 
        if (res.result.major == OPTEE_MSG_REVISION_MAJOR && 
            (int)res.result.minor >= OPTEE_MSG_REVISION_MINOR) 
	    return true; 
	return false; 
}
```

>   ### FAST SMC和STD SMC[^3]
>
>   在Two types of calls are defined: 
>
>   *   Fast Calls used to execute atomic Secure operations.  (原子的安全操作)
>   *    Standard Calls used to start pre-emptible Secure operations.  （可抢占的安全操作）
>
>   在OP-TEE驱动的挂载过程中会使用fast smc的方式从OP-TEE中获取到相关数据，例如从secure world中获取reserve的共享内存信息时就是通过调用如下函数来实现的：
>   ```C
>   invoke_fn(OPTEE_SMC_GET_SHM_CONFIG, 0, 0, 0, 0, 0, 0, 0, &res.smccc);
>   ```
>   在支持smc操作的32位系统中该函数等价于：
>   ```C
>   arm_smccc_smc(OPTEE_SMC_GET_SHM_CONFIG, 0, 0, 0, 0, 0, 0, 0, &res.smccc);
>   ```
>   而OPTEE_SMC_ENABLE_SHM_CACHE的定义如下：
>   ```C
>   #define OPTEE_SMC_FUNCID_GET_SHM_CONFIG 7
>   #define OPTEE_SMC_GET_SHM_CONFIG OPTEE_SMC_FAST_CALL_VAL(OPTEE_SMC_FUNCID_GET_SHM_CONFIG)
>   ```
>   完全展开之`OPTEE_SMC_GET_SHM_CONFIG`宏的值的各个bit中的数值如下如所示：
>   ![](https://raw.githubusercontent.com/carloscn/images/main/typoratypora20221005104922-20221005105257334.png)
>
>   在执行smc操作时，cortex会解读第一个参数中的相关位来决定进入到monitor模式后的相关操作，在ARM官方定义了第一个参数`(a0)`的定义如下：
>
>   <div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221005110114768.png" width="80%" /></div>
>
>   当bit31为1时，则表示进入monitor模式后会执行`fast smc`的中断处理函数，而在不带ATF的ARCH32中，monitor的中断向量表如下：
>   ```
>   FUNC thread_vector_table , :
>   UNWIND( .fnstart)
>   UNWIND( .cantunwind)
>   	b vector_std_smc_entry
>   	b vector_fast_smc_entry
>   	b vector_cpu_on_entry
>   	b vector_cpu_off_entry
>   	b vector_cpu_resume_entry
>   	b vector_cpu_suspend_entry
>   	b vector_fiq_entry
>   	b vector_system_off_entry
>   	b vector_system_reset_entry
>   UNWIND( .fnend)
>   END_FUNC thread_vector_table
>   ```
>   由此可以见:
>   -   驱动**挂载的时候**请求共享内存配置数据的请求将会被`vector_fast_smc_entry`处理。
>   -   当请求来自于CA的请求时，驱动会将第一个参数的`bi31`设置成`0`，也就是CA的请求会被`vector_std_smc_entry`进行处理

## 2.6 判断OP-TEE是否预留了共享内存
OP-TEE驱动与TEE之间需要进行数据的交互，而进行数据交互则需要一定的共享内存来保存OP-TEE和驱动之间共有的数据。所以在驱动初始化时需要检查该共享内存空间是否被预留出来。**通过获取安全世界状态（SWS）中的相关变量的值并判定该相关标识变量是否相等来判定安全世界状态是否预留有共享内存空间**。在**OP-TEE OS启动过程中，执行MMU初始化时会初始化该变量**。在OPTEE驱动端通过调用`optee_msg_exchange_capabilities`函数来获取该变量的值。
```C
static pool optee_msg_exchange_capabilities(optee_invoke_fn *invoke_fn,u32* sec_caps){
	union{
		struct arm_smccc_res smccc;
		struct optee_smc_exchange_capabilities_result result;
	} res;
	u32 a1=0;
//TODO this isn't enough to tell if it's UP system
//(from kernel point of view) or not,is_smp() returns the information needed,
//but can't be called directly from here
	if(!IS_ENABLED(CONFIG_SMP) || nr_cpu_ids ==1)
		a1 |=OPTEE_SMC_NSEC_CAP_UNIPROCESSOR;
//调用smc操作接口，获取secure world中的变量
	invoke_fn(OPTEE_SMC_EXCHANGE_CAPABILITES,a1,0,0,0,0,0,0,&res.smccc);
	if(res.result.status != OPTEE_SMC_RETURN_OK){
		return false;
	}
	//将返回值中的变量赋值为sec_caps
	*sec_caps = res.result.capabilities;
	return true;
}
```

当驱动获取到sec_caps的值之后会查看该值是否为宏`OPTEE_SMC_SEC_CAP_HAVE_RESERVED_SHM`定义的值`BIT(0)`，如果该值不为BIT(0)，则会报错，因为在secure world端都没有预留share memory空间，那驱动与secure world之间也就没法传输数据，所以有没有驱动也就没有必要了。

## 2.7 配置驱动与OP-TEE之间的共享内存
驱动与安全世界状态之间的数据交互是通过共享内存来完成的，**在OP-TEE启动过程中会将作为共享内存的物理内存块预留出来**，具体可查看OPTEE启动代码中的`core_init_mmu_map`函数。O**PTEE驱动初始化阶段会将预留出来作为共享内存的物理内存配置成驱动的内存池，并通知OP-TEE OS执行相同的操作**。配置完成后，安全世界状态就能从共享内存中获取到来自REE侧的数据。

OP-TEE驱动进行probe操作时，会调用到`optee_config_shm_memremap`函数来完成OP-TEE驱动和OP-TEE之间共享内存的配置。该函数定义在`linux/drivers/tee/optee/core.c`文件中。其内容如下：

```c
static struct tee_shm_pool * optee_config_shm_memremap(optee_invoke_fn* invoke_fn,void **memreamped_shm){
	union{
		struct arm_smccc_res smccc;
		struct optee_smc_get_shm_config_result result;
	} res;
	struct tee_shm_pool* pool;
	unsinged long vaddr;
	phys_addr_t paddr;
	size_t size;
	phys_addr_t begin;
	phys_addr_t end;
	void *va;
	struct tee_shm_pool_mem_info priv_info;
	struct tee_shm_pool_mem_info dmabuf_info;

//调用smc类操作，通知OP-TEE OS返回被reserve出来的共享内存的物理地址和大小
	invoke_fn(OPTEE_SMC_GET_SHM_CONFIG,0,0,0,0,0,,&res.smccc);
	if(res.result.status != OPTEE_SMC_RETURN_OK){
		pr_info("shm service not avaliable \n");
		return ERR_PTR(-ENOENT);
	}
//判断是否提供secure world中的cache
	if(res.result.settings != OPTEE_SMC_SHM_CACHED){
		pr_err("only normal cached shared memory supported\n");
		return ERR_PTR(-EINVAL);
	}
//将对齐操作之后的物理内存块的起始地址赋值给paddr,该块内存的大小赋值给size
	begin =roundup(res.result.start,PAGE_SIZE);
	end=rounddown(res.result.start+res.result.size,PAGE_SIZE);
	paddr=begin;
	size=end - begin;
//判断作为共享内存的物理地址的大小是否大于两个page大小
//如果小于则报错，因为驱动配置用于dma操作和普通共享内训的大小分别为一个page大小
	if(size<2*OPTION_SHM_NUM_PRIV_PAGES*PAGE_SIZE){
		pr_err("too small shared memory area \n");
		return ERR_PTR(-EINVAL);
	}
//配置驱动私有内存空间的虚拟地址的启动地址，物理地址的起始地址以及大小，配置
//dma缓存的虚拟起始地址和物理地址以及大小。
//dmabuf与pribuf两个相邻，分贝为一个page的大小
	priv_info.vaddr=vaddr;
	priv_info.paddr=paddr;
	priv_info.size = OPTEE_SHM_NUM_PRIV_PAGES * PAGE_SIZE;
	dmabuf_info.vaddr=vaddr+OPTEE_SHM_NUM_PRIV_PAGES *PAGE_SIZE;
	dmabuf_info.paddr=paddr+OPTEE_SHM_NUM_PRIV_PAGES*PAGE_SIZE;
	dmabuf_info.size=size-OPTEE_SHM_NUM_PRIV_PAGES*PAGE_SIZE;
//将驱动的私有buffer和dma buffer天剑到内存池中，以便驱动在使用本身的alloc
//函数的时候能够从私有共享和dma buffer中分配内存来使用
	pool= tee_shm_pool_alloc_res_mem(&priv_info,&dmabuf_info);
	if(IS_ERR(pool)){
		memunmap(va);
		goto out;
	}
//将驱动与OP-TEE的共享内存赋值给memremaped_shm变量执行的地址
	*memremaped_shm=va;
out:
	return pool;//返回共享内存池的结构体
}
```

在secure world中预留出来的内存块作为驱动与sercure world之间的共享内存使用。在**驱动端将会建立一个内存池**，**以便驱动在需要的使用通过调用驱动的alloc函数来完成共享内存的分配**。而共享内存池的建立是通过调用`tee_shm_pool_alloc_res_mem`来实现的。其函数内容如下：

```c
struct tee_shm_pool* tee_shm_pool_alloc_res_mem(struct tee_shm_pool_mem_info *priv_info,
	struct tee_shm_pool_mem_info *dmabuf_info){
	struct tee_shm_pool *pool=NULL;
	int ret;
//从内核空间的memory中分配一块用于存放驱动内存池结构体变量的内存
	pool = kzalloc(sizeof(*pool),GFP_KERNEL);
	if(!pool){
		ret = -ENOMEM;
		goto err;
	}
	//TODO create the pool for driver private shared memory
//调用pool相关函数完成内存池的创建，设定alloc的分配算法，并将私有化共享内存的
//其实虚拟地址，其实物理地址以及大小信息保存到私有共享内存池中
	ret = pool_res_mem_mgr_init(&pool->private_mgr,pri_info,3/*8 byte aligned*/);
	if(ret)
		goto err;
	
	//TODO create the pool for dma_buf shared memory
//调用pool相关函数完成内存池的创建，设定alloc时分配算法，并将dma共享内存的起始的
//虚拟地址，起始物理地址以及大小信息保存到dma的共享内存池中
	ret = pool_res_mem_mgr_init(&pool->dma_buf_mgr,dmabuf_info,PAGE_SHIFT);
	if(ret)
		goto err;
//设定销毁共享内存池的接口函数
	pool->destory = pool_res_mem_destory;
	return pool;//返回内存池结构体
err:
	if(ret == -ENOMEM)
		pr_err("%s:can't allocate memory for res_mem shared memory pool\n",__func__);
	if(pool && pool->private_mgr.private_data)
		gen_pool_destory(pool->private_mgr.private_data);
	kfree(pool);
	return ERR_PTR(ret);
}
```

## 2.8 分配tee0和teepriv0变量
在OP-TEE驱动进行probe操作时会分配和设置两个tee_device结构体变量，分别用来表示被`libteec`库和`tee_supplicant`使用的设备。分别通过执行`tee_device_alloc(&optee_desc，NULL，pool，optee)`和`tee_device_alloc(&optee_supp_desc，NULL，pool，optee)`来实现，主要是设置驱动被libteec库和tee_supplicant使用时的设备具体操作和设备对应的名称等信息。
* 当libteec库调用文件操作函数执行打开、关闭等操作`/dev/tee0`设备文件时，系统最终将调用到`optee_desc`中具体的函数来实现对应操作。
* 当tee_supplicant调用文件操作函数执行打开、关闭等操作`/dev/teepriv0`设备文件时，系统最终将调用到`optee_supp_desc`中具体的函数来实现对应操作。

上述配置操作都是通过调用`tee_device_all`函数来实现的。
```C
static tee_device* tee_device_alloc(const struct tee_desc *teedesc,struct device*dev,struct tee_shm_pool* pool,void *driver_data){
	struct tee_device* teedev;
	void *ret;
	int rc;
	int offs=0;

//参数检查
	if(!teedesc || !teedesc->name ||!teedesc->ops||
		!teedesc->ops->get_version || !teedesc->ops->open ||
		|| !teedesc->ops->release ||!pool){
		return ERR_PTR(-EINVAL);
	}
//从kernel space中分配用于存放tee_device变量的内存
	teedev=kzalloc(sizeof(*teedev),GFP_KERNEL);
	if(!teedev){
		ret=ERR_PTR(-ENOMEM);
		goto err;
	}
//判定当前分配的设备结构体是提供给libteec还是tee_supplicant,如果该设备时分配
//给tee_supplicant，则将offs设置成16，offs将会在用于设置设备的id时被使用 
	if(teedesc->flags & TEE_DESC_PRIVILEGED)
		offs = TEE_NUM_DEVICES /2;
//查找dev_mask中的从Offs开始的第一个为0的bit位，然后将该值作为设备的id值
	spin_lock(&driver_lock);
	teedev->id=find_next_zero_bit(dev_mask,TEE_NUM_DEVICES,offs);
	if(teedev->id<TEE_NUM_DEVICES)
		set_bit(teedev_id,dev_mask);
	spin_unlock();
//判断设定的设备id是否超出最大数
	if(teedev->id >= TEE_NUM_DEVICES){
		ret =ERR_PTR(-ENOMEM);
		goto err;
	}
// 组合出设备名，对于libteec来说，设备名为tee0。对于tee_supplicant来说，设备名为teepriv0
	snprintf(teedev->name, sizeof(teedev->name), "tee%s%d",
		 teedesc->flags & TEE_DESC_PRIVILEGED ? "priv" : "",
		 teedev->id - offs);
//设定设备的class,tee_class在tee_init函数中被分配。设定执行设备release的操作函数
   和dev.parent
	teedev->dev.class = tee_class;	
	teedev->dev.release = tee_release_device;
	teedev->dev.parent=dev;
//将设备的主设备号和设备ID组合后转化成dev_t类型 
	teedev->dev.devt=MKDEV(MAJOR(tee_devt),teedev->id);
//设置设备名，驱动被libteec使用时设备名为tee0,驱动被tee_supplicant使用时设备名为teepriv0
	rc = dev_set_name(&teedev->dev, "%s", teedev->name);
	if (rc) {
		ret = ERR_PTR(rc);
		goto err_devt;
	}
//设置驱动作为字符设备的操作函数接口，即指定该驱动在执行open,close, ioctl的函数接口
	cdev_init(&teedev->cdev, &tee_fops);
	teedev->cdev.owner = teedesc->owner;	//初始化字符设备的owner
	teedev->cdev.kobj.parent = &teedev->dev.kobj;	//初始化kobj.parent成员
//设置设备的私有数据
	dev_set_drvdata(&teedev->dev, driver_data);
//初始化设备
	device_initialize(&teedev->dev);
//1 as tee_device_unregister() does one final tee_device_put()
	teedev->num_users =1;//标记改设备可以被使用
	init_completion(&teedev->c_no_users);
	mutex_init(&teedev->mutex);
	idr_init(&teedev->idr);

//设定设备的desc成员，该成员包含设备最终执行具体操作的函数接口
	teedev->desc=teedesc;
//设置设备的内存池，主要是驱动与secure world之间共享内存的私有共享内存和dma操作共享内存
	teedev->pool = pool;
	return teedev;
err_devt:
	unregister_chrdev_region(teedev->dev.devt,1);
err:
	pr_err("could not register %s driver\n",teedesc->flags & TEE_DESC_PRIVILEGED?"privlieged":"client");
	if(teedev && teedev->id< TEE_NUM_DEVICES){
		spin_lock(&driver_lock);
		clear_bit(teedev->id,dev_mask);
		spin_unlock(&driver_lock);
	}
	kfree(teedev);
	return ret;
}
```

完成版本检查，共享内存池配置，不同设备的配置之后就需要将这些配置好的设备注册到系统中去。对于被liteec和tee_supplicant使用的设备分别通过调用`tee_device_register(optee->teedev)`和`tee_device_register(optee->supp_teedev)`来实现。其中`optee->teedev`和`optee->supp_teedev`就是在上一章中被配置好的分别被libteec和tee_supplicant使用的设备结构体。调用`tee_device_register`函数来实现将设备注册到系统的目的，该函数内容如下：

```C
int tee_device_register(struct tee_device*teedev){
	int rc;
//判定设备是否已经被注册过
	if(teedev->flags & TEE_DEVICE_FLAG_REGISERED){
		dev_err(&teedev->dev,"attempt to register twice\n");
		return -EINVAL;
	}
//注册字符设备
	rc=cdev_add(&teedev->cdev,teedev->dev.devt,1);
	if(rc){
		dev_err(&teedev->dev,
			"unable to cdev_add() %s, major %d, minor %d, err=%d\n",
			teedev->name, MAJOR(teedev->dev.devt),
			MINOR(teedev->dev.devt), rc
		return rc;
	}
//将设备添加到linux的设备模块中,在该步中将会在/dev目录下创设备驱动文件节点，即对于
//被libteec使用的设备，在该步将创建/dev/tee0设备驱动文件。对于被tee_supplicant使用的设备，在该步将创建/dev/teepriv0设备文件
	rc=device_add(&teedev->dev);
	if(rc){
		dev_err(&teedev->dev,
			"unable to device_add() %s, major %d, minor %d, err=%d\n",
			teedev->name, MAJOR(teedev->dev.devt),
			MINOR(teedev->dev.devt), rc
			goto err_device_add;
	}
//在/sys目录下创建设备的属性文件
	rc=sysfs_create_group(&teedev->dev.kobj,&tee_dev_group);
	if(rc){
		dev_err(&teedev->dev,
			"failed to create sysfs attributes, err=%d\n", rc);
		goto err_sysfs_create_group;
	}
// 设定该设备以及被注册过
	teedev->flags |=TEE_DEVICE_FLAG_REGISTERED;
	return 0;
err_sysfs_create_group:
	device_del(&teedev->dev);
err_device_add:
	cdev_del(&teedev->cdev);
	return rc;
}
```
## 2.9 两个队列初始化
OP-TEE驱动提供两个设备，分别是被libteec库使用的`/dev/tee0`和被tee_supplicant使用的`/dev/teepriv0`。为确保正常世界状态与安全世界状态之间数据交互便利且能在正常世界状态进行异步处理，OP-TEE驱动在挂载时会建立两个类似于消息队列的队列，用于保存正常世界状态的请求数据和安全世界状态的请求。`optee_wait_queue_init`用于初始化`/dev/tee0`设备使用的队列，`optee_supp_init`用于初始化`/dev/teepriv0`设备使用的队列。其代码分别如下：
```C
void optee_wait_queue_init(struct optee_wait_queue* priv){
	mutex_init(&priv->mu);
	INIT_LIST_HEAD(&priv-db);
}

void optee_supp_init(struct optee_supp* supp){
	memset(supp,0,sizeof(*supp));
	mutex_init(&supp->mutex);
	init_comletion(&supp->reqs_c);
	idr_init(&supp->idr);
	INIT_LIST_HEAD(&supp-reqs);
	supp->req_id=-1;
}
```

## 2.10 TEE中的共享cache
当一切执行完之后，最后就剩下通知OP-TEE使能共享内存的缓存了，在OP-TEE驱动的挂载过程中通过调用`optee_enable_shm_cache`函数来实现使能共享内存Cache的操作。该函数内容如下：

```C
void optee_enable_shm_cache(struct optee*optee){
	struct optee_call_waiter w;
	//we need to retry until secure world isn't busy
//确定secure world是否ready
	optee_cp_wait_init(&optee->call_queue,&w);
//进入loop循环，通知secure world执行相应操作，知道返回OK后跳出
	while(true){
		struct arm_smccc_res res;
//通知smc操作，通知secure world执行使能共享内存cache的操作
		optee->invoke_fn(OPTEE_SMC_ENABLE_SHM_CACHE,0,0,0,0,0,0,&res);
		if(res.a0 == OPTEE_SMC_RETURN_OK)
			break;
		optee_cq_wait_for_completion(&optee->call_queue,&w);
	}
	optee_cq_wait_final(&optee->call_queue,&w);
}
```

## 2.11 总结
从OP-TEE驱动的挂载过程来看，OP-TEE驱动会分别针对libteec库和tee_supplicant建立不同的设备/dev/tee0和/dev/teepriv0。同时为两个设备中的des配置各自独有的operation结构体变量，并建立类似消息队列来存放正常世界状态与安全世界状态之间的请求，这样libteec库和tee_supplicant使用OP-TEE驱动时就能做到相对的独立。安全世界状态与OPTEE驱动之间使用共享内存进行数据交互。**用于作为共享内存的物理内存块在OP-TEE启动过程中进行MMU初始化时需要被预留出来**，在OP-TEE驱动的挂载过程中需要将该内存块映射到系统内存中。

# 3. REE侧用户空间对驱动的调用
在Linux用户空间对文件系统中的文件执行打开、关闭、读写以及ioctl操作时，最终都会穿透到Linux内核空间执行具体的操作。而从用户空间陷入到内核空间是通过系统调用（systemcall）来实现的（关于syscall的实现可自行查阅资料了解），进入Linux内核空间后，系统会调用相应的驱动来获取设备对应的file_operations变量，该结构体变量中存放了对文件进行各种操作的具体函数指针。所以从用户空间对文件进行操作时，其整个过程大致如图所示。

调用libteec库中按照GP标准定义的API或tee_supplicant执行具体操作时都会经历图所示的流程：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221005114651.png" width="100%" alt="REE侧用户空间调用OP-TEE驱动的大致流程"/></div>


# 4. OP-TEE驱动中的重要结构体变量
要了解OP-TEE驱动中具体做了哪些操作，首先需要了解在OP-TEE驱动中存在的四个重要的结构体，`libteec`和`tee_supplicant`以及`dma操作使用驱动时`会使用到这四个结构体，这四个结构体变量会在驱动挂载的时候被注册到系统设备模块或者是该设备的自由结构体中以便被userspace使用，而dma操作的时候会对共享内存进行注册。

* **tee_fops** : OP-TEE驱动的file_operations
* **tee_driver_ops** : OP-TEE驱动中`/dev/tee0`设备的tee_driver_ops结构体
* **optee_supp_ops**: OP-TEE驱动中`/dev/teepriv0`设备的tee_driver_ops结构体
* **tee_shm_dma_buf_ops** : OP-TEE驱动中共享驱动缓存操作的dma_buf_ops结构体


## 4.1 OP-TEE驱动的file_operation结构体变量tee_fops

OP-TEE驱动的file_operation结构体变量定义在`linux/drivers/tee/tee_core.c`文件中，该变量中包含了OP-TEE驱动文件操作函数指针，其内容如下：

```c
struct const struct file_operations tee_fops={
	.owner=THIS_MODULE,
	.open=tee_open,
	.release=tee_release,
	.unlocked_ioctl=tee_ioctl,//驱动文件的ioctl操作具体实现的函数指针
	.compat_ioctl=tee_ioctl,//驱动文件ioctl操作具体实现指针
};
```

当在userspace层面调用open, release, ioctl函数操作驱动文件时就会调用到该结构体中的对应函数去执行具体操作。

## 4.2 OP-TEE驱动中`/dev/tee0`设备的tee_driver_ops结构体变量optee_ops

当用户调用libteec中的接口的时，操作的就是OP-TEE驱动的`/dev/tee0`设备，而optee_ops变量中存放的就是针对`/dev/tee0`设备的具体操作函数的指针，用户调用libteec接口时，首先会调用到tee_fops中的成员函数，tee_fops中的成员函数再会去调用到optee_ops中对应的成员函数来完成对`/dev/tee0`设备的实际操作。optee_ops变量定义在`linux/drivers/tee/optee/core.c`文件中。其内容如下：

```c
static struct tee_driver_ops optee_ops = {
	.get_version = optee_get_version,	//获取OP-TEE版本信息的接口函数
	.open = optee_open,  //打开/dev/tee0设备的具体实现，初始化列表和互斥体，返context
	.release = optee_release, //释放掉打开的/dev/tee0设备资源，并通知secure world关闭session
	.open_session = optee_open_session, //打开session，以便CA于TA进行交互
	.close_session = optee_close_session, //关闭已经打开的session，断开CA与TA之间的交互
	.invoke_func = optee_invoke_func, 	//通过smc操作发送CA请求到对应TA
	.cancel_req = optee_cancel_req, //取消CA端已经发送的smc请求
};
```

## 4.3 OP-TEE驱动中`/dev/teepriv0`设备的tee_driver_ops结构体变量optee_supp_ops

当tee_supplicant需要执行相关操作时，操作的就是OP-TEE驱动的`/dev/teepriv0`设备，而optee_supp_ops变量中存放的就是针对`/dev/teepriv0`设备的具体操作函数的指针，tee_supplicant执行相关操作时，首先会调用到tee_fops中的成员函数，tee_fops中的成员函数再会去调用到optee_supp_ops中对应的成员函数来完成对`/dev/teepriv0`设备的实际操作。optee_supp_ops变量定义在`linux/drivers/tee/optee/core.c`文件中。其内容如下：

```c
static struct tee_driver_ops optee_supp_ops = {
	.get_version = optee_get_version,	//获取OP-TEE的版本信息
	.open = optee_open,	//打开/dev/teepriv0设备的具体实现
	.release = optee_release, //释放掉打开的/dev/teepriv0设备，并通知secure world关闭session
	.supp_recv = optee_supp_recv, //接收从OP-TEE发送给tee_supplicant的请求
	.supp_send = optee_supp_send,  //执行完OP-TEE请求的操作后将结果和数据发送给OP-TEE
};
```

## 4.4 OP-TEE驱动中共享驱动缓存操作的dma_buf_ops结构体变量tee_shm_dma_buf_ops

OP-TEE驱动也支持其他设备访问OP-TEE驱动的共享缓存，该部分的功能当前并不算太完全，有一些功能尚未实现。该变量定义在`linux/drivers/tee/tee_shm.c`文件中，当需要被分配dma缓存时就会调用到该变量中对应的函数。其内容如下：

```c
static struct dma_buf_ops tee_shm_dma_buf_ops = {
	.map_dma_buf = tee_shm_op_map_dma_buf,	//暂未实现
	.unmap_dma_buf = tee_shm_op_unmap_dma_buf,	//暂未实现
	.release = tee_shm_op_release,	//释放掉指定的共享内存
	.kmap_atomic = tee_shm_op_kmap_atomic, //暂未实现
	.kmap = tee_shm_op_kmap,	//暂未实现
	.mmap = tee_shm_op_mmap, 	//dma共享内存进行地址映射
};
```

**NOTE**: libteec中接口的执行和tee_supplicant功能的执行都会用上述四个结构体变量中的两个或者多个。

# 5. OP-TEE驱动与OP-TEE共享内存的注册和分配
当libteec和tee_supplicant需要分配和注册于secure world之间的共享内存时，可以通过调用驱动的ioctl方法来实现，然后驱动调用tee_ioctl_shm_alloc来实现具体的分配，注册共享内存的操作。该函数的内容如下：

```c
static int tee_ioctl_shm_alloc(struct tee_context *ctx,
			       struct tee_ioctl_shm_alloc_data __user *udata)
{
	long ret;
	struct tee_ioctl_shm_alloc_data data;
	struct tee_shm *shm;
 
/* 将userspace传递的参数数据拷贝到kernel的buffer中 */
	if (copy_from_user(&data, udata, sizeof(data)))
		return -EFAULT;
 
	/* Currently no input flags are supported */
	if (data.flags)
		return -EINVAL;
 
/* 将共享内存的ID值设置成-1,以便分配好共享内存之后重新赋值 */
	data.id = -1;
 
/* 调用tee_shm_all函数，从驱动与secure world之间的共享内存池中分配对应大小的内存，
并设定对应的ID值 */
	shm = tee_shm_alloc(ctx, data.size, TEE_SHM_MAPPED | TEE_SHM_DMA_BUF);
	if (IS_ERR(shm))
		return PTR_ERR(shm);
 
/* 设定需要返回给userspace的数据 */
	data.id = shm->id;
	data.flags = shm->flags;
	data.size = shm->size;
 
/* 将需要返回的数据从kernespace拷贝到userspace层面 */
	if (copy_to_user(udata, &data, sizeof(data)))
		ret = -EFAULT;
	else
		ret = tee_shm_get_fd(shm);
 
	/*
	 * When user space closes the file descriptor the shared memory
	 * should be freed or if tee_shm_get_fd() failed then it will
	 * be freed immediately.
	 */
	tee_shm_put(shm); //如果分配的是DMA的buffer则要减少count值
	return ret;
}
```

而在tee_shm_all中驱动做了什么操作呢？该函数的内容如下：

```c
struct tee_shm *tee_shm_alloc(struct tee_context *ctx, size_t size, u32 flags)
{
	struct tee_device *teedev = ctx->teedev;
	struct tee_shm_pool_mgr *poolm = NULL;
	struct tee_shm *shm;
	void *ret;
	int rc;
 
/* 判定参数是否合法 */
	if (!(flags & TEE_SHM_MAPPED)) {
		dev_err(teedev->dev.parent,
			"only mapped allocations supported\n");
		return ERR_PTR(-EINVAL);
	}
 
/* 判定flags，表明该操作只能从驱动的私有共享呢村或者DMA buffer中进行分配 */
	if ((flags & ~(TEE_SHM_MAPPED | TEE_SHM_DMA_BUF))) {
		dev_err(teedev->dev.parent, "invalid shm flags 0x%x", flags);
		return ERR_PTR(-EINVAL);
	}
 
/* 获取设备结构体 */
	if (!tee_device_get(teedev))
		return ERR_PTR(-EINVAL);
 
/* 判定设备的poll成员是否存在，该成员在驱动挂载的时候会被初始化成驱动与secure world
之间的共享内存池结构体 */
	if (!teedev->pool) {
		/* teedev has been detached from driver */
		ret = ERR_PTR(-EINVAL);
		goto err_dev_put;
	}
 
/* 分配存放shm的kernel空间的内存 */
	shm = kzalloc(sizeof(*shm), GFP_KERNEL);
	if (!shm) {
		ret = ERR_PTR(-ENOMEM);
		goto err_dev_put;
	}
 
/* 设定该块共享内存的flag以及对应的使用者 */
	shm->flags = flags;
	shm->teedev = teedev;
	shm->ctx = ctx;
 
/* 通过传入的flags来判定是从驱动的私有共享内存池还是工dma buffer中来进行分配 */
	if (flags & TEE_SHM_DMA_BUF)
		poolm = &teedev->pool->dma_buf_mgr;
	else
		poolm = &teedev->pool->private_mgr;
 
/* 调用pool中对应的alloc操作分配size大小的共享内存 */
	rc = poolm->ops->alloc(poolm, shm, size);
	if (rc) {
		ret = ERR_PTR(rc);
		goto err_kfree;
	}
 
/* 获取分配好的共享内存的id */
	mutex_lock(&teedev->mutex);
	shm->id = idr_alloc(&teedev->idr, shm, 1, 0, GFP_KERNEL);
	mutex_unlock(&teedev->mutex);
	if (shm->id < 0) {
		ret = ERR_PTR(shm->id);
		goto err_pool_free;
	}
 
/* 如果指定的是DMA buffer,则指定相关的operation */
	if (flags & TEE_SHM_DMA_BUF) {
		DEFINE_DMA_BUF_EXPORT_INFO(exp_info);
 
		exp_info.ops = &tee_shm_dma_buf_ops;
		exp_info.size = shm->size;
		exp_info.flags = O_RDWR;
		exp_info.priv = shm;
 
		shm->dmabuf = dma_buf_export(&exp_info);
		if (IS_ERR(shm->dmabuf)) {
			ret = ERR_CAST(shm->dmabuf);
			goto err_rem;
		}
	}
 
/* 将分配的内存链接到共享内存队列的末尾 */
	mutex_lock(&teedev->mutex);
	list_add_tail(&shm->link, &ctx->list_shm);
	mutex_unlock(&teedev->mutex);
 
	return shm;
err_rem:
	mutex_lock(&teedev->mutex);
	idr_remove(&teedev->idr, shm->id);
	mutex_unlock(&teedev->mutex);
err_pool_free:
	poolm->ops->free(poolm, shm);
err_kfree:
	kfree(shm);
err_dev_put:
	tee_device_put(teedev);
	return ret;
}
```

从整个过程来看
-   如果在libteec执行`共享内存`的分配或者是`注册操作`时，驱动都会从驱动与secure world的共享内存池中分配一块内存，然后将该分配好的内存的id值返回给libteec，然后在libteec中
-   如果是调用`TEEC_AllocateSharedMemory`函数，则会将该共享内存的id值进行mmap，然后将map后的值赋值给shm中的buffer成员。如果调用的是`TEEC_RegisterSharedMemory`，则会将共享内存id执行mmap之后的值赋值给shm中的shadow_buffer成员。

由此可见，当libteec中是执行注册共享内存操作时，并不是讲userspace的内存直接共享给secure world，而是将userspace的内存与驱动中分配的一块共享内存做shadow操作，是两者实现一个类似映射的关系。

# 6. libteec中的接口在驱动中的handle
libteec提供给上层使用的接口总共有十个，这十个接口的功能和定义介绍请参阅[07_OPTEE-OS_系统集成之（五）REE侧上层软件](https://github.com/carloscn/blog/issues/97)，这十个接口通过系统调用最终会调用到驱动中，在接口libteec中调用Open函数的时候，在驱动中就对调用到file_operations结构体变量tee_fops中的Open成员。同理在libteec接口中调用ioctl函数，则在驱动中最终会调用tee_fops中的ioctl函数。本文将介绍驱动中file_operation结构体变量`tee_fops`中各成员函数的具体实现.

驱动挂载完成后，CA程序通过调用libteec库中的接口调用OP-TEE驱动来穿透到OP-TEE中，然后调用对应的TA程序。OP-TEE驱动在挂载完成后会在/dev目录下分别创建两个设备节点，分别为`/dev/tee0`和`/dev/teepriv`，对`/dev/tee0`设备进行相关操作就能够穿透到OP-TEE中实现特定请求的发送。

## 6.1 tee_open
tee_core:
https://github.com/carloscn/raspi-linux/blob/master/drivers/tee/tee_core.c#L109

调用栈如下：
```bash
libteec(tee_supplicant)
    -> linux kernel_space
        -> find tee_open() (tee_core.c)
            -> call teedev_open() (tee_core.c)
	        -> call optee_open() (optee/core.c)
```

在libteec中调用open函数来打开`/dev/tee0`设备的时候，最终会调用到tee_fops中的open成员指定的函数指针tee_open，该函数的内容如下：

```c
static int tee_open(struct inode*inode,struct file*filp){
	struct tee_context *ctx;
//调用container_of函数，获取设备的tee_device变量的内容 。该变量对于/dev/tee0
//和/dev/teepriv0设备时不一样的，这点可以在驱动过载的过程中查阅
	ctx=teedev_open(container_of(inode->icdev,struct tee_device,cdev));
	if(IS_ERR(ctx)){
		return PTR_ERR(ctx);
	}
	filp->private_data=ctx;
	return 0;
}

static struct tee_context *teedev_open(struct tee_device* teedev){
	int rc;
	struct tee_context* ctx;
//标记该设备的使用者加一
	if(!tee_device_get(teedev))
		return ERR_PTR(-EINVAL);
//分配tee_context结构体空间
	ctx=kzalloc(sizeof(*ctx),GFP_KERNEL);
	if(!ctx){
		rc=-ENOMEM;
		goto err;
	}
//将tee_context结构体汇中teedev变量赋值
	ctx->teedev=teedev;
	INIT_LIST_HEAD(&ctx->list_shm);
//调用设备的des中的open执行设备几倍的open操作
	rc=teedev->desc->ops->open(ctx);
	if(rc)
		goto err;
	return ctx;
err:
	kfree(ctx);
	tee_device_put(teedev);
	return ERR_PTR(rc);
}
```

对于设备级别(`/dev/tee0`和`/dev/teepriv0`)，最终会调用到optee_open函数，该函数内容如下： https://github.com/carloscn/raspi-linux/blob/master/drivers/tee/optee/core.c#L220

```c
static int optee_open(struct tee_context*ctx){
	struct optee_context_data* ctxdata;
	struct tee_device*teedev=ctx->teedev;
	struct optee* optee=tee_get_drvdata(teedev);
//分配optee_context_data 结构体变量空间
	ctxdata=kzalloc(sizeof(*ctxdata),GFP_KERNEL);
	if(!ctxdata)
		return -ENOMEM;
//通过teedev的值是否为 optee->supp_teedev，以此来判定当前的Open操作是打
//开/dev/tee0设备还是/dev/teepriv0设备，如果相等，则表示当前是打开/dev/teepriv0设备
	if(teedev == optee->supp_teedev){
		bool busy=true;//标记/dev/teepriv0正在使用
		mutex_lock(&optee->supp.mutex);
		if(!optee->supp.ctx){
			busy=false;
			optee->supp.ctx=ctx;
		}
		mutex_unlock(&optee->supp.mutex);
		if(busy){
			kfree(ctxdata);
			return -EBUSY;
		}
	}
//初始化互斥提队列
	mutex_init(&ctxdata->mutex);
	INIT_LIST_HEAD(&ctxdata->sess_list);
//赋值
	ctx->data=ctxdata;
	return 0;
}
```

相应的 release的操作也是一样。

## 6.2 获取版本

在libteec中获取OP-TEE版本信息，创建和关闭session，调用TA，分配和注册共享内存和fd以及释放共享内存，**接收来自OP-TEE的请求以及回复数据给OP-TEE都是通过ioctl来完成的**。在libteec和tee_supplicant中通过带上对应的参数调用ioctl函数来实现对应的操作需求，最终会调用到OP-TEE驱动中file_operation结构体变量tee_fops变量中的tee_ioctl函数，该函数的内容如下。
```C
static long tee_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
	struct tee_context *ctx = filp->private_data;	//获取设备的私有数据
	void __user *uarg = (void __user *)arg; 	//定义指向userspace层面输入参数的指针变量
 
	switch (cmd) {
/* 和获取OP-TEE OS的版本 */
	case TEE_IOC_VERSION:
		return tee_ioctl_version(ctx, uarg);
 
/* 分配,注册，释放共享内存操作 */
	case TEE_IOC_SHM_ALLOC:
		return tee_ioctl_shm_alloc(ctx, uarg);
 
/* 注册共享文件句柄 */
	case TEE_IOC_SHM_REGISTER_FD:
		return tee_ioctl_shm_register_fd(ctx, uarg);
 
/* 打开CA与TA通信的session */
	case TEE_IOC_OPEN_SESSION:
		return tee_ioctl_open_session(ctx, uarg);
 
/* 调用TA,让根据参数中的command ID让TA执行特定的command */
	case TEE_IOC_INVOKE:
		return tee_ioctl_invoke(ctx, uarg);
 
/* 通知TA取消掉对某一个来自normal world请求的响应 */
	case TEE_IOC_CANCEL:
		return tee_ioctl_cancel(ctx, uarg);
 
/* 关闭CA与TA之间通信的session */
	case TEE_IOC_CLOSE_SESSION:
		return tee_ioctl_close_session(ctx, uarg);
 
/* 接收来自secure world的请求 */
	case TEE_IOC_SUPPL_RECV:
		return tee_ioctl_supp_recv(ctx, uarg);
 
/* 根据来自secure world的请求执行处理后回复数据给secure world */
	case TEE_IOC_SUPPL_SEND:
		return tee_ioctl_supp_send(ctx, uarg);
	default:
		return -EINVAL;
	}
}
```

当libteec和tee_supplicant调用ioctl来获取OP-TEE OS的版本信息时，驱动会执行tee_ioctl_version函数，该函数内容如下：

```C
static int tee_ioctl_version(struct tee_context *ctx,
			     struct tee_ioctl_version_data __user *uvers)
{
	struct tee_ioctl_version_data vers;
 
/* 调用设备的get_version操作 */
	ctx->teedev->desc->ops->get_version(ctx->teedev, &vers);
 
/* 判定该操作是来至于tee_supplicant还是libteec */
	if (ctx->teedev->desc->flags & TEE_DESC_PRIVILEGED)
		vers.gen_caps |= TEE_GEN_CAP_PRIVILEGED;
 
/* 将获取到的版本信息数据拷贝到userspace层面提供的buffer中 */
	if (copy_to_user(uvers, &vers, sizeof(vers)))
		return -EFAULT;
 
	return 0;
}
```

而设备的`get_version`的内容如下：

```C
static void optee_get_version(struct tee_device *teedev,
			      struct tee_ioctl_version_data *vers)
{
	struct tee_ioctl_version_data v = {
		.impl_id = TEE_IMPL_ID_OPTEE,
		.impl_caps = TEE_OPTEE_CAP_TZ,
		.gen_caps = TEE_GEN_CAP_GP,
	};
	*vers = v;
}
```

## 6.3 libteec中open session的操作
当用户调用libteec中的TEEC_OpenSession接口时会执行驱动中ioctl函数的TEE_IOC_OPEN_SESSION分支去执行tee_ioctl_open_session函数，该函数只会在打开了`/dev/tee0`设备之后才能被使用，其的内容如下：

```C
static int tee_ioctl_open_session(struct tee_context *ctx,
				  struct tee_ioctl_buf_data __user *ubuf)
{
	int rc;
	size_t n;
	struct tee_ioctl_buf_data buf;
	struct tee_ioctl_open_session_arg __user *uarg;
	struct tee_ioctl_open_session_arg arg;
	struct tee_ioctl_param __user *uparams = NULL;
	struct tee_param *params = NULL;
	bool have_session = false;
 
/* 判定设备是否存在open_session函数 */
	if (!ctx->teedev->desc->ops->open_session)
		return -EINVAL;
 
/* 将userspace传入的参数拷贝到kernelspace */
	if (copy_from_user(&buf, ubuf, sizeof(buf)))
		return -EFAULT;
 
/* 判定传入的参数的大小是否合法 */
	if (buf.buf_len > TEE_MAX_ARG_SIZE ||
	    buf.buf_len < sizeof(struct tee_ioctl_open_session_arg))
		return -EINVAL;
 
/* 为兼容性64位系统做出转换，并将数据拷贝到arg变量中 */
	uarg = u64_to_user_ptr(buf.buf_ptr);
	if (copy_from_user(&arg, uarg, sizeof(arg)))
		return -EFAULT;
 
	if (sizeof(arg) + TEE_IOCTL_PARAM_SIZE(arg.num_params) != buf.buf_len)
		return -EINVAL;
 
/* 判定userspace层面传递的参数的个数并在kernelspace中分配对应的空间，将该空间的起
始地址保存在params，然后userspace中的参数数据存放到params中 */
	if (arg.num_params) {
		params = kcalloc(arg.num_params, sizeof(struct tee_param),
				 GFP_KERNEL);
		if (!params)
			return -ENOMEM;
		uparams = uarg->params;
 
 /* 将来自userspace的参数数据保存到刚刚分配的params指向的内存中 */
		rc = params_from_user(ctx, params, arg.num_params, uparams);
		if (rc)
			goto out;
	}
 
/* 调用设备的open_session方法，并将参数传递給该函数  */
	rc = ctx->teedev->desc->ops->open_session(ctx, &arg, params);
	if (rc)
		goto out;
	have_session = true;
 
	if (put_user(arg.session, &uarg->session) ||
	    put_user(arg.ret, &uarg->ret) ||
	    put_user(arg.ret_origin, &uarg->ret_origin)) {
		rc = -EFAULT;
		goto out;
	}
/* 将从secure world中返回的数据保存到userspace对应的buffer中 */
	rc = params_to_user(uparams, arg.num_params, params);
out:
	/*
	 * If we've succeeded to open the session but failed to communicate
	 * it back to user space, close the session again to avoid leakage.
	 */
/* 如果调用不成功则执行close session操作 */
	if (rc && have_session && ctx->teedev->desc->ops->close_session)
 
	if (params) {
		/* Decrease ref count for all valid shared memory pointers */
		for (n = 0; n < arg.num_params; n++)
			if (tee_param_is_memref(params + n) &&
			    params[n].u.memref.shm)
				tee_shm_put(params[n].u.memref.shm);
		kfree(params);
	}
 
	return rc;
}
```

调用设备的open_session操作来完成向OP-TEE发送打开与特定TA的session操作，open_session函数的执行流程如下图所示：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221005122759.png" width="80%" /></div>

整个调用中会调用`optee_do_call_with_arg`函数来完成驱动与secure world之间的交互，该函数的内容如下：

```C
u32 optee_do_call_with_arg(struct tee_context *ctx, phys_addr_t parg)
{
	struct optee *optee = tee_get_drvdata(ctx->teedev);
	struct optee_call_waiter w;
	struct optee_rpc_param param = { };
	u32 ret;
 
/* 设定触发smc操作的第一个参数a0的值为OPTEE_SMC_CALL_WITH_ARG
    通过OPTEE_SMC_CALL_WITH_ARG值可以知道，该函数将会执行std的smc操作 */
	param.a0 = OPTEE_SMC_CALL_WITH_ARG;
	reg_pair_from_64(¶m.a1, ¶m.a2, parg);
	/* Initialize waiter */
/* 初始化调用的等待队列 */
	optee_cq_wait_init(&optee->call_queue, &w);
 
/*进入到loop循环，触发smc操作并等待secure world的返回*/
	while (true) {
		struct arm_smccc_res res;
 
/* 触发smc操作 */
		optee->invoke_fn(param.a0, param.a1, param.a2, param.a3,
				 param.a4, param.a5, param.a6, param.a7,
				 &res);
 
/* 判定secure world是否超时，如果超时，完成一次啊调用，进入下一次循环
    知道secure world端完成open session请求 */
		if (res.a0 == OPTEE_SMC_RETURN_ETHREAD_LIMIT) {
			/*
			 * Out of threads in secure world, wait for a thread
			 * become available.
			 */
			optee_cq_wait_for_completion(&optee->call_queue, &w);
		} else if (OPTEE_SMC_RETURN_IS_RPC(res.a0)) {
/* 处理rpc操作 */
			param.a0 = res.a0;
			param.a1 = res.a1;
			param.a2 = res.a2;
			param.a3 = res.a3;
			optee_handle_rpc(ctx, ¶m);
		} else {
/* 创建session完成之后跳出loop，并返回a0的值 */
			ret = res.a0;
			break;
		}
	}
 
	/*
	 * We're done with our thread in secure world, if there's any
	 * thread waiters wake up one.
	 */
/* 执行等待队列最后完成操作 */
	optee_cq_wait_final(&optee->call_queue, &w);
 
	return ret;
}
```

## 6.4 invoke操作
当完成了session打开之后，用户就可以调用TEEC_InvokeCommand接口来调用对应的TA中执行特定的操作了，TEEC_InvokeCommand函数最终会调用驱动的tee_ioctl_invoke函数来完成具体的操作。该函数内如如下：
```c
static int tee_ioctl_invoke(struct tee_context *ctx,
			    struct tee_ioctl_buf_data __user *ubuf)
{
	int rc;
	size_t n;
	struct tee_ioctl_buf_data buf;
	struct tee_ioctl_invoke_arg __user *uarg;
	struct tee_ioctl_invoke_arg arg;
	struct tee_ioctl_param __user *uparams = NULL;
	struct tee_param *params = NULL;
 
/* 参数检查 */
	if (!ctx->teedev->desc->ops->invoke_func)
		return -EINVAL;
 
/* 数据拷贝到kernel space */
	if (copy_from_user(&buf, ubuf, sizeof(buf)))
		return -EFAULT;
 
	if (buf.buf_len > TEE_MAX_ARG_SIZE ||
	    buf.buf_len < sizeof(struct tee_ioctl_invoke_arg))
		return -EINVAL;
 
	uarg = u64_to_user_ptr(buf.buf_ptr);
	if (copy_from_user(&arg, uarg, sizeof(arg)))
		return -EFAULT;
 
	if (sizeof(arg) + TEE_IOCTL_PARAM_SIZE(arg.num_params) != buf.buf_len)
		return -EINVAL;
 
/* 组合需要传递到secure world中的参数buffer */
	if (arg.num_params) {
		params = kcalloc(arg.num_params, sizeof(struct tee_param),
				 GFP_KERNEL);
		if (!params)
			return -ENOMEM;
		uparams = uarg->params;
		rc = params_from_user(ctx, params, arg.num_params, uparams);
		if (rc)
			goto out;
	}
 
/* 使用对应的session出发smc操作 */
	rc = ctx->teedev->desc->ops->invoke_func(ctx, &arg, params);
	if (rc)
		goto out;
 
/* 检查和解析返回的数据，并将数据拷贝到userspace用户体用的Buffser中 */
	if (put_user(arg.ret, &uarg->ret) ||
	    put_user(arg.ret_origin, &uarg->ret_origin)) {
		rc = -EFAULT;
		goto out;
	}
	rc = params_to_user(uparams, arg.num_params, params);
out:
	if (params) {
		/* Decrease ref count for all valid shared memory pointers */
		for (n = 0; n < arg.num_params; n++)
			if (tee_param_is_memref(params + n) &&
			    params[n].u.memref.shm)
				tee_shm_put(params[n].u.memref.shm);
		kfree(params);
	}
	return rc;
}
```


# 7. tee_supplicant接口在驱动的实现

在[07_OPTEE-OS_系统集成之（五）REE侧上层软件](https://github.com/carloscn/blog/issues/97)一文中介绍了tee_supplicant主要作用，用来实现`secure world端操作REE侧文件系统`，`EMMC的rpmb分区`,`网络socket操作`，`数据库操作`的需求。

tee_supplicant与OP-TEE之间的交互模式类似于生产者与消费者的关系。完成上述需求的整个过程包含驱动接收来自OP-TEE的请求、tee_supplicant从驱动中获取OP-TEE的请求并处理、驱动返回请求操作结果给OP-TEE三部分。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221005124228.png)

当libtee调用驱动来与OP-TEE进行数据的交互的时候，最终会调用`optee_do_call_with_arg`函数完成完成smc的操作，而该函数中有一个loop循环，每次触发smc操作之后会对从secure world中返回的参数`res.a0`进行判断，判定当前从secure world返回的数据是**要执行RPC操作**还是直接**返回到CA**。

如果是来自TEE的RPC请求，则会将请求存放到请求队列req中。然后block住，直到tee_supplicant处理完请求并将`req->c`标记为完成之后才会进入下一个loop，重新出发smc操作，将处理结果返回给TEE。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221005125408.png)


## 7.1 driver获取来自TEE侧的请求

当libteec调用了需要做smc操作的请求之后，最终会调用到驱动的`optee_do_call_with_arg`函数，该函数会进入到死循环，第一条语句就会调用smc操作，进userspace的请求发送到secure world，待从secure world中返回之后。会对返回值进行判定。如果返回的`res.a0`参数是需要驱动做RPC操作，则该函数会调用到`optee_handle_rpc`操作。经过各种参数分析和函数调用之后，程序最后会调用`optee_supp_thrd_req`函数来将来自TEE的请求存放到tee_supplicant的请求队列中。该函数的内容如下：

```C
u32 optee_supp_thrd_req(struct tee_context *ctx, u32 func, size_t num_params,
			struct tee_param *param)
 
{
	struct optee *optee = tee_get_drvdata(ctx->teedev);
	struct optee_supp *supp = &optee->supp;
	struct optee_supp_req *req = kzalloc(sizeof(*req), GFP_KERNEL);
	bool interruptable;
	u32 ret;
 
	if (!req)
		return TEEC_ERROR_OUT_OF_MEMORY;
 
/* 初始化该请求消息的c成员并配置请求数据 */
	init_completion(&req->c);
	req->func = func;
	req->num_params = num_params;
	req->param = param;
 
	/* Insert the request in the request list */
/* 将接受到的请求添加到驱动的TEE请求消息队列中 */
	mutex_lock(&supp->mutex);
	list_add_tail(&req->link, &supp->reqs);
	mutex_unlock(&supp->mutex);
 
	/* Tell an eventual waiter there's a new request */
/* 将supp->reqs_c置位，通知tee_supplicant的receve操作，当前驱动中
   有一个来自TEE的请求 */
	complete(&supp->reqs_c);
 
	/*
	 * Wait for supplicant to process and return result, once we've
	 * returned from wait_for_completion(&req->c) successfully we have
	 * exclusive access again.
	 */
/* block在这里，通过判定req->c是否被置位来判定当前请求是否被处理完毕，
    而req->c的置位是有tee_supplicant的send调用来完成的，如果被置位，则进入到
   while循环中进行返回值的设定并跳出while*/
	while (wait_for_completion_interruptible(&req->c)) {
		mutex_lock(&supp->mutex);
		interruptable = !supp->ctx;
		if (interruptable) {
			/*
			 * There's no supplicant available and since the
			 * supp->mutex currently is held none can
			 * become available until the mutex released
			 * again.
			 *
			 * Interrupting an RPC to supplicant is only
			 * allowed as a way of slightly improving the user
			 * experience in case the supplicant hasn't been
			 * started yet. During normal operation the supplicant
			 * will serve all requests in a timely manner and
			 * interrupting then wouldn't make sense.
			 */
			interruptable = !req->busy;
			if (!req->busy)
				list_del(&req->link);
		}
		mutex_unlock(&supp->mutex);
 
		if (interruptable) {
			req->ret = TEEC_ERROR_COMMUNICATION;
			break;
		}
	}
 
	ret = req->ret;
	kfree(req);
 
	return ret;
}
```

当请求被处理完成之后，函数返回处理后的数据到optee_do_call_with_arg函数中，并进入optee_do_call_with_arg函数中while中的下一次循环，将处理结果返回给secure world.

## 7.2 tee_supplicant从驱动中获取TEE侧的请求
在tee_supplicant会调用read_request函数来从驱动的请求队列中获取当前存在的来自TEE的请求。该函数最终会调用到驱动中的optee_supp_recv函数。该函数的内容如下：
```C
int optee_supp_recv(struct tee_context *ctx, u32 *func, u32 *num_params,
		    struct tee_param *param)
{
	struct tee_device *teedev = ctx->teedev;
	struct optee *optee = tee_get_drvdata(teedev);
	struct optee_supp *supp = &optee->supp;
	struct optee_supp_req *req = NULL;
	int id;
	size_t num_meta;
	int rc;
 
/* 对被用来存放TEE请求参数的数据的buffer进行检查 */
	rc = supp_check_recv_params(*num_params, param, &num_meta);
	if (rc)
		return rc;
 
/* 进入到loop循环中，从 驱动的请求消息队列中获取来自TEE中的请求，直到获取之后才会
跳出该loop*/
	while (true) {
		mutex_lock(&supp->mutex);
/* 尝试从驱动的请求消息队列中获取来自TEE的一条请求 */
		req = supp_pop_entry(supp, *num_params - num_meta, &id);
		mutex_unlock(&supp->mutex);
 
/* 判定是否获取到请求如果获取到了则跳出该loop */
		if (req) {
			if (IS_ERR(req))
				return PTR_ERR(req);
			break;
		}
 
		/*
		 * If we didn't get a request we'll block in
		 * wait_for_completion() to avoid needless spinning.
		 *
		 * This is where supplicant will be hanging most of
		 * the time, let's make this interruptable so we
		 * can easily restart supplicant if needed.
		 */
/* block在这里，直到在optee_supp_thrd_req函数中发送了
complete(&supp->reqs_c)操作后才继续往下执行 */
		if (wait_for_completion_interruptible(&supp->reqs_c))
			return -ERESTARTSYS;
	}
 
/* 设定参数进行异步处理请求的条件 */
	if (num_meta) {
		/*
		 * tee-supplicant support meta parameters -> requsts can be
		 * processed asynchronously.
		 */
		param->attr = TEE_IOCTL_PARAM_ATTR_TYPE_VALUE_INOUT |
			      TEE_IOCTL_PARAM_ATTR_META;
		param->u.value.a = id;
		param->u.value.b = 0;
		param->u.value.c = 0;
	} else {
		mutex_lock(&supp->mutex);
		supp->req_id = id;
		mutex_unlock(&supp->mutex);
	}
 
/* 解析参数，设定tee_supplicant将要执行的具体(加载TA，操作文件系统，操作EMMC的
rpmb分区等)操作和相关参数 */
	*func = req->func;
	*num_params = req->num_params + num_meta;
	memcpy(param + num_meta, req->param,
	       sizeof(struct tee_param) * req->num_params);
 
	return 0;
}
```

从请求消息队列中获取到来自TEE的请求之后，返回到tee_supplicant中继续执行。根据返回的func值和参数执行TEE要求在REE端需要的操作

## 7.3 驱动返回请求操作的结果给TEE侧

当tee_supplicant执行完TEE请求的操作之后，会调用write_response函数来实现将数据返回给TEE。而write_response函数最终会调用到驱动的optee_supp_send函数。该函数主要是调用`complete(&req->c);`操作来完成对该请求的c成员的置位，告诉optee_supp_thrd_req函数执行下一步操作，返回到optee_do_call_with_arg函数中进入该函数中的下一轮loop中，调用smc操作将结果返回给TEE侧。optee_supp_send函数的内容如下：

```C
int optee_supp_send(struct tee_context *ctx, u32 ret, u32 num_params,
		    struct tee_param *param)
{
	struct tee_device *teedev = ctx->teedev;
	struct optee *optee = tee_get_drvdata(teedev);
	struct optee_supp *supp = &optee->supp;
	struct optee_supp_req *req;
	size_t n;
	size_t num_meta;
 
	mutex_lock(&supp->mutex);
/* 驱动中请求队列的pop操作 */
	req = supp_pop_req(supp, num_params, param, &num_meta);
	mutex_unlock(&supp->mutex);
 
	if (IS_ERR(req)) {
		/* Something is wrong, let supplicant restart. */
		return PTR_ERR(req);
	}
 
	/* Update out and in/out parameters */
/* 使用传入的参数，更新请求的参数区域，将需要返回给TEE侧的数据填入到对应的位置 */
	for (n = 0; n < req->num_params; n++) {
		struct tee_param *p = req->param + n;
 
		switch (p->attr & TEE_IOCTL_PARAM_ATTR_TYPE_MASK) {
		case TEE_IOCTL_PARAM_ATTR_TYPE_VALUE_OUTPUT:
		case TEE_IOCTL_PARAM_ATTR_TYPE_VALUE_INOUT:
			p->u.value.a = param[n + num_meta].u.value.a;
			p->u.value.b = param[n + num_meta].u.value.b;
			p->u.value.c = param[n + num_meta].u.value.c;
			break;
		case TEE_IOCTL_PARAM_ATTR_TYPE_MEMREF_OUTPUT:
		case TEE_IOCTL_PARAM_ATTR_TYPE_MEMREF_INOUT:
			p->u.memref.size = param[n + num_meta].u.memref.size;
			break;
		default:
			break;
		}
	}
	req->ret = ret;
 
	/* Let the requesting thread continue */
/* 通知optee_supp_thrd_req函数，一个来自TEE侧的请求已经被处理完毕，
可以继续往下执行 */
	complete(&req->c);
	return 0;
}
```

## 7.4 总结

从tee_supplicant处理来自TEE侧的请求来看主要是有三个点：
-   第一是驱动在触发smc操作之后会进入到loop循环中，根据secure world中的返回值来判定该返回时来自TEE的RPC请求还是最终处理结果，如果是RPC请求，也就是需要驱动或者tee_supplicant执行其他操作，驱动将RPC请求会保存到驱动的请求消息队列中，然后block住等待请求处理结果。
-   第二是在tee_supplicant作为一个常驻进程存在于REE中，它会不停的尝试从驱动的请求消息队列中获取到来自TEE侧的请求。如果请求消息队列中并没有请求则会block住，直到拿到了请求才返回。拿到请求之后会对请求进行解析，然后根据func执行具体的操作。
-   第三是在tee_supplicant处理完来自TEE的请求后，会调用send操作将处理结果存放到该消息队列的参数区域，并使用complete函数通知驱动该请求已经被处理完毕。驱动block住的地方可以继续往下执行，调用smc操作将结果返回给TEE侧。

[^1][^2][^3]
# Ref
[^1]:[linux内核中的subsys_initcall是干什么的?](https://www.bbsmax.com/A/n2d9KQooJD/)
[^2]:[01-OP-TEE驱动篇----驱动编译，加载和初始化（一）.md](https://github.com/BabyMelvin/Aidroid/blob/bd5d1439c2fadbdf4a59721c7ebb7f8d1838580f/fingerprint/op-tee/02-driver/01-OP-TEE%E9%A9%B1%E5%8A%A8%E7%AF%87----%E9%A9%B1%E5%8A%A8%E7%BC%96%E8%AF%91%EF%BC%8C%E5%8A%A0%E8%BD%BD%E5%92%8C%E5%88%9D%E5%A7%8B%E5%8C%96%EF%BC%88%E4%B8%80%EF%BC%89.md)
[^3]:[SMC CALLING CONVENTION  System Software on ARM Platforms.pdf](https://github.com/carloscn/doclib/blob/master/man/arm/standard/SMC%20CALLING%20CONVENTION%20%20System%20Software%20on%20ARM%20Platforms.pdf)
