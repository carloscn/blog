在 [[LS104x] 使用ostree更新rootfs](https://github.com/carloscn/blog/issues/194#top) 中我们在ramdisk中使用ostree的checkout功能还原出一个完整版本rootfs，但是这个方法并没有完全彻底的使用ostree的部署功能。本节将要总结和整理一个ostree的部署功能。类似于该demo：[YOUTUBE - Designing OSTree based embedded Linux systems with the Yocto Project](https://www.youtube.com/watch?v=3i48NbAS2jUube)
在 [[LS104x] 使用ostree更新rootfs](https://github.com/carloscn/blog/issues/194#top) 中我们在ramdisk中使用ostree的checkout功能还原出一个完整版本rootfs，但是这个方法并没有完全彻底的使用ostree的部署功能。本节将要总结和整理一个ostree的部署功能。类似于该demo：[YOUTUBE - Designing OSTree based embedded Linux systems with the Yocto Project](https://www.youtube.com/watch?v=3i48NbAS2jUube)

从ubuntu更新为busybox，更新原理如图所示：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309270931825.png)

Note，这些ostree的部署的逻辑被包含在了meta-updater里面， https://github.com/advancedtelematic/meta-updater/tree/master 。但metaupdater集成度很高，而且该仓库已经很久没有更新了，就导致和新版的yocto集成有很多问题，需要不少的工作量。因此，本文手动部署ostree，并且参考meta-updater里面的逻辑手动补充一些脚本来完成ostree的部署和ostree部署的rootfs的使用。


# 1. 服务器端配置

本文使用HTTP而不是HTTPS，并且服务器使用本机HOST局域网更新。

## 1.1 仓库配置

- 需要进行ostree仓库的初始化
- 需要在ostree仓库中增加busybox的rootfs文件
- **对rootfs文件进行部署标准化改造**
- 需要commit仓库变化
- 需要生成summary文件
- 需要启动服务器监听程序

创建初始化仓库：

`mkdir -p repo && ostree --repo=repo init --mode=archive-z2 && mkdir -p rootfs`

- rootfs文件为存放busybox rootfs文件的路径；
- repo文件夹为仓库的配置文件；

### 1.1.1 rootfs改造

需要把rootfs下面根目录的etc转移到/usr/下面：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309271112368.png)

> The deployment should not have a traditional UNIX `/etc`; instead, it should include `/usr/etc`. This is the “default configuration”. When OSTree creates a deployment, it performs a 3-way merge using the _old_ default configuration, the active system’s `/etc`, and the new default configuration. In the final filesystem tree for a deployment then, `/etc` is a regular writable directory.
> https://ostreedev.github.io/ostree/deployment/

### 1.1.2 kernel文件

除此之外，还需要创建kernel、devicetree和initramfs的文件到下面的文件路径：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309271117761.png)

命名规则如下所示：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309271117320.png)

Note，不要使用软链接符号。

**没有以上两步，ostree无法在板子上部署**。

## 1.2 仓库上传

可能有的需要sudo命令：

`ostree --repo=repo commit --branch=master --subject="image v1 (ls104x) rootfs)" rootfs`


不需要sudo：

`ostree log master --repo=repo`

`ostree summary --repo=repo ./repo/branch.summary -u`

使用服务器监听repo：

`python3 -m http.server 8000 --bind 10.10.192.121 --directory repo`

# 2. 客户端配置（板级）

## 2.1 初始化版本的rootfs

我们需要准备一个能够正常启动，包括ostree和网络功能的的rootfs，作为基本的rootfs。然后我们依赖于这个rootfs，进入Linux runtime对ostree进行部署。

在我的demo中使用的是ubuntu：（当然ubuntu会很大，可以采用一些轻量的rootfs）

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309271121771.png)

使用这个版本进入正常的Linux Runtime，接下来我们要在板子上部署ostree。

## 2.2 部署ostree

部署之前请检查你的板子：
* 联网功能（包括ip是否正确）
* 时间正确
* 足够的空间

### 2.2.1 创建标准的sysroot

进入板子的根目录`cd /` 并且创建一个`sysroot`文件夹 `mkdir sysroot`。

初始化一个标准的sysroot文件夹：

`ostree admin init-fs sysroot`

你可以得到一个：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309271126980.png)

### 2.2.2 增加远程repo

给仓库增加

`ostree remote add --repo=/sysroot/ostree/repo --no-gpg-verify origin http://10.10.192.121:8000/`

从远程仓库拉去数据：

`ostree pull --repo=/sysroot/ostree/repo origin:master`

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309271129072.png)

`ostree log --repo=/sysroot/ostree/repo origin:master`

### 2.2.3 部署仓库

`ostree admin --sysroot=sysroot --os=tcu deploy origin:master`

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309271131277.png)

查看部署情况： `ostree admin --sysroot=sysroot status`

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309271131638.png)

部署之后的文件：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309271132330.png)

在`/sysroot/ostree/deploy/tcu/deploy`目录就可以看到：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309271133880.png)

upgrade：

`ostree admin --sysroot=sysroot --os=tcu upgrade`

通过这个命令，可以看到，ostree把最新的部署rootfs中的etc从/usr/etc拿了出来：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309271343539.png)

之后可以通过一些脚本，在部署完成之后，拿到rootfs的地址，然后在ramdisk里面配合一些脚本解析进行切换。

### 2.2.4 ramdisk脚本

``` bash
#!/bin/sh
#
# Copyright 2018 NXP
#
# SPDX-License-Identifier:      BSD-3-Clause
#

# mount the /proc and /sys filesystems.
/bin/mount -n -t proc none /proc
/bin/mount -n -t sysfs none /sys

a=`/bin/cat /proc/cmdline`

# Mount the root filesystem.
if ! echo $a | /bin/grep -q 'mount=' ; then
    # set default mount device if mountdev is not set in othbootargs env
    mountdev=mmcblk0p3
    # echo Using default mountdev: $mountdev
else
    mountdev=`echo $a | /bin/sed -r 's/.*(mount=[^ ]+) .*/\1/'`
    echo Using specified mountdev: $mountdev
fi

partnum=`echo $mountdev | /usr/bin/awk '{print substr($0,length())}'`
echo partnum: $partnum

MOUNT_P=/new_root
MOUNT_R=/mnt
mkdir ${MOUNT_P}

/bin/mknod /dev/$mountdev b 179 $partnum && /bin/mount -o rw /dev/$mountdev ${MOUNT_P}
if [ $? -ne 0 ];then
    echo "[INFO] mount sd card failed! reboot device!"
    reboot
    exit 0
fi

ls ${MOUNT_P} && ls ${MOUNT_P}/etc > /dev/null
if [ $? -ne 0 ];then
    echo "[INFO] No ${MOUNT_P} or ${MOUNT_P}/etc directory!! reboot device!"
    reboot
    exit 0
fi

ls ${MOUNT_P}/update
if [ $? -ne 0 ];then
    echo "[INFO] current is NXP busybox (primary image), switch to ubuntu (recovery)"
    ROOTFS_DIR="${MOUNT_P}"
else
    echo "[INFO] current is ubuntu (recovery image), update and switch to primary!"
    cd ${MOUNT_P}
    ROOTFS_PATH="${MOUNT_P}/sysroot/ostree/deploy/tcu/deploy"
    OSTREE_NEWEST_COMMIT=`ostree admin --sysroot=sysroot status | grep tcu | head -n1 | sed 's/tcu //g' | tr -d ' '`
    ROOTFS_DIR="${ROOTFS_PATH}/${OSTREE_NEWEST_COMMIT}"
    echo "[INFO] The rootfs real directory is : ${ROOTFS_DIR}"
    ls ${ROOTFS_DIR}
    if [ $? -ne 0 ];then
        echo "[INFO] no rootfs! Go to recovery!"
        ROOTFS_DIR="${MOUNT_P}"
    else
        echo "[INFO] found a new rootfs!"

    fi
fi

# switch_root will fail to function if newroot is not the root of a
# mount. If you want to switch root into a directory that does not
# meet this requirement then you can first use a bind-mounting
# trick to turn any directory into a mount point:
#       mount --bind $DIR $DIR
/bin/mount --bind ${ROOTFS_DIR} ${MOUNT_R}
echo "[INFO] mount move proc sys directories."
/bin/mount -o move /proc ${MOUNT_R}/proc
/bin/mount -o move /sys ${MOUNT_R}/sys

echo "[INFO] exec /bin/busybox switch_root ${MOUNT_R} /sbin/init"
sleep 1
exec /bin/busybox switch_root ${MOUNT_R} /sbin/init </dev/console >dev/console 2>&1

```

需要注意的是，这里有个非常关键的一点，就是使用switch_root来切换rootfs路径问题。由于ostree部署的路径是ramdisk挂载sd卡中子目录，并没有挂载到root（根）上面的路径。这个在switch_root的新版本中是不被允许的。参考：https://man7.org/linux/man-pages/man8/switch_root.8.html 中的note，

> **switch_root** will fail to function if _newroot_ is not the root of a
       mount. If you want to switch root into a directory that does not
       meet this requirement then you can first use a bind-mounting
       trick to turn any directory into a mount point.

如果用了sd卡基于挂载点的子目录，就会有如下错误：
```console
[    4.978901] Kernel panic - not syncing: Attempted to kill init! exitcode=0x00000100
[    4.986556] CPU: 2 PID: 1 Comm: busybox Not tainted 5.10.35 #1
[    4.992381] Hardware name: LS1046A RDB Board (DT)
[    4.997077] Call trace:
[    4.999522]  dump_backtrace+0x0/0x1a8
[    5.003176]  show_stack+0x18/0x68
[    5.006485]  dump_stack+0xd0/0x12c
[    5.009879]  panic+0x16c/0x334
[    5.012924]  do_exit+0x9ec/0xa08
[    5.016143]  do_group_exit+0x44/0xa0
[    5.019709]  __wake_up_parent+0x0/0x30
[    5.023451]  el0_svc_common.constprop.0+0x78/0x1a0
[    5.028233]  do_el0_svc+0x24/0x90
[    5.031540]  el0_svc+0x14/0x20
[    5.034584]  el0_sync_handler+0xb0/0xb8
[    5.038411]  el0_sync+0x178/0x180
[    5.041719] SMP: stopping secondary CPUs
[    5.045635] Kernel Offset: disabled
[    5.049114] CPU features: 0x0240022,21002000
[    5.053374] Memory Limit: none
[    5.056423] ---[ end Kernel panic - not syncing: Attempted to kill init! exitcode=0x00000100 ]---
```

