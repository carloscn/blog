# 08_ELF文件_运行库（入口、库、多线程）

## 1 函数入口

如果现阶段我们还认为函数入口是从main开始的，只能说，我们的在学习高中物理而并非大学物理。虽然没有绝对的否定，但是可以用不准确来形容。对于一个SoC的programmer来说，程序从main开始我觉得这是不允许的。我们必须知道程序真正的入口在哪里。这个函数，如果是我们按照正常的理解，那么后输出的是`main will return`，先输出的`my exit!`，但是事实并非如此。

```C
#include <stdio.h>
#include <stdlib.h>
void my_exit(void) {
    printf("my exit!\n");
}
int main(void) {
    atexit(&my_exit);
    printf("main will return\n");
    return 0;
}
```

<img src="https://raw.githubusercontent.com/carloscn/images/main/typoratyporaimage-20220424135226835.png" alt="image-20220424135226835" width="80%" align="center" />

程序在main返回之后，会记录main函数的返回值，调用atexit注册的函数，然后结束进程。可以看到，c语言并非简单的由main函数进行控制，在此之前肯定做了不少的工作。

**那么程序在main之前做了什么**？我们要知道，每个平台都不一样，x86和ARM的肯定是不一样的，但肯定有一些是共性的，对于ARM平台：bare-mental和有操作系统的也肯定是不一样。他们需要在这个阶段完成的使命是：

*   S1：操作系统（或baremental）创建进程之后，把控制权交到程序的入口，**这个入口往往是运行库中的某个程序入口**。（注意入口不是main，是Entry Point）
*   S2：运行库的Entry Point势必要对程序的环境进行初始化，包括stack、heap、I/O、线程、全局变量。
*   S3：call 我们程序中的main函数。
*   S4：main函数结束完毕之后，还会调到运行库的入口函数，入口函数做一些清理工作，包括全局变量的销毁、stack释放、heap释放、关闭IO、最后结束进程。

### 1.1 glibc入口函数

glibc的entry point是_start（这个入口是由ld链接器默认的链接脚本所指定的，我们也可以通过相关参数设定自己的入口 `ld -e`），我们看下libc里面怎么实现的：

**armv7平台的**：`glibc-2.35/sysdeps/arm/start.S`

**armv8平台的**：`glibc-2.35/sysdeps/aarch64/start.S`

我们先研究一下armv8平台的，armv7的可以参考[^2]

```assembly
/* This is the canonical entry point, usually the first thing in the text
   segment.

   Note that the code in the .init section has already been run.
   This includes _init and _libc_init


   At this entry point, most registers' values are unspecified, except:

   x0/w0	Contains a function pointer to be registered with `atexit'.
		This is how the dynamic linker arranges to have DT_FINI
		functions called for shared libraries that have been loaded
		before this code runs.

   sp		The stack contains the arguments and environment:
		0(sp)			argc
		8(sp)			argv[0]
		...
		(8*argc)(sp)		NULL
		(8*(argc+1))(sp)	envp[0]
		...
					NULL
 */

	.text
ENTRY(_start)
	/* Create an initial frame with 0 LR and FP */
	cfi_undefined (x30)
	mov	x29, #0
	mov	x30, #0

	/* Setup rtld_fini in argument register */
	mov	x5, x0

	/* Load argc and a pointer to argv */
	ldr	PTR_REG (1), [sp, #0]
	add	x2, sp, #PTR_SIZE

	/* Setup stack limit in argument register */
	mov	x6, sp

#ifdef PIC
# ifdef SHARED
        adrp    x0, :got:main
	ldr     PTR_REG (0), [x0, #:got_lo12:main]
# else
	adrp	x0, __wrap_main
	add	x0, x0, :lo12:__wrap_main
# endif
#else
	/* Set up the other arguments in registers */
	MOVL (0, main)
#endif
	mov	x3, #0		/* Used to be init.  */
	mov	x4, #0		/* Used to be fini.  */

	/* __libc_start_main (main, argc, argv, init, fini, rtld_fini,
			      stack_end) */

	/* Let the libc call main and exit with its return code.  */
	bl	__libc_start_main

	/* should never get here....*/
	bl	abort

#if defined PIC && !defined SHARED
	/* When main is not defined in the executable but in a shared library
	   then a wrapper is needed in crt1.o of the static-pie enabled libc,
	   because crt1.o and rcrt1.o share code and the later must avoid the
	   use of GOT relocations before __libc_start_main is called.  */
__wrap_main:
	BTI_C
	b	main
#endif
END(_start)

	/* Define a symbol for the first piece of initialized data.  */
	.data
	.globl __data_start
__data_start:
	.long 0
	.weak data_start
	data_start = __data_start

```

我们先看一下glibc入口的栈使用情况，我给画出来了，栈底有NULL，栈顶是argc，最先吐出来的也是argc，接着就是argv的参数了，最后还有envp环境变量的参数。还要注意的是，寄存器x0被占用，存放我们atexit(func_addr)里面func_addr的值。

<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220424143130337.png" alt="image-20220424143130337" width="70%" />

```C
void _start()
{
    int x0 = atexit_func_address;
    /* Create an initial frame with 0 LR and FP */
    int x29 = 0;   // lR
    int x30 = 0;   // fp
    /* Setup rtld_fini in argument register */
    int x5 = x0;   // rtld_fini

    /* Load argc and a pointer to argv */
    int argc = /* pop from stack */ sp[0];
    char ** argv = /* pop from stack*/ sp[1];

    __libc_start_main (main, argc, argv, init, fini, rtld_fini, stack_end)
}
```

这个_start函数提供了一些参数初始化，然后会调用运行库里面__libc_start_main函数[^1]：

>-   performing any necessary security checks if the effective user ID is not the same as the real user ID.
>-   initialize the threading subsystem.
>-   registering the `*rtld_fini*` to release resources when this dynamic shared object exits (or is unloaded).
>-   registering the `*fini*` handler to run at program exit.
>-   calling the initializer function `(**init*)()`.
>-   calling `main()` with appropriate arguments.
>-   calling `exit()` with the return value from `main()`.

__libc_start_main函数从此和平台没有任何关系了， 仅仅是调用的逻辑不同。__libc_start_main函数在[^3]，被定义成LIBC_START_MAIN在`glibc-2.35/csu`路径。

这里面有很多宏定义隔开的在不同场景下的，我们先看最基本的：

```
__pthread_initialize_minimal()
__cxa_atexit(rtld_fini, NULL, NULL)
__libc_init_first(argc, argv, __environ)
__cxa_atexit(fini, NULL, NULL)
(*init)(argc, argv, __environ)
```

传入的参数fini和rtld_fini用于main结束之后调用。还有个要注意的，atexit是一个链表结构，在exit时候会遍历每个链表：

```c
void exit(int status)
{
	while(__exit_funcs != NULL) {
		__exit_funcs = __exit_funcs->next;
	}
	__exit(status);
}
```

### 1.2 I/O初始化

这部分简单的来说，进程需要在用户空间建立stdin，stdout，stderr对应的FILE FD，让程序可以直接使用printf，scanf等。

## 2 运行库

glibc发布版本主要有三部分组成：

*   头文件：/usr/include
*   二进制文件部分
    *   动态部分：/lib/libc.so.6
    *   静态部分：/usr/lib/libc.a
*   辅助运行库
    *   /usr/lib/crt1.o
    *   /usr/lib/crti.o
    *   /usr/lib/ctrn.o

### 2.1 glibc启动文件

这个文件包含程序的入口函数_start，所以由它负责调用我们1节说的_libc_start_main，包含了基本的启动、退出代码，ctr1.o必须是链接器第一个输入的位置。elf文件这块对于ctr1.o也有要求，目标文件中引入了.init和.finit，运行库会保证所有位于这两个段中的代码会先于/后于main函数执行，所以他们用来实现全局构构造和全局析构。在ctr1.o中包含了一些辅助代码，（比如计算GOT之类的），因此引入了crti.o和ctrn.o两个文件。ctri.o和ctrn.o这两个目标文件中包含的代码实际上是_init()函数和_finit()函数开始和结尾的部分。链接器输入文件顺序是：

`ld crt1.o crti.o [user_objects] [system_libraries] crtn.o`

### 2.2 全局构造和解析

在main前调用函数，glibc的全局构造函数放置在.ctors段内部，因此如果我们手动在.ctors段里面加一些函数指针，就可以让这些函数在全局狗在函数时候（main）前调用：

```c
#include <stdio.h>
#include <stdlib.h>
typedef void (*ctor_t)(void);
void __attribute__((constructor)) my_init(void)
{
    printf("my init!\n");
}
void __attribute__((destructor)) my_exit(void) {
    printf("my exit!\n");
}
void atexit_func(void) {
    printf("atexit !\n");
}
int main(void) {
    atexit(&atexit_func);
    printf("main will return\n");
    return 0;
}
```

注意，由于全局对象的构建和析构都是由运行库来完成的，于是在程序或共享库中有全局对象的时候，不能使用`-nonstartfiles`和`-nostdlib选项`否则，编译不通过。

![image-20220424154657535](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220424154657535.png)

## Ref

[^1]:[Linux Standard Base Specification 3.1  - __libc_start_main ](https://refspecs.linuxbase.org/LSB_3.1.0/LSB-generic/LSB-generic/baselib---libc-start-main-.html)
[^2]:[Linux中誰來呼叫C語言中的main?  (armv7)](http://wen00072.github.io/blog/2015/02/14/main-linux-whos-going-to-call-in-c-language/)
[^3]:[Browse the source code of glibc - libc_start_main](https://code.woboq.org/userspace/glibc/csu/libc-start.c.html)

