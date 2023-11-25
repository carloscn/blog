# (Embedded) NXP-imx6 initialization

note:  The original `README` has been changed to `README.old`

# 1. 编译内核

## 1.1 Compiling the Linux Kernel for IMX6

`git@github.com:carloscn/imx-linux-4.1.15.git`

`make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- distclean`

`make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- imx_v7_defconfig`

`make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig` (optional)

`make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- all -j16`

* The compiled output `zImage` is saved on `arch/arm/boot`
* The compiled `.dtb` files are saved on the `arch/arm/boot/dts`

## 1.2 Starting compiled dirty Linux Kernel

调试模式，使用nfs网盘作为调试内核的模式。

### 1.2.1 config nfs on your host

#### installing nfs-kernel-server on your host
`sudo apt-get install nfs-kernel-server rpcbind`

#### config exports on your host
`sudo vim /etc/exports`

and you need to add the NFS sharing path on the exports:
`/home/carlos/nfs *(rw,sync,no_root_squash)`

#### changing the version of nfs-kernel-server

Note, the nfs version of ubuntu 20.04 is too high to transform file for uboot.
So we need to modify the version config manually by

`sudo vim /etc/default/nfs-kernel-server`

changing the version `RPCNFSDCOUNT="-V 2 8"`

### restarting the nfs

`sudo /etc/init.d/nfs-kernel-server restart`

### 1.2.2 copy the zImage to the nfs

`cp -r arch/arm/boot/zImage /home/carlos/nfs`

`cp -r arch/arm/boot/dts/imx6ull-14x14-evk.dtb /home/carlos/nfs`

### 1.2.3  some operations on your device uboot console

Ensure that the uboot args is `console=ttymxc0,115200 root=/dev/mmcblk1p2 rootwait rw`

`dhcp`

`nfs 80800000 192.168.31.2:/home/carlos/nfs/zImage`

`nfs 83000000 192.168.31.2:/home/carlos/nfs/imx6ull-14x14-evk.dtb`

`bootz 80800000 - 83000000`

### 1.2.4 replacing new kernel zImage on EMMC

* Transfer the zImage from host to device by `scp -r arch/arm/boot/zImage root@192.168.31.210:/run/media/mmcblk1p1/zImage`


# 2. 烧录到eMMC和SD卡 

## 2.1 uboot

### 2.1.1 代码
`git clone git@github.com:carloscn/imx-uboot.git`

### 2.1.2 编译

使用compile.sh编译

```C
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- distclean
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- mx6ull_14x14_ddr512_emmc_defconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- all -j16
```

## 2.2 板载eMMC下载

### 2.2.1 方法一：imdown.elf

使用imdown.elf工具下载到sd卡，在代码仓库中已经集成，`burn_tools`。方法就是插上SD卡，使用命令`sudo ./imxdown.elf u-boot.bin /dev/sdd` 后面的/dev/sdd使用 df -l来查看是不是自己的内存卡。

### 2.2.2 方法二：uboot sd或emmc命令

uboot 支持 EMMC 和 SD 卡，因此也要提供 EMMC 和 SD 卡的操作命令。一般认为 EMMC和 SD 卡是同一个东西，所以没有特殊说明，本教程统一使用 MMC 来代指EMMC 和 SD 卡。uboot 中常用于操作 MMC 设备的命令为“mmc”。

如果当前uboot没有损坏，我们可以使用uboot来更新uboot。编译好的uboot放在nfs或者tftp文件夹里面。使用tftp把数据加载到ram中。

`tftp 80800000 u-boot.imx`

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220924135514.png)

一共是384000字节，（384000/512 = 750个块 = 0x2ee个块）

再由ram加载到sd卡：

`mmc list` ：查看emmc列表
`mmc dev 0 0`：选中sd卡0的第0个分区
`mmc write 80800000 2 2ee` 从ram的80800000起始地址的写 0分区第2个块 长度 2ee

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220924140030.png)

note: 
* 如果是烧写mmc，还需要`mmc partconf 1 1 0 0 //分区配置，EMMC 需要这一步`
* 千万不要烧写SD卡或者EMMC的前两个块，里面保存着分区表。

### 2.2.3  配置uboot环境

EMMC 启动：`setenv bootargs console=ttymxc0,115200 root=/dev/mmcblk1p2 rootwait rw`

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220924142937.png)

网络启动：`setenv bootargs 'console=ttymxc0,115200 root=/dev/nfs nfsroot=192.168.137.18:/home/pjw/linux/nfs/rootfs,proto=tcp rw ip=192.168.137.20:192.168.137.18:192.168.1.1:255.255.255.0::eth0:off'`

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220924143010.png)


### 2.2.4 在boot环境烧写zImage到emmc或sd卡

`fatinfo mmc 1:1`
`fatls mmc 1:1`
`fatload mmc 1:1 80800000 zImage` 把emmc里面的zimage加载到RAM
`tftp 80800000 zImage` 把zImage数据加载到RAM
`fatwrite mmc 1:1 80800000 zImage 0x5c2720` 把ram里面数据保存在emmc分区，命名为zimage 然后可以使用fatls查看。


## 2.3 SD卡下载

### 2.3.1 SD卡分区

我们对SD卡进行分区。imx6的分区应该注意：
* bootloader: 以raw data的形式存在于sd卡中
* kernel：zImage，存于FAT32的分区中；
* rootfs：rootfs，存在于ext4中；

分区：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20230114122538.png)

* 分区1：7.2M 留给boot.imx，raw data
* 分区2：74MB FAT32，主要存储kernel和设备树
* 分区3：rootfs

关于SD 分区已经写成脚本：

https://github.com/carloscn/imx-uboot/blob/master/mk_sdcard.sh

### 2.3.2 kernel

kernel 拷贝到内存卡：
https://github.com/carloscn/imx-linux-4.1.15/blob/master/copy.sh

### 2.3.3 rootfs

下载：
https://drive.google.com/file/d/1Xl0nzFJp6bKqjz07Vi_Wn4V8EYyblknU/view?usp=share_link

`tar -xvf imx6ull_root.tar.bz2 -C /media/carlos/rootfs`

### 2.3.4 修改bootsrc

```
setenv bootcmd 'mmc dev 0; fatload mmc 0:2 80800000 zImage; fatload mmc 0:2 83000000 device.dtb; bootz 80800000 - 83000000;' 

setenv bootargs console=ttymxc0,115200 root=/dev/mmcblk0p3 rootwait rw

saveenv
```

