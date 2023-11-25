# [Embedded] ZYNQ-Secure Storage

如果使用Zynq® UltraScale+™设备，大部分数据是存储在设备的外面的(NVM)，如果想增强数据的保密性，则需要启动Secure Storage来保护数据[^1]。保护数据的算法使用ZYNQ内置的GCM-AES硬件引擎，那么同样，对于GCM-AES来说确保Key的可信是最重要和最根本的事情。eFUSE/OTP保存着Secure Boot和Secure Storage使用的加密的Key，而且内置的PUF（physically unclonable function）对Key的保密性进行了增强。Key的写入eFUSE/OTP和配置PUF的**过程叫做Provisioning[^2]**，也只有在Provisioning之后，Secure Storage和Secure Boot才能使用RSA和GCM-AES保密与认证功能。如何进行Provisioning可以参考[^2]

**本文参考**：
* (AN) Secure Storage Application Note[^1]
* (Provisioning) - Programming BBRAM and eFUSEs Application Note [^2]
* (TRM) Zynq UltraScale+ Device Technical Reference Manual (UG1085) - Security [^3]

# 1. Secure Storage High-Level Design

## 1.1 一般性的安全存储业务模式（非ZYNQ）

本节借助文献[^4]的根文件系统保护和应用程序数据保护来阐述以下对于Secure Storage的业务需求。
* 全盘级别加密：
	* 加密： [dm-crypt](https://gitlab.com/cryptsetup/cryptsetup/wikis/DMCrypt)
	* 认证： [dm-verity](https://source.android.com/docs/security/features/verifiedboot/dm-verity)
	* 完整性： [dm-integrity](https://kernel.googlesource.com/pub/scm/linux/kernel/git/kasatkin/linux-digsig/+/dm-integrity/Documentation/device-mapper/dm-integrity.txt)
* 目录级别：
	* 加密：[ext4](https://wiki.archlinux.org/index.php/ext4#Using_file-based_encryption) [ubifs](https://lwn.net/Articles/707900/) [eCryptfs](https://wiki.archlinux.org/title/ECryptfs)
	* 认证：[IMA/EVM](https://sourceforge.net/p/linux-ima/wiki/Home/)

在台式机或手机上，用于加密文件系统的密钥来自交互式输入的用户密码。物联网和嵌入式设备通常没有这样丰富的操作流程。因此，需要在设备上存储和保护密钥。以下是一些保护机制：
-   **使用 SoC 机制加密密钥，ARM的使用方法**
-   将密钥存储在提供安全存储的外部加密或安全芯片（例如：ATECC508A）中
-   使用外部可信平台模块 (TPM) 芯片
-   远程证明

我们只来阐述第一种使用 SoC 机制加密密钥，ARM的使用方法。很多外部安全芯片如TPM容易受到I2C[总线攻击](https://github.com/nccgroup/TPMGenie)。如果可能的话最好利用主处理器的安全存储功能。**安全存储的最核心的业务逻辑就是：如何保护好密钥**。

我们来看一下加密过程：
* 使用加密的方法保护文件系统；
* 需要加密上述加密的密钥；

在NXP I.MX上面，每个设备都有一个不同的主密钥（通过PUF，PUF是根据每个设备的物理工艺不同，可以实现每个设备的黑密钥差异的硬件设备），该密钥被SoC内部的加解密引擎访问。因此，**就可以编写安全应用程序，例如Linux内核安全驱动程序或者HSM系统，读取该唯一密钥，在HSM程序内部对密钥进行加密**，输出一个key blob用于解密方解密。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202211081517172.png" width="80%" /></div> 

如上图所示：
* 先对文件系统进行加密，使用的密钥就是文件加密的密钥key0；
* 再对key0进行加密（调用HSM或者Linux内核安全程序或者自己编写安全服务程序，需要保证解密方也能被调用）
* 解密过程就是先call安全系统HSM或者Linux内核安全程序或者安全应用解密密钥，拿到密钥之后解密文件系统。

这个就是一般的文件系统的加密模型。

##  1.2 ZYNQ的Secure Storage设计

在上述的加密模型中，需要引入HSM或者Linux内核安全服务程序用来对文件系统加密的密钥进行加解密。在ZYNQ中可以使用PUF硬件来满足加密的需求。PUF硬件根据物理工艺不同，可以对写入eFUSE的密钥进行加扰和解扰（也算是一种加密手段），因此使实际存储在eFUSE上的每个设备的black key都是不一样的，这就保证了唯一性。

我们可以对一批设备假定1万台，管理员生成一个红密钥作为文件系统加密的密钥，管理员需要为每台设备Provisioning这个红密钥到eFUSE上，并启动PUF功能。这一万台设备的实际存储的黑密钥都是不一样的。这就能够抵御侧信道攻击，即便是有人用功率分析，分析出eFUSE的密钥，拿到的也是黑密钥，没有任何的实际用处。

PUF可以理解为一个函数，这个函数输入红密钥、IV、ID等信息，输出一个AES-GCM加密的key blob结果，包含了tag、密文等。当我们要使用密钥解密文件系统的时候，则需要先把key blob输入进去，此时，ZYNQ读取到key blob则对输入的tag和密文解密，解密之后就可以拿到真实的红密钥，也会对tag进行验证，tag验证通过，红密钥才可信，解密者拿红密钥对文件系统进行解密。**ZYNQ只是对密钥加密和解密做了封装，而文件系统的加密解密需要自己的应用程序来做**。

### 1.2.1 加密解密High-Level角度过程

#### Alice的加密

我们来看看Alice加密过程，Alice产生一个Red Key，用于真实文件系统加密，Alice把Red Key注入FPGA中，得到了密文的Key Blob。与此同时，Alice使用Red Key对文件加密，得到加密了的文件系统。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202211081733463.png" width="90%" /></div> 

#### Bob的解密

Bob拿到Key Blob之后喂给FPGA，FPGA会输出Red Key和一个GCM的AUTH结果，如果AUTH结果通过，那么Bob可以使用红密钥对加密的文件进行解密，最后拿到解密后的文件。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202211081739905.png" width="90%" /></div> 

FPGA在这里充当的角色是对Red Key进行加解密验证。

### 1.2.2 FPGA SoC内部过程

如图所示，为Alice加密和Bob解密在External Memory以及FPGA SoC内部的加解密全过程：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202211081833341.png)

在FPGA SoC内部，主要是利用PUF的物理特性，产生一个PUF Key，用做内部数据加密的密钥。FPGA内部依旧使用AES-GCM-256来完成加密操作。Alice和Bob可以在External Memory协商数据加密的方式。

在FPGA内部，生成的黑色密钥是被写入到eFUSE中的，正因为每一个PUF的物理工艺不同导致的即便是红密钥一致，也会让每个设备的eFUSE内部的黑密钥不一致。这就达到了防止DPA的目的。

#### 加密过程

如图，注，**这里的New Data指的就是Alice的Red Key**。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202211081744591.png" width="90%" /></div> 

1. Alice把明文Red Key喂给FPGA；
2. FPGA产生一个PUF使用的Key（PUF物理工艺差异产生的数，不同芯片不一致，同一个芯片就是一致的），用于把Red Key加密成为一个Black Key，并写入eFUSE;
3. FPGA输出Black Key和TAG值，到外部内存；
4. FPGA读回Black Key和TAG值，组成Key Blob；
5. FPGA对Black Key进行解密，解密得到一个TAG1的值；
6. FPGA对比TAG1和TAG是否一致，以此检验是否加密成功；若不一致，进入“惩罚”流程，比如烧写FUSE的User Data，告知解密失败过。

#### 1.2.3 解密过程

解密的过程：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202211081757910.png" width="90%" /></div> 

1. Bob把Key Blob输入到FPGA内部；
2. FPGA读取到Key Blob分离出black key和TAG；
3. FPGA重新让PUF产生一个Key（PUF物理工艺差异产生的数，同一个设备，这个值必然一致）；
4. FPGA使用PUF产生的Key解密black key成为red key，并输出TAG1；
5. FPGA校验TAG1和Key Blob输入的TAG是否一致，若一致则解密成功并输出Red Key。

### 1.2.2 FPGA的安全角度考量

在ZYNQ内部，有很多安全关键因素在设计中被考虑到，我们应该熟悉其背后的设计逻辑，明白他们的设计意图。一种是对于防重放攻击的抵御，一种是防DPA，除此之外还有对black KEY 考量[FIPS legal KEK](https://csrc.nist.gov/csrc/media/projects/cryptographic-module-validation-program/documents/fips140-2/FIPS1402IG.pdf)的因素。

#### 防重放攻击

防重放攻击的设计体现在，Alice对于Red Key的输入不光是有 Red Key和IV这些成分，除此之外还有ID放入到OTP的256bits长度的User Data里面。Alice产生Key之后，可以增加ID，在解密的时候，除了要验证解密的信息的GCM的TAG之外，还要对比解析出来的ID信息，和OTP/eFUSE上的ID进行对比。这样就可以有效的抵御重放攻击。

id在bootgen的时候通过以下方法指定：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202211110955470.png" width="100%" /></div> 

#### 功率分析攻击（DPA）

PUF Key是直接加载到SoC内部的加密引擎中的。为了防止PUF Key被分析出来，ZYNQ给了两个建议：
* 尽量让User Data短。PUF Key在SoC中不是对所有的数据加密，而是加密第一部分，因此存在没有加密的部分。如果User Data过长，就难保后面的数据的保密性。入侵者Mallory很可能使用DPA功率分析分析出后面User Data的数据；
* 如果User Data不得不做的很长。ZYNQ推荐使用Rolling Key/OP Key的方式，进行加密；

##### OP Key

实际上在boot中也不希望使用私密的key频繁加密，以下是提供了OP Key方法来帮忙减少使用私密的key：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202211091413838.png" width="90%" /></div> 

bootgen在生成image的时候，会把op key放到image header里面，还包含了IV这类的信息。当需要进行解密的时候，使用私密的key对第一个块进行解密，从解密的信息中读取OP key，然后利用该key解下一个块，以此类推。这样私密的Key只使用了一次。

Note， **DPA功率分析，可以通过频谱分析出eFUSE上的数据**。

##### Keys Rolling

以下为Key Rolling的过程：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202211091425649.png)

#### FIPS Legay的key审查

使用SoC PUF产生的加密数据，也就是Block Key，如果作为Key的话，处于密码学安全边界之外，换句话说，此密钥不符合FIPS-legal KEK标准。因此我们需要在产生Red Key阶段就产生一个符合FIPS标准的Key，这样加密出来的Key就会处于密码学安全边界内。

# 2. Secure Storage Low-Level Design

## 2.1 eFUSE/OTP Provisioning

eFUSE array包含一个256的块，在这个块里面提供GCM-AES-256的key给加解密引擎。eFUSE的key可以存储为明文形式（red key），混淆模式（gray key），或者是加密模式（black key）。

eFUSE的写入过程叫做Provisioning。通过PJTAG（on MIO）使RPU或者APU处理器写寄存器的方式完成Provisioning。 [Ref, Programming BBRAM and eFUSEs Application Note (XAPP1319)](https://docs.xilinx.com/r/dqE2tE0k~iMhpEDoQwXIKg/kdP~nr5__We0rfXUfmQbXQ?section=XREF_13077_20_Programming).

在一些老的ZYNQ设计中是支持读回操作来验证写入的eFUSE数据是否正确。但对于现阶段的ZYNQ来说具备很高的风险性，所以这个途径已经被**关闭读回**了。取而代之的是，写入eFUSE之后同时会返回CRC32，可以通过对比返回的CRC来确定值是否正确。

从安全角度来看，无法确定“一次性”的功能给eFUSE带来多少的增益。但是要注意，写eFUSE的时候不要用SPA对密钥进行分析，也要确保电压稳定，否则可能导致eFUSE写入失败。

最后，请注意通过APB总线，可以加载Key到 key update register中。这个设计主要是为了boot阶段使用了rolling key，对不同块加密的时候要频繁的换key。就是通过这个寄存器实现。

### HRoTS

ZYNQ的HRoTS基于RSA-4096认证，而且需要基于两类的public key：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202211091421374.png)

Primay的2个公钥，一个需要存储在外部的内存中，一个需要把其hash值存入eFUSE；因此，Primay的pk hash是一个根凭证。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202211091423224.png" width="60%" /></div> 

最小需要做Provisioning的信息如上。

## 2.2 eFUSE layout

存在两类eFUSE，一个是256-PS的eFUSEs，还有是128-PL的eFUSEs。

参考： https://docs.xilinx.com/r/dqE2tE0k~iMhpEDoQwXIKg/2Ubsx6RiXaJnAO2rAuCVYA?section=XREF_67790_Zynq_UltraScale

| Size | Name                                                                                                                                                                                                                                | Description                                                                                                                                                                                                                                                                                                     | XilSKey Name:XSK_EFUSEPS_                            |
|  ----  |  ----  |  ----  |  ----  |
| 32   | USER_{0:7}                                                                                                                                                                                                                          | 256 user defined eFUSEs:**Note:** In the **input.h**file (see text), write data in the XSK_EFUSEPS_USER{0:7}_FUSES macro and execute the write by setting the XSK_EFUSEPS_USER{0:7}_FUSE macro = True.                                                                                              | USER{0:7}_FUSE                                       |
| 1    | USER_WRLK                                                                                                                                                                                                                           | 8 user-defined eFUSE locks.USER_WRLK columns:0: Locks USER_0,1: Locks USER_1,...7: Locks USER_7,**Note:** Each eFUSE permanently locks the entire corresponding user-defined USER_{0:7} eFUSE row so it cannot be changed.                                                                                | USER_WRLK_{0:7}                                      |
| 1    | LBIST_EN                                                                                                                                                                                                                            | Enables logic BIST to run during boot.                                                                                                                                                                                                                                                                          | LBIST_EN                                             |
| 3    | LPD_SC                                                                                                                                                                                                                              | Enables zeroization of registers in low power domain (LBD) during boot.**Note:** Any of the eFUSE programmed will perform zeroization. Xilinx recommends programming all of them.                                                                                                                         | LPD_SC_EN                                            |
| 3    | FPD_SC                                                                                                                                                                                                                              | Enables zeroization of registers in full power domain (FBD) during boot.**Note:** MGTs must be powered to perform zeroization of the FPD.**Note:** Any of the eFUSE programmed will perform zeroization. Xilinx recommends programming all of them.                                                 | FPD_SC_EN                                            |
| 3    | PBR_BOOT_ERROR                                                                                                                                                                                                                      | When programmed, boot is halted on any PMU error.                                                                                                                                                                                                                                                               | PBR_BOOT_ERR                                         |
| 32   | CHASH                                                                                                                                                                                                                               | PUF helper data                                                                                                                                                                                                                                                                                                 | N/A - handled by PUF registration software directly. |
| 24   | AUX                                                                                                                                                                                                                                 | PUF helper data: ECC vector                                                                                                                                                                                                                                                                                     | N/A - handled by PUF registration software directly. |
| 1    | SYN_INVLD                                                                                                                                                                                                                           | Invalidates PUF helper data stored in eFUSEs.                                                                                                                                                                                                                                                                   | XSK_PUF_SYN_INVALID                                  |
| 1    | SYN_LOCK                                                                                                                                                                                                                            | Locks PUF helper data from future programming.                                                                                                                                                                                                                                                                  | XSK_PUF_SYN_WRLK                                     |
| 1    | REG_DIS                                                                                                                                                                                                                             | Disables PUF registration.                                                                                                                                                                                                                                                                                      | XSK_PUF_REGISTER_DISABLE                             |
| 1    | AES_RD                                                                                                                                                                                                                              | Disables the AES key CRC integrity check for eFUSE key storage.                                                                                                                                                                                                                                                 | AES_RD_LOCK                                          |
| 1    | AES_WR                                                                                                                                                                                                                              | Locks AES key from future programming.                                                                                                                                                                                                                                                                          | AES_WR_LOCK                                          |
| 1    | ENC_ONLY[^(1)^](https://docs.xilinx.com/r/dqE2tE0k~iMhpEDoQwXIKg/2Ubsx6RiXaJnAO2rAuCVYA?section=XREF_29338_1_IMPORTANT)[^(2)^](https://docs.xilinx.com/r/dqE2tE0k~iMhpEDoQwXIKg/2Ubsx6RiXaJnAO2rAuCVYA?section=XREF_36827_2_When_the_ENC) | When programmed, all partitions are required to be encrypted. Xilinx recommends using this only if security is required and the hardware root of trust (RSA_EN) is not used.                                                                                                                                    | ENC_ONLY                                             |
| 1    | BBRAM_DIS                                                                                                                                                                                                                           | Disables the use of the AES key stored in BBRAM.                                                                                                                                                                                                                                                                | BBRAM_DISABLE                                        |
| 1    | ERR_DIS                                                                                                                                                                                                                             | Prohibits error messages from being read via JTAG (ERROR_STATUS register).**Note:** The error is still readable from inside the device.                                                                                                                                                                   | ERR_DISABLE                                          |
| 1    | JTAG_DIS[^(1)^](https://docs.xilinx.com/r/dqE2tE0k~iMhpEDoQwXIKg/2Ubsx6RiXaJnAO2rAuCVYA?section=XREF_29338_1_IMPORTANT)                                                                                                                | Disables JTAG. IDCODE and BYPASS are the only allowed commands.                                                                                                                                                                                                                                                 | JTAG_DISABLE                                         |
| 1    | DFT_DIS[^(1)^](https://docs.xilinx.com/r/dqE2tE0k~iMhpEDoQwXIKg/2Ubsx6RiXaJnAO2rAuCVYA?section=XREF_29338_1_IMPORTANT)                                                                                                                 | Disables design for test (DFT) boot mode.                                                                                                                                                                                                                                                                       | DFT_DISABLE                                          |
| 3    | PROG_GATE                                                                                                                                                                                                                           | When programmed, these fuses prohibit the PROG_GATE feature from being engaged. If any of these are programmed, the PL is always reset when the PS is reset.**Note:** Only one eFUSE needs to be programed to prohibit the PROG_GATE feature from being engaged. Xilinx recommends programming all three. | PROG_GATE_DISABLE                                    |
| 1    | SEC_LK                                                                                                                                                                                                                              | When programmed, the device does not enable BSCAN capability while in secure lockdown.                                                                                                                                                                                                                          | SECURE_LOCK                                          |
| 15   | RSA_EN[^(1)^](https://docs.xilinx.com/r/dqE2tE0k~iMhpEDoQwXIKg/2Ubsx6RiXaJnAO2rAuCVYA?section=XREF_29338_1_IMPORTANT)[^(2)^](https://docs.xilinx.com/r/dqE2tE0k~iMhpEDoQwXIKg/2Ubsx6RiXaJnAO2rAuCVYA?section=XREF_36827_2_When_the_ENC)   | When any one of the eFUSEs is programmed, every boot must be authenticated using RSA. Xilinx recommends programming all 15 eFUSEs.                                                                                                                                                                              | RSA_ENABLE                                           |
| 1    | PPK0_WR                                                                                                                                                                                                                             | Primary public key write lock. When programmed, this prohibits future programming of PPK0.                                                                                                                                                                                                                      | PPK0_WR_LOCK                                         |
| 2    | PPK0_INVLD                                                                                                                                                                                                                          | When either of the eFUSEs are programmed, PPK0 is revocated. Xilinx recommends programming both eFUSEs when revocating PPK0.                                                                                                                                                                                    | PPK0_INVLD                                           |
| 1    | PPK1 WR                                                                                                                                                                                                                             | Primary public key write lock. When programmed this prohibits future programming of PPK1.                                                                                                                                                                                                                       | PPK1_WR_LOCK                                         |
| 2    | PPK1_INVLD                                                                                                                                                                                                                          | When either of the eFUSEs are programmed, PPK1 is revocated. Xilinx recommends programming both eFUSEs when revocating PPK1.                                                                                                                                                                                    | PPK1_INVLD                                           |
| 32   | SPK_ID                                                                                                                                                                                                                              | Secondary public key ID.**Note:** Write the SPK ID bits into the XSK_EFUSEPS_SPK_ID eFUSE array and set XSK_EFUSEPS_SPKID = True.                                                                                                                                                                         | SPK_ID                                               |
| 256  | AES                                                                                                                                                                                                                                 | User AES key**Note:** Write data in the XSK_EFUSEPS_AES_KEY macro and execute the write by setting the XSK_EFUSEPS_WRITE_AES_KEYmacro = True.                                                                                                                                                             | AES_KEY                                              |
| 384  | PPK0                                                                                                                                                                                                                                | User primary public key0 HASH**Note:** Write data in the XSK_EFUSEPS_PPK0_HASH macro. To program 256 bits, use the LSBs and set XSK_EFUSEPS_PPK0_IS_SHA3 = False. To program 384 bits, set XSK_EFUSEPS_PPK0_IS_SHA3 = True. Execute the write by setting the XSK_EFUSEPS_WRITE_PPK0_HASH macro = True.    | PPK0_HASH                                            |
| 384  | PPK1                                                                                                                                                                                                                                | User primary public key1 HASH**Note:** Write data in the XSK_EFUSEPS_PPK1_HASH macro. To program 256 bits, use the LSBs and set XSK_EFUSEPS_PPK1_IS_SHA3 = False. To program 384 bits, set XSK_EFUSEPS_PPK1_IS_SHA3 = True. Execute the write by setting the XSK_EFUSEPS_WRITE_PPK1_HASH macro = True.    | PPK1_HASH                                            |
| N/A  | PUF_HD                                                                                                                                                                                                                              | Syndrome of PUF HD. These eFUSEs are programmed using Xilinx provided software, Xilskey                                                                                                                                                                                                                         | N/A - handled by PUF registration software directly. |

这个表，很关键，Xilinx不接受返回材料DMA。在操作这些位的时候，尤其是Provisioning的时候，会影响其他非安全的测试。

>**IMPORTANT**: THESE INSTRUCTIONS MODIFY THE EFUSES ON THE ZCU102 DEVELOPMENT BOARD AND MAY LIMIT FUTURE USE OF THE DEVELOPMENT BOARD FOR NON-SECURE TESTING AND DEBUGGING!

## 2.2 PUF

### 2.2.1 PUF operations

只有CSU可以访问PUF，所以PUF的初始化包括黑密钥的产生，必然是在CSU阶段完成。PUF暴露出以下接口供CSU操作：

|  Command   | Description  |
|  ----  | ----  |
| Registration  | 产生一个新的KEK并且关联helper data |
| Re-registration  | 产生一个新的KEK并且关联**新的**helper data  |
| Reuse  | 使用已经存在的KEK进行加解密，并且关联helper data  |

当一个设备上电的时候，CSU bootROM会检测已经校验过的boot header，确认以下信息：
* 是否使用了PUF；
* 加密的密钥存储在哪里？（eFUSE 或者 boot image）
* helper data存储在哪里？（eFUSE 或者 boot image）

接着CSU初始化PUF，并且加载helper data，最后产生KEK，`这个过程叫做regeneration`。一旦KEK产生成功，CSU bootROM使用它来解密剩下boot image需要的key。

### 2.2.2 PUF Control eFUSEs

eFUSE也给PUF留了一些功能接口：

|  Command   | Description  |
|  ----  | ----  |
| REG_DIS  | 屏蔽PUF的注册 |
| SYN_INVALID  | 使存在eFUSE上的helper data无效  |
| SYN_LOCK  | 屏蔽修改eFUSE上的helper data的功能  |

Xilinx也做了PUF的强度分析，包含加密，安全强度KEK，过温过压的测试的数据报告，需要联系FAE或者销售才可以拿到这个报告。

### 2.2.3 PUF Helper Data

PUF使用大约4Kb的辅助数据来帮助PUF在正确的寿命、规定的工作温度和电压范围内重新创建原始KEK值。辅助数据由Syndrome值、Aux值和Chash值组成（请参见表：PUF辅助数据）。助手数据可以存储在eFUSE或引导映像中。

| PUF Helper Data Field | Size (Bits) | Description |
| ------- | ------------- | ------------- |
| Syndrome | 4060 | 鉴于环形振荡器在温度、电压和时间上的微小变化，这些比特有助于PUF恢复正确的PUF特征 |
| Aux | 24 | 这是一个汉明码，允许PUF对PUF签名执行某种程度的纠错。 |
| Chash | 32 | 这是PUF签名的散列，允许PUF识别重新生成的签名是否正确。•如果CHASH未编程，则只要使用（BH_auth或rsa_en），就可以使用BH黑键。•如果对CHASH进行了编程，则只要（使用bh_auth或rsa_en）且eFUSE综合征数据未失效，就可以使用eFUSE黑键。•如果对CHASH进行了编程，则只要使用（BH_auth或rsa_en）且efuse综合征数据无效，就可以使用BH黑键。 |

# 3. Example

## 3.1 Provisioning

Provisioning是所有安全机制的基础。其目的是把根凭证写入eFUSE或者其他存储密钥的敏感介质。包含RSA的根密钥，也包含GCM的Key。

在ZYNQ中我们需要Provisioning的内容：
* PPK HASH （主引导Primary Public Key Hash）
* GCM-AES Key （用于image解密）
* PSK ID (写入key的id信息，作为PUF解密时候比对，参考'防重放攻击'一节)

在调试阶段为了不伤害eFUSE。对于PPK HASH，zynq提供了bh_boot模式，即可以绕过eFUSE的PPK HASH检测，直接使用AC中的hash值。

同样，为了不伤害eFUSE。对于GCM-AES key，我们可以不使用eFUSE，而把Key烧录到BBRAM中。

**在调试阶段，推荐配置为**：
* **ppk hash**，对于验证，使用bh_boot（不需要包含到Provisioning过程中）
* **gcm aes key**，对于验证，使用bbram （需要包含到Provisioning过程中）
* id 
* **ID**，对于验证，在bootgen阶段选择`spk_id = 0`，不需要包含到Provisioning中。

**在产品阶段，必须配置为**： ppk hash 和 gcm aes key都需要在eFUSE中。

我们可以把完整的Provisioning过程总结为：
* 手动产生两对RSA密钥，产生gcm-aes-key；
* 使能PUF eFUSE的配置；
* 使用写入寄存器的方法写入eFUSE。

**可以通过JTAG烧录eFUSE（这种方法数据操作互动型，具备一定的危险性，eFUSE烧录不可撤销），因此建议使用配置寄存器的方法烧录eFUSE**。

### 3.1.1 Gen Key

密钥生成主要是需要以下材料：
* AES Key Generation
	* 输出nky文件（包含key和iv）
* RSA Asymmetric Key Generation
	* 输出1：psk0.pem
	* 输出2：ssk0.pem
* Generate SHA3 of Public RSA Asymmetric Key
	* 输出是：sha3.txt

### 3.1.2 PUF eFUSE config

PUF eFUSE的配置需要使用Vitis建立AP的工程，使用ZYNQ上面的AP来完成PUF的配置。非常重要的提示：**这一步会修改eFUSE上面的内容**！

需要配置的项目是：
* XSK_PUF_INFO_ON_UART
* XSK_PUF_PROGRAM_EFUSE
* XSK_PUF_PROGRAM_SECUREBITS 
* XSK_PUF_SYN_WRLK
* XSK_PUF_AES_KEY
* XSK_PUF_IV

`XSK_PUF_AES_KEY`是用户指定的，而且这个`XSK_PUF_IV`和AES Key Generation中的IV不相关。这个IV是用于PUF KEK的red key加密。

### 3.1.3 RSA eFUSE config
这一步是配置RSA相关的信息到eFUSE上面，非常重要的提示：**这一步会修改eFUSE上面的内容**！

* XSK_EFUSEPS_RSA_ENABLE
* XSK_EFUSEPS_PPK0_WR_LOCK
* XSK_EFUSEPS_WRITE_PPK0_HASH
* XSK_EFUSEPS_PPK0_HASH

### 3.1.4 RSA Key Revocation Support

RSA密钥提供了撤销一个分区的**secondary**密钥（SPK）的能力，而无需撤销所有分区的密钥。这是通过使用新的BIF参数`spk_select`利用`USER_FUSE0`到`USER_FUSE7` 位域实现（如果这些位域没有用于表示其他信息，只用于表示密钥的id，最多可以撤销256个SPK，如图所示）。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221120140817.png)

下图表示ZYNQ使用SPK_ID进行SPK revocation的过程。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221120140510.png" width="60%" /></div> 

以下是使用辅助密钥创建经过身份验证的映像的步骤：
* 使用bootgen生成RSA密钥对。
* 我们在步骤1中生成了一个辅助密钥（SSK）。如果需要更多的SSK，请重复步骤1以创建SSK密钥。
* 使用bootgen和下面的bif文件模板生成经过验证的引导映像。下面的模板假设bootloader和u-boot使用[sskfile]标记提供的密钥进行了验证，并且该密钥根据eFUSE中存储的SPK_ID进行了验证；PMU FW和ATF images使用sskfile提供的密钥进行验证，并根据存储在USER_eFUSE中的SPK_ID bitmap 验证该密钥。(确保在生成映像时使用命令行参数–efuseppkbits<path_to_sha_txt_file>命令bootgen生成PPK哈希。)
* 使能RSA认证通过设定 “RSA_EN” 在eFUSE上. 参考 [Programming BBRAM and eFUSEs Application Note (XAPP1319)](https://www.xilinx.com/support/documentation/application_notes/xapp1319-zynq-usp-prog-nvm.pdf)
* 写入在第三步创建的PPK的SHA-3 hash 到eFUSE的PPK0 hash位域。

确保在生成image时使用命令行参数–efuseppkbits<path_to_sha_txt_file>命令bootgen生成PPK hash。

**示例**：
image header和FSBL使用不同的SSK进行身份验证（分别为ssk1.pem和ssk2.pem），用以下bif文件：

```
the_ROM_image: {
[auth_params]ppk_select = 0
[pskfile]psk.pem
[sskfile]ssk1.pem
[bootloader, authentication = rsa, spk_select = spk-efuse, spk_id = x00000001, sskfile = ssk2.pem]zynqmp_fsbl.elf
[destination_cpu =a53-0, authentication = rsa, spk_select = user-efuse,spk_id = 0x1, sskfile = ssk3.pem]Application1.elf
[destination_cpu =a53-0, authentication = rsa, spk_select = spk-efuse, spk_id = 0x00000001, sskfile = ssk4.pem]Application2.elf
}
```

相同的SSK将作用于image header和FSBL（ssk2.pem）：
```
the_ROM_image: {
[auth_params]ppk_select = 0 [pskfile]psk.pem
[bootloader, authentication = rsa, spk_select = spk-efuse, spk_id = 0x00000001, sskfile = ssk2.pem]zynqmp_fsbl.elf
[destination_cpu =a53-0, authentication = rsa, spk_select = user-efuse, spk_id = 1, sskfile = ssk3.pem]Application1.elf
[destination_cpu =a53-0, authentication = rsa, spk_select = spk-efuse, spk_id = 0x00000001, sskfile = ssk4.pem]Application2.elf
}
```

注意：
* `spk_select = spk-efuse` 表示 指定的分区将会使用`spk_id`eFUSE位域。 
* `spk_select = user-efuse` 指示 指定的分区将会使用user eFUSE位域，而CSU ROM总是使用`spk_id`eFUSE位域。


## 3.2 PUF Enc/Dec demo

完成上面PUF的注册，我们假定eFUSE和PUF的配置已经OK了，现在我们需要编写AP的固件（baremental程序），来使用PUF的加密和解密功能。

该固件是在xilinx的vitis ide上完成的，vitis已经集成了ZYNQ所用的bsp驱动包，并提供了相应的操作key、加密解密、访问寄存器、控制外设等接口。

打开vitis ide创建工程：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221120142657.png)

选择bsp包：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221120142729.png)

选择processor为APU：CortexA53_0：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221120142810.png)

导入baremental的源码（下面就是源码核心）：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221120142920.png)

源码进行编译，最后生成`BOOT.BIN`文件，将其复制到SD卡的boot分区。启动即可运行。

### 加密

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202211111334247.png" width="90%" /></div> 

对于一个PUF加密过程的程序：
```C
void puf_encrypt(u8 *Iv, u8 *Dst, u8 *Src, u32 Size) {

	XCsuDma_Config *Config;
	XCsuDma_Configure ConfigurValues = {0};

    /* Configure PUF configuration 0 and configure the shutter. */
	XilSKey_WriteReg(XSK_ZYNQMP_CSU_BASEADDR, XSK_ZYNQMP_CSU_PUF_CFG0,
                    XSK_ZYNQMP_CSU_PUF_CFG0_DEFAULT);
	XilSKey_WriteReg(XSK_ZYNQMP_CSU_BASEADDR, XSK_ZYNQMP_CSU_PUF_SHUT,
                    XSK_ZYNQMP_CSU_PUF_SHUT_DEFAULT);


	// Spin up the PUF and connect the key to the AES engine
	XilSKey_WriteReg(XSK_ZYNQMP_CSU_BASEADDR, XSK_ZYNQMP_CSU_PUF_CMD,
                    XSK_ZYNQMP_PUF_REGENERATION);

    /* Wait for the PUF regeneration to complete. */
	usleep(PUF_REGEN_TIME_US);

	/* Initialize & configure the DMA */
	Config = XCsuDma_LookupConfig(XSK_CSUDMA_DEVICE_ID);
	XCsuDma_CfgInitialize(&CsuDma, Config, Config->BaseAddress);

	/* Initialize AES engine */
	XSecure_AesInitialize(&AesInstance, &CsuDma, XSK_PUF_DEVICE_KEY, (u32 *) Iv, NULL);

	/* Set the data endianess for IV */
	XCsuDma_GetConfig(&CsuDma, XCSUDMA_SRC_CHANNEL,
				&ConfigurValues);
	ConfigurValues.EndianType = 1U;
	XCsuDma_SetConfig(&CsuDma, XCSUDMA_SRC_CHANNEL,
					&ConfigurValues);

	/* Enable CSU DMA Dst channel for byte swapping.*/
	XCsuDma_GetConfig(&CsuDma, XCSUDMA_DST_CHANNEL,
			&ConfigurValues);
	ConfigurValues.EndianType = 1U;
	XCsuDma_SetConfig(&CsuDma, XCSUDMA_DST_CHANNEL,
			&ConfigurValues);

	/* Request to encrypt the AES key using PUF Key	 */
	XSecure_AesEncryptData(&AesInstance, (u8 *) Dst, (u8 *) Src, Size);

   /* Clear the PUF key. */
	XilSKey_WriteReg(XSK_ZYNQMP_CSU_BASEADDR, XSK_ZYNQMP_CSU_PUF_CMD,
                    XSK_ZYNQMP_PUF_RESET);
}
```

### 解密

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202211111338407.png" width="90%" /></div> 

```C
s32 puf_decrypt(u8 *Iv, u8 *Dst, u8 *Src, u32 Size, u8 *GcmTagPtr) {
	s32 Status;

	XCsuDma_Config *Config;
	XCsuDma_Configure ConfigurValues = {0};

   /* Configure PUF configuration 0 and configure the shutter. */
	XilSKey_WriteReg(XSK_ZYNQMP_CSU_BASEADDR, XSK_ZYNQMP_CSU_PUF_CFG0,
                    XSK_ZYNQMP_CSU_PUF_CFG0_DEFAULT);
	XilSKey_WriteReg(XSK_ZYNQMP_CSU_BASEADDR, XSK_ZYNQMP_CSU_PUF_SHUT,
                    XSK_ZYNQMP_CSU_PUF_SHUT_DEFAULT);

	// Spin up the PUF and connect the key to the AES engine
	XilSKey_WriteReg(XSK_ZYNQMP_CSU_BASEADDR, XSK_ZYNQMP_CSU_PUF_CMD,
                    XSK_ZYNQMP_PUF_REGENERATION);

   /* Wait for the PUF regeneration to complete. */
	usleep(PUF_REGEN_TIME_US);

	/* Initialize & configure the DMA */
	Config = XCsuDma_LookupConfig(XSK_CSUDMA_DEVICE_ID);
	XCsuDma_CfgInitialize(&CsuDma, Config, Config->BaseAddress);

	/* Initialize AES engine */
	XSecure_AesInitialize(&AesInstance, &CsuDma, XSK_PUF_DEVICE_KEY, (u32 *) Iv, NULL);

	/* Set the data endianess for IV */
	XCsuDma_GetConfig(&CsuDma, XCSUDMA_SRC_CHANNEL,
				&ConfigurValues);
	ConfigurValues.EndianType = 1U;
	XCsuDma_SetConfig(&CsuDma, XCSUDMA_SRC_CHANNEL,
					&ConfigurValues);

	/* Enable CSU DMA Dst channel for byte swapping.*/
	XCsuDma_GetConfig(&CsuDma, XCSUDMA_DST_CHANNEL,
			&ConfigurValues);
	ConfigurValues.EndianType = 1U;
	XCsuDma_SetConfig(&CsuDma, XCSUDMA_DST_CHANNEL,
			&ConfigurValues);

	/* Request to decrypt the AES key using PUF Key	 */
	Status = XSecure_AesDecryptData(&AesInstance, (u8 *) Dst, (u8 *) Src, Size,
                                   (u8 *) GcmTagPtr);

   /* Clear the PUF key. */
	XilSKey_WriteReg(XSK_ZYNQMP_CSU_BASEADDR, XSK_ZYNQMP_CSU_PUF_CMD,
                    XSK_ZYNQMP_PUF_RESET);

   return Status;
}

```

# Ref
[^1]:[External Secure Storage Using the PUF Application Note](https://docs.xilinx.com/r/en-US/xapp1333-external-storage-puf/External-Secure-Storage-Using-the-PUF-Application-Note)
[^2]:[Programming BBRAM and eFUSEs Application Note (XAPP1319)](https://docs.xilinx.com/v/u/en-US/xapp1319-zynq-usp-prog-nvm)
[^3]:[Zynq UltraScale+ Device Technical Reference Manual (UG1085) - Security](https://docs.xilinx.com/r/en-US/ug1085-zynq-ultrascale-trm/Introduction?tocId=A~Ce_pZ6I0b4P1VxSk8qcg)
[^4]:[secure-boot-encrypted-data-storage](https://www.timesys.com/security/secure-boot-encrypted-data-storage/)



