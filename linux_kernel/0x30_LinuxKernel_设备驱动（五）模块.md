# 0x30_LinuxKernel_设备驱动（五）模块
Linux内核是采用宏内核的架构，操作系统中大部分的功能都是在内核中，比如进程管理、内存管理、进程调度之类的，并且这些机制是要在特权模式下运行。与之对应的概念是微内核，微内核顾名思义，就是在内核中仅放入最基础的功能，而在内核外部放在非特权的模式下运行。因此，微内核有天生的可扩展性。

Linux内核在发展过程中也使用了可扩展的机制，这种机制叫做Loadable Kernel Module。可在内核运行的时候加载一组目标代码来实现某个特定的功能。而不需要重新编译内核来实现动态扩展。

Linux内核通过内核模块来实现动态添加和删除某个功能。我们先从驱动开发者角度来看如何操作模块，然后我们再从内核的角度看对模块实现和支持Linux内核做了什么操作。

# 1. 驱动开发者角度

## 1.1 C程序
从一个hello world程序开始：
```C
#include <linux/init.h>
#include <linux/module.h>

static int __init my_test_init(void)
{
    printk("my first kernel module init\n");
    return 0;
}

static void __exit my_test_exit(void)
{
    printk("carlos test goodbye.\n");
}

module_init(my_test_init);
module_exit(my_test_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("carlos wei");
MODULE_DESCRIPTION("this is the hello kernel driver");
MODULE_ALIAS("mytest");
```

从程序上有几个要素：
* `#include <linux/init.h>` 和`module.h`文件，这部分包含了后面注册模块的`module_init`和`exit`函数的声明；
* MODULE_LICENSE这些会在`$ modinfo`中显示；
* 在`$insmod`的时候系统会调用注册的`init`函数；
* 在`$rmmod`的时候系统会调用`exit`函数；

## 1.2 Makefile
在Makefile上，模块的Makefile在使用上有些不同。在x86架构下面：
```makefile
BASEINCLUDE	?= /lib/modules/`uname -r`/build
#BASEINCLUDE	?= /home/carlos/workspace/work/linux-imx-rel_imx_4.1.15_2.1.0_ga
myhello-objs	:=	hello.o
obj-m	:=	myhello.o

all	:
	$(MAKE) -C $(BASEINCLUDE) M=$(PWD) modules;
clean	:
	$(MAKE) -C $(BASEINCLUDE) M=$(PWD) clean;
	rm -rf *.ko
```

在ARM使用交叉编译的Makefile：

```makefile
BASEINCLUDE	?= /home/carlos/workspace/work/linux-imx-rel_imx_4.1.15_2.1.0_ga
myhello-objs	:=	hello.o
obj-m	:=	myhello.o

all	:
	$(MAKE) ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -C $(BASEINCLUDE) M=$(PWD) modules;
clean	:
	$(MAKE) ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -C $(BASEINCLUDE) M=$(PWD) clean;
	rm -rf *.ko
```

这里面圈出一些比较重要的配置：
* 文件需要编译成.o的文件：`<模块名>-objs := <目标文件名>.o`
* 从.o文件编译成.ko文件：`obj-m := <模块名>.o`
* 使用`make`命令编译：
	![](https://raw.githubusercontent.com/carloscn/images/main/typora20220922191707.png)

## 1.3 管理ko文件的几个方法

### 1.3.1 file查看类型
`file myhello.ko`

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220922191822.png)
可以看出这个ko文件是一个elf格式的LSB的可重定位的二进制文件

### 1.3.2 modinfo
`modinfo myhello.ko`
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220922191929.png)

很多信息都是在我们ko文件中表示出来的。

### 1.3.3 加载和删除ko文件
`sudo insmod myhello.ko` 然后使用 `dmesg`就可以看到信息：
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220922192113.png)
调用了init函数。

`sudo rmmod myhello`（可以省略ko）
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220922192200.png)

调用了exit函数。

### 1.3.4 lsmod查看已经加载的模块
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220922192334.png)

### 1.3.5 在/sys/module查看模块信息
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220922192507.png)

### 1.3.6 modprobe[^1]

在Linux操作系统上，modprobe命令可从Linux内核中添加和删除模块。但是操作方式和功能和`insmod`有些不同。对于有“符号共享”的模块而言，modprobe可以解决依赖问题。例如，乙模块export出一个函数sum，甲模块依赖于这个函数，如果使用insmod需要先安装乙模块，再去安装甲模块，否则会出现符号找不到的问题。使用modprobe就会帮助解决这个问题，但是我们要在使用之前做一些建立依赖关系的操作。

**modprobe**使用**depmod**生成的依赖关系列表和硬件映射来将模块智能地加载或卸载到内核中。它分别使用较低级的程序**insmod**和**rmmod**进行实际的插入和删除。虽然可以手动调用**insmod**和**rmmod**，但建议使用**depmod**加载和卸载模块，以确保在进行更改之前考虑任何模块间的依赖关系。

#### blacklist
除`/etc/modprobe.d`目录中的可选配置文件外，所有模块和其他文件均适用。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923084820.png)

黑名单长这个样子：
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923084916.png)

我们可以在`/etc/modprobe.d`下面创建自己的黑名单文件，只要是以conf结尾的文件，都会被收录。我们创建一个blacklist-carlos.conf，里面blacklist mysum 和 blacklist myhello，接着使用 `modprobe --showconfig | grep blacklist`可以查看到我们的设置。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923090710.png)

或者通过命令行的模式：`modprobe.blacklist=modname1,modname2`

### 1.3.7 使用

首先需要把编译好的ko文件统一链接或者直接拷贝到`/lib/modules/uname -r`的目录中，确保probe能够找到。我们这里使用链接的方式 `sudo ln -s pwd/myhello.ko /lib/modules/5.4.0-125-generic/hello`

接着建立依赖关系 `sudo depmod -a`

最后我们就可以不用依赖sum，直接加载myhello了。

`sudo modprobe myhello`

使用dmsg就可以看到，模块先init了myhello依赖的sum模块，然后再加载自己。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220923092400.png)

最后使用rmmod或者`modprobe -r xxx`都可以，但是rmmod要自己解除依赖，modprobe -r可以把由于依赖预先加载的模块也移除掉。

## 1.4 模块参数
内核作为可扩展的模块，为Linux提供了灵活性。但是，有的时候我们需要根据不同的应用场景给内核传递不同的参数，Linux内核提供了一个宏定义来实现模块的参数传递。

### 1.4.1 使用参数

#### 定义单个参数
```c
//1.  定义变量(像定义普通变量的方式一样定义变量)。
//2.  使用后module_param 声明模块变量如module_param(param_name,param_type,RW Authority )。
//eg:
static char*   param_name="xxxxxxx"
module_param(param_name,charp,S_IRUGO)
```
#### 定义参数组
```c
//1.  定义数组变量(像定义普通数组变量的方式一样定义变量)。
//2.  使用后module_param_array 声明模块变量
//eg:
static int  param_name[cnt]="xxxxxxx"
module_param_array(param_name,cnt,charp,S_IRUGO)
```
使用`MODUEL_PARAM_DESC()`宏对参数做一些简单的说明。

#### 参数类型
-   byte 字节参数
-   short 有符号半字参数
-   ushort 无符号半字参数
-   int 有符号整形参数
-   uint 无符号整形参数
-   long 有符号长整形参数
-   ulong 无符号长整形参数
-   charp 字符指针
-   bool bool值
#### 权限
可以使用Linux定义的权限宏或运算组合，也可以使用权限位的数字如0644来表示。
-   S_IRUSR  
    Permits the file's owner to read it.
-   S_IWUSR  
    Permits the file's owner to write to it.
-   S_IRGRP  
    Permits the file's group to read it.
-   S_IWGRP  
    Permits the file's group to write to it

#### example
```C
#include <linux/init.h>
#include <linux/module.h>

static int debug = 1;
module_param(debug, int, 0644);
MODULE_PARM_DESC(debug,"carlos enable debug function\n");

#define carlos_debug_printk(args...)                \
        if (debug) {                                \
            printk(KERN_DEBUG args);                \
        }                                           \

static int __init my_test_init(void)
{
    printk("my first kernel module init\n");
    carlos_debug_printk("init : carlos_debug_printk to debug ko param\n");
    return 0;
}

static void __exit my_test_exit(void)
{
    printk("carlos test goodbye.\n");
    carlos_debug_printk("exit : carlos_debug_printk to debug ko param\n");
}

module_init(my_test_init);
module_exit(my_test_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("carlos wei");
MODULE_DESCRIPTION("this is the hello kernel driver");
MODULE_ALIAS("mytest");
```

我们这里定义一个debug变量，然后这个debug变量可以通过外部的参数传递进来，然后定义了carlos_debug_printk的宏定义，依赖于debug变量。如果在insmod的时候没有任何的指定，那么debug值就是1，就会输出我们打印的init和exit的两个log。
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220922194112.png)

我们通过`sudo insmod myhello.ko debug=0`来传递参数：
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220922194153.png)

就会传递debug = 0这个值，然后在dmsg中就看不到两个log了：
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220922194300.png)

另外，可以通过调试参数来关闭和打开调试信息：
在`/sys/module/myhello/parameters`中就可以看到：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220922194426.png)

#### 1.4.2 符号共享
我们在编写驱动的时候，会把驱动程序哪找功能分好几个内核模块，这些内核之间需要一些接口函数来互相调用，这个怎么实现？Linux提供了两个宏来解决模块之间的互相调用的问题。

```C
EXPORT_SYMBOL()
EXPORT_SYMBOL_GPL()
```

* 第一个是把符号对全部的内核代码公开，也就是将以符号的方法导出给内核的其他模块使用；
* 其中EXPORT_SYMBOL_GPL()只包含GPL许可的模块，内核核心大部分都是以这个形式导出来的，因此必须显式的声明模块为GPL才可以导出来。

内核的导出符号可以通过 `sudo cat /proc/kallsyms` 来查看
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220922200503.png)

我们来做个示例，开发一个sum.c模块，实现一个加法功能，然后由hello.c调用：

```C
// sum.c
#include <linux/init.h>
#include <linux/module.h>

static int __init sum_init(void)
{
    printk("my first kernel module init\n");
    return 0;
}

static void __exit sum_exit(void)
{
    printk("carlos test goodbye.\n");
}

static int sum(int a, int b)
{
    return a + b;
}

int export_api_sum(int a, int b)
{
    return sum(a, b);
}

EXPORT_SYMBOL_GPL(export_api_sum);
module_init(sum_init);
module_exit(sum_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("carlos wei");
MODULE_DESCRIPTION("this is the sum kernel driver");
MODULE_ALIAS("sum");
```

这里使用导出符号，导出export_api_sum。在使用的模块需要extern到他的模块里面。

```C
#include <linux/init.h>
#include <linux/module.h>


extern int export_api_sum(int a, int b);

static int debug = 1;
module_param(debug, int, 0644);
MODULE_PARM_DESC(debug,"carlos enable debug function\n");

#define carlos_debug_printk(args...)                \
        if (debug) {                                \
            printk(KERN_DEBUG args);                \
        }                                           \

static int __init my_test_init(void)
{
    int a = 10, b = 20;
    printk("my first kernel module init\n");
    a = export_api_sum(a, b);
    printk("sum is %d\n", a);
    carlos_debug_printk("init : carlos_debug_printk to debug ko param\n");
    return 0;
}

static void __exit my_test_exit(void)
{
    printk("carlos test goodbye.\n");
    carlos_debug_printk("exit : carlos_debug_printk to debug ko param\n");
}

module_init(my_test_init);
module_exit(my_test_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("carlos wei");
MODULE_DESCRIPTION("this is the hello kernel driver");
MODULE_ALIAS("mytest");
```

Make也需要动动手脚：
```Makefile
BASEINCLUDE	?= /lib/modules/`uname -r`/build
#BASEINCLUDE	?= /home/carlos/workspace/work/linux-imx-rel_imx_4.1.15_2.1.0_ga
mysum-objs	:= sum.o
myhello-objs	:=	hello.o

obj-m	+=  mysum.o
obj-m	+=	myhello.o

all	:
	$(MAKE) -C $(BASEINCLUDE) M=$(PWD) modules;
clean	:
	$(MAKE) -C $(BASEINCLUDE) M=$(PWD) clean;
	rm -rf *.ko
```

编译完成之后，必须要先加载被依赖函数的模块，否则会找不到符号，导致加载不成功。
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220922200347.png)

先加载mysum.ko模块，后加载myhello.ko，成功的使用了这个函数。
![](https://raw.githubusercontent.com/carloscn/images/main/typora20220922200432.png)


# 2. 内核角度



# Ref
[^1]:[modprobe (可从Linux内核中添加和删​​除模块)](https://www.uc23.net/command/452.html)