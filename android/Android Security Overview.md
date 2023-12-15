本文来源于以下的文献，对其进行整理：
* https://source.android.com/docs/security?hl=zh-cn
* https://developer.android.com/quality/privacy-and-security
* https://github.com/OWASP/owasp-mastg/blob/master/Document/0x02a-Frontispiece.md
* https://mas.owasp.org/MASTG/Intro/0x01-Foreword/
* Book -- Android Security Internals An In-Depth Guide to Android's Security Architecture

# 1. Android  Security Arch

## 1.1 Security Model

在信息安全领域，依赖单一的安全机制往往是不足够的，因为每种机制都可能有其局限性和潜在的漏洞。因此，系统安全性更多地依赖于多种机制的组合，这种方法被称为“防御深度”（defense in depth）。一个全面的、多层次的安全策略被认为是保护信息系统安全的最佳实践。因此，有必要拿出一些篇幅，解释Android的安全模型，以联合Android的所有安全话题。

"Android的Security Model"（Android的安全模型）指的是Android操作系统用于保护设备和数据安全的一系列策略、机制和技术。这个安全模型旨在防止恶意软件攻击和未授权访问，同时确保用户数据的隐私和完整性。

### 1.1.1 Android Arch

Android 软件堆栈是指构成 Android 操作系统的各个软件层级和组件。这个堆栈包括从底层的硬件抽象层到用户界面的各个部分。下面是一个简单的概述：

![](https://raw.githubusercontent.com/carloscn/images/main/typoratyporatypora202312141503730.png)

1. **Linux 内核**：Android 基于 Linux 内核，它处理诸如系统安全、内存管理、进程管理、网络堆栈和驱动程序等核心系统功能。
2. **硬件抽象层（HAL）**：这一层为上层的 Android 框架提供标准接口，用于访问硬件功能。每种硬件（如摄像头、蓝牙模块等）都有自己的接口。
3. **原生用户空间（Native Userspace）**：这一层包括各种系统库，它们支持 Android 的核心功能，如图形处理、数据存储和网页浏览等。
4. **Android 运行时（ART）和 Dalvik 虚拟机**：这些是 Android 的应用运行环境。ART 为应用程序提供了垃圾回收、编译和执行环境。
5. **Java API 框架**：这层提供了构建 Android 应用所需的各种 API。开发者可以利用这些 API 访问设备功能，如用户界面、资源管理等。
6. **系统应用**：这包括一系列内置的 Android 应用，如电话、联系人、短信等。

整个堆栈工作在一起，使 Android 设备能够运行各种应用程序和服务，同时为最终用户提供丰富的交互体验。每一层都在为更高层提供服务，同时依赖更低层的功能和资源。

### 1.1.2 Android Security Topics

基于Android Stack 和 https://source.android.com/docs/security?hl=zh-cn 以下为整理出来的基于Android架构的安全话题。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312141530016.png)


# 2. Security Checklist for APPs

根据Android  Security Arch，我们对APP做以下要求。该要求为 https://mas.owasp.org/MASTG/Intro/0x01-Foreword/ OWASP开源组织设计。我们分别从以下维度做限制，分别考虑，架构层级的设计、数据保密性和隐私、使用密码学算法、认证管理、网络安全、以及抗逆向工程的要求。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202312141510231.png)

## 2.1 Architecture, Design and Threat Modeling Requirements

在理想的世界中，安全性会在开发的所有阶段中得到考虑。然而，在现实中，安全性通常只在软件开发生命周期的后期阶段才被考虑。除了技术控制之外，移用安全验证标准（MASVS）要求实施确保在规划移动应用架构时显式地解决安全问题的流程，并且知道所有组件的功能和安全角色。

“V1”类别列出了与应用的架构和设计相关的要求。为了涵盖诸如威胁建模、安全的软件开发生命周期或密钥管理等主题，应该参考相应的OWASP项目和/或其他标准。

| V1   |              | Architecture, design and threat  modelling                   | L1    |L2   |      |  参考 |
| ---- | ------------ | ------------------------------------------------------------ | ---- | ---- | ---- | ------------------------------------------------------------ |
| 1.1  | MSTG-ARCH-1  | All app components are  identified and known to be needed.   | ✓    | ✓    |      | [Architectural Information](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x04b-Mobile-App-Security-Testing.md#architectural-information) |
| 1.2  | MSTG-ARCH-2  | Security controls are never  enforced only on the client side, but on the respective remote endpoints. | ✓    | ✓    |      | [Injection Flaws (MSTG-ARCH-2 and MSTG-PLATFORM-2)](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x04h-Testing-Code-Quality.md#injection-flaws-mstg-arch-2-and-mstg-platform-2) |
| 1.3  | MSTG-ARCH-3  | A high-level architecture for  the mobile app and all connected remote services has been defined and  security has been addressed in that architecture. | ✓    | ✓    |      | [Architectural Information](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x04b-Mobile-App-Security-Testing.md#architectural-information) |
| 1.4  | MSTG-ARCH-4  | Data considered sensitive in the  context of the mobile app is clearly identified. | ✓    | ✓    |      | [Identifying Sensitive Data](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x04b-Mobile-App-Security-Testing.md#identifying-sensitive-data) |
| 1.5  | MSTG-ARCH-5  | All app components are defined  in terms of the business functions and/or security functions they provide. |      | ✓    | N/A  | [Environmental Information](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x04b-Mobile-App-Security-Testing.md#environmental-information) |
| 1.6  | MSTG-ARCH-6  | A threat model for the mobile  app and the associated remote services has been produced that identifies  potential threats and countermeasures. |      | ✓    | N/A  | [Mapping the Application](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x04b-Mobile-App-Security-Testing.md#mapping-the-application) |
| 1.7  | MSTG-ARCH-7  | All security controls have a  centralized implementation.    |      | ✓    | N/A  | [Testing for insecure Configuration of Instant Apps   (MSTG-ARCH-1, MSTG-ARCH-7)](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x05h-Testing-Platform-Interaction.md#testing-for-insecure-configuration-of-instant-apps-mstg-arch-1-mstg-arch-7) |
| 1.8  | MSTG-ARCH-8  | There is an explicit policy for  how cryptographic keys (if any) are managed, and the lifecycle of  cryptographic keys is enforced. Ideally, follow a key management standard  such as NIST SP 800-57. |      | ✓    | N/A  | [Cryptographic policy](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x04g-Testing-Cryptography.md#cryptographic-policy) |
| 1.9  | MSTG-ARCH-9  | A mechanism for enforcing  updates of the mobile app exists. |      | ✓    | N/A  | [Testing enforced updating (MSTG-ARCH-9)](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x05h-Testing-Platform-Interaction.md#testing-enforced-updating-mstg-arch-9) |
| 1.10 | MSTG-ARCH-10 | Security is addressed within all  parts of the software development lifecycle. |      | ✓    | N/A  | [Security Testing and the SDLC](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x04b-Mobile-App-Security-Testing.md#security-testing-and-the-sdlc) |
| 1.11 | MSTG-ARCH-11 | A responsible disclosure policy  is in place and effectively applied. |      | ✓    | N/A  |                                                              |
| 1.12 | MSTG-ARCH-12 | The app should comply with  privacy laws and regulations.    | ✓    | ✓    |      |                                                              |

| V1   |              | 架构、设计和威胁建模                                  | L1    |L2   |      |  参考 |
| ---- | ------------ | ------------------------------------------------------ | ---- | ---- | ---- | ------------------------------------------------------------ |
| 1.1  | MSTG-ARCH-1  | 已识别出所有应用组件，并确认它们是必需的。               | ✓    | ✓    |      | [Architectural Information](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x04b-Mobile-App-Security-Testing.md#architectural-information) |
| 1.2  | MSTG-ARCH-2  | 安全控制不仅仅在客户端强制执行，也要在相应的远端端点强制执行。   | ✓    | ✓    |      | [Injection Flaws (MSTG-ARCH-2 and MSTG-PLATFORM-2)](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x04h-Testing-Code-Quality.md#injection-flaws-mstg-arch-2-and-mstg-platform-2) |
| 1.3  | MSTG-ARCH-3  | 已定义移动应用及所有连接的远程服务的高层架构，并在该架构中解决了安全问题。 | ✓    | ✓    |      | [Architectural Information](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x04b-Mobile-App-Security-Testing.md#architectural-information) |
| 1.4  | MSTG-ARCH-4  | 清晰地识别了在移动应用上下文中被认为是敏感的数据。           | ✓    | ✓    |      | [Identifying Sensitive Data](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x04b-Mobile-App-Security-Testing.md#identifying-sensitive-data) |
| 1.5  | MSTG-ARCH-5  | 所有应用组件都根据它们提供的业务功能和/或安全功能来定义。     |      | ✓    | N/A  | [Environmental Information](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x04b-Mobile-App-Security-Testing.md#environmental-information) |
| 1.6  | MSTG-ARCH-6  | 已生成移动应用及其关联远程服务的威胁模型，识别出潜在威胁和对策。 |      | ✓    | N/A  | [Mapping the Application](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x04b-Mobile-App-Security-Testing.md#mapping-the-application) |
| 1.7  | MSTG-ARCH-7  | 所有安全控制都有一个集中实施的机制。                       |      | ✓    | N/A  | [Testing for insecure Configuration of Instant Apps   (MSTG-ARCH-1, MSTG-ARCH-7)](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x05h-Testing-Platform-Interaction.md#testing-for-insecure-configuration-of-instant-apps-mstg-arch-1-mstg-arch-7) |
| 1.8  | MSTG-ARCH-8  | 明确的密钥管理政策，强制执行密钥生命周期。理想情况下，遵循密钥管理标准，如NIST SP 800-57。 |      | ✓    | N/A  | [Cryptographic policy](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x04g-Testing-Cryptography.md#cryptographic-policy) |
| 1.9  | MSTG-ARCH-9  | 存在强制更新移动应用的机制。                              |      | ✓    | N/A  | [Testing enforced updating (MSTG-ARCH-9)](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x05h-Testing-Platform-Interaction.md#testing-enforced-updating-mstg-arch-9) |
| 1.10 | MSTG-ARCH-10 | 在软件开发生命周期的所有部分中都解决了安全问题。             |      | ✓    | N/A  | [Security Testing and the SDLC](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x04b-Mobile-App-Security-Testing.md#security-testing-and-the-sdlc) |
| 1.11 | MSTG-ARCH-11 | 已经建立并有效执行的负责任披露政策。                       |      | ✓    | N/A  |                                                              |
| 1.12 | MSTG-ARCH-12 | 应用应遵守隐私法律和规定。                                | ✓    | ✓    |      |                                                              |



## 2.2 Data Storage and Privacy

| V2   |                 | Data Storage and Privacy                                     |      |      |      |                                                              |
| ---- | --------------- | ------------------------------------------------------------ | ---- | ---- | ---- | ------------------------------------------------------------ |
| 2.1  | MSTG-STORAGE‑1  | System credential storage  facilities need to be used to store sensitive data, such as PII, user  credentials or cryptographic keys. | ✓    | ✓    |      | [Testing Local Storage for Sensitive Data (MSTG-STORAGE-1 and   MSTG-STORAGE-2)](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x05d-Testing-Data-Storage.md#testing-local-storage-for-sensitive-data-mstg-storage-1-and-mstg-storage-2) |
| 2.2  | MSTG-STORAGE‑2  | No sensitive data should be  stored outside of the app container or system credential storage facilities. |      |      |      | [Testing Local Storage for Sensitive Data (MSTG-STORAGE-1 and   MSTG-STORAGE-2)](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x05d-Testing-Data-Storage.md#testing-local-storage-for-sensitive-data-mstg-storage-1-and-mstg-storage-2) |
| 2.3  | MSTG-STORAGE‑3  | No sensitive data is written to  application logs.           | ✓    | ✓    |      | [Testing Logs for Sensitive Data (MSTG-STORAGE-3)](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x05d-Testing-Data-Storage.md#testing-logs-for-sensitive-data-mstg-storage-3) |
| 2.4  | MSTG-STORAGE‑4  | No sensitive data is shared with  third parties unless it is a necessary part of the architecture. | ✓    | ✓    |      | [Determining Whether Sensitive Data is Sent to Third Parties   (MSTG-STORAGE-4)](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x05d-Testing-Data-Storage.md#determining-whether-sensitive-data-is-sent-to-third-parties-mstg-storage-4) |
| 2.5  | MSTG-STORAGE‑5  | The keyboard cache is disabled  on text inputs that process sensitive data. | ✓    | ✓    |      | [Determining Whether the Keyboard Cache Is Disabled for Text   Input Fields (MSTG-STORAGE-5)](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x05d-Testing-Data-Storage.md#determining-whether-the-keyboard-cache-is-disabled-for-text-input-fields-mstg-storage-5) |
| 2.6  | MSTG-STORAGE‑6  | No sensitive data is exposed via  IPC mechanisms.            | ✓    | ✓    |      | [Determining Whether Sensitive Stored Data Has Been Exposed   via IPC Mechanisms (MSTG-STORAGE-6)](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x05d-Testing-Data-Storage.md#determining-whether-sensitive-stored-data-has-been-exposed-via-ipc-mechanisms-mstg-storage-6) |
| 2.7  | MSTG-STORAGE‑7  | No sensitive data, such as  passwords or pins, is exposed through the user interface. | ✓    | ✓    |      | [Checking for Sensitive Data Disclosure Through the User   Interface (MSTG-STORAGE-7)](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x05d-Testing-Data-Storage.md#checking-for-sensitive-data-disclosure-through-the-user-interface-mstg-storage-7) |
| 2.8  | MSTG-STORAGE‑8  | No sensitive data is included in  backups generated by the mobile operating system. |      | ✓    | N/A  | [Testing Backups for Sensitive Data (MSTG-STORAGE-8)](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x05d-Testing-Data-Storage.md#testing-backups-for-sensitive-data-mstg-storage-8) |
| 2.9  | MSTG-STORAGE‑9  | The app removes sensitive data  from views when moved to the background. |      | ✓    | N/A  | [Finding Sensitive Information in Auto-Generated Screenshots   (MSTG-STORAGE-9)](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x05d-Testing-Data-Storage.md#finding-sensitive-information-in-auto-generated-screenshots-mstg-storage-9) |
| 2.10 | MSTG-STORAGE‑10 | The app does not hold sensitive  data in memory longer than necessary, and memory is cleared explicitly after  use. |      | ✓    | N/A  | [Checking Memory for Sensitive Data (MSTG-STORAGE-10)](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x05d-Testing-Data-Storage.md#checking-memory-for-sensitive-data-mstg-storage-10) |
| 2.11 | MSTG-STORAGE‑11 | The app enforces a minimum  device-access-security policy, such as requiring the user to set a device  passcode. |      | ✓    | N/A  | [Testing the Device-Access-Security Policy (MSTG-STORAGE-11)](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x05d-Testing-Data-Storage.md#testing-the-device-access-security-policy-mstg-storage-11) |
| 2.12 | MSTG-STORAGE‑12 | The app educates the user about  the types of personally identifiable information processed, as well as  security best practices the user should follow in using the app. |      | ✓    | N/A  | [Testing User Education (MSTG-STORAGE-12)](https://github.com/OWASP/owasp-mstg/blob/1.1.3-excel/Document/0x04i-Testing-user-interaction.md#testing-user-education-mstg-storage-12) |
| 2.13 | MSTG-STORAGE‑13 | No sensitive data should be  stored locally on the mobile device. Instead, data should be retrieved from a  remote endpoint when needed and only be kept in memory. |      | ✓    | N/A  |                                                              |
| 2.14 | MSTG-STORAGE‑14 | If sensitive data is still  required to be stored locally, it should be encrypted using a key derived  from hardware backed storage which requires authentication. |      | ✓    | N/A  |                                                              |
| 2.15 | MSTG-STORAGE‑15 | The app’s local storage should  be wiped after an excessive number of failed authentication attempts. |      | ✓    | N/A  |                                                              |

# 3. APP实例分析
