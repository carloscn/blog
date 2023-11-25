
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

部署ostree的SD卡是在Host Linux完成。挂载SD卡：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202310191030483.png)

准备一个空的rootfs，格式化通过：

`sudo mkfs.ext4 -L "rootfs" -b 4096 /dev/sda3` 

SD卡会在host的linux生成rootfs文件夹：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202310191030200.png)

## 2.2 部署ostree

部署之前请检查你的板子：
* 联网功能（包括ip是否正确）
* 时间正确
* 足够的空间

### 2.2.1 创建标准的sysroot

初始化一个标准的sysroot文件夹：

`sudo ostree admin init-fs rootfs`

![](https://raw.githubusercontent.com/carloscn/images/main/typora202310191032911.png)

此时会生成一个最小的rootfs，**这个rootfs的作用是支持initramfs的最小启动**，而不是真正的rootfs。

在这个rootfs创建一个newroot文件夹，作为initramfs挂载真正rootfs的根文件夹和mnt文件夹，并创建update文件，作为initramfs检测update的标志。

`sudo mkdir -p rootfs/newroot && sudo mkdir -p rootfs/mnt && sudo touch rootfs/update`

最后的rootfs格式应该是：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202310191035955.png)


### 2.2.2 增加远程repo

给仓库增加

`sudo ostree remote add --repo=./rootfs/ostree/repo --no-gpg-verify origin http://10.10.192.121:8000/`

从远程仓库拉去数据：

`sudo ostree pull --repo=./rootfs/ostree/repo origin:master`

![](https://raw.githubusercontent.com/carloscn/images/main/typora202310191037056.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202310191044500.png)

`sync`

`sudo ostree log --repo=./rootfs/ostree/repo origin:master`

### 2.2.3 部署仓库

创建操作系统，本文叫做tcu：

`sudo ostree admin os-init tcu --sysroot=rootfs`

Note，如果不创建该操作系统，而是在deploy文件夹手动创建tcu的文件夹，selinux检查会不通过的。报错为：`# SELinux relabel /var` 参考 https://github.com/ostreedev/ostree/pull/872

![](https://raw.githubusercontent.com/carloscn/images/main/typora202310191044970.png)

部署tcu系统到rootfs

![](https://raw.githubusercontent.com/carloscn/images/main/typora202310191045147.png)

`sudo ostree admin --sysroot=rootfs --os=tcu deploy origin:master`

查看部署情况： `sudo ostree admin --sysroot=rootfs status`

部署之后的文件：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202310191045135.png)

在`./rootfs/ostree/deploy/tcu/deploy`目录就可以看到。

upgrade：

`sudo ostree admin --sysroot=rootfs --os=tcu upgrade`

通过这个命令，可以看到，ostree把最新的部署rootfs中的etc从/usr/etc拿了出来：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202310191046140.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora202310191046303.png)

之后可以通过一些脚本，在部署完成之后，拿到rootfs的地址，然后在ramdisk里面配合一些脚本解析进行切换。

### 2.2.4 ramdisk脚本

#### prepare-root

ostree提供了`prepare-root`的工具，主要功能参考： https://ostreedev.github.io/ostree/man/ostree-prepare-root.html 大致意思为，切换部署的sysroot里面。

这部分程序是：

https://github.com/ostreedev/ostree/blob/v2018.8/src/switchroot/ostree-prepare-root.c

其开发成为一个单独的工具，可以很简单的进行交叉编译。

https://github.com/carloscn/ls104x-bsp/tree/master/initramfs/prepare_root

``` bash
#!/bin/sh
set -eu

# -------------------------------------------

log_info() { echo "[INFO] $0[$$]: $*" >&2; }
log_error() { echo "[ERR] $0[$$]: ERROR $*" >&2; }

do_mount_fs() {
	log_info "mounting FS: $*"
	[[ -e /proc/filesystems ]] && { grep -q "$1" /proc/filesystems || { log_error "Unknown filesystem"; return 1; } }
	[[ -d "$2" ]] || mkdir -p "$2"
	[[ -e /proc/mounts ]] && { grep -q -e "^$1 $2 $1" /proc/mounts && { log_info "$2 ($1) already mounted"; return 0; } }
	mount -t "$1" "$1" "$2"
}

bail_out() {
	log_error "$@"
	log_info "Rebooting..."
	#exec reboot -f
	exec sh
}

get_ostree_sysroot() {
	for opt in $(cat /proc/cmdline); do
		arg=$(echo "$opt" | cut -d'=' -f1)
		if [ "$arg" == "ostree_root" ]; then
			echo "$opt" | cut -d'=' -f2-
			return
		fi
	done
	echo "LABEL=otaroot"
}

export PATH=/sbin:/usr/sbin:/bin:/usr/bin:/usr/lib/ostree

log_info "Starting OSTree initrd script"

do_mount_fs proc /proc
do_mount_fs sysfs /sys
do_mount_fs devtmpfs /dev
do_mount_fs devpts /dev/pts
do_mount_fs tmpfs /dev/shm
do_mount_fs tmpfs /run

# check if smack is active (and if so, mount smackfs)
grep -q smackfs /proc/filesystems && {
	do_mount_fs smackfs /sys/fs/smackfs

adjust current label and network label
	echo System >/proc/self/attr/current
	echo System >/sys/fs/smackfs/ambient
}

MOUNT_R=/sysroot
mkdir -p ${MOUNT_R}
ostree_sysroot=$(get_ostree_sysroot)

mount "$ostree_sysroot" ${MOUNT_R} || {
The SD card in the R-Car M3 takes a bit of time to come up
Retry the mount if it fails the first time
	log_info "Mounting $ostree_sysroot failed, waiting 5s for the device to be available..."
	sleep 5
	mount "$ostree_sysroot" ${MOUNT_R} || bail_out "Unable to mount $ostree_sysroot as physical sysroot"
}

ls ${MOUNT_R}
if [ $? -ne 0 ];then
    bail_out "No ${MOUNT_R} directory!! reboot device!"
fi

/usr/bin/ostree-prepare-root ${MOUNT_R}

log_info "exec /bin/busybox switch_root ${MOUNT_R} /sbin/init"
exec switch_root ${MOUNT_R} /sbin/init
sleep 1
bail_out "Failed to switch_root to ${MOUNT_R}"
```

同时uboot的env也需要配置，ramdisk会从uboot提取deploy部署的路径：

```
setenv bootargs "earlycon=uart8250,mmio,0x21c0500 console=ttyS0,115200 root=/dev/ram0 rw rootfstype=ext4 rootwait rootdelay=2 ostree=ostree/boot.0/tcu/0fe7a1e6cd1c37a4d82bbb83633d0bf6ecb8b8ec2b6131a32059549244582dd6/0 ostree_root=/dev/mmcblk0p3"
```