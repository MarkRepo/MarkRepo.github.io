---
title: effective modern c++ 之型别推导
descriptions: 高效使用 c++11 特性
categories: language
tags: c++11
---

# Why

为什么要看这本书？

1. 了解 c++11 特性及用法
2. 了解各个特性的最佳实践
3. 了解如何高效使用这些新特性，
4. 了解新特性背后的运作原理

# what

1. 型别推导
2. auto
3. 转向现代c++
4. 智能指针
5. 右值引用、移动语义和完美转发
6. lamba表达式
7. 并发API
8. 微调

# how

如何学习？如何实践这些特性？怎样加强对新特性的理解和掌握？

1. 按章节、条款学习，每个章节中的条款都集中讲一个大的特性，便于理解和学习
2. 实践：学习的时候，多上手写一写示例代码，再就是平时写cpp要有意识的去运用这些新特性，还可以看看trpc-cpp等新开源项目的代码
3. 如何加强理解：多复习和回忆自己整理和总结的博客，多实践，多思考。

# 一、型别推导

c++98 只支持一套型别推导规则，用于函数模板。c++11对这套规则进行了一些改动，并增加了2套规则：一套用于auto，一套用于decltype。c++14 扩展了能够运用auto和decltype的语境。

## 条款1： 理解模板型别推导

函数模板型别推导要解决的问题:

 ```cpp
template<typename T>
void f(ParamType param);

f(expr);
 ```

编译器要根据 expr 的类型，推导出 T 和 ParamType 的类型，其中ParamType 会带有一些型别饰词，如const，volatile，引用符号`&`、`&&`等。

T  的类型推导结果，不仅依赖expr的类型，还依赖ParamType的类型。分3种情况：

1. ParamType 具有指针或引用型别，但不是个万能引用
2. ParamType 是一个万能引用
3. ParamType 既非指针也非引用

###情况1、ParamType 具有指针或引用型别，但不是个万能引用 

这种情况下，型别推导运作如下：

1. 若expr具有引用型别，先将引用部分忽略
2. 然后，对expr的型别和ParamType的型别执行模式匹配，以决定T的类型

后面的示例中会一直使用 boost::typeindex::type_id_with_cvr 来打印出编译器推导的类型结果(精确)

```cpp
#include <gtest/gtest.h>
#include <boost/type_index.hpp>
#include <iostream>
using namespace std;

template <typename T, typename ParamT>
void print_type() {
  using boost::typeindex::type_id_with_cvr;
  cout << "T = " << type_id_with_cvr<T>().pretty_name()
       << ", Param = " << type_id_with_cvr<ParamT>().pretty_name() << endl;
}

template <typename T>
void f(T& param) {
  print_type<T, decltype(param)>();
}

template <typename T>
void cf(const T& param) {
  print_type<T, decltype(param)>();
}

template <typename T>
void cfp(T* param) {
  print_type<T, decltype(param)>();
}

TEST(TypeTest, typeidcvr) {
  int x = 27;
  const int cx = x;
  const int& rx = x;
  volatile int vx = x;
  f(x);
  f(cx);
  f(rx);  // 引用被忽略
  f(vx);
  cout << "-----------------------------------" << endl;
  cf(x);
  cf(cx);
  cf(rx);  // 引用被忽略
  cf(vx);
  cout << "-----------------------------------" << endl;
  const int* cpx = &x;
  cfp(&x);
  cfp(cpx); // 引用被忽略
}
```

结果：

 ```shell
T = int, Param = int&
T = int const, Param = int const&
T = int const, Param = int const&
T = int volatile, Param = int volatile&
-----------------------------------
T = int, Param = int const&
T = int, Param = int const&
T = int, Param = int const&
T = int volatile, Param = int const volatile&
-----------------------------------
T = int, Param = int*
T = int const, Param = int const*
 ```

结果的含义：

1. 模板的引用形参保留了对象的属性，比如常量性(const)，挥发性(volatile)等，参考推导出的Param类型
2. 当形参是右值引用时，实参也必须是右值引用才符合情况1（参考情况2.2）
3. 指针与引用的型别推导规则一致

### 情况2、ParamType 是一个万能引用

1. 如果expr是个左值， T 和 ParamType 都会被推导为左值引用。结论：
   1. 这是在模板类型推导中，T被推导为引用型别的唯一情形
   2. 模板声明时使用的是右值引用语法，型别推导结果确是左值引用
2. 如果expr是个右值，则应用常规的规则（情况1的规则）

```cpp
#include <gtest/gtest.h>
#include <boost/type_index.hpp>
#include <iostream>
using namespace std;
using boost::typeindex::type_id_with_cvr;

template <typename T, typename ParamT>
void print_type() {
  cout << "T = " << type_id_with_cvr<T>().pretty_name()
       << ", Param = " << type_id_with_cvr<ParamT>().pretty_name() << endl;
}

template <typename T>
void f(T&& param) {
  print_type<T, decltype(param)>();
}

int&& getint() {
  return 27;
}

TEST(TypeTest, typeidcvr) {
  int x = 27;
  const int cx = x;
  const int& rx = x;
  int&& rrx = 27;
  f(x);
  f(cx);
  f(rx);
  f(27);
  f(rrx); // rrx 其实是个左值
  f(getint());
  cout << "decltype(27) = "<< type_id_with_cvr<decltype(27)>().pretty_name() << endl;
  cout << "decltype(getint()) = "<< type_id_with_cvr<decltype(getint())>().pretty_name() << endl;
}
```

结果：

```shell
T = int&, Param = int&
T = int const&, Param = int const&
T = int const&, Param = int const&
T = int, Param = int&&
T = int&, Param = int&
T = int, Param = int&&
decltype(27) = int
decltype(getint()) = int&&
```

表达式的型别与它是左值还是右值没有关系。一个判断方法是看能不能取它的地址。  

万能引用形参的型别推导规则不同于左值引用形参和右值引用形参，关键在于它会区分实参是左值还是右值，参见条款24的解释。  

**问题：右值引用形参的语法跟万能引用是一样的？如果是，编译器如何区分两者，从而使用不同的型别推导规则？**

### 情况3、ParamType 既非指针也非引用

1. 若expr具有引用型别，则忽略其引用部分
2. 然后，若expr是个const对象，也忽略其const属性；若expr是个volatile对象，也忽略其volatile属性。

关键在于，这种情况会按值传递，意味着，param是实参的一个副本，一个全新的对象，他不会继承实参对象的属性。

```cpp
#include <gtest/gtest.h>
#include <boost/type_index.hpp>
#include <iostream>
using namespace std;
using boost::typeindex::type_id_with_cvr;

template <typename T, typename ParamT>
void print_type() {
  cout << "T = " << type_id_with_cvr<T>().pretty_name()
       << ", Param = " << type_id_with_cvr<ParamT>().pretty_name() << endl;
}

template <typename T>
void f(T param) {
  print_type<T, decltype(param)>();
}

TEST(TypeTest, typeidcvr) {
  int x = 27;
  const int cx = x;
  const int& rx = x;
  int&& rrx = 27;
  const char* const ptr = "why, what, how"; // ptr 自身的const 属性被忽略，而不是他指向对象的常量性
  f(x);
  f(cx);
  f(rx);
  f(27);
  f(rrx); // rrx 其实是个左值  
  f(ptr); 
}
```

结果

```shell
T = int, Param = int
T = int, Param = int
T = int, Param = int
T = int, Param = int
T = int, Param = int
T = char const*, Param = char const*
```

### 数组和函数实参

1. 按值传递给函数模板的数组型别将被推导成指针型别
2. 按引用传递给函数模板的数组型别，T 将被推导为数组型别，ParamzType被推导为数组的引用型别。
3. 函数型别也可以退化成函数指针，因此函数型别实参的推导规则与数组一样

```cpp
#include <gtest/gtest.h>
#include <boost/type_index.hpp>
#include <iostream>
using namespace std;
using boost::typeindex::type_id_with_cvr;

template <typename T, typename ParamT>
void print_type() {
  cout << "T = " << type_id_with_cvr<T>().pretty_name()
       << ", Param = " << type_id_with_cvr<ParamT>().pretty_name() << endl;
}

template <typename T>
void f(T param) {
  print_type<T, decltype(param)>();
}

template <typename T>
void rf(T& param) {
  print_type<T, decltype(param)>();
}

int sum(int a, int b) {
  return a+b;
}

TEST(TypeTest, typeidcvr) {
  const char name[] = "markfqwu";
  f(name);
  rf(name);
  f(sum);
  rf(sum);
}
```

结果:

```shell
T = char const*, Param = char const*
T = char const [9], Param = char const (&) [9]
T = int (*)(int, int), Param = int (*)(int, int)
T = int (int, int), Param = int (&)(int, int)
```

由于”数组在很多语境下会被退化成指针“ 这条规则的存在，使得我们无法声明真正的数组型别形参（在编译器看来，数组型别形参和指针型别形参是等价的），使用引用形参的函数模板是获得这种型别的一种手段。这给了我们一种在编译期获取数组大小的途径：

```cpp
#include <gtest/gtest.h>
#include <boost/type_index.hpp>
#include <iostream>
using namespace std;
using boost::typeindex::type_id_with_cvr;

template <typename T, std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept {
  return N;
}

TEST(TypeIndex, arraySize) {
  int keys[] {1,2,3,4,5,6,7,8,9};
  int values[arraySize(keys)]; // arraySize 声明为 constexpr，使其可以在编译期使用，即可以作为数组大小
  cout << sizeof(values) << endl; // 36
}
```

## 条款2  理解Auto型别推导

1. auto的型别推导和模板型别推导是一模一样的，二者之间可以建立起一一映射。除了auto型别推导会假定用大括号括起的初始化表达式代表一个`std::initializer_list`, 但模板型别推导却不会。
2. 在函数的返回值或者lambda表达式的形参中使用auto，会使用模板型别推导而非auto型别推导。(c++14)

### auto 型别推导与模板型别推导一致的情况

auto型别推导是如何与模板型别推导一一映射的呢？

回顾条款1，型别推导要解决的问题是，通过expr的类型来推导T和ParamType的类型：

```cpp
template<typename T>
void f(ParamType param);

f(expr);
```

当某变量采用auto来声明时，**auto就扮演了模板中的T这个角色，而变量的型别饰词扮演的是ParamType的角色，用来初始化auto变量的表达式扮演的是expr**：

```cpp
auto x = 27;         // 型别饰词是auto本身
const auto cx = x;   // 型别饰词是const auto
const auto& crx = x; // 型别饰词是const auto&
```

auto的型别推导规则不仅依赖初始化表达式，也依赖型别饰词。型别饰词与ParamType一样，也分三种情况：

1. 型别饰词是指针或引用，但不是万能引用
2. 型别饰词是万能引用
3. 型别饰词既不是指针也不是引用

为了推导x, cx, crx的类型，编译器的行为仿佛 ”对应于每个声明生成了一个模板和一次使用对应的初始化表达式针对该模板的调用“  一样（下面模板是概念性的，即实际上并没有生成，只是为了解释方便）：

```cpp
#include <gtest/gtest.h>
#include <boost/type_index.hpp>
#include <iostream>

using namespace std;
using boost::typeindex::type_id_with_cvr;

template <typename T, typename ParamT>
void print_type() {
  cout << "T = " << type_id_with_cvr<T>().pretty_name()
       << ", ParamType = " << type_id_with_cvr<ParamT>().pretty_name() << endl;
}

template <typename T>
void print_type() {
  cout << "ParamType = " << type_id_with_cvr<T>().pretty_name() << endl;
}

// 情况3概念性模板
template <typename T>
void func_for_x(T param) {
  print_type<T, decltype(param)>();
}
// 情况3概念性模板
template <typename T>
void func_for_cx(const T param) {
  print_type<T, decltype(param)>();
}
// 情况1概念性模板
template <typename T>
void func_for_rx(T& param) {
  print_type<T, decltype(param)>();
}
// 情况1概念性模板
template <typename T>
void func_for_crx(const T& param) {
  print_type<T, decltype(param)>();
}
// 情况2概念性模板
template <typename T>
void func_for_uref(T&& param) {
  print_type<T, decltype(param)>();
}

int sum(int a, int b) {return a+b;}

TEST(TypeTest, typeidcvr) {
  auto x = 27;          // 情况3
  const auto cx = x;    // 情况3
  const auto& crx = x;  // 情况1

  auto&& uref1 = x;     // 情况2，左值
  auto&& uref2 = cx;    // 同上
  auto&& uref3 = 27;    // 情况2，右值

  const char name[] = "markfqwu"; 
  auto arr1 = name;  // 数组按值传递，退化为指针
  auto& arr2 = name; // 数组按引用传递，得到数组的引用类型
  auto func1 = sum;  // 函数按值传递，退化为函数指针
  auto& func2 = sum; // 函数按引用传递，得到函数的引用类型
  
  cout <<"---------- 编译器推导出的实际类型 ----------" << endl;
  print_type<decltype(x)>();      // 实际推导出的类型
  print_type<decltype(cx)>();
  print_type<decltype(crx)>();
  cout << endl;
  print_type<decltype(uref1)>(); 
  print_type<decltype(uref2)>();
  print_type<decltype(uref3)>();
  cout << endl;
  print_type<decltype(arr1)>();
  print_type<decltype(arr2)>();
  print_type<decltype(func1)>();
  print_type<decltype(func2)>();
  cout <<"---------- 概念性模板输出 ----------" << endl;
  func_for_x(27);                 // 概念性调用语句
  func_for_cx(x);
  func_for_crx(x);
  cout << endl;
  func_for_uref(x);
  func_for_uref(cx);
  func_for_uref(27);
  cout << endl;
  func_for_x(name);
  func_for_rx(name);
  func_for_x(sum);
  func_for_rx(sum);
}
```

结果（可以看到概念性模板推导的ParamType类型与其实际类型（decltype显示的）一致）：

```shell
---------- 编译器推导出的实际类型 ----------
ParamType = int
ParamType = int const
ParamType = int const&

ParamType = int&
ParamType = int const&
ParamType = int&&

ParamType = char const*
ParamType = char const (&) [9]
ParamType = int (*)(int, int)
ParamType = int (&)(int, int)
---------- 概念性模板输出 ----------
T = int, ParamType = int
T = int, ParamType = int const
T = int, ParamType = int const&

T = int&, ParamType = int&
T = int const&, ParamType = int const&
T = int, ParamType = int&&

T = char const*, ParamType = char const*
T = char const [9], ParamType = char const (&) [9]
T = int (*)(int, int), ParamType = int (*)(int, int)
T = int (int, int), ParamType = int (&)(int, int)
```

### 唯一的例外情况，大括号初始化表达式

```cpp
#include <gtest/gtest.h>
#include <boost/type_index.hpp>
#include <iostream>
using namespace std;
using boost::typeindex::type_id_with_cvr;

template <typename T, typename ParamT>
void print_type() {
  cout << "T = " << type_id_with_cvr<T>().pretty_name()
       << ", ParamType = " << type_id_with_cvr<ParamT>().pretty_name() << endl;
}

template <typename T>
void print_type() {
  cout << "ParamType = " << type_id_with_cvr<T>().pretty_name() << endl;
}

// 情况3概念性模板
template <typename T>
void func_for_x(T param) {
  print_type<T, decltype(param)>();
}

template<typename T>
void func_for_init(std::initializer_list<T> param) {
  print_type<T, decltype(param)>();
}

TEST(TypeTest, exception) {
  auto x1 = {27};
  auto x2{27};
  auto x4{1, 2};         // error: initializer for variable 'x4' with type 'auto' contains multiple expressions
  auto x5 = {1, 2, 3.0}; // error: deduced conflicting types ('int' vs 'double') for initializer list element type
  print_type<decltype(x1)>(); // std::initializer_list<int>
  print_type<decltype(x2)>(); // int, 注意这里与书上结果不一致，书上结果是std::initializer_list<int>
  func_for_x({27});    // error:no matching function for call to 'func_for_x'
                       // candidate template ignored: couldn't infer template argument 'T'
  										 // 模板型别推导不知道大括号表达式
  func_for_init({1,2,3}); // T = int, ParamType = std::initializer_list<int>
}
```

x5的推导经过了两个阶段：

1. auto的类型推导，结果为std::initializer_list`<T>`
2. std::initializer_list `<T>`模板中T类型的推导, 这属于模板型别推导的范畴，这里发生了类型冲突，导致推导失败。

### c++14 中的返回值和lambda形参中的auto

这种情况会使用模板型别推导而非auto型别推导，即不支持大括号初始化表达式:

```cpp
auto createInitList(){ // 返回值auto
  return {1,2,3}; // error: cannot deduce return type from initializer list
}

TEST(TypeTest, exception) {
  std::vector<int> v;
  auto resetV = [&v] (const auto& newValue) {v = newValue;}; // lambda 表达式形参
  resetV({1,2,3}); // error: no matching function for call to object of type '(lambda)'
  								 // candidate template ignored: couldn't infer template argument ''
}
```

## 条款3 理解decltype

1. 绝大数情况下，decltype会得出变量或表达式的声明型别而不做任何修改。（参考前面条款中的示例）
2. 对于型别为T的左值表达式，除非该表达式仅有一个名字，否则decltype总是得出T&
3. c++14支持`decltype(auto)`,  和auto一样，他会从其初始化表达式出发来推导型别，但是他的推导使用的是decltype的规则

在c++11中，decltype的主要作用就在于声明那些返回值型别依赖于其形参型别的函数模板，下面以`operator[]`模板举例说明：

```cpp
#include <gtest/gtest.h>
#include <boost/type_index.hpp>
#include <iostream>
#include <deque>
using namespace std;
using boost::typeindex::type_id_with_cvr;

template <typename T>
void print_type() {
  cout << "ParamType = " << type_id_with_cvr<T>().pretty_name() << endl;
}

class Widget{};
void authenticateUser() {}

// c++11 中的返回值型别尾序语法，其好处在于可以使用函数形参,如本例中的c和i
// 对未知型别的对象采用按值传递有着诸多风险(这里指的是Index，不过这里不考虑)： 
// 非必要的复制操作带来的性能隐患、对象的截切（slicing）问题带来的异常行为。
template <typename Cont, typename Index>
auto authAndAccessCpp11(Cont& c, Index i) -> decltype(c[i]){  
  authenticateUser();
  return c[i];
}

// c++11 允许对单表达式的lambda式的返回值型别实施推导，而c++14可以支持一切lambda式和一切函数，包括多表达式。
// 根据条款2，返回值中auto会实施模板型别推导，c[i]一般会返回T&, 因此其引用性会被忽略。
// 这将导致 authAndAccess14Error 函数返回一个右值
// c++14 错误的写法
template <typename Cont, typename Index>
auto authAndAccessCpp14Error(Cont& c, Index i) { // 不正确的写法
  authenticateUser();
  return c[i];
}

// 使用decltype(auto)，auto指定了欲实施推导的型别，decltype指定使用decltype的规则来推导。
// c++14 正确的写法
template <typename Cont, typename Index>
decltype(auto) authAndAccessCpp14Correct(Cont& c, Index i) { // 正确，但需要改进
  authenticateUser();
  return c[i];
}

// 上面的形参是非常量左值引用，所以实参不能是右值容器，因为右值是不能绑定到左值引用的。
// 为了同时支持左值和右值实参，同时为了避免维护两个函数，这时可以考虑万能引用写法。
// c++14 最终的写法
template <typename Cont, typename Index>
decltype(auto) authAndAccessCpp14Final(Cont&& c, Index i) {
  authenticateUser();
  return std::forward<Cont>(c)[i]; // 参考条款25，万能引用要使用std::forward
}

// c++11 的最终写法
template <typename Cont, typename Index>
auto authAndAccessCpp11Final(Cont&& c, Index i)
-> decltype(std::forward<Cont>(c)[i]) {
  authenticateUser();
  return std::forward<Cont>(c)[i];
}

std::deque<int> makeIntDeque() {
  return std::deque<int> {6,7,8,9,10};
}

decltype(auto) f1() {
  int x = 0;
  return x; // 返回int
}

// 情况二如果和c++14中的decltype(auto) 结合使用就会有问题！！
decltype(auto) f2() {
  int x = 0;
  // warning: reference to stack memory associated 
  // with local variable 'x' returned [-Wreturn-stack-address]
  return (x); // 返回int&，糟糕，返回了局部变量的引用！
}

TEST(DeclTypeTest, autoTest) {
  std::deque<int> d{1,2,3,4,5,6};
  authAndAccessCpp11(d, 5) = 10; // ok
  //authAndAccessCpp14Error(d, 5) = 10; // error: expression is not assignable, 
                                        // 无法通过编译，因为函数返回的是一个右值
  authAndAccessCpp14Correct(d, 1) = 15;
  authAndAccessCpp11Final(d, 2) = 11; // 接收左值d
  authAndAccessCpp14Final(d, 5) = 12;
  // 接收右值, makeIntDeque产生的临时deque<int>对象会在整个赋值语句结束后析构，
  // 所以这里相当于创建了一个临时deque对象中某个元素的副本
  auto i1 = authAndAccessCpp11Final(makeIntDeque(), 2); 
  auto i2 = authAndAccessCpp14Final(makeIntDeque(), 3);
  // temp_deque[2] = 8, temp_deque[3] = 9
  cout <<"temp_deque[2] = " << i1 << ", temp_deque[3] = " << i2 << endl; 

  Widget w;
  const Widget& cw = w;
  auto myWidget1 = cw; // auto 推导， 结果为Widget
  // decltype(auto) 不限于在函数返回值的型别推导
  decltype(auto) myWidget2 = cw; // decltype 推导, 结果为 const Widget&
  print_type<decltype(myWidget1)>();  // 验证
  print_type<decltype(myWidget2)>();
  
  // 验证情况2
  int x = 0;
  print_type<decltype(x)>(); // int
  print_type<decltype((x))>(); // int&
  print_type<decltype(f1())>(); // int
  print_type<decltype(f2())>(); // int&
}
```

## 条款4 掌握查看型别推导结果的方法

1. 利用IDE编辑器、编译器错误消息和Boost.TypeIndex库查看推导结果的型别
2. 有些工具产生的结果可能会无用，或者不准确（比如`typeid(T).name()`），所以要理解c++的型别推导规则。