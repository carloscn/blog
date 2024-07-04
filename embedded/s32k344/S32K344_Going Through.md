I'd like to introduce the S32K3x4 EVB-Q257 Evaluation Board to you. The following images illustrate the board's apparent and onboard functional modules.

![](https://raw.githubusercontent.com/carloscn/images/main/typora202404231429618.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora202404231429186.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora202404231429524.png)

# 1. Hello World

In this chapter, I will illustrate how to set up a Hello World level project in the S32K344 IDE named S32DS, then we will learn to build the project and download the built binary to the onboard memory such as Flash, SD card, or other NVM.

## 1.1 Download and Install IDE

Download and Install S32 Design Studio IDE for S32 Platform by the site of https://www.nxp.com/design/design-center/software/development-software/s32-design-studio-ide/s32-design-studio-for-s32-platform:S32DS-S32PLATFORM?tab=Design_Tools_Tab

Go to Help → S32DS Extensions and Updates from the top menu to open the S32DS Extensions and Updates. Navigate to the S32K3xx development package and install it.

![](https://raw.githubusercontent.com/carloscn/images/main/typora202404231445272.png)

Continue with the installation of the Real-Time Drivers for S32K3xx:

![](https://raw.githubusercontent.com/carloscn/images/main/typora202404231445675.png)

Download and install the `.exe` file of the S32K3 Real-Time Drivers for Cortex-M from the [S32K3 Standard Software Package](https://www.nxp.com/webapp/swlicensing/sso/downloadSoftware.sp?catid=SW32K3-STDSW-D).

## 1.2 Set Up Jumpers

|Default Power-Supply Jumper settings|   |   |
|---|---|---|
|Jumper|State|Notes|
|`J13`|CLOSED|5 V from FS26 SBC|
|`J16`|CLOSED|3.3 V from FS26 SBC|
|`J23`|2-3|3.3 V for the VDD_HV_A domain|
|`J24`|2-3|3.3 V for the VDD_HV_A domain|
|`J25`|CLOSED|Shunts for A current measurement bypassed|
|`J29`|2-3|1 Ohm shunt for B current measurement|
|`J30`|5-6|VDD_HV_A_MCU is routed to VDD_HV_B domain|
|`J31`|CLOSED|Shunts for B current measurement bypassed|
|`J321`|1-2|IOREF and AREF at Arduino routed to VDD_HV_A domain|
|`J360`|5-6|V15 domain powered from external NPN ballast transistor|
|`J37`|2-3|V15 NPN ballast transistor powered from VDD_HV_B domain|
|`J374`|CLOSED|External circuits powered from VDD_HV_A domain|
|`J375`|CLOSED|External circuits powered from VDD_HV_B domain|
|`J410`|1-2|The ADC reference is routed to the VDD_HV_A domain.|
|`J685`|1-2|Select voltage level for FS26 DEBUG pin|
|`J686`|CLOSED|Disabled FS26 watchdog after power-up|

|Default Peripherals Jumper settings|   |   |
|---|---|---|
|Jumper|State|Notes|
|`J57`|1-2|USB to Serial IC powered from 5 V|
|`J60`|2-3|USB to Serial IC Reset pin routed to voltage divider|
|`J61`|CLOSED|Ethernet connected to VDD_HV_B_PERF domain|
|`J62`|CLOSED|Ethernet Connected to 3.3 V|
|`J65`|CLOSED|QSPI flash powered from 3.3 V|
|`J402`|CLOSED|QSPI signals level refers to VDD_HV_B_PERF domain|
|`J390`|CLOSED|LIN1 Transceiver enabled|
|`J674`|2-3|LIN1 Controller mode via INH pin|
|`J678`|CLOSED|LIN2 Transceiver enabled|
|`J679`|2-3|LIN2 Controller mode via INH pin|
|`J413`|CLOSED|CAN0 Transceiver enabled|
|`J672`|CLOSED|CAN1 Transceiver enabled|

## 1.3 Plug in the Power Supply

Switch `SW10` to the OFF position (fully to the right).

![](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202404231450679.png)

Connect the 12 V power supply adapter and then switch `SW10` to the ON position (fully to the left).

![](https://raw.githubusercontent.com/carloscn/images/main/typora202404231450547.png)

This power-up procedure manage that SBC starts with the disabled watchdog. Four orange LEDs indicate active 12, 5, 3.3 and 1.5V power supply domains.

Connect a micro-USB cable to the `J55` connector to debug via the on-board S32K3 debugger.

![](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202404231451307.png)

##  1.4 Build

Let's take your S32K3X4EVB-Q257 Evaluation Board for a test drive.

Create a S32DS Project from Example: 

Open S32DS and from the menu, go to: File → New → S32DS Project from Example. Select one of RTD example codes. You may choose between examples with high-level API or with low-level API. For example: `Port_example_K344`.


# 2. Memory Map

S32K3 family devices memory features can be found in the following table:

![](https://raw.githubusercontent.com/carloscn/images/main/typora202404231459611.png)

The flash memory on S32K3 devices is integrated by blocks. There are five blocks as maximum and two blocks as minimum. Detailed information is provided in the following table.

![](https://raw.githubusercontent.com/carloscn/images/main/typora202404231501896.png)

The S32K3 Product Family devices can be integrated from 1 SRAM and up to 3 SRAM blocks.

![](https://raw.githubusercontent.com/carloscn/images/main/typora202404231502058.png)


# 3. Boot Sequence


![](https://raw.githubusercontent.com/carloscn/images/main/typora202404231557405.png)


# Reference

* https://www.nxp.com/products/no-longer-manufactured/s32k3x4-q257-full-featured-general-purpose-development-board:S32K3X4EVB-Q257
* https://www.nxp.com/document/guide/getting-started-with-the-s32k3x4evb-q257-evaluation-board:GS-S32K3X4EVB-Q257?section=out-of-the-box
* https://www.nxp.com/products/processors-and-microcontrollers/s32-automotive-platform/s32k-auto-general-purpose-mcus/s32k3-microcontrollers-for-automotive-general-purpose:S32K3
* https://www.nxp.com.cn/webapp/Download?colCode=11_S32K3XX_BOOT_AND_RESET_SEQUENCE_TRAINING&lang_cd=zh