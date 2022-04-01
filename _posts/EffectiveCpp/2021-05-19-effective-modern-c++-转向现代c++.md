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

缺陷:   **源于大括号初始化物、std::initializer_list以及构造函数重载决议之间的纠结关系**, 如下示例所示：

```cpp
#include <gtest/gtest.h>
#include <iostream>
using namespace std;

// 1. 如果有一个或多个构造函数声明了任何一个具备std::initializer_list型别的形参，那么采用了大括号初始化语法的调用语句 
// 会强烈的（只要有可能就会）优先选用带有std::initializer_list型别形参的重载版本
class Widget{
public:
  //2. 即使是平常会执行复制或移动的构造函数也可能被带有initializer_list型别形参的构造函数劫持(MacOS clang 12.0 不成立)
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
  Widget w1(10, true); // Widget(int, bool) called
  Widget w2{10, true}; // Widget(initializer_list) called (10 和 true 被强制转换为 long double)
  Widget w3(10, 5.0);  // Widget(int, double) called
  Widget w4{10, 5.0};  // Widget(initializer_list) called (10, 5.0 被强制转换为long double)

  //即使是平常会执行复制或移动的构造函数也可能被带有initializer_list型别形参的构造函数劫持(clang 12.0 不成立)
  Widget w5(w4);             // Widget copy constructor was called
  // w4 先被强制转换成float，随后又被强制转换为long double（不成立，这里还是会调用copy constructor）
  Widget w6{w4};             // Widget copy constructor was called
  Widget w7(std::move(w4));  // Widget copy constructor was called
  // w4 先被强制转换成float，随后又被强制转换为long double（不成立，这里还是会调用copy constructor）
  Widget w8{std::move(w5)};  // Widget copy constructor was called

  // error: constant expression evaluates to 10 which cannot be narrowed to type 'bool'
  // error: type 'double' cannot be narrowed to 'bool' in initializer list
  // Widget2 w{10, 5.0}; 
  // 错误，要求窄化转换

  // 这里不会使用initalizer_list版本
  Widget3 w31{10, true}; // Widget3(int, bool) was called
  Widget3 w32{10, 5.0};  // Widget3(int, double) was called

  Widget4 w41{};     // Widget4 defalut constructor was called
  Widget4 w42({});   // Widget4 initializer_list was called
  Widget4 w43{ {} }; // Widget4 initializer_list was called
}
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

1. 0和NULL都不具备指针类型(0为int，NULL是某个整型，依赖于实现)。这导致其在指针和整型之间重载会有决议问题：

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
2. 别名模板可以让人免写 “::type” 后缀。并且在模板内部，对于内嵌的typedef的引用经常要加上typename前缀

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

// 写个函数，取用任意枚举型别的枚举量并以编译期常量形式返回其值，来简化上述用法
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

## 条款11 优先使用删除函数（delete），而非private未定义函数

1. 优先使用删除函数，而非private未定义函数
2. 任何函数都可以删除，包括非成员函数和模板具现

删除成员函数的目的一般是禁止用户使用编译器自动生成的成员函数，这里考虑复制构造和复制赋值函数，在C++98中的实现：

```cpp
template <class charT, class traits = char_traits<charT> >
class basic_ios : public ios_base {
private:
  basic_ios(const basic_ios&);
  basic_ios& operator=(const basic_ios&);
};
```

声明成private，是禁止普通用户去调用他们；而不定义，是防止**特殊用户**调用，比如成员函数，类的友元；后一种情况直到链接阶段报未定义错误，而delete都在编译阶段报错。c++11的做法：

```cpp
template <class charT, class traits = char_traits<charT> >
class basic_ios : public ios_base {
public:
  basic_ios(const basic_ios&) = delete;
  basic_ios& operator=(const basic_ios&) = delete;
};
```

删除函数一般声明为public，而非private；因为c++会先校验访问性，而后校验删除状态，因此可以在编译阶段得到更准确地错误信息。

任何函数都能删除，如下面为了防止char，bool，double实参隐式转化为int而调用`isLucky(int)`，将对应的版本删除：

```cpp
#include <gtest/gtest.h>
#include <iostream>
using namespace std;

bool isLucky(int number) { return true; }
bool isLucky(char) = delete;
bool isLucky(bool) = delete;
bool isLucky(double) = delete;// 拒绝double 和 float，因为float相对于转化为int会优先转化为double

TEST(DeleteTest, delete) {
  isLucky(0);
  isLucky('a'); // error: call to deleted function 'isLucky'
  isLucky(false); // error: call to deleted function 'isLucky'
  isLucky(0.0f); // error: call to deleted function 'isLucky'
}
```

尽管删除函数不可被调用，但它们还是程序的一部分。因此，它们在重载决议时还是会纳入考量。

删除函数模板某个具现版本：

```cpp
#include <gtest/gtest.h>
#include <iostream>
using namespace std;

template <typename T>
void processPointer(T* ptr) {
  cout << "processPointer" << endl;
}

// 删除 void* 对应的具现版本
template <>
void processPointer<void>(void*) = delete;

TEST(DeleteTest, template) {
  int i = 0;
  void* p = (void*)&i;
  processPointer(&i);
  processPointer(p);  // error: call to deleted function 'processPointer'
}

// 对于类内部的函数模板，也无法用private方式来删除，因为模板的特化是必须在名字空间作用域而非类作用域内撰写的。
// 并且特化必须与主模板是同一个访问层级。
class Widget {
public:
  template<typename T>
  void processPointer(T* ptr){}
private:
  template<>
  void processPointer<void>(void*); // 错误的写法
};

template<>
void Widget::processPointer<void>(void*) = delete; // 正确的写法
```

## 条款12 为意在改写的函数，添加override声明

原因：虚函数改写很容易出错，改写的发生要满足以下所有条件：

1. 基类中的函数**必须是虚函数**

2. 基类和派生类中的**函数名字**必须完全相同（析构函数例外）

3. 基类和派生类中的**函数形参型别**必须完全相同

4. 基类和派生类中的**函数常量性（constness）**必须完全相同

5. 基类和派生类中的**函数返回值和异常规格**必须兼容

6. 基类和派生类中的**函数引用饰词（reference qualifier）**必须完全相同

   函数引用饰词是为了实现限制 (区分) 成员函数仅用于左值或右值（\*this）。带有引用饰词的成员函数，不必是虚函数。

   ```cpp
   #include <gtest/gtest.h>
   #include <iostream>
   #include <vector>
   using namespace std;
   
   class Widget {
    public:
     using DataType = std::vector<double>;
     DataType& data() & {  // 对于左值Widget型别，返回左值
       cout << "left value reference" << endl;
       return values;
     }
   
     DataType data() && {  // 对于右值Widget型别，返回右值
       cout << "right value reference" << endl;
       return std::move(values);
     }
   
    private:
     DataType values;
   };
   
   Widget makeWidget() { return Widget{}; }
   
   TEST(DeleteTest, template) {
     Widget w;
     auto vals1 = w.data();             // 调用左值重载版本，vals1采用赋值构造完成初始化
     auto vals2 = makeWidget().data();  // 调用右值重载版本，vals2采用移动构造完成初始化
   }
   ```

C++ 提供overrid来显示标明派生类中的函数是为了改写基类版本，编译器会检查上述所有条件是否满足。

派生类中的override声明，可以辅助你判断更改基类中的虚函数的签名时所带来的的影响。

override 和 final 是c++11 增加的两个**语境关键字**，即仅在特定语境下保留作为关键字，其他情况可以正常作为实体名称：

1. override 仅当出现在成员函数声明的末尾时才有保留关键字意义
2. final 用于虚函数，会阻止他在派生类中被改写。final 用于类，该类被禁止用作基类。

## 条款13 优先使用const_iterator，而非iterator

1. 优先使用const_iterator，而非iterator

2. 在最通用的代码中，优先使用非成员函数版本的begin、end、cbegin和rbegin等，而非其成员函数版本。

   因为必须考虑到某些容器或类似容器的数据结构没有成员函数版本的begin和end（cbegin, cend, rbegin, rend 等），而是以非成员函数版本提供begin和end，如内建数组。

```cpp
// 如果是c++11，需要额外定义cbegin版本，因为c++11只提供非成员函数版本的begin() 和 end()。C++14提供了所有，包括cbegin，cend等。
template <class C>
auto cbegin(const C& container) -> decltype(std::begin(container)) {
  return std::begin(container);  // 不使用成员函数版本，使得在C是一个内建数组或没有成员函数版本的cbegin()时也适用。
                                 // 另外, std::begin传入一个const容器会产生一个const_iterator，因此形参带const
}

template <typename C, typename v>
void findAndInsert(C& container, const V& targetVal, const V& insertVal) {
  using std::cbegin;
  using std::cend;
  auto it = std::find(cbegin(container), cend(container), targetVal);
  container.insert(it, insertVal);
}
```

## 条款14 只要函数不会发射异常，就为其加上noexcept声明

1. noexcept  声明是函数接口的组成部分，这意味着调用方可能会对它有依赖

2. 相对于不带 noexcept 声明的函数，带有 noexcept 声明的函数有更多的机会得到优化

   ```cpp
   int f(int x) noexcept; // f 不会发射异常，c++11 风格; 最优化
   int f(int x) throw();  // f 不会发射异常，c++98 风格；优化不够
   ```

   如果在运行期间，一个异常逸出f的作用域，则f的异常规格被违反。在c++98异常规格下，调用栈会开解至f的调用方，然后执行一些与本条款无关的动作后，程序中止。在c++11异常规格下，程序执行中止前，栈只是**可能**会开解。这个微小的区别可能导致代码生成上的巨大差异

3. noexcept 性质对于移动操作、swap、内存释放函数和析构函数最有价值。

   1. 关于移动，以vector的push_back举例

      当需要扩充底层内存时，c++98的push_back会复制旧vector中的元素到新vector，如果在复制某个元素时发生了异常，会保持旧vector不变。只有在所有元素都成功复制到新vector后，旧vector才会析构。这使得c++98的push_back提供强异常安全保证。c++11中可以使用移动来替代复制以提升性能，但是为了不破坏这种强异常安全保证（如果移动某个元素发生异常，旧vector无法恢复），只有在已知移动操作不会发生异常的情况下，才使用移动优化，否则使用复制。而这就可以通过校验移动函数是否带有noexcept声明来达成。

   2. 标准库为数组和std::pair准备的swap函数

      ```cpp
      template <class T, size_t N>
      void swap(T (&a)[N], T(&b)[N]) noexcept(noexcept(swap(*a, *b)));
      
      template <class T1, class T2>
      struct pair {
        void swap(pair& p) noexcept(noexcept(swap(first, p.first)) && noexcept(noexcept(swap(second, p.second))))
      }  
      ```

      即高阶数据结构的swap行为要具备noexcept性质，一般的，仅当构建他的低阶数据结构具备noexcept性质时才成立。

   3. c++11中，内存释放函数和所有的析构函数默认带有noexcept性质，因此不再需要显示声明。如果某个类的数据成员，其型别的析构函数显示声明为noexcept(false), 则该类的析构函数不具备noexcept，如果成员对象的析构抛出了异常，这种行为未定义。

4. 大多数函数都是异常中立的，不具备 noexcept 性质。

   此类函数自身不抛出异常，但其调用的函数有可能抛出异常，此时要允许异常传递至调用栈的更深一层，因此不具备noexcept（否则，程序会中止）。

## 条款15 只要有可能使用constexpr，就使用它

constexpr既可以修饰对象，也可以修饰函数。

1. constexpr对象都具备const属性且在编译阶段已知，并由编译期已知的值完成初始化。

   编译阶段已知的值的优势：

   1. 可以放在只读内存里
   2. 编译阶段已知的常量整型值可以用于C++要求整型常量表达式的语境中，包含：
      1. 数组大小
      2. 整型模板实参
      3. 枚举量的值
      4. 对齐规格等

   const值不要求使用编译期已知值来初始化。

2. 理解constexpr函数：

   1. constexpr函数可以用在要求编译期常量的语境中。在这样的语境中，若传给constexpr函数的实参是编译期已知的，则结果也会在编译期间计算出来；如果任何一个实参未知，则无法通过编译
   2. 调用constexpr函数时若传入的值有一个或多个在编译期未知，则它的运行方式与普通函数无异，即他是运行期执行结果的计算。这意味着，如果函数的功能同时用于编译期和运行期语境，则可以只声明一个constexpr函数，而不需要维护两个函数
   3. constexpr函数实现的限制：
      1. c++11的限制：
         1. 不得包含多于一个可执行语句，即一条return语句：可以使用`?:`运算符代替`if-else`, 使用递归代替循环
         2. constexpr函数仅限于传入和返回字面型别（literal type），意思是这样的型别能够持有编译期可以决议的值。这包含除void之外所有内建型别，以及带有constexpr声明的构造函数的用户自定义型别。
         3. constexpr成员函数隐式声明为const，这使得设置函数无法声明为constexpr，因为其改变了对象
      2. c++14 去除了上面c++11的所有限制。

   使用示例如下：

   ```cpp
   #include <gtest/gtest.h>
   #include <array>
   #include <string>
   using namespace std;
   
   constexpr int pow11(int base, int exp) noexcept { return (exp == 0 ? 1 : base * pow11(base, exp - 1)); } // c++11
   constexpr int pow14(int base, int exp) noexcept { // c++14
     auto result = 1;
     for (int i = 0; i < exp; ++i) result *= base;
     return result;
   }
   
   constexpr auto numConds = 5;
   
   int readFromDB(string name) {
     if (name == "base") {
       return 2;
     } else {
       return 5;
     }
   }
   
   class Point {
    public:
     // constexpr 构造函数和成员函数的使用与普通 constexpr 函数无异。如果传入编译期已知值,
     // 且这使得对象的所有数据成员都是编译期已知，那么这样初始化出来的对象具备 constexpr 属性，即编译期可知。
     constexpr Point(double xVal = 0, double yVal = 0) noexcept : x(xVal), y(yVal) {}
     constexpr double xValue() const noexcept { return x; }
     constexpr double yValue() const noexcept { return y; }
     constexpr void setX(double newX) noexcept { x = newX; }  // C++14
     constexpr void setY(double newY) noexcept { y = newY; }  // C++14
   
    private:
     double x, y;
   };
   
   constexpr Point midpoint(const Point& p1, const Point& p2) noexcept {
     return {(p1.xValue() + p2.xValue()) / 2, (p1.yValue() + p2.yValue()) / 2};
   }
   
   // C++14
   constexpr Point reflection(const Point& p) noexcept {
     Point result;
     result.setX(-p.xValue());
     result.setY(-p.yValue());
     return result;
   }
   
   TEST(ContstexprTest, compileAndRun) {
     std::array<int, pow11(3, numConds)> result;  // 编译期使用pow
     auto base = readFromDB("base");              // 在执行期取得
     auto exp = readFromDB("exponent");
     auto baseToExp = pow11(base, exp);  // 运行期使用pow
     constexpr Point p1(9.1, 28.8);      // 编译期执行
     constexpr Point p2(10.6, 33.5);
     constexpr auto mid = midpoint(p1, p2);          // 编译期运行
     constexpr auto reflectedMid = reflection(mid);  // 编译期执行,c++14
   }
   ```

比起非constexpr对象或函数而言，constexpr对象或函数可以用在一个作用域更广的语境中。

## 条款16 保证const成员函数的线程安全性

const 成员函数虽声明其对象不可改变，但是有些声明为mutable的成员仍然可以改变，需要注意并发调用的安全性。

对于单个要求同步的变量或内存区域，使用std::atomic 比互斥量性能好。

但是如果有两个或多个变量或内存区域需要作为一个单位进行操作时，就要动用互斥量了。

```cpp
#include <atomic>
#include <cmath>
#include <mutex>
using namespace std;
class Point {
 public:
  double distanceFromOrigin() const noexcept {
    ++callCount;
    return std::sqrt((x * x) + (y * y));
  }

 private:
  mutable std::atomic<unsigned> callCount{0};  // std::atomic 是只移型别
  double x, y;
};

double expensiveComputation1() { return 0.0; }
double expensiveComputation2() { return 1.1; }

class Widget {
 public:
  int magicValue() const {
    std::lock_guard<std::mutex> guard(m);
    if (cacheValid)
      return cachedValue;
    else {
      auto val1 = expensiveComputation1();
      auto val2 = expensiveComputation2();
      cachedValue = val1 + val2;
      cacheValid = true;
      return cachedValue;
    }
  }

 private:
  mutable std::mutex m;  // std::mutex 只移型别
  mutable int cachedValue;
  mutable bool cacheValid{false};
};
```

## 条款17 理解特种成员函数的生成机制

1. c++98 中的4个特种成员函数：默认构造函数，析构函数，复制构造函数，复制赋值函数。

   这些函数仅在需要的时候合成，即程序使用到且未显示声明。生成的函数具有public访问层级和inline属性，且他们都是非虚的（除非是派生类中的析构函数，且在基类中是虚析构函数）

2. c++11中加入了两个新成员：移动构造函数和移动赋值运算符。

   ```cpp
   class Widget {
   public:
   	Widget(Widget&& rhs);
   	Widget& operate=(Widget&& rhs);
   };
   ```

   移动操作也仅在需要的时候合成，执行的也是非静态成员的“按成员移动操作”，包含基类成员。按成员移动只是一个请求，如果对应的型别（数据成员或基类类型）不可移动，会退化为复制操作。核心在于把std::move 用于每个移动源对象，使用其返回值型别进行重载决议，以决定是移动还是复制。因此，按成员移动 = 可移动成员的移动 + 不可移动成员的复制。

3. 两种复制操作（复制构造/赋值）是彼此独立的：声明了其中一个并不会阻止编译器生成另一个。

   两种移动操作不彼此独立：声明了其中一个就会阻止编译器生成另一个。因为编译器认为如果其中一个需要自定义，那么另一个也需要。并且，如果显示声明了复制操作，就会阻止编译器生成移动操作。因为编译器认为如果复制需要自定义，那么一般移动也需要。反之亦然，如果显示声明了移动操作，就会删除复制操作

4. 大三律：如果你声明了复制构造函数、复制赋值运算符或析构函数中的任何一个，你就得声明所有这三个。它根植于这样的思想：如果有改写复制操作的需求，往往意味着该类需要执行某种资源管理。而如果在其中一个需要这种资源管理，那么三个都需要。比如内存。

   1. 大三律推论一：如果显示声明了析构函数，则平凡的按成员复制也不适用于该类（由于会对遗留代码产生影响，因此c++11仍然不会阻止其他两个的生成）
   2. 大三律推论二：由于显示声明复制会阻止生成移动操作，因此只要用户声明了析构函数，就不会生成移动操作。

5. 因此，移动操作的生成要满足以下所有条件：

   1. 该类未声明任何复制操作
   2. 该类未声明任何移动操作
   3. 该类未声明任何析构函数

   这样的机制后面也会应用到复制操作，因为c++11标准规定，在已经存在复制或析构函数的条件下，仍然自动生成复制操作已经成为了被废弃的行为。

6. 如果编译器生成的这些函数有正确的行为，可以使用 `=default` 来显示表达这个想法：

   ```cpp
   class Widget {
   public:
   	~Widget();
   	Widget(const Widget&) = default;
   	Widget& operator=(const Widget&) = default;
   };
   ```

   由于前述的一些机制，导致某些函数的生成被阻止，如果仍然需要编译器生成的版本，也可以使用`=default`来实现：

   ```cpp
   class Base {
   public:
     virtual ~Base() = default; // 析构函数的显示声明会抑制移动操作的生成，因此如果需要的话，使用default
     // 移动操作
     Base(Base&&) = default;
     Base& operator=(Base&&) = default;
     // 复制操作
     Base(const Base&) = default;
     Base& operator=(const Base&) = default;
   };
   ```

   显示使用default来达到生成编译器版本的方式，还可以避免一些陷进，比如因为增加了显示声明的析构函数而导致阻止了移动操作的合成，使得对象的移动退化为复制，从而导致性能问题。这时如果显示使用default来声明移动操作，就完全没有这种问题。

7. 成员函数模板在任何情况下都不会抑制特种成员函数的生成：

   ```cpp
   class Widget{
   public:
   	template<typename T>
   	Widget(const T& rhs);// 即使T为Widget，使得模板具现了复制构造函数的签名版本的函数，也不会阻止编译器生成特种成员函数。
   	
   	template<typename T>
   	Widget& operator=(const T& rhs);
   };
   ```

   





