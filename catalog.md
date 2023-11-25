.
├── android
│   └── 01_Android_搭建Kernel开发环境.md
├── arm_m
│   ├── 01_ARMv7-M_处理器架构技术综述.md
│   ├── 02_ARMv7-M_编程模型与模式.md
│   ├── 03_ARMv7-M_存储系统结构.md
│   └── 04_ARMv7-M_异常处理及中断处理.md
├── arm_v7
│   └── PLACEHOLDER.md
├── arm_v8
│   └── 03_ARMv8_指令集介绍_加载指令集和存储指令集.md
├── arm_v8m
│   ├── 01_ARMv8M_Trustzone_技术综述.md
│   ├── 02_ARMv8M_Trustzone_Features.md
│   └── 03_ARMv8M_Trustzone_stm32l5_implement_and_demos.md
├── bin_utils
│   ├── 01_ELF文件_目标文件格式.md
│   ├── 02_ELF文件结构_浅析内部文件结构.md
│   ├── 03_ELF文件_静态链接.md
│   ├── 04_进程虚拟地址空间.md
│   ├── 05_ELF文件_动态链接.md
│   ├── 06_Linux的动态共享库.md
│   ├── 07_ELF文件_堆和栈调用惯例以ARMv8为例.md
│   ├── 08_ELF文件_运行库（入口、库、多线程）.md
│   ├── 09_ELF文件_基于ARMv7的Linux系统调用原理.md
│   └── 10_ELF文件_ARM的镜像文件(.bin-.hex-.s19).md
├── cortex_m_os
│   ├── 01_ARMv7m_Using_The_RUST_Cross_Compiler.md
│   └── 02_ARMv7m_Access_Register.md
├── design
│   ├── app
│   │   └── Secure Boot的一些调研.md
│   ├── embedded
│   │   └── [ZYNQ] Decrypting Partition by the Decrypt Agent Using PUF key.md
│   └── qt
│       └── Module of the Embedded Boards initialization.md
├── dsp
│   ├── DSP-F2812的串行通信接口SCI.md
│   ├── DSP-F2812的模数转换器ADC.md
│   ├── DSP-F2812的时钟和系统控制.md
│   ├── DSP-F2812的事件管理器EV.md
│   ├── DSP-F2812的通用输入输出多路复用器GPIO.md
│   ├── DSP-F2812的中断系统.md
│   ├── DSP-F2812的CMD文件.md
│   └── DSP-F2812的CPU定时器.md
├── embedded
│   ├── (Embedded) cross-compile the cryptsetup on Xilinx ZYNQ aarch64 platform.md
│   ├── (Embedded) enabling the cryptsetup on ramdisk.md
│   ├── (Embedded) NXP-imx6 initialization.md
│   ├── (Embedded) NXP-IMX-RT1170 hard&soft platform.md
│   ├── [Embedded] x86-UEFI-Secure-Boot.md
│   ├── [Embedded] ZYNQ-使用petalinux建立启动引导项.md
│   ├── [Embedded] ZYNQ-Secure Boot Flow.md
│   ├── [Embedded] ZYNQ-Secure Storage.md
│   ├── [Embedded] ZYNQ-UltraScale+的启动流程.md
│   └── ls104x
│       ├── app
│       │   ├── (LS104x) 使用FIT的kernel格式和initramfs.md
│       │   ├── (LS104x) 使用ostree更新rootfs.md
│       │   ├── (LS104x) ostree的部署.md
│       │   ├── (LS104x) ostree的移植.md
│       │   └── (LS104x) ostree制作sd卡.md
│       └── board
│           ├── [Embedded] NXP-LS1046的启动流程.md
│           ├── [embedded] NXP-LS1046的image操作.md
│           ├── [Embedded] NXP-LS1046 secure boot.md
│           └── (LS104x) Hardware and SDK Building.md
├── english
│   ├── 学习导向.md
│   └── grammar
│       └── 02_语法_名词（Nouns）.md
├── linux_kernel
│   ├── 0x21_LinuxKernel_内核活动（一）之系统调用.md
│   ├── 0x22_LinuxKernel_内核活动（二）中断体系结构（中断上文）.md
│   ├── 0x23_LinuxKernel_内核活动（三）中断体系结构（中断下文）.md
│   ├── 0x24_LinuxKernel_进程（一）进程的管理（生命周期、进程表示）.md
│   ├── 0x26_LinuxKernel_设备驱动（一）综述与文件系统关联.md
│   ├── 0x30_LinuxKernel_设备驱动（五）模块.md
│   ├── 0x31_LinuxKernel_内存管理（一）物理页面、伙伴系统和slab分配器.md
│   ├── 0x32_LinuxKernel_内存管理（二）虚拟内存管理、缺页与调试工具.md
│   └── 0x33_LinuxKernel_同步管理_原子操作_内存屏障_锁机制等.md
├── linux_user
│   ├── Linux System - Managing Linux Services - initramfs (using imx6 + zynq).md
│   └── Linux System - Managing Linux Services - inittab + init.d.md
├── make
│   └── cmake
│       └── PLACEHOLDER.md
├── makefile
│   └── 01_Script_makefile_summary.md
├── optee_os
│   ├── 01_OPTEE-OS_基础之（一）功能综述、简要介绍.md
│   ├── 02_OPTEE-OS_基础之（二）TrustZone和ATF功能综述、简要介绍.md
│   ├── 03_OPTEE-OS_系统集成之（一）编译、实例、在QEMU上执行.md
│   ├── 04_OPTEE-OS_系统集成之（二）基于QEMU的OPTEE启动过程.md
│   ├── 05_OPTEE-OS_系统集成之（三）ATF启动过程.md
│   ├── 06_OPTEE-OS_系统集成之（四）OPTEE镜像启动过程.md
│   ├── 07_OPTEE-OS_系统集成之（五）REE侧上层软件.md
│   ├── 08_OPTEE-OS_系统集成之（六）TEE的驱动.md
│   ├── 09_OPTEE-OS_内核之（一）ARM核安全态和非安全态的切换.md
│   ├── 10_OPTEE-OS_内核之（二）对安全监控模式的调用的处理.md
│   ├── 11_OPTEE-OS_内核之（三）中断与异常的处理.md
│   ├── 12_OPTEE-OS_内核之（四）对TA请求的处理.md
│   ├── 14_OPTEE-OS_内核之（六）线程管理与并发.md
│   ├── 15_OPTEE-OS_内核之（七）系统调用及IPC机制.md
│   ├── 16_OPTEE-OS_应用之（一）TA镜像的签名和加载.md
│   └── 18_OPTEE-OS_应用之（三）可信应用的开发.md
├── README
├── rtos
│   └── PLACEHOLDER.md
├── rust_sys
│   └── 02_SYS_RUST_文件IO.md
├── security
│   ├── 2.0_Security_随机数（伪随机数）.md
│   ├── 3.0_Security_对称密钥算法加解密.md
│   ├── 3.1_Security_对称密钥算法之AES.md
│   ├── 3.2_Security_对称密钥算法之MAC（CMAC + HMAC ）.md
│   ├── 3.3_Security_对称密钥算法之AEAD.md
│   ├── 8.0_Security_pkcs7(CMS)_embedded.md
│   └── 9.0_Security_pkcs11(HSM)_embedded.md
└── vehicle
    └── Uptane.md

28 directories, 93 files
