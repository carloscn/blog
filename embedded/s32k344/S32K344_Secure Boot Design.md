
# 1. Overview

Upper document: [ [S32K344] Boot And Reset Sequence](https://github.com/carloscn/blog/issues/215) 

The secure boot flow involves multiple software bootloaders – this gives a great deal of scope for customisation to ensure that boot process is optimal for specific use cases.

From a high level, the flow is as follows:

![](https://raw.githubusercontent.com/carloscn/images/main/typora202407020935978.png)

And the secure boot flow is:

![](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202407020936337.png)


## 1.1 Chip reset and boot flow

The following figure shows a high-level representation of the chip startup sequence. This sequence consists of several reset stages based on the occurrence of a particular reset event, and the Secure Boot Assist Flash 
(SBAF) is the first code to run on HSE_B secure core after reset, which completes necessary system initialization, parses the Image Vector Table (IVT), executes secure boot and starts the application core.

![](https://raw.githubusercontent.com/carloscn/images/main/typora202406271621822.png)

 There are 3 configurations on the chip boot stage:
 
* Off HSE -> Non-Secure Boot
* On HSE + Configures the Boot Configuration Word (BCW) with Non-Secure Boot -> Non-Secure Boot
* On HSE + Configures the Boot Configuration Word (BCW) with Secure Boot -> Secure Boot

## 1.2 The HSE interface

The Messaging Unit (MU) is the communication interface between the host and the HSE subsystem. It is used by the host to trigger service requests and receive service responses. It is used by the HSE to receive service requests, return service responses and provide several HSE status information relevant to the host.

The MU has two sides, referred to as MUA and MUB. One side (MUA) is under the exclusive control of the HSE, the other side (MUB) is controlled by the host. A value written in a transmit register (TRi) on one side can be read in the corresponding service register (RRi) on the other side. Similarly, select control registers on one side (e.g. FCR) interact with status registers on the other side (e.g. FSR).

Each of the MU instance available in the system has:
* A set of 32-bit readable and writable transmit registers (**TRi**), to provide the address of the service descriptors to process by the HSE.
* A set of 32-bit read-only receive registers (**RRi**), to retrieve the responses to the service requests
* Two 32-bit read-only status registers (**FSR** and **GSR**), to log HSE status and system events
* Control and status registers to manage the access to the transmit and receive registers, and the related interrupt signals

The number if available MU instances and TRi / RRi registers is device (host) dependent:

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora202406280859095.png" width="80%" /> </div>

For more HSE, refer to https://github.com/carloscn/blog/issues/217

## 1.3 Image Vector Table (IVT)

The IVT (sometimes referred as “boot header”) is the main entry point for the system to operate after reset. Refer to https://github.com/carloscn/blog/issues/215 

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora202406280943785.png" width="80%" /> </div>

It contains:
* The storage locations of **Apps(executable)** and **HSE firmware(encrypted)**, and t**he pointers to the configuration of Life Cycle** (LC) and **Extended Resource Domain Controller** (XRDC).
* The Boot Configuration Word (**BCW**) not only configures the start-up behavior (BOOT_SEQ),  but also defines the application cores to be booted (BOOT_TARGET).

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora202406280945290.png" width="80%" /> </div>

The IVT can hold an optional authentication tag that guaranties its integrity and authenticity: it is calculated by the HSE on demand from the host via one specific administration service and verified by the HSE at start-up if it is configured to do so. The storage locations of IVT are fixed by design.

## 1.4 AppBL

AppBL is in Boot Images and its pointer is populated in the IVT file. After the HSE checks, whether the IVT file is passed, the AppBL will be started, as the process illustrated in the following figure, and circled by a red line.

![](https://raw.githubusercontent.com/carloscn/images/main/typora202406280952072.png)

The constituent elements of AppBL structure are shown in the figure below:

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora202406280956974.png" width="80%" /> </div>

### How to the secure boot check it? 

Different mode of the secure boot has different behavior.  

In the **basic secure boot mode**, the HSE can authenticate the AppBL header and content by using AES-GMAC algorithm and ADKP SHA256 hash key, and the first step is to parse the AppBL header to determine the start address and size of the AppBL code.

In the **advanced secure boot mode**, the HSE doesn't need to parse or verify the AppBL header as its mode is implemented by SMR (Secure Memory Region) and CR (Core Rest). The SRM is a block of specail memory region, the user can write the start address and length of the AppBL directly to the SMR service structure to complete the configuration. But for reducing the coupling between different projects, the start address and length of the AppBL code can also be obtained from the AppBL header to configure the SMR.

## 1.5 Secure Boot Modes

As shown in the Table 1 below, there are three available mechanisms to configure the secure boot flow for the application images:

![](https://raw.githubusercontent.com/carloscn/images/main/typora202406281006348.png)

The procedures of configuring these secure boot modes are shown in the figure:

![](https://raw.githubusercontent.com/carloscn/images/main/typora202406281007893.png)

### 1.5.1 Basic Secure Boot (BSB)

Keywords: HSE verify + AppBL header

This secure boot mode is implemented based on the AppBL header and ADKP, and the HSE firmware enables only one application core at a time. The length of AppBL header is 64 bytes and the application code address starts from “AppBL header start address + AppBL header length”.

### 1.5.2 Advanced Secure Boot (ASB)

In the ASB mode, the HSE firmware can boot multiple application cores by configuring Secure Memory Regions (SMR) and Core Reset (CR) tables, which combined to define the application cores behavior. These **tables** are configured via HSE firmware services and are stored in internal data flash memory.

Pre-requisites before the advanced secure boot can be executed are as follows:
* The host shall be granted with Super User (SU) rights.
* LC should be CUST_DEL and subsequent SMRs and CRs should be empty.

### 1.5.3 SHE based Secure Boot (SSB)

Since HSE firmware also executes the SSB flow by using SMR/CR tables similarly, this secure boot mode can be considered as a **special use case of ASB**, the only difference between of them is that only SMR #0 and SHE keys shall be used to implement the SSB flow.

# 2. Advance Secure Boot (ASB)

This section describes how do both Advanced Secure Boot and SHE based Secure Boot modes which are all implemented with SMR and CR tables realize with HSE memory verification services, implemented with SMR (Secure Memory Regions) and CR (Checkpoints) tables, utilize HSE (Hardware Security Engine) memory verification services. The SMRs managed by these services can apply various sanctions(punishment) if secure boot fails. They support multiple authentication schemes (MAC, RSA/ECC signatures) to verify application images and accelerate verification at startup by relying on authenticity checks performed by the HSE. The picture below shows the execution process of the memory verification service based on SMR and CR tables.

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora202406281159876.png" width="80%" /> </div>

 A secure memory region (SMR) is defined by a start address and a size, associated to a proof of authenticity, either a MAC or an RSA/ECC signature, which authenticates the region’s content. The host can define up to 8 SMR clustered into the SMR table. It must also provide the proof of authenticity for each memory region content except for SMR #0. This means 8 different keys are stored in the Key catalogs.

For all SMR that have been defined, the HSE verifies the authenticity of memory contents:
* During the device start-up phase (after reset).
* While the application(s) is(are) running on the host side (during run-time).
* The SMR verification results translate into sanctions imposed on the system by the HSE:
	* Unsuccessful verification can keep the selected subsystems on the host side in **reset state**, those subsystems are referenced in the **Core Reset (CR) table**.
	* Likewise, failing to verify certain SMR can render the selected keys within the HSE unusable, these restrictions are defined individually for each key via the SMR verification map.

## 2.1 Secure Memory Region (SMR)

The SMR contains the key points as the following diagram:

![](https://raw.githubusercontent.com/carloscn/images/main/typora202407011025277.png)

### 2.1.1 SRM table

The SMR table which stored in HSE secure data flash for devices with internal flash allows the host to define up to **8 memory regions** and associate each one with an installation and a verification method. Each SMR entry in the SMR table holds a set of attributes listed in the below:

| Attribute                | Data field       | Description                                                                                                                |
|--------------------------|------------------|----------------------------------------------------------------------------------------------------------------------------|
| Source address           | pSmrSrc          | A pointer to the secure memory region (SMR) to be verified in the application NVM area which HSE can directly read.       |
| Size                     | smrSize          | A 32-bit integer that provides the size in bytes of the secure memory region (SMR) to be verified.                         |
| Destination address      | pSmrDest         | A pointer where the secure memory region (SMR) is copied before verification.                                              |
| Initial authentication proof | pInstAuthTag[] | Pointers to the initial value of a MAC or an RSA/ECC signature provided by the host, used in the SMR verification process if the flag HSE_SMR_CFG_FLAG_INSTALL_AUTH is set. |
| Authentication scheme    | authScheme       | The method used to authenticate the SMR including an authentication tag (i.e. Message Authentication Code (MAC)) or a public key signature scheme (i.e. RSA or ECC signature). |
| Authentication key       | keyHandle        | The handle which points to the authentication key in the NVM key catalog. It must: <br> - Refer to a non-empty key slot having its key usage flag HSE_KF_USAGE_VERIFY set, while HSE_KF_USAGE_SIGN must not be set. <br> - Refer to a key type that matches with the initial authentication scheme selected. |
| Decryption parameters    | smrDecrypt       | Optional parameters for SMR decryption when an encrypted SMR is installed. More details please refer to REF02 Table 84.    |
| Verification period      | checkPeriod      | A 32-bit integer that defines the scaled number of system clock cycles between two consecutive verification processes.      |
| SMR configuration flags  | configFlags      | A binary OR combination of configuration flags between a memory interface and the authenticity tool used for verification.  |
| Version Offset           | versionOffset    | The offset in SMR where the image version can be found. The SMR version offers protection for the image against rollback attacks during update. |

#### Authentication strategy

The necessary preparation for the secure memory region (SMR) installation is to determine the authentication **scheme**, authentication **key** and initial **authentication** proof.

**Scheme** :

As Figure 16 shows, multiple verification schemes include MAC and signature are supported to verify the SMR during either the installation or verification phase, the specific parameters for each different scheme also need to be filled by the user. For example, when using the GMAC or rsaPss scheme, the user needs to manually configure the specific parameters as figure above.

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typoratypora202406281424858.png" width="70%" /> </div>

**Key**:

The authenticity key must be stored in the NVM key catalog and its authentication key usage flag should be HSE_KF_USAGE_VERIFY rather than HSE_KF_USAGE_SIGN.

**Proof**:

The initial authenticity proof (pInstAuthTag[]) is an array of two pointers which point to the address of MAC or SIGN. If the HSE_SMR_CFG_FLAG_INSTALL_AUTH flag is set, it specifies the address of the initial authentication proof. If cleared, this data field is not used (internal hash digest SHA2-256 is used).

It should be noted that for MAC and RSA signature authentication schemes, only pInstAuthTag[0] is used, while both pInstAuthTag[0] and pInstAuthTag[1] are used for ECDSA and EDDDSA signatures (specified by (r,s), with "r" at index 0, and "s" at index 1).

#### Configuration flags

shows some details about the effect of “configFlags” in different conditions.

| scheme | configFlags | effect on auth-proof |
|--------|-------------|----------------------|
| MAC    | 0 | Use internal hash digest (SHA2-256) |
| MAC    | HSE_SMR_CFG_FLAG_INSTALL_AUTH | Compute MAC of the data. |
| SHE    | 0 | Return HSE_SRV_RSP_NOT_ALLOWED. |
| SHE    | HSE_SMR_CFG_FLAG_INSTALL_AUTH | Compute pure CMAC of the data, as HIS-SHE specification required. |
| **SIGN**   | 0 | Use internal hash digest (SHA2-256) |
| **SIGN**   | HSE_SMR_CFG_FLAG_INSTALL_AUTH | Compute RSA/ECC private key signature of the data. |

If the data field “configFlags” is set as `HSE_SMR_CFG_FLAG_INSTALL_AUTH`, the authentication proof(pInstAuthTag) provided by the user in the installation phase, will also be used in the verification phase. **The authentication proof must be written on the S32K3xx internal FLASH(P-Flash or D-Flash).**

If the data field configFlags is cleared, **the HSE will use the internal calculation of the authentication proof** (pInstAuthTag is not used) during the installation phase, save it internally, and use the auth tag for verification during the verification phase. Even if the pInstAuthTag in smrEntry is not used, when the installation service hseSmrEntryInstallSrv_t is called, the application still needs to provide the pAuthTag and authTagLength to ensure the integrity of the initial data, except in SHE secure boot mode.

When configFlags is cleared, SMR verification will be much faster than when configFlags is set to HSE_SMR_CFG_FLAG_INSTALL_AUTH and does not require additional FLASH space storage. However, if the user needs to use SMR to complete secure boot on a device with OTA enabled (HSE AB swap firmware), the configFlags must be set to HSE_SMR_CFG_FLAG_INSTALL_AUTH.

**Note**:
> The authentication proof must be written on the S32K3xx internal FLASH (P-Flash or D-Flash).

### 2.1.2 SHE based Secure Boot (SMR #0)

The SMR #0 is the **only SMR** that can be associated to the SHE AES key BOOT_MAC_KEY  (keyHandle) as the SMR authentication key. In this case, the reference authentication tag is the CMAC (authScheme) value referred to as BOOT_MAC which can be initialized and updated via the SHE key update protocol.

In addition, when host is granted with SU rights, BOOT_MAC can be automatically calculated as described below.

On the first SMR #0 installation using BOOT_MAC_KEY, if BOOT_MAC is empty (i.e. not initialized) and if BOOT_MAC_KEY has been provisioned, the reference authentication tag is calculated by the HSE and saved in BOOT_MAC. When installing SMR #0 using the BOOT_MAC_KEY while the BOOT_MAC is already initialized, the BOOT_MAC value must be updated via the SHE key update 
protocol prior to issuing the SMR installation service.

**In all cases, the data field pInstAuthTag is always discarded and should be set to NULL.**

### 2.1.3 SMR installation (config CMAC key)

The host can request for SMR installation via the HSE service defined by the structure hseSmrEntryInstallSrv_t, the SMR installation service mainly takes in following inputs as following table:

**Parameters of structure `hseSmrEntryInstallSrv_t`**

|Data field|Description|
|---|---|
|`entryIndex`|Identifies the index of SMR entry (in the SMR table) which has to be installed/updated (Refer to #HSE_NUM_OF_SMR_ENTRIES).|
|`pSmrEntry`|Address of SMR entry structure containing the configuration properties to be installed, it will be stored and used in the verification phase (refer to `hseSmrEntry_t`).|
|`accessMode`|Specifies the access mode (ONE-PASS, START, UPDATE, FINISH).|
|`pSmrData`|The address where SMR data to be installed is located.|
|`smrDataLength`|The length of the SMR data.|
|`pAuthTag`|The address where SMR Original authentication tag is to be verified (located on FLASH or SRAM). It is necessary to provide the mac and sign of the data determined by `pSmrData` and `smrDataLength`. When the service is executed, the provided `pAuthTag` will be verified to ensure that the data in the installation phase is correct, except for SHE-boot.|
|`authTagLength`|The length of the SMR authentication proof (TAG or SIGN).|
|`cipher.pIV`|Initialization Vector/Nonce (16 bytes).|
|`cipher.pGmacTag`|The optional GMAC tag (16 bytes) used for AEAD.|

The first-time definition of a SMR entry can be performed when LC is set to CUST_DEL. In addition, most of the data fields in the SMR entry can be modified only when the host is granted with SU rights. 

The SMR installation via this service can be done in one-pass or streaming mode. The streaming mode is useful when the SMR content to install is not entirely available in the system memory when the installation starts (OTA use case). This service does not use a stream ID as HSE uses internal contexts when processing in streaming mode.

### 2.1.3.1. One-pass installation mode

When the SMR content to install is fully available in Flash or RAM, the most convenient way to process it is to run the service in one-pass mode, in this case:

- The data field `accessMode` must be set to `HSE_ACCESS_MODE_ONE_PASS`.
- The data field `pSmrData` must be equal to `pSmrEntry.pSmrSrc`, the source address where the SMR needs to be loaded from.
- The data field `smrDataLength` must be equal to `pSmrEntry.smrSize`, the size in bytes of the SMR to be loaded or verified.
- The data field `authTagLength[0]` (respectively `authTagLength[1]`) must be set with the size of the byte array pointed by the data field `pAuthTag[0]` (respectively `pAuthTag[1]`).

### 2.1.3.2. Streaming installation mode

It is possible to process a SMR installation even if the entire content is not already programmed in Flash or RAM. A typical example for such use case is an image (code or data) that is too big to fit in the available application RAM entirely and is provided to the host in chunks via a communication interface, and each individual chunk is then programmed in Flash, in this case:

- The SMR number (`entryIndex`), configuration (`pSmrEntry`) and decryption initialization vector (`cipher.pIV`) must be provided only in the START call.
- For the START, UPDATE or FINISH calls, the data field `pSmrData` must point to the next SMR chunk to process and the data field `smrDataLength` is set with the size of that chunk. The minimum chunk size is 64 bytes.
- The START and FINISH calls are mandatory, the UPDATE call is optional.
- The address (`pAuthTag[]`) and size (`authTagLength[]`) of the initial authenticity proof must only be provided during the FINISH call.

## 2.2 Core Rest(CR)

The Core Reset (CR) table allows the host to associate each CPU-driven subsystem available in a device with up to 8 SMR, so that sanctions are applied on those subsystems after the pre-boot and post-boot phases, depending on the SMR verification status. For the devices with internal flash user can install maximum 2 core reset entry.

![](https://raw.githubusercontent.com/carloscn/images/main/typora202407011036358.png)

### 2.2.1 CR Table

**Table. CR table entry attributes**

|Attribute|Data field|Description|
|---|---|---|
|Core identifier|`coreId`|A unique number that identifies a CPU-driven subsystem.|
|Pre-boot SMR verification map|`preBootSmrMap`|A set of flags that define which SMR, indexed from 0 to 7 (bit #i for SMR #i), must be verified before releasing from reset the associated subsystem.|
|Alternate Pre-boot SMR verification map|`altPreBootSmrMap`|A set of flags that define which SMR, indexed from 0 to 7 (bit #i for SMR #i), must be verified before releasing from reset the associated subsystem when one or more SMR specified in `preBootSmrMap` failed the verification. This can be used to declare backup image(s) for the associated CPU subsystem.|
|Post-boot SMR verification map|`postBootSmrMap`|A set of flags that define which SMR, indexed from 0 to 7 (bit #i for SMR #i), must be verified after releasing from reset the associated subsystem.|
|Reset address|`pPassReset`|A value of the VTOR of associated application subsystem: <ul><li>After a successful verification of all SMR specified in `preBootSmrMap`. This address must lie within one of the verified SMR.</li><li>Or unconditionally, if there are no SMR specified in `preBootSmrMap`, but they are linked through `postBootSmrMap` to the core reset entry. This is known as parallel secure boot and the verification is done after the core is released from reset. This address must lie within one of the loaded SMR.</li></ul>|
|Alternate reset address|`pAltReset`|A value of the VTOR of associated application subsystem if all the SMR defined in `altSmrVerifMap` pass the verification.|
|Core boot option|`startOption`|Specifies whether the core is automatically started by the HSE at boot-time or if the CR entry is used for on-demand booting at run-time.|
|Sanctions on failed verification|`crSanction`|The sanction HSE applies for the CR entry if one of the associated SMR fails verification.|

#### 2.2.1.1 Core identifier

Each CPU-driven subsystem is identified by a unique number (coreId). The below Table lists the  subsystems and their respective core identifiers in S32K3x4 devices.

<div align='center'> <img src="https://raw.githubusercontent.com/carloscn/images/main/typora202407011040150.png" width="60%" /> </div>

If a CPU subsystem is not listed in the Core Reset table, it is not released from reset by the HSE.

#### 2.2.1.2 Verification Map

Association of a CPU Subsystem with SMR:

The association of a CPU subsystem with a set of SMR is realized via the data fields `preBootSmrMap` and `altPreBootSmrMap`: when bit #i is set to 1, SMR #i is associated with that CPU subsystem. When the verification of all the associated SMR is done, the status of the CPU subsystems at the end of the pre-boot phase depends on several conditions as summarized in Table 9 below.

The reset address provided in the data fields `pPassReset` and `pAltReset` can be an address within the on-chip Flash. The address `pPassReset` must lie within one of the SMR listed in `preBootSmrMap` or `postBootSmrMap`. Similarly, the address `pAltReset` must lie within one of the SMR listed in `altPreBootSmrMap`.

Table. Status of the CPU subsystem (all SMR verified in pre-boot phase)

|Conditions|Status|
|---|---|
|`pPassReset` within a verified SMR|Release from reset at address `pPassReset`.|
|`pPassReset` not within a verified SMR|Verify `altPreBootSmrMap` if configured, otherwise, sanction is the same as if one SMR failed the verification (see below).|
|`pAltReset` within a verified SMR|Release from reset at address `pAltReset`.|
|`pAltReset` not within a verified SMR|Same as if one SMR failed the verification (see below).|

### 2.2.2 Sanctions

The sanction taken by the HSE for a CR entry associated with SMR that failed verification depends on the phase when it is applied  (i.e. pre-boot or post-boot).

#### 2.2.2.1 Pre-boot sanctions

The below Table 10 summarizes the conditions and HSE behavior in terms of sanctions applied in the pre-boot and booting phases. If the sanction is `HSE_CR_SANCTION_DIS_ALL_KEYS`, disable all keys; otherwise, key usage is individually disabled via the smrFlags key attribute.

![](https://raw.githubusercontent.com/carloscn/images/main/typora202407011049773.png)

#### 2.2.2.2 Post-boot sanctions

The below Table summarizes the conditions and HSE behavior in terms of sanctions applied in the post-boot phase.

![](https://raw.githubusercontent.com/carloscn/images/main/typora202407011051152.png)

### 2.2.3 CR Installation

The host can request for installing an entry in the Core Reset (CR) table via the service defined by the structure hseCrEntryInstallSrv_t, the CR table entry installation service mainly takes in following inputs as the table shows:

Table. Parameters of structure `hseSmrCrEntryInstallSrv_t`

|Data field|Description|
|---|---|
|`crEntryIndex`|A CR entry number between 0 and (HSE_NUM_OF_CORE_RESET_ENTRIES - 1).|
|`pCrEntry`|A set of attributes that holds the CR table entry as listed in the above Table.|

The first-time definition of a CR entry can be performed when LC is set to CUST_DEL. Once defined, a CR entry can be updated only when all the associated SMR have been successfully verified first. In addition, to modify any of the values in a CR entry already defined, the host must be granted with SU rights. 

In addition, it’s noted that at least one SMR should be linked to the CR entry via `pCrEntry.preBootSmrMap` or `pCrEntry.postBootSmrMap`.

## 2.3 SMR verification

![](https://raw.githubusercontent.com/carloscn/images/main/typora202407011404667.png)

### 2.3.1 One-time automatic SMR verification

When `BOOT_SEQ` equals 1 in IVT, HSE uses the configuration in SMR and CR tables to boot the application cores securely. As such, the SMR linked with the CR table are automatically verified once by the HSE during startup.

The automatic verification at start-up splits into three phases:

- **The pre-boot phase**: During this phase, the SMR are verified before any CPU subsystem in the host is released from reset; this is the first phase after start-up.
- **The boot phase**: During this phase, the SMR are verified after the first CPU subsystem in the host has been released from reset (when allowed); this is the second phase after start-up.
- **The post-boot phase**: During this phase, the SMR are verified after all CPU subsystems in the host have been released from reset (when allowed); this is the third phase after start-up.

The end of the pre-boot and boot phases can be monitored via the `HSE_STATUS_BOOT_OK` status flag, and the end of the post-boot phase can be monitored via the status flag `HSE_STATUS_INIT_OK` as illustrated in Figure 17 below.

Pre-boot / boot / post-boot phases (BOOT_SEQ == 1): 

![](https://raw.githubusercontent.com/carloscn/images/main/typora202407011405751.png)

### 2.3.1.1. Pre-boot phase

During the pre-boot phase, HSE parses the CR table from the smallest entry index to the highest. For each CR entry, SMR linked via `pCrEntry.preBootSmrMap` data field are verified first. If any of these SMR fails verification, HSE verifies the SMR specified by `pCrEntry.altPreBootSmrMap` data field if configured.

If all the SMR are verified successfully from either of the pre-boot SMR maps, HSE may release from reset the CPU subsystem in the host depending on the core reset release strategy.

If both pre-boot SMR maps (including `altPreBoot`) have at least one SMR for which the verification fails, HSE applies the sanction configured for that CR entry.

### 2.3.1.2. Boot phase and core reset release strategies

While the CR table is parsed in the pre-boot phase, HSE releases the associated CPU from reset according to the core reset release strategy, configurable via `hseAttrCoreResetRelease_t` attribute:

- **ALL_AT_ONCE**: By which HSE parses first the entire CR table and verifies all the associated pre-boot SMR entries and then releases from reset all CPU subsystems configured that passed the verification.
- **ONE_BY_ONE**: By which HSE releases from reset each CPU subsystem one by one, after the associated CR entry and pre-boot SMR entries have been verified successfully.

The pre-boot phase ends when the first CPU subsystem is released from reset.

The end of both pre-boot and boot phases, which implies all configured CPU subsystems being booted, is signaled by the HSE via the status flag `HSE_STATUS_BOOT_OK`.

### 2.3.1.3. Post-boot phase

After all configured application CPU subsystems are released from reset and the booting phase is over, HSE reiterates through the CR table and for each entry, it verifies the associated SMR linked via `pCrEntry.postBootSmrMap` data field.

If any of the SMR verified during the post-boot phase fails verification, HSE applies the configured sanction for the associated CR entry. In this case, `HSE_CR_SANTCION_KEEP_CORE_IN_RESET` is not applicable (i.e. HSE can’t keep an booted application CPU in RESET phase).

### 2.3.1.4. Secure Boot Flow

The following Figure 18 details the SMR verification processes during pre-boot and post-boot phases and the sanctions taken by the HSE at the end of each phase. The red and blue lines represent the boot flow of the pure PRE_BOOT and POST_BOOT modes respectively.

![](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202407011444416.png)

### 2.3.2. On-demand SMR verification

SMR entries which are not linked to the CR table are unverified until the host triggers, at run-time, the verification via the service defined by the structure `hseSmrVerifySrv_t`.

This service takes only one data field (`entryIndex`) that specifies the index of the SMR to verify.

This service can also be used to verify any SMR during run-time. However, when the SMR to verify is in Flash, it must be ensured that no concurrent programming operation is triggered by the host while the verification takes place.

### 2.3.3. Recurrent automatic SMR verification

When its data field `pSmrEntry.checkPeriod` is set to a value different from 0, a SMR is automatically verified recurrently by the HSE during run-time, i.e. during normal operating conditions, once the pre-boot and post-boot phases are over.

The verification recurrence is defined by several system clock cycles, each unit corresponding to 10ms at maximum frequency. For example, if `smrEntry.checkPeriod = 200`, a verification process is triggered every 2s for a system clock frequency of 400MHz, 4s at 200MHz, etc.

It can be configured for any SMR that is loaded in RAM and for which the internal proof of authenticity generated by HSE is used for verification (i.e. `HSE_SMR_CFG_FLAG_INSTALL_AUTH` is not set).

### 2.3.4. SHE secure boot mode (only use SMR #0)

The SMR #0 is the only SMR that can be associated to the SHE AES key `BOOT_MAC_KEY` as the SMR authentication key. In this case, the reference authentication tag is the CMAC value referred to as `BOOT_MAC`.

If `BOOT_SEQ = 1`, authentication process started as mentioned in REF02 section 8.7.2. The reference authentication tag is calculated by the HSE and compared with saved `BOOT_MAC`. If `BOOT_SEQ = 0`, authentication process started as mentioned in REF02 section 8.5.4.2.

The Figure 19 shows the SMR verification process of the SHE based secure boot mode which also uses SMR and CR tables.

![](https://raw.githubusercontent.com/carloscn/images/main/typora202407011445538.png)

### 2.3.5. HSE status

By reading the values of bits 16 to 31 of FSR register in `MU_0` to get the status of HSE as Table 13. HSE global status bits in FSR

|Bit|Description|
|---|---|
|31|RFU|
|30|RFU|
|29|RFU|
|28|`HSE_STATUS_OEM_SUPER_USER`; when set to 1, indicates that SU rights are granted to OWNER_OEM|
|27|`HSE_STATUS_CUST_SUPER_USER`; when set to 1, indicates that SU rights are granted to OWNER_CUST|
|26|`HSE_STATUS_BOOT_OK`; set to 1 when all the secure boot conditions (pre-boot phase) defined in the HSE successfully pass|
|25|`HSE_STATUS_INSTALL_OK`; set to 1 once the key catalogs have been successfully formatted; when cleared to 0, indicates to the host that the key catalogs must be formatted|
|24|`HSE_STATUS_INIT_OK`; set to 1 when the HSE initialization is completed; when cleared to 0, no service request can be made to the HSE (MU disabled)|
|23|`HSE_STATUS_HSE_DEBUGGER_ACTIVE`; set to 1 when a HSE debug session is active|
|22|`HSE_STATUS_HOST_DEBUGGER_ACTIVE`; set to 1 when a host debug session is active|
|21|`HSE_STATUS_RNG_INIT_OK`; set to 1 when the RNG initialization is complete; when cleared to 0, any services using random number is unavailable to the host|
|20|`HSE_SHE_STATUS_SECURE_BOOT_OK`; set to 1 when SMR #0 successfully verified against BOOT_MAC|
|19|`HSE_SHE_STATUS_SECURE_BOOT_FINISHED`; set to 1 when SMR #0 was not successfully verified|
|18|`HSE_SHE_STATUS_SECURE_BOOT_INIT`; set to 1 when SMR #0 has been installed and authenticated with BOOT_MAC_KEY|
|17|`HSE_SHE_STATUS_SECURE_BOOT`; set to 1 when SMR #0 has been installed and BOOT_SEQ equals 1|
|16|RFU|

There are several secure boot related bits which need to be noted:

- `Bits #26`: `HSE_STATUS_BOOT_OK`; set to 1 when all the secure boot conditions (pre-boot phase) defined in the HSE successfully pass.
- `Bits #24`: `HSE_STATUS_INIT_OK`; set to 1 when the post-boot phase is successful.
- `Bits #17 to #20`: in the HSE status relate to the SHE secure boot are not valid when using advanced secure boot.
- `Bits #22`: set to “1” when a host(S32K3xx) debug session is active and set to “0” when debugger disconnected.

### 2.3.6. SMR core status

HSE system attribute service `HSE_SMR_CORE_BOOT_STATUS` request the SMR verification status and core boot status as Table 14 shows, from the HSE.

The structure `hseAttrSmrCoreStatus_t` of this service provides the following information:

![](https://raw.githubusercontent.com/carloscn/images/main/typora202407011446009.png)

- SMR verification status corresponding to the entries present in SMR table (refer to `smrStatus[]`).
- Provides Core Boot status (refer to `coreBootStatus[]`).
- In case BSB is performed, it provides the Core Boot status and the location of loaded application (primary/backup, refer to `coreBootStatus[]`).

# 3. Basic Secure Boot

The BSB is a simplified boot scheme, not like the ASB based on SMR. In this case, **the HSE FW only enables one application core** (the booted core can start other cores) at a time, based on the “Boot Data Sign” service defined by the structure hseBootDataImageSignSrv_t.

In addition, there is no sanction can be used for BSB mode when verification fails, the application core goes into recovery mode and no program will be executed. 

## 3.1 Application debug key/password (ADKP)

The ADKP is an HSE OTP (one-time program) attribute, **not like a normal NVM key which can be erased** (Root Trust of Hardware). It is used to calculate the GMAC of application of 16 bytes through AES-GMAC algorithm in RAM, based on a 256-bit AES key generated from SHA256 operation over the user-defined ADKP as illustrated in the below Figure 20(a). In addition, it should be noted that the ADKP must be set first before using the “Boot Data Sign/Verify” service or debugging protection.

![](https://raw.githubusercontent.com/carloscn/images/main/typora202407020910980.png)

## 3.2 Configuration

The host(S32K344) can request for the calculation of an authentication tag (GMAC) over the host system images via the service defined by the structure `hseBootDataImageSignSrv_t`.

This service is available to the host only when it is granted with the Super User (SU) rights (LC can be CUST_DEL, OEM_PROD or IN_FIELD) and the ADKP is written.

Before sending the service request to HSE, the AppBL header tag needs to be confirmed to be correct, or an error will occur.

A buffer used to store the GMAC (the authentication tag) value is needed for the “Boot Data Sign” service, and the AppBL start address also needs to be provided.

After this service has been completed, the generated GMAC from RAM needs to be copied back to the end of the application according to the codelength in AppBL.

![](https://raw.githubusercontent.com/carloscn/images/main/typora202407020914582.png)

To ensure the configuration above is completed and successful, the “Boot Data Verify” service defined by the structure `hseBootDataImageVerifySrv_t` can be called, the AppBL address needs to be provided.

Note, alignment
> Users must pay attention to that the starting address of the application must be aligned (128byte alignment is required in S32K3xx), otherwise it will not run normally, and there is a 64-byte App header before the App code. So the user can set the linkfile of the App in this way, the App Header can set 0x005040C0 as the starting address, and after 64byte, the starting address of the App Code is 0x00504100. The compiled bin file starts with the App Header and needs to be written to FLASH Address 0x005040C0.

# 4. Enable Secure Boot

## 4.1 Set `BOOT_SEQ`

To enable the secure boot, as Figure shows, the bit filed `BOOT_SEQ` of the BCW in the IVT must be set to “1”, to change from the default startup flow to the secure startup flow.

![](https://raw.githubusercontent.com/carloscn/images/main/typora202407020917173.png)

The data field BOOT_SEQ, effects on the boot sequence flow:
* When 0: releases the host from reset, then runs the HSE firmware
* When 1: keeps the host on reset and runs the HSE firmware firstly; must be set to run a pre-boot verification

## 4.2 Update IVT

The IVT content stored in Flash needs to be updated, it is recommended to having a backup IVT to ensure that the program can be running in case of data corruption of another IVT.

The IVT start address can be selected among one of the values provided in the below Table 15. At reset, the HSE searches for the first valid IVT header tag starting from the lowest address.

![](https://raw.githubusercontent.com/carloscn/images/main/typora202407020925107.png)

This step can be executed after the secure boot configuration is complete and successful, or before installing the SMRs, to protect the IVT by the SMR like any other memory region.

The IVT can protect the IVT content to against unauthorized changes based on the service  “**BOOT_DATA_SIGN**”, which works like the BSB mode. The authentication tag is computed and appended to the end of the IVT. To enable IVT authentication, the one-time programmable HSE system attribute IVT_AUTH must be set to 1. 

# 5. Secure Boot on AB swap

## 5.1 Partition swapping service

The OTA feature will be enabled if the user has installed the HSE firmware AB_SWAP. This operation is irreversible, that is, the device that has installed AB SWAP firmware can no longer install full_mem firmware, and vice versa.

The FLASH space of S32K3xx will be equally divided into two partitions, active and passive, which can be switched by executing service “**HSE_SRV_ID_ACTIVATE_PASSIVE_BLOCK**”, after reset, the active and passive partitions will be swapped.

The user can program the passive partition by the S32K3 FLASH controller or debugger like the device with full_mem installed, but the passive partition cannot execute the program code.

## 5.2 DCM status registers (switching partition)

The host (application) can read the DCM status register (DCMSTAT) to identify which partition is active and which partition is passive, refer to the below table

![](https://raw.githubusercontent.com/carloscn/images/main/typora202407020930766.png)

## 5.3 Implement Secure Boot

To implement secure boot on the OTA enabled device, the users need to know that the SMR table is uniquely stored in the S32K3 HSE secure NVM, and the area protected by each SMR and its auth-tag will also point to a unique address.

Therefore, it is recommended to store the auth-tag in the same fixed address regardless of the active or passive partition, otherwise you need to reinstall the SMR to avoid secure boot failure.

![](https://raw.githubusercontent.com/carloscn/images/main/typora202407020931028.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora202407020932122.png)

The Figure above shows how to implement secure boot on the OTA enabled device in the demo:

1. After downloading the demo program code (`Cfg_v1` and `App_v1`) on the active partition, the active region is “low address (block 0, 1)”, the S32K3xx will run the secure boot configuration program.
2. Detect that no valid program exists in the passive partition, copy the same data as the active partition to the passive partition, which is `Cfg_v1` and `App_v1` program code.
3. Install the SMR to configure secure boot, write the `AuthTag_v2` of the passive partition to the fixed location before `App_v2`.
4. Perform AB swap, after functional reset, the active region is “high address (block 2, 3)”, run `Cfg_v2`.
5. Like `Cfg_v1` did in step 3, `Cfg_v2` will compute the authentication tag of `App_v1` in the passive area and write it into the `AuthTag_v1` area.
6. After the authentication tags on both sides have been written, modify the `BOOT_SEQ` to “1” in the active (block 0) and passive (block 2) IVT at the same time to enable secure boot flow. Since the start address in the SMR has been set to the App code address, the device will run the App program directly instead of the Cfg program.
7. Perform AB swap, after functional reset, the active region is “low address (block 0, 1)”, run `App_v1`.