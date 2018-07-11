---
title: win32 多线程程序设计笔记
description: 记录windows多线程程序设计的知识点
categories: windows
tags:
  - windows
  - multithread
---

## 为什么要千头万绪

1. 合作型多任务与抢占型多任务的区别
2. 进程、线程的区别
	+ `进程 = 内存 + 资源`
	+ 内存划分：
		+ code：程序的可执行部分。
		+ data：程序中的所有变量（不包含局部变量），分为全局变量、静态变量。
		+ stack：堆栈空间，其中有局部变量。
	+ 资源划分：
		+ 核心对象
		+ USER资源
		+ GDI资源
	+ 线程：任何时刻下的状态，被定义在进程的某块内存中,以及cpu寄存器上，其他重要数据存储在进程的共享内存中。
3. `Context switch`  
发生中断的时候，CPU 取得目前这个线程的当前状态，也就是把所有寄存器内容拷贝到堆栈之中，再把它从堆栈拷贝到一个CONTEXT结构;要切换不同的线程，操作系统应先切换该线程所隶属之进程的内存,然后恢复该线程放在CONTEXT 结构中的寄存器值。
4. `Race conditions` : 链表插入实例。
5. `Atomic operation`

## 线程的第一次接触

+ `CreateThread():`

```c
HANDLE CreateThread(
LPSECURITY_ATTRIBUTES lpThreadAttributes,          //描述新线程security属性，NULL表示使用缺省值。
DWORD dwStackSize,                                             //新线程堆栈大小，0表示使用缺省大小：1MB
LPTHREAD_START_ROUTINE lpStartAddress,           // 新线程开始的起始地址，这是一个函数指针
LPVOID lpParameter,                                             //新线程函数参数
DWORD dwCreationFlags,                                      //允许产生一个暂时挂起的线程，默认立即执行。
LPDWORD lpThreadId                                           //新线程ID被传回这里
);
```
+ 函数调用约定  
`__stdcall, __cdecl, __thiscall, __fastcall, __nakedcall, __pascal`
+ 核心对象  
进程，线程，文件，事件，信号量，互斥器，管道
+ CloseHandle()：

```c
BOOL CloseHandle (
HANDLE hObject                                      //代表一个已打开对象handle
);
```
+ 线程对象与线程的区别  
线程的handle是指向“线程核心对象”，而不是指向线程本身；“线程核心对象”引用到的那个线程也会令核心对象开启。因此，线程对象的默认引用计数是2。

+ GetExitCodeThread():

```c
BOOL GetExitCodeThread(
HANDLE hThread,                                     //线程handle
LPDWORD lpExitCode                               //指向一个DWORD,用以接收结束代码
);
```
GetExitCodeThread( )的一个糟糕行为是，当线程还在进行，尚未有所谓结束代码时，它会传回TRUE 表示成功。也就是说你不可能从其返回值中知道“到底是线程还在运行呢，还是它已结束，但返回值为STILL_ACTIVE。
+ ExitThread():

```c
VOID ExitThread(
DWORD dwExitCode                                //dwExitCode 指定此线程之结束代码
);
```

## 快跑与等待

+ 绝对不要在win32中使用`busy loop`。
+ 等待一个线程的结束：

```c
DWORD WaitForSingleObject(
HANDLE hHandle,                   //等待对象的handle
DWORD dwMilliseconds            //等待时间，时间终了，即使handle尚未激发，此函数还是要返回。0立刻返回，INFINITE无穷等待
);
```
函数失败：返回`WAIT_FAILED`  
函数成功：  
	
1. 等待的目标核心对象变成激发状态，返回值为`WAIT_OBJECT_0`  
2. 核心对象激发之前，等待时间终了，返回值为`WAIT_TIMEOUT`  
3. 如果一个拥有mutex的线程结束前没有释放mutex，则传回`WAIT_ABANDONED`

+ 核心对象激发的意义
	+ Thread ：线程运行时处于未激发状态，线程结束对象被激发。对象由CreateThread或CreateRemoteThread产生。
	+ Process：进程运行时处于未激发状态，进程结束对象被激发。对象由CreateProcess或OpenProcess产生。
	+ Change Notification：当一个特定的磁盘子目录发生变化时，此对象即被激发。此对象由FindFirstChangeNotification产生。
	+ Console Input：当console窗口输入缓冲区中有数据可用时，此对象处于激发状态。CreateFile、GetStdFile产生console handle。
	+ Event：对象的状态受控于3个win32函数：SetEvent，PulseEvent，ResetEvent。CreateEvent、OpenEvent可以传回一个Eventobject handle。Event对象的状态也可以被操作系统设定---如果使用与overlapped操作时。
	+ Mutex：如果Mutex没有被任何线程拥有，就是处于激发状态。一旦一个等待mutex的函数返回了，mutex自动重置为未激发状态。CreateMutex、OpenMutex可以获得一个mutex handle。
	+ Semaphore：有个计数器，约束拥有者个数。计数器大于0，semaphore处于激发状态；等于0，处于未激发状态。CreateSemaphore,OpenSemaphore可以传回一个semaphore handle。
补充说明（3）：这些变化指的是：　　

```c
FILE_NOTIFY_CHANGE_FILE_NAME      //产生、删除、重新命名一个文件
FILE_NOTIFY_CHANGE_DIR_NAME       //产生或删除一个子目录
FILE_NOTIFY_CHANGE_ATTRIBUTES     //目录及子目录中的任何属性改变
FILE_NOTIFY_CHANGE_SIZE           //目录及子目录中的任何文件大小的改变
FILE_NOTIFY_CHANGE_LAST_WRITE     //目录及子目录中的任何文件的最后写入时间的改变
FILE_NOTIFY_CHANGE_SECURITY       //目录及子目录中的任何安全属性改变
```
+ 等待多个对象

```c
DWORD WaitForMultipleObjects(
DWORD nCount,                                     //表示lpHandles所指之handles数组的元素个数。最大容量是MAXIMUM_WAIT_OBJECTS
CONST HANDLE *lpHandles,                    //指向一个由对象handles所组成的数据，这些handles不需要为相同的类型。
BOOL bWaitAll,　　　　　　　　　　　　　  //True表示所有的handles都必须激发，函数才返回。否则此函数将在任何一个handle激发返回。
DWORD dwMilliseconds　　　　　　　　　  //等待时间
);
```
返回值：  
（1）如果因时间终了返回，返回值为`WAIT_TIMEOUT`  
（2）如果bWaitAll为True，返回值为`WAIT_OBJECT_0`  
（3）如果bWaitAll为False，则将返回值减去`WAIT_OBJECT_0`，就表示数组中的哪个handle被激发了  
（4）如果等待的对象中有任何的Mutex，返回值可能从`WAIT_ABANDONED_0` 到 `WAIT_ABANDONED_0 + nCount - 1`  
（5）如果函数失败，传回`WAIT_FAILED`

+ MsgWaitForMultipleObjects:  “对象被激发” 或 “消息到达队列” 时被唤醒而返回

```c
DWORD MsgWaitForMultipleObjects(
DWORD nCount,
LPHANDLE pHandles,
BOOL fWaitAll,
DWORD dwMilliseconds,
DWORD dwWakeMask               //欲观察的用户输入消息
);
```
输入消息：

```c
QS_ALLINPUT
QS_HOTKEY
QS_INPUT
QS_KEY
QS_MOUSE
QS_MOUSEBUTTON
QS_MOUSEMOVE

QS_PAINT
QS_POSTMESSAGE
QS_SENDMESSAGE
QS_TIMER
```

## 同步控制

### 理解同步与异步的概念

### Critical Sections：  

+ critical sections：是指“用来处理一份被共享之资源”的程序代码。如内存、数据结构、文件等。
+ critical section并不是核心对象，存在于进程的内存中。
+ `VOID InitializeCriticalSection(LPCRITICAL_SECTION lpCriticalSection)`;初始化`CRITICAL_SECTION`类型变量。
+ `VOID DeleteCriticalSection(LPCRITICAL_SECTION lpCriticalSection)`;清除`CRITICAL_SECTION`类型变量。
+ VOID EnterCriticalSection(LPCRITICAL_SECTION lpCriticalSection);线程进入，锁定lpCriticalSection。
+ VOID LeaveCriticalSection(LPCRITICAL_SECTION lpCriticalSection);线程离开，解除锁定。
+ 一旦线程进入一个 critical section，它就能够一再地重复进入该 critical section。唯一的警告就是，每一个“进入”操作都必须有一个对应的“离开”操作。
+ 不要长时间锁住一份资源：千万不要在一个 critical section 之中调用 Sleep() 或任何 Wait...() API 函数。
+ 避免Danling Critical Sections:Windows NT 和 Windows 95 在管理dangling critical sections 时有极大的不同。在 Windows NT 之中，如果一个线程进入某个 critical section 而在未离开的情况下就结束，该 critical section 会被永远锁住。然而在 Windows 95 中，如果发生同样的事情，其他等着要进入该 critical section 的线程，将获准进入。这基本上是一个严重的问题，因为你竟然可以在你的程序处于不稳定状态时进入该 critical section。

### 死锁

+ 任何时候当一段代码需要两个（或更多）资源时，都有潜在性的死锁阴影。
+ 强制将资源锁定，使它们成为“all-or-nothing”，可以阻止死锁的发生。
+ 哲学家进餐问题：WaitForMutipleObjects,只能用于核心对象，引出了Mutex。

### 互斥器（Mutexes）  

+ 锁住Mutex比critical section 慢，因为要切换到OS核心态。  
+ Mutexes 可跨进程使用，Critical section 只能在同一个进程中使用。  
+ 等待一个 mutex 时，可以指定“结束等待”的时间长度。但对于critical section 则不行。  
+ mutex、critical section相关函数比较：  

```c
InitializeCriticalSection()                   CreateMutex()
                                              OpenMutex()
EnterCriticalSection()                        WaitForSingleObject()
                                              WaitForMultipleObjects()
                                              MsgWaitForMultipleObjects()
LeaveCriticalSection()                        ReleaseMutex()
DeleteCriticalSection()                       CloseHandle()
```
+ 创建mutex时指定名称，使其能跨进程使用。mutex名称是全局性的。  
+  

```c
HANDLE CreateMutex(

LPSECURITY_ATTRIBUTES lpMutexAttributes,   //安全属性，NULL表示使用默认的属性
BOOL bInitialOwner,                                       //TRUE表示调用这个函数的线程拥有该mutex
LPCTSTR lpName                                           //mutex名称，不能包含
);
```
+ “mutex 激发状态”：当没任何线程拥有该 mutex 而且有一个线程正以 Wait...() 等待该 mutex，该mutex 就会短暂地出现激发状态，使 Wait...() 得以返回。  
+ Bool  ReleaseMutex（Handle hMutex） 释放mutex
+ Mutex 的拥有权并非属那个产生它的线程，而是那个最后对此 mutex 进行 Wait...() 操作并且尚未进行 ReleaseMutex() 操作的线程。
+ 处理被舍弃的互斥器：如果线程拥有一个 mutex 而在结束前没有调用 ReleaseMutex()，mutex 不会被摧毁。取而代之的是，该 mutex会被视为“未被拥有”以及“未被激发”，而下一个等待中的线程会被以`WAIT_ABANDONED_0` 通知。如果其他线程正以 WaitForMultipleObjects() 等待此 mutex，该函数也会返回，传回值介于 `WAIT_ABANDONED_0` 和 (`WAIT_ABANDONED_0_n +1`)之间，其中的 n 是指handle 数组的元素个数。线程可以根据这个值了解到究竟哪一个 mutex 被放弃了。至于WaitForSingleObject() ， 则只是传回`WAIT_ABANDONED_0`。
+ CreateMutex() 的第二个参数 bInitialOwner，允许你指定现行线程（current thread）是否立刻拥有即将产生出来的mutex。乍见之下这个参数或许只是提供一种方便性，但事实上它阻止了一种 race condition 的发生，如果没有bInitialOwner，你就必须写下这样的代码：

```c
HANDLE hMutex = CreateMutex(NULL, FALSE, "Sample Name");
int result = WaitForSingleObject(hMutex, INFINITE);
```
但是这样的安排可能会产生 `race condition`。  

+ 一个线程拥有了某个mutex，后面的wait…（）不会被阻塞。

### 信号量
	 
```c
HANDLE CreateSemaphore(
LPSECURITY_ATTRIBUTES lpAttributes,  //安全属性，默认NULL
LONG lInitialCount,                               //semaphore的初值，必须大于或等于0，小于等于IMaximumCount
LONG lMaximumCount,                         //semaphore最大值，同一时间能够锁住semaphore之线程的最大数。
LPCTSTR lpName                                 //semaphore的名称，NULL表示没有名字
);
```
Semaphore没有拥有权的概念，一个线程可以反复调用 Wait...() 函数以产生新的锁定。
这和 mutex绝不相同：拥有 mutex 的线程不论再调用多少次 Wait...() 函数，也不会被阻塞住。

```c
BOOL ReleaseSemaphore(
HANDLE hSemaphore,             //semaphore的handle
LONG lReleaseCount,               //semaphore现值的增额，不可以是负值或0
LPLONG lpPreviousCount          //借此传回semaphore原来的现值
);
```
注意：lpPreviousCount 所传回来的是一个瞬间值。你不可以把lReleaseCount 加上 *lpPreviousCount，就当作是 semaphore 的现值，因为其他线程可能已经改变了 semaphore 的值。  
设定semaphore初值的意义与CreateMutex() 的 bInitialOwner 参数的存在理由是一样的

### 事件

+ Event的唯一目的就是成为激发或未激发状态，这两种状态全由程序控制。

```c
HANDLE CreateEvent(
LPSECURITY_ATTRIBUTES lpEventAttributes,    //安全属性，NULL默认
BOOL bManualReset,//FALSE表示event在变成激发状态之后，自动重置为非激发状态，TRUE表示需要调用ResetEvent才能重置
BOOL bInitialState,//TRUE表示一开始处于激发状态，FALSE处于非激发状态
LPCTSTR lpName //event名称
);
```
+ 三个操作函数：  
	+ SetEvent：把event对象设为激发状态
	+ ResetEvent：把event对象设为非激发状态
	+ PulseEvent：对应Manual Reset Event：把event设为激发状态，唤醒所有等待线程，然后event恢复为非激发状态。对应`Auto Reset Event`： 把event设为激发状态，唤醒一个等待线程，然后event恢复为非激发状态
+ 如果你面对一个 AutoReset event 对象调用 SetEvent() 或 PulseEvent()，而彼时并没有任何线程正在等待。这种情况下这个 event 会被遗失.

### Interlocked Variables

1. InterlockedIncrement，加1后与0比较，对应于 AddRef（）操作
2. InterlockedDecrement，减1后与0比较，对应于Release（）操作
3. InterlockedExchange设定一个新值，传回旧值，多线程安全。

## 不要让线程成为脱缰野马

### 干净的终止一个线程

+ BOOL TerminateThread(HANDLE hThread, DWORD dwExitCode)//dwExitCode：线程结束代码。  
线程在结束前没有机会清理自己,且堆栈不会释放,产生内存泄露。相关的DLLs没有机会获得”线程解除附着”的通知。这是一个危险的函数,非不得已的情况不用。
+ 使用信号（Signals），行不通。
+ 跨越线程，丢出异常。win32API中没有什么标准方法可以把一个异常情况丢到另一个线程中。
+ 设立一个标记：使用一个手动重置（manual-reset）的 event 对象。Worker 线程可以检查该event 的状态或是等待它。

### 线程优先权

+ 以进程的“优先权类别（priority class）”、线程的“优先权层级 （priority level）”和操作系统当时采用的“动态提升（Dynamic Boost）”作为计算基准。拥有最高优先权之线程，即为下一个将执行起来的线程.如果优先权相同,则轮流调度。
+ 4个优先权类别  
	+ `HIGH_PRIORITY_CLASS 13`
	+ `IDLE_PRIORITY_CLASS 4`
	+ `NORMAL_PRIORITY_CLASS  7 or 8`
	+ `REALTIME_PRIORITY_CLASS 24`
+ 7个优先权层级
	+ `THREAD_PRIORITY_HIGHEST +2`
	+ `THREAD_PRIORITY_ABOVE_NORMAL +1`
	+ `THREAD_PRIORITY_NORMAL 0`
	+ `THERAD_PRIORITY_BELOW_NORMAL -1`
	+ `THREAD_PRIORITY_LOWEST -2`
	+ `THREAD_PRIORITY_IDLE    set to 1`
	+ `THREAD_PRIORITY_TIME_CRITICAL  set to 15`
+ 两个相关函数：SetPriorityClass,GetPriorityClass
	+ BOOL SetThreadPriority(HANDLE hThread, int nPriority);
	+ int GetThreadPriority(HANDLE hThread);
+ 动态提升

### 初始化一个线程

1. 初始化理由：调整优先权；绑核等。
2. CreateThread（）第五个参数设为CREATE_SUSPENDED
3. DWORD ResumeThread(HANDLE hThread)//执行一个线程
4. DWORD SuspendThread(HANDLE hThread)//挂起一个 线程

## Overlapped I/O，在你身后变戏法

### Overlapped I/O

overlapped I/O 是 Win32 的一项技术，你可以要求操作系统为你传送数据，并且在传送完毕时通知你。这项技术使你的程序在I/O 进行过程中仍然能够继续处理事务。事实上，操作系统内部正是以线程来完成 overlapped I/O。

### Win32文件操作函数

+ CreateFile（）可以用来打开文件、串行口和并行口、Named pipes、Console。

```c
HANDLE CreateFile(
LPCTSTR lpFileName,                                         // 指向文件名称
DWORD dwDesiredAccess,                                  // 存取模式（读或写）
DWORD dwShareMode,                                      // 共享模式（share mode）
LPSECURITY_ATTRIBUTES lpSecurityAttributes,  // 指向安全属性结构
DWORD dwCreationDisposition,                          // 如何产生
DWORD dwFlagsAndAttributes,                          // 文件属性
HANDLE hTemplateFile                                      // 一个临时文件，将拥有全部的属性拷贝
);
```
>注：overlapped I/O性质：可以在同一时间读写文件的许多部分；多个overlapped请求，执行次序无法保证；overlapped I/O的基本型式是以ReadFile（）和WriteFile（）完成的。

+ ReadFile（）

```c
BOOL ReadFile(
HANDLE hFile,                                     // 欲读之文件
LPVOID lpBuffer,                                 // 接收数据之缓冲区
DWORD nNumberOfBytesToRead,         // 欲读取的字节个数
LPDWORD lpNumberOfBytesRead,        // 实际读取的字节个数的地址
LPOVERLAPPED lpOverlapped               // 指针，指向 overlapped info
);
```
+ WriteFile（）

```c
BOOL WriteFile(
HANDLE hFile,                                   // 欲写之文件
LPCVOID lpBuffer,                             // 储存数据之缓冲区
DWORD nNumberOfBytesToWrite,       // 欲写入的字节个数
LPDWORD lpNumberOfBytesWritten,   // 实际写入的字节个数的地址
LPOVERLAPPED lpOverlapped             // 指针，指向 overlapped info
);
```
如果CreateFile() 的第6个参数被指定为`FILE_FLAG_ OVERLAPPED`，就必须在上述的 lpOverlapped 参数中提供一个指针，指向一个 OVERLAPPED 结构。

+ OVERLAPPED结构

```c
typedef struct _OVERLAPPED {
DWORD Internal;        //通常保留。当GetOverlappedResult传回False，并且GetLastError并非传回ERROR_IO_PENDING,则内含一个视系统而定的状态。
DWORD InternalHigh; //通常被保留，当GetOverlappedResult传回True，则内含“被传输数据的长度”
DWORD Offset;          //读、写偏移位置，从文件头开始算起。若目标设备不支持文件位置，忽略
DWORD OffsetHigh;   //64位文件偏移中较高32位。若目标设备不支持文件位置，忽略
HANDLE hEvent;       //manual reset event，overlapped I/O完成时被激发。ReadFileEX,WriteFileEX忽略这个栏位，彼时被用来传递一个用户自定义的指针。
} OVERLAPPED, *LPOVERLAPPED;
```
OVERLAPPED 结构执行两个重要的功能。第一，它像一把钥匙，用以识别每一个目前正在进行的 overlapped 操作。第二，它在你和系统之间提供了一个共享区域，参数可以在该区域中双向传递。  
通常overlapped结构存放在heap中。

### 被激发的File Handles

+ 异步IO的步骤：CreateFile指定`FILE_FLAG_OVERLAPPED`；设立一个OVERLAPPED结构，调用ReadFile、WriteFile带上这个参数。
+ 文件handle是一个核心对象，一旦操作完毕即被激发。
+ GetOverlappedResult（）

```c
BOOL GetOverlappedResult(
HANDLE hFile,                                        //文件设备的handle
LPOVERLAPPED lpOverlapped,                 //一个指针，指向overlapped结构
LPDWORD lpNumberOfBytesTransferred,  //一个指针，指向DWORD，保存真正被传输的字节数。
BOOL bWait                                          //是否要等待操作完成，TRUE表示等待。
);
```
+ 虽然你要求一个overlapped 操作，但它并不一定就是 overlapped！如果数据已经被放进 cache中，或如果操作系统认为它可以很快速地取得那份数据，那么文件操作就会在ReadFile() 返回之前完成，而 ReadFile() 将传回 TRUE。
+ 一个文件操作为 overlapped，而操作系统把“操作请求”放到队列中等待执行， ReadFile() 和、WriteFile()都会传回 FALSE 以示失败。这个行为并不是很直观， 你必须调用GetLastError() 并确定它传回 `ERROR_IO_PENDING`，那意味着“overlappedI/O 请求”被放进队列之中等待执行。GetLastError() 也可能传回其他的值，例如 `ERROR_HANDLE_EOF`，那就真正代表一个错误了。

### 被激发的event对象

+ 所使用的 event 对象必须是手动重置（manual-reset）而非自动重置（auto-reset）。
+ IOBYEVENT例子。

### 异步过程调用（Asynchronous Procedure Calls，APCs）

+ 使用overlapped I/O与event搭配的两个问题：
	+ WaitForMultipleObjects最多等待64个对象。
	+ 必须不断的根据“哪一个handle被激发”而计算如何反应。
+ 使用Ex版的ReadFile和WriteFile，可以使用异步过程调用机制。只有当线程处于alertable状态时，APCs才会被调用。当线程因为以下5个函数而处于等待状态，且线程的“alertable”标记被设为TRUE，则线程处于alertable状态：
	+ SleepEx（）
	+ WaitForSingleObjectEx（）
	+ WaitForMultipleObjectEx（）
	+ MsgWaitForMultipleObjectsEx（）
	+ SignalObjectAndWait();
+ 用于 overlapped I/O 的 APCs 是一种所谓的 user mode APCs。WindowsNT 另有一种所谓的 kernel mode APCs。Kernel mode APCs 也会像 usermode APCs 一样被保存起来，但一个 kernel mode APC 一定会在下一个timeslice 被调用，不管线程当时正在做什么。 Kernel mode APCs 用来处理系统机能，不在应用程序的控制之中。
+ 提供的 I/O completion routine 应该有这样的型式:

```c
VOID WINAPI FileIOCompletionRoutine(
DWORD dwErrorCode,   //0表示操作完成，ERROR_HANDLE_EOF表示操作已经到了文件尾端。
DWORD dwNumberOfBytesTransferred,//真正被传输的数据字节数
LPOVERLAPPED lpOverlapped//指向overlapped结构，此结构由开启overlapped I/O操作的函数提供
);
```
+ 使用 APCs 时，OVERLAPPED 结构中的 hEvent 栏位不需要用来放置一个 event handle。Win32 文件上说此时 hEvent 栏位可以由程序员自由运用。那么最大的用途就是：首先配置一个结构，描述数据来自哪里，或是要对数据进行一些什么操作，然后将 hEvent 栏位设定指向该结构
+ 在C++ 中产生一个I/O Completion Routines:储存一个指针，指向用户自定义数据（一个对象），然后经由此指针调用一个 C++ 成员函数。由于 static 成员函数是类的一部分，你还是可以调用 private 成员函数。

### 对文件进行overlapped I/O的缺点

+ 似乎 Windows NT 是以“I/O 请求”的大小来决定要不要将此请求先记录下来。所以对于数据量小的操作，overlapped I/O的效率反而更低。
+ 解决办法：以少量的线程负责所有的硬盘 I/O，然后把这些线程的I/O 请求，保持在一个队列之中。这种效率比较高。
+ 有两种情况，overlapped I/O 总是同步执行，甚至即使 `FILE_FLAG_NO_BUFFERING` 已经指定。第一种情况是你进行一个写入操作而造成文件的扩展。第二种情况是你读写一个压缩文件。

### I/O Completion Ports

+ APCs的缺点：最大的问题就是，有好几个 I/O APIs 并不支持 APCs，如listen() 和 WaitCommEvent() 便是两个例子。APCs 的另一个问题是，只有发出“overlapped 请求”的那个线程才能够提供 callback 函数，然而在一个“scalable”（译注）系统中，最好任何线程都能够服务 events。
+ 产生一个I/O Completion Port

```c
HANDLE CreateIoCompletionPort(
HANDLE FileHandle,                        //文件或设备的handle，若为INVALID_HANDLE_VALUE,则产生一个没有和任何handle关联的port。
HANDLE ExistingCompletionPort,      //若此栏位被指定，则FileHandle被加到此port上。指定Null产生一个新的port。
DWORD CompletionKey,                  //用户自定义的一个数值，将被交给提供服务的线程。此值和FileHanlde有关联。
DWORD NumberOfConcurrentThreads//与此I/O completion port 有关联的线程个数。
);
```
+ 与一个文件handle产生关联
　　再次使用CreateIoCompletionPort接口。
+ 在一个I/O Completion Port上等待

```c
BOOL GetQueuedCompletionStatus(
HANDLE CompletionPort,                        //将在其上等待completion port。
LPDWORD lpNumberOfBytesTransferred,  //指向DWORD,收到“被传输的数据字节数”。
LPDWORD lpCompletionKey,                   //指向DWORD,该DWORD将收到由CreateIoCompletionPort定义的key。
LPOVERLAPPED *lpOverlapped,               //overlapped结构指针的地址。
DWORD dwMilliseconds                          //等待的最长时间，时间终了，lpOverlapped被设为NULL，函数传回FALSE。
);
```
在completion port上等待的线程是以先进后出的次序提供服务。
+ 避免Completion Packets  
设定一个 OVERLAPPED 结构，内含一个合法的手动重置（manual-reset）event 对象，放在 hEvent 栏位。然后把该 handle 的最低位设为 1。  
overlap.hEvent = CreateEvent(NULL, TRUE, FALSE, NULL);  
overlap.hEvent = (HANDLE)((DWORD)overlap.hEvent | 0x1);  
WriteFile(hFile, buffer, 128,& dwBytesWritten, &overlap);

### 对Sockets使用Overlapped I/O

分析ECHO例子，多实践socket IOCP。