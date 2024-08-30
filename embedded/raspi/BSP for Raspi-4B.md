# 1. Boot流程

以下是树莓派的boot流程：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202403021452499.png)

1. SoC BootROM，这一阶段非常简单，主要支持读取SD中FAT32文件系统的第二阶段启动程序；
2. SoC BootROM加载并执行启动分区中的`bootcode.bin`，该文件主要功能是解析elf格式文件，并加载并解析同样位于分区中的`start4.elf`；
3. 运行`start4.elf`，读取并解析`config.txt`配置文件，并加载执行真正的u-boot程序。

# 2. SD卡分区

SD卡分区可以参考：

https://data.engrie.be/RaspberryPi/Raspberry_Pi_-_Part_16_-_Devices_Partitions_and_Filesystems_on_Your_Pi.pdf

![](https://raw.githubusercontent.com/carloscn/images/main/typora202403021500013.png)

我的分区是这样的：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202403021504540.png)

## 2.1 boot分区

相关的boot分区在： https://github.com/raspberrypi/firmware 的boot中。里面包含了设备树、config.txt等文件、还有预编译的kernel文件。

由于u-boot中没有预置rpi4的dts文件（device tree source），因此采用了在u-boot运行时动态传入硬件描述dtb(device tree blob)文件的方式，用于u-boot启动时枚举硬件。

另外，在rpi3上运行过64位u-boot的都知道，如果在`config.txt`中没有特别指明kernel的位置，那么`start.elf`（或`start4.elf`）默认需要并启动的文件是`kernel8.img`：

1. `kernel8.img`：64位的Raspberry Pi 4和Raspberry Pi 4；
2. `kernel7l.img`：32位的Raspberry Pi 4（使用LPAE）；
3. `kernel7.img`：32位的Raspberry Pi 4、Raspberry Pi 3和Raspberry Pi 2（未使用LPAE）；
4. `kernel.img`：其他版本的树莓派。

## 2.2 rootfs分区

我们使 https://downloads.raspberrypi.com/raspios_lite_arm64/images/ 树莓派提供的img来做rootfs分区。需要注意的是，这个img文件，包含了boot分区和rootfs分区，我们可以把boot分区的内容删除，然后自己填充kernel和uboot。

另外还要注意，树莓派的用用户登录不再是默认的pi和raspberry了，而是需要把密码放在userconf.txt文件中，并且需要openssl加密。参考： 

`sudo echo "carlos:$(openssl passwd -6 -stdin <<<'123456')" > boot/userconf.txt`

# 3. bsp编译

本仓库已经把所有的kernel和uboot以及firmware都准备好了，直接可以`make prepare` -> `make image`。

## 3.1 编译uboot

在树莓派中需要启动FIT格式的和FIT验签的功能，树莓派是默认没有的。已经在bsp/configs 提供了rpi_4_user_defconfig 用于支持FIT格式的image。

`make u-boot` 会自动编译uboot。

## 3.2 编译kernel

kernel不需要变动，直接`make linux`，会生成Image和Image.gz。

## 3.3 制作FIT格式

`make fit`

没有ramdisk的FIT image `make fit_noram`

# 4. uboot调试命令

```
fatload mmc 0:1 $fdt_addr_r bcm2711-rpi-4-b.dtb
fatload mmc 0:1 $kernel_addr_r Image
booti $kernel_addr_r - $fdt_addr_r

fatload mmc 0:1 4000000 image.ub
bootm 4000000
```

# 5. uboot验签

uboot验签kernel这块真的是超级让我费解：
* BUG1： 经过测试，错误的public key，甚至没有public key居然在uboot验签阶段可以启动！
* BUG2：在设备树里面读不到正确的public key。

根据uboot的签名和验签的机制，签名是在host的mkimage完成的，然后使用-K参数制定uboot的二进制fdt，mkimage会自动把public key导入到uboot的fdt中，然后使用 EXT_DTB=设备树 make 把带有公钥的fdt导入到boot.bin中。

这样uboot启动的时候，会从boot.bin中的fdt读取到public key然后会去校验fit_image中signature。

调试的时候发现一个非常让人费解的问题，我用不同的key去签，然后不更新uboot.bin中的公钥，我发现居然可以启动到内核。ops！！！

后来发现，uboot的逻辑，是否验证通过都会往下走！（好奇怪的设定），我理解kernel验证失败了，压根就不能启动啊！我尝试加log看看为什么会返回，查看到底哪里失败了。如下log是我调试打印出来的， 发现怎么都找不到signature的节点：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202403031047853.png)

这里返回是0，所以这里的逻辑非常粗暴，就是如果没有signature的节点，成功。

![](https://raw.githubusercontent.com/carloscn/images/main/typoratyporatypora202403031034197.png)

其实这里应该做更丰富的逻辑判断，如果用户使能FULL check，而且没有找到key的节点，那么就返回EPERM错误。否则，uboot验签有啥用！

好吧，通过改这个code，可以拦截住错误的key了。然后我们再查一下，为什么会没有signature节点。

我是本地的check_sign工具可以验证image.ub通过呀！受到这个帖子的启发 ： https://www.mail-archive.com/u-boot@lists.denx.de/msg417296.html  我才发现，树莓派这个设定真的是。。。树莓派的上层bootrom会bybpass掉uboot.bin中的fdt，而使用sd卡boot分区的linux的dtb。 而我sd卡中存储的dtb是树莓派出厂的dtb，所以没有任何的public key啊！

我尝试mkimage的时候 -K去给uboot的fdt添加，然后拷贝到sd卡，树莓派板子无法启动。我只能给linux的dtb添加public key，然后拷贝sd卡，结果板子可以启动了，也可以验签了。

![](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202403031045031.png)

真的是让人非遗思索的设计！再一次证明，嵌入式必须一个平台一个说法。



