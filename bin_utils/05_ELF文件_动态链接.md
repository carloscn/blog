# 05_ELF文件_动态链接

Q1：为什么要使用动态链接？

* 第一，静态链接会占用更多的空间，如果a.elf和b.elf在操作系统里面都引用了libc的函数，那么使用静态链接a.elf和b.elf都会复制libc副本到内存中，**增大了内存和磁盘空间**。在内存中共享一个模块：节省内存，还可减少物理页面的换入换出，也可增加CPU缓存的命中率，因为不同进程间的数据和指令访问都集中在了同一个共享模块上。
* 第二，从程序发布角度，如果a.elf和b.elf文件共用的c.so文件出现了bug，那么c.so重新release，a.elf和b.elf也需要重新release，如果这里有个十分highlevel程序调用了多级库，每一级的修改都会要重新release，这无疑是痛苦的。
* 第三，可扩展性，动态链接可以实现可扩展性。开发者可以暴露出接口让用户来实现，实现了插件的功能。

Q2：动态链接有什么弊端？

* DLL hell现象，旧的动态链接和新的动态链接不兼容。
* 执行速度没有静态链接快。（这里涉及查表、重定位等多个过程）动态链接与静态链接相比，性能损失大约在5%以下[^1]，但这点性能损失用来换取程序在空间上的节省和程序构建和升级时的灵活性，是相当值得的。

Q3：Linux和windows操作系统动态链接有什么区别？

* Linux的ELF动态链接文件被称为动态共享对象（DSO, Dynamic shared objects），通常以so结尾；Windows中的动态链接被称为动态链接库(Dynamical linking library)。

## 1 gcc动态链接

### 1.1 生成和使用动态链接

这里举个例子如何生成动态链接库

```C
// libsum.c
#include "libsum.h"

int lib_sum(int a, int b) {
    return a + b;
}
```

```C
// main.c 
#include "libsum.h"
#include <stdio.h>
int main(void)
{
    int a = 0xf, b = 0x7;
    sleep(-1);
    return lib_sum(a, b);
}
```

`aarch64-none-elf-gcc -fPIC -shared -o libsum.so libsum.c -I .`

* -shared: 表示产生共享对象
* -fPIC: 表示地址无关代码

使用lib：

` aarch64-none-elf-gcc main.c libsum.so -o main.elf --specs=nosys.specs`

注意--specs=nosys.specs是arm的baremental环境里面特别要求的，与动态库无关。

运行应用程序：（需要指定LD_LIBRARY_PATH的路径）

`export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:./ && ./main.elf &`

### 1.2 研究动态链接的空间分布

使用`cat /proc/581540/maps`查看进程的的相关的内存映射，这里面主要涉及了3个so文件：

<img width="1049" alt="image-20220406194037155" src="https://user-images.githubusercontent.com/16836611/162553636-c54f2834-7c71-46dc-bbc9-267e9210687c.png">

* libc：里面引用了sleep之类的函数 
* libsum：自己实现的动态库
* ld-2.31:这个是linux动态链接器，动态链接用于管理动态库的映射。

再来看下libsum.so的装载属性`readelf -l libsum.so`

<img width="856" alt="image-20220406194510435" src="https://user-images.githubusercontent.com/16836611/162553642-18588762-22a1-4be3-9ed0-be9e013d6226.png">

Note，动态链接模块的装载地址是从0x0开始的，而真正的地址是0x7fd1e0cb1000，共享对象的最终装载地址在编译的时候不是确定的。

## 2 装载时重定位和地址无关代码

### 2.1 装载时重定位

操作系统需要根据内存空闲情况，动态分配一块物理内存给程序，所以程序被装载的地址是不确定的。系统在装载程序的时候需要对程序的指令和数据中绝对地址的引用进行重定位，基地址变换之后加上偏移量就能拿到重定位后的地址。`静态链接完成的是`**链接时重定位**，现在这种情况被叫做**装载时重定位**。

`gcc -shared a.c -o a.so`此时使用装载时重定位，我理解这也是作为so文件共享对象必然要有的属性，动态链接库必然根据每个依赖他的程序不同，装载时进行重定位。

### 2.2 地址无关代码

`还记得我们在ARCH64架构上面提出的LDR指令和ADRP指令`里面有个对于VMA和LMA加载地址时候崩溃的问题([1.4 ADRP和LDR**的陷阱**](https://github.com/carloscn/blog/issues/12))。里面设计了ADRP和LDR绝对地址和相对于PC地址的偏移的事情。实际上在动态链接里面完全可以看见这样设计的缘由。如果一个进程使用了动态链接，装载时重定位只是相当于把动态链接库的地址重定位了而已，实际上还是一种静态重定位。而如果多个进程同时使用动态链接，就没有动态链接宣称节约内存的优势，因此，必须要实现：**多个进程之间指令部分共用一份，数据自己独享的方式**，这里就引出了地址无关代码的技术。

地址无关代码技术我们在ARCH64指令集里面也可以看到，很多指令都是相对寻址（相对于PC）。我们按照程序员自我修养里面的方式，来研究一下aarch64上面有关4种情况：

* 模块内部的函数调用和跳转
* 模块内部的数据访问
* 模块外部的函数调用和跳转
* 模块外部的数据访问

```C
static int a;
extern int b;
extern void lib_sum();

void bar()
{
    a = 1;
    b = 2;
}

void main()
{
    bar();
    lib_sum(a, b);
}
```

这里面bar()，a是内部的函数和数据；lib_sum，b是外部的。我们编译一下`$ aarch64-none-elf-gcc -c exa.c libsum.so -o exa.o`。

使用objdump查看反汇编：`aarch64-none-elf-objdump -S -h exa.o`

```bash
$ aarch64-none-elf-objdump -S -h exa.o

0000000000400494 <lib_sum@plt>:
  400494:	90000090 	adrp	x16, 410000 <__FRAME_END__+0xf3e0>
  400498:	f9470211 	ldr	x17, [x16, #3584]
  40049c:	91380210 	add	x16, x16, #0xe00
  4004a0:	d61f0220 	br	x17
  
0000000000400668 <bar>:
  400668:	b0000080 	adrp	x0, 411000 <impure_data+0x1e0>
  40066c:	91168000 	add	x0, x0, #0x5a0
  400670:	52800021 	mov	w1, #0x1                   	// #1
  400674:	b9000001 	str	w1, [x0]
  400678:	b0000080 	adrp	x0, 411000 <impure_data+0x1e0>
  40067c:	9115a000 	add	x0, x0, #0x568
  400680:	52800041 	mov	w1, #0x2                   	// #2
  400684:	b9000001 	str	w1, [x0]
  400688:	d503201f 	nop
  40068c:	d65f03c0 	ret

0000000000400690 <main>:
  400690:	a9bf7bfd 	stp	x29, x30, [sp, #-16]!
  400694:	910003fd 	mov	x29, sp
  400698:	97fffff4 	bl	400668 <bar>
  40069c:	b0000080 	adrp	x0, 411000 <impure_data+0x1e0>
  4006a0:	91168000 	add	x0, x0, #0x5a0
  4006a4:	b9400002 	ldr	w2, [x0]
  4006a8:	b0000080 	adrp	x0, 411000 <impure_data+0x1e0>
  4006ac:	9115a000 	add	x0, x0, #0x568
  4006b0:	b9400000 	ldr	w0, [x0]
  4006b4:	2a0003e1 	mov	w1, w0
  4006b8:	2a0203e0 	mov	w0, w2
  4006bc:	97ffff76 	bl	400494 <lib_sum@plt>
  4006c0:	d503201f 	nop
  4006c4:	a8c17bfd 	ldp	x29, x30, [sp], #16
  4006c8:	d65f03c0 	ret
  4006cc:	d503201f 	nop
```

#### 2.2.1 GOT表

模块间的数据访问依托的是.GOT表（全局偏移表Global Offset Table），当代码引用外部的变量的时候，需要通过这个表来查找到偏移量。通过`objdump -h exa.so`
<img width="701" alt="image-20220408145917918" src="https://user-images.githubusercontent.com/16836611/162553650-662a2689-dd00-453e-8150-ab2b43346902.png">

再通过`objdump - R exa.so`
<img width="807" alt="image-20220408150006700" src="https://user-images.githubusercontent.com/16836611/162553652-3826dc72-7cf7-48b5-94b6-b0265809c846.png">

b的位置在0x10be0的位置，.got在0x10bd8，在其偏移量0x8的位置。

PIC的代码不包含任何代码段重定位表。

#### 2.2.2 PLT表

在调用外部库的时候`<lib_sum@plt>`有个符号plt，这个就是延迟绑定技术。这个目的是，可以提高动态链接库的执行速度。在ELF文件中，比如有很多分支里面调用的函数，很低的概率被用到，那么这个函数会被延迟绑定，就是这个函数第一次被用到的时候才会被绑定（查找、重定位）。

我们在ELF段结构里面看到GOT表，就分为两种：

* GOT表
* GOT.plt表

## 3 动态链接相关结构

### 3.1 动态链接器

我们在查看linux进程的/proc/xxx/maps的时候，里面会有ld.so，这个是由系统提供的一个动态链接器（Dynamic Linker）。启动一个包含动态链接的程序的步骤：

* 操作系统加载动态链接器
* 动态链接器初始化，根据环境变量对ELF执行动态链接
* 动态链接器把控制权交给可执行文件的入口函数

在`.interp`段指定ld.so的路径。通常Linux系统下，都是/lib/ld-linux.so.2这个文件。

在`.dynamic`段保存了动态链接器的基本信息，地址、哈希表地址、等等。可以用readelf -d xx.so命令查看。ldd命令还可以查看elf依赖了哪些库。

## Ref

[^1]:[静态链接和动态链接优缺点 ](https://blog.csdn.net/qq_20817327/article/details/108176847)