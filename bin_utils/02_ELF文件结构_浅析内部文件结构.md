# # 02_ELF文件结构_浅析内部文件结构

## 1. 文件结构

### 1.1. Scope

通用的ELF文件，我们可以分为四大类：

* **Header**: 描述基本属性的
* **Sections**：各个段，包括.text .data .bss等
* **Section header tables**：ELF中所有段的段名、段长度、文件偏移、读写权限等
* **Helper tables**：辅助结构，字符串表、符号表。

```
----------------------------------------
       ELF Header（描述基本属性）               
----------------------------------------
       .text （段1）
----------------------------------------
       .data （段2）
----------------------------------------
       .bss  （段3）
----------------------------------------
       ...other 
          sections...
----------------------------------------
       Section header tables （段表）
----------------------------------------
       String Tables   （字符串表）
       Symbol Tables   （符号表）
----------------------------------------
```

### 1.2. Header

`aarch64-none-linux-gnu-gcc main.c -o main`

`readelf -h main`

```C
➜ readelf -h main
ELF Header:
  // e_ident members
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 // ELF 魔数
  Class:                             ELF64                 // 有ELF32 / ELF64
  Data:                              2's complement, little endian 
  Version:                           1 (current)  // always 1
  OS/ABI:                            UNIX - System V  
  ABI Version:                       0
  
  // e_type members
  Type:                              EXEC (Executable file)
  // e_machine members
  Machine:                           AArch64
  // e_version members
  Version:                           0x1
  // e_entry members: 规定ELF程序的入口的虚拟地址，操作系统在加载完程序后从这个地址开始执行进程指令
  Entry point address:               0x4004c0
  // e_phoff members：ELF链接视图和执行视图相关
  Start of program headers:          64 (bytes into file)
  // e_shoff member：段表在文件中的偏移，从13521字节开始
  Start of section headers:          13520 (bytes into file)
  // e_word
  Flags:                             0x0
  // e_ehsize： ELF文件头的大小
  Size of this header:               64 (bytes)
  // e_phentsize： ELF链接视图和执行视图相关
  Size of program headers:           56 (bytes)
  // e_phnum： ELF链接视图和执行视图相关
  Number of program headers:         9
  // e_shentsize： 段表描述符的大小 一般为sizeof(Elf32_Shdr)
  Size of section headers:           64 (bytes)
  // e_shnum: 段表描述符的数量。
  Number of section headers:         38
  // e_shstrndx： 段表字符串表所在的段在段表中的下标
  Section header string table index: 37
```

ELF的类型可以参考：[elf(5) - Linux manual page (man7.org)](https://man7.org/linux/man-pages/man5/elf.5.html)

### 1.3. Section Header Table（段表）

段表的作用是，在ELF中记录每个段的基本属性（段名、段的长度、在文件中的偏移、读写权限及段的其他属性）。编译器和链接器还有装载器都需要依靠段表来定位和访问各个段的属性。在elf文件头中`e_shoff` 决定段表的存储位置。

`readelf -S main`

```
➜  work-temp readelf -S main
There are 38 section headers, starting at offset 0x34d0:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .interp           PROGBITS         0000000000400238  00000238
       000000000000001b  0000000000000000   A       0     0     1
  [ 2] .note.ABI-tag     NOTE             0000000000400254  00000254
       0000000000000020  0000000000000000   A       0     0     4
  [ 3] .hash             HASH             0000000000400278  00000278
       0000000000000028  0000000000000004   A       5     0     8
  [ 4] .gnu.hash         GNU_HASH         00000000004002a0  000002a0
       000000000000001c  0000000000000000   A       5     0     8
  [ 5] .dynsym           DYNSYM           00000000004002c0  000002c0
       0000000000000078  0000000000000018   A       6     1     8
  [ 6] .dynstr           STRTAB           0000000000400338  00000338
       0000000000000044  0000000000000000   A       0     0     1
  [ 7] .gnu.version      VERSYM           000000000040037c  0000037c
       000000000000000a  0000000000000002   A       5     0     2
  [ 8] .gnu.version_r    VERNEED          0000000000400388  00000388
       0000000000000020  0000000000000000   A       6     1     8
  [ 9] .rela.dyn         RELA             00000000004003a8  000003a8
       0000000000000018  0000000000000018   A       5     0     8
  [10] .rela.plt         RELA             00000000004003c0  000003c0
       0000000000000060  0000000000000018  AI       5    23     8
  [11] .init             PROGBITS         0000000000400420  00000420
       0000000000000018  0000000000000000  AX       0     0     4
  [12] .plt              PROGBITS         0000000000400440  00000440
       0000000000000060  0000000000000000  AX       0     0     16
  [13] .text             PROGBITS         00000000004004c0  000004c0
       0000000000000214  0000000000000000  AX       0     0     64
  [14] CARLOS_FUNC       PROGBITS         00000000004006d4  000006d4
       0000000000000030  0000000000000000  AX       0     0     4
  [15] .fini             PROGBITS         0000000000400704  00000704
       0000000000000014  0000000000000000  AX       0     0     4
  [16] .rodata           PROGBITS         0000000000400718  00000718
       000000000000002f  0000000000000000   A       0     0     8
  [17] .eh_frame_hdr     PROGBITS         0000000000400748  00000748
       000000000000005c  0000000000000000   A       0     0     4
  [18] .eh_frame         PROGBITS         00000000004007a8  000007a8
       000000000000012c  0000000000000000   A       0     0     8
  [19] .init_array       INIT_ARRAY       0000000000410de8  00000de8
       0000000000000008  0000000000000008  WA       0     0     8
  [20] .fini_array       FINI_ARRAY       0000000000410df0  00000df0
       0000000000000008  0000000000000008  WA       0     0     8
  [21] .dynamic          DYNAMIC          0000000000410df8  00000df8
       00000000000001e0  0000000000000010  WA       6     0     8
  [22] .got              PROGBITS         0000000000410fd8  00000fd8
       0000000000000010  0000000000000008  WA       0     0     8
  [23] .got.plt          PROGBITS         0000000000410fe8  00000fe8
       0000000000000038  0000000000000008  WA       0     0     8
  [24] .data             PROGBITS         0000000000411020  00001020
       0000000000000018  0000000000000000  WA       0     0     8
  [25] CARLOS_DATA       PROGBITS         0000000000411038  00001038
       0000000000000004  0000000000000000  WA       0     0     4
  [26] .bss              NOBITS           000000000041103c  0000103c
       000000000000000c  0000000000000000  WA       0     0     4
  [27] .comment          PROGBITS         0000000000000000  0000103c
       000000000000005d  0000000000000001  MS       0     0     1
  [28] .debug_aranges    PROGBITS         0000000000000000  000010a0
       0000000000000130  0000000000000000           0     0     16
  [29] .debug_info       PROGBITS         0000000000000000  000011d0
       0000000000000715  0000000000000000           0     0     1
  [30] .debug_abbrev     PROGBITS         0000000000000000  000018e5
       00000000000002a1  0000000000000000           0     0     1
  [31] .debug_line       PROGBITS         0000000000000000  00001b86
       000000000000037d  0000000000000000           0     0     1
  [32] .debug_str        PROGBITS         0000000000000000  00001f03
       00000000000004ea  0000000000000001  MS       0     0     1
  [33] .debug_loc        PROGBITS         0000000000000000  000023ed
       0000000000000182  0000000000000000           0     0     1
  [34] .debug_ranges     PROGBITS         0000000000000000  00002570
       0000000000000090  0000000000000000           0     0     16
  [35] .symtab           SYMTAB           0000000000000000  00002600
       0000000000000b10  0000000000000018          36    90     8
  [36] .strtab           STRTAB           0000000000000000  00003110
       0000000000000258  0000000000000000           0     0     1
  [37] .shstrtab         STRTAB           0000000000000000  00003368
       0000000000000161  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  p (processor specific)
```

以上表为下列结构体的数组


```C
typedef struct
{
  Elf64_Word	sh_name;		/* Section name (string tbl index) */
  Elf64_Word	sh_type;		/* Section type */
  Elf64_Xword	sh_flags;		/* Section flags */
  Elf64_Addr	sh_addr;		/* Section virtual addr at execution 如果该段可以被加载，显示为进程空间的虚拟地址，否则为0 */
  Elf64_Off		sh_offset;		/* Section file offset 这段对.bbs没有意义 */
  Elf64_Xword	sh_size;		/* Section size in bytes */
  Elf64_Word	sh_link;		/* Link to another section */
  Elf64_Word	sh_info;		/* Additional section information */
  Elf64_Xword	sh_addralign;	/* Section alignment 有的段有对齐要求，0/1表示没有对其要求 */
  Elf64_Xword	sh_entsize;		/* Entry size if section holds table */
} Elf64_Shdr;
```

#### 重定位表 Relocation Table

在其他段里面引用了一些段的地址，这些地址单独存成一个重定位表，例如printf("hello world"), 里面的字符串就要被重定位到一个单独的区域。

#### 字符串表 String Table

段名、变量名长度不固定，单独放在一个连续的区域里面，使用偏移来索引字符串。`.strtab`,`.shstrtab`字符串表， 在header里面 e_shstrndx： 段表字符串表所在的段在段表中的下标 表示

## 附录 I： 原始C文件

```c
// main.c
#include <stdio.h>

int a = 84;
int b;
const int g = 0xAA;
void func(int i)
{
    printf("helloworld!%d\n", i);
}

__attribute((section("CARLOS_DATA"))) int name = 4;
__attribute((section("CARLOS_FUNC"))) int func2 (void){
    int m = 9, n = 10;
    int q;
    q = m+n;
    return q;
}

int main(void)
{
    static int var_1 = 85;
    static int var_2;
    int c = 6;
    int d;
    func(var_1 + var_2 + c + d);
    return c;
}
```


