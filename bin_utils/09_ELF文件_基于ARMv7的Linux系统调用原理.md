# *09_ELF文件_基于ARMv7的Linux系统调用原理*

Linux的应用程序在运行的时候，本质是和linux kernel不断交互的过程，而userspace和kernel之间的通信就是通过系统调用**（system call）**来完成的。Linux的内部有300多个系统调用，这些系统调用被定义在`/usr/include/unistd.h`中。我们在学习Linux应用程序的时候，使用文件句柄可以包含file头文件使用fread，fwrite等函数，但也可以使用read、write，这里就可以说出他们两个区别，**fwrite这些函数都是glibc提供的函数，而read，write这些都是系统调用的接口**，在glibc里面底层也是调用系统调用的read、write来实现的。

**系统调用存在一些弊端：**

*   使用不便，使用系统调用的接口就要求程序员具备一些操作系统的知识。
*   操作系统之间不兼容，兼容性还需要程序员来开发。

为了解决这些弊端，引入了运行库，运行库相当于在系统调用上面增加一个兼容层，很多在操作系统需要处理和配置的工作都放在运行库中进行处理。使用运行库的优点可以总结为：

*   使用简便
*   形式统一，运行库叫做标准库，凡是能作为运行库的，必然要遵循某种标准。

但是运行库还是存在诸多缺点：

*   平台兼容性，比如XWindows的界面程序，在linux中，CRT只能把这部分省略。

# 1 系统调用原理

从硬件层面，系统调用时需要CPU做一些支持的，在CPU上面需要**用户模式**（user mode）和**特权模式**（kernel mode）的区分，因此，cpu需要将两种模式区分开，需要提供特权指令和特权执行的环境。在操作系统层面，就需要逻辑地将kernel划分成为用户态和内核态，当一个应用程序运行的时候，自己的业务逻辑是在用户态运行的，而当对于一些内核数据的访问，就需要使用系统调用，临时地从用户态切入内核态。

现代操作系统通过**中断（interrupt，在armv8中称为异常exception**），来从用户态切换到内核态，这个过程依赖于异常处理，这部分和[11_ARMv8_异常处理（二）- Legacy 中断处理](https://github.com/carloscn/blog/issues/49)非常类似。

<div align='center'>
<img src="https://raw.githubusercontent.com/carloscn/images/main/typoratelegram-cloud-document-5-6318986229266253504.jpg" alt="image-20220506140125504" width="90%" />
</div>

x86架构下，把中断分为两种，一种是硬件中断，由外部的硬件中断线触发；还有一种是软中断，通常使用一条指令，int + 中断号向cpu申请中断。系统调用在x86架构下使用的是int指令的软中断实现的。

armv7/armv8架构下和x86有很大的不同，armv7/armv8的异常中断种类中有复位，未定义指令，**软件中断（SWI）**，指令预取终止，IRQ，IFQ。系统调用通过中断指令，可由用户模式下的程序调用特权操作指令。

| Exception                    | Description                                                  |
| ---------------------------- | ------------------------------------------------------------ |
| Reset                        | Occurs when the processor reset pin is asserted. This exception is only expected to occur for signalling power-up, or for resetting as if the processor has just powered up. A soft reset can be done by branching to the reset vector (`0x0000`). |
| Undefined Instruction        | Occurs if neither the processor, or any attached coprocessor, recognizes the currently executing instruction. |
| **Software Interrupt (SWI)** | **This is a user-defined synchronous interrupt instruction.It allows a program running in User mode, for example, to request privileged operations that run in Supervisor mode, such as an RTOS function.** |
| Prefetch Abort               | Occurs when the processor attempts to execute an instruction that was not fetched, because the address was illegal[[1](https://developer.arm.com/documentation/#ftn.id2457941)]. |
| Data Abort                   | Occurs when a data transfer instruction attempts to load or store data at an illegal addressa. |
| IRQ                          | Occurs when the processor external interrupt request pin is asserted (LOW) and the I bit in the CPSR is clear. |
| FIQ                          | Occurs when the processor external fast interrupt request pin is asserted (LOW) and the F bit in the CPSR is clear. |

**还需要注意的是，该异常的优先级是最低的，是6。**

| Vector address | Exception type               | Exception mode       | Priority (1=high, 6=low) |
| -------------- | ---------------------------- | -------------------- | ------------------------ |
| `0x0`          | Reset                        | Supervisor (SVC)     | 1                        |
| `0x4`          | Undefined Instruction        | Undef                | 6                        |
| **`0x8`**      | **Software Interrupt (SWI)** | **Supervisor (SVC)** | **6**                    |
| `0xC`          | Prefetch Abort               | Abort                | 5                        |
| `0x10`         | Data Abort                   | Abort                | 2                        |
| `0x14`         | *Reserved*                   | *Not applicable*     | *Not applicable*         |
| `0x18`         | Interrupt (IRQ)              | Interrupt (IRQ)      | 4                        |
| `0x1C`         | Fast Interrupt (FIQ)         | Fast Interrupt (FIQ) | 3                        |

我们以armv7为例子，说一下SWI异常中断的处理过程。SWI包括一个24位的立即数，这个立即数指示用户特定SWI的功能。通常SWI异常处理的中断分为2级。第一级，SWI异常中断处理程序为汇编程序，用于确定SWI指令中的24位的立即数；第2级具体实现SWI的各个功能，可以是汇编也可以是C程序。[^1]

```assembly
Vector_Init_Block
                LDR    PC, Reset_Addr
                LDR    PC, Undefined_Addr
                LDR    PC, SWI_Addr
                LDR    PC, Prefetch_Addr
                LDR    PC, Abort_Addr
                NOP                     ;Reserved vector
                LDR    PC, IRQ_Addr
                LDR    PC, FIQ_Addr
Reset_Addr      DCD    Start_Boot
Undefined_Addr  DCD    Undefined_Handler
SWI_Addr        DCD    SWI_Handler
Prefetch_Addr   DCD    Prefetch_Handler
Abort_Addr      DCD    Abort_Handler
                DCD    0                ;Reserved vector
IRQ_Addr        DCD    IRQ_Handler
FIQ_Addr        DCD    FIQ_Handler
```

**SWI handlers in assembly language**

```assembly
    CMP    r0,#MaxSWI          ; Range check
    LDRLS  pc, [pc,r0,LSL #2]
    B      SWIOutOfRange
SWIJumpTable
    DCD    SWInum0
    DCD    SWInum1
                    ; DCD for each of other SWI routines
SWInum0             ; SWI number 0 code
    B    EndofSWI
SWInum1             ; SWI number 1 code
    B    EndofSWI
                    ; Rest of SWI handling code
                    ;
EndofSWI
                    ; Return execution to top level 
                    ; SWI handler so as to restore
                    ; registers and return to program.
```

**SWI handlers in C and assembly language**

```c
BL    C_SWI_Handler     ; Call C routine to handle the SWI

void C_SWI_handler (unsigned number)
{ 
    switch (number)
    {case 0 :                /* SWI number 0 code */
        break;
    case 1 :                 /* SWI number 1 code */
        break;
    :
    :
    default :                /* Unknown SWI - report error */
    }
}
```

**Using SWIs in Supervisor mode**

```assembly
    STMFD    sp!,{r0-r3,r12,lr}   ; Store registers.
    MOV      r1, sp               ; Set pointer to parameters.
    MRS      r0, spsr             ; Get spsr.
    STMFD    sp!, {r0}            ; Store spsr onto stack. This is only really needed in case of
                                  ; nested SWIs.
        ; the next two instructions only work for SWI calls from ARM state.
        ; See Example 5.17 for a version that works for calls from either ARM or Thumb.
    LDR      r0,[lr,#-4]          ; Calculate address of SWI instruction and load it into r0.
    BIC      r0,r0,#0xFF000000    ; Mask off top 8 bits of instruction to give SWI number.
        ; r0 now contains SWI number
        ; r1 now contains pointer to stacked registers
    BL       C_SWI_Handler        ; Call C routine to handle the SWI.
    LDMFD    sp!, {r0}            ; Get spsr from stack.
    MSR      spsr_cf, r0          ; Restore spsr.
    LDMFD    sp!, {r0-r3,r12,pc}^ ; Restore registers and return.
```

具体参考[^1]中armv7对于特权的处理。

# 2. 虚拟系统调用

## 2.1 链接形态

如果使用ldd来获取一个可执行文件动态库的依赖情况，`ldd /bin/ls`可能会发现一个比较奇怪的现象（以下图片是用armv7的imx6.u截取），这个和程序员自我修养一本书上有点差异，在程序员自我修养那个书里面使用的linux内核2.5版本，而在新的内核里面就变更linux-gate.so为linux-vdso.so了[^2]。而且映射的地址也对不上了。

![telegram-cloud-document-5-6318986229266253514](https://raw.githubusercontent.com/carloscn/images/main/typoratelegram-cloud-document-5-6318986229266253514.jpg)

linux-vdso.so.1没有和任何文件关联。这个是linux用于支持新型系统调用的“虚拟共享”库（virtual dynamic shared library, VDSO）。这个库被加载到了0x7efa9000地址上面，我们可以通过`cat /proc/self/maps`来查看一个可执行程序的内存映像：

![telegram-cloud-document-5-6318986229266253513](https://raw.githubusercontent.com/carloscn/images/main/typoratelegram-cloud-document-5-6318986229266253513.jpg)

这里面VDSO的设计是一个非常巧妙的设计，里面也有个小故事[^3]，向Linux内核里面添加一个功能，glibc的代价极大，可能需要做很多讨论才可以，而且还有BSD，sysV各种标准。而内核也要适配glibc，双方都背负沉重的历史包袱。因此，linuxer设计者考虑一个非常巧妙的设计，让libc变为以动态链接的形式进入内核。

>可以将vdso看成一个shared objdect file（这个文件实际上不存在）,内核将其映射到某个地址空间，被所有程序所共享。（我觉得这里用到了一个技术：多个虚拟页面映射到同一个物理页面。即内核把vdso映射到某个物理页面上，然后所有程序都会有一个页表项指向它，以此来共享，这样每个程序的vdso地址就可以不相同了）

## 2.2 **什么是VODS**[^4]

*   一个内核提供的成熟的动态链接文件。
*   被内核映射进入所有的用户进程。
*   与一般的.so文件链接规则都是一样的，但是gdb可能会不支持，`The one gdb used to complain about! (warning: Could not load shared library symbols for linux-vdso.so.1)`
*   提供system call的一种手段，可以理解为虚拟的系统调用。

## 2.3 内核和用户空间建立

![image-20220507155806532](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220507155806532.png)

![image-20220507160027590](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220507160027590.png)

关于vsdo的数据，我们可以从数据结构定义处拿到：

```c
struct vdso_data {
        __u64 cs_cycle_last ; /* Timebase at clocksource i n i t */
        __u64 raw_time_sec ; /* Raw time */
        __u64 raw_time_nsec ;
        __u64 xtime_clock_sec ; /* Kernel time */
        __u64 xtime_clock_nsec ;
        __u64 xtime_coarse_sec ; /* Coarse time */
        __u64 xtime_coarse_nsec ;
        __u64 wtm_clock_sec ; /* Wall to monotonic time */
        __u64 wtm_clock_nsec ;
        __u32 tb_seq_count ; /* Timebase sequence counter */
        __u32 cs_mono_mult ; /* NTP−adjusted clocksource multiplier */
        __u32 cs_shift ; /* Clocksource s h i f t (mono = raw) */
        __u32 cs_raw_mult ; /* Raw clocksource multiplier */
        __u32 tz_minuteswest ; /* Whacky timezone stuff */
        __u32 tz_dsttime ;
        __u32 use_syscall ;
} ;
```

在x86上面，vdso导出了一系列函数，比如__kernel_vsyscall函数，这个函数负责虚拟系统调用，这个函数里面会有内核之中调用特权指令，对于armv7请参考，[Using SWIs in Supervisor mode](https://developer.arm.com/documentation/dui0056/d/handling-processor-exceptions/swi-handlers/using-swis-in-supervisor-mode)[^5]，在特权模式下使用SWI异常。

在man手册里面可以找到aarch64 (armv8)系统调用的符号[^6]：

```
   ARM functions
       The table below lists the symbols exported by the vDSO.

       symbol                 version
       ────────────────────────────────────────────────────────────
       __vdso_gettimeofday    LINUX_2.6 (exported since Linux 4.1)
       __vdso_clock_gettime   LINUX_2.6 (exported since Linux 4.1)

       Additionally, the ARM port has a code page full of utility
       functions.  Since it's just a raw page of code, there is no ELF
       information for doing symbol lookups or versioning.  It does
       provide support for different versions though.

       For information on this code page, it's best to refer to the
       kernel documentation as it's extremely detailed and covers
       everything you need to know:
       Documentation/arm/kernel_user_helpers.txt.

   aarch64 functions
       The table below lists the symbols exported by the vDSO.

       symbol                   version
       ──────────────────────────────────────
       __kernel_rt_sigreturn    LINUX_2.6.39
       __kernel_gettimeofday    LINUX_2.6.39
       __kernel_clock_gettime   LINUX_2.6.39
       __kernel_clock_getres    LINUX_2.6.39
```

使用这些函数来实现虚拟系统调用，我们这里就不具体展开到内核讨论了，这部分会在内核里面记录。

# Ref

[^1]:[ARM Developer Suite Developer Guide  - Using SWIs in Supervisor mode](https://developer.arm.com/documentation/dui0056/d/handling-processor-exceptions/swi-handlers/using-swis-in-supervisor-mode)
[^2]:[The story of linux-{gate,vdso}.so](https://perezdecastro.org/old/the-story-of-linux-gatevdsoso.html)
[^3]:[linux-vdso.so.1介绍 ](https://blog.csdn.net/wang_xya/article/details/43985241)
[^4]:[The vDSO on arm64](https://blog.linuxplumbersconf.org/2016/ocw/system/presentations/3711/original/LPC_vDSO.pdf)
[^5]:[ARM Developer Suite Developer Guide  - Using SWIs in Supervisor mode ](https://developer.arm.com/documentation/dui0056/d/handling-processor-exceptions/swi-handlers/using-swis-in-supervisor-mode)
[^6]:[vdso(7) — Linux manual page](https://man7.org/linux/man-pages/man7/vdso.7.html)