# [embedded] NXP-LS1046的image操作

# 1. Walk Thought

## 1.1 boot option

LS1046ARDB supports the following boot options:
-   SD
-   QSPI NOR flash

![](https://raw.githubusercontent.com/carloscn/images/main/typora202211111404162.png)

如果改变默认启动方式需要额外的设置在image中或者RCW中。

## 1.2 QSPI NOR flash banks

LS1046有两个Nor-FLASH通过QSPI接入控制器，一次性只能使用一个。这样设计是的目的是如果一个image损坏之后，还要另一个可以启动，而采取一些挽救性的措施。在uboot中置入了CPLD的工具逻辑，可以通过cpld进行切换。

我们可以把flash 0定义 recovery启动分支，flash 1 为primary启动分支。

NXP通用的手段是，为了保护有一个可以随时启动的镜像，预留flash0的分区不动，而使用flash1分区的镜像进行开发和测试。切换到flash1可以使用cpld的工具来完成。我们可以从uboot的输出log得到确认信息当前是在flash0 还是 flash1：
* flash0： vBANK 0
* flash1： vBANK 1

![](https://raw.githubusercontent.com/carloscn/images/main/typora202211111417052.png)

可以在uboot中使用cpld命令可以切换：

-   Switch to QSPI NOR flash0 (default):
    ```
    =>cpld reset
    ```
-   Switch to QSPI NOR flash1:
    ```
    =>cpld reset altbank 
    ```
-   Switch to SD:
    ```
    =>cpld reset sd 
    ```

## 1.3 recovery information

### 1.3.1 只有flash0 损坏

如果flash0分区损坏了怎么办？ NXP设计了recovery模式来拯救flash0的启动分区镜像。我们可以通过recovery机制，把flash 1的分区镜像恢复到flash 0中。

1.  Download the prebuilt composite firmware image:
    ```
    $ wget http://www.nxp.com/lgfiles/sdk/lsdk2108/firmware_ls1046ardb_qspiboot.img 
    ```
2.  Boot LS1046ARDB from QSPI NOR flash1 with the following switch settings:
    `SW3 = 01001110`, `SW4 = 00111011`, `SW5 = 00100010`
3.  Program QSPI NOR flash0 from QSPI NOR flash1:
    ```
    => tftp $load_addr firmware_ls1046ardb_qspiboot.img
    => sf probe 0:1
    => sf erase 0 +$filesize && sf write $load_addr 0 $filesize
    ```
4.  Reset and boot the board from QSPI NOR flash0:
    ```
    => cpld reset
    ```

### 1.3.2 flash0和flash1全部损坏

如果flash0分区和flash1分区都损坏了，我们则需要使用**CodeWarrior for LS Series, Arm® v8 ISA**这个IDE来恢复板子。参考："8.6 Board Recovery" in [ARM V8 ISA, Targeting Manual](https://www.nxp.com/docs/en/user-guide/CWARMv8TM.pdf?_gl=1*1a2dz9f*_ga*MTQzNzU3OTMzMC4xNjY3NTI4MDAy*_ga_WM5LE0KMSH*MTY2ODE0NjU5MS45LjEuMTY2ODE0Nzc0NC4wLjAuMA..)

## 1.4 burn image

原理上利用uboot的烧录功能完成烧录，烧录image包含两类型：
* 烧录到flash 1
* 烧录到sd卡

烧录到flash 0 请参考recovery information一节。

### 1.4.1 烧录到flash 1

以`firmware_ls1046ardb_qspiboot.img`固件为例：

1. 下载固件：`wget https://www.nxp.com/lgfiles/sdk/lsdk2108/firmware_ls1046ardb_qspiboot.img`
2. power reset板子，让板子从QSPI NOR Flash 0启动，进入FLASH 0的reboot。
3. 使用uboot丰富的功能，比如sd卡，tftp等把固件装在到板子的内存中，再通过uboot的flash写入功能反写数据到flash中。按照以下步骤：
	1. 可以从TFTP加载镜像：`=> tftp $load_addr firmware_ls1046ardb_qspiboot.img`
	2. 可以从SD卡或者U盘读取镜像：`=> load mmc <device:part> $load_addr firmware_ls1046ardb_qspiboot.img`
	3. 可以从SATA读取镜像：`=> load scsi <device:part> $load_addr firmware_ls1046ardb_qspiboot.img`
4. 通过uboot的flash工具把数据写入到flash中：
	1. `sf probe 0:1`
	2. `sf erase 0 +$filesize && sf write $load_addr 0 $filesize`
5. 复位为从flash 1启动：`cpld reset altbank`，系统会自动的从flash 1启动了。

### 1.4.2 烧录到sd卡

以`firmware_ls1046ardb_sdboot.img`固件为例：

1. 下载固件：`wget https://www.nxp.com/lgfiles/sdk/lsdk2108/firmware_ls1046ardb_sdboot.img`
2. power reset板子，让板子从QSPI NOR Flash 0启动，进入FLASH 0的reboot。
3. 使用uboot丰富的功能，比如sd卡，tftp等把固件装在到板子的内存中，再通过uboot的flash写入功能反写数据到flash中。按照以下步骤：
	1. 可以从TFTP加载镜像：`=> tftp $load_addr firmware_ls1046ardb_qspiboot.img`
	2. 可以从SD卡或者U盘读取镜像：`=> load mmc <device:part> $load_addr firmware_ls1046ardb_sdboot.img`
	3. 可以从SATA读取镜像：`=> load scsi <device:part> $load_addr firmware_ls1046ardb_sdboot.img`
4. 通过uboot的mmc工具把数据写入sd卡：`=> mmc dev 0; mmc write $load_addr 8 1f000`
5. 复位为从sd启动：`cpld reset sd`，系统会自动的从sd启动了。

也可以通过host linux来烧录sd卡：

`sudo dd if=firmware_ls1046ardb_uboot_sdboot_1040_5559.img of=/dev/sdb seek=8 bs=512`

对于可启动RAW型数据，建议SD卡分区：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202211141254146.png)

RAM分区作为img文件RAW数据写入（调试用）；
boot分区作为fat32的文件系统，存入引导的相关firmware；
rootfs分区作为ext4的文件系统，存入rootfs；

## 1.5 自动下载并且部署image

NXP提供了`flex-installer`工具来帮助完成image的生成，这个工具可以在host linux上运行，能够直接把输入的images文件打包制作SD启动盘，实现一键制作。

另外，该工具甚至可以在目标板子的uboot环境运行，原理是，uboot使用一种TinyDistro的RAM文件系统，则可以对这个工具提供支持。在uboot环境，就可以直接使用该工具对板子的存储介质数据进行更新。

这个工具我们考虑作为OTA的update agent。

### 1.5.1 离线制作SD启动镜像

![](https://raw.githubusercontent.com/carloscn/images/main/typora202211111511811.png)

假如升级的文件是： `rootfs_lsdk2108_LS_arm64_main.tgz` and `boot_LS_arm64_lts_5.10.tgz`.

1. 连接SD卡到Linux host端，SD卡在Linux上会映射文件节点；
2. 下载 `flex-installer`：`wget https://www.nxp.com/lgfiles/sdk/lsdk2108/flex-installer && chmod +x flex-installer && sudo mv flex-installer /usr/bin`
3. 执行命令，例如`flex-installer -i auto -m ls1046ardb -d <device>`
4. 移除SD卡，插入目标硬件；
5. 设定从SD卡启动，完成制作。

### 1.5.2 在线制作启动镜像

![](https://raw.githubusercontent.com/carloscn/images/main/typora202211111515466.png)

1. 连接SD卡没插入目标板端；
2. 进入uboot，并启动TinyDistro：
	1. For SD boot: `=> run sd_bootcmd`
	2. For QSPI NOR boot: `=> run qspi_bootcmd`
3. 进入TinyDistro以root身份并且配置网络：
	1. 配置IP通过udhcpc方式：`$ udhcpc -i <port name in TinyDistro>`
	2. 通过静态方式：`$ ifconfig <port name in TinyDistro> <IP address> netmask <netmask address> up`
	3. > The port name in Linux TinyDistro corresponding to each of the ports on the reference board chassis is given in section [LS1046ARDB reference information](https://docs.nxp.com/bundle/GUID-487B2E69-BB19-42CB-AC38-7EF18C0FE3AE/page/GUID-4AE0879F-EFD1-46B0-84AF-C156F7BC6973.html#GUID-4AE0879F-EFD1-46B0-84AF-C156F7BC6973
	4. ![](https://raw.githubusercontent.com/carloscn/images/main/typora202211111538410.png)

1. 下载 `flex-installer`：`wget https://www.nxp.com/lgfiles/sdk/lsdk2108/flex-installer && chmod +x flex-installer && sudo mv flex-installer /usr/bin`
2. 执行命令，例如`flex-installer -i auto -m ls1046ardb -d <device>`
3. 结束。

## 1.6 LSDK memory 分布

### 1.6.1 Flash Layout

The following table shows the memory layout of various firmwares stored in **NOR/NAND/QSPI/XSPI** flash device or SD card on all Layerscape Reference Design Boards.

![](https://raw.githubusercontent.com/carloscn/images/main/typora202211111522089.png)

### 1.6.2 Storage layout on SD/USB/SATA for LSDK images deployment

![](https://raw.githubusercontent.com/carloscn/images/main/typora202211111523346.png)

### 1.6.3 LSDK userland

使用`boot`启动默认存储介质的rootfs。

To boot LSDK userland from the specified USB/SD/SATA storage device under U-Boot:
```
=> run bootcmd_usb0
or
=> run bootcmd_mmc1
or 
=> run bootcmd_scsi0
```
To boot TinyLinux under U-Boot:
```
=> run sd_bootcmd
or 
=> run nor_bootcmd
or
=> run qspi_bootcmd
or
=> run xspi_bootcmd
```

NXP默认支持这些rootfs：
![](https://raw.githubusercontent.com/carloscn/images/main/typora202211111526965.png)

如果想build其他的rootfs需要参考：
>If you want to build specific userland from source, refer to the section "How to build LSDK with Flexbuild" -> "How to build various userland with custom packages" for the detailed steps to build various userland.

# 2. Build LSDK

与下载一样，NXP提供了`flex-builder`的工具集，可以帮助我们快速的编译image。后面我们都使用这个工具集进行编译固件。

http://www.nxp.com/lsdk?_gl=1*1prvicj*_ga*MTQzNzU3OTMzMC4xNjY3NTI4MDAy*_ga_WM5LE0KMSH*MTY2ODE1NDg5Mi4xMS4xLjE2NjgxNTQ5NTAuMC4wLjA. 

在Layerscape Software Development Kit中包含。

## 2.1 自动部署build环境

通常用户想要直接部署LSDK的固件和文件系统可以通过`flex-installer`实现。如果用户想要创建一个自定义的image集合，我们可以指定`rcw_<boottype> in <flexbuild_dir>/configs/board/<machine>/manifest`来取代默认的RCW，我们可以修改组件(RCW, U-Boot, ATF, linux, app components)或者使用不同的tag/branch在 configs/sdk.yml中。最后可以运行下面的脚本来自动部署image：

``` bash
- To automatically build all images (generating LSDK composite firmware, Linux kernel, app components, bootpartition, rootfs) for specific machine 
Usage: 
$ flex-builder -m <machine> 
or
$ bld -m <machine> 
The <machine> can be: ls1012aqds, ls1012afrwy, ls1021atwr, ls1028ardb, ls1043ardb, ls1046ardb, ls1046afrwy, ls1088ardb_pb, ls2088ardb, lx2160ardb_rev2, ls2162aqds. 
Example: flex-builder -m ls1046ardb 

- To automatically build all images for all Layerscale machines in <arch> architecture 
Usage: 
$ flex-builder -i auto -a <arch>
The <arch> can be: arm32, arm64 

- To automatically build all images for both arm32 and arm64 all Layerscale machines 
Usage: 
$ flex-builder all 
Optionally, users can change the default build options in <flexbuild-dir>/configs/lsdk.yml to enable/disable some build features. For example, you can enable/disable some app components by setting <component_name>: y or n on demand.
```

## 2.2 LSDK的firmware编译

LSDK的固件包含：
* RCW/PBL
* ATF
* Bootloader（u-boot）
* Secure Header
* Ethernet MAC/PHY 固件
* dtb
* kernel
* tiny initrd RFS

**这些固件可以被编写到flash设备的0x0地址，SD卡的block#8分区**。

```
Usage:
$ flex-builder -i mkfw -m <machine> [-b <boot_type>]

Examples:
$ flex-builder -i mkfw -m ls1043ardb -b sd
  firmware_ls1043ardb_sdboot.img and firmware_ls1043ardb_sdboot_secure.img will be generated.
```

当然我们也可以不all-in，可以分开编译，nxp提供了以下拆分的方法：

### 2.2.1 RCW TF-A/U-Boot/UEFI

该平台支持TF-A。flex-builder能够自动的编译ATF的依赖RCW, U-Boot/UEFI, OPTEE, and CST，最后产生bl2.pbl和fip.bin两个二进制文件。

```
Usage: 
flex-builder -c atf -m <machine> -b <boottype> [-s] 
Example: 
$ flex-builder -c atf -m ls1043ardb -b sd 
  (build uboot-based ATF image for SD boot) 

$ flex-builder -c atf -m ls1046ardb -b qspi -s 
 (build uboot-based ATF image for QSPI-NOR secure boot) 

$ flex-builder -c atf -m lx2160ardb_rev2 -b xspi 
  (build uboot-based ATF image for FlexSPI-NOR boot) 

$ flex-builder -c atf -m ls2088ardb -b nor -B uefi 
 (build uefi-based ATF image for IFC-NOR boot) 
```

需要注意：
1. 如果使用一个自定义的RCW，我们需要重新配置 `rcw_<boottype>`变量在`<flexbuild>/configs/board/<machine>/manifest`,然后重新运行，`flex-builder -i clean-firmware; flex-builder -c atf -m <machine> -b <boottype>` 来产生新的ATF image。ATF的依赖会自动编译。
2. flexbuild的-s参数是给secure boot使用的，`FUSE_PROVISIONING`默认没有被使能，如果需要的话，则在`configs/sdk.yml`文件中配置`CONFIG_FUSE_PROVISIONING`。
3. **flex-builder可以自动下载相关源码**；

### 2.2.2 Linux kernel

To build kernel using the default tree/branch/tag configurations specified in configs/sdk.yml:
```
$ flex-builder -c linux -a arm64 (for all Layerscale arm64 platforms) 
or 
$ bld -c linux

$ flex-builder -c linux -a arm32 (for all Layerscale arm32 platforms)
or
$ bld -c linux -a arm32
```

To build kernel with specified tree/branch/tag and additional fragment config:
```
Usage:  flex-builder -c linux:<kernel-repo>:<branch|tag> [ -a arm64 -B fragment:<custom>.config ]
Example: 
$ flex-builder -c linux:linux:LSDK-21.08 -a arm32 
$ flex-builder -c linux:linux:LSDK-21.08-RT -a arm64
```

参考： https://docs.nxp.com/bundle/GUID-487B2E69-BB19-42CB-AC38-7EF18C0FE3AE/page/GUID-5DC5A745-0B03-4B40-BFF0-CC9D11A712DF.html

### 2.2.3 rootfs

1. 我们可以选择改变内核的源，`<flexbuild_dir>/components/linux/linux` 或者改变默认的分支，然后运行`$ flex-builder -c linux -a arm64`
2. 我们也可以选择改变不使用默认的rootfs，因为rootfs比较大，NXP为我们编译好了 `rootfs_<lsdk_version>_yocto_tiny_arm64.cpio.gz` 可以直接使用。
3. 产生itb：
```
    $ flex-builder -i mkitb -r <distro_type>:<distro_scale> -a <arch> 
    For example: 
    $ flex-builder -i mkitb -r yocto:tiny -a arm64 
    $ flex-builder -i mkitb -r yocto:devel -a arm32 
    $ flex-builder -i mkitb -r ubuntu:lite -a arm64 
    $ flex-builder -i mkitb -r ubuntu:main -a arm64
```
4. 加载itb到板子的RAM启动板子：
```
    => tftp $load_addr <itb_img>
    => bootm $load_addr#<board_name>
    e.g. 
    => tftp $load_addr lsdk2108_yocto_tiny_LS_arm64.itb
    => bootm $load_addr#ls1028ardb
```

### 2.2.4 打包boot分区

一个完整的boot分区应该包含：`kernel image, DTB, distro bootscript, secure boot headers, tiny initrd and so on` flexbuilder可以自动解决这些依赖关系。我们可以产生`boot_LS_arm64_lts_<version>.tgz`这个格式的boot分区。

```bash
$ flex-builder -i mkboot -a arm64 
or 
$ flex-builder -i mkboot -a arm32 
or 
$ flex-builder -i mkboot -a arm64 -t (for secure boot with IMA-EVM)
```

如果我们要是换Linux kernel的话，也可以通过直接替换boot分区的内容对linux kernel进行替换。

>Step1: Optionally, run '`flex-builder -i repo-fetch -B linux`' to download linux source, then modify Linux kernel source code in `<flexbuild>`/components/linux/linux if needed.
>
>Step2: Optionally, customize kernel options in interactive menu by command "`flex-builder -c linux:custom`"
>
>Step3: Build new kernel by command "_flex-builder -c linux_"
>
>Step4: Generate new linux boot tarball by command "_flex-builder -i mkboot_"
>
>The new kernel tarball `<flexbuild>`/build/images/boot_LS_arm64_lts_5.10_`<timestamp>.`tgz will be generated.

## 2.3 APP级的编译

应用层级包含了：
* graphics
* weston
* ros
* dpdk
* vpp
* restool
* opencv
* networking
* **securiry**

```
Usage:  flex-builder -c <component> [ -a <arch> -r <distro_type>:<distro_scale> ]
Example:
$ flex-builder -c apps               
 (build all arm64 apps components against Ubuntu main userland by default)

$ flex-builder -c apps -r yocto:devel 
 (build all arm64 apps components against Yocto-based devel userland)

$ flex-builder -c graphics -r ubuntu:desktop                
 (build all arm64 graphics components against Ubuntu desktop arm64 userland)

$ flex-builder -c weston -r ubuntu:desktop
 (build weston component against Ubuntu desktop arm64 userland)

$ flex-builder -c ros -r ubuntu:desktop -F
 (build ROS against ubuntu desktop arm64 userland)

$ flex-builder -c dpdk
 (build DPDK against Ubuntu main arm64 userland)

$ flex-builder -c ovs_dpdk
$ flex-builder -c pktgen_dpdk
$ flex-builder -c vpp
$ flex-builder -c fmc
$ flex-builder -c restool
$ flex-builder -c tsntool
$ flex-builder -c opencv
$ flex-builder -c onnxruntime

(Note: '-a arm64 -r ubuntu:main' is used by default in case '-a <arch> -r <distro_type>:<distro_scale>' is not specified)
```

这里的安全层级是我们最关心的了，To build security subsystem, including components: openssl, secure_obj, libpkcs11, cst, keyctl_caam, optee_os, optee_client, optee_test.
```
$ flex-builder -c security
```

如果我们编写了这些app，我们可以通过以下的方式快速换出新的APP。

Step1: Clean the old apps images as below
```
$ flex-builder clean-apps
$ make clean -C compoments/apps/<subsystem>/<component_name> 
 (optional, only for some components which support 'make clean')
```
Step2: Modify component source code in directory `components/apps/<subsystem>/<component-name>` according to demand (optional)
Step3: Build the component and generate the compressed app component tarball
```
$ flex_builder -c <component-name> [ -r <distro_type>:<distro_scale> -a <arch> ]
 (or run 'flex_builder -c apps' to build all components)
$ flex-builder -i packapps [ -r <distro_type>:<distro_scale> -a <arch> ]
(Note: "-r ubuntu:main -a arm64" is used by default if unspecified)
```
Step4: Log in LSDK Linux system on target board, download app_components_LS_arm64.tgz and replace the existing app component as below
```
root@localhost:/# wget <webserver_path>/app_components_arm64_ubuntu_main.tgz (or by scp command)
root@localhost:~# tar xfm app_components_arm64_ubuntu_main.tgz -C /
root@localhost:~# reboot
```

## 2.4 制作userland包
参考：
https://docs.nxp.com/bundle/GUID-487B2E69-BB19-42CB-AC38-7EF18C0FE3AE/page/GUID-45FB4385-7CBB-4EEB-B273-21B20F2EA1F4.html

## 2.5 离线更新LSDK

如果只更新boot分区：
```
$ flex-installer -b boot_LS_arm64_lts_5.10.tgz -d /dev/mmcblk0 (or /dev/sdx)
```

如果只更新rootfs：
```
$ flex-installer -r rootfs_<sdk_version>_ubuntu_arm64.tgz -d /dev/mmcblk0 (or /dev/sdx)
```

如果两个都更新：
```
$ flex-installer -b <boot_partition> -r <rootfs> -d /dev/mmcblk0 (or /dev/sdx)
```

* For SD boot, first generate composite firmware by command '_flex-builder -i mkfw -m -b sd_', then program it into SD card under U-Boot prompt, or program it by command 'flex-installer -f `<firmware>` -d /dev/sdx' or ‘sudo dd if=`<firmware> `of=/dev/mmcblk0 bs=1k seek=4 conv=fsync’ under Linux machine.
* For non-SD boot, first generate composite firmware by command '_flex-builder -i mkfw -m -b nor|qspi|xspi_', then program it into NOR|QSPI|FlexSPI flash device under U-Boot prompt.

[^1]

# Ref
[^1]:[Layerscape Software Development Kit User Guide](https://docs.nxp.com/bundle/GUID-487B2E69-BB19-42CB-AC38-7EF18C0FE3AE/page/GUID-38B02850-872C-4567-8088-75AC149FD52C.html)