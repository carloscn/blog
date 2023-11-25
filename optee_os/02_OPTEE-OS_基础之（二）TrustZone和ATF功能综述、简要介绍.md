# 02_OPTEE-OS_基础之（二）TrustZone和ATF功能综述、简要介绍

TrustZone的原理以及在ARMv7和 ARMv8架构下TrustZone技术实现的差异。TrustZone对系统实现了硬件隔离，将系统资源划分成安全和非安全两种类型，同时在系统总线上增加安全读写信号位，通过读取安全读写信号位电平来确定当前处理器的工作状态，从而判断是否具有该资源的访问权限。因此，TrustZone从硬件级别实现了对系统资源的保护。

ARM可信任固件（ARM Trusted Firmware，ATF）是由ARM官方提供的底层固件，该固件统一 了ARM底层接口标准，如电源状态控制接口（Power Status Control Interface，PSCI）、安全启 动需求（Trusted Board Boot Requirements， TBBR）、安全世界状态（SWS）与正常世界状态 （NWS）切换的安全监控模式调用（secure monitor call，smc）操作等。ATF旨在将ARM底层的操作统一使代码能够重用和便于移植。

# 1. TrustZone
我们来了解一下TrustZone在硬件上提供了哪些支持 ，能够实现物理的隔离。ARM早在ARMv6架构中就引入了TrustZone技术，且在ARMv7和 ARMv8中得到增强，TrustZone技术能提供芯片级别对硬件资源的保护和隔离，当前在手机芯片领域已被广泛应用。

## 1.2 一个支持TZ的SoC设计
一个完整的片上系统（System on Chip，SoC） 由ARM核、系统总线、片上RAM、片上ROM以及其他外围设备组件构成。只有支持TrustZone技术的ARM核配合安全扩展组件，才能为整个系统提供芯片硬件级别的保护和隔离。如下图所示，为一个使能了TZ的SoC系统。

![使能trustzone的SoC系统](https://raw.githubusercontent.com/carloscn/images/main/typora20221001104719.png)

**对于TZ我们需要关注的是，在硬件上TrustZone要起到下面的作用**：
* 隔离功能（安全状态和非安全状态）
* 外设和内存 （物理上分开）
* 总线请求

支持TrustZone技术的ARM核在运行时将工作状态划分为两种：**安全状态和非安全状态**。当处理器核处于安全状态时只能运行TEE侧的代码，且具有REE侧地址空间的访问权限。当处理器核处于非安全状态时只能运行REE侧的代码，且只能通过事先定义好的客户端接口来获取TEE侧中特定的数据和调用特定的功能。

系统通过调用安全监控模式调用（secure monitor call，smc）指令（或者依靠中断）实现**ARM核的安全状态与非安全状态之间的切换**。而ARM核对系统资源的访问请求是否合法，则由SoC上的安全组件通过判定ARM核发送到SoC系统总线上的访问请求中的安全 状态读写信号位（Non-secure bit，NS bit）来决定。只有当ARM核处于安全状态（NS bit=0）时发送到系统总线上的读写操作才会被识别为安全读写操作，对应TEE侧的数据资源才能被访问。反之，当ARM核处于非安全状态（NS bit=1）时，ARM核发送到系统总线上的读写操作请求会被作为非安全读写操作，**安全组件会根据对资源的访问权限配置来决定是否响应该访问请求**。这也是TrustZone技术能实现对系统资源硬件级别的保护和隔离的根本原因。

### 1.2.1 ARMv7 的TZ

ARMv7架构中使用了TrustZone技术的系统软件层面的框图[^1]：

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221001104944.png" width="85%" />
</div>

在ARMv7架构中CPU在运行时具有不同的特权等级，分别是PL0（USR）、PL1（FIQ/IRQ、 SYS、ABT、SVC、UND和MON）以及PL2（Hyp），即ARMv7架构在原有七种模式之上扩展了**Monitor模式和Hyp模式**。Hyp模式是ARM核用于实现虚拟化技术的一种模式。系统只有在Monitor模式下才能实现安全状态和非安全状态的切换。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221001110015.png" width="66%" />
</div>

当系统在REE侧或者TEE侧运行时，系统执行`SMC`（安全监控模式调用）指令进入Monitor模式， 通过判定系统SCR寄存器中对应的值来确定请求来 源（REE/TEE）以及发送目标（REE/TEE），相关寄存器中的值只有当系统处于安全态时才可以更改。具体如何转变可以参考：[09_OPTEE-OS_内核之（一）ARM核安全态和非安全态的切换](https://github.com/carloscn/blog/issues/99)

### 1.2.2 ARMv8的TZ

在ARMv8架构中改用执行等级（Execution Level，EL）EL0～EL3来定义ARM核的运行等级， 其中EL0～EL2等级分为安全态和非安全态。 ARMv8架构与ARMv7架构中ARM核运行权限的对应关系如图所示[^2]：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221001105958.png)

ARMv7架构中的PL0（USR）对应ARMv8架构中的EL0；PL1（SVC/ABT/IRQ/FIQ/UND/SYS）对应ARMv8架构中的EL1，ARMv7架构中的Hyp模式对应ARMv8架构中的EL2，而ARMv7架构中的 Mon（Monitor）则对应于ARMv8架构中的EL3。 ARMv8架构同样也是使用安全监控模式调用指令使处理器进入EL3，在EL3中运行的代码负责处理器安全状态和非安全状态的切换，其中关于TEE和REE切换的处理方式与ARMv7架构中Monitor模式下的处理方式类似，具体如何转变可以参考：[09_OPTEE-OS_内核之（一）ARM核安全态和非安全态的切换](https://github.com/carloscn/blog/issues/99)。


<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221001143627.png" width="100%" />
</div>

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221001143648.png)

# 2. ARM安全扩展组件
ARM安全扩展组件，相当于对于TZ的实现，就类似于我们实现温湿度测量仪，必须要实现温度传感器采集、湿度传感器采集还有显示面板这些子模块的功能。对于TZ也是一样，实现TZ在ARM上必然要进行安全扩展组件的支持，以支撑TZ的实现。

TrustZone技术之所以能提高系统的安全性，是因为对外部资源和内存资源的硬件隔离。这些硬件隔离包括**片内内存隔离**（中断隔离、片上RAM和ROM的隔离、片外RAM和ROM的隔离）、**外围设备的硬件隔离**、**外部RAM和ROM的隔离**等。实现硬件层面的各种隔离，需要对整个系统的硬件和处理器核做出相应的扩展。这些扩展包括： 
* 对**处理器核的虚拟化**，也就是将ARM处理器的运行状态分为安全态和非安全态。 
* 对**总线的扩展**，增加安全位读写信号线。
* 对**内存管理单元（Memory Management Unit，MMU）的扩展**，增加页表的安全位。[14_ARMv8_内存管理（二）-ARM的MMU设计](https://github.com/carloscn/blog/issues/54)
* 对**缓存（Cache）的扩展**，增加安全位。 [17_ARMv8_高速缓存（二）ARM cache设计](https://github.com/carloscn/blog/issues/58)
* 对**快表（TLB）的扩展**，增加安全位。[19_ARMv8_TLB管理(Translation Lookaside buffer)](https://github.com/carloscn/blog/issues/60)
* 对其他**外围组件进行了相应的扩展**，提供安全操作权限控制和安全操作信号。

我们围绕这几项来阐述一下ARM的扩展组件设计，加深对于安全访问的理解。

**必选组件**：
-   AMBA3 AXI总线：安全机制基础设施。
-   ARMv8A Core El2：虚拟安全和非安全核。
-   TZASC(TrustZone Address Space Controller)：将内存分成多个区域，每个区域可以单独配置为安全或非安全区域。只能用于内存设备，不能用于块设备。
-   TZPC(TrustZone Protection Controller)：根据需要控制外设安全特性。

**可选组件**：
-   TZMA(TrusztZone Memory Adapter)：将片上RAM和ROM分成不同安全和非安全区域。
-   AXI-to-APB bridge：桥接APB总线，配合TZPC使APB总线外设支持TrustZone安全特性。

## 2.1 AXI总线上安全状态位的扩展
为了支持TrustZone技术，控制处理器在不同状态下对硬件资源访问的权限，ARM对先进可扩展接口（Advanced eXtensible Interface，AXI）系统总线进行了扩展。在原有AXI总线基础上对每一个**读写信道增加了一个额外的控制信号位**，用来表示当前的读写操作是安全操作还是非安全操作，该信号位称为安全状态位（NS bit）或者非安全状态位 （Non-Secure bit）。 

* **AWPROT[1]**：总线写事务——低位表示安全写事务操作，高位表示非安全写事务操作。 
* **ARPROT[1]**：总线读事务——低位表示安全读事务操作，高位表示非安全读事务操作。 

当主设备通过总线发起读写操作时，从设备或者外围资源同时也需要将对应的PROT控制信号发送到总线上。总线或者从设备的解码逻辑必须能够解析该PROT控制信号，以便保证安全设备在非安全态下不被非法访问。所有的非安全主设备必须将安全状态位置成高位，这样就能够保证非安全主设备无法访问到安全从设备。如果一个非安全主设备 试图访问一个安全从设备，将会在总线或者从设备上触发一个错误操作，至于该错误如何处理就依赖于从设备的处理逻辑和总线的配置。通常这种非法操作最终将产生一个SLVERR（slave error）或者DECERR（decode error）。

简单来说，AXI总线能够识别对于安全和非安全设备的访问（读写粒度的识别）。

主控告知访问权限，内存系统决定是否允许访问。内存系统权限检查一般是通过互联总线。比如NIC-400可以设置如下属性：
-   **安全**：仅安全访问可以通过，互联总线对非安全访问产生异常，非安全访问不会抵达设备。
-   **非安全**：仅非安全访问可以通过，互联总线对安全访问产生异常，安全访问不会抵达设备。
-   **启动可配置**：系统初始化时可以配置设备为安全或非安全。默认是安全。
-   **TrustZone aware**：互联总线允许所有访问通过，连接的设备自身负责隔离。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221001132232.png" width="77.7%" />
</div>

A系列处理器都是TrustZone aware的，每次总线访问发送争取的安全状态。但还存在一些非处理器的总线主控，比如GPU、DMA等。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221001132629.png)

因此将总线主控分为两种：
-   TrustZone aware：每次总线访问都能提供合适安全信息。
-   Non-TrustZone  aware：此类主设备访问总显示无法提供安全信息、或每次都是同样安全信息。

对于Non-Trusted-aware需要采取的措施有：
-   设计时固定地址：给设备固定的访问地址。
-   可配逻辑单元：为主设备访问添加安全信息逻辑单元。
-   SMMU：对于一个可信主设备，SMMU可以向安全状态下的MMU一样处理。

## 2.2 AXI-to-APB桥的作用

TrustZone同样能够保护外围设备的安全，例如：中断控制、时钟、I/O设备，因此Trust-Zone架构还 能用来解决更加广泛的安全问题。比如一个安全中断控制器和安全时钟允许一个非中断的安全任务来监控系统，能够为DRM提供可靠的时钟，能够为用 户提供一个安全的输入设备从而保证用户密码数据不会被恶意软件窃取。 

AMBA3规范包含了一个低门数、低带宽的外设总线，被称作外设总线（Advanced Peripheral Bus，APB），APB通过AXI-to-APB桥连接到系统总线上。而APB总线并不具有安全状态位，**为实现 APB外设与TrustZone技术相兼容，APB-to-AXI桥将负责管理APB总线上设备的安全**。APB-to-AXI桥会拒绝不匹配的安全事务设置，并且不会将该事务请求发送给外设。

#### 实现外围设备隔离

其他外围设备都会挂载到APB总线上，然后通 过AXI-to-APB桥连接到AXI总线上，AXI-to-APB结合TZPC组件的TZPCDECROT的值及访问请求的PROT信号来判定该访问是否有效。当处理器需要 访问外围设备时，会将地址和PROT信号发送到 AXI总线上。 

AXI-to-APB桥会对接收到的请求进行解析，获取需要访问的所需外围设备，然后通过查询 TZPCDECROT的值来判断外设的安全类型，再根据PROT信号就能判定该请求的安全类型。如果该请求是非安全请求，但需要访问的外围设备属于安全设备，则AXI-to-APB会判定该访问无效。 

通过对TZPC中的TZPCDECROT寄存器进行编程能够设置外设的安全类型，从而做到外设在硬件层面的隔离。

## 2.3 TZ地址空间控制组件

TrustZone地址空间控制组件（TrustZone Address Space Controller，TZASC）[1]是AXI总线上的一个主设备，**TZASC能够将从设备全部的地址空间分割成一系列的不同地址范围**。在安全状态下， 通过编程TZASC能够将这一系列分割后的地址区域设定成安全空间或者是非安全空间。被配置成安全属性的区域将会拒绝非安全的访问请求。 

使用TZASC主要是将一个AXI从设备分割成几个安全设备，例如off-Soc、DRAM等。ARM的动态内存控制器（Dynamic Memory Controller，DMC） 并不支持安全和非安全分区的功能。如果将DMC接到TZASC上，就能实现DRAM支持安全区域和非安全区域访问的功能。需要注意的是，**TZASC组件只支持存储映射设备对安全和非安全区域的划分与扩展**，但不支持对块设备（如EMMC、NAND flash 等）的安全和非安全区域的划分与扩展。为使用TZASC组件的例子[^3]：

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221001124901.png" width="100%" />
</div>

The TZASC example system contains:
* an AXI to APB bridge restricting APB traffic to the TZASC to secure only
* a TZASC
* AXI bus masters:
	* an ARM processor
	* a Digital Signal Processor (DSP).
* AXI infrastructure component
* a Dynamic Memory Controller (DMC).

主控告知访问权限，内存系统决定是否允许访问。内存系统权限检查一般是通过互联总线。比如NIC-400可以设置如下属性，对于DDR，需要将整个区域划分为若干安全和非安全区域。通过TZASC可实现：

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221001132447.png" width="77.7%" />
</div>

#### 实现片外DRAM的隔离

一个完整的系统必然会有片外RAM，对片外 RAM的隔离是通过TZASC组件实现的，ARM本身的DMC可以将DRAM分割成不同的区域，这些区域是没有安全和非安全分类。将DMC与TZASC相连 后再挂到总线上，通过对TZASC组件进行编程可以将DRAM划分成安全区域和非安全区域。当主设备 访问DRAM时，除需要提供物理地址之外，还会发送PROT信号。TZASC组件首先会判定主设备需要访问的DARM地址是属于安全区域还是非安全区域，然后再结合接收到的PROT信号来判定该次访问是否有效。如果PROT信号为非安全访问操作， 且访问的DRAM地址属于安全区域，则TZASC就不会响应这次访问操作，这样就能实现DRAM中安全区域和非安全区域的隔离。

## 2.4 TZ内存适配组件
TrustZone内存适配器组件（TrustZone Memory Adapter，TZMA）**允许对片上静态内存（on-SoC Static Memory）或者片上ROM进行安全区域和非安全区域的划分**。TZMA支持最大2MB空间的片上静态RAM的划分，可以将2MB空间划分成两个部 分，高地址部分为非安全区域，低地址部分为安全区域，两个区域必须按照4KB进行对齐。分区的具体大小通过TZMA的输入信号R0SIZE来控制，该信号来自TZPC的输出信号TZPCR0SIZE。即通过编程TZPC可以动态地配置片上静态RAM或者ROM的大小。。使用TZMA组件的链接框图如图所示[^4]：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221001125618.png)

仅有安全访问可以配置TZASC的寄存器。比如TZC-400可以支持9个区域的配置。

#### 实现片上RAM和片上ROM隔离

芯片内部存在小容量的RAM或者ROM，以供芯片上电时运行芯片ROM或者存放芯片自身相关的数据。TrustZone架构对该部分也进行了隔离操作。 隔离操作通过使用TZMA和TZPC组件来实现。 

TZMA用来将片上RAM或者ROM划分成安全 区域和非安全区域，安全区域的大小则由接入的 TZPCR0SIZE信号来决定。而TZPCR0SIZE的值可以通过编程TZPC组件中的TZPCR0SIZE寄存器来实现。

当处理器核访问片上RAM或者ROM时， TZMA会判定访问请求的PROT信号是安全操作还是非安全操作，如果处理器发出的请求为非安全请求而该请求又尝试去访问安全区域时，TZMA就会认为该请求为非法请求。这样就能实现片上RAM和 ROM的隔离，达到非安全态的处理器核无法访问片上安全区域的RAM和ROM。

## 2.5 TZ保护控制器组件
TrustZone保护控制器组件（TrustZone Protection Controller，TZPC）是用来设定 TZPCDECPORT信号和TZPCR0SIZE等相关控制信号的。**这些信号用来告知APB-to-AXI对应的外设是安全设备还是非安全设备，而TZPCR0SIZE信号用来控制TZMA对片上RAM或片上ROM安全区域大小的划分**。TZPC包含三组通用寄存器 TZPCDECPROT[2：0]，每组通用寄存器可以产生8 种TZPCDECPROT信号，也就是TZPC最多可以将 24个外设设定成安全外设。TZPC组件还包含一个TZPCROSIZE寄存器，该寄存器用来为TZMA提供分区大小信息。TZPC组件的接口示意如图所示:

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221001130346.png" width="70%" />
</div>

当上电初始化时，TZPC的TZPCDECROT寄存 器中的位会被清零，同时TZPCR0SIZE寄存器会被设置成0x200，表示接入到TZMA上的片上RAM或者ROM的安全区域大小为2MB。通过修改TZPC的寄存器配置的值可实现用户对资源的特定配置。TZPC的使用例子如图所示：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221001130652.png)


## 2.6 TZ中断控制器

在原来的ARM芯片中，使用VIC来对外部中断 源进行控制和管理，支持TrustZone后，ARM提出 了TZIC组件，在芯片设计时，该组件作为一级中断 源控制器，控制所有的外部中断源，通过编程TZIC 组件的相关寄存器来设定哪个中断源为安全中断源 FIQ，而未被设定的中断源将会被传递给VIC进行 处理。一般情况下VIC会将接收到的中断源设定成 普通中断请求（Interrupt Request，IRQ），如果在 VIC中将接收到的中断源设定成FIQ，则该中断源 会被反馈给TZIC组件，TZIC组件会将安全中断源 送到安全世界状态中进行处理。

在支持TrustZone的SoC上，ARM添加了 TrustZone中断控制器（TrustZone Interrupt Controller，TZIC）。**TZIC的作用是让处理器处于非安全态时无法捕获到安全中断**。TZIC是第一级中断控制器，所有的中断源都需要接到TZIC上。TZIC根据配置来判定产生的中断类型，然后决定是将该中断信号先发送到非安全的向量中断控制器 （Vector Interrupt Controller，VIC）后以nIRQ信号发送到处理器，还是以nTZICFIQ信号直接发送到 处理器。通过对TZIC的相关寄存器进行编程，可对TZIC进行配置并设定每个接入到TZIC的中断源的中断类型。TZIC具有众多寄存器，细节说明可以参考相关ARM的文档。在TZIC中用来设置中断源类 型的寄存器为TZICIntSelect，如果TZICIntSelect中的某一位被设置成1，则该相应的中断源请求会被设置成快速中断请求（Fast Interrupt Request， FIQ）。如果某一位被设置成0，则该中断源的中断 请求会被交给VIC进行处理。如果VIC的IntSelect将 获取到的中断源设置成FIQ，那么该中断源会被再次反馈给TZIC进行处理[^5]。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221001131236.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221001131244.png)

GIC支持TrustZone，每个中断源，即GIC规格中的INTID，可被配置成如下一种中断组：
-   Group 0：安全中断，即FIQ。
-   Secure Group 1：安全中断，IRQ或者FIQ。
-   Non-secure Group 1：非安全中断，IRQ或者FIQ。

可以通过`GIC[D|R]_IGROUP[D|R]和GIC[D|R]_IGRPMODR<n>`寄存器来配置中断类型。这些寄存器仅在安全状态下可配置。安全中断仅能被安全访问修改；非安全总线访问读取安全中断寄存器返回全0。EL3软件处理Group 0中断，Secure Group 1中断被S.EL1/2软件处理。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221001132748.png)

处理器包括两种中断异常：**IRQ和FIQ**。

GIC产生何种中断异常取决于：中断所属中断组和处理器当前安全状态。

| 中断组             | 安全状态 | 中断异常类型 | 处理者       |
| ------------------ | -------- | ------------ | ------------ |
| Group 0            | Secure   | FIQ          | EL3 Firmware |
| Non-secure         |          |              |              |
| Secure Group 1     | Secure   | IRQ          | TOS          |
| Non-secure         | FIQ      | EL3 Firmware |              |
| Non-secure Group 1 | Secure   | FIQ          | EL3 Firmware |
| Non-secure         | IRQ      | Linux        |              |

如下图Non-secure和Secure各有一个IRQ vector，EL3有一个FIQ vector；并且SCR_EL3.FIQ=1、SCR_EL3.IRQ=0。左边中断表示发生在非安全世界，右边表示发生在安全世界。

-   Group 0中断无论发生在Secure还是Non-secure，都产生FIQ，在EL3处理。
-   Secure Group 1发生在Secure状态中断为IRQ，由S.EL1处理。
-   Secure Group 1发生在Non-secure状态中断为FIQ，由EL3处理。
-   Non-secure Group 1发生在Secure状态中断为FIQ，由EL3处理。
-   Non-secure Group 1发生在Non-secure状态中断为IRQ，由NS.EL1处理。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221001132842.png)


## 2.7 Cache和MMU扩展

### 2.7.1 MMU扩展

在支持TrustZone的SoC上，**会对MMU进行虚拟化，使得寄存器TTBR0、TTBR1、TTBCR在安全状态和非安全状态下是相互隔离的**，因此两种状态下的虚拟地址转换表是独立的。 存放在**MMU中的每一条页表描述符都会包含一个安全状态位**，用以表示被映射的内存是属于安全内存还是非安全内存。

>#### 安全与非安全 
>[14_ARMv8_内存管理（二）-ARM的MMU设计](https://github.com/carloscn/blog/issues/54)：
>
>从安全和非安全的角度来看，arm定义了两个物理地址空间，安全地址空间和非安全地址空间。在理论上，两个地址空间应该是相互平行和独立，一个系统也应该被设计出隔离两个空间的属性。然而在大部分真实的系统中，指示把安全和非安全作为一个访问属性，而非两个真实的隔离的空间，所以安全和非安全的概念是逻辑上的，而不是物理上的。我们设定，normal world只能够访问被标记为非安全的空间，而secure world有着更高的权限，既可以访问安全的空间，又可以访问非安全空间。实现这个控制，是在页表上做的手脚。
>
>在安全世界和非安全世界也有cache一致性的问题。例如，由于安全地址0x8000和非安全地址0x8000是逻辑上不同的地址，但是他们可能会在cache中共存。有一个理想的手段就是禁止安全和非安全的互相访问，但实际上我们只要禁止非安全向安全的访问就可以。为了避免安全世界的冲突，必须使用非安全访问非安全的空间，在安全的页表中加上非安全的映射。
>![](https://raw.githubusercontent.com/carloscn/images/main/typoratyporaimage-20220501160512513.png)
>
>**在Page Descriptor中(页表entry中)**，有NS比特位（BIT[5]），表示当前的映射的内存属于安全内存还是非安全内存：
>
>![](https://raw.githubusercontent.com/carloscn/images/main/typora20221001135724.png)

### 2.7.2 Cache扩展

**Cache也同样进行了扩展，Cache中的每一项都会按照安全状态和非安全状态打上对应的标签**，在不同的状态下，处理器只能使用对应状态下的Cache。如下所示，以为cortex-A78为例，L1 Data Cache TAG中 ，有一个NS比特位（BIT[33]），表示当前缓存的cacheline是secure的还是non-secure的：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221001140303.png)

以Cortex-a72为例子，我们可以找到这个L1-D Tag，第30位来表示这个物理地址是安全或者是非安全。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221001141702.png)

### 2.7.3 TLB扩展

虚拟化的MMU共享转换监测缓冲区（Translation Lookaside Buffer， TLB），同样**TLB中的每一项也会打上安全状态位标记**，只不过该标记是用来表示该条转换是正常世界状态转化的还是安全世界状态转化的。

以为cortex-A78为例，L1 Data TLB entry中 ，有一个NS比特位（BIT[35]），表示当前缓存的entry是secure的还是non-secure的。Arm ® Cortex ®-A78 Core - Technical Reference Manual

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221001144407.png)

## 2.8 调试和跟踪

不同的用户有不同的调试需求，可以通过如下信号配置：
-   DBGEN：顶层调试总开关，控制所有状态调试。
-   SPIDEN：安全调试开关，控制安全状态调试。

 三种使用场景：
-   芯片开发人员：DBGEN=1和SPIDEN=1，完全外部调试功能。
-   OEM开发：DBGEN1但SPIDEN=0，仅可在非安全状态调试。
-   产品发布：DBGEN0和SPIDEN=0，禁止安全和非安全调试。应用程序仍可调试。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221001132935.png)

## 2.9 其他设备

一个实际TrustZone系统中还需要包括如下设备：
-   OTP或者EFuses
-   Non-volatile counter
-   Trusted RAM和Trusted ROM


# 3. ATF

## 3.1 为什么使用ATF

ARM可信任固件（ARM Trusted Firmware，ATF）是由ARM官方提供的底层固件，该固件统一了ARM底层接口标准，如电源状态控制接口（Power Status Control Interface，PSCI）、安全启 动需求（Trusted Board Boot Requirements， TBBR）、安全世界状态（SWS）与正常世界状态 （NWS）切换的安全监控模式调用（secure monitor call，smc）操作等。ATF旨在将ARM底层的操作统一使代码能够重用和便于移植。

## 3.2 ATF的主要功能
ATF的源代码共分为bl1、bl2、bl31、bl32、 bl33部分，其中bl1、bl2、bl31部分属于固定的固 件，bl32和bl33分别用于加载TEE OS和REE侧的镜像。整个加载过程可配置成安全启动的方式，每一个镜像文件在被加载之前都会验证镜像文件的电子签名是否合法。 

ATF主要完成的功能如下：

* 初始化安全世界状态运行环境、异常向量、 控制寄存器、中断控制器、配置平台的中断。 
* 初始化ARM通用中断控制器（General Interrupt Controller，GIC）2.0版本和3.0版本的驱动初始化。
* 执行ARM系统IP的标准初始化操作以及安全扩展组件的基本配置。 
* 安全监控模式调用（Secure Monitor Call， SMC）请求的逻辑处理代码（Monitor模式/EL3）。 
* 实现可信板级引导功能，对引导过程中加载的镜像文件进行电子签名检查。 
* 支持自有固件的引导，开发者可根据具体需求将自有固件添加到ATF的引导流程中。

Trusted Firmware提供了满足ARM安全规格的参考代码，包括TBBR(Trusted Board  Boot Requirements)和SMCC。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221001145243.png)

SMC Dispatcher：处理非安全世界的SMC请求，决定哪些SMC由Trusted Firmware在EL3处理，哪些转发给TEE进行处理。

Trusted Firmware处理PSCI任务、或者SOC相关工作。一个典型的基于TrustZone系统软件调用栈关系图：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221001145447.png)

* 安全世界的Trusted OS提供一系列安全服务，比如key管理或DRM。非安全世界应用需要使用这些服务，但是又无法直接使用。通过使用一些库中API来获取这些能力，比如libDRM.so。
* 这些库和Trusted service之间通信往往通过message queue或者Mailbox。他们之间通信所用内存往往被称为WSM(World Shared Memory)。这些内存必须能够被Trusted service和库访问，这就意味着这些内存是非安全内存。
* 应用通过库发送一个请求到Mailbox或message queue，然后出发内核中TrustZone驱动。
* TrustZone驱动负责和TEE部分交互，包括为message queue申请内存和注册这些内存。由于安全和非安全运行在两个虚拟地址空间，所以无法通过虚拟地址进行通信。
* TrustZone驱动通过调用SMC进入安全状态，控制权通过EL3的Secure Monitor传递到TEE中的TOS。TOS从message queue内存中获取内容个Trusted service进行处理。
	-   **Trusting the message**：由于message是从非安全世界传递的，所以需要安全世界需要对这些内容进行一些认证。
	-   **Scheduling**：对于PSCI类型快速处理并且不频繁请求，进入EL3处理完后退出到非安全状态。对于一些需要TOS处理的任务，不能被非安全中断打断，避免造成安全服务不可用。
	-   **OP-TEE**：OP-TEE内核运行在S.EL1，可信应用运行在S.EL0。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221001145557.png)

非安全世界的App一般不直接使用TEE Client API，而是中间Service API提供一类服务。TEE Client API和内核中OP-TEE驱动交互，OP-TEE驱动SMC处理底层和OP-TEE内核的通信。

EL3 Firmware/Secure Monitor做SMC处理，根据需要将安全请求发送到OP-TEE内核处理。

## 3.3 ATF与TEE的关系

为规范和简化TrustZone OS的集成，在ARMv8 架构中，ARM引入ATF作为底层固件并开放了源 码，用于完成系统中BootLoader、Linux内核、TEE OS的加载和启动以及正常世界状态和安全世界状态的切换。ATF将整个启动过程划分成不同的启动阶段，由BLx来表示。例如，TEE OS的加载是由ATF 中的bl32来完成的，安全世界状态和正常世界状态之间的切换是由bl31来完成的。在加载完TEE OS之后，TEE OS需要返回一个处理函数的接口结构体变量给bl31。当在REE侧触发安全监控模式调用指令时，bl31通过查询该结构体变量就可知需要将安全监控模式调用指令请求发送给TEE中的那个接口并完成正常世界状态到安全世界状态的切换。


# REF
[^1]:[# ARM Cortex-A Series Programmer's Guide for ARMv7-A - ## TrustZone hardware architecture](https://developer.arm.com/documentation/den0013/d/Security/TrustZone-hardware-architecture)
[^2]:[10_ARMv8_异常处理（一） - 入口与返回、栈选择、异常向量表](https://github.com/carloscn/blog/issues/47)
[^3]:[# CoreLink TrustZone Address Space Controller TZC-380 Technical Reference Manual r0p1](https://developer.arm.com/documentation/ddi0431/c?lang=en)
[^4]:[## PrimeCell Infrastructure AMBA 3 AXI TrustZone Memory Adapter (BP141) Revision: r0p0 Technical Overview](https://developer.arm.com/documentation/dto0017/a/?lan)
[^5]:[# AMBA 3 TrustZone Interrupt Controller (SP890) Technical Overview](https://developer.arm.com/documentation/dto0013/b/?lang=en)
[^6]:[How to run OP-TEE on QEMU ArmV8/V7](https://blooggspott.blogspot.com/2022/04/op-tee-3.html)
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


