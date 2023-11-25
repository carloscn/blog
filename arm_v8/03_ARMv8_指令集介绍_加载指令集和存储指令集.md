Github地址：[carloscn/uncle-ben-os at car_lab_06 (github.com)](https://github.com/carloscn/uncle-ben-os/tree/car_lab_06)

### ARMv8指令集介绍

* A64指令集只能运行在aarch64
* 所有A64汇编都是32 bits宽的
  * 关注指令的使用、有什么limitation
  * A64能访问的地址数据是64位宽的
* A64支持全部的大写或者小写方式
  * ARM官方大写
  * 应用使用小写
* 寄存器命名
  * Wn表示32bits宽的寄存器
  * Xn表示64bits宽的寄存器
  * WZR表示32位内容全为0的寄存器
  * XZR表示64位内容全为0的寄存器
  * ...

### LDR指令

`LDR Xd, [Xn, $offset]` 

* 【释义】：将Xn寄存器中存储的地址+offset地址偏移存 组成一个新的地址，把这个地址里面存储的值放在Xd寄存器中。[]有取地址内存储的数值的含义。

* 【示例】：

  * S1: 使用MOV指令把0x80000加载到X1寄存器： `MOV x1, 0x80000`  （如果是一个数，而非#0x80000, 则是一个地址）

  * S2: 使用MOV指令把16数值加载到X3寄存器： `MOV x3, 16`

  * S3: 使用LDR指令读取X1地址里面存储的值，存储到X0中:  `LDR x0,[x1] `,  这个不允许->`LDR x2,[0x80000]`

  * S4:使用LDR指令读取X1 + 8地址里面存储的值，存储到X2中：`LDR x2,[x1, #8]`

  * S5:使用LDR指令读取(X1 + X3)地址里面存储的值，存储到X4中： `LDR x4,[x1, x3]`

  * S6: 使用LDR指令读取(X1+(X3<<3))地址里面存储的值，存储到X5中： `LDR x5,[x1,x3,lsl #3]`

* 【注意】：
  * 给的数不加任何标志的视为地址
  * 需要给立即数的场景而非地址的值，使用#
  * []有取地址值的意思
  * LDR lsl扩展指令，只支持1和3
  * `LDR x2,[x1, #8]` x1的值**不会**被更新为0x80008

* 【变基模式】：

  * 前变基模式 pre-index： 先更新偏移地址，后访问地址 （注意有叹号!表示）

    `LDR x6, [x1, #8]!` : 将x1里面的地址增加偏移#8并赋给x1，最后将新的x1寄存器内的地址的值给x6寄存器

  * 后变基模式 post-index： 先访问内存地址，再更新偏移地址

    `LDR x6, [x1], #8` : 将x1寄存器内的地址的值赋给x6寄存器，并将x1地址偏移+8。

* 【伪指令】:

   伪指令与指令的最大不同在于，伪指令属于编译器处理的范畴，伪指令会被编译展开为多条指令；指令是CPU处理的命令的最小单元。

  * `LDR x7,=0x80000` -> 等同于 MOV x7, 0x80000
  * 需要区别 `LDR x7, 0x800000`; 这条指令的意义是，将当前PC寄存器的地址的 + 0x80000的偏移，取出地址内容填充到x7寄存器中。

### STR指令

从一个寄存器的值吐到内存中，支持立即数和寄存器操作。把Xd的值，存储到[xn|sp]里面。

immediate-post-index: `STR Xd, [Xn|SP], #<simm>` 

immediate-pre-index: `STR Xd, [Xn|SP, #<simm>]!`

* 【示例】：

  ```assembly
  ; Example 1:
  MOV x2, 0x400000             ; -> x2 is 0x400000
  LDR x6, =0x1234abce          ; -> x6 is 0x1234abce
  STR x6, [x2, #8]!            ; -> 把x6的值（0x1234abce），存储到0x400008地址的内存里面
  ; What's value of x2? And the value in 0x400000 address?  
  
  ;Example 2:
  MOV x2, 0x500000			 ; -> x2 is 0x500000
  STR x6, [x2], #8			 ; -> 把x6的值（0x1234abce），存储到0x500000里面，并将x2寄存器变为0x500008
  ; What's value of x2? And the value in 0x400000 address?  
  ```

### MOV/MOVZ指令

MOV底层原理实际上是MOVZ，MOV 16-bit的立即数到寄存器。

`MOV xd, #<imm>` 16位立即数

`MOVZ xd, #<imm16>, LSL #<shift>` 16位的立即数，逻辑左移动 16,32,48位

### LDP/STP指令

相比于LDR和STR指令(8 bytes)，LDP和STP指令用于多字节(16 bytes)操作，

【释义】：

* LDP ：`LDP x3, x7, [x0]` -> 从x0的值为基地址，加载地址到X3寄存器，存储x0+8到x7寄存器。
* STP ：`STP x1, x2, [x4]`->  以x4的值为基地址，存储x1地址的值到x4，存储x2地址的值到x4 + 8。

【练习】：

练习1： 使用LDR和STR多字节加载和存储命令实现memset()函数，假设内存地址s是16字节对齐，count也是16字节对齐。例如：memset(0x200000, 0x55, 32)

```C
// memset_a_byte
void *memset_a_byte (void *s, int c) {
    char *xs = s;
    *xs++ =c;
    return s;
}
```

```assembly
// 使用STR指令，单字节操作
.global my_memset_test:
my_memset_test:

// 保存地址s到x1寄存器，保存c的值到x2寄存器，保存长度到x3寄存器
MOV x1, 0x2000000   // 这个值是需要被修改的 肯定需要STP
MOV x2, 0x55        // 这个是个固定的参数
ADD x0, x1, 32
// 确定原子操作 向地址写值，然后地址增加
wrt:
STR x2, [x1], #8	// 把x2里面的值存储到x1里面(0x55 -> 0x200000)，接着0x200008加一
cmp x1, x0
b.cc wrt

ret
```

```assembly
// 使用STP指令，双字节操作
.global my_memset_test:
my_memset_test:

// 保存地址s到x1寄存器，保存c的值到x2寄存器，保存长度到x3寄存器
MOV x1, 0x2000000   // 这个值是需要被修改的 肯定需要STP
MOV x2, 0x55        // 这个是个固定的参数
ADD x0, x1, 32
// 确定原子操作 向地址写值，然后地址增加
wrt:
STP x2, x2, [x1], #16	// 把x2里面的值存储到x1里面(0x55 -> 0x200000)，接着0x200008加一
cmp x1, x0
b.cc wrt

ret
```

练习二：同上，使用非对齐的memset(0x200004, 0x55, 37) 

```assembly
// 需要汇编和C语言混合编程实现对于非16字节对齐的地址和长度进行memset操作
// 汇编实现一个16字节的memset
// C语言用于对非对齐部分进行C语言单字节的处理，用汇编实现16字节对齐地址和16字节对齐长度的处理。

// 函数调用为 memset(0x200004, 0x55, 37)
.global asm_memset_16_byte_align:
asm_memset_16_byte_align:
ADD x4, x0, x2
wrt:
STP x1, x1, [x0], #16
CMP x0, x4
b.cc wrt
ret

void *memset (void *s, int c, int count)
{
   int align = 16;
   if (s & (align - 1)) {
     // 处理非对齐地址
   }
   // 对齐部分直接调用 asm_memset_16_byte_align(s, c, l);
   // 非对齐部分直接c语言指针访问赋值。
}

```

### 一些需要注意的地方

* FAQ1：加载一个很大的数值到通用寄存器，例如0xFFFF0000FFFF0000,  使用MOV指令，是否正确？

  错误，MOV 后面的立即数为16-bit，应该是使用LDR x1,=0xFFFF......0000 伪指令来加载大数。

* FAQ2：加载一个寄存器的值，使用移位：MOV x1, (1<<0) | (1<<2)|(1<<20)|(1<<40)|(1<<55)

  错误，同样是MOV立即数16-bit，使用LDR x1, = (1<<0)|......|(1<<55).

* FAQ3： 字符串的LDR指令

  ```
  string1:
  	.string "Booting at EL"
  LDR x0, string1      // 加载string1字符串的ascii码值到寄存器，最高限制在X寄存器大小也就是 64-bit，如果是W寄存器就是32-bit    
  LDR x1, =string1     // 加载string1字符串的地址到x1
  ```

* FAQ3： 定义数据LDR指令

  ```
  my_data:
  	.word 0x40
  LDR x0, my_data     //加载0x40到X0，等同于MOV x0,0x40 前提是不超过16bits
  LDR x1, =my_data    //加载存储my_data的地址到x1
  ```

* 一种易错的死机状态： 树莓派4b上面的寄存器都是32bit的，下面代码配置26到U_IBRD_REG寄存器，有什么问题？

  ```
  LDR x1, =U_IBRD_REG
  MOV x2, #26
  STR x2, [x1]
  
  //正确写法：
  LDR w1, =U_IBRD_REG
  MOV w2, #26
  STR w2, [w1]
  ```

  错误点在于树莓派4b寄存器访问都是32bit的，现在使用X寄存器，为64位的寄存器，应该使用W寄存器，32位寄存器访问。

## GDB-Tips

* 启动GDB和QEMU链接

  * `> gdb-multiarch --tui benos.elf`
  * `gdb> c`
  * `gdb> target remote localhost:1234`

  * `gdb> b ldr_test`   // 设定断点
  * `gdb> c`
  * `gdb> next`  //下一步
  * `gdb> info register`  // 查看所有寄存器
  * `gdb> info x1 x2 x3` // 查看x1/x2/x3寄存器
  * `gdb> x 0x80000`  // 读取内存0x80000值 32位
  * `gdb> x/xg 0x80000` // 读取内存0x80000值64位

## Addressing

参考[03_ARMv7-M_存储系统结构](https://github.com/carloscn/blog/issues/124)的地址对齐访问设计。我们对ARMv8架构的对齐操作进行整理。

和Cortex-M一样，独占和顺序(ordered)访问（相对于指令预取和乱序访问）不可以对非对齐的地址进行访问。但是**所有的load和store是支持非对齐访问的**。

### 块传输（bulk transfers）

* 这些 `LDM`, `STM`, `PUSH,` 和 `POP`指令不存在与A64指令集。所有的块传输都是通过`STP`和`LDP` 
* The `LDNP` and `STNP` instructions provide a streaming or non-temporal hint, that the data does not need to be retained in caches. 指令提供了一个流式或非临时提示，即数据不需要保留在缓存中。
* The `PRFM`, or prefetch memory instructions enable targeting of a prefetch to a specific cache level .“PRFM”或预取存储器指令能够将预取定为特定的高速缓存级别。

### load/store

* All Load/Store instructions now support consistent addressing modes。现在，所有加载/存储指令都支持一致寻址模式。This makes it much easier, for example, to treat `char`, `short`, `int` and `long long` in the same way when loading and storing quantities from memory. 例如，这使得在从内存加载和存储量时更容易以相同的方式处理“char”、“short”、“int”和“long-long”。
*  浮点寄存器和NEON寄存器现在支持与核心寄存器相同的寻址模式，从而更容易互换使用这两个寄存器组。

### Alignment checking

当执行在AArch64模式的时候，需要对取指令的load和store操作需要使用栈指针，所以会对PC和SP进行对齐检查。

https://developer.arm.com/documentation/den0024/a/An-Introduction-to-the-ARMv8-Instruction-Sets/The-ARMv8-instruction-sets/Addressing


