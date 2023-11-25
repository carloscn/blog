# 0x21_LinuxKernel_内核活动（一）之系统调用
[# _09_ELF文件_基于ARMv7的Linux系统调用原理_](https://github.com/carloscn/blog/issues/56) 指示了从处理器的角度出发，使用系统调用需要什么处理，本文将从Linux内核的角度来观察系统调用在操作系统逻辑需要的处理。在ARM处理器的系统调用，ARMv7提供了`SWI`指令让ARMv7处理器进入到了特权状态，以便能访问特权内存及使用特权指令。类似的，ARMv8提供了`SVC`指令。

从进程管理和调度角度而言，[# 0x24_LinuxKernel_进程（一）进程的管理（生命周期、进程表示）](https://github.com/carloscn/blog/issues/8)进程一直忙于与内核交互（包括请求系统资源、访问外设、与其他进程通信、读取文件），**这些操作不光需要Linux内核软件的逻辑设计，还需要上述CPU处理器级别的操作支持**。进程管理和调度是系统调用的最大用户，它需要最大效能的通过系统调用为进程们合理的分配处理器和系统的资源。在linux上为进程管理提供了标准库的接口，在标准库的实现中调用内核函数。因此，**系统调用被封装在标准库中**。

从Linux内核的角度，系统调用的过程有点复杂，这涉及了执行性态的转化（用户态->内核态->用户态）。在这期间不同的**虚拟地址和各种处理器特性，还有控制权如何在APP层和内核层交互，任务状态的通知与同步，参数和返回值的传递**。这是系统调用中非常重要的内容。

# 用户侧syscall

## 追踪系统调用
这里有个例子使用`strace`工具来追踪系统调用：
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <stdint.h>
#include <fcntl.h>

#define debug_log printf("%s:%s:%d--",__FILE__, __FUNCTION__, __LINE__);printf

// $host arm-linux-gnueabihf-gcc test_ptrace.c -o test_ptrace.elf
// $host scp -r test_ptrace.elf text.txt root@192.168.31.210:/home/root/
// $remote strace -o log.txt ./test_ptrace.elf
int main(int argc, char *argv[])
{
    int handle, bytes;
    void *ptr = NULL;

    handle = open("text.txt", O_RDONLY); // (1)
    ptr = (void*)malloc(150); // (2)
    if (ptr == NULL) {
        debug_log("malloc failed\n");
        return 0;
    }

    bytes = read(handle, ptr, 150);  // (3)
    debug_log("%s\n", (char *)ptr);

    close(handle);  // (4)
    free(ptr);  // (5)
    ptr = NULL;
    return 0;
}
```

我们在imx6（armv7）板子上进行运行程序：`strace -o log.txt ./test_ptrace.elf`

```text
root@ATK-IMX6U:~# cat log.txt
execve("./test_ptrace.elf", ["./test_ptrace.elf"], [/* 17 vars */]) = 0
brk(NULL)                               = 0xd1c000
uname({sysname="Linux", nodename="MiWiFi-RA67-srv", ...}) = 0
mmap2(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x76fed000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat64(3, {st_mode=S_IFREG|0644, st_size=63260, ...}) = 0
mmap2(NULL, 63260, PROT_READ, MAP_PRIVATE, 3, 0) = 0x76faf000
close(3)                                = 0
open("/lib/tls/v7l/neon/vfp/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat64("/lib/tls/v7l/neon/vfp", 0x7eca46b0) = -1 ENOENT (No such file or directory)
open("/lib/tls/v7l/neon/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat64("/lib/tls/v7l/neon", 0x7eca46b0) = -1 ENOENT (No such file or directory)
open("/lib/tls/v7l/vfp/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat64("/lib/tls/v7l/vfp", 0x7eca46b0)  = -1 ENOENT (No such file or directory)
open("/lib/tls/v7l/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat64("/lib/tls/v7l", 0x7eca46b0)      = -1 ENOENT (No such file or directory)
open("/lib/tls/neon/vfp/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat64("/lib/tls/neon/vfp", 0x7eca46b0) = -1 ENOENT (No such file or directory)
open("/lib/tls/neon/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat64("/lib/tls/neon", 0x7eca46b0)     = -1 ENOENT (No such file or directory)
open("/lib/tls/vfp/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat64("/lib/tls/vfp", 0x7eca46b0)      = -1 ENOENT (No such file or directory)
open("/lib/tls/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat64("/lib/tls", 0x7eca46b0)          = -1 ENOENT (No such file or directory)
open("/lib/v7l/neon/vfp/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat64("/lib/v7l/neon/vfp", 0x7eca46b0) = -1 ENOENT (No such file or directory)
open("/lib/v7l/neon/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat64("/lib/v7l/neon", 0x7eca46b0)     = -1 ENOENT (No such file or directory)
open("/lib/v7l/vfp/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat64("/lib/v7l/vfp", 0x7eca46b0)      = -1 ENOENT (No such file or directory)
open("/lib/v7l/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat64("/lib/v7l", 0x7eca46b0)          = -1 ENOENT (No such file or directory)
open("/lib/neon/vfp/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat64("/lib/neon/vfp", 0x7eca46b0)     = -1 ENOENT (No such file or directory)
open("/lib/neon/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat64("/lib/neon", 0x7eca46b0)         = -1 ENOENT (No such file or directory)
open("/lib/vfp/libc.so.6", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat64("/lib/vfp", 0x7eca46b0)          = -1 ENOENT (No such file or directory)
open("/lib/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\1\1\1\0\0\0\0\0\0\0\0\0\3\0(\0\1\0\0\0\254n\1\0004\0\0\0"..., 512) = 512
fstat64(3, {st_mode=S_IFREG|0755, st_size=1210000, ...}) = 0
mmap2(NULL, 1279344, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x76e76000
mprotect(0x76f99000, 65536, PROT_NONE)  = 0
mmap2(0x76fa9000, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x123000) = 0x76fa9000
mmap2(0x76fac000, 9584, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x76fac000
close(3)                                = 0
mmap2(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x76fec000
set_tls(0x76fec4c0, 0x76fecb98, 0x76fef050, 0x76fec4c0, 0x76fef050) = 0
mprotect(0x76fa9000, 8192, PROT_READ)   = 0
mprotect(0x76fee000, 4096, PROT_READ)   = 0
munmap(0x76faf000, 63260)               = 0
open("text.txt", O_RDONLY)              = 3
brk(NULL)                               = 0xd1c000
brk(0xd3d000)                           = 0xd3d000
read(3, "hello test pstrace testing text."..., 150) = 35
fstat64(1, {st_mode=S_IFCHR|0600, st_rdev=makedev(207, 16), ...}) = 0
ioctl(1, TCGETS, {B115200 opost isig icanon echo ...}) = 0
write(1, "test_ptrace.c:main:22--hello tes"..., 58) = 58
write(1, "\n", 1)                       = 1
close(3)                                = 0
exit_group(0)                           = ?
+++ exited with 0 +++
```

如果去进入内核中去review code去查看哪些系统调用是一个非常头疼的过程，使用这个工具会顺序化的将系统调用列出来。

我们上面的例子一共使用了5+个系统调用接口：
* (1) `open`：
* (2)`malloc`：malloc在内部执行`brk`系统调用
* (3)`read`
* (4)`write`
* (5)`close`

实际上`printf`里面也涉及了系统调用，使用write系统调用显示结果。

## 信号机制和系统调用

在使用信号机制和系统调用配合的时候要十分小心。因为在Linux设计上，为了提高信号机制的响应性能，采用了信号中断系统调用的设计。而Linux的系统调用即便是被信号打断，也会重启系统调用，这个方法被称为*可重启系统调用(restartable system call)*。该技术第一次被引入到System V的标准里面。

这个机制工作流程是：信号call进来之后，系统调用被迫停止，切换至信号执行程序。此时系统调用函数不会有返回值，内核在处理完信号处理程序之后，将自动重启被打断的系统调用。Linux通过配置信号为`SA_RESTART`标志来支持BSD方案。

我们可以做一个堵塞式`read(STDIN_FILENO)`和信号处理标志位判定的例子：

```C
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <stdint.h>
#include <fcntl.h>
#include <signal.h>

#define debug_log printf("%s:%s:%d--",__FILE__, __FUNCTION__, __LINE__);printf

// $host arm-linux-gnueabihf-gcc test_sig_syscall.c -o test_sig_syscall.elf
// $host scp -r test_sig_syscall.elf root@192.168.31.210:/home/root/test_sig_syscall.elf

static volatile int signaled = 0;

static void handler(int signum)
{
    debug_log("signaled called %d\n", signum);
    signaled = 1;
}

int main(int argc, char *argv[])
{
    char ch;
    struct sigaction sigact;
    sigact.sa_handler = handler;
    sigact.sa_flags = SA_RESTART;   // enable restart on system call
    sigaction(SIGINT, &sigact, NULL);

    while(read(STDIN_FILENO, &ch, 1) != 1 && !signaled);

    return 0;
}
```

如果启动`sigact.sa_flags = SA_RESTART`，那么信号处理机制将要启动restart设计，在read阻塞期间，有信号call进来之后会进入到handler处理程序里，接着read内会自动的重启这个请求。在用户侧的表现就是，while循环没有也因为signaled成为1而终止循环，而是需要再去从stdin中读取键盘的输入之后才能结束（这个为BSD行为）。若注释掉`sigact.sa_flags = SA_RESTART`，系统调用同样也会被信号中断进入到信号处理函数里面，但不会重启系统调用，而是退出了系统调用，相应的系统调用函数会返回`-EINTR`作为标识。在用户侧的表现是，ctrl-c信号发出之后，while循环因为signaled变量置位而立即退出了（这个为System V行为）。

# 可用的系统调用

linux提供了种种系统调用，我们可以总结如下的脑图：

![系统调用](https://raw.githubusercontent.com/carloscn/images/main/typora%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8.svg)

# 系统调用实现

从内核角度来看，系统调用比较复杂，至少包含：
* 问题1：用户空间和内核空间的虚拟地址意义转换的问题
* 问题2：APP控制权到内核控制权转换的问题（切入内核态，返回用户态，资源守护）
* 问题3：用户态到内核态处理器状态变换的问题（接洽不同平台的处理器）
* 问题4：参数传递和系统调用返回值的问题
* 问题5：系统调用完成之后如何通知用户空间的问题

## 处理函数实现

### handler函数

如图所示为系统调用的处理函数所在的位置，上面的所有问题都会在这个handler内部进行解决。handler处于kernelspace内部，因此可以调用内核函数。

![](https://raw.githubusercontent.com/carloscn/images/main/typora20220809133628.png)

处理函数是一类系统调用处理函数的统称，并不是一个程序，也不是所有的内核的handler都在一个文件或者文件集合中实现。这部分程序散落在内核的各个文件，例如文件系统就在fs/内核子目录下，内存管理的系统调用就在mm/内核子目录下。内核对这部分定义规则如下：

* 每个函数的前缀都是`sys_`，将该函数唯一地标识为一个系统调用；
* 所有的处理程序都最多接受5个参数；
* 所有的程序都在内核态执行。

进入内核空间之后，就是用的是内核的堆栈信息，在上面定义的结构体和指针在函数中直接可以访问获得。例如，获取当前进程的UID信息的`getuid()`系统调用：

```C
// kernel/timer.c

asmlinkage long sys_getuid(void) {
	/* Only we change this so SMP safe */
	return current->uid;
}
```

`current`是一个指针，指向的是进程的`task_struct`实例，**由内核自动设置current指针**。

注意，`asmlinkage`在<linkage.h>中定义，这个汇编宏什么都没有做，只是通知编译器该函数的特别调用规范。在这里汇编宏的出现可以引发我们注意了，这已经很靠近底层或者强烈依赖架构的硬件了。从用户态切换到内核态，以及调用分派和参数传递，都是由平台的汇编语言代码实现的。为了允许用户态和内核态的切换，用户进程必须通过一条专用的机器指令，引起处理器/内核对该进程关注。

### ARMv8的系统调用实现[^1]

在所有的平台上，**系统调用的参数都是通过寄存器直接传递的**，因为分为内核栈和用户栈，这两个是独立的栈，系统调用相比于普通调用特殊之处就是不能使用栈作为参数残敌。在ARM64上面通过`SVC`指令实现系统调用。我们可以举个例子：

```C
int main() { 
	char *s="Hello ARM64"; 
	write(1,s,strlen(s)); 
	exit(0); 
}
```
我们来看一下对应的反汇编：
```assembly
.data

/* Data segment: define our message string and calculate its length. */
helloworld:
    .ascii        "Hello, ARM64!\n"
helloworld_len = . - helloworld

.text

/* Our application's entry point. */
.globl _start
_start:
    /* syscall write(int fd, const void *buf, size_t count) */
    mov     x0, #1              /* fd := STDOUT_FILENO */
    ldr     x1, =helloworld     /* buf := msg */
    ldr     x2, =helloworld_len /* count := len */
    mov     w8, #64             /* write is syscall #64 */
    svc     #0                  /* invoke syscall */

    /* syscall exit(int status) */
    mov     x0, #0               /* status := 0 */
    mov     w8, #93              /* exit is syscall #1 */
    svc     #0                   /* invoke syscall */
```

`write`系统调用函数，参数传递为1、"hello arm64"、长度的三个参数。这三个参数分别被放在`X0, X1, X2`寄存器中。然后把64（write的系统调用唯一编号，后面会讲）给w8寄存器，然后使用`svc`指令让处理器进入到系统调用中。处理器会进行状态切换，查表系统调用表找到64号，从寄存器中dump出参数执行，完成系统调用。

### 系统调用表

sys_call_table保存一组指向处理器例程的函数指针，可用于查找处理程序。因为该表是用汇编语言指令在内核的数据段中产生的，其内容因不同平台而不同。但原理都是一致的，根据系统调用的编号找到表中适合的位置，由此获取指向目标的处理程序函数的指针。系统调用列表可以参考[^2]

ARM64的表在：https://elixir.bootlin.com/linux/v3.19.8/source/include/uapi/asm-generic/unistd.h

ARMv7处理器的表在：https://elixir.bootlin.com/linux/v3.19.8/source/arch/arm/kernel/calls.S

我们这里就举几个例子：
```C
/* 0 */		CALL(sys_restart_syscall)
		CALL(sys_exit)
		CALL(sys_fork)
		CALL(sys_read)
		CALL(sys_write)
/* 5 */		CALL(sys_open)
		CALL(sys_close)
		CALL(sys_ni_syscall)		/* was sys_waitpid */
		CALL(sys_creat)
		CALL(sys_link)
/* 10 */	CALL(sys_unlink)
		CALL(sys_execve)
		CALL(sys_chdir)
		CALL(OBSOLETE(sys_time))	/* used by libc4 */
		CALL(sys_mknod)
/* 15 */	CALL(sys_chmod)
		CALL(sys_lchown16)
		CALL(sys_ni_syscall)		/* was sys_break */
		CALL(sys_ni_syscall)		/* was sys_stat */
		CALL(sys_lseek)
/* 20 */	CALL(sys_getpid)
		CALL(sys_mount)
		CALL(OBSOLETE(sys_oldumount))	/* used by libc4 */
		CALL(sys_setuid16)
		CALL(sys_getuid16)
/* 25 */	CALL(OBSOLETE(sys_stime))
		CALL(sys_ptrace)
		CALL(OBSOLETE(sys_alarm))	/* used by libc4 */
		CALL(sys_ni_syscall)		/* was sys_fstat */
		CALL(sys_pause)
/* 30 */	CALL(OBSOLETE(sys_utime))	/* used by libc4 */
		CALL(sys_ni_syscall)		/* was sys_stty */
		CALL(sys_ni_syscall)		/* was sys_getty */
		CALL(sys_access)
		CALL(sys_nice)
/* 35 */	CALL(sys_ni_syscall)		/* was sys_ftime */
		CALL(sys_sync)
		....
		....
		....
```

### 用户态返回
每个系统调用都MUST通知到用户进程，是否执行了相关的系统调用程序，执行结果如何（返回值）。在函数的普通调用中，一个函数返回值经常被使用。系统调用函数从表面看，也是通过函数的返回值来告诉用户执行结果如何的，但这个看上去并没有普通函数的调用返回值那么简单。原因上面说了，普通函数在相同的栈内执行，通过栈空间传递数据即可，而用户态返回后同样是内核栈切到用户栈，彼此独立的栈成了传递参数的最大阻碍。因此，要做到普通函数调用的返回值的形式，**必然通过处理器上的寄存器来完成**。

ARM64上面的系统调用退出也是通过系统调用`exit`完成。

```assembly
    /* syscall exit(int status) */
    mov     x0, #0               /* status := 0 */
    mov     w8, #93              /* exit is syscall #1 */
    svc     #0                   /* invoke syscall */
```

从核心态切回到用户态，把返回值放在内核栈上，然后从内核栈拷贝值到寄存器中。

### 内核态访问用户空间

在设计上，内核尽可能的保持内核空间和用户空间的独立，但有些情况，内核代码必须要访问到用户应用程序的虚拟内存。但是这就引发了一个问题，用户态切换到内核态的时候相应的栈已经切换为内核栈，内存管理里面的虚拟地址也换成了内核的虚拟地址空间映射。因此，内核访问用户空间并不是随心所欲的，也只能是内核执行由用户应用程序发起的同步操作才有意义。

什么时候不得不在用户空间发起内核访问自己的情形呢？
* 传递参数超过6个的时候，必须借助C结构体实例来传。系统调用借助寄存器传递一个存储参数的指针传递给内核。内核使用指针来访问的是用户空间的内存。
* 由系统调用的副效应产生大量的数据，不能通过返回值机制传递到用户进程。必须通过内存共享的方式来交换数据。当然这个共享的数据空间必然是在用户侧，否则用户无法访问这段内存。

所以为了解决用户态的虚拟地址被切换成内核态的虚拟地址，内核无法访问到用户态的地址空间（不能直接反引用用户空间的指针），linux提供特定的函数，确保用户空间的内存区域还存在映射。用户空间通过`__user`属性标记，以支持C check tools的源代码自动化检查。

在用户空间和内核空间之间复制数据的函数。大多数情况下，是`copy_to_user`和`copy_from_user`。


# REF


[^1]:[How to implement system call in ARM64?](https://www.cnblogs.com/dream397/p/15632453.html)
[^2]:[https://thog.github.io/syscalls-table-aarch64/latest.html](https://thog.github.io/syscalls-table-aarch64/latest.html)
[^3]:[]()
[^4]:[]()
[^5]:[]()
[^6]:[]()
[^7]:[]()
[^8]:[]()
[^9]:[]()
[^10]:[]()
[^11]:[]()
[^12]:[]()
[^13]:[]()
[^14]:[]()
[^15]:[]()
[^16]:[]()
[^17]:[]()
[^18]:[]()
[^19]:[]()
[^20]:[]()