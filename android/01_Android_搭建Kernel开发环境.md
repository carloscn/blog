# 1. 构建Android系统

## 1.1 源代码下载

因为国外地址可能访问不了或者访问速度慢，我们需要添加国内数据源：

```
sudo gedit /etc/apt/sources.list
```

把清华源复制到源列表，之前的源不要动

```
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-updates main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-backports main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-security main restricted universe multiverse
deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ focal-security main restricted universe multiverse
```

保存之后：

sudo apt-get update sudo apt-get upgrade

配置好国内的数据源之后，我们就可以开始源代码的下载了，网速要尽量快，因为AOSP的源代码光压缩包就有441G。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202311081247212.png)

谷歌给的网站 [https://source.android.com/docs/security/features/trusty/download-and-build?hl=zh-cn](https://source.android.com/docs/security/features/trusty/download-and-build?hl=zh-cn) 的网络不好。我们可以使用清华镜像：

可访问 [https://cs.android.com](https://cs.android.com) 或 [https://github.com/aosp-mirror](https://github.com/aosp-mirror) 在线搜索及浏览 AOSP 源码。

参考 Google 教程 [https://source.android.com/setup/build/downloading，](https://source.android.com/setup/build/downloading%EF%BC%8C) 将 [https://android.googlesource.com/](https://android.googlesource.com/) 全部使用 [https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/](https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/) 代替即可。

由于 AOSP 镜像造成CPU/内存负载过重，限制了并发数量，因此建议：

- sync的时候并发数不宜太高，否则会出现 503 错误，即-j后面的数字不能太大，**建议选择4**。 请尽量选择流量较小时错峰同步。 过程摘录 (参考 [https://lug.ustc.edu.cn/wiki/mirrors/help/aosp](https://lug.ustc.edu.cn/wiki/mirrors/help/aosp) 编写)
    
- 下载 repo 工具:

`mkdir ~/bin`

`PATH=~/bin:$PATH`

`curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo`

`chmod a+x ~/bin/repo` 或者使用tuna的git-repo镜像

使用每月更新的初始化包 由于首次同步需要下载约 60GB 数据，过程中任何网络故障都可能造成同步失败，我们强烈建议您使用初始化包进行初始化。

下载 `https://mirrors.tuna.tsinghua.edu.cn/aosp-monthly/aosp-latest.tar`[，下载完成后记得根据](https://mirrors.tuna.tsinghua.edu.cn/aosp-monthly/aosp-latest.tar%EF%BC%8C%E4%B8%8B%E8%BD%BD%E5%AE%8C%E6%88%90%E5%90%8E%E8%AE%B0%E5%BE%97%E6%A0%B9%E6%8D%AE) checksum.txt 的内容校验一下。

由于所有代码都是从隐藏的 .repo 目录中 checkout 出来的，所以我们只保留了 .repo 目录，下载后解压 再 repo sync 一遍即可得到完整的目录。

使用方法如下:

`curl -OC - https://mirrors.tuna.tsinghua.edu.cn/aosp-monthly/aosp-latest.tar # 下载初始化包 tar xf aosp-latest.tar`

`cd AOSP` # 解压得到的 AOSP 工程目录

这时 ls 的话什么也看不到，因为只有一个隐藏的 .repo 目录

`repo sync` # 正常同步一遍即可得到完整目录或 `repo sync -l` 仅checkout代码

此后，每次只需运行 repo sync 即可保持同步。 我们强烈建议您保持每天同步，并尽量选择凌晨等低峰时间

> 传统初始化方法 建立工作目录:
> 
> `mkdir WORKING_DIRECTORY`
> 
> `cd WORKING_DIRECTORY` 初始化仓库:
> 
> `repo init -u https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/platform/manifest` 如果提示无法连接到 [gerrit.googlesource.com](http://gerrit.googlesource.com)，请参照git-repo的帮助页面的更新一节。
> 
> 如果需要某个特定的 Android 版本(列表)：
> 
> `repo init -u https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/platform/manifest -b android-4.0.1_r1` 同步源码树（以后只需执行这条命令来同步）：

建立 AOSP 镜像需要占用约 **850G** 磁盘。

## 1.2 编译AOSP

Note, 根据一些教程提供的经验值可能需要编译6个小时以上，实际编译1~2个小时；

安装依赖库，AOSP编译需要依赖很多库，而这些Ubuntu系统可能没有自带，因此我们需要提前安装，避免编译到一半报错浪费时间。
参考： https://source.android.com/docs/setup/start/initializing?hl=zh-cn

`sudo apt-get install git-core gnupg flex bison build-essential zip curl zlib1g-dev libc6-dev-i386 libncurses5 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig`

初始化编译环境，执行编译命令：

```bash
source build/envsetup.sh
```

输入`lunch`：

```
$ lunch

You're building on Linux

Lunch menu .. Here are the common combinations:
     1. aosp_arm-trunk_staging-eng
     2. aosp_arm64-trunk_staging-eng
     3. aosp_barbet-trunk_staging-userdebug
     4. aosp_bluejay-trunk_staging-userdebug
     5. aosp_bluejay_car-trunk_staging-userdebug
     6. aosp_bramble-trunk_staging-userdebug
     7. aosp_bramble_car-trunk_staging-userdebug
     8. aosp_cf_arm64_auto-trunk_staging-userdebug
     9. aosp_cf_arm64_phone-trunk_staging-userdebug
     10. aosp_cf_riscv64_phone-trunk_staging-userdebug
     11. aosp_cf_x86_64_auto-trunk_staging-userdebug
     12. aosp_cf_x86_64_auto_mdnd-trunk_staging-userdebug
     13. aosp_cf_x86_64_foldable-trunk_staging-userdebug
     14. aosp_cf_x86_64_only_phone_hsum-trunk_staging-userdebug
     15. aosp_cf_x86_64_pc-trunk_staging-userdebug
     16. aosp_cf_x86_64_phone-trunk_staging-userdebug
     17. aosp_cf_x86_64_tv-trunk_staging-userdebug
     18. aosp_cf_x86_phone-trunk_staging-userdebug
     19. aosp_cf_x86_tv-trunk_staging-userdebug
     20. aosp_cheetah-trunk_staging-userdebug
     21. aosp_cheetah_car-trunk_staging-userdebug
     22. aosp_cheetah_hwasan-trunk_staging-userdebug
     23. aosp_cloudripper-trunk_staging-userdebug
     24. aosp_coral-trunk_staging-userdebug
     25. aosp_coral_car-trunk_staging-userdebug
     26. aosp_felix-trunk_staging-userdebug
     27. aosp_flame-trunk_staging-userdebug
     28. aosp_flame_car-trunk_staging-userdebug
     29. aosp_lynx-trunk_staging-userdebug
     30. aosp_oriole-trunk_staging-userdebug
     31. aosp_oriole_car-trunk_staging-userdebug
     32. aosp_panther-trunk_staging-userdebug
     33. aosp_panther_car-trunk_staging-userdebug
     34. aosp_panther_hwasan-trunk_staging-userdebug
     35. aosp_raven-trunk_staging-userdebug
     36. aosp_raven_car-trunk_staging-userdebug
     37. aosp_ravenclaw-trunk_staging-userdebug
     38. aosp_redfin-trunk_staging-userdebug
     39. aosp_redfin_car-trunk_staging-userdebug
     40. aosp_redfin_vf-trunk_staging-userdebug
     41. aosp_slider-trunk_staging-userdebug
     42. aosp_sunfish-trunk_staging-userdebug
     43. aosp_sunfish_car-trunk_staging-userdebug
     44. aosp_tangorpro-trunk_staging-userdebug
     45. aosp_tangorpro_car-trunk_staging-userdebug
     46. aosp_trout_arm64-trunk_staging-userdebug
     47. aosp_trout_x86_64-trunk_staging-userdebug
     48. aosp_whitefin-trunk_staging-userdebug
     49. aosp_x86-trunk_staging-eng
     50. aosp_x86_64-trunk_staging-eng
     51. arm_krait-trunk_staging-eng
     52. arm_v7_v8-trunk_staging-eng
     53. armv8-trunk_staging-eng
     54. armv8_cortex_a55-trunk_staging-eng
     55. armv8_kryo385-trunk_staging-eng
     56. car_ui_portrait-trunk_staging-userdebug
     57. car_x86_64-trunk_staging-userdebug
     58. db845c-trunk_staging-userdebug
     59. gsi_car_arm64-trunk_staging-userdebug
     60. gsi_car_x86_64-trunk_staging-userdebug
     61. hikey-trunk_staging-userdebug
     62. hikey64_only-trunk_staging-userdebug
     63. hikey960-trunk_staging-userdebug
     64. hikey960_tv-trunk_staging-userdebug
     65. hikey_tv-trunk_staging-userdebug
     66. poplar-trunk_staging-eng
     67. poplar-trunk_staging-user
     68. poplar-trunk_staging-userdebug
     69. qemu_trusty_arm64-trunk_staging-userdebug
     70. rb5-trunk_staging-userdebug
     71. riscv64-trunk_staging-eng
     72. sdk_car_arm64-trunk_staging-userdebug
     73. sdk_car_md_x86_64-trunk_staging-userdebug
     74. sdk_car_portrait_x86_64-trunk_staging-userdebug
     75. sdk_car_x86_64-trunk_staging-userdebug
     76. silvermont-trunk_staging-eng
     77. uml-trunk_staging-userdebug
     78. yukawa-trunk_staging-userdebug
     79. yukawa_sei510-trunk_staging-userdebug

Which would you like? [aosp_arm-trunk_staging-eng]
Pick from common choices above (e.g. 13) or specify your own (e.g. aosp_barbet-eng): 

```

这里选择`73` 或者在运行lunch时候直接指定参数：

`lunch 73` or `lunch sdk_car_md_x86_64-trunk_staging-userdebug`

https://source.android.com/docs/setup/build/building?hl=zh-cn#choose-a-target

使用 `lunch` 选择要构建的目标。`lunch product_name-build_variant` 会选择 product_name 作为需要构建的产品，并选择 build_variant 作为需要构建的变体，然后将这些选择存储在环境中，以便供后续对 `m` 和其他类似命令的调用读取:

* 通过lunch指令设置编译目标，就是生成的镜像要运行在什么样的设备上，我这里是准备在模拟器上跑所以选择73(即sdk_car_md_x86_64-trunk_staging-userdebug)，模拟器是x86_64的平台，运行速度会更快。auto的意思是车载系统。
* 编译目标的格式:BUILD-BUILDTYPE，比如上面的aosp_x86_64-eng的BUILD是aosp_x86_64，BUILDTYPE是eng。BUILD指的是特定功能的组合的特定名称，即表示编译出的镜像可以运行在什么环境。其中，aosp(Android Open Source Project)代表Android开源项目；arm表示系统是运行在arm架构的处理器上，arm64则是指64位arm架构处理器，x86则表示x86架构的处理器。BUILD TYPE则指的是编译类型,通常有三种：

|构建类型|使用情况|
|---|---|
|user|权限受限；适用于生产环境|
|userdebug|与“user”类似，但具有 root 权限和调试功能；是进行调试时的首选编译类型|
|eng|具有额外调试工具的开发配置|

https://source.android.com/docs/setup/build/running?hl=zh-cn#selecting-device-build

如需查看当前的启动设置，请运行以下命令：
`echo "$TARGET_PRODUCT-$TARGET_BUILD_VARIANT"`

使用 `m` 构建所有内容。`m` 可以使用 `-jN` 参数处理并行任务。如果您没有提供 `-j` 参数，构建系统会自动选择您认为最适合您系统的并行任务计数。

```shell
m
```

- **`droid`** - `m droid` 是正常 build。此目标在此处，因为默认目标需要名称。
- **`all`** - `m all` 会构建 `m droid` 构建的所有内容，加上不包含 `droid` 标记的所有内容。构建服务器会运行此命令，以确保包含在树中且包含 `Android.mk` 文件的所有元素都会构建。
- **`m`** - 从树的顶部运行构建系统。这很有用，因为您可以在子目录中运行 `make`。如果您设置了 `TOP` 环境变量，它便会使用此变量。如果您未设置此变量，它便会从当前目录中查找相应的树，以尝试找到树的顶层。您可以通过运行不包含参数的 `m` 来构建整个源代码树，也可以通过指定相应名称来构建特定目标。
- **`mma`** - 构建当前目录中的所有模块及其依赖项。
- **`mmma`** - 构建提供的目录中的所有模块及其依赖项。
- **`croot`** - `cd` 到树顶部。
- **`clean`** - `m clean` 会删除此配置的所有输出和中间文件。此内容与 `rm -rf out/` 相同。

## 1.3 运行模拟器

在编译完成之后，就可以通过以下命令运行Android模拟器了，命令如下：

```shell
source build/envsetup.sh
lunch(选择刚才你设置的目标版本，比如这里我选择的是31)
emulator
```

如果你是在编译完后立刻运行模拟器，由于我们之前已经执行过source及lunch命令了，因此现在你只需要执行命令就可以运行模拟器：

```shell
emulator
```

这里如果你的虚拟机没有开启虚拟化功能，是不能直接跑模拟器的，会报错，这时，需要关闭虚拟机，配置虚拟化选项（例如VMWARE中的这个配置）。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202311081300006.png)

如果启动不来或者模拟器黑屏可以打开kernel的启动log：

`$ emulator -verbose -show-kernel -shell`

![](https://raw.githubusercontent.com/carloscn/images/main/typora202311081303500.png)

terminal:

![](https://raw.githubusercontent.com/carloscn/images/main/typora202311081303070.png)
