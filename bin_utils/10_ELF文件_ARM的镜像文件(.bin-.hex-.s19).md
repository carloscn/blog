# 10_ELF文件_ARM的镜像文件(.bin-.hex-.s19)

ARM的编译器直接编译出文件应该是elf格式的，类似于Linux，它是有elf解析器可以对elf进行解析和提取的，而baremental环境是鲜有elf解析器，因为它对于baremental的环境实在是太大了。因此这里就需要我们自己去handle二进制文件，把他们放在一个正确的内存里面。

每个厂家的bootROM不一样，所以**对于二进制文件的存储要求也不一样**。这个规则也是看二级厂家是怎么定义，但是这里还是能抽象出一些比较通用的规则。image文件主要是用来做大规模量产的。既要做大规模量产，由于各芯片厂家制定的标准不一，所以实际上image文件有很多种格式。本文包含其中最具有代表性也应用最广泛的3种image文件格式：

* 通用的bin
* intel的hex
* S-Record

本文以NXP rt1176 Cortex-M7的处理器为demo，来研究一下
* image文件的种类
* image文件与代码段的管理

生成的二进制文件列表如图所示：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304281101045.png)

本文的顺序及内容参考[^5]，并对[^5]进行扩展和扩充。软件参考[^4] 。

# 1. image文件种类

## 1.1 Binary

第一种格式叫binary，以.bin为文件后缀，这种格式是一种通用image格式，其完全是机器码裸数据的集合，没有其他任何多余信息，这个数据可以直接被编程器/下载器下载到芯片内部非易失性存储器里，不需要任何额外的数据转换。由于是纯二进制编码的文件，所以普通text编辑器无法正确查看这个文件，需要用专用的十六进制编辑器（比如Hex Editor）才能正常打开。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304281102678.png)

在elf会有**分段**的信息对数据进行管理。可binary是如何进行分段？**bin文件会在非有效地址区域插入无效字节以保证有效地址处都是正确的数据**。这在IDE里或相关转换工具里都会有相应option（例如 --gap-fill），可以让用户设置填充字节pattern（比如0x00，0xFF等）


### 1.1.1 binary 编码

关于指令和二进制文件编码的方式可以参考：[Binary Encoding of ARM Data Processing Instructions](https://www.youtube.com/watch?v=6Zco5PgHlsU&ab_channel=PEEYUSHKP)

ARM Image Format参考： https://developer.arm.com/documentation/dui0041/c/ARM-Image-Format?lang=en

### 1.1.2 binary 定义

arm编译器通常编译出来的信息是elf文件，我们通常使用工具`objcopy`指定binary可以产生binary文件。根据TI的定义[^1]，

>When objcopy generates a raw binary file, it will essentially produce a memory dump of the contents of the input object file. All symbols and relocation information will be discarded. The memory dump will start at the load address of the lowest section copied into the output file.

所有的符号和重定位的信息全部被丢弃。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304281339797.png)

分段的信息在hole处被填充。.stack的信息被丢弃。

### 1.1.3 binary 寻址

那么既然这样binary文件如何寻址呢？`target address = lowest initialized address + file offset`

最低为初始化地址是init section最低位的起始地址。在图中就是secB的地址0x100。一个二进制文件不会记录开始的地址。

### 1.1.4 hole

binary的格式非常简单，似乎没有什么严谨的方式来表示一个hole。只能使用pattern来填充。通常都是使用0来填充。objcopy可以通过--gap-fill的参数来指定用什么填充hole[^2] 。

`avr-objcopy -I ihex -O binary --gap-fill 0xFF --pad-to 0x1FFE fkmegax8.hex fkmegax8.bin`

可想而知，如果一个binary充满了hole那么这个binary就会非常占用空间。因此我们在开发的过程中应该避免hole的产生。以下的编程技巧可以减少hole的产生：

* 对初始化的section要尽可能的放在一起；
* 对于支持多个二进制文件的，把二进制分组（不一定支持），这样这样就能控制每个二进制的起始地址，hole自然就不存在了；

### 1.1.5 Cortex-M binary[^3]

以上说的是一些比较通用的定义，但是每个处理器架构对二进制处理方式不一样。我们参考cortex-m的一些设计，在这里引出：

过程如下，最后被linker生成一个elf文件，通过objcopy生成binary文件：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304281408329.png)

生成的二进制文件如下：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304281413448.png)

* 0x20020000开始的位置是stack pointer (注意大小端)
* 0x0800027D是counter
* 紧接着是vector table
* 紧接着是code

注意大端模式和小端模式在这个binary格式上面有很大的影响力。

#### 栈

在我的demo中，如图所示

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304281450943.png)

可以读出来，栈指针应该是在0003f000位置，我们去map文件查找，果然是栈顶的位置。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304281450958.png)

而栈顶是在，start.S汇编中

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304281451771.png)

通常start.S包含[^3]：
* reset handler. This is called at startup (or reset) of the dev board；
* .data段；
* .bss段；
* 中断向量表；
* main函数

在linkerfile中定义.stack (也就是vStackTop)

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304281459587.png)

根据IDE的RAM定义：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304281457534.png)

0x2000 + 0x3d000 - 0x1000 + 0x1000  = 0x3f000

栈指针的位置就能按照上面的方法推算出来。

#### vector

向量表在map文件中指向了绝对的0x2000的位置，也就是text头的位置。根据 https://github.com/carloscn/blog/issues/127 的2.4 中断向量，CortexM中的确是这样处理的。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304281511949.png)

我们可以看下linker文件中，把vec放在text的头部：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304281513154.png)

而text是映射到了RAM排列的第一个位置：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304281515108.png)

所以RAM排列也很重要。


## 1.2 Intel Hex[^5]

第二种格式叫Intel hex，以.hex为文件后缀，这种格式是Intel公司推行的一种image格式标准，其不仅含有机器码裸数据**还含有地址信息等额外信息**，与bin文件不同的是，hex文件可以直接通用普通text编辑器打开查看，hex文件采用的ASCII编码，hex文件内的机器码数据不可以直接被下载进芯片内部，需要在帧数据解析的过程中进行转换[^5]。

由于hex文件并不是纯机器码文件，还含有其他额外信息，那么hex文件就需要按某种约定格式进行数据组织，数据组织方式叫帧格式，hex文件是由n帧数据组成的[^5]。

Hex也可以从ELF文件转换而来，工具还是objcopy

`arm-none-eabi-objcopy -v -O ihex "${BuildArtifactFileName}" "${BuildArtifactFileBaseName}.hex"`

### 1.2.1 Hex格式

要想解析hex文件，必须要先了解其帧格式，hex每帧都由下表列出的6部分组成：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304281433049.png)

| 帧段名          | Start code                  | Byte count     | Address            | Record type              | Data       | Checksum |
| ----------------- | ----------------------------- | ---------------- | -------------------- | -------------------------- | ------------ | ---------- |
| 帧段内容        | 固定前导码':'，十六进制0x3A | 机器码数据长度 | 机器码数据存储地址 | 帧类型"00"-"05"，有6种帧 | 机器码数据 | 校验和   |
| 帧段长度(bytes) | 1                           | 1*2            | 2*2                | 1*2                      | (0-255)*2  | 1*2      |

编程器/下载器在解析hex文件时，先找到帧前导码，然后找到帧类型，如果该帧为数据帧，再根据帧机器码长度，将该帧机器码数据全部读出放到缓存里，在做完帧和校验后，如果没有错误，最后根据帧机器码存储地址将帧机器码数据下载到芯片指定存储器地址处，至此一帧处理结束，进入下一帧，直到所有帧全部处理完。需要注意的是由于hex文件是ASCII编码，所以相比bin文件长度至少大2倍以上，demo文件大小有如图所示：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304281438414.png)

关于checksum的计算方法，其是将Byte count、Address、Record type、Data四个段内所有byte全部相加得到sum，截取sum的LSB（最低字节），再对该LSB取其补码（默认该LSB为负数的低8bit数据位(注意8bit中没有符号位)，其反码为原码各bit取反，其补码为反码+1）得到checksum。比如LSB是0xA5，其反码为0x5A，补码为0x5B，则checksum为0x5B。

前面说到一共有6种类型的帧，其中最重要也是数量最多的帧是数据帧，除了数据帧之外还有其他5种帧，下面来统一介绍：

| 帧类型码 | 帧类型               | 帧描述                                                                      | 帧举例                                                                     |
| ---------- | ---------------------- | ----------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| "00"     | 数据帧               | 以16bit地址描述开始的最大255个字节有效机器码的数据帧                        | 含11bytes机器码从0x0010地址处开始的数据帧:0B0010006164647265737320676170A7 |
| "01"     | 文件结尾帧           | 用于表明hex文件的结尾                                                       | 统一的文件结尾帧:00000001FF                                                |
| "02"     | 拓展段地址帧         | 多用于80x86架构芯片，ARM Cortex-M架构不用                                   | :020000021200EA                                                            |
| "03"     | 起始段地址帧         | 多用于80x86架构芯片，ARM Cortex-M架构不用                                   | :0400000300003800C1                                                        |
| "04"     | 拓展线性地址帧       | 用于32bit地址存储空间芯片，与数据帧配合使用，指引编程器将数据下载到正确地址 | 标明拓展地址为0xFFFF的帧:02000004FFFFFC                                    |
| "05"     | 起始程序地址（PC）帧 | 指示调试器，程序初始PC地址，方便在线调试                                    | 标明起始PC为0x000000CD的帧:04000005000000CD2A                              |


## 1.3 Motorola S-Record[^5]

第三种格式叫Motorola S-Record，以.s19或.srec为文件后缀，这种格式是Motorola公司推行的一种image格式标准，其与Intel hex文件比较类似，都是ASCII编码的文件，可以通过普通text编辑器打开查看，其也由帧数据组成，只是帧格式与Intel hex有差别，还是按照介绍Intel hex文件那样先来看S-Record文件的帧格式。

To generate a S19/Motorola S-Record, use the following:

`arm-none-eabi-objcopy -v -O srec "${BuildArtifactFileName}" "${BuildArtifactFileBaseName}.s19"`

| 帧段名          | Start code                  | Record type             | Byte count                                 | Address            | Data       | Checksum |
| ----------------- | ----------------------------- | ------------------------- | -------------------------------------------- | -------------------- | ------------ | ---------- |
| 帧段内容        | 固定前导码'S'，十六进制0x53 | 帧类型'0'-'9'，有10种帧 | 帧数据长度（包含后续段地址、数据、校验和） | 机器码数据存储地址 | 机器码数据 | 校验和   |
| 帧段长度(bytes) | 1                           | 1                       | 1*2                                        | (2-4)*2            | (0-255)*2  | 1*2      |


编程器/下载器在解析S-Record文件时，先找到帧前导码，然后找到帧类型，如果该帧为数据帧，再根据帧长度，将该帧机器码数据全部读出放到缓存里，在做完帧和校验后，如果没有错误，最后根据帧机器码存储地址将帧机器码数据下载到芯片指定存储器地址处，至此一帧处理结束，进入下一帧，直到所有帧全部处理完。  

关于checksum的计算方法，其是将Byte count、Address、Data三个段内所有byte全部相加得到sum，截取sum的LSB（最低字节），再对该LSB取其反码（默认该LSB为负数的低8bit数据位(注意8bit中没有符号位)，其反码为原码各bit取反）得到checksum。比如LSB是0xA5，其反码为0x5A，则checksum为0x5A。

前面说到一共有10种类型的帧，其中最重要也是数量最多的帧是数据帧，数据帧按地址长度可分为16bit、24bit、32bit地址长度数据帧，除了数据帧，还有其他种类帧，下面来统一介绍：

| 帧类型码 | 帧类型                  | 帧描述                                               | 帧举例                                                                          |
| ---------- | ------------------------- | ------------------------------------------------------ | --------------------------------------------------------------------------------- |
| '0'      | 文件起始帧              | 用于表明S-Record文件的开始                           | 标明文件名为HDR的文件起始帧S00600004844521B                                     |
| '1'      | 数据帧x16地址           | 以16bit地址描述开始的最大255个字节有效机器码的数据帧 | 含14bytes机器码从0x0038地址处开始的数据帧S111003848656C6C6F20776F726C642E0A0042 |
| '2'      | 数据帧x24地址           | 以24bit地址描述开始的最大255个字节有效机器码的数据帧 | 含4bytes机器码从0x100000地址处开始的数据帧S2081000000400FA05E5                  |
| '3'      | 数据帧x32地址           | 以32bit地址描述开始的最大255个字节有效机器码的数据帧 | 含4bytes机器码从0x13000160地址处开始的数据帧S309130001600400FA057F              |
| '4'      | N/A                     | 未定义                                               | N/A                                                                             |
| '5'      | 数据帧总数帧x16         | 用16bit count记录数据帧总帧数的总数帧                | 标明总数据帧为4帧的总数帧S5030004F8                                             |
| '6'      | 数据帧总数帧x24         | 用24bit count记录数据帧总帧数的总数帧                | 标明总数据帧为80000帧的总数帧S604080000F3                                       |
| '7'      | 起始程序地址（PC）帧x32 | 含32bit起始PC的帧                                    | 标明起始PC为0x10000000的帧S70510000000EA                                        |
| '8'      | 起始程序地址（PC）帧x24 | 含24bit起始PC的帧                                    | 标明起始PC为0x100000的帧S804100000EB                                            |
| '9'      | 起始程序地址（PC）帧x16 | 含16bit起始PC的帧                                    | 标明起始PC为0x0000的帧S9030000FC                                                |


## 1.4 转换工具

`sudo apt-get install srecord`

`srec_cat input.hex -Intel -o output.srec -Motorola`

`srec_cat input.srec -Motorola -o output.hex -Intel`


# Ref
[^1]:[An Introduction to Binary Files](https://software-dl.ti.com/ccs/esd/documents/sdto_cgt_an_introduction_to_binary_files.html)
[^2]:[Use objcopy --gap-fill only for special section](https://stackoverflow.com/questions/13530661/use-objcopy-gap-fill-only-for-special-section)
[^3]:[In Depth Analysis of an ARM Cortex-M4 Program](https://www.tmdarwen.com/latest/in-depth-analysis-of-an-arm-cortex-m-program)
[^4]:[MCUXpresso IDE: S-Record, Intel Hex and Binary Files](https://mcuoneclipse.com/2017/03/29/mcuxpresso-ide-s-record-intel-hex-and-binary-files/)
[^5]:[痞子衡嵌入式：ARM Cortex-M文件那些事（8）- 镜像文件(.bin/.hex/.s19)](https://www.cnblogs.com/henjay724/p/8361693.html)
[^6]:[Conversions between HEX and SREC files](https://www.feaser.com/en/blog/2017/11/conversions-between-hex-and-srec-files/)

