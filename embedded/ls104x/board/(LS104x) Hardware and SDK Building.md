# 1. 自研TCU硬件概述
- 12V供电
- UART 115200 （TX对TX， RX对RX）
- SD卡/EMMC启动（如果==检测有SD卡则从SD卡启动，没有SD卡从EMMC启动==）

## 1.1 硬件外观

下图为TCU的硬件平台：
![](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202311231719751.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora202311231726489.png)

==Note，== **==串口是TX对TX，RX对RX==**==，并非交叉接法。==

## 1.2 硬件原理图

请访问公司的共享网盘：[https://office.autox.clu/Products/Files/#1339](https://office.autox.clu/Products/Files/#1339)

# 2. 系统及SDK

TCU的系统及软件在网站：[https://code.autox.ds/internal/tcu](https://code.autox.ds/internal/tcu) （请使用分支multipath_release）

![](https://raw.githubusercontent.com/carloscn/images/main/typora202311231720251.png)

### 2.1.1 准备源码

在仓库中选择：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202311231720038.png)

- flexbuild_lsdk2108.tgz是nxp原生的flexbuild环境
- ls1043_autox是TCU团队适配板子修改的文件

创建一个自己的工作区：`${HOME}/work/tcu_flex` 配置： `cp -rfv ls1043_autox/* flexbuild_lsdk2108/`

![](https://raw.githubusercontent.com/carloscn/images/main/typora202311231720567.png)

`cd flexbuild_lsdk2108`

按照下面的脚本执行：

```
source setup.env 				#执行这个脚本，配置环境变量
```

解决sdk.yml文件仓库不存在的问题，复制以下内容到config/sdk.yml，替换SDK.yml文件： https://github.com/carloscn/ls104x-bsp/blob/master/bsp/configs/sdk.yml


### 2.1.2 编译SDK

```
flex-builder -i clean-firmware               #清除之前编译的firmware文件

flex-builder -c atf -m ls1043ardb -b sd && flex-builder -i mkfw -m ls1043ardb -b sd      #编译sd卡，emmc固件

flex-builder -c atf -m ls1043ardb -b qspi && flex-builder -i mkfw -m ls1043ardb -b qspi  #编译qspi固件
```

后编译完之后如图log：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202311231721583.png)

### 2.1.3 固件烧录方式

#### 方式1：直接烧录到SD卡

![](https://raw.githubusercontent.com/carloscn/images/main/typora202311231721532.png)

```
sudo dd if=firmware_ls1043ardb_sdboot.img of=/dev/sdx seek=8 bs=512 && sync    #系统下烧写固件到sd卡
```

#### 方式2：烧录到flash或者emmc

以firmware_ls1043ardb_sdboot.img固件为例：
1. 下载固件：从仓库中提取firmware_ls1043ardb_sdboot.img
2. power reset板子，让板子从QSPI NOR Flash 0启动，进入FLASH 0的reboot。
3. 使用uboot丰富的功能，比如sd卡，tftp等把固件装在到板子的内存中，再通过uboot的flash写入功能反写数据到flash中。按照以下步骤：
    1. 可以从TFTP加载镜像：`setenv ipaddr 192.168.0.10;setenv serverip 192.168.0.100;ping 192.168.0.100` => `tftp $load_addr firmware_ls1046ardb_qspiboot.img` => `sf probe 0:0;sf erase 0x0 +$filesize && sf write $load_addr 0x0 $filesize`
    2. 可以从SD卡或者U盘读取镜像：=> `load mmc <device:part> $load_addr firmware_ls1046ardb_sdboot.img`
    3. 可以从SATA读取镜像：=> `load scsi <device:part> $load_addr firmware_ls1046ardb_sdboot.img`
4. 通过uboot的mmc工具把数据写入sd卡：=> `mmc dev 0; mmc write $load_addr 8 1f000`
5. 复位为从sd启动：cpld reset sd，系统会自动的从sd启动了。

## 2.2 单独编译固件

我们在开发某个固件的时候如果对整个工程全部编译就十分的麻烦，本节介绍如何单独编译固件及单独替换/烧录固件。

### 2.2.1 单独编译Uboot + ATF + RCW

#### 编译

```
flex-builder -c atf -m ls1043ardb -b sd      # build uboot-based ATF image for SD boot
flex-builder -c atf -m ls1043ardb -b qspi    # build uboot-based ATF image for QSPI-NOR
flex-builder -c atf -m ls1043ardb -b sd -s   # build uboot-based ATF image for SD secure boot on ls1043ardb
```
- -s 表示是否支持secure boot

所有的ATF的文件就在这里：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202311231722077.png)

bl2_sd.pbl:
- BL2 binary: 平台初始化的程序
- RCW binary
fip.bin:
- BL31: Secure runtime firmware
- BL32: Trusted OS, for example, OP-TEE (optional)
- BL33: U-Boot/UEFI image

具体可以参考： https://github.com/carloscn/blog/issues/151

#### 烧录

**烧录 bl2_sd.pbl on SD/eMMC card:**

=> `tftp 82000000 bl2_sd.pbl`
=> `mmc write 82000000 8 <blk_cnt>`

> Here, blk_cnt refers to number of blocks in SD card that need to be written as per the file size. For example, when you load bl2_sd.pbl from the TFTP server, if the bytes transferred is 82809 (14379 hex), then blk_cnt is calculated as 82809/512 = 161 (A1 hex). For this example, mmc write command will be: => mmc write 82000000 8 A1.

**烧录 fip.bin on SD/eMMC card:**
=> `tftp 82000000 fip.bin`
=> `mmc write 82000000 800 <blk_cnt>`

>Here, blk_cnt refers to number of blocks in SD card that need to be written as per the file size. For example, when you load fip.bin from the TFTP server, if the bytes transferred is 1077157 (106fa5 hex) , then blk_cnt is calculated as1077157/512 = 2103 (837 hex) . For this example, mmc write command will be: => mmc write 82000000 800 837.

### 2.2.3 单独编译Linux Kernel和设备树

`flex-builder -c linux`

![](https://raw.githubusercontent.com/carloscn/images/main/typora202311231725498.png)

具体参考：[https://docs.nxp.com/bundle/GUID-9160E9B1-ADBC-4313-9837-773DA05262FB/page/GUID-47B8F1F5-3A8F-45F4-A096-4D3DCDE8D07C.html](https://docs.nxp.com/bundle/GUID-9160E9B1-ADBC-4313-9837-773DA05262FB/page/GUID-47B8F1F5-3A8F-45F4-A096-4D3DCDE8D07C.html)

### 2.2.4 单独编译OPTEE

`flex-builder -c optee`

![](https://raw.githubusercontent.com/carloscn/images/main/typora202311231725975.png)
