---
title: effective modern c++ 之 转向现在c++
descriptions: 高效使用 c++11 特性
categories: language
tags: c++11
---

# 三、转向现代c++

## 条款7 在创建对象时注意区分()和{}

大括号初始化的优点:      

 **大括号初始化可以应用的语境最为宽泛，可以阻止隐式窄化型别转换，还对最令人苦恼之解析语法免疫**

缺陷:   
**源于大括号初始化物、std::initializer_list以及构造函数重载决议之间的纠结关系**, 如下示例所示：

```cpp
#include <gtest/gtest.h>
#include <iostream>
using namespace std;

// 1. 如果有一个或多个构造函数声明了任何一个具备std::initializer_list型别的形参，那么采用了大括号初始化语法的调用语句 
// 会强烈的（只要有可能就会）优先选用带有std::initializer_list型别形参的重载版本
class Widget{
public:
  //2. 即使是平常会执行复制或移动的构造函数也可能被带有initializer_list型别形参的构造函数劫持(clang 12.0 不成立)
  Widget(const Widget& w){
    cout << "Widget copy constructor was called" << endl;
  }

  Widget(int i, bool b) {
    cout << "Widget(int, bool) called" << endl;
  }

  Widget(int i, double d) {
    cout << "Widget(int, double) called" << endl;
  }

  Widget(initializer_list<long double> il) {
    cout << "Widget(initializer_list) called" << endl;
  }

  operator float() const{
    cout << "Widget float() was called" << endl;
    return 0.0;
  }
};

// 3. 即使最优选的带有initialzer_list型别形参的构造函数无法被调用时，这种决心还是会占上风(成立)
class Widget2 {
 public:
  Widget2(int i, bool b){
    cout << "Widget2(int,boo) was called" << endl;
  }

  Widget2(int i, double d) {
    cout << "Widget2(int, double) was called" << endl;
  }

  Widget2(std::initializer_list<bool> il){
    cout << "Widget2(initialzer_list<bool>) was called" << endl;
  }
};

// 4. 只有在找不到任何办法将大括号初始化物中的实参转换成std::initializer_list模板中的型别时，
// 编译器才会退而去检查普通的重载决议
class Widget3 {
 public:
  Widget3(int i, bool b){
    cout << "Widget3(int, bool) was called" << endl;
  }

  Widget3(int i, double d) {
    cout << "Widget3(int, double) was called" << endl;
  }


  Widget3(std::initializer_list<string> il){
    cout << "Widget3(initialzer_list<string>) was called" << endl;
  }
};

// 5. 使用一对空大括号构造对象，而类有默认构造函数和initializer_list型别形参的构造函数时，会选择默认构造函数
class Widget4 {
 public:
  Widget4() {
    cout << "Widget4 defalut constructor was called" << endl;
  }

  Widget4(std::initializer_list<int> il) {
    cout << "Widget4 initializer_list was called" << endl;
  }
};


// c++规定，任何能够解析为声明的都要解析为声明，这会带来副作用。
// 最令人苦恼的语法解析指的是，程序员本来想要以默认方式构造一个对象，结果却一不小心声明了一个函数
Widget w2();

TEST(DeclTypeTest, autoTest) {
  Widget w1(10, true);
  Widget w2{10, true}; // 10 和 true 被强制转换为 long double
  Widget w3(10, 5.0);
  Widget w4{10, 5.0}; // 10, 5.0 被强制转换为long double

  //即使是平常会执行复制或移动的构造函数也可能被带有initializer_list型别形参的构造函数劫持(clang 12.0 不成立)
  Widget w5(w4);
  // w4 先被强制转换成float，随后又被强制转换为long double（不成立，这里还是会调用copy constructor）
  Widget w6{w4}; 
  Widget w7(std::move(w4)); 
  // w4 先被强制转换成float，随后又被强制转换为long double（不成立，这里还是会调用copy constructor）
  Widget w8{std::move(w5)}; 

  // error: constant expression evaluates to 10 which cannot be narrowed to type 'bool'
  // error: type 'double' cannot be narrowed to 'bool' in initializer list
  // Widget2 w{10, 5.0}; 
  // 错误，要求窄化转换

  // 这里不会使用initalizer_list版本
  Widget3 w31{10, true};
  Widget3 w32{10, 5.0};

  Widget4 w41{};   // default constructor
  Widget4 w42({}); // initializer_list constructor
  Widget4 w43{{}}; // initializer_list constructor
}
```

结果：

```shell
Widget(int, bool) called
Widget(initializer_list) called
Widget(int, double) called
Widget(initializer_list) called

Widget copy constructor was called
Widget copy constructor was called
Widget copy constructor was called
Widget copy constructor was called

Widget3(int, bool) was called
Widget3(int, double) was called

Widget4 defalut constructor was called
Widget4 initializer_list was called
Widget4 initializer_list was called
```

结论：

1. 最好把构造函数设计成客户无论使用小括号还是大括号都不会影响调用的重载版本才好
2. initializer_list型别形参的构造函数可能导致其他重载版本连露脸的机会都没有
3. 选择大括号或者小括号风格，并坚持下去（我选择大括号）

在模板内进行对象创建时，到底应该使用小括号，还是大括号，会成为一个棘手问题。

```cpp
template<typename T, typename... Ts>
void doSomeWork(Ts&&... params) {
  // 使用小括号还是大括号，应该由用户决定，所以这里哪一个都不是很好
  T localObject1(std::forward<Ts>(params)...); // 采用小括号
  T localObject2{std::forward<Ts>(params)...}; // 采用大括号
  for(auto v: localObject1) {
    cout << v << ", "; // 20, 20, 20, 20, 20, 20, 20, 20, 20, 20, 
  }
  cout << endl;
  for (auto v: localObject2) {
    cout << v << ", "; // 10, 20
  }
  cout << endl;
}

TEST(TemplateTest, template) {
  std::vector<int> v;
  doSomeWork<std::vector<int>>(10, 20);
}
```

## 条款8 优先使用nullptr,  而非0或NULL

nullptr的型别是std::nullptr_t，而非整型，它可以隐式转化到所有裸指针型别。

nullptr可以提升代码的清晰性，明确表明是指针类型。优先使用nullptr的两点理由：

1. 0和NULL都不具备指针类型(0为int，NULL是某个整型，依赖于实现)。这导致其在指针和整型之前重载会有决议问题：

```cpp
#include <gtest/gtest.h>
#include <iostream>
using namespace std;
// 建议：不要在整型和指针之间重载
void foo(int) { cout << "foo(int)" <<endl; }
void foo(bool) {  cout << "foo(bool)" << endl;}
void foo(void*) {  cout << "foo(void*)" << endl;}
TEST(NullTest, foo) {
  foo(0); // foo(int)
  foo(nullptr); // foo(void*)
  // foo(NULL); // macos clang 12.0: call to 'foo' is ambiguous, 3个都可以
}
```

2. 模板型别推导会将0推导成int，将NULL推导为某种整型(MacOS 为 long)，所以在模板定义里使用nullptr表示指针才是正确的:

```cpp
#include <gtest/gtest.h>
#include <iostream>
#include <memory>
#include <mutex>
using namespace std;

class Widget {
 public:
  Widget() {}
};

int f1(std::shared_ptr<Widget> spw) { return 0; }
double f2(std::unique_ptr<Widget> upw) { return 0.0; }
bool f3(Widget* pw) {
  cout << "hello nullptr" << endl;
  return false;
}

using MuxGuard = std::lock_guard<std::mutex>;

template <typename FuncType, typename MuxType, typename PtrType>
decltype(auto) lockAndCall(FuncType func, MuxType& mutex, PtrType ptr) {
  MuxGuard g(mutex);
  return func(ptr);
}

std::mutex f1m, f2m, f3m;
TEST(TypeIndex, nullptr) {
  // error: no viable conversion from 'int' to 'std::__1::shared_ptr<Widget>'
  // lockAndCall(f1, f1m, 0);
  // error: no viable conversion from 'long' to 'std::__1::unique_ptr<Widget, std::__1::default_delete<Widget> >'
  // lockAndCall(f2, f2m, NULL);
  lockAndCall(f3, f3m, nullptr);
}
```

## 条款9 优先使用别名声明，而非 typedef

1. 别名声明可以模板化，称为别名模板，typedef不支持
2. 别名模板可以让人免写 “::type” 后缀，并且在模板内部，对于内嵌的typede的引用经常要加上typename前缀

参考如下示例代码来看别名声明和typedef的使用差异

```cpp
#include <gtest/gtest.h>
#include <iostream>
#include <list>
using namespace std;
class MyAlloc {};

// 别名声明
template <typename T>
using MyAllocList = std::list<T, MyAlloc<T>>;

MyAllocList<int> li;

template <typename T>
class Widget {
  MyAllocList<T> list; //  非依赖型别
};

// typedef
template <typename T>
struct MyAllocList2 {
  typedef std::list<T, MyAlloc<T>> type;
};

MyAllocList2<int>::type li2;

template <typename T>
class Widget2 {
  // typedef 定义的是带依赖型别，其前面要有typename，因为编译器不确定type是一个类型，比如在某个MyAllocList2的特化中并不是一个类型
  typename MyAllocList2<T>::type li2;
}

// c++11 中自定义别名模板以使用c++14型别特征库，因为c++11型别特征库是使用typedef实现的
template <T>
using remove_const_t = typename remove_const<T>::type;
```

## 条款10 优先选用限定作用域的枚举型别，而非不限作用域的枚举型别

为什么c++98中的枚举型别称为**不限作用域的枚举型别**？

因为c++中有一个通用规则， 如果在一对大括号里声明一个名字，则该名字的可见性被限定在括号括起来的作用域内。

但是c++98中的枚举型别定义中的枚举量名字会泄漏到枚举型别所在作用域：

```cpp
enum Color{black, white, red};
//这里可以直接访问black,white,red
...
```

不限作用域的枚举型别有以下特点：

1. 可以隐式转换到整数型别。
2. 其没有默认底层型别，因此默认不支持前置声明，但支持指定底层型别，指定后支持前置声明。（前置声明的好处是可以降低编译依赖性。）
3. 默认情况下，编译器通常为不限作用域的的枚举型别选用足够表示其枚举量的最小底层类型。

而在c++11中的限定作用域的枚举型别：

```cpp
enum class Color {black, white, red};
Color::white;  // 需要加类型限定
```

他有以下几个优点：

1. 可以降低名字空间污染
2. 他是强类型，不存在隐式转换到任何请他类型的途径。
3. 其底层类型默认为int，默认支持前置声明和底层型别指定。

```cpp
enum Color: std::uint8_t; // 前置声明，底层型别 uint8_t
enum class Status; // 前置声明, 底层型别是int
enum class Status: std::uint32_t; // 前置声明，指定底层类型为 uint32_t
```

不限作用域的枚举型别还是有用的，比如：引用c++11中的std::tuple型别的各个域时。这种情况下，两种枚举型别的不同优化做法如下：

```cpp
#include <gtest/gtest.h>
#include <iostream>
#include <utility>
#include <string>
using namespace std;

using UserInfo = std::tuple<string, string, size_t>;
UserInfo uInfo;
auto val = = std::get<1>(uInfo); // 不使用枚举型别，不知道1代表的是哪个域

// 使用不限作用域的枚举型别优化
enum UserInfoFields {uiName, uiEmail, uiReputation};
UserInfo uInfo;
auto val = std::get<uiEmail>(uInfo);

// 使用限定作用域的优化
enum class UserInfoFields {uiName, uiEmail, uiReputation};
UserInfo uInfo;
auto val = std::get<static_cast<std::size_t>(UserInfoFields::uiEmail)>(uInfo);

// 上面的写法太过繁琐，使用函数来获取枚举量代表的值
// 写个函数，取用任意枚举型别的枚举量并以编译期常量形式返回其值
template<typename E>
constexpr typename std::underlying_type<E>::type // constexpr 可以在编译期计算出值
toUType(E enumerator) noexcept { // noexcept 表示不会抛出异常
  return static_cast<typename std::underlying_type<E>::type>(enumerator);
}

// c++14 的优化写法
template <typename E>
constexpr auto toUType(E enumerator) noexcept {
  return static_cast<std::underlying_type_t<E>>(enumerator);
}

// toUType 函数返回值作为get模板实参，其必须是编译期常量
auto val = std::get<toUType(UserInfoFields::uiEmail)>(uInfo);
```

建议使用第二种，虽然要付出点代价，但可以得到限定枚举型别的好处。

## 条款11 优先使用删除函数，而非private未定义函数

## 条款12 为意在改写的函数，添加override声明

## 条款13 优先使用const_iterator，而非iterator

## 条款14 只要函数不会抛出异常，就为其加上noexcept声明

## 条款15 只要有可能使用constexpr，就是用它

## 条款16 保证const成员函数的安全性

## 条款17 理解特种成员函数的生成机制

