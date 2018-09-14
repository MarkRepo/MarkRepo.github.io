---
title: 程序员的自我修养(链接、装载与库) --- 静态链接
descriptions: 描述静态链接的过程
categories: kernel
tags: link
---

## 概述

本文使用如下示例，描述怎样把a.o和b.o两个目标文件链接成一个可执行文件ab。

```c
/* a.c */
extern int shared;

int main()
{
    int a = 100;
    swap(&a, &shared);
}

/* b.c */
int shared = 1;

void swap(int* a, int* b)
{
    *a ^= *b ^= *a ^= *b;
}
```
使用 `gcc -c a.c b.c` 得到a.o和b.o两个目标文件。

## 空间与地址分配

对于多个输入文件，由于按序叠加的方式会造成很多零散的段，每个段会有地址和空间的对齐要求，使内存空间产生大量的内部碎片，非常浪费空间。所以链接器采用**相似段合并**的方案合并到输出文件。如下图所示：

![section-merge](/assets/images/compile&link/section-merge.png)

`.bss`段在目标文件和可执行文件中并不占用文件的空间，但是它在装载时占用地址空间。所以链接器在合并各个段的同时，也将`.bss`段合并，并且分配虚拟空间。
链接器为目标文件分配地址和空间中的**地址和空间**有两个含义： 第一个是在输出的可执行文件中的空间；第二个是在装载后的虚拟地址中的虚拟地址空间。对于有实际数据的段，它们在文件中和虚拟地址中都要分配空间；而对于BSS段，分配空间的意义只局限于虚拟地址空间。本文描述的空间分配只关注虚拟地址空间的分配，因为这关系到链接过程中的地址计算。  
链接器采用"两步链接（Two-pass Linking）"的方法，将连接过程分两步：

+ 空间与地址分配：扫描所有的输入目标文件，获得各个段的长度、属性、位置，并将它们合并，计算合并后各个段的长度与位置，建立映射关系。收集所有输入目标文件中的符号表中所有的符号定义和符号引用，统一放到全局符号表。
+ 符号解析与重定位：使用第一步中收集到的所有信息，读取输入文件中段的数据、重定位信息，并且进行符号解析与重定位、调整代码中的地址等。重定位过程是链接过程的核心。

使用ld链接器将a.o，b.o连接起来：  
`$ld a.o b.o -e main -o ab`  (`-e main`: 将main作为程序入口，ld默认是`_start`)  
使用objdump查看连接前后地址的分配情况如下所示(数值已改成与原书一致)：

```
[root@centos6 link-test]# objdump -h a.o
....
Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         00000034  00000000  00000000  00000034  2**2
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
  1 .data         00000000  00000000  00000000  00000068  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000000  00000000  00000000  00000068  2**2
                  ALLOC
....
[root@centos6 link-test]# objdump -h b.o
....
Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         0000003a  00000000  00000000  00000034  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .data         00000004  00000000  00000000  00000074  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000000  00000000  00000000  00000078  2**2
                  ALLOC
....
[root@centos6 link-test]# objdump -h ab
....
Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         00000072  08048094  08048094  00000094  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .data         00000004  08049108  08049108  00000108  2**2
                  CONTENTS, ALLOC, LOAD, DATA
....
```
>VMA表示Virtual Memory Address,即虚拟地址，LMA表示Load Memory Address,即加载地址。正常情况下这两个值一样，但是在有些嵌入式系统中，特别是那些程序放在ROM的系统中，LMA和VMA是不同的。这里只关注VMA。

链接后程序中所使用的地址已经是程序在进程中的虚拟机地址，即VMA。链接过程前后，目标文件、可执行文件、程序虚拟地址如下图所示(图中0x08048166 应该是0x08048106)：

![VMA](/assets/images/compile&link/VMA.png)

图中只关注代码段和数据段，本例中没有bss段。 根据操作系统进程虚拟地址空间的分配规则，在linux下，ELF可执行文件默认从地址0x08048000开始分配。
经过扫描和空间分配阶段后，输入文件中的各个段在连接后的虚拟地址已经确定，因为符号在段内的相对位置是固定的，所以这时候其实所有符号的虚拟地址已经可以计算出来了。如本例中，三个全局符号的虚拟地址如下表所示：

符号 	| 类型 	| 虚拟地址
:--- 	| :--- 	| :---
main 	| 函数 	| 0x08048094
swap 	| 函数	| 0x080480c8
shared 	| 变量	| 0x08048108

## 符号解析与重定位

使用objdump查看a.o中是如何使用两个外部符号shared和swap的(测试输出与原书不一致，但不影响讨论)：

```
[root@centos6 link-test]# objdump -d a.o
a.o:     file format elf32-i386
Disassembly of section .text:

00000000 <main>:
   0:	55                   	push   %ebp
   1:	89 e5                	mov    %esp,%ebp
   3:	83 e4 f0             	and    $0xfffffff0,%esp
   6:	83 ec 20             	sub    $0x20,%esp
   9:	c7 44 24 1c 64 00 00 	movl   $0x64,0x1c(%esp)
  10:	00 
  11:	c7 44 24 04 00 00 00 	movl   $0x0,0x4(%esp)
  18:	00 
  19:	8d 44 24 1c          	lea    0x1c(%esp),%eax
  1d:	89 04 24             	mov    %eax,(%esp)
  20:	e8 fc ff ff ff       	call   21 <main+0x21>
  25:	c9                   	leave  
  26:	c3                   	ret    
```
当源代码a.c编译成a.o时，编译器并不知道shared和swap的地址。shared在偏移0x11处被mov(绝对地址指令)引用，其地址置为0x00000000；swap在偏移0x20处被call指令引用，0xe8是操作码，查阅IA-32体系开发者手册可知，这条指令是一条**近址相对位移调用指令**，后面4个字节就是被调用函数的相对于调用指令的下一条指令的偏移量。在没有重定位之前，相对偏移被置为0xFFFFFFFC，它是常量-4的补码形式，也是一个临时的假地址。链接器在完成地址和空间分配之后就可以根据符号的虚拟地址对每个需要重定位的指令进行地址修正。使用objdump查看ab的代码段如下所示（与原书不一致，但不影响讨论）：

```
[root@centos6 link-test]# objdump -d ab
ab:     file format elf32-i386
Disassembly of section .text:

08048094 <main>:
 8048094:	55                   	push   %ebp
 8048095:	89 e5                	mov    %esp,%ebp
 8048097:	83 e4 f0             	and    $0xfffffff0,%esp
 804809a:	83 ec 20             	sub    $0x20,%esp
 804809d:	c7 44 24 1c 64 00 00 	movl   $0x64,0x1c(%esp)
 80480a4:	00 
 80480a5:	c7 44 24 04 f8 90 04 	movl   $0x80490f8,0x4(%esp)
 80480ac:	08 
 80480ad:	8d 44 24 1c          	lea    0x1c(%esp),%eax
 80480b1:	89 04 24             	mov    %eax,(%esp)
 80480b4:	e8 03 00 00 00       	call   80480bc <swap>
 80480b9:	c9                   	leave  
 80480ba:	c3                   	ret    
 80480bb:	90                   	nop

080480bc <swap>:
 80480bc:	55                   	push   %ebp
...
```
经过修正以后shared的地址为0x080409f8(原书为0x08049108), swap地址为0x00000003(原书为0x00000009)(小端字节序)。详细的地址计算参考下文。

### 重定位表

链接器如何知道哪些指令要被调整？这些指令的哪些部分要被调整？如何调整？答案是重定位表。对于每个要被重定位的ELF段都有一个对应的重定位表，用来描述如何修改相应的段里的内容。查看目标文件的重定位表：

```
[root@centos6 link-test]# objdump -r a.o
a.o:     file format elf32-i386

RELOCATION RECORDS FOR [.text]:
OFFSET   TYPE              VALUE 
00000015 R_386_32          shared
00000021 R_386_PC32        swap
```
每个要被重定位的地方叫一个重定位入口（Relocation Entry），重定位入口的偏移（Offset）表示该入口在要被重定位的段中的位置。对照前面的反汇编结果，这里的0x15 和 0x21 就是代码段中mov和call指令的地址部分。

对于32位的Intel x86系列处理来说，重定位表的结构是一个`Elf32_Rel`结构的数组，每个元素对应一个重定位入口，elf.h中定义如下：

```c
/* Relocation table entry without addend (in section of type SHT_REL).  */

typedef struct
{
  Elf32_Addr    r_offset;               /* Address */
  Elf32_Word    r_info;                 /* Relocation type and symbol index */
} Elf32_Rel;
/*
 * 1. r_offset: 重定位入口的偏移。对可重定位文件来说，这个值是该重定位入口所要修正的位置的第一个字节相对于段起始的偏移。
 *              对可执行文件或共享对象文件来说，这个值是该重定位入口所要修正的位置的第一个字节的虚拟地址。
 * 2. r_info： 重定位入口的类型和符号。这个成员的低8位表示重定位入口的类型，高24位表示重定位入口的符号在符号表中的下标。
 */
```
符号解析的概念：  
重定位的过程伴随着符号解析过程，每个目标文件都可能定义一些符号，也可能引用到定义在其他目标文件的符号。重定位过程中，每个重定位的入口都是一个外部符号的引用，当链接器需要对某个符号的引用进行重定位时，他就要确定这个符号的目标地址。这时候链接器会去查找有所有输入目标文件的符号表组成的全局符号表，找到相应的符号后进行重定位，如果没有找到，就会报符号未定义的错误。

### 指令修正方式

不同的处理器指令对于地址的的格式和寻址方式都不一样，对于32为 Intel x86处理器来说jmp,call,mov指令寻址方式千差万别，主要有以下方面的区别：

+ 近址寻址或远址寻址
+ 绝对寻址或相对寻址
+ 寻址长度为8、16、32或64位

对于32位 x86平台下的ELF文件的重定位入口所修正的指令寻址方式只有两种：

+ 绝对近址32位寻址
+ 相对近址32为寻址

这两种重定位指令修正方式每个被修正的位置的长度都为32位，即4字节。而且都是近址寻址，不用考虑Intel的段间远址寻址。  
X86基本重定位类型：  

宏定义  | 值 | 重定位修正方法
:--- | :--- | :---
`R_386_32` | 1 | 绝对寻址修正 S + A
`R_386_PC32` | 2 | 相对寻址修正 S + A -P

>A = 保存在被修正位置的值  
 P = 被修正的位置相对于段开始的偏移量或者虚拟地址（对于可执行文件），该值可通过r_offset计算得到  
 S = 符号的实际地址，即由r_info的高24位指定的符号的实际地址

## COMMON块

参考"目标文件有什么"一文中链接器对强符号和弱符号的定义和处理方式来理解COMMON块机制，COMMON用于都是弱符号的情况。  
编译器和链接器支持COMMON块机制： 这种机制来源于Fortan，早期的Fortan没有动态分配空间的机制，程序员必须事先声明它所需要的临时使用空间的大小。Fortan把这种空间叫做COMMON块，当不同的目标文件需要的COMMON块空间大小不一致时，以最大的那块为准。  
现代的链接器在处理弱符号的时候采用就是与COMMON块一样的机制，如SimpleSection.c中的`global_uninit_var`。
直接导致需要COMMON块机制的原因是编译器和链接器允许不同类型的弱符号存在，但最本质的原因还是链接器不支持符号类型，即链接器无法判断各个符号的类型是否一致。  
在目标文件中，编译器为什么不直接把未初始化的全局变量也当作未初始化的局部静态变量一样处理，为它在BSS段分配空间，而是将其标记为一个COMMON类型的变量？当编译器将一个编译单元编译成目标文件的时候，如果改编译单元包含了弱符号（未初始化的全局变量就是典型的弱符号），那么该弱符号最终所占空间的大小在此时是未知的，因为有可能其他编译单元中该符号所占的空间比本编译单元所占空间要大。所以编译器此时无法为该符号在BSS段分配空间，因为所需要空间的大小未知。但是链接器在链接过程中可以确定弱符号的大小，因为当链接器读取了所有输入目标文件后，任何一个弱符号的最终大小就可以确定了，所以它可以在最终输出文件的BSS段为其分配空间。所以总体来看，未初始化的全局变量最终还是被放在BSS段的。

GCC的`-fno-common`允许把所有未初始化的全局变量不以COMMON块的形式处理，或使用`__attribute__`扩展：`int global __attribute__((nocommon));`;一旦一个未初始化的全局变量不以COMMON块的形式存在，那么它就相当于一个强符号，如果其他目标文件中还有同一个变量的强符号定义，链接时就会发生符号重定义错误。

## C++相关问题

C++复杂特性背后的数据结构在不同的编译器和链接器之间不能通用，使C++程序的二进制兼容性成了问题。

### 重复代码消除

C++编译器在很多时候会产生重复的代码，比如模板、外部内联函数和虚函数表都有可能在不同的编译单元里生成相同的代码。当模板在一个编译单元里被实例化时，它并不知道自己在别的编译单元也被实例化了。所以当一个模板在多个编译单元同时实例化成相同的类型的时候，必然会产生重复的代码。如果保留这些重复的代码，会导致空间浪费、地址出错、指令运行效率低等问题。  
一个有效的做法就是将每个模板的实例代码都单独地存放在一个段里，每个段只包含一个模板实例。比如有个模板函数`add<T>()`,某个编译单元以类型int和float实例化该模板函数，那么目标文件中会包含两个该模板实例的段，假设叫`.temp.add<int>`和`.temp.add<float>`。这样，当别的编译单元也以int或float类型实例化该模板函数后，也会生成相同的名字，这样链接器最终链接时就可以区分并删除相同的模板实例段，然后合并到最后的代码段。  
GNU GCC编译器和VISUAL C++编译器都采用了类似的方法。GCC把这种类似的需要在最终链接时合并的段叫"Linke Once",他的做法是将这种类型的段命名为`.gnu.linkonce.name`，其中name是该模板函数实例的修饰后名称。 VISUAL C++ 则把这种类型的段叫做"COMDAT"，这种段的属性字段都有`IMAGE_SCN_LNK_COMDAT`这个标记，链接器看到这个标记后，认为这个段是COMDAT类型的，会将重复的段删掉。  
外部内联函数，虚函数，默认构造函数，默认拷贝构造函数和赋值操作符也有类似的问题，解决方法与模板的类似。这种方法存在的问题是，如果相同名称的段有不同的内容，这可能由于不同的编译单元使用了不同的编译器版本或者编译优化选项，导致同一个函数编译出来的实际代码不同。这种情况下链接器可能会随意选择其中任何一个副本作为链接的输入，同时提供一个警告信息。

### 函数级别链接

如果目标文件包含成百上千个函数或变量，使用其中任意一个函数或变量都要将其整个链接进来，会导致输出文件特别大。为此，VISUAL C++编译器提供了一个编译选项叫**函数级别链接**，这个选项的作用就是让所有的函数都像前面的模板函数一样，单独保存到一个段里面。当链接器要用到某个函数时就将它合并到输出文件中，没有用到的不合并。这种做法可以很大程度减少输出文件长度，减少空间浪费。但是这个优化选项会减慢编译和连接的过程，因为链接器必须要计算各个函数之间的依赖关系，并且所有函数都保存到独立的段中，使段数量大大增加，目标文件变大，重定位过程因此变得复杂。  
GCC 提供了类似的机制，选项`-ffunction-sections`, `-fdata-sections`将每个函数或变量分别保持到独立的段中。

### C++ 与ABI

ABI(Application Binary Interface)应用程序二进制接口，是两个二进制程序模块之间接口的调用约定，包含符号修饰保准、变量内存布局、函数调用方式等这些跟可执行代码二进制兼容性相关的内容。
影响ABI的因素很多，硬件、编程语言、编译器、链接器、操作系统等都会影响ABI。对于C语言的目标代码来说，以下几个方面会决定目标文件之间是否二进制兼容：

+ 内置类型的大小和在存储器中放置方式（大端，小端，对齐方式等）
+ 组合类型（struct、union、数组等）存储方式和内存分布
+ 外部符号与用户定义的符号之间的命名方式和解析方式，如函数名func在C语言目标文件中是否被解析成外部符号`_func`
+ 函数调用方式，比如参数入栈顺序、返回值如何保持等
+ 堆栈的分布方式，比如参数和局部变量在堆栈里的位置，参数传递方法等。
+ 寄存器使用约定，函数调用时哪些寄存器可以修改，哪些需要保存等。

C++ 的二进制兼容更不容易，增加的内容：

+ 继承类体系的分布方式，如基类，虚基类在继承类中的位置等。
+ 指向成员函数的指针的内存分布，如何通过指向成员函数的指针来调用成员函数，如何传递this指针。
+ 如何调用虚函数，vtable的内容和分布形式，vtable指针在object中的位置等
+ template如何实例化
+ 外部符号的修饰
+ 全局对象的构造和析构
+ 异常的产生和捕获机制
+ 标准库的细节问题，RTTI如何实现等
+ 内嵌函数访问细节等。

## 静态库链接

静态库可以看成是一组目标文件的集合，即将很多目标文件压缩打包在一起形成一个文件，并且对其进行编号和索引，以便于查找和检索。linux上的压缩工具是ar，Visual C++上是`lib.exe`。例如使用ar查看linux上的glibc静态库libc.a：

```
[root@centos6 ~]# ar -t /usr/lib/libc.a
init-first.o
libc-start.o
sysdep.o
version.o
check_fds.o
libc-tls.o
elf-init.o
dso_handle.o
errno.o
init-arch.o
errno-loc.o
....
```
在程序链接的过程中，链接器会自动寻找所有需要的符号及他们所在的目标文件，将这些文件从`libc.a`中解压出来，最终将他们链接在一起成为一个可执行文件。
事实上，现在linux系统上的库很复杂，当我们编译和链接一个普通C程序的时候不仅要用到C语言库libc.a，还有其他一些辅助性质的目标文件和库。下面使用GCC编译hello.c,使用`-verbose`选项显示编译链接过程的详细步骤：

```
[root@centos6 link-test]# gcc -static --verbose -fno-builtin hello.c
Using built-in specs.
Target: i686-redhat-linux
Configured with: ../configure --prefix=/usr --mandir=/usr/share/man --infodir=/usr/share/info --with-bugurl=http://bugzilla.redhat.com/bugzilla --enable-bootstrap --enable-shared --enable-threads=posix --enable-checking=release --with-system-zlib --enable-__cxa_atexit --disable-libunwind-exceptions --enable-gnu-unique-object --enable-languages=c,c++,objc,obj-c++,java,fortran,ada --enable-java-awt=gtk --disable-dssi --with-java-home=/usr/lib/jvm/java-1.5.0-gcj-1.5.0.0/jre --enable-libgcj-multifile --enable-java-maintainer-mode --with-ecj-jar=/usr/share/java/eclipse-ecj.jar --disable-libjava-multilib --with-ppl --with-cloog --with-tune=generic --with-arch=i686 --build=i686-redhat-linux
Thread model: posix
gcc version 4.4.7 20120313 (Red Hat 4.4.7-23) (GCC) 
COLLECT_GCC_OPTIONS='-static' '-v' '-fno-builtin' '-mtune=generic' '-march=i686'
 （第一步）/usr/libexec/gcc/i686-redhat-linux/4.4.7/cc1 -quiet -v hello.c -quiet -dumpbase hello.c -mtune=generic -march=i686 -auxbase hello -version -fno-builtin -o /tmp/ccEd06Wk.s
ignoring nonexistent directory "/usr/lib/gcc/i686-redhat-linux/4.4.7/include-fixed"
ignoring nonexistent directory "/usr/lib/gcc/i686-redhat-linux/4.4.7/../../../../i686-redhat-linux/include"
#include "..." search starts here:
#include <...> search starts here:
 /usr/local/include
 /usr/lib/gcc/i686-redhat-linux/4.4.7/include
 /usr/include
End of search list.
GNU C (GCC) version 4.4.7 20120313 (Red Hat 4.4.7-23) (i686-redhat-linux)
	compiled by GNU C version 4.4.7 20120313 (Red Hat 4.4.7-23), GMP version 4.3.1, MPFR version 2.4.1.
GGC heuristics: --param ggc-min-expand=98 --param ggc-min-heapsize=128793
Compiler executable checksum: 791975168633d7689ef6edf2c3d79d46
COLLECT_GCC_OPTIONS='-static' '-v' '-fno-builtin' '-mtune=generic' '-march=i686'
 （第二步）as -V -Qy -o /tmp/ccRngVoZ.o /tmp/ccEd06Wk.s
GNU assembler version 2.20.51.0.2 (i686-redhat-linux) using BFD version version 2.20.51.0.2-5.42.el6 20100205
COMPILER_PATH=/usr/libexec/gcc/i686-redhat-linux/4.4.7/:/usr/libexec/gcc/i686-redhat-linux/4.4.7/:/usr/libexec/gcc/i686-redhat-linux/:/usr/lib/gcc/i686-redhat-linux/4.4.7/:/usr/lib/gcc/i686-redhat-linux/:/usr/libexec/gcc/i686-redhat-linux/4.4.7/:/usr/libexec/gcc/i686-redhat-linux/:/usr/lib/gcc/i686-redhat-linux/4.4.7/:/usr/lib/gcc/i686-redhat-linux/
LIBRARY_PATH=/usr/lib/gcc/i686-redhat-linux/4.4.7/:/usr/lib/gcc/i686-redhat-linux/4.4.7/:/usr/lib/gcc/i686-redhat-linux/4.4.7/../../../:/lib/:/usr/lib/
COLLECT_GCC_OPTIONS='-static' '-v' '-fno-builtin' '-mtune=generic' '-march=i686'
 （第三步）/usr/libexec/gcc/i686-redhat-linux/4.4.7/collect2 --build-id -m elf_i386 --hash-style=gnu -static /usr/lib/gcc/i686-redhat-linux/4.4.7/../../../crt1.o /usr/lib/gcc/i686-redhat-linux/4.4.7/../../../crti.o /usr/lib/gcc/i686-redhat-linux/4.4.7/crtbeginT.o -L/usr/lib/gcc/i686-redhat-linux/4.4.7 -L/usr/lib/gcc/i686-redhat-linux/4.4.7 -L/usr/lib/gcc/i686-redhat-linux/4.4.7/../../.. /tmp/ccRngVoZ.o --start-group -lgcc -lgcc_eh -lc --end-group /usr/lib/gcc/i686-redhat-linux/4.4.7/crtend.o /usr/lib/gcc/i686-redhat-linux/4.4.7/../../../crtn.o
```
第一步调用ccl程序，这个程序实际上是GCC的C语言编译器，它将hello.c编译成一个临时的汇编文件`/tmp/ccEd06WK.s`。然后第二步中调用as程序，as是GNU的汇编器，他将`/tmp/ccEd06Wk.s`汇编成临时目标文件`/tmp/ccRngVoZ.o`,它其实就是前面的hello.o。接着，最关键的步骤是最后一步，GCC调用collect2程序来完成最后的链接，collect2可以看成是ld链接器的一个包装，它会调用ld链接器来完成对目标文件的链接，然后再对链接结果进行一些处理，主要是收集所有与程序初始化有关的信息并且构造初始化的结构。可以看到在第三步中，至少有下列几个库和目标文件被链接入到最终可执行文件：crt1.o, crti.o, crtbeginT.o, libgcc.a, libgcc_eh.a, libc.a, crtend.o, crtn.o 。

为什么静态运行库里面一个目标文件只包含一个函数，比如libc.a里面printf.o只有printf()函数？  
链接器在链接静态库的时候是以目标文件为单位的，比如我们引用了printf函数，那么链接器就会把库中包含该函数的目标文件链接进来。如果很多函数放在一个目标文件中，很可能很多没用的函数都被遗弃链接进了输出结果中，造成空间的浪费。

## 链接过程控制

控制链接器链接过程主要有三种方法：

+ 使用命令行给链接器指定参数
+ 将链接指令放在目标文件里面，编译器经常会通过这种方法相链接器传递指令。比如VISUAL C++编译器会把链接参数放在PE目标文件的.drectve段以用来传递参数。
+ 使用链接控制脚本。

下面主要以ld链接器来介绍。VISUAL C++把链接控制脚本叫做模块定义文件，扩展名为`.def`。ld在用户没有指定连接脚本时使用默认连接脚本。使用`ld -verbose`命令查看ld默认的链接脚本。原书中指出默认链接脚本存放在/usr/lib/ldscripts，且对于可执行文件和共享库的默认链接脚本为`elf_i386.x`和`elf_i386.xs`。但是在新版的GCC中已经将默认的链接脚本集成到ld中，可以使用`readelf --string-dump=.rodata /usr/bin/ld`查看。

### 最 “小”的程序

该helloworld程序：

+ 不使用任何库
+ 不使用main，使用nomain作为程序入口
+ 使用链接脚本将程序的所有段合并到`tinytext`  

```c
char* str = "Hello world!\n";

void print()
{
    asm( "mov $13,%%edx \n\t"
        "movl %0,%%ecx \n\t"
        "movl $0,%%ebx \n\t"
        "movl $4,%%eax \n\t"
        "int $0x80     \n\t"
        ::"r"(str):"edx","ecx","ebx");
}

void exit()
{
    asm( "movl $42, %ebx \n\t"
        "movl $1,%eax \n\t"
        "int $0x80    \n\t");
}

void nomain()
{
    print();
    exit();
}
```
print函数使用了WRITE系统调用，exit使用了EXIT系统调用，使用GCC内嵌汇编实现。系统调用通过0x80中断实现，其中eax为调用号，ebx，ecx，edx等通用寄存器用来传递参数。比如WRITE调用是往一个文件句柄写入数据，原型是：  
`int write(int filedesc, char* buffer, int size)`;

+ WRITE调用的调用号为4，所以eax = 4；
+ filedesc 文件描述符，使用ebx传递，stdout=0，ebx=0
+ buffer，缓冲区地址，使用ecx传递，ecx = str
+ size 要写入的字节数，使用edx传递，edx = 13

同样的，exit系统调用号为1， 退出码为42。使用普通命令行的方式编译和链接：

```
[root@centos6 link-test]# gcc -c -fno-builtin TinyHelloWorld.c
[root@centos6 link-test]# ld -static -e nomain -o TinyHelloWorld TinyHelloWorld.o
```
得到一个1112字节大小的文件，参数解释：

+ -fno-builtin: GCC编译器提供了很多内置函数，他会把一些常用的C库函数替换成编译器的内置函数，以达到优化的功能。比如GCC会将只有字符串采纳数的printf函数替换成puts函数，以节省格式解析的时间。exit函数也是GCC的内置函数之一，所以使用该参数关闭gcc内置函数功能
+ -static :ld使用静态链接，默认是动态链接
+ -e nomain: 指定入口函数nomain，即设置ELF文件头的`e_entry`成员为nomain函数地址。

使用objdump查看,会发现它有四个段有内容，`.text .rodata .data .comment`：

```
[root@centos6 link-test]# objdump -h TinyHelloWorld
TinyHelloWorld:     file format elf32-i386

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  2 .text         0000003f  08048094  08048094  00000094  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  3 .rodata       0000000e  080480d3  080480d3  000000d3  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  5 .data         00000004  080490e4  080490e4  000000e4  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  6 .comment      0000002d  00000000  00000000  000000e8  2**0
                  CONTENTS, READONLY
```
接下来使用链接控制脚本将多个段合并成一个tinytext段。链接控制过程就是控制输入段如何变成输出段:

```
ENTRY(nomain)

SECTIONS
{
    . = 0x08048000 + SIZEOF_HEADERS;
    tinytext : { *(.text) *(.data) *(.rodata)}
    /DISCARD/ : { *(.comment)}
}
```
ENTRY指定了程序的入口，SECTIONS部分是链接脚本的主体，指定了输入段到输出段的变换规则，解释如下：

+ `. = 0x08048000 + SIZEOF_HEADERS` 设置当前虚拟地址。`.`表示当前虚拟地址，`SIZEOF_HEADERS`是输出文件的文件头大小。因为这条语句后面紧跟着输出段`tinytext`，所以tinytext的起始虚拟地址即为0x08048000 + `SIZEOF_HEADERS`。它将当前虚拟地址设置一个比较巧妙的值，以便于装载时页映射更为方便。
+ tinytext ： { *(.text) *(.data) *(.rodata)}，段转换规则，意思是将所有输入文件中的名为`.text .data .rodata` 的段合并到输出文件tinytext。
+ /DISCARD/ : { *(.comment)}，将所有输入文件中的名字为`.comment`的段丢弃，不保存到输出文件。

使用如下命令编译TinyHelloWorld：

```
[root@centos6 link-test]# gcc -c -fno-builtin TinyHelloWorld.c
[root@centos6 link-test]# ld -static -T TinyHelloWorld.lds -o TinyHelloWorld TinyHelloWorld.o
```
得到784字节的文件。使用objdump查看,确实只有一个tinytext段：

```
[root@centos6 link-test]# objdump -h TinyHelloWorld
TinyHelloWorld:     file format elf32-i386

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 tinytext      00000052  08048074  08048074  00000074  2**2
                  CONTENTS, ALLOC, LOAD, CODE
```
但是使用readelf查看：

```
[root@centos6 link-test]# readelf -S TinyHelloWorld
There are 8 section headers, starting at offset 0x108:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] tinytext          PROGBITS        08048074 000074 000052 00 WAX  0   0  4
  ....
  [ 5] .shstrtab         STRTAB          00000000 0000c8 00003d 00      0   0  1
  [ 6] .symtab           SYMTAB          00000000 000248 0000a0 10      7   6  4
  [ 7] .strtab           STRTAB          00000000 0002e8 000028 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)
  I (info), L (link order), G (group), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
```
默认情况下ld链接器会产生`.shstrtab .symtab .strtab`这三个段，对可执行文件来说，符号表和字符串表是可选的，段表字符串表是必不可少的。ld -s 参数禁止产生符号表，或使用strip命令删除符号表和字符串表。

### ld链接脚本语法简介

链接脚本有语句组成，语句分两种，一种是命令语句，另一种是赋值语句。

+ 语句之间使用分号作为分割符，对于命令语句可以使用换行来结束该语句。
+ 表达式与运算符，与C类似，比如`+ - * / += -= *= & | >> <<`等
+ 注释和字符引用，使用/**/作为注释。脚本中使用到的文件名、格式名或段名等凡是包含`;`或其他分隔符的，都要使用双引号将名字全称引用起来，如果文件名包含引号，无法处理。

命令语句一般的格式是由一个关键字和紧跟其后的参数组成。如下表：

命令语句   |   说明
:---   | :---
ENTRY(symbol) | 指定符号symbol的值为入口地址。
STARTUP(filename) | 将文件filename作为连接过程的第一个输入文件
SEARCH_DIR(path) | 将路径path加入到ld链接器的库查找目录
INPUT(file,file,...)| 将指定文件作为连接过程中的输入文件，逗号可改为空格。
INCLUDE filename | 将指定文件包含进本链接脚本
PROVIDE(symbol) | 在链接脚本中定义某个符号

>ld 有多种方法可以设置进程入口地址，按优先级排序（有高到低）  
1. ld 命令行 -e 选项  
2. 链接脚本ENTRY(symbol)命令  
3. 如果定义了`_start`符号，使用`_start`符号值  
4. 如果存在.text段，使用.text段的第一字节的地址  
5. 使用值0

SECTIONS命令的基本格式：

```
SECTINS
{
	...
	secname : { contents }
	...
}
```

secname 表示输出段名，后面必须有一个空格，以避免歧义；contents描述了一套规则和条件，表示符合这种条件的输入段将合并到这个输出段中。输出段的命名方法必须符合输出文件格式的要求。`/DISCARD/` 是一个特殊段名，表示将符合contents条件的段丢弃，不输出到文件中。contents中可以包含若干个条件，以空格分隔。如果输入段符合条件中的任意一个即表示这个输入端符合contents规则。条件写法如下：  
filename(sections)  
filename表示输入文件名，sections表示输入段名。可使用正则中的某些规则，如*,?,[] 等：

+ `file1.o(.data)`: 	file1.o 中的.data段
+ `file1.o(.data .rodata)`： 	file1.o中的.data 或 .rodata
+ `file1.o` : 	file1.o 中的所有段
+ `*(.data)`： 	所有输入文件中名字为.data的段
+ `[a-z]*(.text*[A-Z])`： 所以以小写字母开头的文件中所有段名以.text开头，以大写字母结尾的段。

## BFD 

BFD(Binary File Descriptor library)的目标是通过统一的接口处理不同的目标文件格式，它将目标文件抽象成一个统一的模型。现在GCC，ld，GDB以binutils的其他工具都是通过BFD库来处理目标文件，而不是直接操作目标文件。BFD开发库包含在binutils-devel里面，安装并使用它，下面是一个例子，输出支持的所有目标文件格式：

```c
/* target.c */
#include <stdio.h>
#include "bfd.h"

int main()
{
    const char** t = bfd_target_list();
    while(*t){
        printf("%s\n", *t);
        t++;
    }
}
```
```
[root@centos6 link-test]# gcc -o target target.c -lbfd
[root@centos6 link-test]# ./target 
elf32-i386
a.out-i386-linux
pei-i386
elf64-x86-64
elf64-l1om
elf64-little
elf64-big
elf32-little
elf32-big
srec
symbolsrec
verilog
tekhex
binary
ihex
trad-core
```







