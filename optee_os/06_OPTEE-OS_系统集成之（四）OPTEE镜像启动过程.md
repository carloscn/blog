# 06_OPTEE-OS_系统集成之（四）OPTEE镜像启动过程
**我们来总结一下关于ATF启动各个步骤的目的，由此引出OPTEE的启动过程**。
* **BootROM（EL3）**: 加载**ATF的bl1**的二进制文件到flash（可能有对二进制格式的解析）、引导bl1
* **ATF-bl1（EL3/S-EL1）** : 初始化CPU、 设定异常向量、将bl2的镜像加载到安全RAM中，跳转到bl2
* **ATF-bl2（EL3）**: bl2镜像将为后续镜像的加载执行相关的初始化操作，主要是内存、MMU、串口以及EL3软件运行环境的设置，并且**加载bl31/bl32/bl32**的镜像到内存中。
* **ATF-bl31（EL3）**：在bl2中触发安全监控模式调用后会跳转到bl31中执行，bl31最主要的作用是**建立EL3运行态的软件配置，在该阶段会完成各种类型的安全监控模式调用ID的注册和对应的ARM核状态的切换**，bl31运行在EL3。
* **ATF-bl32（S-EL1）**：bl31中的`runtime_svc_init`函数会初始化OP-TEE对应的服务，通过调用该服务项的初始化函数来完成OP-TEE的启动。对于OP-TEE的服务项会通过DECLARE_RT_SVC宏在编译时被存放到**rt_svc_des**段中。该段中的init成员会被初始化成opteed_setup函数，由此开始进入到OP-TEE OS的启动。如图所示为ATF对于optee镜像启动的初始化过程。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221004101541.png" width="55%" />
</div>

# 1. OP-TEE的入口函数
OP-TEE镜像的入口函数是在编译OP-TEE OS时通过链接文件来确定的，OP-TEE在编译时是按照optee_os/core/arch/arm/kernel/kern.ld.S文件链接生 成OP-TEE OS的镜像文件，在kern.ld.S文件中通过ENTRY宏来指定OP-TEE OS的入口函数，在OP- TEE中指定的入口函数是_start，对于ARM32位系统，该函数定义在`optee_os/core/arch/arm/generic_entry_a32.S`文件中，对于ARM64位系统而言，该函数定义在`optee_os/core/arch/arm/generic_entry_a64.S`文件中。

![kern.ld.S - _start](https://raw.githubusercontent.com/carloscn/images/main/typora20221004130334.png)

`_start`入口函数在`generic_entry_a64.S`，参考： https://github.com/carloscn/user-optee-os/blob/master/core/arch/arm/kernel/entry_a64.S

![generic_entry_a64.S - _start](https://raw.githubusercontent.com/carloscn/images/main/typora20221004130547.png)


## 1.1 OP-TEE的内核初始化过程

`_start`会调用reset函数进入OP-TEE OS的启动过程。由于对称多处理（Symmetrical Multi-Processing，SMP）架构的原因，在reset函数中会对 主核和从核进行不同的启动操作，分别调用`reset_primary`函数和`reset_secondary`函数来完成。

ARM64的OP-TEE的_start函数定义在generic_entry_a64.S文件中，而且该函数不像ARM32位系统一样会进入reset中去执行OP-TEE启 动，而是直接在_start函数中就完成整个启动过程，在进行初始化操作之前会注册一个异常向量表，该异常向量表会在唤醒从核阶段被使用，当主核通知唤醒从核时，从核会查找该异常向量表，然后命中对应的处理函数并执行从核的启动操作。**ARM64的OP-TEE的启动过程与ARM32的OP-TEE的启动过程几乎一样**。

### 1.1.1 reset入口函数执行内容 (32位)
reset函数是主核和从核启动的第一个函数，该函数的执行流程如图：

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221004132713.png" width="55%" />
</div>

进入到reset函数后，系统会将`_start`的地址写入`VBAR`寄存器作为中断向量表的起始地址使用，在启动从核时，从核知道会到该地址去获取应该执行代码来完成从核的启动。(从这个地方也可以看到，从核的启动是依赖于ARM的异常处理的handler)

以下图片是Linux启动多核的时候的图片，我们可以借鉴以下思路[^1]：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221004133108.png)

所以启动从核的时候要把从核的入口函数写到异常向量表，当异常发生的时候调用调用这个函数。

```Assembly
LOCAL_FUNC reset , : 
UNWIND( .fnstart) 
UNWIND( .cantunwind) 
    bootargs_entry //获取启动带入的参数,主要是启动地址、device tree地址等 
    /* 使能对齐检查并禁用数据和指令缓存 */ 
    read_sctlr r0 //读取sctlr中的数据,获取当前CPU控制寄存器中的值 
#if defined(CFG_SCTLR_ALIGNMENT_CHECK) 
    orr r0, r0, #SCTLR_A //设定对齐校验 
#else
    bic r0, r0, #SCTLR_A 
#endif
    bic r0, r0, #SCTLR_C //关闭数据cache 
    bic r0, r0, #SCTLR_I //关闭指令cache 
#if defined(CFG_HWSUPP_MEM_PERM_WXN) && defined(CFG_CORE_RWDATA_NOEXEC)
    orr r0, r0, #(SCTLR_WXN | SCTLR_UWXN) 
#endif
    write_sctlr r0 //将r0写入到sctlr中,用于关闭cache 
    isb 
    /* 早期ARM核安全监控模式态的特殊配置 */ 
    bl plat_cpu_reset_early //执行CPU早期初始化 
    ldr r0, =_start //设定r0寄存器的值为_start函数的地址 
    write_vbar r0 //将_start函数的地址写入VBAR寄存器中,用于启动时使用 
#if defined(CFG_WITH_ARM_TRUSTED_FW) 
    b reset_primary //支持ATF时跳转到reset_primary中执行 
#else
    bl get_core_pos //判定当前CPU CORE的编号 
    cmp r0, #0 //将获得的CPU编号与0对比 
    beq reset_primary //如果当前core是主核,则使用reset_primary进行初始化 
    b reset_secondary //如果当前core是从核,则使用reset_secondary进行初始化 
#endif 
UNWIND( .fnend) 
END_FUNC reset
```

- `plat_cpu_reset_early`函数将会设定SCR寄存器中的安全标志位，用于标记当前CPU是处于安全世界状态中，并且将`_start`地址写入`VBAR`寄存器，用于 在需要启动从核时系统能找到启动代码的入口地址。
- `reset_primary`函数是主核启动代码的入口函数，该函数将会启动主核的基本初始化、配置运行环境，然后再开始执行唤醒从核的操作。
- **对于从核的唤醒操作，如果系统支持PSCI，从核的唤醒是在REE OS启动时，发送PSCI给EL3或Monitor模式的代码来启动从核**；
- **如果不使用PSCI，而是选择在OP-TEE中使能`CFG_SYNC_BOOT_CPU`，则OP-TEE会在主核启动结束后唤醒从核**。

>PSCI, 是Power State Coordination Interface的缩写，是由ARM定义的电源管理接口规范，ARM也提供了官方的设计文档《Arm Power State Coordination Interface Platform Design Document》

### 1.1.2 reset_primary
本小节以`CONFIG_BOOT_SYNC_CPU`（也就是不支持PSCI的，使能为例，在使能PSCI系统中，不需要使能此宏）。reset_primary函数是OP-TEE对CPU主核进行初始化操作的函数，该函数会初始化系统的MMU，并调用`generic_boot_init_primary`函数完成OP-TEE运行环境的建立，然后触发sev操作来唤醒从核，待所有CPU核都启动完成之后，OP-TEE会触发安全监控 模式调用（smc），通知系统OP-TEE启动已完成并将CPU的状态切换回到正常世界状态，该函数的执行流程如图：

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221004150322.png" width="77%" />
</div>

代码可以参考： https://github.com/carloscn/user-optee-os/blob/master/core/arch/arm/kernel/entry_a32.S#L364

这里要特别注意的是：`generic_boot_init_primary`函数是OP-TEE建立系统运行环境的入口函数，该函数会进行建立线程运行空间、初始化OP-TEE内核组件等操作。

### 1.1.3 _generic_boot_init_primary

`generic_boot_init_primary`函数会调用`init_primary_helper`函数来完成系统运行环境的建立，如果系统支持ATF，则该函数会返回OP-TEE的处理句柄，该处理句柄主要包含各种安全监控模式调用的处理函数、安全世界状态（SWS）的中断以及其他事件的处理函数，ATF中的bl31解析完安全监控模式调用或中断请求后会在安全世界状态调用该处理句柄来处理对应的事件。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221004151626.png" width="77%" />
</div>

### 1.1.4 call_initcalls函数
init_teecore函数通过调用call_initcalls来启动系统的服务以及安全驱动的挂载，该函数的内容如下：

这个函数是一个弱符号的定义，

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221004151901.png)

在执行call_initcalls函数之前，系统已完成了memory、CPU相关设置、中断控制器、共享内存、 线程堆栈设置、TA运行内存的分配等操作。所以这里call_initcalls以C语言函数的形式呈现。call_initcalls是通过遍历OP-TEE镜像文件的_initcall段中从_initcall_start到_initcall_end之间的所有函数。这个技术我们可以学习一下。

OP-TEE镜像文件中`_initcalls`段的内容是通过使 用`__define_initcall`宏来告知编译器的，在编译时会将使用该宏定义的函数保存到OP-TEE镜像文件的 `_initcall`段中。该宏定义如下：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221004152247.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221004152358.png)

- initcall_t：是一个函数指针类型(typedef int(*initcall_t)(void))。
- SCA....本质是个__attribute__((__section__())：将fn对象放在一个由括号中的名称指定的section中。

例如：`__define_initcall("1", init_operation)`

则该宏的作用是声明一个名称为 __initcall_init_operation的函数指针，将该函数指针初始化为init_operation，并在编译时将该函数的内容存放在名称为“.initcall1”的段中。 core/arch/arm/kernel/kern.ld.S文件中存在如下内容：

```
__initcall_start = .; 
KEEP(*(.initcall1)) 
KEEP(*(.initcall2)) 
KEEP(*(.initcall3)) 
KEEP(*(.initcall4))
__initcall_end = .;
```

即在`__initcall_start`到`__initcall_end`之间保存的 是initcall1到initcall4之间的内容，而在整个OP-TEE源代码的`core/include/initcall.h`文件中， __define_initcall宏被使用的情况如下：

```C
#define __define_initcall(level, fn) \ 
#define service_init(fn) __define_initcall("1", fn) 
#define service_init_late(fn) __define_initcall("2", fn) 
#define driver_init(fn) __define_initcall("3", fn) 
#define driver_init_late(fn) __define_initcall("4", fn)
```

所以遍历执行从`__initcall_start`到`__initcall_end`之间的内容就是启动OP-TEE的服务以及完整安全驱动的挂载。

## 1.2 OP-TEE服务项的启动

OP-TEE服务项的启动分为：`service_init`以及`service_init_late`，需要被启动的服务项通过使用这两个宏，在编译时，相关服务的内容将会被保存到 initcall1和initcall2中。

### 1.2.1 service_init
在OP-TEE使用中使用service_init宏定义的服务项如下：
```C
service_init(register_supplicant_user_ta); 
service_init(verify_pseudo_tas_conformance); 
service_init(tee_cryp_init); 
service_init(tee_se_manager_init);
```
如果开发者有实际需求，可以将自己希望添加的服务项功能按照相同的方式添加到系统中。在当前的OP-TEE中默认是启动上述四个服务，分别定义在以下文件：

|service|path|
|----|----|
|register_supplicant_user_ta|core/arch/arm/kernel/ree_fs_ta.c|
|verify_pseudo_tas_conformance|core/arch/arm/kernel/pseudo_ta.c|
|tee_cryp_init|core/tee/tee_cryp_utl.c|
|tee_se_manager_init| core/tee/se/manager.c|

#### register_supplicant_user_ta
该操作主要是注册OP-TEE加载REE侧的TA镜像时需要使用的操作接口，当REE侧执行open session操作时，**TEE侧会根据UUID的值在REE侧的文件系统中查找该文件**，然后通过RPC请求通知tee_supplicant从REE的文件系统中读取与UUID对应的TA镜像文件的内容并传递到TEE侧。

#### verify_pseudo_tas_conformance
该函数主要是用来校验OP-TEE中静态TA的合法性，需要检查OP-TEE OS中静态TA的UUID、函数指针以及相关的flag。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221004153609.png)

OP-TEE OS镜像文件中的 `__start_ta_head_section`与`__stop_ta_head_section`之间保存的是OP-TEE所有静态TA的内容，其值的定义见`core/arch/arm/kernel/kern.ld.S`文件，分别表示`ta_head_section`段的起始地址和末端地址。 

在编译OP-TEE的静态TA时，使用`pseudo_ta_register`宏来告知编译器将静态TA的内容保存到`ta_head_section`段中，该宏定义在 core/arch/arm/include/kernel/pseudo_ta.h文件中，内容如下：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221004153817.png)

共有六个静态TA在OP-TEE编译时会被打包进OP-TEE的镜像文件中，分别如下：

|ta|path|
|--------|-------|
|gprof| core/arch/arm/pta/gprof.c |
|interrupt_tests.ta| core/arch/arm/pta/Iiterrupt_tests.c |
|stats.ta| core/arch/arm/pta/stats.c |
|se_api_self_tests.ta| core/arch/arm/pta/se_api_self_tests.c|
|socket| core/arch/arm/tee/pta_socket.c |
|invoke_tests.pta| core/arch/arm/pta/pta_invoke_test.c|

#### tee_crypt_init
该部分主要完成OP-TEE提供的密码学接口功能的初始化操作，调用crypto_ops结构体中的init进 行初始化操作，该结构体变量定义在core/lib/libtomcrypt/src/tee_ltc_provider.c文件中，变量中定义了各种算法的操作函数指针。完成注册后，TA就可以通过调用该变量中的对应函数指针来实现OP-TEE中各种密码学算法接口的调用。

#### tee_se_manager_init
该部分主要完成对SE（secure engine）模块的管理，为上层提供对SE模块的操作接口。

### 1.2.2 service_init_late
`service_init_late`宏定义的内容将会在编译时被 链接到OP-TEE镜像文件的initcall2段中，OP-TEE中使用该宏来定义OP-TEE中使用的**密钥管理操作**， 在`core/tee/tee_fs_key_manager.c`文件中，使用该宏来将`tee_fs_key_manager`函数保存到`initcall2`段中， 在OP-TEE启动时被调用，用来生成或读取OP-TEE在使用时会使用到的key，该函数内容如下：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221004154558.png)

这些key将会在使用安全存储功能时用到，用于生成加密、解密安全文件的FEK，其中`tee_otp_get_hw_unique_key`函数可根据不同的平台进行修改，只要保证读取到的值的唯一性且安全即可，当前一般做法是读取一次性编程区域（One Time Programmable，OTP）或efuse中的值，该值将在芯片生产或者工厂整机生产时烧录到OTP中，当然也有其他的实现方式。

## 1.3 OP-TEE驱动的挂载

安全设备在使用之前都需要执行一定的配置和初始化，而该部分操作是在OP-TEE启动时执行的。OP-TEE编译时通过使用`driver_init`宏和 `driver_init_late`宏来实现将安全设备驱动编译到OP- TEE OS镜像文件中，使用这两个宏定义设备驱动后，安全设备驱动的初始化操作将会被编译到OP- TEE镜像文件的`initcall3`和`initcall4`段中，以Hikey为例，其使用了`driver_init`宏来定义`peripherals_init`的初始化操作，所以在使用hikey运行OP-TEE时会去挂载外围安全设备并执行相关的初始化。


# Ref
[^1]:[Booting ARM Linux on MPCore](https://medium.com/@srinivasrao.in/booting-arm-linux-on-mpcore-95db62dabf50)
[^2]:[]()
[^3]:[]()
[^4]:[]()
[^5]:[]()
[^6]:[]()
[^7]:[]()
[^8]:[]()
[^9]:[]()
[^10]:[]()
[^11]:[]()
[^12]:[]()
[^13]:[]()
[^14]:[]()
[^15]:[]()
[^16]:[]()
[^17]:[]()
[^18]:[]()
[^19]:[]()
[^20]:[]()