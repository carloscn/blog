在NXP的官方手册中，并没有给出如何使用initramfs。本文基于NXP的LS1046a的平台，对kernel、设备树以及initramfs进行FIT格式的打包，接着介绍如何启动initramfs以及如何从initramfs切换到真正的rootfs。LS1046a的分区已经FIT image的格式如下：

<div  align="center"><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202309211659118.png" alt="" width="375"></img></div>

## 1. initramfs制作

### 1.1 导出initramfs

我们需要准备正确的initramfs的系统。通常initramfs系统有一些对于硬件的初始化的操作，所以对于不同嵌入式的平台，initramfs是不能够通用的。我们需要使用芯片原厂提供的initramfs。在NXP的bsp包中并没有给出initramfs的源文件，我们需要借助yocto工程中导出initramfs。

<div  align="center"><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202309211700536.png" alt="" width="100%"></img></div>

可以按照我下面的配置，也可以在initramfs的官方文档中使用他们的配置：

```bash
IMAGE_FSTYPES += "ext4 ext4.gz "
INITRAMFS_IMAGE = "core-image-base"
INITRAMFS_IMAGE_BUNDLE = "1"
IMAGE_FSTYPES = "ext4.gz"
```

initramfs的配置可以参考官方文档：[https://docs.yoctoproject.org/ref-manual/variables.html#term-INITRAMFS\_IMAGE](https://docs.yoctoproject.org/ref-manual/variables.html#term-INITRAMFS\_IMAGE)

* 如果你写的是`core-image-base`使用`bitbake core-image-base`&#x20;
* 如果你写的是`core-image-minimal-initramfs` 使用`bitbake core-image-minimal-initramfs`

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309211700156.png)

输出的cpio.gz的文件就是initramfs的源文件。

### 1.2 修改initramfs

cpio.gz文件是一个gzip过的cpio文件。首先要解压：

`gzip core-image-minimal-initramfs-ls1046ardb-20230920085754.cpio.gz`

然后创建一个文件夹：

`mkdir ramdisk && cd ramdisk && cp -r ../core-image-minimal-initramfs-ls1046ardb-20230920085754.cpio .`

解cpio：

`cpio -idmv < core-image-minimal-initramfs-ls1046ardb-20230920085754.cpio`

开始修改文件（我目前的修改是改的rootfs的函数，主要意思就是把SD卡的mmcblk0p3分区挂载在/rootfs上）该脚本会自动switch\_root到/rootfs

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309211701867.png)

打包这里需要使用脚本操作（因为必须是root权限）

```bash
# /bin/bash

cd ramdisk
sudo find . | sudo cpio -H newc -o | gzip -9 > new_initramfs.cpio.gz

rm -rf ../new_initramfs.cpio.gz
mv new_initramfs.cpio.gz ../
```

**这个脚本一定要sudo运行**，否则会出现：[https://wowothink.com/866559ba/](https://wowothink.com/866559ba/) 提到的`only root can use`问题：

```
mount: only root can use "--types" option (effective UID is 1000)
mount: only root can use "--types" option (effective UID is 1000)
mount: only root can use "--types" option (effective UID is 1000)
mount: only root can use "--types" option (effective UID is 1000)

ls: cannot access '/sys/class/udc': No such file or directory
No udc Available!
```

参考：[https://www.linuxquestions.org/questions/linux-software-2/nfs-rootfs-mount-only-root-can-mount-proc-on-proc-4175595326/](https://www.linuxquestions.org/questions/linux-software-2/nfs-rootfs-mount-only-root-can-mount-proc-on-proc-4175595326/)

> 也就是说，之所以出现这种错误，是`UID`不匹配，我的`.cpio.gz.u-boot`文件中的文件`UID`是为1000，但是在mfgtools中执行的`UID`是为0(root用户)，导致出现了问题。

此时，我们拿到了我们修改过的new\_initramfs.cpio.gz 文件，一会，这部分要放入到FIT image中。

## 2. FIT image制作

这部分和一般的FIT image制作没有什么区别，我们需要准备：

* Image.gz或者Image文件
* 设备树二进制文件
* initramfs.cpio.gz文件

撰写its脚本，命名为`ls1046a_fit.its`：

````json
```
/dts-v1/;

/ {
        description = "kernel+dtb/fdt fit image";
        #address-cells = <1>;

        images {
                kernel {
                        description = "kernel image";
                        data = /incbin/("Image.gz");
                        type = "kernel";
                        arch = "arm64";
                        os = "linux";
                        compression = "gzip";
			load = <0x94200000>;
			entry = <0x94200000>;
			kernel-version = <1>;
                        hash {
                                algo = "sha256";
                        };
                };

		initrd {
			description = "initrd for arm64";
			data = /incbin/("new_initramfs.cpio.gz");
			type = "ramdisk";
			arch = "arm64";
			os = "linux";
			compression = "gzip";
			load = <0x80000000>;
			entry = <0x80000000>;
			hash {
				algo = "sha256";
			};
		};

                fdt {
                        description = "dtb blob";
                        data = /incbin/("fsl-ok1046a-1133-5a59-c2.dtb");
                        type = "flat_dt";
                        arch = "arm64";
                        compression = "none";
                        load = <0xA0000000>;
                        fdt-version = <1>;
                        hash {
                                algo = "sha256";
                        };
                };
        };
        configurations {
                default = "standard";
                standard {
                        description = "Standard Boot";
                        kernel = "kernel";
                        fdt = "fdt";
                        ramdisk = "initrd";
                        hash {
                                algo = "sha256";
                        };
                };
        };
};
````

**需要注意的是地址问题，注意load的地址和uboot的加载FIT image的地址不要重叠，否则无法启动**。

生成FIT image的命令是，如果找不到这个工具可以安装`sudo apt install u-boot-tools`：

`mkimage -f ls1046a_fit.its image.ub`

最后我们得到一个image.ub文件。

## 3. uboot脚本

除此之外，还需要修改uboot脚本，让uboot去读取我们的FIT的image，并且从FIT格式的image启动。

### 3.1 bootargs

`setenv bootargs 'console=ttyS0,115200 root=/dev/mmcblk0p3 rw rootwait earlycon=uart8250,mmio,0x21c0500; '`

注意：root=/dev/mmcblk0p3 或者 /dev/ram0 这个都无所谓了，因为FIT格式会强制从initramfs启动，而且真正的rootfs也是由initramfs的脚本控制切换。

### 3.2 bootcmd

`setenv bootcmd 'mmc dev 0; fatload mmc 0:2 0xb0000000 image.ub; bootm 0xb0000000'`

我这部分是放在了`/dev/mmcblk0p2`中，因此从这里读取image.ub。

uboot阶段：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309211701550.png)

initramfs和rootfs阶段：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309211701723.png)

程序放在：[https://github.com/carloscn/ls104x-bsp/tree/master/initramfs](https://github.com/carloscn/ls104x-bsp/tree/master/initramfs)

## End