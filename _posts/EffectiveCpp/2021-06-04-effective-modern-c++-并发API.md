---
title: effective modern c++ 之 并发API
descriptions: 高效使用 c++11 特性
categories: language
tags: c++11
---

# 条款35 优先选用基于任务而非基于线程的程序设计

基于任务的程序设计表现着更高阶的抽象，它把你从线程管理的细节中解放了出来。有关线程的几个概念：

1. 硬件线程是实际执行计算的线程，每个cpu内核有一个或多个线程

2. 软件线程，是操作系统线程，用以实施跨进程管理，以及进行硬件线程调度的线程。

   软件线程是一种有限的资源，如果试图获取的数量超出系统能提供的数量时，会抛出std::system_error异常

3. std::thread是c++进程里的对象，用作底层软件线程的句柄

```cpp
int doAsyncWork();
std::thread t(doAsyncWork); // 基于线程的设计
auto fut = std::async(doAsyncWork); // 基于任务的设计
```

总结：

1. std::thread的API未提供直接获取异步运行函数返回值的途径，而且如果那些函数抛出异常，程序就会终止

2. 基于线程的程序设计要求手动管理线程耗尽、超订、负载均衡，以及新平台适配等问题

   超订：就绪状态的软件线程超过了硬件线程数量

3. 经由应用了默认启动策略的std::async进行基于任务的程序设计，大部分上述问题都能找到解决之道

需要使用线程设计的几种场景：

1. 需要访问底层线程实现的API
2. 需要有能力为你的应用优化线程用法
3. 需要实现超越c++并发API的线程技术

# 条款36 如果异步是必要的，则指定std::launch::async

std::async的两种启动策略：

1. `std::launch::async` 启动策略意味着函数f必须以异步方式运行，即在另一个线程上执行
2. `std::launch::deferred` 启动策略意味着函数f只会在std::async所返回的期值的get或wait得到调用时才运行

默认启动策略是` async | deferred`,这种弹性使得async能够承担处理线程管理的细节，如线程耗尽、超订、负载均衡等问题，但也有一些暗流：

（假设调用auto fut = std::async(f)的线程为t）

1. 无法预知f是否会和t并发运行，因为可能被调度为推迟运行（比如超订场景）
2. 无法预知f是否运行在与调用fut的get或wait函数的线程不同的某线程上
3. 连f是否最终会执行都是无法预知的。因为无法保证在程序的所有路径上，fut的get或wait都会得到调用

因此以默认策略使用std::async需要满足以下几个条件：

1. 任务不需要与调用get或wait的线程并发执行
2. 读、写哪个线程的thread_local变量并无影响
3. 或者可以给出保证在std::async 返回的期值之上调用get或wait，或者可以接受任务永不运行
4. 使用wait_for或wait_until的代码会将任务被推迟的可能性纳入考量

自动使用std::launch::async 策略的做法：

```cpp
template<typename F, typename... Ts>
inline std::future<typename std::result_of<F(Ts...)>::type>
reallyAsync(F&& f, Ts&&... params) {
  return std::async(std::launch::async, std::forward<F>(f), std::forward<Ts>(params)...);
}
```

总结：

1. std::async 的默认启动策略既允许任务以异步方式执行，也允许任务以同步方式运行

2. 这种弹性会导致使用thread_local变量时的不确定性，隐含着任务可能永远不会执行，还会影响运用了基于超时的wait调用的程序逻辑

   ```cpp
   using namespace std::literals;
   void f(){std::this_thread::sleep_for(1s);}
   auto fut = std::async(f);
   // 如果f被延迟执行，wait_for总是返回std::future_status::deferred， 导致基于超时while不会退出
   while(fut.wait_for(100ms) != std::future_status::ready) {}
   // 纠正方法：检验std::async返回的期值，确定f是否被推迟，如果被推迟，避免进入基于超时的循环
   if (fut.wait_for(0s) == std::future_status::deferred) {} 
   else { while(fut.wait_for(100ms) != std::future_status_ready) {} }
   ```

3. 如果异步是必要的，则指定std::launch::async

# 条款37 使std::thread型别对象在所有路径皆不可联结

每个std::thread对象均处于两种状态之一：可联结和不可联结。

可联结的std::thread对应底层以异步方式已运行或可运行的线程。底层线程处于阻塞或等待调度，或已运行结束，都是可联结的。

不可联结的std::thread对象包括：

1. 默认构造的std::thread。此类std::thread没有可执行函数，因此没有对应的底层执行线程
2. 已被移动的std::thread。移动的结果是底层执行线程被对应到另一个thread对象
3. 已联结的std::thread（join）。联结后，不再对应到已结束运行的底层执行线程
4. 已分离的std::thread（detach）。分离操作会把thread对象和底层执行线程的对应关系断开。

可联结性非常重要：如果可联结的线程对象的析构函数被调用，程序的执行就终止了。为什么标准库会选择这种行为？

1. 如果选择隐式join，会导致难以追踪的性能问题
2. 如果选择隐式detach，会导致难以调试的未定义行为。如使用了已回收的栈。

因此，必须确保从thread对象作用域出去的任何路径，使它成为不可联结状态。比如移动、join、detach.

使用RAII来选择thread对象销毁时使用join还是detach:

```cpp
#include <thread>
using std::thread;

class ThreadRAII {
 public:
  enum class DtorAction { join, detach };
  // 1. 只接受右值型别的std::thread，移入ThreadRAII
  // 2. 形参顺序的设计（thread在前，action在后）更符合调用者的直觉
  ThreadRAII(thread&& t, DtorAction a) : _a(a), _t(std::move(t)) {}
  ~ThreadRAII() {
    // 1. 校验thread以确保可联结。这是必要的，因为针对一个不可联结的线程调用join或detach会产生未定义行为
    // 2. 这里不需要考虑竞险。因为thread只能通过成员函数变成不可联结。但是析构函数调用时，不应该有其他线程调用该对象的成员函数
    if (_t.joinable()) {
      if (_a == DtorAction::join) {
        _t.join();
      } else {
        _t.detach();
      }
    }
  }
  // 析构函数的显示声明抑制了编译器生成移动函数，这里显示声明
  ThreadRAII(ThreadRAII&&) = default;
  ThreadRAII& operator=(ThreadRAII&&) = default;

  // 提供get可以避免让ThreadRAII去重复std::thread的所有接口，意味着ThreadRAII适用于直接使用thread的语境
  thread& get() { return _t; }

  // 一个成员变量的初始化有可能会依赖另一个成员变量，又因为std::thread型别对象初始化之后可能会马上用来运行函数，
  // 所以把他们声明在类的最后是个好习惯，这保证了底层线程执行时可以安全的访问其他成员变量。
 private:
  DtorAction _a;
  thread _t;
};
```

# 条款38 对变化多端的线程句柄析构函数行为保持关注

std::thread对象和期值都可以视作系统线程的句柄。本节讨论期值析构函数的行为。

首先讨论了被调方的结果存储位置：

1. 不会存储在被调方的std::promise型别对象里
2. 不能存储在调用方的期值中

结论：结果会存在堆上的一个共享对象中，这个位置称为共享状态。共享状态由指涉到他的期值和被调方的std::promise共同操纵。

期值的析构函数行为由与其关联的共享状态决定：

1. 指涉到经由std::async启动的未推迟任务的共享状态的最后一个期值会保持阻塞，直至该任务结束
2. 其他所有期值对象的析构函数只仅仅将期值对象析构就结束了

这可以总结为一个常规行为和非常规行为：

1. 常规行为指析构函数仅会析构期值对象
2. 非常规行为，只有在期值满足以下全部条件时才会发挥作用：
   1. 期值所指涉的共享状态是由于调用了std::async才创建的
   2. 该任务的启动策略是std::launch::async，这可能是运行时系统的选择，也可能是显示指定的
   3. 该期值是指涉到该共享状态的最后一个期值

std::package_task会包装一个可调用对象，将其执行结果置入一个共享状态，指涉到该共享状态的期值可以经由std::package_task::get_future得到:

```cpp
int calcValue() {return 0;}

{
	std::package_task<int()> pt(caclValue); // package_task对象一经创建，就会运行在线程之上。
	auto fut = pt.get_future();
	std::thread t(std::move(pt)); // package_task不能复制
	... // 这里t的析构函数的行为参考条款37，不管如何fut的析构函数都执行常规行为。
}
```

此时get_future获取的期值不是由std::async创建，因此其析构函数执行常规行为

# 条款39 考虑针对一次性事件通信使用以void为模板型别实参的期值

线程间通信，即让一个任务通知另一个异步运行的任务发生了特定的事件，有以下几种方式：

1. 条件变量

   把检测条件的任务称为检测任务，把对条件变量作出反应的任务称为反应任务。则反应任务等待条件变量，检测任务在事件发生时通知条件变量：

   ```cpp
   #include <gtest/gtest.h>
   #include <condition_variable>
   #include <future>
   #include <iostream>
   #include <thread>
   #include <chrono>
   
   std::condition_variable cv;
   std::mutex mu;
   TEST(CondVar, wait) {
     using namespace std::chrono_literals;
   
     // 异步启动反应任务
     auto fut = std::async(std::launch::async, []{
       std::cout << "before wait!" << std::endl;
       {
   	    std::unique_lock<std::mutex> lk(mu);
     	  cv.wait(lk);      
       }
       std::cout << "after wait!" << std::endl;
     });
     // sleep_for 确保反应任务已在wait
     std::this_thread::sleep_for(10ms);
     std::cout << "check variable!" << std::endl;
     // 检测任务发现事件已发生，通知条件变量，如果要通知多个反应任务，使用notify_all
     cv.notify_one();
   }
   ```

   使用条件变量作为线程间通信存在的问题：

   1. 代码异味（code smell）：本例中源于条件变量必须使用互斥体mutex，但检测和反应任务之间大有可能根本不需要这种介质。例如，检测任务负责初始化数据结构，然后把他交给反应任务使用，两者之间不会并发访问。

   2. 如果检测任务在反应任务调用wait之前就通知了条件变量，则反应任务将失去响应。

   3. 反应任务的wait语句无法应对虚假唤醒。所谓的虚假唤醒是指，即使没有通知条件变量，针对该条件变量等待的代码也可能被唤醒。正确的代码通过确认等待的条件确实已经发生，并将其作为唤醒后的首个动作来处理这种情况。

      ```cpp
      cv.wait(lk, []{return 事件是否已发生})；
      ```

      但反应任务可能无法确认它正在等待的事件是否已经发生。例如本例中是由检测任务来确认，这也是为什么反应任务等待的是个条件变量。

2.   使用共享的布尔标记位

   ```cpp
   std::atomic<bool> flag(false);
   TEST(ShareFlag, wait) {
     using namespace std::chrono_literals;
     // 异步启动反应任务
     auto fut = std::async(std::launch::async, []{
       std::cout << "before wait!" << std::endl;
       while(!flag){
         std::cout << "wait flag" << std::endl;
       }
       std::cout << "after wait!" << std::endl;
     });
     std::cout << "check variable!" << std::endl;
     // 条件满足时设置flag
     flag = true;
   }
   ```

   存在的问题：反应任务的轮询成本高昂。

3. 结合条件变量和基于标记位的设计

   ```cpp
   std::condition_variable cv;
   std::mutex mu;
   bool flag(false);
   
   TEST(CondVar, wait) {
     using namespace std::chrono_literals;
   
     // 异步启动反应任务
     auto fut = std::async(std::launch::async, []{
       std::cout << "before wait!" << std::endl;
       std::this_thread::sleep_for(10ms);
       {
         std::unique_lock<std::mutex> lk(mu);
         cv.wait(lk, []{return flag;});
       }
       std::cout << "after wait!" << std::endl;
     });
     std::cout << "check variable!" << std::endl;
     {
       std::lock_guard<std::mutex> g(mu);
       flag = true;
     }
     cv.notify_one();
     std::cout << "after notify" << std::endl;
   }
   ```

   这种方式没有“只使用条件变量”的问题，即使wait在notify之后调用

   存在的问题: 探测任务和反应任务之间的沟通方式非常奇特，即通过条件变量通知事件已发生，但是却要检查flag标志位才能确定。不够干净利落。

4. 让反应任务等待检测任务设置的期值

   发送端是std::promise型别对象，并且接收端是期值的通信信道，可以用于任何需要将信息从一处传到另一处的场合。

   这种设计简单易行。检测任务有一个std::promise型别对象（即，信道的写入端），反应任务有对对应的期值。当检测任务发现它正在查找的事件已经发生，会设置std::promise对象（即，向信道写入）。与此同时，反应任务调用wait以等待它的期值。该wait调用会阻塞反应任务直至std::promise型别对象被设置为止。std::promise和std::future的模板形参，表示的是要通过信道发送数据的型别。如果没有数据要发送，使用void来同步。

   ```cpp
   TEST(CondVar, wait) {
     using namespace std::chrono_literals;
     std::promise<void> p;
   
     // 异步启动反应任务
     auto fut = std::async(std::launch::async, [&p]{
       std::cout << "before wait!" << std::endl;
       p.get_future().wait();
       std::cout << "after wait!" << std::endl;
     });
     // 检测任务设置promise
     std::this_thread::sleep_for(10ms);
     std::cout << "check variable!" << std::endl;
     p.set_value();
   }
   ```

   promise和future之间的共享状态是动态分配的，因此需要考虑堆上分配和回收的成本。

   std::promise型别对象只能设置一次，因此只能通信一次，不能重复使用。（条件变量可以重复通知，标志位可以重复设置）

   ```cpp
   std::promise<void> p;
   
   void react() {std::cout << "react" << std::endl;}
   void detect() {
     ThreadRAII tr(std::thread([]{
       p.get_future().wait();
       react();
     }), ThreadRAII::DtorAction::join);
     //...   // 如果这里发生异常，detect函数将失去响应，因为tr的析构函数将永远不会完成
     p.set_value();
   }
   ```

   针对多个反应任务：

   ```cpp
   std::promise<void> p;
   void react() { std::cout << "react" << std::endl; }
   void detect() {
     auto sf = p.get_future().share();
     std::vector<ThreadRAII> vt;
     for(int i = 0; i < 10; i++) {
       vt.emplace_back( 
         std::thread([sf]{
           sf.wait();
           react();
         }), ThreadRAII::DtorAction::join);
     }
     //...   // 如果这里发生异常，detect函数将失去响应，理由同上。
     p.set_value();
   }
   ```

   ThreadRAII 参考条款37

# 条款40 对并发使用std::atomic, 对特种内存使用volatile

1. std::atomic 型别特性：一旦构造出std::atomic型别对象，其上所有的成员函数都保证被其他线程视为原子的。

2. std::atomic型别对象的运用会对代码可以如何重新排序施加限制。其中之一是，**不得将任何代码提前至后续会出现std::atomic型别变量的写入操作的位置**。如在共享标志位的运用中：

   ```cpp
   std::atomic<bool> flag(false);
   auto imptVlaue = computeImportantValue();
   flag = true;
   ```

   这意味着不仅编译器必须保持impValue和flag的赋值顺序，他们还必须生成代码以确保底层硬件也保证这个顺序。

3. 使用volatile不具备原子性，也不会给代码施加同样的重新排序方面的约束。因此他对于并发编程无用。volatile的用处是告诉编译器，正在处理的内存不具备常规行为。常规内存的特征是

   1. 如果你向某个内存位置写入了值，该值会一直保留在那里，直到他被覆盖为止。
   2. 如果向某内存位置写入某值，期间未读取该内存位置，然后再次写入该内存位置，则第一次写入可以消除，因为其写入结果从未使用过。

   ```cpp
   int x;
   auto y = x; // 读取x
   y = x;      // 再次读取x；根据第一特征，该赋值语句会被优化掉，因为和y的初始化冗余了
   
   x=10;
   x=20; // 根据第二特征，x=10 会被优化掉
   ```

4. 特种内存不具备常规内存特征，常见的特种内存是用于内存映射I/O的内存。这种内存的位置实际上是用于与外部设备通信，而非用于读取或写入常规内存，即RAM。因此针对常规内存的优化不再有效。volatile 用于告诉编译器，正在处理的是特种内存。他的意思是告知编译器“不要对在此内存上的操作做任何优化”。

5. std::atomic复制操作被删除，因此

   ```cpp
   std::atomic<int> x;
   auto y = x;// 错误，因为y被推导为std::atomic<int>,但是无法做到在一个原子操作中读取x并赋值给y（硬件限制）
   y = x; // 错误，理由同上
   
   // 正确的做法
   std::atomic<int> y(x.load());
   y.store(x.load());
   
   // 优化为
   register = x.load(); // x只读取了一次，这是在特种内存时必须避免的那种优化
   std::atomic<int> y(register);
   y.store(register);
   ```

6. 总结：

   std::atomic 对于并发程序设计有用，不能用于访问特种内中

   Volatile 对于访问特种内存有用，但不能用于并发程序设计

   两者可以一起使用，比如在多线程同时访问特种内存时：

   ```cpp
   volatile std::atomic<int> = val;// 原子操作，并且无法被编译器优化
   ```

   

