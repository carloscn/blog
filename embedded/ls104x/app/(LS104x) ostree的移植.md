ostree工具是一个依赖其他库非常多的工具，我们可以借助一些工具来帮助自己完成ostree的移植。例如buildroot就是一个非常好的工具。在LS104x中的`flexbuild`工具已经集成了buildroot，可以使用buildroot来完成ostree。

关于flexbuilder的使用，可以参考：https://docs.nxp.com/bundle/GUID-C241BB12-95F6-4D6B-A205-7EFD35551DE2/page/GUID-47B8F1F5-3A8F-45F4-A096-4D3DCDE8D07C.html

# 1. 编译带有ostree的ramdisk

建议使用原生的bash来运行这些脚本。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309211649242.png)

## 1.1 buildroot的配置

先对buildroot进行配置：

`flex-builder -i mkrfs -r buildroot:imaevm:custom -a arm64`

之后会弹出界面：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309211650898.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202309211651830.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309211651672.png)

还需要配置网络库：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309211651235.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309211651859.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309211651352.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309211651613.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309211651374.png)



## 1.2 buildroot的编译

`flex-builder -i mkrfs -r buildroot:imaevm -a arm64`

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309211653813.png)

编译完之后生成的文件，解压rootfs.tar.gz文件为ramdisk文件夹：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309211653705.png)

我们可以根据自己需求来修改脚本了，我这里修改了mmc的挂在分区：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309211654176.png)

然后输出了ostree的help来测试ostree已经可以运行：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309211655493.png)

使用gen_ramdisk.sh生成脚本来打包ramdisk：

```bash
# /bin/bash

cd ramdisk
sudo find . | sudo cpio -H newc -o | gzip -9 > new_initramfs.cpio.gz

rm -rf ../new_initramfs.cpio.gz
mv new_initramfs.cpio.gz ../             
```

**Note，一定要用sudo权限运行这个脚本。**


## 2. 生成FIT image

参考上一节。https://github.com/carloscn/blog/issues/192


## 3. 测试

ramdisk中已经启动了ostree：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309211657879.png)

