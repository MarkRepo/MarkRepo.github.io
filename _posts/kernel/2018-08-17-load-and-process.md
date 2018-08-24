---
title: 可执行文件的装载
categories: kernel
tags: 
  - load
  - process
---

装载涉及到的进程虚拟地址空间和页映射的概念，不在这里描述。

## 从操作系统的角度看可执行文件的装载

### 进程的建立

对操作系统来说，一个进程最关键的特征是它拥有独立的虚拟地址空间。创建一个进程，然后装载相应的可执行文件并且执行，最开始只需要做三件事：

+ 创建一个独立的虚拟地址空间。
+ 读取可执行文件头，并且建立虚拟空间与可执行文件的映射关系。
+ 将CPU的指令寄存器设置成可执行文件的入口地址，启动运行。

**创建虚拟地址空间**。由页映射机制知道，一个虚拟空间由一组页映射函数将虚拟空间的各个页映射至相应的物理空间。那么创建一个虚拟空间实际上并不是创建空间而是创建映射函数所需要的相应的数据结构，在i386 的Linux下，创建虚拟地址空间只是分配一个页目录（Page Directory）就可以了，甚至不设置页映射关系，这些映射关系等到后面程序发生页错误的时候再进行设置。  
**读取可执行文件头，并且建立虚拟空间与可执行文件的映射关系**。上一步是虚拟空间到物理内存的映射，这一步建立虚拟空间与可执行文件的映射关系。当程序执行发生页错误时，操作系统将从物理内存中分配一个物理页，然后将该“缺页”从磁盘中读取到内存中，再设置缺页的虚拟页和物理页的映射关系，这样程序才得以正常运行。但是很明显的一点是，当操作系统捕获到缺页错误时，它应知道程序当前所需要的页在可执行文件中的哪一个位置。这就是虚拟空间与可执行文件之间的映射关系。从某种角度来看，这一步是整个装载过程中最重要的一步，也是传统意义上“装载”的过程。  
**将CPU指令寄存器设置成可执行文件入口，启动运行**。操作系统通过设置CPU的指令寄存器将控制权转交给进程，由此进程开始执行。这一步在操作系统层面上比较复杂，它涉及内核堆栈和用户堆栈的切换、CPU运行权限的切换。从进程的角度看可以简单地认为操作系统执行了一条跳转指令，直接跳转到可执行文件的入口地址。
>Linux中将进程虚拟空间中的一个段叫做虚拟内存区域（VMA, Virtual Memory Area）。VMA是进程的数据结构，用于保存虚拟空间与可执行文件的映射关系，以及页的属性等。当发生页错误时，操作系统通过VMA计算出“缺页”在可执行文件中的偏移。VMA是一个很重要的概念，对于理解程序的装载执行和操作系统如何管理进程的虚拟空间有非常重要的帮助。

下图表达了三种空间之间的关系：
![page-fault](/assets/images/load-dynamic/page-fault.jpg)

## 进程虚存空间分布

### ELF文件的链接视图和执行视图

由于ELF文件被映射时，是以系统的页长度作为单位的。如果每个段（section）都单独映射，当段数量较多时，就会造成内存空间的浪费。实际上操作系并不关心可执行文件各个段所包含的实际内容，它只关心一些跟装载相关的问题，最主要的是段的权限（可读、可写、可执行）。ELF文件中，段的权限往往只有为数不多的几种组合，基本上是三种：

+ 以代码段为代表的权限为*可读可执行*的段。
+ 以数据段和BSS段为代表的权限为*可读可写*的段。
+ 以只读数据段为代表的权限为*只读*的段。

对于相同权限的段，可以把它们合并到一起当作一个段进行映射，这就是Segment的概念，一个“Segment”包含一个或多个属性类似的“Section”。
Segment从装载的角度重新划分了ELF的各个段。在将目标文件链接成可执行文件的时候，链接器会尽量把相同权限属性的段分配在同一空间。在ELF中把这些属性相似的、又连在一起的段叫做一个“Segment”，而系统正是按照“Segment”而不是“Section”来映射可执行文件的。  
下图展示segment如何节约内存空间：
![Segment.jpg](/assets/images/load-dynamic/Segment.jpg)

下面看一个实际的例子，代码如下：

```c
#include <stdlib.h>

int main()
{
    while(1){
        sleep(1000);
    }
    return 0;
}
```
使用静态连接的方式编译连接成可执行文件SectionMapping.elf。

```
$gcc -static SectionMapping.c -o SectionMapping.elf
```
使用readelf 可以看到该可执行文件有33个段：

```
$readelf -S SectionMapping.elf
There are 33 section headers, starting at offset 0x74594:

Section Headers:
  [Nr] Name              Type     Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL     00000000 000000 000000 00      0   0  0
  [ 1] .note.ABI-tag     NOTE     080480d4 0000d4 000020 00   A  0   0  4
  [ 2] .init             PROGBITS 080480f4 0000f4 000017 00  AX  0   0  4
  [ 3] .text             PROGBITS 08048110 000110 055948 00  AX  0   0 16
  [ 4] __libc_freeres_fn PROGBITS 0809da60 055a60 000a8b 00  AX  0   0 16
  [ 5] .fini             PROGBITS 0809e4ec 0564ec 00001c 00  AX  0   0  4
  [ 6] .rodata           PROGBITS 0809e520 056520 0169e8 00   A  0   0 32
  [ 7] __libc_subfreeres PROGBITS 080b4f08 06cf08 00002c 00   A  0   0  4
  [ 8] __libc_atexit     PROGBITS 080b4f34 06cf34 000004 00   A  0   0  4
  [ 9] .eh_frame         PROGBITS 080b4f38 06cf38 003a0c 00   A  0   0  4
  [10] .gcc_except_table PROGBITS 080b8944 070944 0000a1 00   A  0   0  1
  [11] .tdata            PROGBITS 080b99e8 0709e8 000010 00 WAT  0   0  4
  [12] .tbss             NOBITS   080b99f8 0709f8 000018 00 WAT  0   0  4
  [13] .ctors            PROGBITS 080b99f8 0709f8 000008 00  WA  0   0  4
  [14] .dtors            PROGBITS 080b9a00 070a00 00000c 00  WA  0   0  4
  [15] .jcr              PROGBITS 080b9a0c 070a0c 000004 00  WA  0   0  4
  [16] .data.rel.ro      PROGBITS 080b9a10 070a10 00002c 00  WA  0   0  4
  [17] .got              PROGBITS 080b9a3c 070a3c 000008 04  WA  0   0  4
  [18] .got.plt          PROGBITS 080b9a44 070a44 00000c 04  WA  0   0  4
  [19] .data             PROGBITS 080b9a60 070a60 000720 00  WA  0   0 32
  [20] .bss              NOBITS   080ba180 071180 001ad4 00  WA  0   0 32
  [21] __libc_freeres_pt NOBITS   080bbc54 071180 000014 00  WA  0   0  4
  [22] .comment          PROGBITS 00000000 071180 002df0 00      0   0  1
  [23] .debug_aranges    PROGBITS 00000000 073f70 000058 00      0   0  8
  [24] .debug_pubnames   PROGBITS 00000000 073fc8 000025 00      0   0  1
  [25] .debug_info       PROGBITS 00000000 073fed 0001ad 00      0   0  1
  [26] .debug_abbrev     PROGBITS 00000000 07419a 000066 00      0   0  1
  [27] .debug_line       PROGBITS 00000000 074200 00013d 00      0   0  1
  [28] .debug_str        PROGBITS 00000000 07433d 0000bb 01  MS  0   0  1
  [29] .debug_ranges     PROGBITS 00000000 0743f8 000048 00      0   0  8
  [30] .shstrtab         STRTAB   00000000 074440 000152 00      0   0  1
  [31] .symtab           SYMTAB   00000000 074abc 007ab0 10      32 898  4
  [32] .strtab           STRTAB   00000000 07c56c 006e68 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)
  I (info), L (link order), G (group), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
```
readelf命令也可以用来查看ELF的Segment。正如描述Section的结构叫做段表，描述Segment的结构叫程序头（Program Header），它描述了ELF文件该如何被操作系统映射到进程的虚拟空间：

```
$ readelf -l SectionMapping.elf

Elf file type is EXEC (Executable file)
Entry point 0x8048110
There are 5 program headers, starting at offset 52

Program Headers:
  Type        Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD        0x000000 0x08048000 0x08048000 0x709e5 0x709e5 R E 0x1000
  LOAD        0x0709e8 0x080b99e8 0x080b99e8 0x00798 0x02280 RW  0x1000
  NOTE        0x0000d4 0x080480d4 0x080480d4 0x00020 0x00020 R   0x4
  TLS         0x0709e8 0x080b99e8 0x080b99e8 0x00010 0x00028 R   0x4
  GNU_STACK   0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0x4

 Section to Segment mapping:
  Segment Sections...
   00     .note.ABI-tag .init .text __libc_freeres_fn .fini .rodata __libc_subfreeres __libc_atexit .eh_frame .gcc_except_table
   01     .tdata .ctors .dtors .jcr .data.rel.ro .got .got.plt .data .bss __libc_freeres_ptrs
   02     .note.ABI-tag
   03     .tdata .tbss
   04
```
可以看到共有5个Segment。从装载的角度看，只关心两个LOAD类型的Segment，因为只有它是需要被映射的，其他的诸如NOTE、TLS、GNU_STACK都是在装载时起辅助作用的，这里忽略。  
下图展示了该执行文件的段与进程虚拟空间的映射关系：
![Segment2.jpg](/assets/images/load-dynamic/VMA.png)

SectionMapping.elf被重新划分成了三个部分：可读可执行的，被映射到VMA0；可读可写的，被映射到VMA1；其他的，包含调试信息和字符串表等段，这些段在程序执行时没有用，所以不需要被映射。可以看到，一个Segment映射到一个VMA。  
Segment和Section从不同的角度划分ELF文件，称为不同的视图（View），Section对应链接视图，Segment对应执行视图。在谈到ELF装载时，“段”专门指“Segment”；而在其他的情况下，“段”指的是“Section”。

ELF可执行文件中的程序头表也是一个结构体数组，结构体如下所示

```c
typedef struct
{
  Elf32_Word    p_type;                 /* Segment type */
  Elf32_Off     p_offset;               /* Segment file offset */
  Elf32_Addr    p_vaddr;                /* Segment virtual address */
  Elf32_Addr    p_paddr;                /* Segment physical address */
  Elf32_Word    p_filesz;               /* Segment size in file */
  Elf32_Word    p_memsz;                /* Segment size in memory */
  Elf32_Word    p_flags;                /* Segment flags */
  Elf32_Word    p_align;                /* Segment alignment */
} Elf32_Phdr;
```
结构体的成员与`readelf –l`的结果一一对应。各个成员的基本含义，如下表所示
![segment-struct](/assets/images/load-dynamic/segment-struct.jpg)

对于LOAD类型的Segment来说，`p_memsz`的值不可以小于`p_filesz`，否则就是不符合常理的。如果`p_memsz`大于`p_filesz`，表示该Segment在内存中所分配的空间大小超过文件中实际的大小，多余的部分全部填充为0。这样在构造ELF可执行文件时不需要再额外设立BSS的Segment，可以把数据Segment的`p_memsz`扩大，额外的部分就是BSS。如前面例子中，BSS就已经被合并到了数据类型的段中。

### 堆和栈

操作系统使用VMA管理进程的地址空间。例如程序的堆（heap），栈（stack）在虚拟空间中也是以VMA的形式存在的。在Linux下可以通过`/proc`来查看进程的虚拟空间分布：

```
$ ./SectionMapping.elf &
[1] 21963
$ cat /proc/21963/maps
08048000-080b9000 r-xp 00000000 08:01 2801887    ./SectionMapping.elf
080b9000-080bb000 rwxp 00070000 08:01 2801887    ./SectionMapping.elf
080bb000-080de000 rwxp 080bb000 00:00 0          [heap]
bf7ec000-bf802000 rw-p bf7ec000 00:00 0          [stack]
ffffe000-fffff000 r-xp 00000000 00:00 0          [vdso]
```
上面的输出结果中：第一列是VMA的*地址范围*；第二列是VMA的*权限*，p表示私有（COW, Copy on Write），s表示共享。第三列是*偏移*，表示VMA对应的Segment在映像文件中的偏移；第四列表示映像文件所在设备的*主设备号和次设备号*；第五列表示映像文件的*节点号*。最后一列是映像文件的*路径*。  
前两个VMA映射到可执行文件中的Segment。另外三个段的文件所在设备主设备号和次设备号及文件节点号都是0，表示它们没有映射到文件中，这种VMA叫做匿名虚拟内存区域。其中两个区域分别是堆（Heap）和栈（Stack）。  
操作系统通过给进程空间划分出一个个VMA来管理进程的虚拟空间；基本原则是将相同权限属性的、有相同映像文件的映射成一个VMA；一个进程基本上可以分为如下几种VMA区域：

+ 代码VMA，权限只读、可执行；有映像文件。
+ 数据VMA，权限可读写、可执行；有映像文件。
+ 堆VMA，权限可读写、可执行；无映像文件，匿名，可向上扩展。
+ 栈VMA，权限可读写、不可执行；无映像文件，匿名，可向下扩展。

一个常见进程的虚拟空间分布如下图所示：
![heap-stack](/assets/images/load-dynamic/heap-stack.png)

>`/proc`目录里面看到的VMA2的结束地址跟预测的不一样，按照`readelf -S`的输出计算应该是0x080bc000，但实际上是0x080bb000。这是因为Linux在装载ELF文件时实现了一种Hack的做法，因为Linux的进程虚拟空间管理的VMA的概念并非与Segment完全对应，Linux规定一个VMA可以映射到某个文件的一个区域，或者是没有映射到任何文件；而我们这里的第二个Segment要求是，前面部分映射到文件中，而后面一部分不映射到任何文件，直接为0，也就是说前面的从`.tdata`段到`.data`段部分要建立从虚拟空间到文件的映射，而`.bss`和`__libcfreeres_ptrs`部分不要映射到文件。这样这两个概念就不完全相同了，所以Linux实际上采用了一种取巧的办法，它在映射完第二个`Segment`之后，把最后一个页面的剩余部分清0，然后调用内核中的`do_brk()`，把`.bss`和`__libcfreeres_ptrs`的剩余部分放到堆段中。有兴趣的读者可以阅读位于Linux内核源代码`fs/Binfmt_elf.c`中的`load_elf_interp()`和`elf_map()`两个函数。

### 段地址对齐

对于Intel 80x86系列处理器来说，默认的页大小为4096字节。在物理内存和进程虚拟地址空间之间建立映射关系时，内存空间的长度必须是4 096的整数倍，并且这段空间在物理内存和进程虚拟地址空间中的起始地址必须是4 096的整数倍。由于有着长度和起始地址的限制，对于可执行文件来说，它应该尽量地优化自己的空间和地址的安排，以节省空间。  
假设有一个ELF可执行文件，它有三个段（Segment）需要装载。如下表：
![SEG0-SEG1-SEG2.png](/assets/images/load-dynamic/SEG0-SEG1-SEG2.png)
最简单的映射办法是每个段分开映射，对于长度不足一个页的部分则占一个页。按照这样的映射方式，各个段的虚拟地址和长度如下图
![SEG0-SEG1-SEG2-addr.png](/assets/images/load-dynamic/SEG0-SEG1-SEG2-addr.png)
![segment-not-merge](/assets/images/load-dynamic/segment-not-merge.png)
三个段的总长度只有12 014字节，却占据了5个页，即20480字节，空间使用率只有58.6%。  
为了解决这种问题，UNIX系统让那些各个段接壤部分共享一个物理页面，然后将该物理页面分别映射两次。而且将ELF的文件头也看作是系统的一个段，将其映射到进程的地址空间，这样做的好处是进程中的某一段区域就是整个ELF文件的映像，对于一些须访问ELF文件头的操作（比如动态链接器就须读取ELF文件头）可以直接通过读写内存地址空间进行。从某种角度看，好像是整个ELF文件从文件最开始到某个点结束，被逻辑上分成了以4096字节为单位的若干个块，每个块都被装载到物理内存中，对于那些位于两个段中间的块，它们将会被映射两次。如下图所示：
![SEG0-SEG1-SEG2-addr.png](/assets/images/load-dynamic/SEG0-SEG1-SEG2-map.png)
![segment-not-merge](/assets/images/load-dynamic/segment-merge.png)
因为段地址对齐的关系，各个段的虚拟地址就往往不是系统页面长度的整数倍了。比如在SectionMapping.elf的例子中，为什么VMA1的起始地址是0x080B99E8？而不是0x080B89E8或干脆是0x080B9000？  
VMA0的起始地址是0x08048000，长度是0x709E5，所以它的结束地址是0x080B89E5。而VMA1因为跟VMA0的最后一个虚拟页面共享一个物理页面，并且映射两遍，所以它的虚拟地址应该是0x080B99E5，又因为段必须是4字节的倍数，则向上取整至0x080B99E8。  
根据上面的段对齐方案推算出一个规律：在ELF文件中，对于任何一个可装载的Segment，`p_vaddr`与`p_offset`关于对齐属性(页大小)同余。

### 进程堆栈初始化

操作系统在进程启动前会将跟进程运行环境相关的信息提前保存到进程的虚拟空间的栈中（也就是VMA中的Stack VMA），如系统环境变量和进程的运行参数。
假设系统中有两个环境变量：`HOME=/home/user；PATH=/usr/bin`。运行该程序的命令行是：`$ prog 123`并且假设堆栈段底部地址为0xBF802000，进程初始化后的堆栈如图所示:
![process-stack](/assets/images/load-dynamic/process-stack.png)
栈顶寄存器esp指向的位置是初始化以后堆栈的顶部，最前面的4个字节表示命令行参数的数量，紧接的就是指向参数字符串的指针；后面跟了一个0；接着是两个指向环境变量字符串的指针,后面紧跟一个0表示结束。进程启动以后，系统库会把堆栈里的初始化信息中的参数信息传递给main()函数，即argc和argv。

## Linux内核装载ELF过程简介

当我们在Linux系统的bash下输入一个命令执行某个ELF程序时，Linux系统是怎样装载这个ELF文件并且执行它的呢？  
首先在用户层面，bash进程会调用fork()系统调用创建一个新的进程，然后新的进程调用execve()系统调用执行指定的ELF文件，原先的bash进程继续返回等待刚才启动的新进程结束，然后继续等待用户输入命令。execve()系统调用被定义在unistd.h，它的原型如下：  
`int execve(const char *filename, char *const argv[], char *const envp[]);`  
它的三个参数分别是被执行的程序文件名、执行参数和环境变量。  
在进入execve()系统调用之后，Linux内核就开始进行真正的装载工作。在内核中，execve()系统调用相应的入口是`sys_execve()`，它被定义在arch\i386\kernel\Process.c。`sys_execve()`进行一些参数的检查复制之后，调用`do_execve()`。`do_execve()`会首先查找被执行的文件，如果找到文件，则读取文件的前128个字节，其目的是判断文件的格式，比如a.out，ELF，脚本程序等。
然后调用`search_binary_handle()`去搜索和匹配合适的可执行文件装载处理过程。Linux中所有被支持的可执行文件格式都有相应的装载处理过程，`search_binary_handle()`会通过判断文件头部的魔数确定文件的格式，并且调用相应的装载处理过程。比如ELF可执行文件的装载处理过程叫做`load_elf_binary()`；a.out可执行文件的装载处理过程叫做`load_aout_binary()`；而装载可执行脚本程序的处理过程叫做`load_script()`。`load_elf_binary()`被定义在fs/Binfmt_elf.c，这个函数的代码比较长，它的主要步骤是：

1. 检查ELF可执行文件格式的有效性，比如魔数、程序头表中段（Segment）的数量。
2. 寻找动态链接的“.interp”段，设置动态链接器路径。
3. 根据ELF可执行文件的程序头表的描述，对ELF文件进行映射，比如代码、数据、只读数据。
4. 初始化ELF进程环境，比如进程启动时EDX寄存器的地址应该是DT_FINI的地址（参照动态链接）。
5. 将系统调用的返回地址修改成ELF可执行文件的入口点，这个入口点取决于程序的链接方式，对于静态链接的ELF可执行文件，这个程序入口就是ELF文件的文件头中e_entry所指的地址；对于动态链接的ELF可执行文件，程序入口点是动态链接器。

当`load_elf_binary()`执行完毕，返回至`do_execve()`再返回至`sys_execve()`时，上面的第5步中已经把系统调用的返回地址改成了被装载的ELF程序的入口地址了。所以当sys_execve()系统调用从内核态返回到用户态时，EIP寄存器直接跳转到了ELF程序的入口地址，于是新的程序开始执行，ELF可执行文件装载完成。

