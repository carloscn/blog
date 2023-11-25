# 01_OPTEE-OS_基础之（一）功能综述、简要介绍

我们利用十一假期的时间，把TEE这部分的知识进行整理和归纳，也算是对于ARM64架构进行复习。

在没有集成trustzone的环境有一个风险就是当获取root权限之后，就可以随心所欲访问所有的数据，这样的操作就十分的危险。为了保障这一部分数据在root权限下不被轻松截获，因此在硬件层级引入了trustzone技术提供了（Trusted Execution Environment - TEE）。

# 1. TEE如何保障数据安全
ARM从ARMv6的架构开始引入了TrustZone技术。 TrustZone技术将CPU的工作状态分为了**正常世界状态 （Normal World Status，NWS）和安全世界状态 （Secure World Status，SWS）**。支持TrustZone技术的芯片提供了对外围硬件资源的硬件级别的保护和安全隔离。当CPU处于正常世界状态时，任何应用都无法访问安全硬件设备，也无法访问属于安全世界状态下的内存、缓存（Cache）以及其他外围安全硬件设备。

TEE基于TrustZone技术提供可信运行环境，还为开发人员提供了API，以方便他们开发实际应用程序。

在整个系统的软件层面，一般的操作系统（如Linux、Android、Windows等）以及应用运行在正常世界状态中，TEE运行在安全世界状态中，正常世界状态内的开发资源相对于安全世界状态较为丰富，因此通常称运行在正常世界状态中的环境为丰富执行环境（Rich Execution Environment，REE），而可信任的操作系统以及上层的可信应用（Trusted Application，TA）运行于安全世界状态，运行在安全世界状态中的系统就是前文提到的TEE。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221001094600.png)

对CPU的工作状态区分之后，处于正常世界状态中的Linux即使被root也无法访问安全世界状态中的任何资源，包括操作安全设备、访问安全内存数据、获取缓存数据等。

这是因为CPU在访问安全设备或者安全内存地址空间时，芯片级别的安全扩展组件会去**校验CPU发送的访问请求的安全状态读写信号位**（Non-secure bit，NS bit）是0还是1，以此来判定当前CPU发送的资源访问请求是安全请求还是非安全请求。而处于非安全状态的CPU将访问指令发送 到系统总线上时，其访问请求的安全状态读写信号位都会被强制设置成1，表示当前CPU的访问请求为非安全请求。而非安全请求试图去访问安全资源 时会被安全扩展组件认为是非法访问的，于是就禁止其访问安全资源，因此该CPU访问请求的返回结果要么是访问失败，要么就是返回无效结果，这也就实现了对系统资源硬件级别的安全隔离和保护。

在真实环境中，可以将用户的敏感数据保存到TEE中，并由可信应用（Trusted Application，TA）使用重要算法和处理逻辑来完成对数据的处理。当需要使用用户的敏感数据做身份验证时，则通过在REE侧定义具体的请求编号（IDentity，ID）从TEE 侧获取验证结果。验证的整个过程中用户的敏感数据始终处于TEE中，REE侧无法查看到任何TEE中的数据。对于REE而言，TEE中的TA相当于一个黑盒，只会接受有限且提前定义好的合法调用（TEEC），而至于这些合法调用到底是什么作用，会使用哪些数据，做哪些操作在REE侧是无法知晓的。如果在 REE侧发送的调用请求是非法请求，TEE内的TA是不会有任何的响应或是仅返回错误代码，并不会暴露任何数据给REE侧。

# 2. TEE解决方案

TEE是一套完整的安全解决方案，主要包含：
* 正常世界状态的客户端应用（Client Application， CA）
* 安全世界状态的可信应用，可信硬件驱动 （Secure Driver，SD）
* 可信内核系统（Trusted Execution Environment Operation System，TEE OS），
 
其系统配置、内部逻辑、安全设备和安全资源的划分是与CPU的集成电路（Integrated Circuit， IC）设计紧密挂钩的，**使用ARM架构设计的不同 CPU，TEE的配置完全不一样**。国内外针对不同领域的CPU也具有不同的TEE解决方案。

国内外各种TEE解决方案一般都**遵循GP（Global Platform）规范**进行开发并实现相同的API。GP规范规定了TEE解决方案的架构以及供TA开发使用的API原型，开发者可以使用这些规定的 API开发实际的TA并能使其正常运行于不同的TEE解决方案中。

# 3. 手机领域的TEE

各家TEE解决方案的内部操作系统的逻辑会不一样，但都能提供GP规范规定的API，对于二级厂 商或TA开发人员来说接口都是统一的。这些TEE解决方案在智能手机领域主要用于实现在线支付（如 微信支付、支付宝支付）、数字版权保护（DRM、 Winevine Level 1、China DRM）、用户数据安全保护、安全存储、指纹识别、虹膜识别、人脸识别等其他安全需求。这样可以降低用户手机在被非法 root之后带来的威胁。 Google规定在Android M之后所有的Android设备在使用指纹数据时都需要用TEE来进行保护，否则无法通过Google的CTS认证授权，另外Android也 建议使用硬件Keymaster和gatekeeper来强化系统安全性。

# 4. 物联网领域的TEE

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221001100701.png)

物联网（Internet of Thing，IoT）领域和车载系统领域将会是未来TEE方案使用的另外一个重要 方向，大疆无人机已经使用TEE方案来保护无人机用户的私人数据、航拍数据以及关键的飞控算法。 ARM的M系列也开始支持TrustZone技术，如何针对资源受限的IoT设备实现TEE也是未来TEE的重要发展方向之一。 而在车载领域NXP芯片已经集成OP-TEE作为TEE方案，MediaTek的车载芯片也已集成了 Trustonic的TEE方案，相信在车载系统领域TEE也将渐渐普及。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221001100645.png)

参考[^1][^2]
# 5. OP-TEE

OP-TEE是由非营利的开源软件工程公司Linaro开发的，从 git上可以获取OP-TEE的所有源代码，且OP-TEE支 持的芯片也越来越多，相信未来OP-TEE将有可能是TEE领域的Linux，并得到更加广泛的运用。 OP-TEE是按照GP规范开发的，支持QEMU、 Hikey（Linaro推广的96Board系列平台之一，使用 Hisilicon处理器）以及其他通用的ARMv7/ARMv8 平台，开发环境搭建方便，便于开发者开发自有的 上层可信应用，且OP-TEE提供了完整的软件开发 工具包（Software Development Kit，SDK），方便编译TA和CA。OP-TEE遵循GP规范，支持各种加解密和电子签名验签算法以便实现DRM、在线支付、指纹和虹膜识别功能。OP-TEE也支持在芯片中集成第三方的硬件加解密算法。除此之外，在 IoT和车载芯片领域也大都使用OP-TEE作为TEE解决方案。OP-TEE由Linaro组织负责维护，安全漏洞补丁更新和代码迭代速度较快，系统的健壮性也越来越好，所以利用OP-TEE来研究TrustZone技术的实现并开发TA和CA将会是一个很好的选择。

# Ref
[^1]:[Overview of the OP-TEE open source projec](https://wiki.stmicroelectronics.cn/stm32mpu/wiki/OP-TEE_overview)
[^2]:[Demystifying ARM TrustZone for Microcontrollers (and a Note on Rust Support)](https://medium.com/swlh/demystifying-arm-trustzone-for-microcontrollers-and-a-note-on-rust-support-54efc62c290)