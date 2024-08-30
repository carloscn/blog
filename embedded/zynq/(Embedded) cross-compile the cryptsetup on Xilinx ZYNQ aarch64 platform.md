# [Embedded] cross-compile the cryptsetup on Xilinx ZYNQ aarch64 platform

## 1.1 Dev Env

This document is to describe how to transplant the `cryptsetup` (userspace) and `dm-crypt` (kernel space) to an aarch64 ARM platform. I'm holding a Xilinx [# Zynq® UltraScale+™ MPSoC](https://www.xilinx.com/products/silicon-devices/soc/zynq-ultrascale-mpsoc.html#productAdvantages) [ZCU111](https://www.xilinx.com/products/boards-and-kits/zcu111.html) platform basing arm AARCH64 four Cortex-A53s. My target is enabling the `cryptsetup` and `dm-crypt` to encrypt the Linux rootfs in a SD card. The dev env of zynqmp is Petalinux 2021, please note that the version of glibc when you transplanted `cryptsetup` and its corresponding libraries. 

## 1.2 Transplanting the cryptsetup on AARCH64

### 1.2.1 Toolschains

The AArch64 GNU/Linux toolchains (aarch64-none-linux-gnu) shall be required to cross-compile `cryptsetup` and its corresponding libraries. By now, glibc the newest the version is `GLIBC 2.33`, the executable files cross-compiled by it cannot be executed on Linux Runtime complied by the petalinux 2021. Therefore, the old toolchains should be downloaded by: [gcc-arm-10.2-2020.11-x86_64-aarch64-none-linux-gnu.tar.xz](https://developer.arm.com/-/media/Files/downloads/gnu-a/10.2-2020.11/binrel/gcc-arm-10.2-2020.11-x86_64-aarch64-none-linux-gnu.tar.xz?revision=972019b5-912f-4ae6-864a-f61f570e2e7e&rev=972019b5912f4ae6864af61f570e2e7e&hash=A973F165C6D012E0738F90FB4A0C2BA7)

For more information about toolchains configuration, please refer to the link: https://github.com/carloscn/blog/issues/26

The toolchains set:

![](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202301031046193.png)

### 1.2.2 libjson-c

`libjson-c` is required by `cryptsetup`, so we shall cross-compile the `libjson-c` for `cryptsetup`. We selected `json-c-0.13.1.tar.gz` on linuxfromscratch.org/blfs/view/svn/general/json-c.html. You can also get the tarball by following the command directly.

The compiling steps are shown as following:

`wget "https://s3.amazonaws.com/json-c_releases/releases/json-c-0.13.1.tar.gz" --no-check-certificate`

`tar -xvf json-c-0.13.1.tar.gz && cd json-c-0.13.1`

``./configure --prefix=`pwd`/out --host=aarch64-none-linux-gnu``

`make`

`make install`

The dynamic libraries are outputted by the building result:

![](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202301031115977.png)

### 1.2.3 util-linux

The `util-linux` provides `libuuid` and `libblkid` that are required by `cryptsetup`.  You can get the `util-linux` source code by the link https://mirrors.edge.kernel.org/pub/linux/utils/util-linux/

`wget "https://mirrors.edge.kernel.org/pub/linux/utils/util-linux/v2.34/util-linux-2.34.tar.gz" --no-check-certificate`

`tar -xvf util-linux-2.34.tar.gz && cd util-linux-2.34`

``./configure --prefix=`pwd`/out --host=aarch64-none-linux-gnu --without-ncurses``

`make uuidd -j8`

`make blkid -j8`

Because there is no `libtinfo` for aarch64 toolchains, we cannot make use of `make install` to gather the libraries. So the libraries shall be arranged manually.

`cd out`

`mkdir lib`

`cp -Lvr ../.libs/* ./lib`

`mkdir include && mkdir include/uuid && mkdir include/blkid`

`cp -r ../libuuid/src/uuid.h ./include/uuid`

`cp -r ../libblkid/src/blkid.h ./include/blkid`

### 1.2.4 libpopt

`wget "http://jaist.dl.sourceforge.net/project/cross-stuff/cross-stuff/1.0/popt-1.7.tar.gz" --no-check-certificate`

`tar -xvf popt-1.7.tar.gz && cd popt-1.7`

``./configure --prefix=`pwd`/out --host=aarch64-none-linux-gnu``

The makefile system of `libpopt` cannot setup an aarch64 host, we need to add aarch64 to the makefile system manually.

`rm -rf config.guess config.sub`

`wget -O config.guess 'http://git.savannah.gnu.org/gitweb/?p=config.git;a=blob_plain;f=config.guess;hb=HEAD'`

`wget -O config.sub 'http://git.savannah.gnu.org/gitweb/?p=config.git;a=blob_plain;f=config.sub;hb=HEAD'`

Then re-execute ``./configure --prefix=`pwd`/out --host=aarch64-none-linux-gnu``

`make -j8`

`make install`

![](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202301031141932.png)

### 1.2.5 libdevmapper

`libdevmapper` is required by `cryptsetup`, so we shall cross-compile the `libdevmapper` for `cryptsetup`. We selected [2.02.133-1ubuntu10](https://launchpad.net/ubuntu/+source/lvm2/2.02.133-1ubuntu10) on https://launchpad.net/ubuntu/+source/lvm2. You can also get the tarball by the following command direclty.

`wget "https://launchpad.net/ubuntu/+archive/primary/+sourcefiles/lvm2/2.02.133-1ubuntu10/lvm2_2.02.133.orig.tar.xz" --no-check-certificate`

`tar -xvf lvm2_2.02.133.orig.tar.xz && cd lvm2-2.02.133`

The just compiled `libuuid` and `libblkid` are requried by the `libdevmapper`. The configure cmd shall be "``./configure --prefix=`pwd`/out --host=aarch64-none-linux-gnu CFLAGS="-I/home/haochenwei/work/temp/cryptsetup_build/tools/util-linux-2.34/out/include" LDFLAGS="-L/home/haochenwei/work/temp/cryptsetup_build/tools/util-linux-2.34/out/lib"``", make use of `C_FLAGS` and `LD_FLAGS` to specify the `libuuid` and `libblkid` just compiled. Note the `C_FLAGS` and `LD_FLAGS` shall use the absolute path.

Then the config files are generated by the `configure` cmd. We need to modify the `include/configure.h` and comment the two lines:

![](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202301031214173.png)

Modify the `./lib/filters/filter-sysfs.c` to add the two lines:

```C
#include <sys/types.h>
#include <sys/sysmacros.h>
```

![](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202301031227603.png)

Then `make -j8`

`make install`

The errors of `make install` will prompt permission denial. You need to `sudo chown -R yourname:yourname /etc/lvm`.

Then `make install`

![](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202301031230749.png)

### 1.2.6 libssl

The openssl will provide the `libssl`, so we just need to compile the openssl for AARCH64. We use the openssl 1.1.1s on https://www.openssl.org/source/openssl-1.1.1s.tar.gz.

`wget "https://www.openssl.org/source/openssl-1.1.1s.tar.gz"  --no-check-certificate`

`tar -xvf openssl-1.1.1s.tar.gz && cd openssl-1.1.1s`

``CC=aarch64-none-linux-gnu-gcc ./config --prefix=`pwd`/out no-asm``

Furtherly, we need to change the Makefile:

![](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202301031239850.png)

`make -j8`

`make install -j8`

![](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202301031240139.png)

### 1.2.7 cryptsetup

Now, we cross-compile the `cryptsetup`:

`wget "https://gitlab.com/cryptsetup/cryptsetup/-/archive/v2.2.x/cryptsetup-v2.2.x.tar.gz" --no-check-certificate`

Before the cross-compiling, we need to arrange out compiled libraries above. The libraries source code on my host are shown as following:

![](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202301031244478.png)

We'd better create a `deps` directory to store the above libraries. 

`cd deps`

`cp -Lvr ../json-c-0.13.1/out/* .`

`cp -Lvr ../lvm2-2.02.133/out/* .`

`cp -Lvr ../openssl-1.1.1s/out/* .`

`cp -Lvr ../popt-1.7/out/* .`

`cp -Lvr ../util-linux-2.34/out/* .`

Then copy the all include files and libraries to toolchains `usr`:

`cp -r include/* ~/opt/cross-compile/gcc-arm-10.2-2020.11-x86_64-aarch64-none-linux-gnu/aarch64-none-linux-gnu/libc/usr/include/`

`cp -r lib/* ~/opt/cross-compile/gcc-arm-10.2-2020.11-x86_64-aarch64-none-linux-gnu/aarch64-none-linux-gnu/libc/usr/lib/`

Now, we will compile the `cryptsetup`:

`tar -xvf cryptsetup-v2.2.x.tar.gz && cd cryptsetup-v2.2.x`

We need to change the `/run/cryptsetup` -> `/etc/cryptsetup` by `grep -rni -s "/run" | grep config`

`./autogen.sh`

Then we need to config the makefile by:

``./configure --host=aarch64-none-linux-gnu --prefix=`pwd`/out`

Before the `make install`, we need to change the Makefile:

![](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202301031330239.png)

`make install`

Returning to the `deps` dir,  `cp -Lvr ../cryptsetup-v2.2.x/out/* .` to copy all files.

Besides, we need to copy the `libgcc_s.so.1` of toolchains to the lib. `cp -Lvr /home/haochenwei/opt/cross-compile/gcc-arm-10.2-2020.11-x86_64-aarch64-none-linux-gnu/aarch64-none-linux-gnu/libc/usr/lib64/libgcc_s.so* .`


## 1.3 Enabling dm-crypt and AES engine on Linux Kernel

`petalinux-config -c kernel`

```
[*] Enable loadable module support
Device Drivers --->
    [*] Multiple devices driver support (RAID and LVM) --->
        <*> Device mapper support
        <*>   Crypt target support
```

![image-20230101182215875](https://raw.githubusercontent.com/carloscn/images/main/typoratyporatyporaimage-20230101182215875.png)

Next, the Linux kernel needs to support the set of cryptographic APIs that the administrator wants to use for encryption. These can be found under the _Cryptographic API_ section:

```
[*] Cryptographic API --->
    <*> XTS support
    <*> SHA224 and SHA256 digest algorithm
    <*> AES cipher algorithms
    <*> User-space interface for hash algorithms
    <*> User-space interface for symmetric key cipher algorithms
```

![](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202301031341048.png)

Then config the Xilinx AES accelerator, https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/64749783/ZynqMP+AES+Driver to use the Xilinx AES engine. For more information about Xilinx AES engine, https://www.xilinx.com/content/dam/xilinx/support/documents/ip_documentation/aes/v1_1/pg383-aes.pdf

![](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202301031346309.png)


## 1.4 Transforming the files to rootfs

The built files can be set to a rootfs or a ramdisk.

![](https://raw.githubusercontent.com/carloscn/images/main/typoratyporatypora202301031354972.png)

Then run the rootfs, the cryptsetup can be used normally:

![image-20230101194936257](https://raw.githubusercontent.com/carloscn/images/main/typoratyporatyporaimage-20230101194936257.png)

[^1][^2]

## Ref
[^1]:[64位ARM移植cryptsetup工具(aarch64-linux-gnu交叉编译)-环境篇](https://blog.csdn.net/huadianyue/article/details/103013950?spm=1001.2014.3001.5501)
[^2]:[dm-crypt](https://wiki.gentoo.org/wiki/Dm-crypt)