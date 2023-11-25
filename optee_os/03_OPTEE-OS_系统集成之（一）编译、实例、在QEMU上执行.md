# 03_OPTEE-OS_系统集成之（一）编译、实例、在QEMU上执行

# 1. OP-TEE运行环境的搭建编译

OP-TEE是开源的TEE解决方案，任何人都可从github库中获取OP-TEE的源代码，本章主要包括如何从github中获取OP-TEE的源代码、如何搭建运 行环境以及整个OP-TEE工程的编译过程。本书以 QEMU作为运行平台，Hikey或者其他平台的编译和使用方式与QEMU平台类似。

## 1.1 环境准备

https://optee.readthedocs.io/en/latest/building/prerequisites.html

## 1.2 准备和编译

`mkdir optee-repo && cd optee-repo`
`curl https://mirrors.tuna.tsinghua.edu.cn/git/git-repo -o ~/bin/repo`
`chmod a+x ~/bin/repo`
`export PATH=~/bin:$PATH`
`~/bin/repo init -u https://github.com/OP-TEE/manifest.git -m qemu_v8.xm`
`repo sync`
`cd ./build`
`make -j2 toolchains`
`make -f qemu_v8.mk all -j16`
`make -f qemu_v8.mk run-only`

## 1.3 xtest运行

![](https://raw.githubusercontent.com/carloscn/images/main/typoratelegram-cloud-document-5-6178954339613149462.jpg)

在qemu的shell输入`c`就会启动系统

![](https://raw.githubusercontent.com/carloscn/images/main/typoratelegram-cloud-document-5-6178954339613149465.jpg)

输入`xtest`开始运行： 

![](https://raw.githubusercontent.com/carloscn/images/main/typoratelegram-cloud-document-5-6178954339613149458.jpg)


## 1.4 初识optee

Normal world会显示如下Linux相关的log：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221002132708.png)

OPTEE端也会显示内存映射的详细信息还有OPTEE的版本信息。
```
I/TC: OP-TEE version: 3.15.0-dev (gcc version 10.2.1 20201103 (GNU Toolchain for the A-profile Architecture 10.2-2020.11 (arm-10.16))) #8 Thu Dec 23 15:55:20 UTC 2021 aarch64
```

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221002132838.png)

一旦OPTEE加载成功之后，就会切换到Normal World的Linux

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221002132944.png)

在Linux端能够看到Linux的OPTEE的驱动probe的信息，到这里可以推断出OPTEE是以Linux驱动的结构存在于Linux Kernel中的。

```bash
[    2.019187] optee: probing for conduit method.
[    2.021676] optee: revision 3.15 (6be0dbca)
[    2.027383] optee: dynamic shared memory is enabled
[    2.058506] optee: initialized driver
```

可以在/dev下面看到optee的设备节点：
```bash
#ls -al /dev/tee*
crw-rw----    1 root     teeclnt   247,   0 Dec 24 07:33 /dev/tee0
crw-rw----    1 root     tee       247,  16 Dec 24 07:33 /dev/teepriv0
#
```

OP-TEE Client (tee-supllicant)以进程的形式运行在后台，而且随着linux启动。
```bash
#ps -A | grep tee
  145 tee      /usr/sbin/tee-supplicant -d /dev/teepriv0
  146 root     [optee_bus_scan]
  180 root     grep tee
#
```

安全应用会产生ta文件在文件系统`/lib/optee_armtz`
```bash
# ls /lib/optee_armtz/
12345678-5b69-11e4-9dbb-101f74f00099.ta
25497083-a58a-4fc5-8a72-1ad7b69b8562.ta
2a287631-de1b-4fdd-a55c-b9312e40769a.ta
380231ac-fb99-47ad-a689-9e017eb6e78a.ta
484d4143-2d53-4841-3120-4a6f636b6542.ta
528938ce-fc59-11e8-8eb2-f2801f1b9fd1.ta
5b9e0e40-2636-11e1-ad9e-0002a5d5c51b.ta
5ce0c432-0ab0-40e5-a056-782ca0e6aba2.ta
5dbac793-f574-4871-8ad3-04331ec17f24.ta
60276949-7ff3-4920-9bce-840c9dcf3098.ta
614789f2-39c0-4ebf-b235-92b32ac107ed.ta
731e279e-aafb-4575-a771-38caa6f0cca6.ta
873bcd08-c2c3-11e6-a937-d0bf9c45c61c.ta
8aaaf200-2450-11e4-abe2-0002a5d5c51b.ta
a4c04d50-f180-11e8-8eb2-f2801f1b9fd1.ta
a734eed9-d6a1-4244-aa50-7c99719e7b7b.ta
b3091a65-9751-4784-abf7-0298a7cc35ba.ta
b689f2a7-8adf-477a-9f99-32e90c0ad0a2.ta
b6c53aba-9669-4668-a7f2-205629d00f86.ta
c3f6e2c0-3548-11e1-b86c-0800200c9a66.ta
cb3e5ba0-adf1-11e0-998b-0002a5d5c51b.ta
d17f73a0-36ef-11e1-984a-0002a5d5c51b.ta
e13010e0-2ae1-11e5-896a-0002a5d5c51b.ta
e626662e-c0e2-485c-b8c8-09fbce6edf3d.ta
e6a33ed4-562b-463a-bb7e-ff5e15a493c8.ta
ee90d523-90ad-46a0-859d-8eea0b150086.ta
f157cda0-550c-11e5-a6fa-0002a5d5c51b.ta
f4e750bb-1437-4fbf-8785-8d3580c34994.ta
ffd2bded-ab7d-4988-95ee-e4962fff7154.ta
#
```

有一些实例："/usr/bin"
```bash
# ls /usr/bin/optee*
/usr/bin/optee_example_acipher         /usr/bin/optee_example_plugins
/usr/bin/optee_example_aes             /usr/bin/optee_example_random
/usr/bin/optee_example_hello_world     /usr/bin/optee_example_secure_storage
/usr/bin/optee_example_hotp
# 
# ls /usr/bin/xtest 0
/usr/bin/xtest
#
```

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221002133545.png)

# 2. 增加carlos_test实例

https://github.com/carloscn/optee_examples/tree/carlos/carlos_test

![telegram-cloud-document-5-6181196767808194354](https://user-images.githubusercontent.com/16836611/193448224-84a85837-e367-4ddb-9e36-08feb435b2d7.jpg)
