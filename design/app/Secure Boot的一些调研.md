# 1. Background

Secure Boot需要信任根公钥（Root of Trust Public Key，ROTPK）的原因在于它为整个安全启动过程提供了一个基本的、可信赖的起点。这个起点是确保系统安全性的关键部分，用以确保用以验证启动代码的公钥的完整性和真实性，是建立信任链的基础。本文主要讨论，**如何保证信任链的“根”的安全性**。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202311221425797.png)

在系统芯片（SoC）和其他嵌入式系统中常用的“信任链”（Chain of Trust）安全启动流程。以下是关键方面的概述：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202311221426443.png)


**信任链概念**

- 启动过程中的安全验证：信任链确保系统上加载的每个组件未被篡改。这个过程从重置开始，硬件组件首先验证第一阶段的软件。随后，已验证的软件加载额外的软件。
- SoC硬件上电启动：当SoC硬件上电时，CPU会自动从起始地址（如重置向量）开始执行。起始地址和启动选择器被配置到内部闪存地址空间的预定义位置或掩膜ROM中。在工厂配置后，这个起始地址处的代码被认为是不可变的。这些特性为不可变的引导加载程序不能轻易被绕过提供了强有力的保证。
- 不可变引导加载程序：不可变的引导加载程序使用公钥来检查第一阶段的代码是否为真实的，然后再执行它：
    - 在单阶段启动过程中，不可变引导加载程序可能验证单个映像或同时验证多个映像。
    - 在多阶段启动过程中，可能有多个加载程序，每个加载程序使用密钥来验证和加载其他映像。
- 基于产品需求的实现决策：实现必须根据产品需求确定需要多少启动阶段。无论阶段数量如何，每个启动阶段都必须验证其加载的所有软件。

# 2. Supply chain scenarios

在ARM公司的[Platform Security Boot Guide](https://developer.arm.com/documentation/PRD29-GENC-009492/c/TrustZone-Software-Architecture/Booting-a-secure-system)中，对3章 Security Requirements中对信任链的提供方式做了要求。在此规范要求至少有一个被称为**信任根公钥（Root of Trust Public Key，ROTPK）的公钥**，**该公钥负责使用公钥加密技术安全地验证第一阶段的代码**。其原则是：

- 它也可能用于验证授权签名密钥的证书。
- ROTPK的哈希值或ROTPK本身必须位于SoC非易失性存储器（NVM）的不可变部分或安全子系统中。
- 通常更倾向于使用哈希值，因为它在不可变存储中占用的空间较少。
- 供应链中不同供应商可能包含多个ROTPK。

例如，SoC供应商可能能够使用**内置的ROTPK安全地启动他们自己的代码**，然后使用单独的OEM ROTPK来验证OEM固件。OEM ROTPK可能在供应链的不同点进行配置，那里可能有运营流程来减少配置过程中的风险。

当评估使用公钥哈希与使用证书哈希在安全启动（Secure Boot）过程中作为信任根的安全性时，重要的是要理解这两种方法的不同特点和它们在实际应用中的含义。

### 使用公钥哈希：

- 简单性：直接使用公钥哈希相对简单，因为它直接指向一个特定的密钥。这减少了处理和验证所需的计算复杂度。
- 直接性：由于直接引用了特定的公钥，因此不存在解析或验证证书链的需要。
- 灵活性限制：使用公钥哈希的主要缺点是灵活性较低。如果需要更换密钥，可能需要更新固件或软件，这在一些环境中可能是不切实际的。

### 使用证书哈希：

- **额外的安全层**：证书提供了一个额外的安全层，包括了关于密钥所有者和发行者的信息。**这有助于提供更丰富的安全上下文**。
- **更灵活的密钥管理**：通过使用证书，更换密钥变得更加容易，因为可以通过更新证书而不需要改变在设备上存储的哈希值。
- **增加的复杂性**：验证证书链比单纯验证公钥哈希更复杂，增加了计算和实现的复杂度。
    
### 使用CMAC/HMAC：

- **密钥用途的限制**：通常，在使用MAC算法时，同一个密钥既用于生成消息认证码（MAC），又用于验证这个MAC。这种做法有一个安全上的缺陷，即它不提供非抵赖性——**发送方无法证明它确实发送了某条消息，因为任何拥有密钥的人都可以生成和验证MAC。**
- ****配置密钥属性**：为了解决这个问题，可以特别配置密钥，使其只用于MAC的生成或仅用于验证。这意味着一个密钥只能用于创建MAC，而另一个不同的密钥用于验证这个MAC。**
- ****硬件级别的执行**：这种密钥属性的配置由硬件强制执行。也就是说，硬件设备（如安全模块或特定的加密处理器）在设计上会确保密钥只能用于其被配置的目的（生成或验证），这样就可以防止密钥被滥用。**
- ****用于资源紧缺或者对启动时间敏感的芯片MAC**。**

## 2.1 Single provider

下图展示了最简单的案例，比如一个受限制的或嵌入式设备，其中制造商控制着整个软件栈和签名过程。制造商提供了信任根公钥（ROTPK）。在这种情况下，不可能进行撤销。

**Note，这种情况仅对超受限的微控制器（MCU）或简单的外围设备有益。参考：[https://documentation-service.arm.com/static/5fae7507ca04df4095c1caaa?token=](https://documentation-service.arm.com/static/5fae7507ca04df4095c1caaa?token=) 3.4.1 Single Provider**

![](https://raw.githubusercontent.com/carloscn/images/main/typora202311221427576.png)

如果系统拥有多个软件提供商，并且具有足够的计算能力去验证证书链，那么建议创建单独的签名密钥，以减轻私有镜像签名密钥丢失的风险。

## 2.2 Multiple dependent providers

信任链的第一个环节是**信任根公钥的管理**，它用于验证所有后续的**证书和镜像**。一个示例场景如下：
- 原始设备制造商（OEM）提供他们自己的信任根公钥（ROTPK）。
- 原始设备制造商（OEM）对属于软件供应商凭据的证书进行签名。可能存在多个软件供应商。
- 软件供应商使用他们经过认证的凭证来签名：
    - 生产镜像签名凭证（针对特定型号系列的唯一凭证）
    - 调试镜像签名凭证（针对特定型号系列的唯一凭证）
- SoC（系统级芯片）在出厂时，已经在一次性编程（OTP）内存中配置了原始设备制造商（OEM）的信任根公钥（ROTPK）。

如果一个镜像签名者需要更换他们的镜像签名密钥，那么他们必须联系信任根公钥（ROTPK）的所有者（下图中的根证书颁发机构，Root CA）。这需要镜像签名者和根证书颁发机构（即ROTPK的所有者）之间保持持续的商业关系。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202311221428401.png)

## 2.3 Multiple Independent Providers

该系统可能包含足够的片上存储空间，以容纳多个独立的信任根公钥（ROTPK）。每个ROTPK提供一个独立的信任链，允许不同的制造商独立于彼此授权和撤销固件签名密钥。例如，一个ROTPK可以对应于SoC供应商，而另一个可能对应于原始设备制造商（OEM）。或者，OEM拥有一个ROTPK，同时允许客户安装一个单独的ROTPK来验证非安全处理环境（NSPE）固件。下图展示了一个简化的独立提供商的信任链。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202311221428567.png)


## 2.4 CMAC/HMAC作为认证的方式

在安全引导中，通常有两种不同的方法用于检查真实性和完整性，根据AUTOSAR的标准[https://www.autosar.org/fileadmin/standards/R22-11/FO/AUTOSAR_TR_SecureHardwareExtensions.pdf](https://www.autosar.org/fileadmin/standards/R22-11/FO/AUTOSAR_TR_SecureHardwareExtensions.pdf) ：

- 使用CMAC/HMAC作为真实性和完整性检查（低安全性，高性能，节约空间）
    
- 使用RSA/**ECDSA**非对称密钥验签作为真实性和完整性检查（高安全性，运算时间长，占用空间大）

使用CMAC或HMAC算法可以导致快速启动时间。该解决方案的最大挑战是加密密钥和参考MAC的存储。用于对称算法的私钥需要安全地存储在受保护的安全环境（如HSM）中。此外，由于MAC生成和MAC验证使用相同的密钥，因此默认情况下不提供不可否认性。可以总结为：

- **密钥和参考MAC的存储**：对称算法的一个主要挑战是安全地存储加密密钥和参考MAC值。
- **需要安全环境**：用于对称算法的私钥需要在受保护的安全环境中存储，如硬件安全模块（HSM）。
- **非抵赖性缺失**：由于使用相同的密钥进行MAC的生成和验证，这种方法默认不提供非抵赖性（即无法证明发送者发送了消息）。
- **解决方案**：为了解决这个缺陷，可以配置密钥只用于MAC的生成或验证，这一属性由硬件强制执行。

使用CMAC/HMAC的这种方法，**显然是降低安全性追求性能和资源的节约**。对于那些对于boot时间严苛的ECU或者资源受限的ECU中，会使用该方法。例如：[s32k144](https://www.nxp.com/products/processors-and-microcontrollers/s32-automotive-platform/s32k-auto-general-purpose-mcus/s32k1-microcontrollers-for-automotive-general-purpose:S32K1)，[RH850](https://www.renesas.com/us/en/products/microcontrollers-microprocessors/rh850-automotive-mcus) 。

## 2.2 Multiple dependent providers

在一些性能强大的SoC中，采用multiple dependent providers的形式，即采用证书的HASH写入eFUSE作为信任根。例如NXP的rt1170就是该设计。

- NXP-rt1170参考：[https://wiki.autox.clu/doc/x-nav-imx-rt1170s-secure-boot-UqmdC6HKXt](https://wiki.autox.clu/doc/x-nav-imx-rt1170s-secure-boot-UqmdC6HKXt)
    
**从NXP的产品线可以看到**，NXP将统一实现，High Assurance Boot标准应用于NXP的SoC及MCU，NXP的很多产品线都开始逐步的采用，参考 [https://variwiki.com/index.php?title=High_Assurance_Boot](https://variwiki.com/index.php?title=High_Assurance_Boot) 这些产品是：

- 低功耗 i.MX6
- 高性能 SoC i.MX 8
- 跨界 RT117x
- Layerscape，部分工具采用HAB中的工具，例如cst签名工具，参考[https://docs.nxp.com/bundle/GUID-3FFCCD77-5220-414D-8664-09E6FB1B02C6/page/GUID-0D3D0BD8-45E2-4D2D-BD79-E5591C34225D.html](https://docs.nxp.com/bundle/GUID-3FFCCD77-5220-414D-8664-09E6FB1B02C6/page/GUID-0D3D0BD8-45E2-4D2D-BD79-E5591C34225D.html)。

**德州仪器的最新的Sitara Processor**中也采用了证书的方式，参考：[https://www.ti.com/lit/pdf/spry305](https://www.ti.com/lit/pdf/spry305)

![](https://raw.githubusercontent.com/carloscn/images/main/typora202311221429571.png)

**NVIDIA的Jetson系列的Processor也采用了证书的方式（也支持single provider，只使用公钥的形式）**，[https://docs.nvidia.com/jetson/archives/r35.4.1/DeveloperGuide/text/SD/Security/SecureBoot.html](https://docs.nvidia.com/jetson/archives/r35.4.1/DeveloperGuide/text/SD/Security/SecureBoot.html)

## 2.3 Multiple Independent Providers

这个暂时没有找到这样的设计。只存在于ARM的secure boot guideline里。

## 2.4 小结

总结：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202311221429383.png)

从总结主流的SoC和MCU，在secure boot中，供应商更喜欢使用Multiple dependent providers的形式，主要考虑：

- 在于证书提供了额外的安全性、灵活性和可管理性。
- 证书可以用来建立复杂的信任链结构，允许不同级别和角色的实体之间进行有效的权限控制和验证。
- 在许多行业和应用中，使用证书是遵循安全标准和法规的要求。它们符合公共密钥基础设施（PKI）的标准，这对于跨设备和系统的兼容性很重要。
- 与单纯的公钥哈希相比，证书可以包含更多关于密钥所有者的信息，使安全验证过程更为详尽和全面。
- 而证书的实现的复杂性随着SoC的性能和存储的提升，也让这个弊端逐步削弱。

**一个标准的secure boot应该是：**

> 1. Secure boot is integral for creating a secure system of chain of trust.
> 2. It provides:
>     - @1 - Authentication (unauthorized images not allowed to run) 
>     - @2 - Integrity (‘tampered’ images shall be detected)
> 3. It typically uses:
>     - Digital signatures
>         - Ensures authentication and integrity
>         - Private key -> used for signing 
>         - Public key -> used to verify 
>     - (Optionally) Image/data encryption
>         - Used for confidentiality 
>         - Used for anti-cloning/counterfeit 
> 4. When validation fails, sanctions are applied
> 5. Secure boot needs to coexist with the software update strategy