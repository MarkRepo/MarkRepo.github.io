---
title: 链接、装载与库 --- 目标文件里有什么
descriptions: 
categories: kernel
tags: ELF
---

## 目标文件的格式

常用的可执行文件格式包括windows下的PE(Portable Executable)和Linux的ELF(Executable Linkable Format),它们都是COFF(Common file format)的变种。目标文件与可执行文件的内容与结构很相似，从广义上可以看成是一种类型的文件。此外，动态链接库和静态链接库也按照可执行文件格式存储。ELF文件标准将系统中使用ELF格式的文件分为以下4类：

+ 可重定位文件： 包含代码和数据，可用于连接成可执行文件或共享目标文件，静态库可归为这类； 例如 Linux 的 `.o`文件。
+ 可执行文件： 包含可以直接执行的程序。
+ 共享目标文件： 包含代码和数据，在两种情况下使用：
	+ 链接器使用共享目标文件跟可重定位文件链接，产生新的目标文件
	+ 动态链接器将共享目标文件与可执行文件结合，作为进程映像的一部分来运行
+ 核心转储文件： 进程意外终止时，系统将进程的地址空间的内容及终止时的一些其他信息转储到Coredump文件。

使用File命令查看相应的文件格式：

```shell
[root@centos6 link-test]# file SimpleSection.o
SimpleSection.o: ELF 32-bit LSB relocatable, Intel 80386, version 1 (SYSV), not stripped
[root@centos6 link-test]# file /bin/bash
/bin/bash: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.18, stripped
[root@centos6 link-test]# file /lib/ld-2.12.so
/lib/ld-2.12.so: ELF 32-bit LSB shared object, Intel 80386, version 1 (SYSV), dynamically linked, not stripped
```

## ELF 文件结构描述

以下所有描述基于如下实例：

```c
/*
 * SimpleSection.c
 * 
 * Linux:
 *   gcc -c SimpleSection.c
 *
 * Windows:
 *   cl SimpleSection.c /c /Za
 */

int printf(const char* format, ...);

int global_init_var = 84;
int global_uninit_var;

void func1( int i )
{
	printf("%d\n", i);
}

int main(void)
{
	static int static_var = 85;
	static int static_var2;

	int a = 1;
	int b;

	func1( static_var + static_var2 + a + b);

	return a;
}
```
>以下分析基于32位Intel x86平台下的ELF文件格式

使用GCC编译该文件 `gcc -c SimpleSection.c`, 得到 SimpleSection.o 文件， 大小为1104 字节（跟编译器版本和机器平台有关）。

### 文件头

EFL目标文件的最前部是ELF文件头，它描述了整个文件的基本属性。紧接着是ELF文件各个段(Section)。  
与段有关的重要结构是段表，该表描述了ELF文件中包含的所有段的信息。  
使用readelf命令详细查看ELF文件，如下所示()：

```
[root@centos6 link-test]# readelf -h SimpleSection.o
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              REL (Relocatable file)
  Machine:                           Intel 80386
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          0 (bytes into file)
  Start of section headers:          280 (bytes into file)
  Flags:                             0x0
  Size of this header:               52 (bytes)
  Size of program headers:           0 (bytes)
  Number of program headers:         0
  Size of section headers:           40 (bytes)
  Number of section headers:         11
  Section header string table index: 8
```

ELF文件头结构及相关常数和类型定义在`/usr/include/elf.h`中，包含32位和64位版本，部分定义如下所示：

```c
/* Type for a 16-bit quantity.  */
typedef uint16_t Elf32_Half;
typedef uint16_t Elf64_Half;

/* Types for signed and unsigned 32-bit quantities.  */
typedef uint32_t Elf32_Word;
typedef int32_t  Elf32_Sword;
typedef uint32_t Elf64_Word;
typedef int32_t  Elf64_Sword;

/* Types for signed and unsigned 64-bit quantities.  */
typedef uint64_t Elf32_Xword;
typedef int64_t  Elf32_Sxword;
typedef uint64_t Elf64_Xword;
typedef int64_t  Elf64_Sxword;

/* Type of addresses.  */
typedef uint32_t Elf32_Addr;
typedef uint64_t Elf64_Addr;

/* Type of file offsets.  */
typedef uint32_t Elf32_Off;
typedef uint64_t Elf64_Off;

/* Type for section indices, which are 16-bit quantities.  */
typedef uint16_t Elf32_Section;

/* Types for signed and unsigned 64-bit quantities.  */
typedef uint64_t Elf32_Xword;
typedef int64_t  Elf32_Sxword;
typedef uint64_t Elf64_Xword;
typedef int64_t  Elf64_Sxword;

/* Type of addresses.  */
typedef uint32_t Elf32_Addr;
typedef uint64_t Elf64_Addr;

/* Type of file offsets.  */
typedef uint32_t Elf32_Off;
typedef uint64_t Elf64_Off;

/* Type for section indices, which are 16-bit quantities.  */
typedef uint16_t Elf32_Section;
typedef uint16_t Elf64_Section;

/* Type for version symbol information.  */
typedef Elf32_Half Elf32_Versym;
typedef Elf64_Half Elf64_Versym;

#define EI_NIDENT (16)

/*ELF header*/

typedef struct
{
  unsigned char e_ident[EI_NIDENT];     /* Magic number and other info */
  Elf32_Half    e_type;                 /* Object file type */
  Elf32_Half    e_machine;              /* Architecture */
  Elf32_Word    e_version;              /* Object file version */
  Elf32_Addr    e_entry;                /* Entry point virtual address */
  Elf32_Off     e_phoff;                /* Program header table file offset */
  Elf32_Off     e_shoff;                /* Section header table file offset */
  Elf32_Word    e_flags;                /* Processor-specific flags */
  Elf32_Half    e_ehsize;               /* ELF header size in bytes */
  Elf32_Half    e_phentsize;            /* Program header table entry size */
  Elf32_Half    e_phnum;                /* Program header table entry count */
  Elf32_Half    e_shentsize;            /* Section header table entry size */
  Elf32_Half    e_shnum;                /* Section header table entry count */
  Elf32_Half    e_shstrndx;             /* Section header string table index */
} Elf32_Ehdr;

//文件类型，成员e_type的值
#define ET_REL          1               /* Relocatable file */
#define ET_EXEC         2               /* Executable file */
#define ET_DYN          3               /* Shared object file */
#define ET_CORE         4               /* Core file */

//ELF文件类型定义，Magic的第5位
#define EI_CLASS        4               /* File class byte index */
#define ELFCLASSNONE    0               /* Invalid class */
#define ELFCLASS32      1               /* 32-bit objects */
#define ELFCLASS64      2               /* 64-bit objects */
#define ELFCLASSNUM     3

//字节序定义，Magic的第6位
#define EI_DATA         5               /* Data encoding byte index */
#define ELFDATANONE     0               /* Invalid data encoding */
#define ELFDATA2LSB     1               /* 2's complement, little endian */
#define ELFDATA2MSB     2               /* 2's complement, big endian */
#define ELFDATANUM      3

//ELF文件的主版本号， Magic的第7位
#define EV_NONE         0               /* Invalid ELF version */
#define EV_CURRENT      1               /* Current version */
#define EV_NUM          2

//ELF平台属性，成员e_machine的值
#define EM_NONE          0              /* No machine */
#define EM_M32           1              /* AT&T WE 32100 */
#define EM_SPARC         2              /* SUN SPARC */
#define EM_386           3              /* Intel 80386 */
#define EM_68K           4              /* Motorola m68k family */
#define EM_88K           5              /* Motorola m88k family */
#define EM_860           7              /* Intel 80860 */
#define EM_MIPS          8              /* MIPS R3000 big-endian */
#define EM_S370          9              /* IBM System/370 */
#define EM_MIPS_RS3_LE  10              /* MIPS R3000 little-endian */
```

除了`Elf32_Ehdr` 中的 `e_ident` 这个成员对应readelf 输出结果中的Magic，与"Class", "Data","Version","OS/ABI"和"ABI Version" 这5个参数相关，剩下的参数与"Elf32_Ehdr"中的成员一一对应。

+ ELF魔数： readelf 输出中Magic的16个字节： `7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00`   
前4个字节是所有ELF文件都必须相同的标识码，0x7F是ASCII字符里面的DEL控制符，0x45、0x4c、0x46, 分别对应'E'、'L'、'F'。这4个字节称为ELF文件的魔数，几乎所有的可执行文件格式的最开始的几个字节都是魔数，如a.out格式最开始的两个字节为 0x01、0x07;PE/COFF 文件最开始的两个字节为0x4d、0x5a, 即ASCII字符`MZ`。这种魔数用来确认文件的类型，操作系统在加载可执行文件的时候会确认魔数是否正确，如果不正确会拒绝加载。随后三个字节的含义见上面elf.h的定义，后面9个字节ELF标准没有定义，一般填0。
+ 文件类型： 系统通过这个常量来判断ELF的真正的文件类型，而不是通过文件的扩展名。相关常量见上面elf.h的定义。
+ 机器类型： ELF文件格式被设计成可在多个平台下使用，但不表示同一个ELF文件可以在不同的平台使用，而是表示不同平台下的EFL文件都遵循同一套ELF标准。e_machine成员表示ELF文件的平台属性，相关常量在elf.h中定义。

### 段表(Section Header Table)

段表是保存ELF文件中所有段的基本属性的结构，是ELF文件中除了文件头以外最重要的结构。编译器、链接器和装载器都是依靠段表来定位和访问各个段的属性的。段表在ELF文件中的位置由ELF文件头的"e_shoff"成员决定。  
使用readelf工具查看ELF的各个段，如下所示：

```
[root@centos6 link-test]# readelf -S SimpleSection.o
There are 11 section headers, starting at offset 0x110:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .text             PROGBITS        00000000 000034 00005b 00  AX  0   0  4
  [ 2] .rel.text         REL             00000000 000428 000028 08      9   1  4
  [ 3] .data             PROGBITS        00000000 000090 000008 00  WA  0   0  4
  [ 4] .bss              NOBITS          00000000 000098 000004 00  WA  0   0  4
  [ 5] .rodata           PROGBITS        00000000 000098 000004 00   A  0   0  1
  [ 6] .comment          PROGBITS        00000000 00009c 00002a 00   0  0   1  0
  [ 7] .note.GNU-stack   PROGBITS        00000000 0000c6 000000 00      0   1  0
  [ 8] .shstrtab         STRTAB          00000000 0000c6 000051 00      0   1  0
  [ 9] .symtab           SYMTAB          00000000 0002d0 0000f0 10     10  10  4
  [10] .strtab           STRTAB          00000000 0003c0 000066 00      0   1  0
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)
  I (info), L (link order), G (group), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
```
上面的输出结果就是ELF文件段表的内容。段表结构简单，是一个以`Elf32_Shdr`结构体为元素的数组。数组元素的个数等于段的个数， 每个`Elf32_Shdr`结构体对应一个段。`Elf32_Shdr`又被称为段描述符(Section Descriptor)。第一个元素是无效的段描述符，类型为NULL。  
段描述符结构的定义如下所示：

```c
/* Section header.  */

typedef struct
{
  Elf32_Word    sh_name;                /* Section name (string tbl index) */
  Elf32_Word    sh_type;                /* Section type */
  Elf32_Word    sh_flags;               /* Section flags */
  Elf32_Addr    sh_addr;                /* Section virtual addr at execution */
  Elf32_Off     sh_offset;              /* Section file offset */
  Elf32_Word    sh_size;                /* Section size in bytes */
  Elf32_Word    sh_link;                /* Link to another section */
  Elf32_Word    sh_info;                /* Additional section information */
  Elf32_Word    sh_addralign;           /* Section alignment */
  Elf32_Word    sh_entsize;             /* Entry size if section holds table */
} Elf32_Shdr;

/*说明：
 * 1. sh_name: 段名是一个字符串，位于".shstrtab"字符串表，sh_name是段名字符串在表中的偏移
 * 2. sh_addr： 段加载后在进程地址空间中的虚拟地址，如果该段不能加载，则为0
 * 3. sh_link, sh_info： 段的链接信息，详见后文
 * 4. sh_addralign： 段地址对齐，有些段对段地址对齐有要求，比如假设有个段刚开始的位置包含了一个double变量，
 *    因为Intel x86系统要求浮点数的存储地址必须是本身的整数倍,也就是说保存double变量的地址必须是8字节的整数倍。
 *    这样对一个段来说，他的sh_addr必须是8的整数倍。
 *    由于地址对齐都是2的指数倍， sh_addralign表示地址对齐数量中的指数，即 sh_addralign = 3
 *    表示对齐为2的3次方，即8倍。所以一个段的地址必须满足条件：sh_addr % (2 ** sh_addralign) = 0
 *    如果sh_addralign为0或1，表示该段没有对齐要求。
 * 5. sh_entsize： 有些段包含了一些固定大小的项，比如符号表，它包含的每个符号所占的大小都是一样的， 
 *    对于这种段，sh_entsize表示每个项的大小。如果为0，表示不包含固定大小的项。
*/

/* Legal values for sh_type (section type).  */

#define SHT_NULL          0             /* Section header table entry unused */
#define SHT_PROGBITS      1             /* Program data */
#define SHT_SYMTAB        2             /* Symbol table */
#define SHT_STRTAB        3             /* String table */
#define SHT_RELA          4             /* Relocation entries with addends */
#define SHT_HASH          5             /* Symbol hash table */
#define SHT_DYNAMIC       6             /* Dynamic linking information */
#define SHT_NOTE          7             /* Notes */
#define SHT_NOBITS        8             /* Program space with no data (bss) */
#define SHT_REL           9             /* Relocation entries, no addends */
#define SHT_SHLIB         10            /* Reserved */
#define SHT_DYNSYM        11            /* Dynamic linker symbol table */

/* Legal values for sh_flags (section flags).  */

#define SHF_WRITE            (1 << 0)   /* Writable */
#define SHF_ALLOC            (1 << 1)   /* Occupies memory during execution */
#define SHF_EXECINSTR        (1 << 2)   /* Executable */

```
>段的名字对于编译器，链接器是有意义的，但是对于操作系统来说并没有意义，对于操作系统来说，一个段该如何处理取决于他的属性和权限，即由段的类型和段的标志这两个成员决定。

对照`Elf32_Shdr` 和 `readelf -S` 的输出结果，可以看到，结构体的每一个成员对应于输出结果中从第二列Name开始的每一列。于是SimpleSection的段表和所有段的位置和长度如下图所示（图中有两处错误：第一`.strtab`没有画出，第二是`.symtab`长度标错，具体参见上面的readelf命令输出即可）：

![elf-shdr](/assets/images/compile&link/elf-shdr.png)

SectionTable 的长度为0x1b8，440个字节， 包含11个段描述符，每个段描述符40个字节，这个长度等于sizeof(`Elf32_Shdr`)。最后一个段`.rel.text`结束后，长度为0x450， 即1104，刚好是SImpleSection.o 的文件长度。中间Section Table和`.rel.text` 都因为对齐的原因，与前面的段之间分别有一个字节和两个字节的间隔。

+ 段的类型： 如前面所说，段名只在链接和编译中有意义，但它不能真正的表示段的类型。我们也可以将一个数据段命名为`.text`， 对于编译器和链接器来说，主要决定段的属性的是段的类型（`sh_type`）和段的标志位(sh_flags)。 段的类型常量在elf.h中定义。
+ 段的标志位： 表示段在进程虚拟地址空间中的属性，比如是否可写，是否可执行等。相关常量定义在elf.h中。
+ 段的链接信息： （`sh_link`, `sh_info`）, 如果段的链接类型是链接相关的(动态或静态)，比如重定位、符号表等，那么`sh_link,sh_info`所包含的意义如表所示，对于其他类型的段，这两个成员没有意义。

`sh_type` | `sh_link` | `sh_info`
:--- | :--- | :---
`SHT_DYNAMIC` | 使用的字符串表在段表中的下标 | 0
`SHT_HASH` | 使用的符号表在段表中的下标 | 0
`SHT_REL` | 使用的符号表在段表中的下标| 该重定位表所作用的段在段表中的下标
`SHT_RELA` |使用的符号表在段表中的下标| 该重定位表所作用的段在段表中的下标
`SHT_SYMTAB` | 操作系统相关 | 操作系统相关
`SHT_DYNSYM` | 操作系统相关 | 操作系统相关
`other` | SHN_UNDEF | 0

### 重定位表

链接器在处理目标文件时，须要对代码段和数据段中绝对地址的引用进行重定位，这些重定位的信息记录在ELF文件的重定位表里面，对于每个需要重定位的代码段或数据段，都会有一个相应的重定位表。如`.rel.text`是针对`.text`段的重定位表，`.data`段没有对绝对地址的引用，所以没有重定位表`.rel.data`。
一个重定位表同时是ELF中的一个段， 段的类型(`sh_type`)为`SHT_REL`，它的`sh_link`表示符号表的下标，`sh_info`表示它作用于哪个段（在段表的下标）。

### 字符串表

EFL 文件中用到了很多字符串，比如段名，变量名等，因为字符串的长度往往是不定的，所以用固定的结构来表示比较困难。一种很常见的做法是把字符串集中起来保存到一个表，然后使用字符串在表中的偏移来引用字符串。如下表所示

偏移  | +0 | +1 | +2 | +3 | +4 | +5 | +6 | +7 | +8 |+9
+0    | \0 | h|e|l|l|o|w|o|r|l
+10 | d | \0|M|y|v|a|r|i|a|b
+20 | l|e | \0

那么偏移与它们对应的字符串如下表

偏移 | 字符串
0 | 空字符串
1 | helloworld
6 | world
12 | Myvariable

通过这种方法，在ELF文件中引用字符串只需给出一个数字下标即可，不用考虑字符串长度问题。一般字符串表在ELF文件中也以段的形式保存，常见的段名`.strtab`，`.shstrtab`。这两个字符串表分别为字符串表(String Table)和段表字符串表(Section Header String Table)。字符串表用来保存普通字符串，比如符号的名字；段表字符串表用来保存段表中用到的字符串，最常见的就是段名。  
ELF文件头中`e_shstrndx`成员表示`.shstrtab` 段在段表的下标，即段表字符串表在段表的下标。
所以只要分析ELF文件头，就可以得到段表和段表字符串表的位置，从而解析整个ELF文件。

### 代码段

使用objdump工具查看代码段的内容如下(以下输出为自测，与原书有出入，但不影响描述)：

```
[root@centos6 link-test]# objdump -s -d SimpleSection.o

SimpleSection.o:     file format elf32-i386

Contents of section .text:
 0000 5589e583 ec188b45 08894424 04c70424  U......E..D$...$
 0010 00000000 e8fcffff ffc9c355 89e583e4  ...........U....
 0020 f083ec20 c7442418 01000000 8b150400  ... .D$.........
 0030 0000a100 0000008d 04020344 24180344  ...........D$..D
 0040 241c8904 24e8fcff ffff8b44 2418c9c3  $...$......D$...
....(略去无关内容)
Disassembly of section .text:

00000000 <func1>:
   0: 55                    push   %ebp
   1: 89 e5                 mov    %esp,%ebp
   3: 83 ec 18              sub    $0x18,%esp
   6: 8b 45 08              mov    0x8(%ebp),%eax
   9: 89 44 24 04           mov    %eax,0x4(%esp)
   d: c7 04 24 00 00 00 00  movl   $0x0,(%esp)
  14: e8 fc ff ff ff        call   15 <func1+0x15>
  19: c9                    leave  
  1a: c3                    ret    

0000001b <main>:
  1b: 55                    push   %ebp
  1c: 89 e5                 mov    %esp,%ebp
  1e: 83 e4 f0              and    $0xfffffff0,%esp
  21: 83 ec 20              sub    $0x20,%esp
  24: c7 44 24 18 01 00 00  movl   $0x1,0x18(%esp)
  2b: 00 
  2c: 8b 15 04 00 00 00     mov    0x4,%edx
  32: a1 00 00 00 00        mov    0x0,%eax
  37: 8d 04 02              lea    (%edx,%eax,1),%eax
  3a: 03 44 24 18           add    0x18(%esp),%eax
  3e: 03 44 24 1c           add    0x1c(%esp),%eax
  42: 89 04 24              mov    %eax,(%esp)
  45: e8 fc ff ff ff        call   46 <main+0x2b>
  4a: 8b 44 24 18           mov    0x18(%esp),%eax
  4e: c9                    leave  
  4f: c3                    ret 
```

### 数据段和只读数据段

`.data`段保存的是那些已经初始化了的全局变量和局部静态变量。SimpleSection.c中`golbal_init_varable` 和 `static_var`属于`.data`段。因此长度为8.  
SimpleSection.c里面printf，用到了字符串常量"%d\n",它是一种只读据， 被放到`.rodata`段，
可以从输出结果看到`.rodata`这个段的4个字节是这个字符串常量的ASCII字节序，最后以`\0`结尾。  
`.rodata`存放只读数据，一般是程序里面的只读变量（const 变量）和字符串常量。 单独设立`.rodata`有很多好处，不光是在语义上支持const关键字，而且操作系统在加载的时候可以将`.rodata`段的属性映射成只读，这样对于这个段的任何修改操作都会作为非法处理，保证了程序的安全性。  
有时编译器会把字符串常量放到`.data`段，而不单独放到`.rodata`段。

```
[root@centos6 link-test]# objdump -x -s -d SimpleSection.o
....
Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  1 .data         00000008  00000000  00000000  00000090  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  3 .rodata       00000004  00000000  00000000  00000098  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  ....
Contents of section .data:
 0000 54000000 55000000                    T...U...        
Contents of section .rodata:
 0000 25640a00                             %d..            
```

### BSS段

CONTENTS 表示该段在文件中存在， BSS段没有CONTENTS，表示它实际上在ELF文件中不存在内容。`.note.GNU-stack` 堆栈提示段，虽然有CONTENTS,但是Size为0，这是个很奇怪的段，暂时忽略它，认为在ELF中也不存在。

```
[root@centos6 link-test]# objdump  -x -s -d SimpleSection.o
...
Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         00000050  00000000  00000000  00000034  2**2
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
  1 .data         00000008  00000000  00000000  00000090  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000004  00000000  00000000  00000098  2**2
                  ALLOC
  3 .rodata       00000004  00000000  00000000  00000098  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .comment      0000002e  00000000  00000000  0000009c  2**0
                  CONTENTS, READONLY
  5 .note.GNU-stack 00000000  00000000  00000000  000000c6 2**0
                  CONTENTS, READONLY
```

`.bss`段存放未初始化的全局变量和局部静态变量，因为未初始化的全局变量和局部静态变量默认值都为0，所以他们在`.data`段分配空间并存放0是没有必要的。程序运行时他们的确是要占内存空间的，并且可执行文件必须记录所有未初始化的全局变量和局部静态变量的大小总和，即为`.bss`段。所以`.bss`段只是为未初始化的全局变量和局部静态变量预留位置而已，它并没有内容，所以在文件中也不占据空间。   
SimpleSection.o 中，`global_uninit_va` 和`static_var2` 就是被存放在`.bss`段，更准确的说是`.bss`段位他们预留了空间。但是如上所示该段大小只有4个字节，与这两个变量的大小8个字节不符。  
其实可以通过符号表看到，只有`static_var2`被存放在了`.bss段`， 而`global_uninit_var`没有被存放在任何段，只是一个未定义的"COMMON 符号"。这跟不同的语言和不同的编译器有关，有些编译器会将全局的未初始化变量存放在目标文件的.bss段，有些则不存放，只是预留一个未定义的全局变量符号，等到最终链接成可执行文件时再在`.bss`段分配空间。原则上讲，我们可以简单的把它当作全局未初始化变量存放在`.bss`段。另外，编译单元内部可见的静态变量（比如给`global_unint_var`加上`static`修饰）的确是存放在.bss段的。符号表如下所示：

```
[root@centos6 link-test]# objdump -x -s -d SimpleSection.o
....
SYMBOL TABLE:
00000000 l    df *ABS*  00000000 SimpleSection.c
00000000 l    d  .text  00000000 .text
00000000 l    d  .data  00000000 .data
00000000 l    d  .bss 00000000 .bss
00000000 l    d  .rodata  00000000 .rodata
00000004 l     O .data  00000004 static_var.1243
00000000 l     O .bss 00000004 static_var2.1244
00000000 l    d  .note.GNU-stack  00000000 .note.GNU-stack
00000000 l    d  .comment 00000000 .comment
00000000 g     O .data  00000004 global_init_var
00000004       O *COM*  00000004 global_uninit_var
00000000 g     F .text  0000001b func1
00000000         *UND*  00000000 printf
0000001b g     F .text  00000035 main
```

### 其他段

常用的段名 | 说明
.rodatal | 存放只读数据，如字符串常量，全局const变量。与`.rodata`一样.
.comment | 存放编译器版本信息
.debug  | 调试信息
.dynamic | 动态链接信息
.hash | 符号哈希表
.line | 调试时的行号表，即源代码行号与编译后指令的对照表
.note | 而外的编译器信息
.strtab | 字符串，存放ELF中用到的普通字符串
.symtab | 符号表
.shstrtab | 段表字符串表
.plt , .got | 动态链接的跳转表和全局入口表
.init, .fini | 程序初始化与终结代码段

这些段名都有`.`作为前缀，表示这些段的名字是系统保留的。一个ELF文件可以拥有几个相同段名的段。  
GCC提供了扩展机制，使程序员可以指定变量所处的段：

```c
__attribute__((section("FOO"))) int global = 42;
__attribute__((section("BAR"))) void foo()
{}
```

## 链接的接口： 符号

在连接中，目标文件之间的相互拼合实际上是目标文件之间对地址的引用，即对函数和变量的地址的引用。比如B.o 用到了A.o中的函数foo，就称A.o定义了函数foo，B.o引用了A.o中的函数foo。定义和引用的概念同样适用于变量。我们将函数和变量统称为符号，函数名和变量名就是符号名。  
整个链接过程都基于符号才能正确完成。每个目标文件都有一个相应的符号表用于链接中的符号管理，表中记录了目标文件中用到的所有符号。每个定义的符号都有一个相应的值，叫符号值(Symbol Value)， 对于变量和函数，符号值是他们的地址。  
将符号表中的符号分类如下：

+ 定义在本目标文件的全局符号，可以被其他目标文件引用。
+ 在本目标文件中引用的外部符号，即定义在其他目标文件的全局符号。
+ 段名，这种符号一般由编译器产生，它的值是该段的起始地址。
+ 局部符号，只在编译单元内部可见。被调试器用来分析程序或崩溃时的coredump。局部符号对连接过程没有作用，被链接器忽略。
+ 行号信息，即目标文件指令与源代码中代码行的对应关系，可选。

对于链接来说只需关注全局符号和外部符号， 段名、局部符号、行号等对于其他目标文件是不可见的。使用nm命令查看SimpleSection.o 的符号结果如下：

```c
[root@centos6 link-test]# nm SimpleSection.o
00000000 T func1
00000000 D global_init_var
00000004 C global_uninit_var
0000001b T main
         U printf
00000004 d static_var.1286
00000000 b static_var2.1287
```

### ELF 符号表结构

ELF文件中符号表往往是一个段，段名一般叫`.symtab`。符号表是一个 `Elf32_Sym`结构的数组，每个`Elf32_Sym`结构对应一个符号。数组的第一个元素是无效的未定义符号。elf.h 中定义如下：

```c
/* Symbol table entry.  */

typedef struct
{
  Elf32_Word    st_name;                /* Symbol name (string tbl index) */
  Elf32_Addr    st_value;               /* Symbol value */
  Elf32_Word    st_size;                /* Symbol size */
  unsigned char st_info;                /* Symbol type and binding */
  unsigned char st_other;               /* Symbol visibility, 0, current not used */
  Elf32_Section st_shndx;               /* Section index */
} Elf32_Sym;

/*
 *说明：
 * 1. st_info: 符号类型和绑定信息，低4位表示符号的类型，高28位表示符号绑定信息。参考下面相关的常量定义。
 * 2. st_shndx: 符号所在段，如果符号定义在本目标文件中，那么这个成员表示符号所在的段在段表的下标；
 *    如果不是定义在本目标文件，或者对于有些特殊符号，sh_shndx的值有些特殊。参考下面符号所在段特殊常量。
 * 3. st_value: 见下文。
*/

/* Legal values for ST_BIND subfield of st_info (symbol binding).  */

#define STB_LOCAL       0               /* Local symbol */
#define STB_GLOBAL      1               /* Global symbol */
#define STB_WEAK        2               /* Weak symbol 弱引用符号*/

/* Legal values for ST_TYPE subfield of st_info (symbol type).  */

#define STT_NOTYPE      0               /* Symbol type is unspecified */
#define STT_OBJECT      1               /* Symbol is a data object ，如变量，数组*/
#define STT_FUNC        2               /* Symbol is a code object ，函数或其他可执行代码*/
#define STT_SECTION     3               /* Symbol associated with a section ，表示一个段，该符号必须是STB_LOCAL的*/
#define STT_FILE        4               /* Symbol's name is file name ，文件名，一般是目标文件对应的源文件名，
                                           一定是STB_LOCAL的，并且st_shndx一定是SHN_ABS*/

/**/
/* Special section indices.  符号所在段特殊常量*/

#define SHN_UNDEF       0               /* Undefined section */
#define SHN_ABS         0xfff1          /* Associated symbol is absolute */
#define SHN_COMMON      0xfff2          /* Associated symbol is common */

/*
 * 1. SHN_UNDEF: 表示该符号未定义，这个符号表示该符号在本目标文件引用到，但是定义在其他目标文件。
 * 2. SHN_ABS: 表示该符号包含一个绝对的值，比如表示文件名的符号
 * 3. SHN_COMMON: 表示该符号是一个`COMMON块`类型的符号，一般来说未初始化的全局符号定义就是这种类型的。
*/
```

符号值(`st_value`): 每个符号都有一个对应的值，对于函数和变量来说，符号值是这个函数或变量的地址。更准确的讲应该按下面几种情况区别对待：

+ 在目标文件中，如果是符号的定义并且该符号不是`COMMON块`类型的（`st_shndx`不为`SHN_COMMON`）,则`st_value`表示该符号在段中的偏移。即符号所对应的函数或变量位于有`st_shndx`指定的段，偏移`st_value`的位置。这也是目标文件中定义全局符号的最常见情况。
+ 在目标文件中，如果符号是`COMMON块`类型（`st_shndx` 为`SHN_COMMON`）,则`st_value`表示该符号的对齐属性，如SimpleSection.o中的`global_uninit_var`
+ 在可执行文件中， `st_value` 表示符号的虚拟地址。这个虚拟地址对于动态链接器十分有用。将在动态链接器部分讲述。

使用readelf命令查看SimpleSection.o 中的各个符号在符号表中的状态，如下所示：

```
[root@centos6 link-test]# readelf -s SimpleSection.o

Symbol table '.symtab' contains 15 entries:
   Num:    Value  Size Type    Bind   Vis      Ndx Name
     0: 00000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 00000000     0 FILE    LOCAL  DEFAULT  ABS SimpleSection.c
     2: 00000000     0 SECTION LOCAL  DEFAULT    1 
     3: 00000000     0 SECTION LOCAL  DEFAULT    3 
     4: 00000000     0 SECTION LOCAL  DEFAULT    4 
     5: 00000000     0 SECTION LOCAL  DEFAULT    5 
     6: 00000004     4 OBJECT  LOCAL  DEFAULT    3 static_var.1243
     7: 00000000     4 OBJECT  LOCAL  DEFAULT    4 static_var2.1244
     8: 00000000     0 SECTION LOCAL  DEFAULT    7 
     9: 00000000     0 SECTION LOCAL  DEFAULT    6 
    10: 00000000     4 OBJECT  GLOBAL DEFAULT    3 global_init_var
    11: 00000004     4 OBJECT  GLOBAL DEFAULT  COM global_uninit_var
    12: 00000000    27 FUNC    GLOBAL DEFAULT    1 func1
    13: 00000000     0 NOTYPE  GLOBAL DEFAULT  UND printf
    14: 0000001b    64 FUNC    GLOBAL DEFAULT    1 main
```
readelf 的输出格式与`Elf32_Sym`的各个成员一一对应。第6列vis目前在C/C++中未使用，暂时忽略。对于fun1和main，类型为`STT_FUNC`, Size表示函数指令所占的字节数。对于`static_var.1243`, 是使用了符号修饰的结果。对于`STT_SECTION`类型的符号，表示下标为Ndx的段的段名，上面的命令没有显示，可以使用`objdump -t`命令显示这些段名符号。

### 特殊符号

当使用ld作为链接器来连接产生可执行文件时，它会定义很多特殊的符号，这些符号并没有在你的程序中定义，但是可以直接声明并且引用它，称这些符号为特殊符号。
其实这些符号是被定义在ld链接器的连接脚本中的，参考“连接过程控制”一节。目前只需认为这些符号是特殊的，你无须定义它们，但可以声明他们并且使用。链接器会在将程序连接成可执行文件的时候将其解析成正确的值。几个代表性的符号如下：

+ `__executable_start`: 该符号为程序的起始地址，注意不是入口地址，是程序最开始的地址
+ `__etext或 _etext 或 etext`： 该符号为代码段结束地址，即代码段最末尾的地址。
+ `_edata 或 edata`： 该符号为数据段结束地址，即数据段最末尾的地址。
+ `_end or end`： 该符号为程序结束地址。

以上地址都为程序被装载时的虚拟地址， 在装载部分再讨论装载后的虚拟地址。使用实例：

```c
/*
 * SpecialSymbol.c
 */
#include <stdio.h>

extern char __executable_start[];
extern char etext[], _etext[], __etext[];
extern char edata[], _edata[];
extern char end[], _end[];

int main(int argc, char const *argv[])
{
  printf("Executable Start %X\n", __executable_start);
  printf("Text End %X %X %X\n", etext, _etext, __etext);
  printf("Data End %X %X\n", edata, _edata);
  printf("Executable End %X %X\n", end, _end);

  return 0;
}
```
```
[root@centos6 link-test]# gcc SpecialSymbol.c -o SpecialSymbol
[root@centos6 link-test]# ./SpecialSymbol 
Executable Start 8048000
Text End 8048508 8048508 8048508
Data End 8049700 8049700
Executable End 8049708 8049708
```

### 符号修饰与函数签名

为防止符号名冲突，C++引入了名称空间。

#### C++ 符号修饰

C++中的类，继承，虚机制，重载，名称空间等特性使得符号管理变得复杂。  
代码示例

```cpp
int func(int);
float func(float);

class C {
  int func(int);
  class C2 {
    int func(int);
  };
};

namespace N{
  int func(int);
  class C {
    int func(int);
  };
}
```
上面的代码包含6个同名函数func， 他们的返回类型和参数及所在的名称空间不同。  
函数签名包含了一个函数的信息，包括函数名、参数类型、所在的类和名称空间及其他信息。用于识别不同的函数。  
编译器和链接器在处理符号时，使用某种名称修饰的方法，使得每个函数签名对应一个修饰后名称，即符号名。  
上面的6个函数签名在GCC编译器下，修饰后名称如下表所示

函数签名 | 修饰后名称
:--- | :---
int func(int) | _Z4funci
float func(float) | _Z4funcf
int C::func(int) | _ZN1C4funcEi
int C::C2::func(int) | _ZN1C2C24funcEi
int N::func(int) | _ZN1N4funcEi
int N::C::func(int) | _ZN1N1C4funcEi

GCC 的基本C++名称修饰方法如下： 所有的符号都以`_Z`开头，对于嵌套的名字（在名称空间或者在类里面），后面紧跟N， 然后是各个名称空间和类的名字，每个名字前是名字字符串长度，再以`E`结尾。对于一个函数来说，它的参数列表紧跟在E后面，对于int类型参数，就是字母i。  
binutils 里面提供了一个叫`c++filt`的工具可以用来解析被修饰过的名称，如：

```
[root@centos6 link-test]# c++filt _ZN1N1C4funcEi
N::C::func(int)
```
签名和名称修饰机制不光被使用到函数上，C++中的全局变量和静态变量也有同样的机制。 全局变量跟函数一样是一个全局可见的名称，同样遵循上面的名称修饰机制。
比如名称空间foo中的全局变量bar，修饰后名字为： `_ZN3foo3barE`。注意，变量的类型没有被加到修饰后名称中，所以不论这个变量是整型，浮点型，甚至是一个全局对象，它的名称都是一样的。
名称修饰机制也被用来防止静态变量的名字冲突。比如main()和func()函数里面都有个静态变量叫foo， 为了区分这两个变量，GCC将他们的符号名分别修饰成`_ZZ4mainE3foo` 和 `_ZZ4funcvE3foo`两个不同的名字。更具体的修饰方法参考GCC名称修饰标准。  
不同的编译器厂商的名称修饰方法可能不同，所以不同的编译器对同一个函数签名可能对应不同的修饰后名称，这是导致不同的编译器之间不能互操作的原因之一。

### extern "C"

C++为了与C兼容，在符号管理上，C++有一个用来声明或定义 C 符号的`extern "C"`关键字用法：

```cpp
extern "C" {
  int func(int);
  int var;
}
```

C++ 编译器会将在extern "C" 的大括号内部的代码当作C语言代码处理，使C++的名称修饰机制不起作用。Visual C++平台下会将C语言的符号进行修饰，修饰后名称分别是`_func`, `_var`； Linux版本的GCC编译器下不对C语言符号进行修饰。 

很多时候我们会碰到有些头文件声明了一些C语言的函数和全局变量， 但是这个头文件会被C语言代码或C++代码包含。比如C语言库函数中string.h中声明了memset函数， 原型如下：  
`void *memset(void*, int, size_t);`  
如果不加任何处理，当C语言程序包含string.h的时候，并且用到了memset这个函数，编译器会将memset符号引用正确处理。
但是在C++语言中，编译器会认为memset是一个C++函数，将memset符号修饰成`_Z6memsetPvii`,这样链接器就无法与C语言库中的memset符号进行链接。所以对于C++来说，必须使用extern "C"来声明memset。但是C不支持extern "C"语法，如果为了兼容C和C++定义两套头文件，未免过于麻烦。此时，可以使用C++的宏"__cplusplus"来解决，C++编译器会在编译C++程序时默认定义这个宏，使用条件宏来判断当前编译单元是不是C++代码，具体代码如下：

```cpp
#ifdef __cplusplus
extern "C" {
#endif

void *memset(void*, int , size_t);

#ifdef __cplusplus
}
#endif
```

### 弱符号与强符号

对于C/C++来说，编译器默认函数和初始化了的全局变量为**强符号**，未初始化的全局变量为**弱符号**。可以通过GCC的`__attribute__((weak))` 来定义任何一个强符号为弱符号。注意，强符号和弱符号是针对定义来说的，不是针对符号的引用。比如：

```c
extern int ext;
int weak;
int strong = 1;
__attribute__((weak)) weak2 = 2;

int main()
{
  return 0;
}
```
weak 和weak2 是弱符号，strong和main是强符号， ext既非强符号也非弱符号，因为它是一个外部符号的引用。针对强弱符号的概念，链接器按如下规则处理与选择被多次定义的全局符号：

+ 规则1： 不允许强符号被多次定义，否则链接器报符号重定义错误。
+ 规则2： 如果一个符号在某个目标文件中是强符号，在其他文件中都是弱符号，那么选择强符号
+ 规则3： 如果一个符号在所有目标文件中都是弱符号，那么选择占用空间最大的一个。（不要使用多个不同类型的弱符号，容易导致程序错误）

目标文件在最终链接成可执行文件时，它对外部符号的引用必须被正确决议，如果没有找到该符号的定义，链接器会报符号未定义错误，这种被称为强引用。
对于弱引用，如果符号有定义，链接器将符号的引用决议；如果符号未定义，链接器对于该引用不报错，一般默认其为0，或者是一个特殊值，以便于程序代码能够识别。
弱引用和弱符号主要用于库的链接过程。  
在GCC中，通过`__attribute__((weakref))`扩展关键字声明一个外部函数的引用为弱引用。如：

```c
__attribute__((weakref)) void foo();
int main(){ foo() }
```
将其编译成一个可执行文件，GCC并不会包链接错误。但运行时报错，因为foo的地址为0，发生非法地址访问错误。改进如下：

```c
__attribute__((weakref)) void foo()
int main(){ if(foo) foo(); }
```

弱符号和弱引用对库十分有用，比如库中定于的弱符号可以被用户定义的强符号所覆盖，从而使用自定义版本库函数。或者将程序对某些扩展功能模块的引用定义为弱引用，当链接扩展模块时，功能模块可以正常使用；如果去掉该模块，也可以正常链接，只是缺少了相应的功能，使得程序的功能更加容易裁剪和组合。  
下面是程序运行时动态判断是否支持多线程从而选择单线程或者多线程版本的例子（多线程链接时有`-lpthread`选项）：

```c
#include <stdio.h>
#include <pthread.h>

int pthread_create(phtread_t*, const pthread_attr_t*, void*(*)(void*), void*) __attribute__((weak));
int main(){
  if(pthread_create){
    printf("This is multi-thread version!\n");
  }else{
    printf("This is single-thread version!\n");
  }
}
```
















