# 15_OPTEE-OS_内核之（七）系统调用及IPC机制
OP-TEE运行时分为用户空间和内核空间，以此来保证OP-TEE运行时用户空间和内核空间的相互独立。TA程序、OP-TEE提供的一些外部库、各种算法的对外接口都存在于用户空间，而**OP-TEE的线程管理、TA管理、内存管理**等都运行于内核空间。用户空间的程序无法直接访问到内核空间的资源和内存，如果用户空间的程序需要访问内核空间的资源可以通过OP-TEE的系统调用（System Call）的来实现。

OP-TEE按照GP规范定义的大部分接口都是给OP-TEE中的TA使用的。GP统一定义了高级加密标准（Advanced Encryption Standard，ASE）、RSA、安全散列算法（Secure Hash Algorithm，SHA）、哈希消息论证码（Hash-based Message Authentication Code，HMAC）、基于密码的密钥导出算法（Password-Based Key Derivation Function，PBKDF2）等算法的调用接口，该部分在OP-TEE编译时会被编译成libutee.a库文件。TA可通过调用该库中的相关接口来完成对数据的加解密以及签名和验签等操作。如果板级具有硬件密码学引擎实现，调用这些算法接口后最终会使用底层驱动引擎来完成密码学的相关操作。**密码学引擎驱动是处于内核空间的，这也就衍生出了OP-TEE的系统调用的需求**。

进程间通信（Inter-Process Communication，IPC）机制是指系统中进程或线程之间的通信机制，用于实现线程与线程之间进行通信、数据交互等功能。Linux具有多种方式能够实现进程或线程之间的通信和数据共享，例如：消息队列、信号量、共享内存等。而在OP-TEE中**并未**提供如此丰富的IPC方法，本章将介绍OP-TEE中的IPC机制。

# 1. 系统调用
OP-TEE用户空间的接口一般定义成utee_xxx_xxx的形式，而其对应的系统调用则为syscall_xxx_xxx。即在OP-TEE的用户空间调用utee_xxx_xxx函数，OP-TEE最终会调用syscall_xxx_xxx来实现处理，可参考Linux中系统调用的概念。

OP-TEE的系统调用是通过让ARM核进入svc模式来使系统陷入内核态中，然后根据系统调用ID来命中系统调用的内核实现，整个系统调用的过程：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007100023.png)

这个过程可以理解为，当用户的调用触发了进入了optee的系统调用，optee需要使用svc产生异常，进入异常之后进行一些上文的备份，调入异常handler，在handler里面找到系统调用的函数，开始执行，执行完成之后返回恢复上文，最后回到用户空间。

OP-TEE系统调用的关键点是通过svc从OP-TEE用户空间切换到OP-TEE的内核空间。使用切换时带入的系统调用ID，在OP-TEE的系统调用数组中找到对应的函数并执行，完成系统调用后切换ARM核的模式返回到用户空间。

一个系统调用的定义是在用户空间通过UTEE_SYSCALL宏来实现的。在OP-TEE中，所有的utee_xxx类的接口都使用该宏定义在utee_syscalls_asm.S文件中，该宏使用汇编实现`optee_os/lib/libutee/arch/arm/utee_syscalls_a32.S`。

OP-TEE的内核空间中定义了一个系统调用的数组表——`tee_svc_syscall_table`，该数组中包含了当前OP-TEE中支持的所有系统调用在内核空间的实现，该数组定义在optee_os/core/arch/arm/tee/arch_svc.c文件中。由于该数组较大，在此就不贴出。在用户空间中触发svc后，会调用tee_svc_handler函数，该函数会使用在用户空间传入的scn值从tee_svc_syscall_table中查找到系统调用的实现，tee_svc_syscall_table[scn]内容所指向的函数即为系统调用在OP-TEE内核空间的具体实现。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007100557.png)

tee_svc_handler会调用tee_svc_do_call来执行tee_svc_syscall_table[scn]中定义的函数。在执行tee_svc_syscall_table[scn]之前会保存相关寄存器，以便执行完系统调用后恢复到执行系统调用之前的用户空间的状态，而且还需要将用户空间中带入的数据复制到内核空间供tee_svc_syscall_table[scn]中的函数使用。

系统调用主要是给用户空间的接口提供对内核空间接口的调用，使用户空间可以访问到内核空间的资源。例如在使用安全存储功能时，对object的所有操作最终都是在内核空间完成的，包括安全文件查找、文件树建立、RPC请求发送等。所以理解OP-TEE中系统调用的实现，对理解OP-TEE在用户空间提供的接口的具体实现有很大帮助。

# 2. IPC机制

动态TA是以线程的方式运行于OP-TEE的用户空间，OP-TEE的IPC机制用于实现各**线程之间的相互调用、线程调用安全驱动、线程调用OP-TEE内核空间的服务**。**OP-TEE中并未有类似消息队列、信号量等专门用于线程间通信的机制，但OP-TEE提供动态TA调用其他TA或安全驱动的方法和接口，从而实现OP-TEE中各线程间的通信**。

## 2.1 原理
OP-TEE中的IPC机制主要是为满足OP-TEE用户空间运行的线程调用其他线程、静态TA、安全驱动的需求。其原理的核心是利用**系统调用来访问其他线程或者安全驱动**。当线程需要调用其他线程或者安全驱动时，首先会通过系统调用陷入到OPTEE的内核态，然后执行类似CA调用TA的操作，建立会话并通过调用命令的方式让其他TA来完成相应的操作。线程调用安全驱动时，同样是通过调用系统调用陷入到OP-TEE的内核态，然后调用服务或安全驱动提供给OP-TEE内核空间的接口来完成TA对安全驱动和服务的调用。

## 2.2 实现
OP-TEE的IPC机制是通过系统调用陷入到内核中来实现的。调用其他TA的操作有专门的接口，而访问安全驱动和OP-TEE的服务则是通过在内核态中调用服务提供的内核级接口来实现的。

### 2.2.1 TA调用其他TA的实现
一个TA调用其他TA时，OP-TEE通过建立两者间的会话，并调用命令来实现。GP规范定义了如表中的三个接口，这些接口可在OP-TEE的用户空间被调用。

| API名称             | API作用                                        |
| ------------------- | ---------------------------------------------- |
| TEE_OpenTASession   | 创建两个TA之间的session                        |
| TEE_CloseTASession  | 关闭两个TA之间的session                        |
| TEE_InvokeTACommand | 通过创建session和commandID调用另外的TA提供操作 |

当一个TA需要调用其他的TA时，首先需要使用TEE_OpenTASession创建两个TA之间的会话，再使用TEE_InvokeTACommand调用到已经建立的会话的TA中的具体操作，待不再需要调用其他TA时，则调用TEE_InvokeTACommand函数关闭会话来断开两个TA间的联系。

#### 1. TEE_OpenTASession
TEE_OpenTASession的实现与CA中创建与TA的会话的过程大致相同，但TEE_OpenTASession是通过系统调用的方式来触发OP-TEE分配线程并创建会话，而CA则是通过触发安全监控模式调用（smc）来让OP-TEE分配线程并创建会话。

#### 2. TEE_InvokeTACommand
调用TEE_InvokeTACommands时带入命令ID的值就能调用TA中具体的命令，其过程与CA的命令调用操作几乎一致。

## 2.3 TA调用系统服务和安全驱动
**动态TA实现具体功能时需要调用到安全驱动或系统底层的资源**。例如密码学操作、加载TA镜像文件操作、对SE模块的操作等。这些资源提供的接口都处于OP-TEE的内核空间，当用户空间的TA需要使用这些资源来实现具体功能时，则需要让TA的调用操作通过系统调用的方式进入到内核空间，然后再调用特定的接口。

### 2.3.1 OP-TEE中服务和安全驱动的构成
OP-TEE使用系统服务的方式统一管理各功能模块，安全驱动的操作接口会接入到系统服务中，系统服务是在OP-TEE启动过程中执行initcall段中的内容时被启动，service_init的启动等级设置为1，而driver_init的启动等级设置成3。故在OP-TEE的启动过程中，首先会启动使用service_init宏定义的功能函数，再初始化安全驱动。各系统服务、安全驱动和上层的TA之间的关系如图所示。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221007102355.png" width="60%" /></div>

用户态的TA通过系统调用陷入OP-TEE内核空间，然后在对应的系统调用中使用系统服务提供的接口或变量来完成对安全驱动或其他资源的操作。

## 2.4 TA对密码学系统服务的调用实现

### 2.4.1 Libtomcrypt版本（默认）
```
-   some_function()                             (Trusted App) -
[1]   TEE_*()                      User space   (libutee.a)
------- utee_*() ----------------------------------------------
[2]       tee_svc_*()              Kernel space
[3]         crypto_*()                          (libtomcrypt.a and crypto.c)
[4]           /* LibTomCrypt */                 (libtomcrypt.a)
```

TA需要实现计算摘要、产生随机数、加解密、签名验签等操作时就会调用到密码学系统服务提供的接口。OP-TEE的内核空间中有一个变量——crypto_ops，该变量中保存了各种密码学算法的调用接口，在路径optee_os/core/drivers/crypto/crypto_api，其内容如下：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007125736.png)

比如你要使用hash的operation，那就定义一个这样的结构体：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007125834.png)

然后，把do_hash_xxx的函数的实体挂载上面，例如do_hash_update，这样比较方便的使用钩子函数，我们可以对ta进行算法替换：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007125928.png)

#### crypto service的初始化
OP-TEE在启动时会调用crypto_ops.init指定的函数初始化整个密码学系统服务，对于OP-TEE默认即调用tee_ltc_init函数来初始化密码学系统服务，该函数optee_os/core/lib/libtomcrypt/tomcrypt.c。对于TA使用的底层库，我们也可以置换成其他的crypto的库。

### 2.4.2 hw版本[^4]

OPTEE默认在内核内部集成了一个libtomcrypt的三方轻量库用于加解密做hash这些加密算法学的操作。实际在工程中，由于SoC有secure engine之类的SoC设计，可以使用硬件加速的方法来承担这部分加密算法加速。我们可以用这些硬件加速器的驱动来完成匹配。

加密算法例如，例如hash、mac、skcipher都遵循下面的步骤：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007133956.png)

crypto hash * API uses an ops struct to perform operations：

```C
struct crypto_hash_ops {
	TEE_Result (* init ) (...) ;
	TEE_Result (* update ) (...) ;
	TEE_Result (* final ) (...) ;
	void (* free_ctx ) (...) ;
	void (* copy_state ) (...) ;
};
```

我们可以把硬件实现的驱动挂载到上面去。OP-TEE提供了这个接口`drvcrypt`。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007134338.png)

我见到的做法是，使用mbedtls来封装这些操作，在mbedtls使用ALT来做出选择。

### 2.4.3 硬件加速应用（Linux）
在Linux上面也可以使用加密库，通过以下方式[^2]：
* 软件库（可以自定义）
* OpenSSL
	* Linux Kernel Crypto API （LCA）
	* OP-TEE

#### OpenSSL
在Linux上的OpenSSL提供了加密算法，可以使用软件实现，也可以通过LCA来实现。

##### LCA
LCA的框架如图所示，可以使用AF_ALG engine的方法，还可以使用Cryptodev engine两个路径。最终最用到硬件加速引擎上面：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007140710.png)

##### OP-TEE
还有一种方法是把OPSSL的引擎换成OP-TEE，使用TA的方法访问[^3]：
![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007140928.png)

这样的话在TA应用里面实现具体的算法，对应出来。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007141136.png)


#### TA调用具体算法的实现

调用crypto_ops中的接口时，会根据需要被调用密码学算法的名称从数组变量中找到对应的元素，然后使用元素中保存的算法操作接口来完成密码学操作。如果芯片集成了硬件加解密引擎，加密算法的实现，则可使用硬件cipher驱动提供的接口来完成。本节以调用SHA1算法为例介绍其实现过程。

在TA中如果需要使用SHA1算法计算数据的摘要，则需要调用TEE_DigestUpdate接口来实现，该函数的完整执行过程如图所示。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221007131616.png)


## 2.5 对SE功能模块进行操作的系统服务

安全元件（Secure Element）简称 SE，通常以芯片或 SD 卡等形式提供。为防止外部恶意解析攻击，保护数据安全，在芯片中具有加密/解密逻辑电路[^1]。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221007132348.png" width="80%" /></div>

SE 具备极强的安全等级，但对外提供的接口和功能极其有限。首先 CPU 性能极低，无法处理大量数据，其次 SE 与智能设备通常采用非常缓慢的串口连接。SE 的应用场景有限，主要关注于保护内部密钥，一般用于对安全要求很高的场景。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221007132125.png" width="80%" /></div>

>Secure Elements (SEs) may be connected to the REE or exclusively to the TEE. 
• An SE connected exclusively to the TEE is accessible by a TA without using any resources from the REE. Thus the communication is considered trusted. 
• An SE connected to the REE is accessible by a TA using resources lying in the REE. Proprietary security protection such as a secure channel has to be implemented to protect the communication between the TA and the SE against attacks in the REE.

在OP-TEE内核空间调用类似tee_se_reader_xxx的接口会调用到OP-TEE的SE系统服务，用于操作具体的SE模块。若需要在TA中操作SE模块，可将tee_se_reader_xxx类型的接口重新封装成系统调用，然后在TA中调用封装的接口就能实现TA对SE模块的操作。在OP-TEE中要使用具体的SE模块需要初始化SE功能模块的系统服务，并挂载具体SE模块的驱动。

SE模块的系统服务是通过在OP-TEE启动过程中调用tee_se_manager_init函数来实现的，该函数只会初始化该系统服务的上下文空间。

## 2.6 加载TA的镜像服务
当CA调用libteec库中用于创建与某个动态TA的会话时，会从REE侧的文件系统中加载TA镜像文件到OP-TEE，加载TA镜像的过程就会使用到该系统服务提供的接口函数。

[12_OPTEE-OS_内核之（四）对TA请求的处理](https://github.com/carloscn/blog/issues/102)详细介绍了OP-TEE创建会话的实现过程。OP-TEE会使用tee_ta_init_user_ta_session函数来完成加载TA镜像并初始化会话的操作。加载TA镜像文件时，会使用user_ta_store变量中的接口发送RPC请求，通知tee_supplicant对REE侧文件系统中的TA镜像文件执行打开、读取、获取TA镜像文件大小、关闭TA镜像文件的操作。user_ta_store变量在该系统服务启动时被赋值，具体函数内容如下：
```C
static const struct user_ta_store_ops ops = {
    .open = ta_open,          //发送RPC请求使tee_supplicant打开TA镜像文件
    .get_size = ta_get_size,  //发送RPC请求,获取TA镜像文件的大小
    .read = ta_read,          //发送RPC请求读取TA镜像的内容
    .close = ta_close,        }; //发送RPC请求关闭打开的TA镜像文件
/* OP-TEE启动时被调用,使用service_init宏将该函数编译到initcall段中 */
static TEE_Result register_supplicant_user_ta(void) {
    return tee_ta_register_ta_store(&ops); } /* 将user_ta_store变量的地址赋值成ops */

TEE_Result tee_ta_register_ta_store(const struct user_ta_store_ops *ops){
    user_ta_store = ops;
    return TEE_SUCCESS;
}
```

# 3. 总结
系统调用主要是给用户空间的接口提供对内核空间接口的调用，使用户空间可以访问到内核空间的资源。例如在使用安全存储功能时，对object的所有操作最终都是在内核空间完成的，包括安全文件查找、文件树建立、RPC请求发送等。所以理解OP-TEE中系统调用的实现，对理解OP-TEE在用户空间提供的接口的具体实现有很大帮助。

本章还介绍了OP-TEE中各种系统服务以及TA调用另外一个TA的原理和实现。每个TA具有独立的运行空间，OP-TEE中的一个TA调用另一个TA执行特定操作的过程是OP-TEE中的一种IPC的方式。OP-TEE中各种系统服务起到类似框架层的作用，安全驱动或其他子模块提供的操作接口会接入到对应的系统服务中。系统服务通过接口变量或其他方式将操作接口暴露给OP-TEE的内核空间，用户空间的TA通过系统调用的方式在OP-TEE内核空间调用这些接口，从而实现TA对安全驱动或其他模块的资源操作。

# Ref
[^1]:[聊一聊可信执行环境](https://copyfuture.com/blogs-details/20210507145009218c)
[^2]:[Linux Kernel 密碼學演算法實作流程](https://szlin.me/2017/04/05/linux-kernel-%E5%AF%86%E7%A2%BC%E5%AD%B8%E6%BC%94%E7%AE%97%E6%B3%95%E5%AF%A6%E4%BD%9C%E6%B5%81%E7%A8%8B/)
[^3]:[OpenVPN authentication hardened with ARM TrustZone](https://www.amongbytes.com/post/20210112-optee-openssl-engine/)
[^4]:[using op-tee as a cryptography engine](https://github.com/carloscn/doclib/blob/master/ppt/arm/optee_cryptograph_elc2021.pdf)






