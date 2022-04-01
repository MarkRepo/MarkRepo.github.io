---
title: effective modern c++ 之 智能指针
descriptions: 高效使用 c++11 特性
categories: language
tags: c++11, shared_ptr, unique_ptr
---

# 四、智能指针

## 条款18 使用 std::unique_ptr 管理具有专属所有权的资源

1. `unique_ptr` 是小巧、高速的、具备只移型别的智能指针，对托管资源实施专属所有权语义

   1. 默认情况下可以认为其与裸指针有着相同的尺寸，并且对于大多数的操作（包括解引用），他们都是精确地执行了相同的指令

   2. `unique_ptr`的一个常用法是在对象的继承谱系中作为**工厂函数**的返回型别

      ```cpp
      class Investment {
        public:
        	virtual ~Investment();
      };
      class Stock: public Investment{};
      class Bond: public Investment{};
      class RealEstate : public Investment{};
      
      template <typename... Ts>
      std::unique_ptr<Investment> makeInvestment(Ts&&... params) {
        //...
      };
      // 使用方式
      {
        auto pInvestment = makeInvesment(arguments);
      }
      ```

      `unique_ptr ` 确保其管理的对象在所有可能的路径都会被析构，无需用户关心，除非一些非正常终止的特殊情况：

      1. 异常传播影响到了某个线程的主函数，如main
      2. 违反了noexcept异常规格, 见条款14
      3. std::abort, std::Exit, std::exit, std::quick_exit 等调用

2. 默认的，析构资源使用delete，但可以指定自定义删除器。有状态的函数对象和采用函数指针实现的删除器会增加unique_ptr 型别的对象尺寸（无状态的函数对象和无捕获的lambda表达式不会增加尺寸，即与裸指针一样  ）

   ```cpp
   // 注意删除器的参数类型
   auto delInvmt = [](Investment* pInvest) {
     makeLogEntry(pInvest);
     delete pInvest;
   }
   
   template <typename... Ts>
   std::unique_ptr<Investment, decltype(delInvmt)> // c++14 中可以使用auto代替
   makeInvestment(Ts&&... params) {
     std::unique_ptr<Investment, decltype(delInvmt)> pInv(nullptr, delInvmt);
     
     if (/*应创建一个Stock对象*/) {
       pInv.reset(new Stock(std::forward<Ts>(param)...));
     } else if (...) {
       pInv.reset(new Bond(std::forward<Ts>(param)...));
     } else {
       pInv.reset(new RealEstate(std::forward<Ts>(param)...));
     }
     return pInv;
   };
   ```

   注意：自定义删除器时，unique_ptr的类型改变了。

3. `unique_ptr` 转化为 `shared_ptr` 是容易实现的

   ```cpp
   std::shared_ptr<Investment> sp = makeInvestment(argument);
   ```

4. c++11中自己实现make_unique

   ```cpp
   template <typename T, typename... Ts>
   std::unique_ptr<T> make_unique(Ts... params) {
     return std::unique_ptr<T>(new T(std::forward<Ts>(params)...));
   }
   ```

## 条款19 使用std::shared_ptr管理具备共享所有权的资源

1. std::shared_ptr 提供方便的手段，实现了任意资源在共享所有权语义下进行生命周期管理的垃圾回收

2. shared_ptr的尺寸通常是裸指针的两倍（一个指向管理的对象，另一个指向控制块），还会带来额外控制块的开销（控制块必须动态分配），并要求原子化的引用计数操作。控制块包含引用计数、弱计数（weak_ptr）、其他数据（自定义删除器，分配器等）。

3. 一个对象的控制块由创建首个管理该对象的`std::shared_ptr`的函数来确定。因为无法从一个对象推断出是否有其他的shared_ptr已经管理该对象。推论：

   1. `std::make_shared` 总是创建一个控制块
   2. 从`std::unique_ptr`构造`shared_ptr`时，会创建一个控制块
   3. 使用裸指针创建`shared_ptr`时，会创建控制块

   注意：这些规则导致从同一个裸指针触发构造多个`shared_ptr`会产生未定义的行为，因为会析构多次。

4. `enable_shared_from_this<T>`  模板是为了解决通过this指针构造`shared_ptr`，导致多个`shared_ptr`管理同一个对象的问题（如果在类外部将对象托管给`shared_ptr`）。它定义了一个成员函数`shared_from_this()`, 其会创建一个`shared_ptr`管理该对象，但同时不会创建控制块。从其内部实现角度看，`shared_from_this` 查询当前对象的控制块，并创建一个指涉到该控制块的新`shared_ptr`。这样的实现依赖于当前对象已有一个与其关联的`shared_ptr`，否则`shared_from_this`将抛出异常。为了避免用户在有一个`shared_ptr`指向该对象前，就调用了会引发`shared_from_this`调用的成员函数，继承自 `enable_shared_from_this`的类通常会将其构造函数声明为`private`访问层级，并且只允许用户通过调用返回`std::shared_ptr`的工厂函数来创建对象:

   ```cpp
   std::vector<std::shared_ptr<Widget>> processWidgets;
   class Widget: public std::enable_shared_from_this<Widget> {
     public:
     	// 将实参完美转发给 private 构造函数的工厂函数
     	template<typename... Ts>
     	static std::shared_ptr<Widget> Create(Ts&&... Params);
     
     	void process() {
         processWidgets.emplace_back(shared_from_this());
       }
     private:
     	Widget();
   };
   ```

5. 默认的资源析构使用delete，但也支持自定义删除器。删除器的型别对shared_ptr的型别没有影响， 因此不会改变shared_ptr的尺寸。

   ```cpp
   auto loggingDel = [](Widget* pw) {
   	makeLogEntry(pw);
   	delete pw;
   }
   std::shared_ptr<Widget> spw(new Widget, loggingDel); // make_shared不支持自定义析构器
   ```

6. 避免使用裸指针型别的变量来创建shared_ptr指针，使用make_shared代替。如果必须使用裸指针，则直接传递new 运算符的结果。

7. shared_ptr可移动，但无法转为unique_ptr, 同时它也没有数组版本`shared_ptr<T[]>`。

## 条款20 对于类似 std::shared_ptr 但有可能空悬的指针使用 std::weak_ptr

1. weak_ptr 不参与引用计数，即不共享对象的所有权，因此它需要判断所指对象是否已经析构，即空悬。

2. weak_ptr 不是一种独立的智能指针，他是shared_ptr 的一种扩充，一般是通过 shared_ptr 来创建。它本身不能提领，也不能检查是否为空。

   ```cpp
   auto spw = std::make_shared<Widget>();
   std::weak_ptr<Widget> wpw(spw);
   if (wpw.expire()) // 通过expire来判断是否失效
   ```

3. **校验weak_ptr是否失效，以及在未失效时提供对所指对象的访问**, 这需要是一个原子操作，否则会带来竞险。这个原子操作可以通过由 weak_ptr 来创建shared_ptr 实现。有两种形式，选择哪种取决于在失效时期望得到什么结果：

   1. 使用 std::weak_ptr::lock, 返回一个shared_ptr，如果已经失效，**shared_ptr 为空**
   2. 使用weak_ptr 作为实参来构造shared_ptr, 如果已经失效，**抛出异常**

   ```cpp
   std::shared_ptr<Widget> spw1 = wpw.lock(); // 方式1，可用auto简化
   std::shared_ptr spw2(wpw); // 方式2
   ```

4. weak_ptr的用武之地包括缓存、观察者列表，以及避免shared_ptr 指针环路

   1. 缓存一些计算成本高的结果对象，并在对象不再使用时将其删除，避免缓存拥塞。缓存管理器持有weak_ptr

   ```cpp
   // 不带缓存的 工厂函数， loadWidget是个耗时操作
   std::unique_ptr<const Widget> loadWidget(WidgetID id);
   // 带缓存的工厂函数，由用户决定对象的生存期
   std::shared_ptr<const Widget> fastLoadWidget(Widget id) {
     static std::unordered_map<WidgetID, std::weak_ptr<const Widget>> cache;
     auto objPtr = cache[id].lock();
     if (!objPtr) {
       objPtr = loadWidget(id);
       cache[id] = objPtr;
     }
     return objPtr;
   }
   ```

   2. 在观察者模式中，每个主题包含一个容器持有其观察者的weak_ptr ，因为主题不控制观察者的生存期，在使用时检查是否空悬。

## 条款21 优先使用 std::make_unique 和 std::make_shared, 而非直接使用 new

1. 相比于直接使用new，make系列函数消除了重复代码、改进了异常安全性、并且对于make_shared和allocated_shared而言，生成的目标代码会尺寸更小、速度更快

   ```cpp
   // 1. 消除重复(下面使用new的方式，Widet型别重复输入)
   auto spw1(std::make_shared<Widget>());
   std::shared_ptr<Widget> spw2(new Widget);
   
   // 2. make版本改进了异常安全性，原因在于new版本 processWidget 调用时，参数的准备可能是下面的顺序执行：
   // (1) new Widget
   // (2) computePriority() 
   // (3) shared_ptr 构造函数
   // computePriority 可能异常从而导致new Widget 资源泄漏，因为此时其还没有放到shared_ptr里面管理
   int computePriority();
   void processWidget(std::shared_ptr<Widget> spw, int priority);
   
   // new 版本
   processWidget(std::shared_ptr<Widget>(new Widget), computePriority());// 存在潜在的资源泄漏
   // make 版本
   processWidget(std::make_shared<Widget>(), compuptePriority());//不会发生资源泄漏
   
   // 3. 性能的提升：make_shared会分配单块内存既保存Widget对象又保存与其关联的控制块，
   // 相对于new而言，这种优化减少了一次内存分配，并且减小了程序的静态尺寸
   ```

2. 不适于使用make系列函数的场景包括需要定制删除器、以及直接传递大括号初始化物（参考条款7）

   1. 在make系列函数里，对形参进行完美转发的代码使用的是圆括号而非大括号
   2. 因此如果需要使用大括号初始化物，必须使用new；或者使用auto型别推导中转(条款30中的变通方案)：

   ```cpp
   auto initList = {10, 20};
   auto spv = std::make_shared<std::vector<int>>(initList);
   ```

3. 对于shared_ptr，不建议使用make系列函数的额外场景包括：

   1. 自定义内存管理的类

      通常自定义内存管理类被设计成仅用来分配和释放该类精确尺寸的内存块。因为std::allocated_shared 所要求的内存数量并不等于动态分配对象的尺寸，而是还要加上控制块的尺寸。

   2. 内存紧张的系统、非常大的对象、以及存在比指涉到相同对象的shared_ptr生存期更久的weak_ptr

      如前所述，使用make系列函数分配的对象和控制块位于同一块内存。当对象的引用计数变为0时，对象被析构，但是托管对象的内存直到与其关联的控制块也被析构时才会释放，因为同一动态分配的内存块同时包含了两者。而控制块中除了引用计数，还有弱计数，用来管理weak_ptr。因此weak_ptr会指涉到控制块，只有当最后一个shared_ptr 和最后一个weak_ptr都被析构时，控制块才会被析构，进而对象的内存才会被释放。如果对象比较大，且最后一个shared_ptr被析构和最后一个weak_ptr被析构之间的时间隔不能忽略时，那么对象的析构和内存的释放之间就会产生延迟。此时应该使用new，但是得避免前文提到的异常安全问题，同时考虑到性能：

      ```cpp
      std::shared_ptr<Widget> spw(new Widget, cusDel);
      processWidget(std::move(spw), computePriority()); //使用std::move，用移动避免复制以优化性能，避免了引用计数的操作
      ```

## 条款22 使用 Pimpl 习惯用法时，将特殊成员函数的定义放到实现文件中

1. Pimpl惯用法(`pointer to implementation`)通过降低类的客户和类实现者之间的依赖性，减少了构建遍数

   1. 声明一个指针型别的数据成员，指涉到一个非完整型别
   2. 动态分配和回收持有从前在原始类里的那些数据成员的对象，而分配和回收代码则放在实现中

2. 对于采用std::unique_ptr来实现的pImpl指针，须在类的头文件中声明特种成员函数，但在实现文件中实现他们。即使默认函数的实现有着正确的行为，也必须这样做。

   以下面示例来说明，如果我们没有声明widget的析构函数，编译器将为我们合成一个，并在内部调用unique_ptr的析构函数，unique_ptr默认使用delete来析构其管理的对象，然而在delete之前，典型的实现会使用c++11中的static_assert来确保裸指针未指涉到非完整型别。因此这里的`struct Impl`非完整型别将导致static_assert失败，这个错误信息和`Widget w;`的析构位置有关，因为特种成员函数基本上都是隐式inline的。如在MacOS clang12.0下报错如下：

   `error: invalid application of 'sizeof' to an incomplete type 'Widget::Impl' static_assert(sizeof(_Tp) > 0`

   解决办法是，保证析构函数在生成析构代码时，Widget::Impl是个完整型别。这就是为什么要在实现文件中实现的原因。

3. 上述建议不 适用于shared_ptr。因为对自定义析构器的支持的实现方式不同，shared_ptr对其指涉到的型别并不要求是完整型别。

下面用代码示例来解释：

```cpp
// gadget.h
class Gadget {};
```

```cpp
// widget.h
#include <memory>
using std::unique_ptr;
using std::shared_ptr;

class Widget {
 public:
  Widget();
  ~Widget();

  Widget(Widget&& rhs);
  Widget& operator=(Widget&& rhs);

  Widget(const Widget& rhs);
  Widget& operator=(const Widget& rhs);

 private:
  struct Impl;
  unique_ptr<Impl> pImpl;
};
```

```cpp
// widget.cc
#include "widget.h"

#include <string>
#include <vector>

#include "gadget.h"

struct Widget::Impl {
  std::string name;
  std::vector<double> data;
  Gadget g1, g2, g3;
};

Widget::Widget() : pImpl(std::make_unique<Impl>()) {}
Widget::~Widget() = default;

Widget::Widget(Widget&& rhs) = default;
Widget& Widget::operator=(Widget&& rhs) = default;

Widget::Widget(const Widget& rhs) : pImpl(std::make_unique<Impl>(*rhs.pImpl)) {}
Wdiget& Widget::operator=(const Widget& rhs) {
  *pImpl = *rhs.pImpl;
  return *this;
}
```

```cpp
// test.cc 只依赖 widget.h, 而不依赖 widget.cc 中依赖的头文件
#include <gtest/gtest.h>

#include "widget.h"

TEST(SharedPtrTest, enableSharedFromThis) { 
  Widget w; 
}
```

BUILD:

```bazel
load("@rules_cc//cc:defs.bzl", "cc_library", "cc_test")

cc_library(
    name = "gadget",
    hdrs = ["gadget.h"],
    visibility = ["//visibility:public"],
)

cc_library(
    name = "widget",
    hdrs = [
        "widget.h",
    ],
    visibility = ["//visibility:public"],
    deps = [
        ":gadget",
    ],
)

cc_test(
    name = "test",
    srcs = [
        "test.cc",
    ],
    deps = [
        ":widget",
        "@gtest",
        "@gtest//:gtest_main",
        "@local_boost//:libboost",
    ],
)
```

