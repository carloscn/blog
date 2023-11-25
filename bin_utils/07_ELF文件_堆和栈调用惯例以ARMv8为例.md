# 07_ELF文件_堆和栈调用惯例以ARMv8为例

## 1 栈与调用惯例

### 1.1 栈的概念

栈和堆的概念非常重要，程序员的修养是以x86架构讲的堆栈的概念，我们以ARMv8 AArch64为主来研究一下堆栈。

<img src="https://raw.githubusercontent.com/carloscn/images/main/typorafa0e9d749b2a79012.png" alt="fa0e9d749b2a79012" width="40%" />

栈的概念我们可以重力翻转之后的桌子上的一摞书为例子，栈顶就是最下面眼镜的位置，栈底就是桌子。栈的顺序就是我们最后放的眼镜，是先被拿出来的。栈（stack）是一种数据结构，计算机里面的栈使用栈数据结构管理内存。为什么要将“重力翻转”？因为栈是一种从高地址向低地址生长的存储结构，栈底对应高地址，栈顶对应低地址。

<img src="https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220419103804560.png" alt="image-20220419103804560" width="67%" />

这里的SP被称为“堆栈帧（Stack Frame）”或者“活动记录（Activate Record）”。堆栈帧会保存以下记录：

* 函数返回地址和参数，如果传递参数≤8个，那么使用X0~X7通用寄存器来传递，当参数多于8个，需要使用栈来传递参数。
* 临时变量，例如局部变量
* 保存上下文

### 1.2 不同架构出栈和入栈

**入栈过程：**

<img src="https://raw.githubusercontent.com/carloscn/images/main/typoratelegram-cloud-photo-size-5-6266933850319990826-y.jpg" alt="telegram-cloud-photo-size-5-6266933850319990826-y" width="80%" />

A32指令集提供了PUSH和POP指令来实现入栈和出栈[^1]，但是A64指令集已经去掉了PUSH和POP指令，只需要复用stp和ldp指令就可以实现入栈和出栈[^2]。

For example:

```assembly
// Broken AArch64 implementation of `push {x1}; push {x0};`.
  str   x1, [sp, #-8]!  // This works, but leaves `sp` with 8-byte alignment ...
  str   x0, [sp, #-8]!  // ... so the second `str` will fail.
```

In this particular case, the stores could be combined:

```assembly
// AArch64 implementation of `push {x0, x1}`.
  stp   x0, x1, [sp, #-16]!
```

However, in a simple compiler, it is not always easy to combine instructions in that way.

If you're handling `w` registers, the problem will be even more apparent: these have to be pushed in sets of four to maintain stack pointer alignment, and since this isn't possible in a single instruction, the code can become difficult to follow. This is what [VIXL](https://github.com/armvixl/vixl/blob/330dc71/src/a64/macro-assembler-a64.cc#L1255) generates, for example:

```assembly
// AArch64 implementation of `push {w0, w1, w2, w3}`.
  stp   w0, w1, [sp, #-16]!   // Allocate four words and store w0 and w1 at the lower addresses.
  stp   w2, w3, [sp, #8]      // Store w2 and w3 at the upper addresses.
```

这里AArch64实现入栈和出栈操作：

```assembly
.globalmain
main:
		/* 栈往下扩展16个字节 */
		stp x29, x30, [sp, #-16]!
		
		/* 把栈继续往下扩展8字节 */
		add sp, sp, #-8
		mov x8, #1
		/* 保存x8到SP */
		str x8, [sp]
		
		/* 释放刚才扩展的8字节的栈空间 */
		add sp, sp, #8
		
		/* main函数返回0 */
		mov w0, 0
		
		/* 恢复x29和x30寄存器的值，使SP指向原位置 */
		ldp x29, x30, [sp], #16
		ret
```

### 1.3 fomit-frame-pointer

使用aarch64-none-elf-gcc编译器参数`-fomit-frame-pointer`可以取消帧指针：

* 好处：不使用任何帧指针，直接计算变量的位置
* 坏处：无法trace，寻址变慢

![image-20220419122840960](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20220419122840960.png)

使用fomit-frame-pointer的反汇编可以看到，123行sp已经不会备份到x29。

### 1.4  调用惯例Call convention

函数调用方和被调用方需要按照统一的协议去压栈和出栈，否则会有问题。调用惯例

* 函数参数的传递顺序和方式
* 栈的维护方式
* 名字修饰，默认是 _cdecl `__attribute__((cdecl))`

#### 1.4.1 函数参数压栈和出栈

我们定义一个这样的函数，有30个参数，看看arm编译器如何处理参数的压栈和出栈，另外对参数的类型也需要有观察。

```c
static int s11( long a1,
                char a2,
                int a3,
                int a4,
                int a5,
                int a6,
                int a7,
                int a8,
                int a9,
                int a10
                )
{
    return  \
    a1 +    \
    a2 +    \
    a3 +    \
    a4 +    \
    a5 +    \
    a6 +    \
    a7 +    \
    a8 +    \
    a9 +    \
    a10;
}

int call_stack(void) {
    int a = s11(1,2,3,4,5,6,7,8,9,10);
    return a;
}
```

这段函数的反汇编是：

##### A64: ARMv8 AArch64

```assembly
00000000000000d0 <s11>:
  d0:   d100c3ff        sub     sp, sp, #48
  d4:   f90017e0        str     x0, [sp, #40] 		//a1
  d8:   39009fe1        strb    w1, [sp, #39]			//a2
  dc:   b90023e2        str     w2, [sp, #32]			//a3
  e0:   b9001fe3        str     w3, [sp, #28]			//a4
  e4:   b9001be4        str     w4, [sp, #24]			//a5
  e8:   b90017e5        str     w5, [sp, #20]     //a6
  ec:   b90013e6        str     w6, [sp, #16]     //a7
  f0:   b9000fe7        str     w7, [sp, #12]     //a8
  f4:   39409fe0        ldrb    w0, [sp, #39]     // load a2 from stack
  f8:   f94017e1        ldr     x1, [sp, #40]     // load a1 from stack
  fc:   0b010001        add     w1, w0, w1        // a1 + a2
 100:   b94023e0        ldr     w0, [sp, #32]     // +a3
 104:   0b000021        add     w1, w1, w0
 108:   b9401fe0        ldr     w0, [sp, #28]     // +a4
 10c:   0b000021        add     w1, w1, w0
 110:   b9401be0        ldr     w0, [sp, #24]     // ....
 114:   0b000021        add     w1, w1, w0
 118:   b94017e0        ldr     w0, [sp, #20]
 11c:   0b000021        add     w1, w1, w0
 120:   b94013e0        ldr     w0, [sp, #16]
 124:   0b000021        add     w1, w1, w0
 128:   b9400fe0        ldr     w0, [sp, #12]
 12c:   0b000021        add     w1, w1, w0
 130:   b94033e0        ldr     w0, [sp, #48]     // load a9 from stack 
 134:   0b000021        add     w1, w1, w0        // +a9
 138:   b9403be0        ldr     w0, [sp, #56]     // load a10 from stack
 13c:   0b000020        add     w0, w1, w0        // +a10
 140:   9100c3ff        add     sp, sp, #48
 144:   d65f03c0        ret

0000000000000148 <call_stack>:
 148:   d100c3ff        sub     sp, sp, #48
 14c:   a9017bfd        stp     x29, x30, [sp, #16]
 150:   910043fd        add     x29, sp, #0x10
 154:   52800140        mov     w0, #0xa                        // #10
 158:   b9000be0        str     w0, [sp, #8]
 15c:   52800120        mov     w0, #0x9                        // #9
 160:   b90003e0        str     w0, [sp]
 164:   52800107        mov     w7, #0x8                        // #8
 168:   528000e6        mov     w6, #0x7                        // #7
 16c:   528000c5        mov     w5, #0x6                        // #6
 170:   528000a4        mov     w4, #0x5                        // #5
 174:   52800083        mov     w3, #0x4                        // #4
 178:   52800062        mov     w2, #0x3                        // #3
 17c:   52800041        mov     w1, #0x2                        // #2
 180:   d2800020        mov     x0, #0x1                        // #1
 184:   97ffffd3        bl      d0 <s11>
 188:   b9002fe0        str     w0, [sp, #44]
 18c:   b9402fe0        ldr     w0, [sp, #44]
 190:   a9417bfd        ldp     x29, x30, [sp, #16]
 194:   9100c3ff        add     sp, sp, #48
 198:   d65f03c0        ret
```

##### A32: ARMv7 AArch32

```assembly
000000b0 <s11>:
  b0:   b480            push    {r7}
  b2:   b085            sub     sp, #20
  b4:   af00            add     r7, sp, #0
  b6:   60f8            str     r0, [r7, #12]
  b8:   607a            str     r2, [r7, #4]
  ba:   603b            str     r3, [r7, #0]
  bc:   460b            mov     r3, r1
  be:   72fb            strb    r3, [r7, #11]
  c0:   7afa            ldrb    r2, [r7, #11]
  c2:   68fb            ldr     r3, [r7, #12]
  c4:   441a            add     r2, r3
  c6:   687b            ldr     r3, [r7, #4]
  c8:   441a            add     r2, r3
  ca:   683b            ldr     r3, [r7, #0]
  cc:   441a            add     r2, r3
  ce:   69bb            ldr     r3, [r7, #24]
  d0:   441a            add     r2, r3
  d2:   69fb            ldr     r3, [r7, #28]
  d4:   441a            add     r2, r3
  d6:   6a3b            ldr     r3, [r7, #32]
  d8:   441a            add     r2, r3
  da:   6a7b            ldr     r3, [r7, #36]   ; 0x24
  dc:   441a            add     r2, r3
  de:   6abb            ldr     r3, [r7, #40]   ; 0x28
  e0:   441a            add     r2, r3
  e2:   6afb            ldr     r3, [r7, #44]   ; 0x2c
  e4:   4413            add     r3, r2
  e6:   4618            mov     r0, r3
  e8:   3714            adds    r7, #20
  ea:   46bd            mov     sp, r7
  ec:   f85d 7b04       ldr.w   r7, [sp], #4
  f0:   4770            bx      lr
  f2:   bf00            nop

000000f4 <call_stack>:
  f4:   b580            push    {r7, lr}
  f6:   b088            sub     sp, #32
  f8:   af06            add     r7, sp, #24
  fa:   2300            movs    r3, #0
  fc:   607b            str     r3, [r7, #4]
  fe:   2308            movs    r3, #8
 100:   603b            str     r3, [r7, #0]
 102:   230a            movs    r3, #10
 104:   9305            str     r3, [sp, #20]
 106:   2309            movs    r3, #9
 108:   9304            str     r3, [sp, #16]
 10a:   2308            movs    r3, #8
 10c:   9303            str     r3, [sp, #12]
 10e:   2307            movs    r3, #7
 110:   9302            str     r3, [sp, #8]
 112:   2306            movs    r3, #6
 114:   9301            str     r3, [sp, #4]
 116:   2305            movs    r3, #5
 118:   9300            str     r3, [sp, #0]
 11a:   2304            movs    r3, #4
 11c:   2203            movs    r2, #3
 11e:   2102            movs    r1, #2
 120:   2001            movs    r0, #1
 122:   f7ff ffc5       bl      b0 <s11>
 126:   6078            str     r0, [r7, #4]
 128:   687a            ldr     r2, [r7, #4]
 12a:   683b            ldr     r3, [r7, #0]
 12c:   4413            add     r3, r2
 12e:   4618            mov     r0, r3
 130:   3708            adds    r7, #8
 132:   46bd            mov     sp, r7
 134:   bd80            pop     {r7, pc}
 136:   bf00            nop
```

<img src="https://raw.githubusercontent.com/carloscn/images/main/typoratelegram-cloud-photo-size-5-6266933850319990918-y.jpg" alt="telegram-cloud-photo-size-5-6266933850319990918-y" width="70%" />

前8个参数被压入寄存器中，后面的参数被直接压到栈中。返回参数被放在x0中，返回地址在x30中。

#### 1.4.2 函数调用压栈和出栈

```C
static int s0(void) {
    return 0;
}

static int s1(void) {
    return s0();
}

static int s2(void) {
    return s1();
}
static int s3(void) {
    return s2();
}
static int s4(void) {
    return s3();
}
static int s5(void) {
    return s4();
}
static int s6(void) {
    return s5();
}
static int s7(void) {
    return s6();
}
static int s8(void) {
    return s7();
}
static int s9(void) {
    return s8();
}
static int s10(void){
    return s9();
}

int call_stack(void) {
    int a = 0;
    int c = 8;
    c = s10();

    return a + c;
}
```

反汇编：

##### A64: ARMv8 AArch64

```assembly
Disassembly of section .text:

0000000000000000 <s0>:
   0:   52800000        mov     w0, #0x0                        // #0
   4:   d65f03c0        ret

0000000000000008 <s1>:
   8:   a9bf7bfd        stp     x29, x30, [sp, #-16]!
   c:   910003fd        mov     x29, sp
  10:   97fffffc        bl      0 <s0>
  14:   a8c17bfd        ldp     x29, x30, [sp], #16
  18:   d65f03c0        ret

000000000000001c <s2>:
  1c:   a9bf7bfd        stp     x29, x30, [sp, #-16]!
  20:   910003fd        mov     x29, sp
  24:   97fffff9        bl      8 <s1>
  28:   a8c17bfd        ldp     x29, x30, [sp], #16
  2c:   d65f03c0        ret

0000000000000030 <s3>:
  30:   a9bf7bfd        stp     x29, x30, [sp, #-16]!
  34:   910003fd        mov     x29, sp
  38:   97fffff9        bl      1c <s2>
  3c:   a8c17bfd        ldp     x29, x30, [sp], #16
  40:   d65f03c0        ret

0000000000000044 <s4>:
  44:   a9bf7bfd        stp     x29, x30, [sp, #-16]!
  48:   910003fd        mov     x29, sp
  4c:   97fffff9        bl      30 <s3>
  50:   a8c17bfd        ldp     x29, x30, [sp], #16
  54:   d65f03c0        ret

0000000000000058 <s5>:
  58:   a9bf7bfd        stp     x29, x30, [sp, #-16]!
  5c:   910003fd        mov     x29, sp
  60:   97fffff9        bl      44 <s4>
  64:   a8c17bfd        ldp     x29, x30, [sp], #16
  68:   d65f03c0        ret

000000000000006c <s6>:
  6c:   a9bf7bfd        stp     x29, x30, [sp, #-16]!
  70:   910003fd        mov     x29, sp
  74:   97fffff9        bl      58 <s5>
  78:   a8c17bfd        ldp     x29, x30, [sp], #16
  7c:   d65f03c0        ret

0000000000000080 <s7>:
  80:   a9bf7bfd        stp     x29, x30, [sp, #-16]!
  84:   910003fd        mov     x29, sp
  88:   97fffff9        bl      6c <s6>
  8c:   a8c17bfd        ldp     x29, x30, [sp], #16
  90:   d65f03c0        ret

0000000000000094 <s8>:
  94:   a9bf7bfd        stp     x29, x30, [sp, #-16]!
  98:   910003fd        mov     x29, sp
  9c:   97fffff9        bl      80 <s7>
  a0:   a8c17bfd        ldp     x29, x30, [sp], #16
  a4:   d65f03c0        ret

00000000000000a8 <s9>:
  a8:   a9bf7bfd        stp     x29, x30, [sp, #-16]!
  ac:   910003fd        mov     x29, sp
  b0:   97fffff9        bl      94 <s8>
  b4:   a8c17bfd        ldp     x29, x30, [sp], #16
  b8:   d65f03c0        ret

00000000000000bc <s10>:
  bc:   a9bf7bfd        stp     x29, x30, [sp, #-16]!
  c0:   910003fd        mov     x29, sp
  c4:   97fffff9        bl      a8 <s9>
  c8:   a8c17bfd        ldp     x29, x30, [sp], #16
  cc:   d65f03c0        ret

0000000000000148 <call_stack>:
 148:   a9be7bfd        stp     x29, x30, [sp, #-32]!
 14c:   910003fd        mov     x29, sp
 150:   b9001fff        str     wzr, [sp, #28]
 154:   52800100        mov     w0, #0x8                        // #8
 158:   b9001be0        str     w0, [sp, #24]
 15c:   97ffffd8        bl      bc <s10>
 160:   b9001be0        str     w0, [sp, #24]
 164:   b9401fe1        ldr     w1, [sp, #28]
 168:   b9401be0        ldr     w0, [sp, #24]
 16c:   0b000020        add     w0, w1, w0
 170:   a8c27bfd        ldp     x29, x30, [sp], #32
 174:   d65f03c0        ret
```

每个函数都在将sp - 16的位置，让栈向下增，栈空间逐步加大， 把x29和x30，栈指针和返回地址存入栈空间，然后函数返回后弹出栈。

##### A32: ARMv7 AArch32

```C
Disassembly of section .text:

00000000 <s0>:
   0:   b480            push    {r7}
   2:   af00            add     r7, sp, #0
   4:   2300            movs    r3, #0
   6:   4618            mov     r0, r3
   8:   46bd            mov     sp, r7
   a:   f85d 7b04       ldr.w   r7, [sp], #4
   e:   4770            bx      lr

00000010 <s1>:
  10:   b580            push    {r7, lr}
  12:   af00            add     r7, sp, #0
  14:   f7ff fff4       bl      0 <s0>
  18:   4603            mov     r3, r0
  1a:   4618            mov     r0, r3
  1c:   bd80            pop     {r7, pc}
  1e:   bf00            nop

00000020 <s2>:
  20:   b580            push    {r7, lr}
  22:   af00            add     r7, sp, #0
  24:   f7ff fff4       bl      10 <s1>
  28:   4603            mov     r3, r0
  2a:   4618            mov     r0, r3
  2c:   bd80            pop     {r7, pc}
  2e:   bf00            nop

00000030 <s3>:
  30:   b580            push    {r7, lr}
  32:   af00            add     r7, sp, #0
  34:   f7ff fff4       bl      20 <s2>
  38:   4603            mov     r3, r0
  3a:   4618            mov     r0, r3
  3c:   bd80            pop     {r7, pc}
  3e:   bf00            nop

00000040 <s4>:
  40:   b580            push    {r7, lr}
  42:   af00            add     r7, sp, #0
  44:   f7ff fff4       bl      30 <s3>
  48:   4603            mov     r3, r0
  4a:   4618            mov     r0, r3
  4c:   bd80            pop     {r7, pc}
  4e:   bf00            nop

00000050 <s5>:
  50:   b580            push    {r7, lr}
  52:   af00            add     r7, sp, #0
  54:   f7ff fff4       bl      40 <s4>
  58:   4603            mov     r3, r0
  5a:   4618            mov     r0, r3
  5c:   bd80            pop     {r7, pc}
  5e:   bf00            nop

00000060 <s6>:
  60:   b580            push    {r7, lr}
  62:   af00            add     r7, sp, #0
  64:   f7ff fff4       bl      50 <s5>
  68:   4603            mov     r3, r0
  6a:   4618            mov     r0, r3
  6c:   bd80            pop     {r7, pc}
  6e:   bf00            nop

00000070 <s7>:
  70:   b580            push    {r7, lr}
  72:   af00            add     r7, sp, #0
  74:   f7ff fff4       bl      60 <s6>
  78:   4603            mov     r3, r0
  7a:   4618            mov     r0, r3
  7c:   bd80            pop     {r7, pc}
  7e:   bf00            nop

00000080 <s8>:
  80:   b580            push    {r7, lr}
  82:   af00            add     r7, sp, #0
  84:   f7ff fff4       bl      70 <s7>
  88:   4603            mov     r3, r0
  8a:   4618            mov     r0, r3
  8c:   bd80            pop     {r7, pc}
  8e:   bf00            nop

00000090 <s9>:
  90:   b580            push    {r7, lr}
  92:   af00            add     r7, sp, #0
  94:   f7ff fff4       bl      80 <s8>
  98:   4603            mov     r3, r0
  9a:   4618            mov     r0, r3
  9c:   bd80            pop     {r7, pc}
  9e:   bf00            nop

000000a0 <s10>:
  a0:   b580            push    {r7, lr}
  a2:   af00            add     r7, sp, #0
  a4:   f7ff fff4       bl      90 <s9>
  a8:   4603            mov     r3, r0
  aa:   4618            mov     r0, r3
  ac:   bd80            pop     {r7, pc}
  ae:   bf00            nop
    
000000f4 <call_stack>:
  f4:   b580            push    {r7, lr}
  f6:   b082            sub     sp, #8
  f8:   af00            add     r7, sp, #0
  fa:   2300            movs    r3, #0
  fc:   607b            str     r3, [r7, #4]
  fe:   2308            movs    r3, #8
 100:   603b            str     r3, [r7, #0]
 102:   f7ff ffcd       bl      a0 <s10>
 106:   6038            str     r0, [r7, #0]
 108:   687a            ldr     r2, [r7, #4]
 10a:   683b            ldr     r3, [r7, #0]
 10c:   4413            add     r3, r2
 10e:   4618            mov     r0, r3
 110:   3708            adds    r7, #8
 112:   46bd            mov     sp, r7
 114:   bd80            pop     {r7, pc}
 116:   bf00            nop
```

#### 1.4.3 ARMv8的函数调用标准

函数调用标准（Procedure Call Standard, PCS）用来描述父/子函数是如何编译、链接的，尤其是父函数和子函数之间调用关系的约定，如栈的布局、参数的传递、还有C语言类型的长度等等。每个处理器体系结构都有不同的标准。下面以ARM64为例介绍函数调用的标准（参考： Procedure Call Standard for ARM 64-bit Architecture[^6] [^7] )

ARM64体系结构的通用寄存器：

| 寄存器         | 描述                                                     |
| -------------- | -------------------------------------------------------- |
| SP寄存器       | SP寄存器                                                 |
| x30 (LR寄存器) | 链接寄存器                                               |
| x29 (FP寄存器) | 栈帧指针（Frame Pointer）寄存器                          |
| x19~x28        | 被调用函数保存的寄存器，在子函数中使用时需要保存到栈中。 |
| x18            | 平台寄存器                                               |
| x17            | 临时寄存器IPC（intra-precedure-call）临时寄存器          |
| x16            | 临时寄存器或第一个IPC临时寄存器                          |
| x9~x15         | 临时寄存器                                               |
| x8             | 间接结果位置寄存器，用于保存程序返回的地址               |
| x0~x7          | 用于传递子函数参数和结果，                               |

### 2 堆与内存管理

堆的概念我们已经知道了，而且我们还用过大名鼎鼎的malloc函数，甚至malloc_align函数，但是我们似乎没有研究过在Linux里面malloc原理是什么样子的，在今天的这个topic我们再进一步的了解一下堆，后面我们在学习linux内核的内存管理的时候会更详细的讲解一下malloc如何实现的。

#### 2.1 Linux进程堆管理

Linux进程地址空间，除了文件、共享库还有栈之外，剩余的未分配的空间都可以作为Heap的空间地址，堆和栈相反，堆是向上增长的。运行库向操作系统申请一批空间地址，又程序自己“零售”给内部程序。

![](https://raw.githubusercontent.com/carloscn/images/main/typora161204769-04c36184-b05a-442d-93e0-af87504c6171.png)

Linux进程堆管理有两种方式：

* brk()系统调用
* mmap()

brk()系统调用实际上就设置进程数据段（data段+bss段的统称）的结束地址，如果我们将数据段结束地址向高地址不断滚动，那么扩大的空间就是我们可以用的heap的空间，glibc里面有个sbrk函数。

mmap()的作用是向操作系统申请一段虚拟内存地址，如果指定文件路径是可以将空间映射到文件，如果没有指定文件路径，那么就是匿名空间(Anonymous)，匿名空间就可以作为堆空间。mmap可以指定申请空间的大小和起始地址，如果起始地址设定为0，那么mmap会自动跳转到合适的位置，申请的空间还可以指定权限。

```c
void *malloc(size_t nbytes)
{
     void *ret = mmap(0, bytes, PROT_READ|PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS, 0, 0);
     __check_ret__(ret);
     return ret;
}
```

glibc的malloc函数处理逻辑是这样的：

* 对于小于128KB的请求，它会在现有的堆空间分配。
* 对于大于128KB的请求，它会使用mmap函数为它分配一段匿名空间，然后再从匿名空间分配用户空间。

#### 2.2 堆分配算法

* 空闲链表法
* 位图法
* 对象池法

### 2.3 堆碎片化问题

#### 2.3.1 碎片产生[^3]

```C
int main()
{
        int *heap_d;
        int *heap_e;
        int *heap_f;
        heap_d = (int *)malloc(10);
        heap_e = (int *)malloc(10);
        printf("The d address is %p\n",heap_d);
        printf("The e address is %p\n",heap_e);
        free(heap_d);
        heap_d = NULL;
        heap_f = (int *)malloc(30);
        printf("The f address is %p\n",heap_f);
        return 0;
}
```

```
The d address is 0xf0d010 mem_d
The e address is 0xf0d030 mem_e
The f address is 0xf0d460 mem_f
 
可想而知，总共三段内存分配
mem_d|mem_e|
free
     |mem_e|
           |mem_f|
|xxxx|     |     |
xxx为无用内存，碎片，即使分配后已经free和置NULL操作。
越来越多的malloc使用，会促进内存碎片化加剧，最终内存不足。
```

#### 2.3.2 baremental/freeRTOS堆空间

嵌入式设备没有MMU，无法实现内存动态映射。所以没有操作系统兜底的嵌入式设备一定要小心，就算是有操作系统也要对内存分配了如指掌，否则就会出现意想不到的问题，内存碎片的问题就是很头疼的问题。

##### freeRTOS

freeRTOS对于堆的管理分为5个heap管理方式[^5]，十分复杂。

- [heap_1](https://www.freertos.org/a00111.html#heap_1) - the very simplest, does not permit memory to be freed.
- [heap_2](https://www.freertos.org/a00111.html#heap_2) - permits memory to be freed, but does not coalescence adjacent free blocks.
- [heap_3](https://www.freertos.org/a00111.html#heap_3) - simply wraps the standard malloc() and free() for thread safety.
- [heap_4](https://www.freertos.org/a00111.html#heap_4) - coalescences adjacent free blocks to avoid fragmentation. Includes absolute address placement option.
- [heap_5](https://www.freertos.org/a00111.html#heap_5) - as per heap_4, with the ability to span the heap across multiple non-adjacent memory areas.

##### baremental[^4]

malloc和free并不能实现动态的内存的管理。这需要在启动阶段专门给其分配一段空闲的内存区域作为malloc的内存区。如STM32中的启动文件startup_stm32f10x_md.s中可见以下信息：

```
Heap_Size       EQU     0x00000800

                AREA    HEAP, NOINIT, READWRITE, ALIGN=3
__heap_base
Heap_Mem        SPACE   Heap_Size
__heap_limit
```

其中，Heap_Size即定义一个宏定义。数值为 0x00000800。Heap_Mem则为申请一块连续的内存，大小为 Heap_Size。简化为C语言版本如下：

```
#define Heap_Size 0x00000800
unsigned char Heap_Mem[Heap_Size] = {
     0};
```

在这里申请的这块内存，在接下来的代码中，被注册进系统中给malloc和free函数所使用：

```
__user_initial_stackheap
LDR     R0, =  Heap_Mem  ;  返回系统中堆内存起始地址
LDR     R1, =(Stack_Mem + Stack_Size)
LDR     R2, = (Heap_Mem +  Heap_Size); 返回系统中堆内存的结束地址
LDR     R3, = Stack_Mem
BX      LR
```

在函数中使用malloc，如果是大的内存分配，而且malloc与free的次数也不是特别频繁，使用malloc与free是比较合适的，但是如果内存分配比较小，而且次数特别频繁，那么使用malloc与free就有些不太合适了。因为过多的malloc与free容易造成内存碎片，致使可使用的堆内存变小。尤其是在对单片机等没有MMU的芯片编程时，慎用malloc与free。

对于堆碎片化的问题，可以采用堆分配算法避免，比如内存池。

内存池，简洁地来说，就是预先分配一块固定大小的内存。以后，要申请固定大小的内存的时候，即可从该内存池中申请。用完了，自然要放回去。注意，内存池，每次申请都只能申请固定大小的内存。这样子做，有很多好处：

* 每次动态内存申请的大小都是固定的，可以有效防止内存碎片化。(至于为什么，可以想想，每次申请的都是固定的大小，回收也是固定的大小)

* 效率高，不需要复杂的内存分配算法来实现。申请，释放的时间复杂度，可以做到O(1)。

* 内存的申请，释放都在可控的范围之内。不会出现以后运行着，运行着，就再也申请不到内存的情况。

内存池，并非什么很厉害的技术。实现起来，其实可以做到很简单。只需要一个链表即可。在初始化的时候，把全局变量申请来的内存，一个个放入该链表中。在申请的时候，只需要取出头部并返回即可。在释放的时候，只需要把该内存插入链表。以下是一种简单的例子(使用移植来的linux内核链表，对该链表的移植，以后有时间再去分析)：

```C
#define MEM_BUFFER_LEN  5    //内存块的数量
#define MEM_BUFFER_SIZE 256 //每块内存的大小

//内存池的描述，使用联合体，体现穷人的智慧。就如，我一同学说的：一个字节，恨不得掰成8个字节来用。
typedef union mem {
     
struct list_head list;
unsigned char buffer[MEM_BUFFER_SIZE];
}mem_t;

static union mem gmem[MEM_BUFFER_LEN];

LIST_HEAD(mem_pool);

//分配内存
void *mem_pop(){
     
    union mem *ret = NULL;
    psr_t psr;

    psr = ENTER_CRITICAL();
    if(!list_empty(&mem_pool)) { //有可用的内存池 
        ret = list_first_entry(&mem_pool, union mem, list);
        //printf("mem_pool = 0x%p  ret = 0x%p\n", &mem_pool, &ret->list);
        list_del(&ret->list);
 }
 EXIT_CRITICAL(psr);
 return ret;//->buffer;
}


//回收内存
void mem_push(void *mem){
     
    union mem *tmp = NULL; 
    psr_t psr;

    tmp = (void *)mem;//container_of(mem, struct mem, buffer);
    psr = ENTER_CRITICAL();
    list_add(&tmp->list, &mem_pool);
    //printf("free = 0x%p\n", &tmp->list);

    EXIT_CRITICAL(psr);
}

//初始化内存池
void mem_pool_init(){
     
    int i;
    psr_t psr;
    psr = ENTER_CRITICAL();
    for(i=0; i        list_add(&(gmem[i].list), &mem_pool);
        //printf("add mem 0x%p\n", &(gmem[i].list));
 }
 EXIT_CRITICAL(psr);
}
```

### 2.4 使用malloc和free一些建议

* 不建议在中断中使用malloc。
* 线程不一定安全，在-pthread进行编译是线程安全的，在freeRTOS的heap_3.c中进行封装pvPortMalloc是安全的，但是在其他环境要持怀疑态度。
* malloc不一定会成功，需要check结果
* malloc和free一定要成对出现。
* free之后给指针加NULL，防止野指针。
* 为了安全考虑，malloc之后的内存，需要memset置空后free掉，防止那块内存被分配可以读到数据。

## 3 Reference

[^1]:[ARM Compiler armasm Reference Guide Version 6.00 - PUSH and POP ](https://developer.arm.com/documentation/dui0802/a/A32-and-T32-Instructions/PUSH-and-POP)
[^2]:[arm-community-blogs - Using the Stack in AArch32 and AArch64](https://community.arm.com/arm-community-blogs/b/architectures-and-processors-blog/posts/using-the-stack-in-aarch32-and-aarch64)
[^3]:[如何看待malloc产生内存碎片](https://www.shuzhiduo.com/A/8Bz8A2kkJx/)
[^4]:[linux malloc free 内存碎片_嵌入式裸机编程中使用malloc、free会怎样？](https://www.cxyzjd.com/article/weixin_39625747/113086126)
[^5]:[FreeRTOS kernel - Memory Management](https://www.freertos.org/a00111.html#heap_1)
[^6]:[**Procedure Call Standard for the Arm® 64-bit Architecture (AArch64).pdf**](https://github.com/carloscn/doclib/blob/master/man/arm/standard/Procedure%20Call%20Standard%20for%20the%20Arm%C2%AE%2064-bit%20%20Architecture%20(AArch64).pdf)
[^7]:[https://github.com/ARM-software/abi-aa](https://github.com/ARM-software/abi-aa/releases)