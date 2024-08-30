# [Embedded] ZYNQ-使用petalinux建立启动引导项（QSPI/JTAG/SD/TFTP）

早在2018年我们使用ZYNQ使用SD卡制作了Linaro的启动镜像，参考：[ZYNQ的Linux Linaro系统镜像制作SD卡启动](https://github.com/carloscn/blog/issues/39)  但是我们是自己收集的Image的素材，例如：

-  vivado编译生成PL：bit文件
-  sdk生成的PS：fsbl文件
-  uboot生成的：uboot.elf （自行编译）
-  linux生成的: uimage （自行编译）

Note，没有启动ATF，所以ATF层级引导在上面的例子中不存在。

这种方式属于传统的方式，但是petalinux工程给了我们一个更为友善和方便的方法，我们只需要使用petalinux的工程（包含linux、uboot、atf、根文件系统、下载工具等等）就可以实现上面我们自行编译的工作，而且还支持JTAG、SD、QEMU等多种方式启动。

# petalinux安装

对于ubuntu的发行版，严格按照petalinux文档要求进行，新的发行版没有赛灵思的测试，会出现各种问题，不要在环境上纠缠和浪费时间。对于petalinux安装和后面建立的工程，建议预留至少50G的空间，可使用`df -h`查看所剩空间。

下载petalinux的安装工具，在使用petalinux安装工具之前，至少你的ubuntu环境需要安装以下工具集：

`sudo apt-get install tofrodos iproute2 gawk xvfb gcc git make net-tools ncurses libncurses5-dev tftpd zlib1g-dev libssl-dev flex bison libselinux1 gnupg wget diffstat chrpath socat xterm autoconf libtool tar unzip texinfo gcc-multilib build-essential libsdl1.2-dev libglib2.0-dev libssl-dev screen pax gzip`

根据petalinux的手册，petalinux的安装最好使用bash，因此使用zsh或者其他csh的用户请切换回bash环境： `bash` 

安装包需要755的执行权限：`chmod 755 ./petalinux-v2021.2-final-installer.run`

注意，不要root权限安装，否则编译出来的文件将都会是root属性的文件，这就会导致，boot.bin文件无法被启动。([ZYNQ的Linux Linaro系统镜像制作SD卡启动](https://github.com/carloscn/blog/issues/39) 参见2.4节)

> Note: You can not install PetaLinux with root user. If you try to run PetaLinux as root, you may get a bitbake sanity check failure that prevents the build from continuing. This check is done because it is very risky to run builds as root; if any build script mistakenly tries to install files to the root path (/) instead of where it is supposed to, it must be made to fail immediately and not (in the worst case) overwrite files critical to your Linux system's operation, which means in /bin or /etc. Thus, running the build as root is not supported. The only time root access is needed is (completely outside of a build) when the runqemu script uses sudo to set up TAP devices for networking.

`mkdir -p ~/opt/petalinux`

`./petalinux-v2021.2-final-installer.run --log petalinux_installer.log --dir ~/opt/petalinux`

进入petalinux的安装目录`cd ~/opt/petalinux`，设置petalinux的环境变量： `source settings.sh` （注意，**当你的bash终端重启或者启用一个新的终端的时候，请不要忘记source这个环境变量**）

测试：`echo $PETALINUX`

# 创建与编译工程

下载赛灵思官网给的BSP包（是一个bsp扩展名结尾的文件，本质是一个gzip格式的文件），bsp包是和petalinux安装包下载同一个网址。

使用petalinux工具集中的工具解压工程：`petalinux-create -t project -s <path-to-bsp>`

在bsp包内默认含有xsa配置文件，使用该文件配置kconfig文件： `petalinux-config --get-hw-description ./hardware/xilinx-zcu111-2021.2/outputs/project_1.xsa`

修改MACHINE_NAME为你的demo板子的名字：

>Ensure DTG Settings > (template) MACHINE_NAME is selected and change the template to any of the below mentioned possible values.

我的是`zcu111-reva`

工程编译：`petalinux-build` ，注意，在编译的过程，编译脚本会自己解决依赖问题，因此花费时间十分长，根据网络情况和机器性能，请预留出1~4小时的时间在build上面。

# 打包

## 生成boot.bin文件

The boot image can be put into Flash or SD card. When you power on the board, it can boot from the boot image. A boot image usually contains：

* first stage boot loader image, 
* FPGA bitstream, 
* PMU firmware, 
* TF-A, 
* U-Boot.

```bash
petalinux-package --boot --u-boot
```

This generates BOOT.BIN, BOOT_bh.bin, 

package工具的具体使用： https://docs.xilinx.com/r/en-US/ug1144-petalinux-tools-reference-guide/petalinux-package-boot

## gen mcs image

An MCS image for Zynq UltraScale+ MPSoC usually contains :

* First Stage Boot Loader Image(FSBL), 
* FPGA bitstream, 
* PMU firmware, 
* Arm® trusted firmware, 
* U-Boot, 
* DTB
* Kernel Fit image (optional).

这里有一个一键生成的方式，把所需要的所有的在petalinux工程材料全部收集打包：

```
petalinux-package --boot --u-boot
```

还有一个MCS格式的image的uboot：

```
petalinux-package --boot --u-boot --format MCS
```

This generates boot.mcs in `<plnx-proj-root>/images/linux` directory.

使用预先编译的文件打包：Execute the following command to generate the MCS image to boot up to U-Boot using prebuilt images:

```
petalinux-package --boot --u-boot pre-built/linux/images/u-boot.elf --dtb pre-built/linux/images/system.dtb --pmufw pre-built/linux/images/pmufw.elf --fsbl pre-built/linux/images/zynqmp_fsbl.elf --atf pre-built/linux/images/bl31.elf  --format MCS
```

Execute the following command to generate the MCS image to boot up to Linux using build images:

```
$ petalinux-package --boot --u-boot --kernel --offset 0xF40000 --format MCS
```

Execute the following command to generate the MCS image to boot up to Linux using prebuilt images:

```
petalinux-package --boot --u-boot pre-built/linux/images/u-boot.elf --dtb pre-built/linux/images/system.dtb --pmufw pre-built/linux/images/pmufw.elf --fsbl pre-built/linux/images/zynqmp_fsbl.elf --atf pre-built/linux/images/bl31.elf --kernel pre-built/linux/images/image.ub --offset 0xF40000 --boot-script pre-built/linux/images/boot.scr --format MCS
```

This generates boot.mcs in the images/linux directory containing images to boot up to Linux using Fit image(with tiny root file system if `switch_root` enabled) loaded at 0xF40000 Flash offset.

This section describes how to package newly built images into a prebuilt directory. This step is typically done when you want to distribute your project as a BSP to other users. 需要建立在已经build完boot基础上。

## 打包已经编译过的 Image

This section describes how to package newly built images into a prebuilt directory. This step is typically done when you want to distribute your project as a BSP to other users.

```
$ petalinux-package --prebuilt --fpga <FPGA bitstream>
```

想要去发包做bsp工程的时候才去使用，开发阶段不会用到。

BSP包在team内部使用或者给客户使用。进入到工程目录外面  `petalinux-package --bsp -p <plnx-proj-root> --output MY.BSP`

这个地方用到的时候再去实验：https://docs.xilinx.com/r/en-US/ug1144-petalinux-tools-reference-guide/Packaging-BSP 

# Booting PetaLinux Prebuilt Images

You can boot PetaLinux image using the `petalinux-boot` command. 
* `--qemu` option for software emulation (QEMU)
* `--jtag` option to boot on hardware. 

## Boot Levels for Prebuilt Option

`--prebuilt <BOOT_LEVEL>` boots prebuilt images (override all settings). Supported boot levels are 1 to 3. 

The command for JTAG boot:
```
petalinux-boot --jtag --prebuilt <BOOT_LEVEL> --hw_server-url hostname:3121
```

The command for the QEMU boot is as follows:
```
petalinux-boot --qemu --prebuilt <BOOT_LEVEL>
```

注意这个level的解释：
1. Level 1: Download the prebuilt FPGA bitstream.It boots **FSBL and PMU firmware** for Zynq® UltraScale+™ MPSoC.
2. Level 2: Download the prebuilt FPGA bitstream and boot the prebuilt U-Boot. It boots PMU firmware, FSBL, and TF-A before booting U-Boot.
3. Level 3: Downloads PMU firmware, prebuilt FSBL, prebuilt kernel, prebuilt FPGA bitstream, linux-boot.elf, DTB, and the prebuilt TF-A on target.

## Booting PetaLinux Image on QEMU

This section describes how to boot a PetaLinux image under software emulation (QEMU) environment.
	
`petalinux-boot --qemu --prebuilt 3`
	
uboot image:
```
petalinux-boot --qemu --u-boot/--uboot <specify custom u-boot.elf path>
```

kernel image:
```
petalinux-boot --qemu --kernel <specify custom Image path>
```

除此之外还能制定dtb
```
$ petalinux-boot --qemu --kernel <specify custom kernel path> --dtb <specify custom dtb path>
```

注意，QEMU不支持 level 1的启动。

### PetaLinux Image on Hardware with JTAG

硬件启动模式拨码开关（注意，ON是0，OFF是1）：
![](https://raw.githubusercontent.com/carloscn/images/main/typora20221105154231.png)

> The JTAG boot communicates with XSDB which in turn communicates with the hw_server. The TCP port used is 3121; ensure that the firewall is disabled for this port. 

使用JTAG的方式启动需要一个叫做hw_server的服务程序，运行在接JTAG的主机上面。hw_server在petalinux安装文件的`~/opt/petalinux/tools/xsct/bin/hw_server`的监听程序，启动该程序`./hw_server`，接着根据log输出获取到`hw_server-url`的参数。

可以通过远程的`xsct`来使用JTAG调试，可以参考：
https://www.hackster.io/caglayandokme/jtag-booting-an-embedded-linux-image-on-zynq-ultrascale-a0462e

**在硬件上需要确保**：
* JTAG-UART 的microusb接口已经接入到了主机上，并且通过lsusb可以看到驱动已经生效；
* 启动模式拨到JTAG的启动模式，ZCU111的demo板子是`0000`为JTAG启动模式；
* 有网线的话建议接入网线（可选）
* 串口打开`sudo minicom -D /dev/ttyUSB1`，根据你自己主机不同的串口号选择，通常zcu111会映射3个端口，可以遍历试试看看哪个是console输出口。

`petalinux-boot --jtag --prebuilt 3 --hw_server-url <hostname:3121>`

我的URL是 `TCP:ubuntu:3121`，因此我的命令是：

`petalinux-boot --jtag --prebuilt 3 --hw_server-url TCP:ubuntu:3121`

JTAG有丰富的调试方法，可以单独启动某个环节，具体请参考：https://docs.xilinx.com/r/2021.2-English/ug1144-petalinux-tools-reference-guide/Additional-Options-for-Booting-with-JTAG 

注意，**JTAG口速率较慢，下载image时间十分长，我这边大概是30分钟能把level3的启动image加载完毕**。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221105161432.png)

可以看到minicom ttyUSB1 已经启动ATF/uboot/内核了。

### PetaLinux Image on Hardware with SD Card

格式化SD卡需要：
https://docs.xilinx.com/r/en-US/ug1144-petalinux-tools-reference-guide/Partitioning-and-Formatting-an-SD-Card

* boot分区，需要是fat32格式；
* rootfs分区，需要是ext4格式；

1.  Copy the following files from `<plnx-proj-root>/images/linux` or `<plnx-proj-root>/pre-built/linux/images/` into the root directory of the first partition, which is in FAT32 format in the SD card:
    -   BOOT.BIN
    -   image.ub
    -   boot.scr
2.  Extract the `rootfs.tar.gz` folder into the ext4 partition of the SD card.
3.  Connect the serial port on the board to your workstation.
4.  Open a console on the workstation and start the preferred serial communication program (For example: kermit, minicom, gtkterm) with the baud rate set to 115200 on that console.
5.  Power off the board.
6.  Set the boot mode of the board to SD boot. Refer to the board documentation for details.
7.  Plug the SD card into the board.
8.  Power on the board.
9.  A boot message displays on the serial console.

### PetaLinux Image on Hardware with TFTPD

#### 配置TFTPD

关于TFTPD的host端的配置和安装可以参考 https://blog.csdn.net/OnlyLove_/article/details/122645876

host端的tftp路径`~/ftpd`，需要进入FPGA工程进行配置：

Follow these steps to configure PetaLinux for TFTP/PXE boot and build the system image:

1.  Change to root directory of your PetaLinux project.
    ```
    $ cd <plnx-proj-root>
    ```
2.  Launch the top level system configuration menu.
    ```
    $ petalinux-config
    ```
3.  Select Image Packaging Configuration.
4.  Select Copy final images to tftpboot and set tftpboot directory. By default, the TFTP directory ID is `yourpath` Ensure this matches the TFTP server setup of your host.
5.  Save configuration settings and build system image as explained in [Build System Image](https://docs.xilinx.com/r/kgfmiE8rjPkdKCj26LruKQ/HJ2BMz2KcVbuqUMLt6iVSA).

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221105160824.png)

`petalinux-build` 运行一下刷新一下配置环境。

#### 使用TFTPD启动

原理是本质利用uboot的TFTPD功能，同样我们也可以利用uboot使用uboot的各种功能。

1.  Power off the board.
2.  Connect the JTAG port on the board to the workstation using a JTAG cable.
3.  Connect the serial port on the board to your workstation.
4.  Connect the Ethernet port on the board to the local network via a network switch.
5.  Ensure that the mode switches are set to JTAG mode. Refer to the board documentation for details.
6.  Power on the board.
7.  Open a console on your workstation and start with preferred serial communication program (for example, kermit, minicom) with the baud rate set to 115200 on that console.
8.  Run the `petalinux-boot` command as follows on your workstation
```
$ petalinux-boot --jtag --prebuilt 2 --hw_server-url <hostname:3121>
```
9. When autoboot starts, hit any key to stop it. The example of a workstation console output for successful U-Boot.
10. Check whether the `TFTP` server IP address is set to the IP Address of the host where the image resides. This can be done using the following command:
	```
    ZynqMP> print serverip
	```
1.  Set the server IP address to the host IP address using the following command:
    ```
    ZynqMP> setenv serverip <HOST IP ADDRESS>
    ```
2.  Set the board IP address using the following command:
    ```
    ZynqMP> setenv ipaddr <BOARD IP ADDRESS>
    ```
3.  Get the pxe boot file using the following command:
    ```
    ZynqMP> pxe get
    ```
4.  Boot the kernel using the following command:
    ```
    ZynqMP> pxe boot
    ```

###  PetaLinux Image on Hardware with QSPI/OSPI

QSPI启动主要是从外部的存储FLASH中读取数据启动，ZYNQ提供了一种方法，原理是利用JTAG启动uboot，接着利用uboot的tftp功能把`BOOT.BIN`，`Image`，`rootfs.cpio.gz.u-boot`，`boot.scr`通过tftpboot功能导入到RAM中，然后通过sf的功能把读入RAM的数据写入到FLASH里面，这样就完成了Flash烧写的功能。

#### 配置FLASH Partition

This sections provides details on how to configure flash partition with PetaLinux menuconfig.

1.  Change into the root directory of your PetaLinux project.
    ```
    $ cd <plnx-proj-root>
    ```
2.  Launch the top level system configuration menu.
    ```
    $ petalinux-config
    ```
3.  Select **Subsystem AUTO Hardware Settings > Flash Settings**.
4.  Select a flash device as the Primary Flash.
5.  Set the name and the size of each partition.

##### 配置boot src

1.  Change into the root directory of your PetaLinux project.
    ```
    $ cd <plnx-proj-root>
    ```
2.  Launch the top level system configuration menu.
    ```
    $ petalinux-config
    ```
3.  Select u-boot Configuration > u-boot script configuration.
4.  In the u-boot script configuration submenu, you have the following options
	![](https://raw.githubusercontent.com/carloscn/images/main/typora20221105171915.png)
	![](https://raw.githubusercontent.com/carloscn/images/main/typora20221105171939.png)
	![](https://raw.githubusercontent.com/carloscn/images/main/typora20221105171958.png)

#### QSPI启动

1.  Set Primary Flash as boot device and set the Boot script configuration to load the images into QSPI/OSPI flash. For more information, see [Configuring Primary Flash Partition](https://docs.xilinx.com/r/kgfmiE8rjPkdKCj26LruKQ/Zzg03yP3sscHkfEZD~Gk4Q) and [Configuring U-Boot Boot Script (boot.scr)](https://docs.xilinx.com/r/kgfmiE8rjPkdKCj26LruKQ/PeCxatMmW04vU0ngPx5bew).
2.  Build the system image. For more information, see [Build System Image](https://docs.xilinx.com/r/kgfmiE8rjPkdKCj26LruKQ/HJ2BMz2KcVbuqUMLt6iVSA).
3.  Boot a PetaLinux Image on hardware with JTAG up to U-boot, see [Booting PetaLinux Image on Hardware with JTAG](https://docs.xilinx.com/r/kgfmiE8rjPkdKCj26LruKQ/tT4pEHSMlszA84lsZkV1rg).
4.  Make sure you have configured TFTP server in host.
5.  To check or update the offsets for kernel, boot.scr, and root file system, see [Configuring U-Boot Boot Script (boot.scr)](https://docs.xilinx.com/r/kgfmiE8rjPkdKCj26LruKQ/PeCxatMmW04vU0ngPx5bew). If they do not match, the process may fail at the U-Boot prompt.
6.  Set the server IP address to the host IP address using the following command at U-Boot prompt.
    ```
    ZynqMP> setenv serverip <HOST IP ADDRESS>
    ```
    1.  Detect Flash Memory.
        ```
        ZynqMP> sf probe 0 0 0
        ```
    2.  Erase Flash Memory.
        ```
        ZynqMP> sf erase 0 <Flash size in hex>
        ```
    3.  Read images onto Memory and write into Flash.
        -   Read BOOT.BIN.
            ```
            ZynqMP> tftpboot 0x80000 BOOT.BIN
            ```
        -   Write BOOT.BIN.
            ```
            ZynqMP> sf write 0x80000 0x0 $filesize
            ```
            Example: `sf write 0x80000 0x0 0x121e7d0`
        -   Read image.ub.
            ```
            ZynqMP> tftpboot 0x80000 image.ub
            ```
        -   Write image.ub.
            ```
            ZynqMP>sf write 0x80000 <Fit Image Flash Offset Address> $filesize
            ```
            Example: `sf write 0x80000 0xF40000 0xF00000`
        -   Read boot.scr
            ```
            ZynqMP> tftpboot 0x80000 boot.scr
            ```
        -   Write boot.scr
            ```
            ZynqMP> sf write 0x80000 <boot.scr Flash Offset Address> $filesize
            ```
            Example: sf write 0x80000 0x03e80000 0x80000
7.  Enable QSPI flash boot mode on board.
8.  Reset the board (booting starts from flash).

关于QSPI的地址问题，我们有必要来解释一下。对于QSPI从Flash启动，我们需要在uboot中把：
* BOOT.BIN
* image.ub
* boot.src

三个文件写到Flash里面才可以，因此就需要我们工程师来手动注意写入的地址是什么。BOOT.BIN内包含了所有的引导程序，内置的uboot需要依赖boot.src来找到image.ub(kernel)的入口地址。他们三个就是这样的关系。至于这三个地址是什么，我们展开来说下：

首先，对于Flash，我们是要有分区配置的：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221105182147.png)

我们可以看到，分区分为boot、bootenv、kernel，只有这三个，而这里面并没有地址偏移量，只是描述了分区大小。因此，三个地址并不是从这个地方找，但我们可以记忆地址，作为erase flash的长度。

###### BOOT.BIN 地址
对于BOOT.BIN：永远都是0地址开始，因此，`ZynqMP> sf write 0x80000 0x0 $filesize`

###### boot.src 地址
对于boot.src的写入地址，这个要在`petalinux-config`中无法找到，我们需要在手册中确认：
![](https://raw.githubusercontent.com/carloscn/images/main/typora20221105182543.png)

默认的，如果采用QSPI的方式，那么地址是0x3E80000，因此，我们需要写入到这个位置，`ZynqMP> sf write 0x80000 0x3e80000 $filesize`

###### image.ub 地址
这个地址是在`petalinux-config`中配置的：
![](https://raw.githubusercontent.com/carloscn/images/main/typora20221105182744.png)

从这个配置中，我们可以看到地址是 0x3F80000，因此，我们需要写入image.ub到 `ZynqMP> sf write 0x80000 0x3f80000 $filesize`

