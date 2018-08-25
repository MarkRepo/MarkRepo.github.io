---
title: 运行库
categories: kernel
tags: library
---

## 入口函数和初始化

程序的入口点是一个程序的初始化和结束部分，往往是运行库的一部分。典型的程序运行步骤大致如下：

+ 操作系统在创建进程后，把控制权交到了程序的入口，这个入口往往是运行库中的某个入口函数。
+ 入口函数对运行库和程序运行环境进行初始化，包括堆、I/O、线程、全局变量构造，等等。
+ 入口函数在完成初始化之后，调用main函数，正式开始执行程序主体部分。
+ main执行完毕以后，返回到入口函数进行清理工作，包括全局变量析构、堆销毁、关闭I/O等，然后进行系统调用结束进程

### GLIBC入口函数

glibc的启动过程对于“静态/动态”，“可执行文件/共享库”差别很大，下面以“静态/可执行文件”链接的情况来讨论。glibc的启动代码位于源码目录`libc/csu`里，程序入口为`_start`（由ld链接器默认链接脚本指定，可通过参数自定义）。`_start`由汇编实现，并且和平台相关，看i386的`_start`实现：

```
libc\sysdeps\i386\elf\Start.S:
_start:
    xorl %ebp, %ebp
    popl %esi
    movl %esp, %ecx

    pushl %esp
    pushl %edx    
    pushl $__libc_csu_fini
    pushl $__libc_csu_init
    pushl %ecx    
    pushl %esi    
    pushl main
    call __libc_start_main 

    hlt
```
这里省略了一些不重要的代码。在调用`_start`前，装载器会把用户的参数和环境变量压入栈中，按照其压栈的方法，实际上栈顶的元素是argc，而接着其下就是argv和环境变量的数组。此时的栈布局如下图所示：
![argc-argv](/assets/images/runtime/argc-argv.png)
虚线箭头是执行pop %esi之前的栈顶（%esp），而实线箭头是执行之后的栈顶（%esp）。  
把`_start`改写为更具可读性的伪代码：

```c
void _start()
{
    %ebp = 0;  // xorl %ebp, %ebp；将ebp清零，目的是表明当前是程序的最外层函数
    int argc = pop from stack  //popl %esi  将argc 存入esi
    char** argv = top of stack;//movl %esp, %ecx  栈顶地址传给%ecx，指向argv及环境变量数组。
    __libc_start_main( main, argc, argv, __libc_csu_init, 
    				   __libc_csu_fini, edx, top of stack );
}
```
argv中隐含的环境变量表要在`__libc_start_main`函数里提取出来。  
实际执行代码的函数是`__libc_start_main`，代码很长，一段一段地看：

```c
_start -> __libc_start_main:

int __libc_start_main (
            int (*main) (int, char **, char **),
            int argc, 
            char * __unbounded *__unbounded ubp_av,  //即argv，包含环境变量表
            __typeof (main) init,   //main调用前的初始化工作。
            void (*fini) (void),    //main结束后的收尾工作。
            void (*rtld_fini) (void), //和动态加载有关的收尾工作，rtld是runtime loader的缩写。
            void * __unbounded stack_end) //栈底地址
{
#if __BOUNDED_POINTERS__
    char **argv;
#else
# define argv ubp_av
#endif
      int result;
...
}
```
>GCC支持bounded类型指针（bounded指针用`__bounded`关键字标出，若默认为bounded指针，则普通指针用`__unbounded`标出），这种指针占用3个指针的空间，在第一个空间里存储原指针的值，第二个空间里存储下限值，第三个空间里存储上限值。`__ptrvalue`、`__ptrlow`、`__ptrhigh`分别返回这3个值，有了3个值以后，内存越界错误便很容易查出来了。 并且要定义`__BOUNDED_POINTERS__`这个宏才有作用，否则这3个宏定义是空的。不过，尽管bounded指针看上去似乎很有用，但是这个功能却在2003年被去掉了。因此现在所有关于bounded指针的关键字其实都是一个空的宏。鉴于此，接下来在讨论libc代码时都默认不使用bounded指针（即不定义`__BOUNDED_POINTERS__`）。

接下来的代码如下：

```c
char** ubp_ev = &ubp_av[argc + 1];
INIT_ARGV_and_ENVIRON;
__libc_stack_end = stack_end;
```
INIT_ARGV_and_ENVIRON这个宏定义于libc/sysdeps/generic/bp-start.h，展开后本段代码变为：

```c
char** ubp_ev = &ubp_av[argc + 1];
__environ = ubp_ev;
__libc_stack_end = stack_end;
```
如下图所示
![environ](/assets/images/runtime/environ.png)
虚线箭头代表`ubp_av`，而实线箭头代表`__environ`。将栈底地址存储在一个全局变量里，以留作它用。  
分两步给`__environ`赋值是因为`INIT_ARGV_and_ENVIRON`根据bounded支持的情况有多个版本，以上假定不支持bounded的版本。  
接下来有另一个宏：

```c
DL_SYSDEP_OSCHECK (__libc_fatal);  //检查操作系统版本
```
接下来的代码颇为繁杂，过滤掉大量信息之后，将一些关键的函数调用列出：

```c
__pthread_initialize_minimal();
__cxa_atexit(rtld_fini, NULL, NULL);   //glibc 内部函数，等同于atexit
__libc_init_first (argc, argv, __environ);
__cxa_atexit(fini, NULL, NULL);
(*init)(argc, argv, __environ);
```
在`__libc_start_main`的末尾，关键的是这两行代码：

```
    result = main (argc, argv, __environ);
    exit (result);
}
```
在最后，main函数终于被调用，并退出。然后看看exit的实现：

```c
_start -> __libc_start_main -> exit:

void exit (int status)
{
    while (__exit_funcs != NULL)
    {
        ...
        __exit_funcs = __exit_funcs->next;  //存储__cxa_atexit和atexit注册的函数的链表
    }
    ...
    _exit (status);
}
```
最后的_exit函数由汇编实现，且与平台相关，下面列出i386的实现：

```
_start -> __libc_start_main -> exit -> _exit:

_exit:
    movl    4(%esp), %ebx
    movl    $__NR_exit, %eax
    int     $0x80
    hlt
```
可见`_exit`仅仅调用了exit系统调用。即`_exit`调用后，进程就会直接结束。程序正常结束有两种情况，一种是main函数的正常返回，一种是程序中用exit退出。在`__libc_start_main`里可以看到，即使main返回了，exit也会被调用。因此exit是进程正常退出的必经之路，把atexit注册的函数的任务交给exit来完成可以说万无一失。  
>在Linux里，进程必须使用exit系统调用结束。一旦exit被调用，程序的运行就会终止，因此实际上`_exit`末尾的hlt不会执行，从而`__libc_start_main`永远不会返回，以至`_start`末尾的hlt指令也不会执行。`_exit`里的hlt指令是为了检测exit系统调用是否成功。如果失败，程序就不会终止，hlt指令就可以发挥作用强行把程序给停下来。而`_start`里的hlt的用处也是如此，是为了预防某种没有调用exit（这里指的不是exit系统调用）就回到`_start`的情况（例如有人误删了`__libc_main_start`末尾的exit）。

### MSVC CRT入口函数

下面是Visual Studio 2003里crt0.c（位于VC安装目录的crt\src）的一部分。删除了一些条件编译的代码，留下了比较重要的部分。MSVC的CRT默认的入口函数名为mainCRTStartup：

```c
int mainCRTStartup(void)
{
    ...
	//堆还没有被初始化，alloca是唯一可以不使用堆的动态分配机制
	//alloca在栈上分配任意大小的空间（只要栈的大小允许），函数返回时自动释放。  
	posvi = (OSVERSIONINFOA *)_alloca(sizeof(OSVERSIONINFOA));
    posvi->dwOSVersionInfoSize = sizeof(OSVERSIONINFOA);
    
    GetVersionExA(posvi);                        //获取当前操作系统版本信息
    _osplatform = posvi->dwPlatformId;
    _winmajor = posvi->dwMajorVersion;			//主版本号，全局变量
    _winminor = posvi->dwMinorVersion;
    _osver = (posvi->dwBuildNumber) & 0x07fff;   //OS 版本，全局变量

    if ( _osplatform != VER_PLATFORM_WIN32_NT )
        _osver |= 0x08000;

    _winver = (_winmajor << 8) + _winminor;      //OS版本，全局变量
	...
}
```
由于没有初始化堆，所以很多事情没法做，当务之急是赶紧把堆先初始化了：

```c
if ( !_heap_init(0) )    // _heap_init 对堆初始化，如果失败，那么程序就直接退出了。
      fast_error_exit(_RT_HEAPINIT);

__try {
        if ( _ioinit() < 0 )   //初始化I/O
            _amsg_exit(_RT_LOWIOINIT);

        _acmdln = (char *)GetCommandLineA();
        _aenvptr = (char *)__crtGetEnvironmentStringsA();

        if ( _setargv() < 0 )	//初始化main函数的argv参数
            _amsg_exit(_RT_SPACEARG);

        if ( _setenvp() < 0 )   //设置环境变量
            _amsg_exit(_RT_SPACEENV);

        initret = _cinit(TRUE);    //其他的C库设置

        if (initret != 0)
                _amsg_exit(initret);
        __initenv = _environ;

        mainret = main(__argc, __argv, _environ);   //调用main函数

        _cexit();
    }
__except ( _XcptFilter(GetExceptionCode(), GetExceptionInformation()) )
    {
        mainret = GetExceptionCode();
        _c_exit();
    } /* end of try - except */
    return mainret;
```
mainCRTStartup的总体流程就是：

1. 初始化和OS版本有关的全局变量。
2. 初始化堆。
3. 初始化I/O。
4. 获取命令行参数和环境变量。
5. 初始化C库的一些数据。
6. 调用main并记录返回值。
7. 检查错误并将main的返回值返回。

在内核中，每一个进程都有一个私有的“打开文件表”，这个表是一个指针数组，每一个元素都指向一个内核的打开文件对象。而fd，就是这个表的下标。当用户打开一个文件时，内核会在内部生成一个打开文件对象，并在这个表里找到一个空项，让这一项指向生成的打开文件对象，并返回这一项的下标作为fd。由于这个表处于内核，并且用户无法访问到，因此用户即使拥有fd，也无法得到打开文件对象的地址，只能够通过系统提供的函数来操作。  
在C语言里，操纵文件的渠道则是FILE结构，不难想象，C语言中的FILE结构必定和fd有一对一的关系，每个FILE结构都会记录自己唯一对应的fd。
FILE、fd、打开文件表和打开文件对象的关系如图所示：
![IO](/assets/images/runtime/IO.png)
Windows中的句柄是打开文件表的下标经过某种线性变换之后的结果。  
I/O初始化的职责就是在用户空间中建立stdin、stdout、stderr及其对应的FILE结构，使得程序进入main之后可以直接使用printf、scanf等函数。

### MSVC CRT的入口函数初始化

MSVC的入口函数初始化主要包含 堆初始化和I/O初始化。堆初始化：

```c
mainCRTStartup -> _heap_init()：//位于heapinit.c（删去了64位系统的条件编译部分）

HANDLE _crtheap = NULL;

int _heap_init (int mtflag)
{
    if ( (_crtheap = HeapCreate( mtflag ? 0 : HEAP_NO_SERIALIZE, 
        BYTES_PER_PAGE, 0 )) == NULL )
        return 0;

    return 1;
}
```
在32位的编译环境下，MSVC的堆初始化过程出奇地简单，它仅仅调用了HeapCreate这个API创建了一个系统堆。因此不难想象，MSVC的malloc函数必然是调用了HeapAlloc这个API，将堆管理的过程直接交给了操作系统。

I/O初始化相对于堆的初始化则要复杂很多。MSVC中FILE结构的定义:

```c
struct _iobuf {
    char *_ptr;
    int   _cnt;
    char *_base;
    int   _flag;
    int   _file;  //最重要的字段，通过它可以访问到内部文件句柄表的某一项。
    int   _charbuf;
    int   _bufsiz;
    char *_tmpfname;
    };
typedef struct _iobuf FILE;
```
在Windows中，用户态使用句柄（Handle）来访问内核文件对象，句柄本身是一个32位的数据类型，在有些场合使用int来储存，有些场合使用指针来表示。
在MSVC的CRT中，已经打开的文件句柄的信息使用数据结构ioinfo来表示：

```c
typedef struct {
    intptr_t osfhnd;  //打开文件的句柄，intptr_t：8字节整数类型
    char osfile;    //文件的打开属性，见下文
    char pipech;    //用于管道的单字符缓冲
}ioinfo;

//osfile 打开文件属性
#define FOPEN(0x01) 	//句柄被打开。
#define FEOFLAG(0x02)	//已到达文件末尾。
#define FCRLF(0x04)		//在文本模式中，行缓冲已遇到回车符（见第11.2.2节）。
#define FPIPE(0x08)		//管道文件。
#define FNOINHERIT(0x10)//句柄打开时具有属性_O_NOINHERIT（不遗传给子进程）。
#define FAPPEND(0x20)	//句柄打开时具有属性O_APPEND（在文件末尾追加数据）。
#define FDEV(0x40)		//设备文件。
#define FTEXT(0x80)		//文件以文本模式打开。
```
在crt/src/ioinit.c中，有一个数组：

```c
int _nhandle;      //实际元素个数
ioinfo * __pioinfo[64]; // 等效于ioinfo __pioinfo[64][32];
```
这就是用户态的打开文件表。它是一个二维数组，第二维的大小为32，总共可以容纳2048个句柄。
FILE结构中的`_file`的值，和此表的两个下标直接相关联。访问文件时，CRT使用`_osfhnd`宏从FILE结构转换到操作系统的句柄。

```c
#define _osfhnd(i)  ( _pioinfo(i)->osfhnd )
#define _pioinfo(i) ( __pioinfo[(i) >> 5] + ((i) & ((1 << 5) -  1)) )
```
FILE：`_file`的第5位到第10位是第一维坐标（共6位），`_file`的第0位到第4位是第二维坐标（共5位）。  
Windows的FILE、句柄和内核对象的关系如图所示：
![windows-io](/assets/images/runtime/windows-io.png)
MSVC的I/O初始化就是构造用户态打开文件表。首先，`_ioinit`初始化了`__pioinfo`数组的第一个二级数组：

```c
mainCRTStartup -> _ioinit()：  //定义于crt/src/ioinit.c中

if ( (pio = _malloc_crt( 32 * sizeof(ioinfo) ))
             == NULL )
{
    return -1;
}

__pioinfo[0] = pio;
_nhandle = 32;
for ( ; pio < __pioinfo[0] + 32 ; pio++ ) {
    pio->osfile = 0;
    pio->osfhnd = (intptr_t)INVALID_HANDLE_VALUE; //无效值，-1
    pio->pipech = 10;
}
```
接下来，`_ioinit`将初始化预定义的打开文件，这包括两部分：

+ 从父进程继承的打开文件句柄。
+ 操作系统提供的标准输入输出。

应用程序可以使用API GetStartupInfo来获取继承的打开文件，GetStartupInfo的参数如下：  
`void GetStartupInfo(STARTUPINFO* lpStartupInfo);`  
STARTUPINFO是一个结构，调用GetStartupInfo之后，该结构就会被写入各种进程启动相关的数据。在该结构中，有两个保留字段为：

```c
typedef struct _STARTUPINFO {
    ……
    WORD cbReserved2;
    LPBYTE lpReserved2;
    ……
} STARTUPINFO;
```
这两个字段的用途没有正式的文档说明，但实际是用来传递继承的打开文件句柄。当这两个字段的值都不为0时，说明父进程遗传了一些打开文件句柄。操作系统使用这两个字段传递句柄的方法如下所示：

```c
/*
 *lpReserved2字段实际是一个指针，指向一块内存，这块内存的结构如下:
 *字节[0,3]：传递句柄的数量n。
 *字节[4, 3+n]：每一个句柄的属性（各1字节，表明句柄的属性，同ioinfo结构的_osfile字段）。
 *字节[4+n之后]：每一个句柄的值（n个intptr_t类型数据，同ioinfo结构的_osfhnd字段）。
 *_ioinit函数使用如下代码获取各个句柄的数据：
 */
cfi_len = *(__unaligned int *)(StartupInfo.lpReserved2);
posfile = (char *)(StartupInfo.lpReserved2) + sizeof( int );
posfhnd = (__unaligned intptr_t *)(posfile + cfi_len);
```
其中`__unaligned`关键字告诉编译器该指针可能指向一个没有进行数据对齐的地址，编译器会插入一些代码来避免发生数据未对齐而产生的错误。这段代码执行之后，lpReserved2指向的数据结构会被两个指针分别指向其中的两个数组，如图所示。
![lpReserved](/assets/images/runtime/lpReserved.png)
接下来`_ioinit`就将这些数据填入自己的打开文件表中:

```c
cfi_len = __min( cfi_len, 32 * 64 );
//然后要给打开文件表分配足够的空间以容纳所有的句柄：
for ( i = 1 ; _nhandle < cfi_len ; i++ ) {  //__pioinfo[0]已经预先分配，直接从__pioinfo[1]开始分配
    if ( (pio = _malloc_crt( 32 * sizeof(ioinfo) )) == NULL )
    {
        cfi_len = _nhandle;
        break;
    }
    __pioinfo[i] = pio;
    _nhandle += 32;				//总是等于已经分配的元素数量
    for ( ; pio < __pioinfo[i] + 32 ; pio++ ) {
        pio->osfile = 0;
        pio->osfhnd = (intptr_t)INVALID_HANDLE_VALUE;
        pio->pipech = 10;
    }
}
//分配了空间之后，将数据填入
for ( fh = 0 ; fh < cfi_len ; fh++, posfile++, posfhnd++ ) 
{
    if ( (*posfhnd != (intptr_t)INVALID_HANDLE_VALUE) && //过滤不符合条件的句柄
               (*posfile & FOPEN) &&
               ((*posfile & FPIPE) ||
               (GetFileType( (HANDLE)*posfhnd ) != 
            FILE_TYPE_UNKNOWN)) )
    {
        pio = _pioinfo( fh ); //通过_pioinfo宏转换为打开文件表中的对应元素
        pio->osfhnd = *posfhnd; //每一个句柄的数据
        pio->osfile = *posfile;
    }
}
```
接下来初始化标准输入输出。当继承句柄的时候，有可能标准输入输出（fh=0,1,2）已经被继承了，因此在初始化前首先要先检验这一点，代码如下：

```c
for ( fh = 0 ; fh < 3 ; fh++ ) 
{
    pio = __pioinfo[0] + fh;

    if ( pio->osfhnd == (intptr_t)INVALID_HANDLE_VALUE ) //如果句柄是无效的，表明没有继承自父进程
    {
        pio->osfile = (char)(FOPEN | FTEXT);
        if ( ((stdfh = (intptr_t)GetStdHandle( stdhndl(fh) )) //获取默认的标准输入输出句柄
                != (intptr_t)INVALID_HANDLE_VALUE) 
                && ((htype =GetFileType( (HANDLE)stdfh )) //获取该默认句柄的类型
                != FILE_TYPE_UNKNOWN) )
        {
            pio->osfhnd = stdfh;
            if ( (htype & 0xFF) == FILE_TYPE_CHAR )
                pio->osfile |= FDEV;
            else if ( (htype & 0xFF) == FILE_TYPE_PIPE )
                pio->osfile |= FPIPE;
        }
        else {
            pio->osfile |= FDEV;
        }
    }
    else  {
        pio->osfile |= FTEXT;
    }
}
```
在I/O初始化完成之后，所有的I/O函数就可以自由使用了。MSVC的I/O初始化主要工作是：

+ 建立打开文件表。
+ 如果能够继承自父进程，那么从父进程获取继承的句柄。
+ 初始化标准输入输出。

## C/C++运行库

### C语言运行库

任何一个C程序，它的背后都有一套庞大的代码来进行支撑，以使得该程序能够正常运行。这套代码至少包括入口函数，及其所依赖的函数所构成的函数集合。当然，它还理应包括各种标准库函数的实现。这样的一个代码集合称之为运行时库（Runtime Library）。而C语言的运行库，即被称为C运行库（CRT）。   
一个C语言运行库大致包含了如下功能：

+ 启动与退出：包括入口函数及入口函数所依赖的其他函数等。
+ 标准函数：由C语言标准规定的C语言标准库所拥有的函数实现。
+ I/O：I/O功能的封装和实现，参见上一节中I/O初始化部分。
+ 堆：堆的封装和实现，参见上一节中堆初始化部分。
+ 语言实现：语言中一些特殊功能的实现。
+ 调试：实现调试功能的代码。

### C语言标准库

ANSI C标准库由24个头文件组成，仅包含数学函数、字符/字符串处理，I/O等基本方面，例如：

+ 标准输入输出（stdio.h）
+ 文件操作（stdio.h）
+ 字符操作（ctype.h）
+ 字符串操作（string.h）
+ 数学函数（math.h）
+ 资源管理（stdlib.h）
+ 格式转换（stdlib.h）
+ 时间/日期（time.h）
+ 断言（assert.h）
+ 各种类型上的常数（limits.h & float.h）

除此之外，C语言标准库还有一些特殊的库，用于执行一些特殊的操作，例如：

+ 变长参数（stdarg.h）
+ 非局部跳转（setjmp.h）

接下来看看两组特殊函数的细节。

#### 变长参数

变长参数是C语言的特殊参数形式，例如如下函数声明：  
`int printf(const char* format, ...);`  
在函数的实现部分，使用stdarg.h里的多个宏来访问各个额外的参数：

```c
//假设lastarg是变长参数函数的最后一个具名参数，那么在函数内部定义类型为va_list的变量：
va_list ap;
//该变量以后将会依次指向各个可变参数。ap必须用宏va_start初始化一次，其中lastarg必须是函数的最后一个具名的参数。
va_start(ap, lastarg);
//此后，可以使用va_arg宏来获得下一个不定参数（假设已知其类型为type）：
type next = va_arg(ap, type);
//在函数结束前，还必须用宏va_end来清理现场。
va_end(ap);
```
变长参数的实现得益于C语言默认的cdecl调用惯例的自右向左压栈传递方式，参考“内存”一文中的参数布局。同时cdecl调用惯例保证了参数的正确清除。因为有些调用惯例（如stdcall）是由被调用方负责清除堆栈的参数，然而，被调用方在这里其实根本不知道有多少参数被传递进来，所以没有办法清除堆栈。而cdecl恰好是调用方负责清除堆栈，因此没有这个问题。  
分析va_list等宏的实现：

+ `va_list`实际是一个指针，用来指向各个不定参数。由于类型不明，因此这个`va_list`以`void*`或`char*`为最佳选择。
+ `va_start`将`va_list`定义的指针指向函数的最后一个参数后面的位置，这个位置就是第一个不定参数。
+ `va_arg`获取当前不定参数的值，并根据当前不定参数的大小将指针移向下一个参数。
+ `va_end`将指针清0。

按照以上思路，va系列宏的一个最简单的实现就可以得到了，如下所示：

```c
#define va_list char*
#define va_start(ap,arg) (ap=(va_list)&arg+sizeof(arg))
#define va_arg(ap,t) (*(t*)((ap+=sizeof(t))-sizeof(t)))
#define va_end(ap) (ap=(va_list)0)
```

如果需要在定义宏的时候使用变长参数，可以由编译器的变长参数宏实现:

```c
//在GCC编译器下，变长参数宏可以使用“##”宏字符串连接操作实现：
#define printf(args…) fprintf(stdout, ##args)
//在MSVC下，使用__VA_ARGS__这个编译器内置宏：
#define printf(…) fprintf(stdout,__VA_ARGS__)
```

#### 非局部跳转

非局部跳转即使在C语言里也是一个备受争议的机制。使用非局部跳转，可以实现从一个函数体内向另一个事先登记过的函数体内跳转，而不用担心堆栈混乱。下面看一个示例：

```c
#include <setjmp.h>
#include <stdio.h>
jmp_buf b;
void f()
{
    longjmp(b, 1);
}
int main()
{
    if (setjmp(b))
        printf("World!");
    else
    {
        printf("Hello ");
        f();
    }
}
```
这段代码按常理不论setjmp返回什么，也只会打印出“Hello ”和“World！”之一，然而事实上的输出是：  
Hello World!  
实际上，当setjmp正常返回的时候，会返回0，因此会打印出“Hello ”的字样。而longjmp的作用，就是让程序的执行流回到当初setjmp返回的时刻，并且返回由longjmp指定的返回值（longjmp的参数2），也就是1，自然接着会打印出“World！”并退出。换句话说，longjmp可以让程序“时光倒流”回setjmp返回的时刻，并改变其行为，以至于改变了未来。  
是的，这绝对不是结构化编程。

### glibc与MSVC CRT

glibc和MSVCRT事实上是标准C语言运行库的超集，它们各自对C标准库进行了一些扩展。  
glibc除了C标准库之外，还有几个辅助程序运行的运行库，这几个文件可以称得上是真正的“运行库”。它们就是/usr/lib/crt1.o、/usr/lib/crti.o和/usr/lib/crtn.o。 

#### glibc 启动文件

crt1.o里面包含的就是程序的入口函数`_start`，由它负责调用`__libc_start_main`初始化libc并且调用main函数进入真正的程序主体。实际上最初开始的时候它并不叫做crt1.o，而是叫做crt.o，包含了基本的启动、退出代码。由于当时有些链接器对链接时目标文件和库的顺序有依赖性，crt.o这个文件必须被放在链接器命令行中的所有输入文件中的第一个，为了强调这一点，crt.o被更名为crt0.o，表示它是链接时输入的第一个文件。

后来由于C++的出现和ELF文件的改进，出现了必须在main()函数之前执行的全局/静态对象构造和必须在main()函数之后执行的全局/静态对象析构。为了满足类似的需求，运行库在每个目标文件中引入两个与初始化相关的段“.init”和“.finit”。运行库会保证所有位于这两个段中的代码会先于/后于main()函数执行，所以用它们来实现全局构造和析构就是很自然的事情了。链接器在进行链接时，会把所有输入目标文件中的“.init”和“.finit”按照顺序收集起来，然后将它们合并成输出文件中的“.init”和“.finit”。但是这两个输出的段中所包含的指令还需要一些辅助的代码来帮助它们启动（比如计算GOT之类的），于是引入了两个用来帮助实现初始化函数的目标文件crti.o和crtn.o。

与此同时，为了支持新的库和可执行文件格式，crt0.o也进行了升级，变成了crt1.o。crt0.o和crt1.o之间的区别是crt0.o为原始的，不支持“.init”和“.finit”的启动代码，而crt1.o是改进过后，支持“.init”和“.finit”的版本。这一点从反汇编crt1.o可以看到，它向libc启动函数`__libc_start_main()`传递了两个函数指针`“__libc_csu_init”`和`“__libc_csu_fini”`，这两个函数负责调用`_init()`和`_finit()`。

为了方便运行库调用，最终输出文件中的“.init”和“.finit”两个段实际上分别包含的是`_init()`和`_finit()`这两个函数。crti.o和crtn.o这两个目标文件中包含的代码实际上是`_init()`函数和`_finit()`函数的开始和结尾部分，当这两个文件和其他目标文件安装顺序链接起来以后，刚好形成两个完整的函数`_init()`和`_finit()`。用objdump查看这两个文件的反汇编代码：

```
$ objdump -dr /usr/lib/crti.o

crti.o:     file format elf32-i386

Disassembly of section .init:

00000000 <_init>:
   0:   55                      push   %ebp
   1:   89 e5                   mov    %esp,%ebp
   3:   53                      push   %ebx
   4:   83 ec 04                sub    $0x4,%esp
   7:   e8 00 00 00 00          call   c <_init+0xc>
   c:   5b                      pop    %ebx
   d:   81 c3 03 00 00 00       add    $0x3,%ebx
                        f: R_386_GOTPC  _GLOBAL_OFFSET_TABLE_
  13:   8b 93 00 00 00 00       mov 0x0(%ebx),%edx
                        15: R_386_GOT32 __gmon_start__
  19:   85 d2                   test   %edx,%edx
  1b:   74 05                   je     22 <_init+0x22>
  1d:   e8 fc ff ff ff          call   1e <_init+0x1e>
                        1e: R_386_PLT32 __gmon_start__

Disassembly of section .fini:

00000000 <_fini>:
   0:   55                      push   %ebp
   1:   89 e5                   mov    %esp,%ebp
   3:   53                      push   %ebx
   4:   83 ec 04                sub    $0x4,%esp
   7:   e8 00 00 00 00          call   c <_fini+0xc>
   c:   5b                      pop    %ebx
   d:   81 c3 03 00 00 00       add    $0x3,%ebx
                        f: R_386_GOTPC  _GLOBAL_OFFSET_TABLE_

$ objdump -dr /usr/lib/crtn.o

crtn.o:     file format elf32-i386

Disassembly of section .init:
00000000 <.init>:
   0:   58                      pop    %eax
   1:   5b                      pop    %ebx
   2:   c9                      leave
   3:   c3                      ret
Disassembly of section .fini:

00000000 <.fini>:
   0:   59                      pop    %ecx
   1:   5b                      pop    %ebx
   2:   c9                      leave
   3:   c3                      ret
```

于是在最终链接完成之后，输出的目标文件中的“.init”段只包含了一个函数`_init()`，这个函数的开始部分来自于crti.o的“.init”段，结束部分来自于crtn.o的“.init”段。为了保证最终输出文件中“.init”和“.finit”的正确性，必须保证在链接时，crti.o必须在用户目标文件和系统库之前，而crtn.o必须在用户目标文件和系统库之后。链接器的输入文件顺序一般是：  
`ld crt1.o crti.o [user_objects] [system_libraries] crtn.o`  
由于crt1.o（crt0.o）不包含“.init”段和“.finit”段，所以不会影响最终生成“.init”和“.finit”段时的顺序。输出文件中的“.init”段看上去应该下图所示（对于“.finit”来说也一样）。
![init](/assets/images/runtime/init.png)

GCC提供了两个参数“-nostartfile”和“-nostdlib”，分别用来取消默认的启动文件和C语言运行库。  
由于`.init`和`.finit`的特殊性，一些用户监控程序性能、调试等工具经常利用它们进行一些初始化和反初始化的工作。可以使用`“__attribute__((section(“.init”)))”`将函数放到.init段里面，但是要注意的是普通函数放在“.init”是会破坏它们的结构的，因为函数的返回指令使得_init()函数会提前返回，必须使用汇编指令，不能让编译器产生“ret”指令。

#### GCC平台相关目标文件

在链接时碰到过的诸多输入文件中，已经解决了crt1.o、crti.o和crtn.o，剩下的还有几个crtbeginT.o、libgcc.a、libgcc_eh.a、crtend.o。严格来讲，这几个文件实际上不属于glibc，它们是GCC的一部分，它们都位于GCC的安装目录下：  
`/usr/lib/gcc/i486-Linux-gnu/4.1.3/crtbeginT.o`  
`/usr/lib/gcc/i486-Linux-gnu/4.1.3/libgcc.a`  
`/usr/lib/gcc/i486-Linux-gnu/4.1.3/libgcc_eh.a`  
`/usr/lib/gcc/i486-Linux-gnu/4.1.3/crtend.o`   
首先是crtbeginT.o及crtend.o，这两个文件是真正用于实现C++全局构造和析构的目标文件。C++这样的语言的实现是跟编译器密切相关的，而glibc只是一个C语言运行库，它对C++的实现并不了解。而GCC是C++的真正实现者，它对C++的全局构造和析构了如指掌。于是它提供了两个目标文件crtbeginT.o和crtend.o来配合glibc实现C++的全局构造和析构。事实上是crti.o和crtn.o中的“.init”和“.finit”提供一个在main()之前和之后运行代码的机制，而真正全局构造和析构则由crtbeginT.o和crtend.o来实现。(`.init`,`.finit`调用`.crtbeginT.o`、`.crtend.o`中的函数)  
libgcc.a 中包含用于处理不同平台之间差异的例程，动态版为libgcc_s.so。`libgcc_eh.a`包含支持C++的异常处理的平台相关函数。

#### MSVC CRT

同一个版本的MSVC CRT根据 静态/动态链接，单线程/多线程，调试/发布，是否支持C++，是否支持托管代码等属性的组合提供多种子版本。微软提供了一套运行库的命名方法，静态版和动态版完全不同。静态版命名规则为：

+ libc [p] [mt] [d] .lib
+ p  表示 C Plusplus，即C++标准库。
+ mt 表示 Multi-Thread，即表示支持多线程。
+ d  表示 Debug，即表示调试版本。

比如静态的非C++的多线程版CRT的文件名为libcmtd.lib。动态版的CRT的每个版本一般有两个相对应的文件，一个用于链接的.lib文件，一个用于运行时用的.dll动态链接库。它们的命名方式与静态版的CRT非常类似，稍微有所不同的是，CRT的动态链接库DLL文件名中会包含版本号。  
默认情况下，如果在编译链接时不指定链接哪个CRT，编译器会默认选择LIBCMT.LIB，即静态多线程CRT。

## C++全局构造与析构

### glibc全局构造与析构

“.init”和“.finit”段最终会被拼成两个函数`_init()`和`_finit()`，下面探究这两个函数是如何实现全局对象的构造和析构的细节。
为了表述方便，使用下面的代码编译出来的可执行文件进行分析：

```c
class HelloWorld
{
public:
    HelloWorld();
    ~HelloWorld();
};
HelloWorld Hw;
HelloWorld::HelloWorld()
{
    ......
}
HelloWorld::~HelloWorld()
{
    ......
}

int main()
{
    return 0;    
}
```
`_start`传递进来的init实际指向了`__libc_csu_init`函数。这个函数的定义：

```c
_start –> __libc_start_main -> __libc_csu_init: //位于Glibc源码目录csu\Elf-init.c

void __libc_csu_init (int argc, char **argv, char **envp)
{
    …
    _init ();   //即 .init 段

    const size_t size = __init_array_end - __init_array_start;
    for (size_t i = 0; i < size; i++)
          (*__init_array_start [i]) (argc, argv, envp);
}
```
看到这里，似乎线索要断了，因为`“_init”`函数的实际内容并不定义在Glibc里面，它是由各个输入目标文件中的“.init”段拼凑而来的。不过除了分析源代码之外，还有一个终极必杀就是反汇编目标代码，随意反汇编一个可执行文件就可以发现`_init()`函数的内容：

```c
_start –> __libc_start_main -> __libc_csu_init –> _init：

Disassembly of section .init:

80480f4 <_init>:
80480f4:       55                   push    %ebp
80480f5:       89 e5                mov     %esp,%ebp
80480f7:       53                   push    %ebx
80480f8:       83 ec 04             sub     $0x4,%esp
80480fb:       e8 00 00 00 00       call    8048100 <_init+0xc>
8048100:       5b                   pop     %ebx
8048101:       81 c3 9c 39 07 00    add     $0x7399c,%ebx
8048107:       8b 93 fc ff ff ff    mov     -0x4(%ebx),%edx
804810d:       85 d2                test    %edx,%edx
804810f:       74 05                je      8048116 <_init+0x22>
8048111:       e8 ea 7e fb f7       call    0 <_nl_current_LC_CTYPE>
8048116:       e8 95 00 00 00       call    80481b0 <frame_dummy>
804811b:       e8 b0 6e 05 00       call    809efd0 <__do_global_ctors_aux>
8048120:       58                   pop     %eax
8048121:       5b                   pop     %ebx
8048122:       c9                   leave
8048123:       c3                   ret
```
`_init`调用`__do_global_ctors_aux`函数，它来自于GCC提供的目标文件crtbegin.o，位于gcc/Crtstuff.c，简化以后代码如下：

```c
_start –> __libc_start_main -> __libc_csu_init –> _init -> __do_global_ctors_aux：

void __do_global_ctors_aux(void)
{
    /* Call constructor functions.  */
    unsigned long nptrs = (unsigned long) __CTOR_LIST__[0];  //第一个元素是数组元素的个数
    unsigned i;

    for (i = nptrs; i >= 1; i--)
         __CTOR_LIST__[i] (); //之后的元素是函数指针
}
```
`__CTOR_LIST__`里面存放所有全局对象的构造函数的指针。  
对于每个编译单元(.cpp)，GCC编译器会遍历其中所有的全局对象，生成一个特殊的函数，这个特殊函数的作用就是对本编译单元里的所有全局对象进行初始化。通过对本节开头的代码进行反汇编，可以看到GCC在目标代码中生成了一个名为`_GLOBAL__I_Hw`的函数，由这个函数负责本编译单元所有的全局\静态对象的构造和析构，它的代码可以表示为：

```c
static void GLOBAL__I_Hw(void)
{
    Hw::Hw(); // 构造对象
    atexit(__tcf_1); // 一个神秘的函数叫做__tcf_1被注册到了exit
}
```
如果一个目标文件里有`GLOBAL__I_Hw`函数，编译器会在这个编译单元产生的目标文件(.o)的“.ctors”段里放置一个指针指向它，
链接器在连接这些目标文件时，会将同名的段合并在一起，这样，每个目标文件的“.ctors”段将会被合并为一个“.ctors”段，其中的内容是各个目标文件的“.ctors”段的内存拼接而成。由于每个目标文件的.ctors段都只存储了一个指针（指向该目标文件的全局构造函数），因此拼接起来的.ctors段就成为了一个函数指针数组，每一个元素都指向一个目标文件的全局构造函数。  
crtbegin.o：作为所有“.ctors”段的开头部分，crtbegin.o的“.ctor”段里面存储的是一个4字节的-1(0xFFFFFFFF)，由链接器负责将这个数字改成全局构造函数的数量。然后这个段还将起始地址定义成符号`__CTOR_LIST__`，这样实际上`__CTOR_LIST__`所代表的就是所有.ctor段最终合并后的起始地址了。  
crtend.o：这个文件里面的.ctors内容就更简单了，它的内容就是一个0，然后定义了一个符号`__CTOR_END__`，指向.ctor段的末尾。
链接器在链接用户的目标文件的时候，crtbegin.o总是处在用户目标文件的前面，而crtend.o则总是处在用户目标文件的后面。在合并crtbegin.o、用户目标文件和crtend.o时，链接器按顺序拼接这些文件的.ctors段，因此最终形成.ctors段的过程将如图所示。
![ctor](/assets/images/runtime/ctor.png)
在了解了可执行文件的“.ctors”段的结构之后，再回过头来看`__do_global_ctor_aux`的代码就很容易了。`__do_global_ctor_aux`从`__CTOR_LIST__`的下一个位置开始，按顺序执行函数指针，直到遇上NULL（`__CTOR_END__`）。如此每个目标文件的全局构造函数都能被调用。

【小实验】  
glibc的全局构造函数是放置在.ctors段里的，因此如果手动在.ctors段里添加一些函数指针，就可以让这些函数在全局构造的时候（main之前）调用：

```c
#include <stdio.h>
void my_init(void) 
{
       printf("Hello ");
}

typedef void (*ctor_t)(void); 
//在.ctors段里添加一个函数指针
ctor_t __attribute__((section (".ctors"))) my_init_p = &my_init; 

int main() 
{
       printf("World!\n");
       return 0;
}
```
事实上，gcc里有更加直接的办法来达到相同的目的，那就是使用`__attribute__((constructor))`:

```c
#include <stdio.h>
void my_init(void) __attribute__ ((constructor));
void my_init(void) 
{
       printf("Hello ");
}
int main() 
{
       printf("World!\n");
       return 0;
}
```

对于早期的glibc和GCC，在完成了对象的构造之后，在程序结束之前，crt还要进行对象的析构。实际上正常的全局对象析构与前面介绍的构造在过程上是完全类似的，而且所有的函数、符号名都一一对应，比如“.init”变成了“.finit”、`“__do_global_ctor_aux”`变成了`“__do_global_dtor_aux”`、`“__CTOR_LIST__”`变成了`“__DTOR_LIST__”`等。在前面介绍入口函数时可以看到，`__libc_start_main`将`“__libc_csu_fini”`通过`__cxa_exit()`注册到退出列表中，这样当进程退出前exit()里面就会调用`“__libc_csu_fini”`。`“_fini”`的原理和`“_init”`基本是一样的，在这里不再一一赘述了。  
为了保证全局对象构造和析构的顺序（即先构造后析构），链接器必须包装所有的“.dtor”段的合并顺序必须是“.ctor”的严格反序，这增加了链接器的工作量，于是后来人们放弃了这种做法，采用了一种新的做法，就是通过`__cxa_atexit()`在exit()函数中注册进程退出回调函数来实现析构。

这就要回到之前在每个编译单元的全局构造函数`GLOBAL__I_Hw()`中看到的神秘函数。编译器对每个编译单元的全局对象，都会生成一个特殊的函数来调用这个编译单元的所有全局对象的析构函数，它的调用顺序与`GLOBAL__I_Hw()`调用构造函数的顺序刚好相反。例如对于前面的例子中的代码，编译器生成的所谓的神秘函数内容大致是：

```c
static void __tcf_1(void) //这个名字由编译器生成
{
    Hw.~HelloWorld();
}
```
此函数负责析构Hw对象，由于在`GLOBAL__I_Hw`中我们通过`__cxa_exit()`注册了`__tcf_1`，而且通过`__cxa_exit()`注册的函数在进程退出时被调用的顺序满足先注册后调用的属性，与构造和析构的顺序完全符合，于是它就很自然被用于析构函数的实现了。  
由于全局对象的构建和析构都是由运行库完成的，于是在程序或共享库中有全局对象时，记得不能使用“-nonstartfiles”或“-nostdlib”选项，否则，构建与析构函数将不能正常执行（除非你很清楚自己的行为，并且手工构造和析构全局对象）。

>Collect2  
collect2是ld的一个包装，它最终还是调用ld完成所有的链接工作，那么collect2这个程序的作用是什么呢？  
在有些系统上，汇编器和链接器并不支持本节中所介绍的“.init”“.ctor”这种机制，于是为了实现在main函数前执行代码，必须在链接时进行特殊的处理。Collect2这个程序就是用来实现这个功能的，它会“收集”（collect）所有输入目标文件中那些命名特殊的符号，这些特殊的符号表明它们是全局构造函数或在main前执行，collect2会生成一个临时的.c文件，将这些符号的地址收集成一个数组，然后放到这个.c文件里面，编译后与其他目标文件一起被链接到最终的输出文件中。  
在这些平台上，GCC编译器也会在main函数的开始部分产生一个`__main`函数的调用，这个函数实际上就是负责collect2收集来的那些函数。`__main`函数也是GCC所提供的目标文件的一部分，如果我们使用“-nostdlib”编译程序，可能得到`__main`函数未定义的错误，这时候只要加上“-lgcc”把它链接上即可。

### MSVC CRT的全局构造和析构

看看MSVC的入口函数mainCRTStartup里是否有全局构造的相关内容：

```c
mainCRTStartup:

mainCRTStartup() 
{
    …
    _initterm( __xc_a, __xc_z );
    …
}
```
其中`__xc_a`和`__xc_z`是两个函数指针，而initterm的内容则是：

```c
mainCRTStartup -> _initterm:

// file: crt\src\crt0dat.c
static void __cdecl _initterm (_PVFV * pfbegin,_PVFV * pfend)
{
        while ( pfbegin < pfend )
        {
            if ( *pfbegin != NULL )
                (**pfbegin)();
            ++pfbegin;
        }
}
```
其中`_PVFV`定义是：  `typedef void (__cdecl *_PVFV)();`，所以`__xc_a`和`__xc_z`都是函数指针的指针。对照Glibc/GCC的实现，`_initterm`与`__do_global_ctors_aux`一模一样，它依次遍历所有的函数指针并且调用它们, `__xc_a`就是这个指针数组的开始地址，相当于`__CTOR_LIST__`；而`__xc_z`则是结束地址，相当于`__CTOR_END__`。  
`__xc_a`和`__xc_z`不是mainCRTStartup的参数或局部变量，而是两个全局变量，它们的值在mainCRTStartup调用之前就已经正确地设置好了。mainCRTStartup作为入口函数是真正第一个执行的函数，那么MSVC是如何在此之前就将这两个指针正确设置的呢？看看`__xc_a`和`__xc_z`的定义：

```c
// file: crt\src\cinitexe.c
_CRTALLOC(".CRT$XCA") _PVFV __xc_a[] = { NULL };
_CRTALLOC(".CRT$XCZ") _PVFV __xc_z[] = { NULL };
//其中宏_CRTALLOC 定义于crt\src\sect_attribs.h：
……
#pragma section(".CRT$XCA",long,read)
#pragma section(".CRT$XCZ",long,read)
……
#define _CRTALLOC(x) __declspec(allocate(x))
```
形如`#pragma section`的指令语法如下：  
`#pragma section( "section-name" [, attributes] )`  
作用是在生成的obj文件里创建名为section-name的段，并具有attributes属性。因此这两条pragma指令实际在obj文件里生成了名为`.CRT$XCA`和`.CRT$XCZ`的两个段。下面再来看看`_CRTALLOC`这个宏，该宏的定义为`__declspec(allocate(x))`，这个指示字表明其后的变量将被分配在段x里。所以`__xc_a`被分配在段`.CRT$XCA`里，而`__xc_z`被分配在段`.CRT$XCZ`里。  
当编译的时候，每一个编译单元都会生成名为`.CRT$XCU`（U是User的意思）的段，在这个段中编译单元会加入自身的全局初始化函数。当链接的时候，链接器会将所有相同属性的段合并，值得注意的是：在这个合并过程中，所有输入的段在被合并到输出段时，是据字母表顺序依次排列。于是在本例中，各个段链接之后的状态可能如图所示。
![crtxtu](/assets/images/runtime/crtxtu.png)
由于`.CRT$XT*`这些段的属性都是只读的，且它们的名字很相近，所以它们会被按顺序合并到一起，最后往往被放到只读段中，成为`.rdata`段的一部分。这样就自然地形成了存储所有全局初始化函数的数组，以供`_initterm`函数遍历。MSVC CRT的全局构造实现在机制上与Glibc基本是一样的，只不过它们的名字略有不同，MSVC CRT采用这种段合并的模式与`.ctor`的合并及`__CTOR_LIST__`和`__CTOR_END__`的地址确定何其相似！这再一次证明了虽然各个操作系统、运行库、编译器在细节上大相径庭，但是在基本实现的机制上其实是完全相通的。

【小实验】  
自己添加初始化函数：

```c
#include <iostream>

#define SECNAME ".CRT$XCG"
#pragma section(SECNAME,long,read)
void foo()
{
  std::cout << “hello” << std::endl;
}
typedef void (__cdecl *_PVFV)();
__declspec(allocate(SECNAME)) _PVFV dummy[] = { foo };

int main()
{
  return 0;
}
```
运行这个程序，可以得到如“hello”的输出。为了验证A～Z的这个字母表排列，读者可以修改SECNAME，使之不处于.CRT$XCA和.CRT$XCZ之间，理论上不会得到任何输出。而如果将段名改为.CRT$XCV（V的字典序在U之后），那么foo函数将在main执行之后执行。

最后来看看MSVC的全局析构的实现，在MSVC里，只需要在全局变量的定义位置上设置一个断点，就可以看到在`.CRT$XC?`中定义的全局初始化函数的内容。仍然使用开头的HelloWorld来作为示例：

```c
#include <iostream>
class HelloWorld
{
public:
    HelloWorld() {std::cout << "hi\n";}
    ~HelloWorld(){std::cout << "bye\n";}
};
HelloWorld Hw;
int main()
{
    return 0;    
}
```
这里在`HelloWorld Hw`的位置上设置断点。运行程序并中断之后查看反汇编可以得到初始化函数的内容：

```
011B1B70  mov   eax,dword ptr [__imp_std::cout (11B2054h)] 
011B1B75  push  offset string "hi\n" (11B2124h) 
011B1B7A  push  eax  
011B1B7B  call  std::operator<<<std::char_traits<char> > (11B1140h) 
011B1B80  push  offset `dynamic atexit destructor for 'Hw'' (11B1B90h) 
011B1B85  call  atexit (11B13B0h) 
011B1B8A  add   esp,0Ch 
011B1B8D  ret 
```
在这里可以看见这段程序首先调用了内联之后的HelloWorld的构造函数，然后和g++相同，调用atexit将一个名为dynamic atexit destructor for 'Hw''的函数注册给程序退出时调用。而这个dynamic atexit destructor for 'Hw''函数的定义也能很容易找到：

```
`dynamic atexit destructor for 'Hw'`:
011B1B90  mov   eax,dword ptr [__imp_std::cout (11B2054h)] 
011B1B95  push  offset string "bye\n" (11B2128h) 
011B1B9A  push  eax  
011B1B9B  call  std::operator<<<std::char_traits<char> > (11B1140h) 
011B1BA0  add   esp,8 
011B1BA3  ret   
```
可以看出，这个函数的作用就是在对象Hw调用内联之后进行析构。Glibc下通过`__cxa_exit()`向exit()函数注册全局析构函数；MSVC CRT也通过atexit()实现全局析构，它们除了函数命名不同之外几乎没有区别。
