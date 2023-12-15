系统芯片（SoC，System on a Chip）的密钥管理机制对于确保整个系统的安全性至关重要。SoC通常是集成了多个功能模块（如处理器核心、内存、输入/输出控制等）的单个芯片，广泛应用于智能手机、平板电脑、嵌入式系统等设备中。SoC的密钥管理机制的重要性可以从以下几个方面来理解：

1. **数据保护**：SoC的密钥管理机制用于加密和解密数据，确保存储或传输的数据安全，防止敏感信息泄露。
2. **安全引导**：密钥管理机制可以用于实现安全引导过程，确保设备在启动时只加载和执行经过验证的、可信的软件。
3. **设备认证**：SoC可以利用密钥管理机制实现设备级别的身份认证，确保设备在网络或系统中的身份是合法和可信的。
4. **数字版权管理（DRM）**：在处理版权受保护的内容（如数字媒体内容）时，密钥管理机制用于实施DRM策略，防止非法复制和分发。
5. **安全通信**：密钥管理对于保障设备之间的安全通信至关重要，例如在进行加密通信或执行其他安全相关操作时。
6. **防止篡改和攻击**：有效的密钥管理机制可以帮助防止各种安全威胁，如篡改攻击、重放攻击等。
7. **合规性**：对于需要遵守特定安全标准和法规的应用（如金融服务、健康信息处理），密钥管理机制有助于确保合规。
8. **多样性和灵活性**：在复杂的应用场景中，密钥管理机制提供了处理多种安全需求的灵活性和扩展性。

SoC的密钥管理机制是确保整个系统安全、保护数据隐私和完整性、维护设备和网络的安全信任链的基础。在设计和实施SoC时，对密钥管理机制的考虑是至关重要的。

本文对一般的SoC中的key管理机制进行调研，并进行总结，并以NXP等著名的SoC为例，来作为Key管理的设计。

* Key种类
* Key管理 （存储or派生）
* Key的使用场景
* 一些Key管理的SoC设计

# 1. Key Types

我们可以按照这个角度进行分类：
* Non-Volatile Keys（非易失性密钥）
* Volatile Keys（易失性密钥）

| 特点     | 非易失性密钥（Non-Volatile Keys）            | 易失性密钥（Volatile Keys）                 |
|--------|-----------------------------------------|-----------------------------------------|
| 存储   | 存储在非易失性存储器中，如闪存或电池支持的RAM | 存储在易失性存储器中，如普通RAM             |
| 用途   | 用于长期存储，如设备根密钥、身份认证密钥等   | 用于短期操作，如会话密钥、临时加密/解密密钥等 |
| 安全性 | 需要严格的安全措施防止未授权访问或篡改      | 即使被泄露，也只会对短期会话构成风险         |
| 生命周期 | 通常在设备整个生命周期内保持不变，或者仅在特定情况下更新 | 生命周期通常很短，可能仅限于单个会话或特定操作 |

## 1.1 非易失性Key

在密钥管理中，非易失性密钥（Non-Volatile Keys）是指那些存储在非易失性存储器（如eFUSE、闪存、EEPROM或NVRAM）中的密钥，这些密钥即使在设备断电或重启后也不会丢失。

在SoC的设计中，经常有Master Key（或者HUK）存入到eFUSE中，然后通过在SoC的总线限定，只准SHE（Security Hardware Engine）读取，叫法没有统一。 可以总结为有以下特点：

* Keys stored in the OTP (eFUSE)
* Key的可读性受到OTP的配置影响（例如，LCS生命周期，具体需要看硬件设计）
* Fixed key length
* Dereferenced by NVK index
* Crypto Engine uses only

## 1.2 易失性Key

易失性密钥（Volatile Keys）是在设备运行时生成并且在断电或重启后消失的密钥。这些密钥在安全系统中通常用于短期操作，如加密会话数据或临时认证。

在SoC的设计中，该Key存在于SRAM中，在权限管理上没有Master Key那么严格，通常其他硬件也可以访问到该Key。这部分key有以下特点：

* Keys are stored in the dedicated SRAM
* Variable key length
	* DES/TDES/SM4/CHACHA20/AES-128/192/256
	* MAYBE: RSA/EC??
* Dereferenced by VK index
* Crypto Engine uses, and **probable other h/w entities**

一些PUF技术中，也可以产生易失性Key， https://www.researchgate.net/figure/Simple-PUF-based-cryptographic-volatile-key-generator-structure_fig10_289601897

也可以参考 ZYNQ的  https://github.com/carloscn/blog/issues/149 

# 2. Key管理

SoC（System on a Chip）的设计在密钥管理方面扮演着至关重要的角色，因为它直接决定了密钥管理的有效性和安全性。考虑到不同的SoC可能有不同的硬件架构和安全特性，理解特定SoC的密钥管理机制需要具体分析。首先，SoC的硬件结构必须支持高级的密钥管理功能。这意味着需要有专门的硬件模块或组件，比如安全加密处理器或安全存储区域，来处理和存储密钥。同时，为了确保密钥在SoC内部传输的安全性，必须实现安全的总线架构。这可能包括加密总线通信或限制对敏感数据路径的访问，以防止潜在的侧信道攻击或数据泄露。SoC应该提供一个完善的密钥管理系统，包括密钥的生成、存储、使用和销毁。这可能涉及到复杂的软件和硬件交互，以确保密钥的生命周期管理符合安全要求。SoC需要支持SHE或类似的技术，以提供额外的安全措施，比如防篡改和抗侧信道攻击。

为了更好地理解这些概念，可以选择一个具体的SoC实例来详细介绍其**Key Ladder**密钥管理机制。通过分析这个实例，可以揭示SoC设计如何影响密钥的生成、保护、使用和管理。

## 2.1 SoC机制

这幅图展示了一个集成在系统芯片(SoC)中的密码引擎的架构设计，主要用于处理和管理安全密钥和执行加密运算。

![image-20231215112255539](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202312151122592.png)

在这个示例中，

* **Crypto Engine**核心处理单元，处理各种加密算法。它可能包括多种加密模块，如对称加密算法(Symmetric Cryptography Algorithms, SCA)，哈希消息认证码(HMAC)，和一次性密码(One-Time Password, OTP)。
* **Key Store**，它负责存储安全密钥，以保护密钥不被非授权访问。Key Store通过AXI总线与密码引擎连接，这意味着密钥存储操作可能需要较高的数据传输速率。
* **Key Master**: 作为密钥管理的中心，负责处理所有与密钥相关的操作，如生成、分发、存储和废弃密钥。它可能还负责确保密钥只能在授权的条件下使用。
* **Secure Bus**: 一个独立的总线系统，专门用于安全相关的通信。这表明设计了一个隔离的环境，用于执行敏感操作，以提高整体系统的安全性。

在设计的时候需要考虑的安全特性包含：

* “Anti-Scan”，扫描攻击是一种旨在揭露集成电路内部结构和数据的技术，包括通过各种手段如逻辑分析、电源分析等来收集信息。常常抵抗的方法是，物理屏障，加密，随机化等。
* Dedicated pin 2 pin connection
	* Bury lines in a deep metal layer
* Secure bus (Proprietary)

## 2.2 SoC权限管理

Key Store Access Restrictions

对于key存储访问权限需要严格限制，读/写/使用。在SoC的driver中应该设定角色：

![image-20231215140702335](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202312151407443.png)

- **读取限制** — 输入数据
    - 安全的密码引擎主机可以读取密钥。
    - 非安全的密码引擎主机在运行时不能读取密钥。
    - CPU在运行时不能读取密钥。
- **写入限制** — 输出数据
    - 安全的密码引擎主机可以写入密钥，例如密钥装载授权数据（KLAD）、密钥派生函数（KDF）。
    - 安全CPU可能可以写入密钥。
    - 非安全的密码引擎主机/CPU不能写入密钥。
- **使用限制** — 加密密钥
    - 基于安全性、操作和密码算法限制密码引擎的使用
        - 安全主机和非安全主机的不同权限
        - 解密或加密的权限，或者两者都可以
        - 支持的算法包括DES/TDES/SM4/AES-128/192/256, HMAC
    - 存在其他的secure aware IP可能使用这个keys。

## 2.3 派生 Key Derivation

Key ladder（密钥阶梯）机制是一种密钥派生方法，用于生成一系列相关联的加密密钥。这种机制在内容保护和密钥管理系统中尤为常见，用于确保加密密钥的安全传输和存储。这一机制的核心是通过使用一系列密钥派生步骤来确保上游密钥（root key或者master key）的安全性，同时允许下游密钥（derived keys）在不暴露上游密钥的情况下用于各种操作。

在Key ladder机制中，通常有以下几个特点：

1. **密钥层级**: 密钥被组织成层级结构。在顶层是最重要的密钥（根密钥），它通常是固定的并且安全存储。这个密钥用于生成下一级别的密钥。
2. **密钥派生**: 从上一层密钥派生下一层密钥。这个过程可以通过加密算法，如AES、TDES等，以及一个称为密钥派生函数（Key Derivation Function, KDF）的算法来完成。
3. **一次性使用**: 下游密钥通常是一次性使用的，或者用于特定的会话或者事务。一旦使用完毕，这些密钥就会被废弃，新的操作会派生新的密钥。
4. **安全传递**: Key ladder机制允许密钥以安全的方式从一个设备传递到另一个设备，因为在传递过程中只有派生的密钥被传递，而不是根密钥。
5. **权限控制**: 使用这种机制可以对不同级别的密钥进行权限控制，比如某些密钥只能用于加密，而另一些只能用于解密。
6. **密钥隔离**: 不同级别的密钥用于不同的目的，确保即使下游密钥被破解，上游密钥也不会受到影响。

 ![image-20231215142323202](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202312151423276.png)

这个来源于 https://www.etsi.org/deliver/etsi_ts/103100_103199/103162/01.01.01_60/ts_103162v010101p.pdf

简易版就是：

![image-20231215143504605](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202312151435646.png)

对于输入：

* Support non-volatile and volatile keys.
* Support processing EKs from memory.
* Support processing EKs from the key store.

对于输出：

* Support output data to the selected key slots.
	* Never reveal keys to CPU.
	* Either internal or external (Connect KLAD to external key store using dedicated secure bus.
)

