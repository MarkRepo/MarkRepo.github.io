---
title: 程序员的自我修养(链接、装载与库) --- 动态链接
description: 描述动态链接的过程
categories: kernel
tags: dynamic-link
---

## 动态链接例子

静态链接浪费内存和磁盘空间并且更新困难。动态链接的基本思想： 把链接过程推迟到运行时进行。  
用以下例子说明动态链接的过程：

```c
/* Program1.c */
#include "Lib.h"

int main()
{   
    foobar(1);
    return 0;
}

/* Program2.c */
#include "Lib.h"

int main()
{   
  foobar(2);
    return 0;
}

/* Lib.c */
#include <stdio.h>

void foobar(int i) 
{
    printf("Printing from Lib.so %d\n", i);
}

/* Lib.h */
#ifndef LIB_H
#define LIB_H

void foobar(int i);

#endif
```
Lib.c被编译成共享对象，Programe1、Program2程序都调用了Lib.so里面的foobar()函数：

```
gcc -fPIC -shared -o Lib.so Lib.c
gcc -o Program1 Program1.c ./Lib.so
gcc -o Program2 Program2.c ./Lib.so
```
从Program1的角度看编译和链接的过程如下图：
![dynamic-Program1](/assets/images/load-dynamic/dynamic-program1.png)

Lib.so为什么要参与链接过程？  
Program1.c被编译成Program1.o时，编译器还不知道foobar()的地址，当链接器将Program1.o链接成可执行文件时，必须确定Program1.o中所引用的foobar()函数的性质。如果foobar()是一个定义于其他静态目标模块中的函数，那么链接器将会按照静态链接的规则，将Program1.o中的foobar地址引用重定位；如果foobar()是一个定义在某个动态共享对象中的函数，那么链接器就会将这个符号的引用标记为一个动态链接的符号，不对它进行地址重定位，把这个过程留到装载时再进行。  
Lib.so中保存了完整的符号信息（因为运行时进行动态链接还须使用符号信息），把Lib.so也作为链接的输入文件之一，链接器在解析符号时就可以知道：foobar是一个定义在Lib.so的动态符号。这样链接器就可以对foobar的引用做特殊的处理，使它成为一个对动态符号的引用。

为查看动态链接程序运行时地址空间分布，在foobar函数打印之后加入`sleep(-1);`，让程序停住。

```
$./Program1 &
[1] 12985
Printing from Lib.so 1
$ cat /proc/12985/maps
08048000-08049000 r-xp 00000000 08:01 1343432    ./Program1
08049000-0804a000 rwxp 00000000 08:01 1343432    ./Program1
b7e83000-b7e84000 rwxp b7e83000 00:00 0
b7e84000-b7fc8000 r-xp 00000000 08:01 1488993    /lib/tls/i686/cmov/libc-2.6.1.so
b7fc8000-b7fc9000 r-xp 00143000 08:01 1488993    /lib/tls/i686/cmov/libc-2.6.1.so
b7fc9000-b7fcb000 rwxp 00144000 08:01 1488993    /lib/tls/i686/cmov/libc-2.6.1.so
b7fcb000-b7fce000 rwxp b7fcb000 00:00 0
b7fd8000-b7fd9000 rwxp b7fd8000 00:00 0
b7fd9000-b7fda000 r-xp 00000000 08:01 1343290    ./Lib.so
b7fda000-b7fdb000 rwxp 00000000 08:01 1343290    ./Lib.so
b7fdb000-b7fdd000 rwxp b7fdb000 00:00 0
b7fdd000-b7ff7000 r-xp 00000000 08:01 1455332    /lib/ld-2.6.1.so
b7ff7000-b7ff9000 rwxp 00019000 08:01 1455332    /lib/ld-2.6.1.so
bf965000-bf97b000 rw-p bf965000 00:00 0          [stack]
ffffe000-fffff000 r-xp 00000000 00:00 0          [vdso]
$ kill 12985
[1]+  Terminated              ./Program1
```
ld-2.6.so是Linux下的动态链接器，它与普通共享对象一样被映射到了进程的地址空间，在系统开始运行Program1之前，首先会把控制权交给动态链接器，由它完成所有的动态链接工作以后再把控制权交给Program1，然后开始执行。  
使用readelf查看Lib.so的装载属性：

```
$ readelf -l Lib.so

Elf file type is DYN (Shared object file)
Entry point 0x390
There are 4 program headers, starting at offset 52

Program Headers:
  Type        Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD        0x000000 0x00000000 0x00000000 0x004e0 0x004e0 R E 0x1000
  LOAD        0x0004e0 0x000014e0 0x000014e0 0x0010c 0x00110 RW  0x1000
  DYNAMIC     0x0004f4 0x000014f4 0x000014f4 0x000c8 0x000c8 RW  0x4
  GNU_STACK   0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0x4

 Section to Segment mapping:
  Segment Sections...
00 .hash .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rel.dyn 
   .rel.plt .init .plt .text .fini 
01 .ctors .dtors .jcr .dynamic .got .got.plt .data .bss
02 .dynamic
03
```
由以上输出可以猜想共享对象的最终装载地址在编译时是不确定的，而是在装载时，装载器根据当前地址空间的空闲情况，动态分配一块足够大小的虚拟地址空间给相应的共享对象。

## 地址无关代码

### 装载时重定位

动态链接模块被装载映射至虚拟空间后，指令部分是在多个进程之间共享的，由于装载时重定位的方法需要修改指令，所以没有办法做到同一份指令被多个进程共享，因为指令被重定位后对于每个进程来讲是不同的。所以装载时重定位的方法并不适合用来解决共享对象的地址问题。动态连接库中的可修改数据部分对于不同的进程来说有多个副本，所以它们可以采用装载时重定位的方法来解决。如果只使用“-shared”，那么输出的共享对象就是使用装载时重定位的方法。

### 地址无关代码

装载时重定位是解决动态模块中有绝对地址引用的办法之一，缺点是指令部分无法在多个进程之间共享。为解决共享对象指令中对绝对地址的重定位问题，要让程序模块中共享的指令部分在装载时不需要因为装载地址的改变而改变，所以实现的基本想法就是把指令中那些需要被修改的部分分离出来，跟数据部分放在一起，这样指令部分就可以保持不变，而数据部分可以在每个进程中拥有一个副本。这种方案就是目前被称为地址无关代码（PIC, Position-independent Code）的技术。  
先来分析模块中各种类型的地址引用方式。把共享对象模块中的地址引用按照是否为跨模块分成两类：模块内部引用和模块外部引用；按照不同的引用方式分为指令引用和数据访问，得到4种情况：

+ 模块内部的函数调用、跳转等。
+ 模块内部的数据访问，比如模块中定义的全局变量、静态变量。
+ 模块外部的函数调用、跳转等。
+ 模块外部的数据访问，比如其他模块中定义的全局变量。

![address-find](/assets/images/load-dynamic/address-find.png)
编译器不能确定b和ext()在模块外部还是模块内部的其他目标文件，所以只能当模块外部来处理。

+ 模块内部调用或跳转，采用相对地址调用/跳转，不需要重定位。 关于共享对象全局符号介入（Global Symbol Interposition）问题，在“动态链接的实现”中详细介绍。
+ 模块内部数据的访问，采用相对于当前指令加上固定的偏移量来实现。现代的体系结构中，数据的相对寻址往往没有相对于当前指令地址（PC）的寻址方式，所以ELF用了一个很巧妙的办法来得到当前的PC值。ELF的共享对象里面得到PC值的方法：

```
0000044c <bar>:
 44c: 55                    push   %ebp
 44d: 89 e5                 mov    %esp,%ebp
 44f: e8 40 00 00 00        call   494 <__i686.get_pc_thunk.cx>
 454: 81 c1 8c 11 00 00     add    $0x118c,%ecx
 45a: c7 81 28 00 00 00 01  movl   $0x1,0x28(%ecx)         // a = 1
 461: 00 00 00 
 464: 8b 81 f8 ff ff ff     mov 0xfffffff8(%ecx),%eax
 46a: c7 00 02 00 00 00     movl   $0x2,(%eax)             // b = 2
 470: 5d                    pop    %ebp
 471: c3                    ret    

00000494 <__i686.get_pc_thunk.cx>:
 494: 8b 0c 24              mov    (%esp),%ecx
 497: c3                    ret 
```
这是对上面的例子中的代码先编译成共享对象然后反汇编的结果。  
44f,454,45a这三行是bar()中访问变量a的相应代码。`__i686.get_pc_thunk.cx`函数的作用就是把返回地址的值放到ecx寄存器，即把call的下一条指令的地址放到ecx寄存器。
>当处理器执行call指令以后，下一条指令的地址会被压到栈顶，而esp寄存器就是始终指向栈顶的，那么当`__i686.get_pc_thunk.cx`执行mov (%esp),%ecx的时候，返回地址就被赋值到ecx寄存器了。

接着是add和mov指令，变量a的地址是add指令地址（保存在ecx寄存器）加上两个偏移量0x118c和0x28，即如果模块被装载到0x10000000地址，那么变量a的地址是0x10000000 + 0x454 + 0x118c + 0x28 = 0x10001608。如下图所示:
![inner-data](/assets/images/load-dynamic/inner-data.png)

+ 对于模块间的数据访问，ELF会在数据段里面建立一个指向这些变量的指针数组，称为全局偏移表（Global Offset Table，GOT）。
基本机制如下图所示：
![inter-data](/assets/images/load-dynamic/inter-data.png)
当指令中需要访问变量b时，程序会先找到GOT，然后根据GOT中变量所对应的项找到变量的目标地址。每个变量都对应一个4个字节的地址，链接器在装载模块的时候会查找每个变量所在的地址，然后填充GOT中的各个项，以确保每个指针所指向的地址正确。由于GOT本身是放在数据段的，所以它可以在模块装载时被修改，并且每个进程都可以有独立的副本，相互不受影响。  
GOT如何做到指令的地址无关性？从第2种类型的数据访问可知，模块在编译时可以确定模块内部变量相对于当前指令的偏移，那么也可以在编译时确定GOT相对于当前指令的偏移，然后根据变量地址在GOT中的偏移就可以得到变量的地址。GOT中每个地址对应于哪个变量是由编译器决定的。  
回顾bar()的反汇编代码，为访问变量b，程序首先计算出变量b的地址在GOT中的位置，即0x10000000 + 0x454 + 0x118c + (-8) = 0x100015d8（0xfffffff8为-8的补码表示），然后使用寄存器间接寻址方式给变量b赋值2。  
使用objdump查看GOT的位置：

```
$ objdump -h pic.so
...
17 .got       00000010  000015d0  000015d0  000005d0  2**2
              CONTENTS, ALLOC, LOAD, DATA
...
```
可以看到GOT在文件中的偏移是0x15d0，再来看看pic.so需要在动态链接时重定位的项：

```
$ objdump -R pic.so
...
DYNAMIC RELOCATION RECORDS
OFFSET   TYPE              VALUE
...
000015d8 R_386_GLOB_DAT    b
...
```
可以看到变量b的地址需要重定位，它位于0x15d8，也就是GOT中偏移8，相当于是GOT中的第三项（每四个字节一项）。从上面重定位项中看到，变量b的地址的偏移为0x15d8，正好对应了我们前面通过指令计算出来的偏移值，即0x100015d8 – 0x10000000 = 0x15d8

+ 对于模块间调用和跳转，也可以采用模块间数据访问的方式来解决。不同的是，GOT中相应的项保存的是目标函数的地址，当模块需要调用目标函数时，可以通过GOT中的项进行间接跳转，基本的原理如下图所示：
![inter-call](/assets/images/load-dynamic/inter-call.png)
调用ext()函数的方法与上面访问变量b的方法基本类似，先得到当前指令地址PC，然后加上一个偏移得到函数地址在GOT中的偏移，然后一个间接调用：

```
call   494 <__i686.get_pc_thunk.cx>
add    $0x118c,%ecx
mov    0xfffffffc(%ecx),%eax
call   *(%eax)
```
这种方法很简单，但是存在一些性能问题，实际上ELF采用了一种更加复杂和精巧的方法，将在后面关于动态链接优化中进行更为具体的介绍。

综上，4种地址引用方式在理论上都实现了地址无关性，总结如下：
![address-table](/assets/images/load-dynamic/address-table.png)

-fpic和-fPIC  
使用GCC -fPIC参数产生地址无关代码。GCC 还有一个 -fpic 参数，功能与-fPIC完全一样。唯一的区别是，“-fPIC”产生的代码要大，而“-fpic”产生的代码相对较小，而且较快。但由于地址无关代码都是跟硬件平台相关的，不同的平台有着不同的实现，“-fpic”在某些平台上会有一些限制，比如全局符号的数量或者代码的长度等，而“-fPIC”则没有这样的限制。所以一般使用“-fPIC”参数。

如何区分一个DSO是否为PIC？  
`readelf –d foo.so | grep TEXTREL`  
如果上面的命令有任何输出，那么foo.so就不是PIC的，否则就是PIC的。PIC的DSO是不会包含任何代码段重定位表的，TEXTREL表示代码段重定位表地址。

PIC与PIE  
地址无关代码技术除了可以用在共享对象上面，它也可以用于可执行文件，一个以地址无关方式编译的可执行文件被称作地址无关可执行文件（PIE, Position-Independent Executable）。与GCC的“-fPIC”和“-fpic”参数类似，产生PIE的参数为“-fPIE”或“-fpie”

### 共享模块的全局变量问题

上面的情况中没有包含定义在模块内部的全局变量的情况。一种特殊情况：当一个模块引用了一个定义在共享对象的全局变量的时候，比如一个共享对象定义了一个全局变量global，而模块module.c中是这么引用的：

```c
extern int global;
int foo()
{
   global = 1;
}
```
如上文所说，编译器无法判断global是否为跨模块间的调用。  
假设module.c是程序可执行文件的一部分，由于程序主模块的代码并不是地址无关代码，即不使用PIC机制，它引用这个全局变量的方式跟普通数据访问方式一样，编译器会产生这样的代码：  
`movl   $0x1,XXXXXXXX`  
XXXXXXXX就是global的地址。由于可执行文件在运行时并不进行代码重定位，所以变量的地址必须在链接过程中确定下来。为了能够使得链接过程正常进行，链接器会在创建可执行文件时，在它的“.bss”段创建一个global变量的副本。导致同一个变量同时存在于多个位置中的问题。    
于是解决的办法只有一个，那就是所有的使用这个变量的指令都指向位于可执行文件中的那个副本。ELF共享库在编译时，默认都把定义在模块内部的全局变量当作定义在其他模块的全局变量，也就是说当作前面的类型四，通过GOT来实现变量的访问。当共享模块被装载时，如果某个全局变量在可执行文件中拥有副本，那么动态链接器就会把GOT中的相应地址指向该副本，这样该变量在运行时实际上最终就只有一个实例。如果变量在共享模块中被初始化，那么动态链接器还需要将该初始化值复制到程序主模块中的变量副本；如果该全局变量在程序主模块中没有副本，那么GOT中的相应地址就指向模块内部的该变量副本。  
假设module.c是一个共享对象的一部分，那么GCC编译器在-fPIC的情况下，就会把对global的调用按照跨模块模式产生代码。原因也很简单：编译器无法确定对global的引用是跨模块的还是模块内部的。即使是模块内部的，即模块内部的全局变量的引用，按照上面的结论，还是会产生跨模块代码，因为global可能被可执行文件引用，从而使得共享模块中对global的引用要使用可执行文件中的global副本。

两个进程使用同一个共享对象中定义的全局变量不会相互影响，因为共享对象的数据段在每个进程都有独立的副本。多进程共享全局变量叫共享数据段，多线程访问不同的全局变量叫线程私有存储。

### 数据段地址无关性

数据部分也有绝对地址引用的问题，如：

```c
static int a;
static int* p = &a;
```
指针p的地址就是一个绝对地址，它指向变量a，而变量a的地址会随着共享对象的装载地址改变而改变。
对于数据段来说，它在每个进程都有一份独立的副本，可以选择装载时重定位的方法来解决数据段中绝对地址引用问题。对于共享对象来说，如果数据段中有绝对地址引用，那么编译器和链接器就会产生一个重定位表，这个重定位表里面包含了`R_386_RELATIVE`类型的重定位入口。当动态链接器装载共享对象时，如果发现该共享对象有这样的重定位入口，那么动态链接器就会对该共享对象进行重定位。
>默认情况下，如果可执行文件是动态链接的，那么GCC会使用PIC的方法来产生可执行文件的代码段部分，以便于不同的进程能够共享代码段，节省内存。所以动态链接的可执行文件中存在`.got`段。（又说主程序模块不使用PIC机制？实际情况是global既会在BSS段产生一个副本，可执行文件也有`.got`段，有待深入了解。）

## 延迟绑定（PLT）

由于动态链接下对于全局数据的访问和跨模块的调用都要进行复杂的GOT定位，然后间接寻址或调用，导致程序的运行速度减慢大概1%~%5。又因为动态链接的链接工作在运行时完成，导致程序的启动速度减慢。  
程序运行过程中，会有很多函数没有用到（错误处理函数，没有使用的功能模块等），所以没有必要一开始就把所有函数都链接好，ELF采用延迟绑定的方法，基本思想是当函数第一次被用到时才由动态链接器进行绑定（符号查找，重定位等），没用到的不绑定。这提高了程序的启动速度。

ELF使用PLT（Procedure Linkage Table）来实现延迟绑定，它使用了一些很精巧的指令序列来完成。在Glibc中，动态链接器完成绑定工作的函数叫`_dl_runtime_resolve()`，它必须知道绑定发生在哪个模块中的哪个函数，因此假设其函数原型为`_dl_runtime_resolve(module, function)`。
当调用某个外部模块的函数时，并不直接通过GOT跳转，而是通过一个叫作PLT项的结构来进行跳转，每个外部函数在PLT中都有一个相应的项，比如bar()函数在PLT中的项的地址称为bar@plt。来看看bar@plt的实现：

```
bar@plt:
jmp *(bar@GOT)
push n
push moduleID
jump _dl_runtime_resolve
```
bar@plt的第一条指令是通过GOT间接跳转的指令。bar@GOT表示GOT中保存bar()这个函数相应的项。如果链接器在初始化阶段已将bar()的地址填入该项，就跳转到bar()。但为了实现延迟绑定，链接器在初始化阶段并没有将bar()的地址填入到该项，而是将上面代码中第二条指令`push n`的地址填入到bar@GOT中，这个步骤不需要查找任何符号，所以代价很低。很明显，第一条指令的效果是跳转到第二条指令，相当于没有进行任何操作。第二条指令将一个数字n压入堆栈中，这个数字是bar这个符号引用在重定位表`.rel.plt`中的下标。接着又是一条push指令将模块的ID压入到堆栈，然后跳转到`_dl_runtime_resolve`。这实际上就是在实现`_dl_runtime_resolve`函数调用，它在进行一系列符号解析和重定位工作以后将bar()的真正地址填入到bar@GOT中。  
之后当我们再次调用bar@plt时，第一条jmp指令就能够跳转到真正的bar()函数中，bar()函数返回的时候会根据堆栈里面保存的EIP直接返回到调用者，而不会再继续执行bar@plt中第二条指令开始的那段代码，那段代码只会在符号未被解析时执行一次。  

上面描述的是PLT的基本原理，PLT真正的实现要比它的结构稍微复杂一些。ELF将GOT拆分成了两个表叫做“.got”和“.got.plt”。其中“.got”用来保存全局变量引用的地址，“.got.plt”用来保存函数引用的地址，即把外部函数的引用分离到“.got.plt”中。另外“.got.plt”的前三项是有特殊意义的：

+ 第一项保存的是“.dynamic”段的地址，这个段描述了本模块动态链接相关的信息
+ 第二项保存的是本模块的ID。
+ 第三项保存的是`_dl_runtime_resolve()`的地址。

其中第二项和第三项由动态链接器在装载共享模块的时候负责将它们初始化。“.got.plt”的其余项分别对应每个外部函数的引用。如下图所示：
![got.plt](/assets/images/load-dynamic/got.plt.png)

PLT的结构也稍有不同，为了减少代码的重复，ELF把上面例子中的最后两条指令放到PLT中的第一项。并且规定每一项的长度是16个字节，刚好用来存放3条指令。实际的PLT基本结构代码如下：

```
PLT0:
push *(GOT + 4)
jump *(GOT + 8)
...
bar@plt:
jmp *(bar@GOT)
push n
jump PLT0
```
PLT在ELF文件中以独立的段存放，段名通常叫做“.plt”，因为它本身是一些地址无关的代码，所以可以跟代码段等一起合并成同一个可读可执行的“Segment”被装载入内存。
>共享对象中存在`.plt`,`.rel.plt`,`.got`,`.got.plt` 4个相关的段。

## 动态链接相关结构

动态链接情况下，操作系统首先会读取可执行文件的头部，检查文件的合法性，然后从头部中的“Program Header”中读取每个“Segment”的虚拟地址、文件地址和属性，并将它们映射到进程虚拟空间的相应位置。接着，以同样的映射的方式将动态链接器ld.so加载到进程的地址空间中，将控制权交给动态链接器的入口地址。然后动态链接器开始执行一系列自身的初始化操作，根据当前的环境参数，开始对可执行文件进行动态链接。最后动态链接器将控制权转交到可执行文件的入口地址，程序开始正式执行。

### .interp段

在动态链接的ELF可执行文件中，有一个`.interp`段，用于保存可执行文件所需要的动态链接器的路径字符串。使用objdump查看`.interp`的内容：

```
$ objdump -s a.out

a.out:     file format elf32-i386

Contents of section .interp:
 8048114 2f6c6962 2f6c642d 6c696e75 782e736f  /lib/ld-linux.so
 8048124 2e3200                                     .2.
```
在Linux的系统中，/lib/ld-linux.so.2通常是一个软链接，指向真正的动态链接器。  
动态链接器在Linux下是Glibc的一部分，属于系统库级别的，版本号跟系统中的Glibc库版本号一样。  
当系统中的Glibc库更新时，/lib/ld-linux.so.2这个软链接就会指向到新的动态链接器，
而可执行文件本身不需要修改`.interp`中的动态链接器路径来适应系统的升级。

### .dynamic段

动态链接ELF中最重要的结构应该是`.dynamic`段，它保存了动态链接器所需要的基本信息，比如依赖于哪些共享对象、动态链接符号表的位置、动态链接重定位表的位置、共享对象初始化代码的地址等。`.dynamic`段是一个结构数组，结构定义在elf.h中：

```c
typedef struct {
    Elf32_Sword d_tag;            //类型
    union {
        Elf32_Word d_val;         //类型对应的值
        Elf32_Addr d_ptr;		  //类型对应的指针
    } d_un;
} Elf32_Dyn;

//常见类型值

#define DT_NULL         0               /* Marks end of dynamic section */
#define DT_NEEDED       1               /* Name of needed library */
#define DT_HASH         4               /* Address of symbol hash table */
#define DT_STRTAB       5               /* Address of string table */
#define DT_SYMTAB       6               /* Address of symbol table */
#define DT_RELA         7               /* Address of Rela relocs */
#define DT_RELAENT      9               /* Size of one Rela reloc */
#define DT_STRSZ        10              /* Size of string table */
#define DT_INIT         12              /* Address of init function */
#define DT_FINI         13              /* Address of termination function */
#define DT_SONAME       14              /* Name of shared object */
#define DT_RPATH        15              /* Library search path (deprecated) */
#define DT_REL          17              /* Address of Rel relocs */
#define DT_RELENT       19              /* Size of one Rel reloc */
```
`.dynamic`段可以看成是动态链接下ELF文件的“文件头”。使用readelf查看“.dynamic”段的内容：

```
$ readelf -d Lib.so

Dynamic section at offset 0x4f4 contains 21 entries:
  Tag        Type                     Name/Value
 0x00000001 (NEEDED)                  Shared library: [libc.so.6]
 0x0000000c (INIT)                    0x310
 0x0000000d (FINI)                    0x4a4
 0x00000004 (HASH)                    0xb4
 0x6ffffef5 (GNU_HASH)                0xf8
 0x00000005 (STRTAB)                  0x1f4
 0x00000006 (SYMTAB)                  0x134
 0x0000000a (STRSZ)                   139 (bytes)
 0x0000000b (SYMENT)                  16 (bytes)
 0x00000003 (PLTGOT)                  0x15c8
 0x00000002 (PLTRELSZ)                32 (bytes)
 0x00000014 (PLTREL)                  REL
 0x00000017 (JMPREL)                  0x2f0
 0x00000011 (REL)                     0x2c8
 0x00000012 (RELSZ)                   40 (bytes)
 0x00000013 (RELENT)                  8 (bytes)
 0x6ffffffe (VERNEED)                 0x298
 0x6fffffff (VERNEEDNUM)              1
 0x6ffffff0 (VERSYM)                  0x280
 0x6ffffffa (RELCOUNT)                2
 0x00000000 (NULL)                    0x0
```
Linux还提供了ldd命令查看一个程序主模块或一个共享库依赖于哪些共享库：

```
$ ldd Program1
        linux-gate.so.1 =>  (0xffffe000)
        ./Lib.so (0xb7f62000)
        libc.so.6 => /lib/tls/i686/cmov/libc.so.6 (0xb7e0d000)
        /lib/ld-linux.so.2 (0xb7f66000)
```

### 动态符号表

动态符号表，段名通常叫做`.dynsym`，用于表示模块之间的符号导入导出关系。`.dynsym`只保存了与动态链接相关的符号，`.symtab`中往往保存了所有符号，包括`.dynsym`中的符号。一般动态链接的模块同时拥有`.dynsym`和`.symtab`两个表。  
与`.symtab`类似，动态符号表也需要一些辅助的表，比如动态符号字符串表`.dynstr`。
由于动态链接在程序运行时查找符号，为了加快符号的查找过程，往往还有辅助的符号哈希表`.hash`。  
用readelf查看ELF文件的动态符号表及它的哈希表：

```
$readelf -sD Lib.so

Symbol table for image:
  Num Buc:    Value  Size   Type    Bind 		Vis   	Ndx 	Name
    9   0: 00000310   0     FUNC    GLOBAL 	  DEFAULT   9 		_init
    7   0: 000015ec   0     NOTYPE  GLOBAL 	  DEFAULT 	ABS 	_edata
    4   0: 00000000  685    FUNC    GLOBAL 	  DEFAULT 	UND 	sleep
    2   0: 00000000   0     NOTYPE  WEAK 	  DEFAULT 	UND 	_Jv_RegisterClasses
    1   0: 00000000   0     NOTYPE  WEAK 	  DEFAULT 	UND 	__gmon_start__
   10   0: 0000042c   57    FUNC 	GLOBAL 	  DEFAULT  	11 		foobar
    6   1: 000015f0   0     NOTYPE 	GLOBAL 	  DEFAULT 	ABS 	_end
   11   1: 000004a4   0     FUNC 	GLOBAL 	  DEFAULT  	12 		_fini
    5   2: 00000000  245    FUNC   	WEAK 	  DEFAULT 	UND 	__cxa_finalize
    8   2: 000015ec   0     NOTYPE 	GLOBAL 	  DEFAULT 	ABS 	__bss_start
    3   2: 00000000   57    FUNC 	GLOBAL 	  DEFAULT 	UND 	printf

Symbol table of `.gnu.hash` for image:
  Num Buc:    Value  Size   Type     Bind    Vis   Ndx Name
    6   0: 000015f0   0  	NOTYPE 	GLOBAL DEFAULT ABS _end
    7   0: 000015ec   0  	NOTYPE 	GLOBAL DEFAULT ABS _edata
    8   1: 000015ec   0  	NOTYPE 	GLOBAL DEFAULT ABS __bss_start
    9   1: 00000310   0    	FUNC 	GLOBAL DEFAULT   9 _init
   10   2: 0000042c   57    FUNC 	GLOBAL DEFAULT  11 foobar
   11   2: 000004a4   0    	FUNC 	GLOBAL DEFAULT  12 _fini
```
动态链接符号表的结构与静态链接的符号表几乎一样。

### 动态链接重定位表

在动态链接中，导入符号的地址在运行时才确定，所以需要在运行时将这些导入符号的引用修正，即需要重定位。不论是可执行文件还是共享对象，不管是否使用PIC机制，只要有导入符号，就需要重定位。对于使用PIC技术的可执行文件或共享对象来说，虽然它们的代码段不需要重定位（因为地址无关），但是数据段还包含了绝对地址的引用，因为代码段中绝对地址相关的部分被分离了出来，变成了GOT，而GOT实际上是数据段的一部分。除了GOT以外，数据段还可能包含绝对地址引用。  
动态链接的文件中，重定位表叫做`.rel.dyn`和`.rel.plt`。`.rel.dyn`是对数据引用的修正，它所修正的位置位于`.got`以及数据段；而`.rel.plt`是对函数引用的修正，它所修正的位置位于`.got.plt`。使用readelf查看一个动态链接的文件的重定位表：

```
$ readelf -r Lib.so

Relocation section '.rel.dyn' at offset 0x2c8 contains 5 entries:
 Offset     Info    Type          Sym.Value  Sym. Name
000015e4  00000008 R_386_RELATIVE
000015e8  00000008 R_386_RELATIVE
000015bc  00000106 R_386_GLOB_DAT 00000000   __gmon_start__
000015c0  00000206 R_386_GLOB_DAT 00000000   _Jv_RegisterClasses
000015c4  00000506 R_386_GLOB_DAT 00000000   __cxa_finalize

Relocation section '.rel.plt' at offset 0x2f0 contains 4 entries:
 Offset     Info    Type             Sym.Value  Sym. Name
000015d4  00000107 R_386_JUMP_SLOT   00000000   __gmon_start__
000015d8  00000307 R_386_JUMP_SLOT   00000000   printf
000015dc  00000407 R_386_JUMP_SLOT   00000000   sleep
000015e0  00000507 R_386_JUMP_SLOT   00000000   __cxa_finalize
$readelf -S Lib.so
...
[19] .got        PROGBITS     000015bc 0005bc 00000c 04  WA  0   0  4
[20] .got.plt    PROGBITS     000015c8 0005c8 00001c 04  WA  0   0  4
[21] .data       PROGBITS     000015e4 0005e4 000008 00  WA  0   0  4 
...
```
不同的重定位类型表示重定位时不同的地址计算方法，这里有几种简单的重定位入口类型：`R_386_RELATIVE`、`R_386_GLOB_DAT`和`R_386_JUMP_SLOT`。其中`R_386_GLOB_DAT`和`R_386_JUMP_SLOT` 类型表示被修正的位置只需要直接填入符号的地址即可。根据`.rel.plt`中符号的偏移，可以得到`.got.plt`的结构如下图所示：
![got.plt2](/assets/images/load-dynamic/got.plt2.png)

类似于`R_386_JUMP_SLOT`是对“`.got.plt`”的重定位，`R_386_GLOB_DAT`是对“`.got`”的重定位。
`R_386_RELATIVE`类型的重定位实际上就是基址重置（Rebasing），专门用来重定位指针变量类型。  
如果某个ELF文件是以PIC模式编译的（动态链接的可执行文件一般是PIC的），并调用了一个外部函数bar，则bar会出现在“.rel.plt ”中；而如果不是以PIC模式编译，则bar将出现在“.rel.dyn”中。看看不使用PIC的方法编译的重定位表：

```
$gcc -shared Lib.c -o Lib.so
$readelf -r Lib.so

Relocation section '.rel.dyn' at offset 0x2c8 contains 8 entries:
 Offset     Info    Type          Sym.Value  Sym. Name
0000042c  00000008 R_386_RELATIVE
000015c4  00000008 R_386_RELATIVE
000015c8  00000008 R_386_RELATIVE
00000431  00000302 R_386_PC32     00000000   printf
0000043d  00000402 R_386_PC32     00000000   sleep
000015a4  00000106 R_386_GLOB_DAT 00000000   __gmon_start__
000015a8  00000206 R_386_GLOB_DAT 00000000   _Jv_RegisterClasses
000015ac  00000506 R_386_GLOB_DAT 00000000   __cxa_finalize

Relocation section '.rel.plt' at offset 0x308 contains 2 entries:
 Offset     Info    Type             Sym.Value  Sym. Name
000015bc  00000107 R_386_JUMP_SLOT   00000000   __gmon_start__
000015c0  00000507 R_386_JUMP_SLOT   00000000   __cxa_finalize
```
“printf”和“sleep”从“.rel.plt”到了“.rel.dyn”，并且类型也从`R_386_JUMP_SLOT`变成了`R_386_PC32`。
而`R_386_RELATIVE`类型多出了一个偏移为0x0000042c的入口，这个入口是什么呢？通过对Lib.so的反汇编可以知道，这个入口是用来修正传给printf的第一个参数，即我们的字符串常量“Printing from Lib.so %d\n”的地址。为什么这个字符串常量的地址在PIC时不需要重定位而在非PIC时需要重定位呢？很明显，PIC时，这个字符串可以看作是普通的全局变量，它的地址是可以通过PIC中的相对当前指令的位置加上一个固定偏移计算出来的；而在非PIC中，代码段不再使用这种相对于当前指令的PIC方法，而是采用绝对地址寻址，所以它需要重定位。

### 动态链接时进程堆栈初始化信息

当操作系统把控制权交给动态链接器时，会把一些链接需要的必要信息传给它，保存在进程的堆栈里。比如可执行文件有几个段（“Segment”）、每个段的属性、程序的入口地址等。前面提到过，进程初始化的时候，堆栈里面保存了关于进程执行环境和命令行参数等信息。事实上，堆栈里面还保存了动态链接器所需要的一些辅助信息数组（Auxiliary Vector）。辅助信息的格式也是一个结构数组，它的结构被定义在“elf.h”：

```c
typedef struct
{
    uint32_t a_type;   //类型
    union
    {
        uint32_t a_val; //类型对应的值, union是历史遗留问题，可以当作不存在
    } a_un;
} Elf32_auxv_t;

//重要且常见的类型值，动态链接器在启动时所需要的
#define AT_NULL         0               /* End of vector */
#define AT_IGNORE       1               /* Entry should be ignored */
#define AT_EXECFD       2               /* File descriptor of program */
#define AT_PHDR         3               /* Program headers for program */
#define AT_PHENT        4               /* Size of program header entry */
#define AT_PHNUM        5               /* Number of program headers */
#define AT_PAGESZ       6               /* System page size */
#define AT_BASE         7               /* Base address of interpreter */
#define AT_FLAGS        8               /* Flags */
#define AT_ENTRY        9               /* Entry point of program */
```
辅助信息数组的结构在进程堆栈中位于环境变量指针的后面。假设辅助信息有4个，分别是：

+ AT_PHDR，值为0x08048034，程序表头位于0x08048034。
+ AT_PHENT，值为20，程序表头中每个项的大小为20字节。
+ AT_PHNUM，值为7，程序表头共有7个项。
+ AT_ENTRY，0x08048320，程序入口地址为0x08048320。

那么进程的初始化堆栈就如下图所示：
![process-stack2](/assets/images/load-dynamic/process-stack2.png)
下面的程序将堆栈中初始化的信息全部打印出来：

```c
#include <stdio.h>
#include <elf.h>

int main(int argc, char* argv[])
{
    int* p = (int*)argv;
    int i;
    Elf32_auxv_t* aux;    
    printf("Argument count: %d\n", *(p-1));
    
	for(i = 0; i < *(p-1); ++i) {
        printf("Argument %d : %s\n", i, *(p + i) );
	}

    p += i;
    p++; // skip 0
    
    printf("Environment:\n");
    while(*p) {
        printf("%s\n",*p);
        p++;
    }

    p++; // skip 0

    printf("Auxiliary Vectors:\n");
    aux = (Elf32_auxv_t*)p;
    while(aux->a_type != AT_NULL) {
        printf("Type: %02d Value: %x\n", aux->a_type, aux->a_un.a_val);
        aux++;
    }

    return 0;
}
```

## 动态链接的步骤和实现

动态链接的步骤分为3步：动态链接器自举，装载共享对象，重定位和初始化。

### 动态链接器自举

动态链接器本身也是一个共享对象，但有一些特殊性。首先，动态链接器本身不可以依赖于其他任何共享对象；其次动态链接器本身所需要的全局和静态变量的重定位工作由它本身完成。对于第一个条件，可以人为的控制在编写动态链接器时保证不使用任何系统库、运行库；对于第二个条件，动态链接器必须在启动时有一段非常精巧的代码可以完成这项艰巨的工作而同时又不能用到全局和静态变量。这种具有一定限制条件的启动代码往往被称为自举（Bootstrap）。  
动态链接器入口地址即是自举代码的入口，当操作系统将进程控制权交给动态链接器时，动态链接器的自举代码即开始执行。自举代码首先会找到它自己的GOT。而GOT的第一个入口保存的即是“.dynamic”段的偏移地址，由此找到了动态连接器本身的“.dynamic”段。通过“.dynamic”中的信息，自举代码便可以获得动态链接器本身的重定位表和符号表等，从而得到动态链接器本身的重定位入口，先将它们全部重定位。从这一步开始，动态链接器代码中才可以开始使用自己的全局变量和静态变量。  
实际上在动态链接器的自举代码中，除了不可以使用全局变量和静态变量之外，甚至不能调用函数，即动态链接器本身的函数也不能调用。因为使用PIC模式编译的共享对象，对于模块内部的函数调用也是采用跟模块外部函数调用一样的方式，即使用GOT/PLT的方式，所以在GOT/PLT没有被重定位之前，自举代码不可以使用任何全局变量，也不可以调用函数。

### 装载共享对象

完成基本自举以后，动态链接器将可执行文件和链接器本身的符号表都合并到一个符号表当中，称它为全局符号表（Global Symbol Table）。然后链接器开始寻找可执行文件所依赖的共享对象。在“.dynamic”段中，有一种类型的入口是DT_NEEDED，它所指出的是该可执行文件（或共享对象）所依赖的共享对象。由此，链接器可以列出可执行文件所需要的所有共享对象，并将这些共享对象的名字放入到一个装载集合中。然后链接器开始从集合里取一个所需要的共享对象的名字，找到相应的文件后打开该文件，读取相应的ELF文件头和“.dynamic”段，然后将它相应的代码段和数据段映射到进程空间中。如果这个ELF共享对象还依赖于其他共享对象，那么将所依赖的共享对象的名字放到装载集合中。如此循环直到所有依赖的共享对象都被装载进来为止，当然链接器可以有不同的装载顺序，如果把依赖关系看作一个图的话，那么这个装载过程就是一个图的遍历过程，链接器可能会使用深度优先或者广度优先或者其他的顺序来遍历整个图，这取决于链接器，比较常见的算法一般都是广度优先的。  
当一个新的共享对象被装载进来的时候，它的符号表会被合并到全局符号表中，所以当所有的共享对象都被装载进来的时候，全局符号表里面将包含进程中所有的动态链接所需要的符号。

符号的优先级  
在动态链接器按照各个模块之间的依赖关系，对它们进行装载并且将它们的符号并入到全局符号表时，如果两个不同的模块定义了同一个符号会怎么样？举个例子：4个共享对象a1.so、a2.so、b1.so和b2.so，源文件为a1.c、a2.c、b1.c和b2.c：

```c
/* a1.c */
#include <stdio.h>
void a(){
    printf("a1.c\n");
}

/* a2.c */
#include <stdio.h>
void a(){
    printf("a2.c\n");
}

/* b1.c */
void a();
void b1(){
    a();
}

/* b2.c */
void a();
void b2(){
    a();
}
```
a1.c和a2.c中都定义了名字为“a”的函数，b1.c和b2.c都用到了外部函数“a”，但由于源代码中没有指定依赖于哪个共享对象中的函数“a”，所以在编译时指定依赖关系，b1.so依赖于a1.so，b2.so依赖于a2.so，将b1.so与a1.so进行链接，b2.so与a2.so进行链接：

```
$ gcc -fPIC -shared a1.c -o a1.so
$ gcc -fPIC -shared a2.c -o a2.so
$ gcc -fPIC -shared b1.c a1.so -o b1.so
$ gcc -fPIC -shared b2.c a2.so -o b2.so
$ ldd b1.so
        linux-gate.so.1 =>  (0xffffe000)
        a1.so => not found
        libc.so.6 => /lib/tls/i686/cmov/libc.so.6 (0xb7e86000)
        /lib/ld-linux.so.2 (0x80000000)
$ldd b2.so
        linux-gate.so.1 =>  (0xffffe000)
        a2.so => not found
        libc.so.6 => /lib/tls/i686/cmov/libc.so.6 (0xb7e17000)
        /lib/ld-linux.so.2 (0x80000000)
```
那么当有程序同时使用b1.c中的函数b1和b2.c中的函数b2会怎么样呢？比如有程序main.c：

```c
/* main.c */
#include <stdio.h>

void b1();
void b2();

int main()
{
    b1();
    b2();
    return 0;
}
```
编译并执行：

```
$gcc main.c b1.so b2.so -o main -Xlinker -rpath ./
./main
a1.c
a1.c
```
“-XLinker –rpath ./”表示链接器在当前路径寻找共享对象，否则链接器会报无法找到a1.so和a2.so错误。  
很明显，main依赖于b1.so和b2.so；b1.so依赖于a1.so；b2.so依赖于a2.so，所以当动态链接器对main程序进行动态链接时，b1.so、b2.so、a1.so和a2.so都会被装载到进程的地址空间，并且它们中的符号都会被并入到全局符号表，通过查看进程的地址空间信息可看到：

```
$ cat /proc/14831/maps
08048000-08049000 r-xp 00000000 08:01 1344643    ./main
08049000-0804a000 rwxp 00000000 08:01 1344643    ./main
b7e83000-b7e84000 rwxp b7e83000 00:00 0
b7e84000-b7e85000 r-xp 00000000 08:01 1343481    ./a2.so
b7e85000-b7e86000 rwxp 00000000 08:01 1343481    ./a2.so
b7e86000-b7e87000 r-xp 00000000 08:01 1343328    ./a1.so
b7e87000-b7e88000 rwxp 00000000 08:01 1343328    ./a1.so
b7e88000-b7fcc000 r-xp 00000000 08:01 1488993    /lib/tls/i686/cmov/libc-2.6.1.so
b7fcc000-b7fcd000 r-xp 00143000 08:01 1488993    /lib/tls/i686/cmov/libc-2.6.1.so
b7fcd000-b7fcf000 rwxp 00144000 08:01 1488993    /lib/tls/i686/cmov/libc-2.6.1.so
b7fcf000-b7fd3000 rwxp b7fcf000 00:00 0
b7fde000-b7fdf000 r-xp 00000000 08:01 1344641    ./b2.so
b7fdf000-b7fe0000 rwxp 00000000 08:01 1344641    ./b2.so
b7fe0000-b7fe1000 r-xp 00000000 08:01 1344637    ./b1.so
b7fe1000-b7fe2000 rwxp 00000000 08:01 1344637    ./b1.so
b7fe2000-b7fe4000 rwxp b7fe2000 00:00 0
b7fe4000-b7ffe000 r-xp 00000000 08:01 1455332    /lib/ld-2.6.1.so
b7ffe000-b8000000 rwxp 00019000 08:01 1455332    /lib/ld-2.6.1.so
bfdd2000-bfde7000 rw-p bfdd2000 00:00 0          [stack]
ffffe000-fffff000 r-xp 00000000 00:00 0          [vdso]
```
这4个共享对象的确都被装载进来了，那a1.so中的函数a和a2.so中的函数a是不是冲突了呢？为什么main的输出结果是两个“a1.c”呢？也就是说a2.so中的函数a似乎被忽略了。这种一个共享对象里面的全局符号被另一个共享对象的同名全局符号覆盖的现象又被称为**共享对象全局符号介入（Global Symbol Interpose）**。  
关于全局符号介入这个问题，实际上Linux下的动态链接器是这样处理的：它定义了一个规则，那就是**当一个符号需要被加入全局符号表时，如果相同的符号名已经存在，则后加入的符号被忽略**。从动态链接器的装载顺序可以看到，它是按照广度优先的顺序进行装载的，首先是main，然后是b1.so、b2.so、a1.so，最后是a2.so。当a2.so中的函数a要被加入全局符号表时，先前装载a1.so时，a1.so中的函数a已经存在于全局符号表，那么a2.so中的函数a只能被忽略。所以整个进程中，所有对于符合“a”的引用都会被解析到a1.so中的函数a，这也是为什么main打印出的结果是两个“a1.c”而不是理想中的“a1.c”和“a2.c”。所以当程序使用大量共享对象时应该非常小心符号的重名问题。

全局符号介入与地址无关代码  
前面介绍地址无关代码时，对于第一类模块内部调用或跳转的处理时，简单地将其当作是相对地址调用/跳转。但实际上这个问题比想象中要复杂，结合全局符号介入，关于调用方式的分类的解释会更加清楚。还是拿前面“pic.c”的例子来看，由于可能存在全局符号介入的问题，foo函数对于bar的调用不能够采用第一类模块内部调用的方法，因为一旦bar函数由于全局符号介入被其他模块中的同名函数覆盖，那么foo如果采用相对地址调用的话，那个相对地址部分就需要重定位，这又与共享对象的地址无关性矛盾。所以对于bar()函数的调用，编译器只能采用第三种，即当作模块外部符号处理，bar()函数被覆盖，动态链接器只需要重定位“.got.plt”，不影响共享对象的代码段。  
为了提高模块内部函数调用的效率，有一个办法是把bar()函数变成编译单元私有函数，即使用“static”关键字定义bar()函数，这种情况下，编译器要确定bar()函数不被其他模块覆盖，就可以使用第一类的方法，即模块内部调用指令，可以加快函数的调用速度。

### 重定位和初始化

当上面的步骤完成之后，链接器开始重新遍历可执行文件和每个共享对象的重定位表，将它们的GOT/PLT中的每个需要重定位的位置进行修正。因为此时动态链接器已经拥有了进程的全局符号表，所以这个修正过程也显得比较容易，跟前面提到的地址重定位的原理基本相同。  
重定位完成之后，如果某个共享对象有“.init”段，那么动态链接器会执行“.init”段中的代码，用以实现共享对象特有的初始化过程，比如最常见的，共享对象中的C++的全局/静态对象的构造就需要通过“.init”来初始化。相应地，共享对象中还可能有“.finit”段，当进程退出时会执行“.finit”段中的代码，可以用来实现类似C++全局对象析构之类的操作。  
如果进程的可执行文件也有“.init”段，那么动态链接器不会执行它，因为可执行文件中的“.init”段和“.finit”段由程序初始化部分代码负责执行，在后面的“库”这部分详细介绍程序初始化部分。  
当完成了重定位和初始化之后，所有的准备工作就宣告完成了，所需要的共享对象也都已经装载并且链接完成了，这时候动态链接器就如释重负，将进程的控制权转交给程序的入口并且开始执行。

### Linux动态链接器实现

由装载时分析execve()系统调用可知，对于动态链接的可执行文件，内核会分析它的动态链接器地址（在“.interp”段），将动态链接器映射至进程地址空间，然后把控制权交给动态链接器。 动态连接器不仅是共享对象，还是个可执行的程序，有跟可执行文件一样的ELF文件头（包括e_entry、段表等）。
其实Linux的内核在执行execve()时不关心目标ELF文件是否可执行（文件头`e_type`是`ET_EXEC`还是`ET_DYN`），它只是简单按照程序头表里面的描述对文件进行装载然后把控制权转交给ELF入口地址（没有“.interp”就是ELF文件的`e_entry`；如果有“.interp”的话就是动态链接器的`e_entry`）。  
Linux的ELF动态链接器是Glibc的一部分，它的源代码位于Glibc的源代码的elf目录下面，它的实际入口地址位于sysdeps/i386/dl-manchine.h中的`_start`（普通程序的入口地址`_start()`在sysdeps/i386/elf/start.S）。
`_start`调用位于elf/rtld.c的`_dl_start()`函数。`_dl_start()`执行ld.so的自举。自举后调用`_dl_start_final`，收集一些基本的运行数值，进入`_dl_sysdep_start`，这个函数进行一些平台相关的处理之后就进入了`_dl_main`，这就是真正意义上的动态链接器的主函数了。`_dl_main`在一开始会进行一个判断：

```c
if (*user_entry == (ElfW(Addr)) ENTRY_POINT)
{
  //...
}
```
如果指定的用户入口地址是动态链接器本身，说明动态链接器是被当作可执行文件在执行。在这种情况下，动态链接器就会解析运行时的参数，并且进行相应的处理。`_dl_main`的主要工作就是对程序所依赖的共享对象进行装载、符号解析和重定位。  

+ 动态链接器本身是静态链接的。使用ldd来判断：

```
$ ldd /lib/ld-linux.so.2
        statically linked
```
+ 动态链接器可以是PIC的也可以不是，使用PIC会更加简单一些。一方面，如果不是PIC的话，会使得代码段无法共享，浪费内存；另一方面也会使ld.so本身初始化更加复杂，因为自举时还需要对代码段进行重定位。实际上的ld-linux.so.2是PIC的。
+ 动态链接器被当作可执行文件运行时，那么的装载地址应该是多少？  
ld.so的装载地址跟一般的共享对象没区别，即为0x00000000。这个装载地址是一个无效的装载地址，作为一个共享库，内核在装载它时会为其选择一个合适的装载地址。

## 显式运行时链接

显式运行时链接是一种更加灵活的模块加载方式，也叫做运行时加载，相应的模块叫动态状态库。
在Linux中，从文件本身的格式上来看，动态库实跟一般的共享对象没有区别。主要的区别是共享对象是由动态链接器在程序启动之前负责装载和链接的，这一系列步骤都由动态连接器自动完成，对于程序本身是透明的；而动态库的装载则是通过一系列由动态链接器提供的API，具体地讲共有4个函数：打开动态库（dlopen）、查找符号（dlsym）、错误处理（dlerror）以及关闭动态库（dlclose），程序通过这几个API对动态库进行操作。这几个API的实现是在/lib/libdl.so.2里面，它们的声明和相关常量被定义在系统标准头文件<dlfcn.h>。

### dlopen()

dlopen()用来打开一个动态库，并将其加载到进程的地址空间，完成初始化过程，它的C原型定义为：

```c
void *dlopen(const char *filename, int flag);
```
filename 是被加载的动态库的路径，如果是绝对路径，则该函数将会尝试直接打开该动态库；如果是相对路径，那么dlopen()会尝试在以一定的顺序去查找该动态库文件：

1. 查找有环境变量`LD_LIBRARY_PATH`指定的一系列目录。
2. 查找由/etc/ld.so.cache里面所指定的共享库路径。
3. /lib、/usr/lib。

如果filename为0，dlopen会返回全局符号表的句柄，即可以在运行时找到全局符号表里面的任何一个符号，并且可以执行它们，这有些类似高级语言反射（Reflection）的特性。全局符号表包括了程序的可执行文件本身、被动态链接器加载到进程中的所有共享模块以及在运行时通过dlopen打开并且使用了`RTLD_GLOBAL`方式的模块中的符号。  
flag表示函数符号的解析方式，常量`RTLD_LAZY`表示使用延迟绑定，即PLT机制；而`RTLD_NOW`表示当模块被加载时即完成所有的函数绑定工作，如果有任何未定义的符号引用的绑定工作没法完成，那么dlopen()就返回错误。两种绑定方式必须选其一。另外还有一个常量`RTLD_GLOBAL`可以跟它们一起使用（通过“|”操作），它表示将被加载的模块的全局符号合并到进程的全局符号表中，使得以后加载的模块可以使用这些符号。在调试程序的时候可以使用`RTLD_NOW`作为加载参数，因为如果模块加载时有任何符号未被绑定的话，可以使用dlerror()立即捕获到相应的错误信息；而如果使用`RTLD_LAZY`的话，这种符号未绑定的错误会在加载后发生，则难以捕获。  
dlopen的返回值是被加载的模块的句柄，这个句柄在后面使用dlsym或者dlclose时需要用到。如果加载模块失败，则返回NULL。如果模块已经通过dlopen被加载过了，那么返回的是同一个句柄。另外如果被加载的模块依赖其他模块，需要先手工加载依赖的模块。  
dlopen的加载过程基本跟动态链接器一致，在完成装载、映射和重定位以后，就会执行“.init”段的代码然后返回。

### dlsym()

dlsym函数是运行时装载的核心部分，通过这个函数找到所需要的符号。定义如下：  
`void * dlsym(void *handle, char *symbol);`  
第一个参数是由dlopen()返回的动态库的句柄；第二个参数即所要查找的符号的名字，一个以“\0”结尾的C字符串。如果dlsym()找到了相应的符号，则返回该符号的值；没有找到相应的符号，则返回NULL。dlsym()返回的值对于不同类型的符号，意义是不同的。如果查找的符号是个函数，那么它返回函数的地址；如果是个变量，它返回变量的地址；如果这个符号是个常量，那么它返回的是该常量的值。这里有一个问题是：如果常量的值刚好是NULL或者0，需要使用dlerror()函数。如果符号找到了，那么dlerror()返回NULL，如果没找到，dlerror()就会返回相应的错误信息。  
符号不仅仅是函数和变量，有时还是常量，比如表示编译单元文件名的符号等，这一般由编译器和链接器产生，而且对外不可见，但它们的确存在于模块的符号表中。dlsym()是可以查找到这些符号的，可以通过“objdump –t”来查看符号表，常量在符号表里面的类型是“`*ABS*`”。  

关于符号优先级（及全局符号介入）的问题，当多个同名符号冲突时，先装入的符号优先，把这种优先级方式称为装载序列（Load Ordering）。不管是之前由动态链接器装入的还是之后由dlopen装入的共享对象，动态链接器在进行符号的解析以及重定位时，都是采用装载序列。  
dlsym()对符号的查找优先级分两种类型。第一种是在全局符号表中进行符号查找，即dlopen()时，参数filename为NULL，那么由于全局符号表使用的装载序列，所以dlsym()使用的也是装载序列。第二种是对某个通过dlopen()打开的共享对象进行符号查找的话，那么采用叫做依赖序列（Dependency Ordering）的优先级。依赖序列以被dlopen()打开的那个共享对象为根节点，对它所有依赖的共享对象进行广度优先遍历，直到找到符号为止。

### dlerror()

每次调用dlopen()、dlsym()或dlclose()以后，都可以调用dlerror()函数来判断上一次调用是否成功。dlerror()的返回值类型是char*，如果返回NULL，则表示上一次调用成功；如果不是，则返回相应的错误消息。

### dlclose()

dlclose()的作用跟dlopen()刚好相反，它的作用是将一个已经加载的模块卸载。系统会维持一个加载引用计数器，每次使用dlopen()加载某模块时，相应的计数器加一；每次使用dlclose()卸载某模块时，相应计数器减一。只有当计数器值减到0时，模块才被真正地卸载掉。卸载的过程跟加载刚好相反，先执行“.finit”段的代码，然后将相应的符号从符号表中去除，取消进程空间跟模块的映射关系，然后关闭模块文件。

下面的例子将数学库模块用运行时加载的方法加载到进程中，获取sin()函数符号地址，调用sin()并且返回结果：

```c
#include <stdio.h>
#include <dlfcn.h>

int main(int argc, char* argv[])
{
    void* handle;
    double (*func)(double);
    char* error;

    handle = dlopen(argv[1],RTLD_NOW);
    if(handle == NULL) {
        printf("Open library %s error: %s\n", argv[1], dlerror());
        return -1;
    }

    func = dlsym(handle,"sin");
    if( (error = dlerror()) != NULL ) {
        printf("Symbol sin not found: %s\n", error);
        goto exit_runso;
    }

    printf( "%f\n", func(3.1415926 / 2) );

exit_runso:
    dlclose(handle);
}
```
```
$gcc –o RunSoSimple RunSoSimple.c –ldl
$./RunSoSimple /lib/libm-2.6.1.so
1.000000
```
-ldl 表示使用DL库（Dynamical Loading），它位于/lib/libdl.so.2。
