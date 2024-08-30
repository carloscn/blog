# 1. Hardware

## 1.1 Components layout

The reference board quick guide is shown in the link: [https://www.nxp.com.cn/docs/en/quick-reference-guide/MIMXRT1170-EVK-QSG.pdf](https://www.nxp.com.cn/docs/en/quick-reference-guide/MIMXRT1170-EVK-QSG.pdf "https://www.nxp.com.cn/docs/en/quick-reference-guide/MIMXRT1170-EVK-QSG.pdf") and https://www.nxp.com/document/guide/getting-started-with-the-i-mx-rt1170-evaluation-kit:GS-MIMXRT1170-EVK#title3_1

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20230520100424.png" width="85%" /></div>

Plug the power adapter wire into the MIMXRT1170-EVK board 5V DC IN header (`J43`) and switch on 5V DC IN (`SW5`).

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20230520100449.png" width="85%" /></div>

**As an embedded developer, you may pay more attention to the following:**

*  Serial port: the `Debug USB port` will map the onboard linker service and serial port `/dev/ttyACM0`.
* Debug port:
    -   Onboard linker service (DAP-Link)
    -   External J-Link.

The OpenSDA circuit (CMSIS–DAP) is an open-standard serial and debug adapter. It bridges serial and debug communications between a USB host and an embedded target processor

CMSIS-DAP features a Mass Storage Device (MSD) bootloader, which provides a quick and easy mechanism for loading different CMSIS-DAP Applications such as flash programmers, run-control debug interfaces, serial-to-USB converters, etc. Two or more CMSIS-DAP applications can run simultaneously. For example, run-control debug application and serial-to-USB converter run in parallel to provide a virtual COM communication interface while allowing code debugging via CMSIS-DAP with just a single USB connection.

For the IMXRT1170 EVK board, J11 is the connector between the USB host and the target processor. Jumper to serial downloader mode uses the stable DAP-Link debugger function. If developer wants to make OpenSDA going to the bootloader mode, jumper J27 to 1-2 and press SW3 when power on. Meanwhile, the OpenSDA supports drag/drop feature for U-Disk.

# 2. Software

[i.MX](http://i.MX "http://i.MX") RT1170 crossover MCUs are part of the EdgeVerse™ [edge computing](https://www.nxp.com/applications/enabling-technologies/edge-computing:EDGE-COMPUTING "https://www.nxp.com/applications/enabling-technologies/edge-computing:EDGE-COMPUTING") platform and are setting speed records at 1 GHz. The dual core [i.MX](http://i.MX "http://i.MX") RT1170 MCU runs on the Arm® Cortex®-M7 core at 1 GHz and Arm Cortex-M4 at 400 MHz. For more information, please refer to https://www.nxp.com/products/processors-and-microcontrollers/arm-microcontrollers/i-mx-rt-crossover-mcus/i-mx-rt1170-first-ghz-crossover-mcu-with-arm-cortex-m7-and-cortex-m4-cores:i.MX-RT1170.

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20230520100606.png" width="85%" /></div>

## 2.1 System Booting

### 2.1.1 Boot overview

The boot process begins at the Power-On Reset (POR) where the hardware reset logic forces the ARM core to begin the execution starting from the on-chip boot ROM. The boot ROM uses the state of the **BOOT_MODE** **register** and **eFUSEs** to determine the boot device. The boot ROM code also allows to download the programs to be run on the device. The example is a provisioning program that can make further use of the serial connection to provide a boot device with a new image.

**Device Configuration Data (DCD)**. DCD feature allows the boot ROM code to obtain the SOC configuration data from an external program image residing on the boot device. As an example, the DCD can be used to program the SDRAM controller (SEMC) for optimal settings, improving the boot performance. The DCD is restricted to the memory areas and peripheral addresses that are considered essential for the boot purposes.

The imx-rt1160 bootflow is shown in the following figure:

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20230520100645.png" width="40%" /></div>

#### Boot ROM stage

The figure found here show the high-level boot ROM code flow:

![](https://raw.githubusercontent.com/carloscn/images/main/typora20230520101307.png)

**The mainly features of the Boot Rom include:**

-   Support for booting from various boot device
-   Serial downloader support (USB OTG and UART)
-   Device Configuration Data (DCD) and plugin
-   Digital signature and encryption based High-Assurance Boot (HAB)
-   Wake-up from the low-power modes
-   Encrypted eXecute In Place (XIP) on Serial NOR via FlexSPI interface powered by Bus  
    Encryption Engine (BEE)
-   Encrypted boot on devices except the Serial NOR by Data Co-Processor (DCP) controller  
    The Boot Rom supports these boot devices:
    -   Serial NOR Flash via FlexSPI
    -   Serial NAND Flash via FlexSPI
    -   Parallel NOR Flash via Smart External Memory Controller (SEMC)
    -   RAWNAND Flash via SEMC
    -   SD/MMC
    -   SPI NOR/EEPROM

#### Secondary bootable Image stage

##### Overview

There are two types of [i.MX](http://i.MX) MCU bootable image:

- **Normal boot image:** This type of image can boot directly by boot ROM.
- **Plugin boot image:** This type of image can be used to load a boot image from devices that are not

	natively supported by boot ROM.

Both types of image can be unsigned, signed, and encrypted for different production phases and different
security level requirements:

- **Unsigned Image:** The image does not contain authentication-related data and is used during

	development phase.
- **Signed Image**: The image contains authentication-related data (CSF section) and is used during

	production phase. For more information, please refer to [[Embedded\] NXP-IMX-RT1170 secure boot](https://github.com/carloscn/blog/issues/188)

**Encrypted Image:** The image contains encrypted application data and authentication-related data and
is used during the production phase with higher security requirement.  For more information, please refer to [[Embedded\] NXP-IMX-RT1170 secure boot](https://github.com/carloscn/blog/issues/188).

##### Boot Options

The imx RT family supports a number of different boot sources and includes the option for memory to be copied to **on-chip** or external destination memory, as well as [Execute in Place (**XIP**)](https://en.wikipedia.org/wiki/Execute_in_place) for some interfaces. For more information, please refer to https://www.nxp.com/docs/en/application-note/AN12107.pdf.

The FlexSPI NOR Boot flow is shown in the following diagram:

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20230520102153.png" width="80%" /></div>

> The on-chip RAM = FlexRAM (512KB + 256KB) + OCRAM(1.25MB)
