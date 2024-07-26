After (POR) the chip reset, the non-changeable code in the BAF (Boot Assist Firmware), aka BootROM, will be executed by a Cortex-M0 core inside the HSE, instead of the CM7 core in the chip.

One of the primary tasks of the BAF is to determine whether the chip is in normal boot or booting from the low-cost standby mode. In the normal boot mode, the chip will parse the IVT (Image Vector Table), which is located before the interrupt vector table in the FLASH. The primary role of IVT is to configure the key information for the entire project and the kernel. However, when booting from standby, both the normal wakup or the fast weakup are supported.

After parsing the IVT fields, the system determines whether the secure boot is supported. In the secure boot mode, the concept of the life cycle is fundamental. The Life cycle acts as an irreversible security state machine designed for information security control and key management, influencing the chip's status. Moreover, the Life Cycle cannot be revoked. If the secure boot verification passes, the CM7 core is booted and retrieves the kernel start address from the IVT. The CM7 core then jumps to this address from the IVT and begins execution. Meanwhile, the M0 core in the HSE switches to a low-power mode.

![](https://raw.githubusercontent.com/carloscn/images/main/typora202407051433738.png)

# IVT Parsing

The BAF mentioned above parses the IVT fields and retrieves the key configuration of the application project and the kernel. The IVT is 256 bytes in total and must be placed at the beginning of either PFlash or DFlash. The S32K344 chip has a total of four PFlash blocks and one DFlash block, providing five possible addresses for the IVT. The BAF will sequentially search these addresses block by block until it finds the IVT, which point it will stop searching.

![](https://raw.githubusercontent.com/carloscn/images/main/typora202407051559053.png)

# Boot Configuration Word

As mentioned above, the offset 04H in an IVT is the Boot Configuration Word. The detailed configurable fields are shown in the following table:

* Bit3: `Boot_SEQ` indicates whether secure boot is enabled.
* Bit5:  Indicates whether the watchdog in the CM7 core is enabled.
* Bit1-Bit0: Indicates whether the two CM7 cores are enabled. Even if both the cores are enabled, the main core must be booted first. After the main core's clock is configured, the slave core then booted by the main core. The enabled core in the IVT must be the main core and the bootloader must also execute on the main core.

![](https://raw.githubusercontent.com/carloscn/images/main/typora202407051614130.png)


# Requirments for RTD project

After finishing the BAF execution, control is handed over to the program entry `Reset_Handler`in the file `startup_cm7.s` , which contains the chip's entire booting program in ARM assembly lauguage. The `startup_cm7.s` file will call functions in start.c and system.c during its execution. The program handles many of the chip's mechanisms,  such as the data initialization, global variable initialization, and the initialization of the MPU, Cache, FPU, watchdog.

![](https://raw.githubusercontent.com/carloscn/images/main/typora202407051627536.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora202407051630327.png)

Another important file in the startup is the linker file. The linker file defines various sections and segments, storing addresses for local variables and space usage during initialization. It also provides the actual memory map of the target MCU, ensuring that the built binary is correctly mapped to the MCU's memory.

In NXP demo projects, there are generally two linker files: one for Flash and one for RAM. The RAM linker file is used exclusively for temporary debugging. The compiled code is executed from RAM, but all data is lost upon restart.

![](https://raw.githubusercontent.com/carloscn/images/main/typora202407051719278.png)

# RTD Booting

startup_cm7.s:

![](https://raw.githubusercontent.com/carloscn/images/main/typora202407051719500.png)


# Reference

* [带你深入了解NXP汽车通用控制器S32K3 (一)：概述篇](https://www.arrowsolution.com.cn/web/knowledge/article/62)
* [连载 | 带你深入了解NXP汽车通用控制器S32K3 (二)：开发环境搭建篇](https://www.arrowsolution.com.cn/web/knowledge/article/68)
* [连载 | 带你深入了解NXP汽车通用控制器S32K3 (五)：启动详解篇](https://www.arrowsolution.com.cn/web/knowledge/article/78)
* [连载 | 带你深入了解NXP汽车通用控制器S32K3 (七)：Bootloader原理篇](https://www.arrowsolution.com.cn/web/knowledge/article/83)
* [连载 | 带你深入了解NXP汽车通用控制器S32K3 (九)：用S32DS 开发 EB 配置Mcal的工程](https://www.arrowsolution.com.cn/web/knowledge/article/97)