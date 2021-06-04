---
title: effective modern c++ 之 lambda 表达式
descriptions: 高效使用 c++11 特性
categories: language
tags: c++11
---

# 条款31 避免默认捕获模式

1. 理解lambda表达式、闭包、闭包类概念

   1. lambda表达式是表达式的一种，如`[](int value){return value % 2 == 0;}`
   2. 闭包是lambda式创建的运行期对象（运行期）
   3. 闭包类就是实例化闭包的类。每个lambda表达式都会触发编译器生成一个独一无二的闭包类。

2. 按引用的默认捕获模式会导致空悬指针问题

3. 按值的默认捕获极易受空悬指针影响（尤其是this），并会误导人们认为lambda是自洽的(独立的)

4. 捕获只能针对于在创建lambda式的作用域内可见的非静态局部变量（包括形参）,不包括成员变量

   ```cpp
   vector<std::function<bool(int)>> filters;
   
   class Widget {
    public:
     void addFilter() const;
   
    private:
     int divisor;
   };
   
   void Widget::addFilter() const {
     // divisor 不是局部变量和形参，无法捕获，因此这里实际捕获的是this指针（this指针无法被隐式捕获，因此[]中的=是必须的）
     filters.emplace_back([=](int value) { return value % divisor == 0; }); 
     // 更好的方式是捕获局部变量副本
     auto divisorCopy = divisor;
     filters.emplace_back([=](int value) { return value % divisorCopy == 0; }); 
     // c++14 广义lambda
     filters.emplace_back([divisor = divisor](int value) { return value % divisor == 0; }); 
   }
   ```

   为什么说默认值捕获的lambda不总是自洽的？因为lambda除了依赖捕获的局部变量和形参外，还会依赖静态存储期对象。这样的对象可以在lambda内使用，但是他们不能被捕获。包括全局作用域变量，在类中、函数中、文件中以static饰词声明的变量。

# 条款32 使用初始化捕获将对象移入闭包

c++11 不支持lambda表达式捕获对象时移入闭包

1. 使用c++14的初始化捕获将对象移入闭包

   ```cpp
   class Widget {
    public:
     bool isValidated() const;
     bool isArchived() const;
    private:
     int divisor;
   };
   
   TEST(LambdaTest, test1) {
     auto pw = std::make_unique<Widget>();
     //初始化捕获，也即广义lambda捕获，可以在[=]左边指定由lambda生成的闭包类中的成员变量的名字，右边指定一个表达式用以初始化该成员变量
     //[=]左右两边位于不同的作用域，左侧的作用域是闭包内的作用域，右侧的作用域则与lambda定义处的作用域相同
     auto func = [pw = std::move(pw)] { return pw->isValidated() && pw->isArchived(); };
     // 初始化表达式
     auto func2 = [pw = std::make_unique<Widget>()] { return pw->isValidated() && pw->isArchived(); };
   }
   ```

2. 在c++11中，经由手工实现的类或std::bind去模拟初始化捕获

   1. 使用手工实现的类来实现按移动捕获

   ```cpp
   class IsValAndArch {
    public:
     using DataType = std::unique_ptr<Widget>;
     explicit IsValAndArch(DataType&& ptr) : pw(std::move(ptr)) {}
     bool operator()() const { return pw->isValidated() && pw->isArchived(); }
   
    private:
     DataType pw;
   };
   
   TEST(LambdaTest, test1) { auto func = IsValAndArch(std::make_unique<Widget>()); }
   ```

   2. 使用std::bind

      1. 把需要捕获的对象移动到std::bind产生的函数对象中
      2. 使lambda表达式有一个指涉到被“捕获”的对象的引用

      比较c++14 和 c++11的移入闭包方法：

   ```cpp
   std::vector<double> data;
   auto func = [data = std::move(data)] {};                                          // c++14
   auto func = std::bind([](const std::vector<double>& data) {}, std::move(data));   // c++11
   
   auto func = [pw = std::make_unique<Widget>()]{return pw->isValidated() && pw->isArched();}; // c++14
   
   auto func = std::bind(                                                                      // c++11
     [](const std::unique_ptr<Widget>& pw){return pw->isValidated() && pw->isArched();}, 
     make_unique<Widget>()
   );
   ```

   ​	std::bind第一个实参是个可调用对象，剩下的参数是传给该对象的值。bind也会生成函数对象，称为绑定对象。绑定对象含有传递给bind的所有实参的副本。对于左值实参，对其内副本实施复制构造，对于右值实参，实施移动构造。当绑定对象被调用时，其内部的实参副本对象会按引用传递给可调用对象，对于上例来说，data的在函数对象中的副本会传递给lambda表达式的形参（引用）。

   默认情况下，lambda生成的闭包类中的operator()成员函数会带有const饰词。但是绑定对象里移动构造得到的data副本却不带有const饰词，所以为了达到与lambda一样的效果（防止副本被修改），lambda的形参声明为常量引用。如果lambda声明带有mutable，闭包里的operator ()函数声明就不会带有const饰词，因此lambda形参不需要带有const：

   ```cpp
   std::vector<double> data;
   auto func = std::bind([](std::vector<double>& data)mutable{}, std::move(data));
   ```

   总结：移动对象到c++11的闭包无法实现，但可以移动对象到绑定对象；c++11中的模拟：先移动构造一个对象到绑定对象中的副本，然后按引用把该副本传递给lambda表达式；绑定对象的声明周期和闭包相同。

# 条款33 对auto&&型别的形参使用decltype, 以std::forward之

c++14 支持泛型lambda表达式，即可在形参中使用auto。这个特性的实现是：在闭包类中的operator()采用模板实现，如：

```cpp
auto f = [](auto x){return func(normalize(x));};
// 则闭包类中函数调用运算符如下：
class SomeCompilerGeneratedClassName {
 public:
   template<typename T>
   auto operator()(T x)const {return func(normalize(x));}
};

// 使用完美转发的lambda表达式
auto f = [](auto&& x){return func(normalize(std::forward<decltype(x)>(x)));};
// 闭包类中的函数调用运算符实现：
class SomeCompilerGeneratedClassName {
 public:
   template<typename T>
   auto operator()(T&& x)const {return func(normalize(std::forward<decltype(x)>(x)));} // Target
};

// 可变参数完美转发lambda表达式
auto f = [](auto&&... params){return func(normalize(std::forward<decltype(params)>(params)...));};
```

可以参考条款3中decltype的类型推导和条款28中引用折叠实现forward，来解释这里为什么可以使用`decltyp(x)`来作为forward的模板实参。分析如下：

参考上例中`Target`，如果x是左值，万能引用推导T为`T&`，因此`decltyp(x)`为左值引用`T&`。

如果x是右值，万能引用推导T为非引用型别T，因此`decltyp(x)`为右值引用`T&&`，将其代入forward的实现，可以发现能得到一致的结果：

```cpp
// forward实现
template <typename T>
T&& forward(remove_reference_t<T>& param) {
  return static_cast<T&&>(param);
}

// decltype(x) == T&, 引用折叠后,与条款28传入左值结果一致
T& forward(T& param){return static_cast<T&>(param);}

// decltype(x) == T&&
T&& && forward(T& param){return static_cast<T&& &&>(param);}
// 引用折叠后, 与条款28中传入右值结果一致
T&& forward(T& param){return static_cast<T&&>(param);}
```

# 条款34 优先选用lambda，而非std::bind

1. lambda 表达式比起bind而言，可读性更好，表达力更强，可能运行效率也更高。
2. 仅在c++11中，std::bind在实现移动捕获，或是绑定到具备模板化的函数调用运算符的函数对象的场合中（多态函数对象），可能还有点用。但是在c++14中，lambda提供的初始化捕获和泛型形参的支持，可以完全取代这两个场景。

```cpp
#include <gtest/gtest.h>
#include <chrono>
class Widget {};

using Time = std::chrono::steady_clock::time_point;
enum class Sound { Beep, Siren, Whistle };
enum class Volume { Normal, Loud, LoudPlusPlus };
using Duration = std::chrono::steady_clock::duration;

void setAlarm(Time t, Sound s, Duration d);
void setAlarm(Time t, Sound s, Duration d, Volume v);

enum class CompLevel { Low, Normal, High };
Widget Compress(const Widget& w, CompLevel lev);

class PolyWidget {
 public:
  template <typename T>
  void operator()(const T& param);
};

TEST(LambdaTest, test2) {
  // lambda可读性更好，性能更好
  auto setSoundL = [](Sound s) {
    using namespace std::chrono;
    setAlarm(steady_clock::now() + hours(1), s, seconds(30));
  };
  using namespace std::chrono;
  using namespace std::placeholders;
  // 第二个bind用于延迟表达式评估，使得now在setAlarm被调用时调用，而不是第一个bind调用时。
  using SetAlarm3ParamType = void (*)(Time t, Sound s, Duration d);
  auto setSoundB = std::bind(static_cast<SetAlarm3ParamType>(setAlarm),  // 使用函数指针降低了setAlarm被内联的机会
                             std::bind(std::plus<steady_clock::time_point>(), steady_clock::now(), hours(1)),
                             _1,
                             seconds(30));

  // lambda 表达能力更强
  int lowVal = 5, highVal = 10;
  auto betweenL = [lowVal, highVal](int val) { return lowVal <= val && val <= highVal; };
  auto betweenB = std::bind(std::logical_and<bool>(),
                            std::bind(std::less_equal<int>(), lowVal, _1),
                            std::bind(std::less_equal<int>(), _1, highVal));

  // 参数的类型和传递方式，lambda都更清晰
  Widget w;
  auto compressRateL = [w](CompLevel lev) { return Compress(w, lev); };
  auto compressRateB = std::bind(Compress, w, _1);
  // std::bind总是复制或移动实参（参考条款32），但可以用std::ref来按引用传递
  auto compressRateB2 = std::bind(Compress, std::ref(w), _1);
  compressRateL(CompLevel::High);
  compressRateB(CompLevel::High);

  // 动态函数对象,可以接受任何型别的实参。c++11 使用std::bind, c++14使用lambda泛型形参
  // bind 绑定对象的函数调用运算符利用了完美转发，因此可以接受任何型别的实参。
  PolyWidget pw;
  auto boundPW = std::bind(pw, _1);
  boundPW(1930);
  boundPW(nullptr);
  boundPW("mark");
  auto boundPWL = [pw](const auto& param) { pw(param); };
}
```



