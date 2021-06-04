---
title: effective modern c++ 之 右值引用、移动语义、完美转发
descriptions: 高效使用 c++11 特性
categories: language
tags: c++11
---

# 条款23 理解std::move 和std::forward

move用于为移动操作做铺垫，forward用于参数转发。

1. std::move 实施的是无条件的向右值型别的强制型别转换。他本身不会执行移动操作。大致的实现如下：

   ```cpp
   // c++14
   template <typename T>
   decltype(auto) move(T&& param){
     // 由于是万能引用模板，T有可能推断为T&，因此这里需要使用remove_reference_t保证返回的是右值引用（参考条款1）
     using ReturnType = std::remove_reference_t<T>&&;
     return static_cast<ReturnType>(param);
   }
   ```

   1. 如果想取得对某个对象执行移动操作的能力，则不要将其声明为常量，因为针对常量对象的移动操作将一声不吭的转化为复制操作。
   2. std::move 不仅不实际移动任何东西，甚至不保证经过其强制型别转换后的对象具备可移动的能力，唯一可保证的是move的结果是一个右值。

   参考如下示例：

   ```cpp
   #include <gtest/gtest.h>
   #include <iostream>
   using namespace std;
   
   class MyString {
    public:
     MyString() : pstr(nullptr), len(0) {}
     // 指涉到常量的左值引用允许绑定到一个常量右值型别的实参，因此它可以接收const MyString 右值。
     // (常量左值引用可以绑定到右值，右值引用，常量右值引用)
     MyString(const MyString& rhs) { cout << "MyString copy constructor" << endl; }
     MyString(MyString&& rhs) { cout << "MyString move constructor" << endl; }
   
    private:
     char* pstr;
     int len;
   };
   
   class Annotation {
    public:
     // std::move 将const MyString型别实参转化为 const MyString 型别的右值，它会选择copy constructor
     explicit Annotation(const MyString text) : value(std::move(text)) {}
   
    private:
     MyString value;
   };
   
   TEST(SharedPtrTest, enableSharedFromThis) {
     MyString mstr;
     Annotation a1(mstr);  // MyString copy construtor
   }
   ```

2. 仅当传入的实参被绑定到右值时，std::forward才针对该实参实施向右值型别的强制型别转换，除此外，它不会执行任何操作。

   std::forward是如何知晓其实参是否通过右值完成初始化？该信息是被编码到模板形参T中的，在传递给forward后，将这些信息解码出来，参考条款28.

   ```cpp
   #include <gtest/gtest.h>
   
   #include <iostream>
   using namespace std;
   
   void process(const int& lv) { cout << "handle left value " << lv << endl; }
   
   void process(int&& rv) { cout << "handle right value " << rv << endl; }
   
   template <typename T>
   void logAndProcess(T&& param) {
     cout << "before handle" << endl;
     process(std::forward<T>(param));
   }
   
   TEST(ForwardTest, forward) {
     int v = 10;
     logAndProcess(v); // handle left value 10
     logAndProcess(std::move(v)); // handle right value 10
   }
   ```

# 条款24 区分万能引用和右值引用

`T&&`有两种含义，一是表示右值引用，仅能绑定到右值，主要用于识别出可移对象；二是表示万能引用，既可以表示右值引用，也可以表示左值引用。

1.  如果函数模板形参精确的具备`T&&`型别，并且T的型别是推导而来，或如果对象使用`auto&&`声明其型别，则该形参或对象就是个万能引用
2. 如果型别声明并不精确的具备`T&&`的形式、或者型别推导并未发生，则`T&&`就代表右值引用
3. 采用右值初始化万能引用，就会得到一个右值引用。采用左值初始化万能引用，就会得到一个左值引用

参考下面的示例：

```cpp
#include <gtest/gtest.h>

void f1(int&& param) {}  // 右值引用, 不涉及型别推导

template <typename T>
void f2(std::vector<T>&& param) {}  // 右值引用，不精确具备T&&

template <typename T>
void f3(const T&& param) {}  // 右值引用，不精确具备T&&

template <typename T>
void f4(T&& param) {}  // 万能引用，型别推导 + 精确T&&

template <typename T>
class MyVector {
 public:
  void push_back(T&&);  // T 由MyVector<T>决定，因此push_back不涉及型别推导，这里是右值引用
  
  template <typename... Args>
  void emplace_back(Args&&... args);  // 万能引用
};

TEST(ForwardTest, forward) {
  int&& rv = 1;     // 右值引用
  auto&& rv2 = rv;  // 万能引用
  std::vector<int> vi;
  // no known conversion from 'std::vector<int>' to 'std::vector<int> &&' for 1st argumen
  f2(vi);  // 错误，不能给一个右值引用绑定一个左值
}
```

本条款是一个抽象，底层的真相是“引用折叠”，参考条款28

# 条款25 针对右值引用实施std::move, 针对万能引用实施std::forward

1. 当转发右值引用给其他函数时，应当对其实施向右值的无条件强制型别转换（std::move）,因为他们一定绑定到右值（可移动）。

   而转发万能引用时，应当对其实施向右值的有条件转换（std::forward）,因为他们不一定绑定到右值。

2. 依据左值和右值进行重载的用法相比万能引用，会导致代码膨胀、与习惯用法背离、执行期效率折损等问题，最重要的是这种设计可扩展性太差。

3. 在按值返回的函数中，如果返回的是绑定到一个右值引用或一个万能引用的对象，则当你返回该引用时，应该对其实施std::move或者std::forward.

   ```cpp
   Matrix operator+(Matrix&& lhs, const Matrix& rhs) {
     lhs += rhs;
     return lhs; // lhs是个左值这一事实对强波编译器将其复制入返回值存储位置
   }
   
   Matrix operator+(Matrix&& lhs, const Matrix& rhs) {
     lhs += rhs;
     return std::move(lhs); // 正确的用法，如果Matrix不支持移动，会自动使用复制
   }
   
   template<typename T>
   T reduceAndCopy(T&& frac) {
     frac.reduce();
     // 对于右值，是移入返回值；对于左值，复制入返回值。如果没有std::forward，由于frac本身是左值，会强制复制入返回值
     return std::forward<T>(frac);
   }
   ```

4. 若局部对象可能适用于返回值优化，则请勿针对其实施std::move或std::forward。原因：

   1. 编译器若要在一个按值返回的函数里省略对局部对象的复制或者移动（RVO），需要满足两个前提条件

      1. 局部对象型别和返回值型别相同
      2. 返回的就是局部对象本身

      返回一个局部对象的引用(std::move的结果) 并不满足实施RVO的前提条件，因此会抑制RVO

   2. 即使实施RVO的前提条件满足，但编译器选择不执行复制省略的时候，返回对象必须作为右值处理。因此，如果RVO条件满足，要么复制省略，要么std::move被隐式的实施与返回的局部对象。因此没有必要手动实施std::move

   ```cpp
   Widget makeWidget() {
   	Widget w;
   	return w;// 不应该使用std::move(w);
   }
   ```

# 条款26 避免依万能引用型别进行重载

1. 把万能引用作为重载候选型别，几乎总会让该重载版本在始料未及的情况下被调用到
2. 完美转发构造函数的问题尤其严重，因为对于非常量的左值型别而言，他们一般都会形成相对于复制构造函数的更加匹配，并且他们还会劫持派生类中对基类的复制和移动构造函数的调用。

引子，万能引用提升了效率：

```cpp
#include <gtest/gtest.h>
#include <chrono>
#include <ctime>
#include <iomanip>
#include <iostream>
#include <set>
#include <string>
using namespace std;

multiset<string> names;
vector<string> backNames{"a", "b", "c"};

void log(chrono::time_point<std::chrono::system_clock> now, string logStr) {
  auto t_c = chrono::system_clock::to_time_t(now);
  cout << logStr << put_time(localtime(&t_c), " %c, %Z") << endl;
}

void logAndAdd(const string& name) {
  auto now = chrono::system_clock::now();
  log(now, "logAndAdd");
  names.emplace(name);  // name 自身是左值，所以都是复制进入names，可以用万能引用优化
}

// 使用万能引用优化执行效率
template <typename T>
void logAndAdd2(T&& name) {
  auto now = chrono::system_clock::now();
  log(now, "logAndAdd2");
  names.emplace(std::forward<T>(name));
}

string nameFromIndex(int idx) { return backNames[idx]; }
void logAndAdd2(int idx) {
  auto now = chrono::system_clock::now();
  log(now, "logAndAdd2");
  names.emplace(nameFromIndex(idx));
}

TEST(ForwardTest, forward) {
  string petName("Darla");
  logAndAdd(petName);        // 传递左值，复制入names，无法优化
  logAndAdd(string("Dog"));  // 传递右值，可用万能引用优化为移动
  logAndAdd("Pig");  // 传递字面量，会构造string临时对象，再复制进入names。可用万能引用优化，直接在names中构造
  logAndAdd2(petName);        // 复制
  logAndAdd2(string("Dog"));  // 移动
  logAndAdd2("Pig");          // 直接构造
  logAndAdd2(0);
  // short idx = 1;
  // logAndAdd2(idx); // 编译失败，没有string(short)  构造函数
}
```

示例中编译失败的原因：形参为万能引用的函数，是C++中最贪婪的。他们会在具现过程中，和几乎任何实参型别都会产生精确匹配。

构造函数的示例：

```cpp
#include <gtest/gtest.h>
#include <string>
using namespace std;

vector<string> backNames{"a", "b", "c"};
string nameFromIdx(int idx) { return backNames[idx]; }

class Person {
 public:
  template <typename T>
  explicit Person(T&& n) : name(std::forward<T>(n)) {}
  explicit Person(int idx) : name(nameFromIdx(idx)) {}

 private:
  string name;
};

class SpecialPerson : public Person {
 public:
  // Person(rhs) 调用了基类的完美转发构造函数,rhs是一个 SpecialPerson, 万能引用是精确匹配，编译错误，下同
  SpecialPerson(const SpecialPerson& rhs) : Person(rhs) {}
  // Person(std::move(rhs)) 调用了基类的完美转发构造函数
  SpecialPerson(SpecialPerson&& rhs) : Person(std::move(rhs)) {}
};

TEST(CtorTest, ctor) {
  Person p("Nancy");
  // 编译失败，对于非常量的复制构造选择了精确地万能引用版本，从而导致string构造函数接收了一个Person对象参数
  // auto cloneOfP(p);
  auto cloneOfP(std::move(p));  // 转为右值后，与编译器生成的移动构造函数精确匹配
  const Person cp("Nancy");
  auto cloneOfCp(cp);  // 常量cp导致调用了复制构造函数，常规函数比模板实例优先级更高
}
```

如果要针对大多数的实参型别实施转发，只针对某些实参型别实施特殊处理，请参考条款27

# 条款27 熟悉依万能引用型别进行重载的替代方案

1. 如果不使用万能引用和重载的组合，替代方案包括使用彼此不同的函数名字、传递const T&型别的形参、传值和标签分派

   1. 使用不同的函数名字舍弃重载

   2. const T& 型别的形参，参考条款26引子中的例子，缺点是执行效率会低一点

   3. 传值，参考条款41，当你知道肯定需要复制形参时，考虑按值传递对象。与条款26对比：

      ```cpp
      class Person {
       public:
        explicit Person(std::string n) :name(std::move(n)) {}
        explicit Person(int idx): name(nameFromIdx(idx)) {}
       private:
        std::string name;
      };
      ```

   4. 标签分派， 使用单版本的万能引用，加上额外的标签类型形参影响重载决议

      ```cpp
      std::multiset<std::string> names;
      string nameFromIdx(int idx);
      
      template<typename T>
      void logAndAddImpl(T&& name, std::false_type) {
        names.emplace(std::forward<T>(name));
      }
      
      void logAndAddImpl(int idx, std::true_type) {
        logAndAdd(nameFromIdx(idx));
      }
      
      template <typename T>
      void logAndAdd(T&& name) {
        //万能引用，T可被推导为T&，此时is_integral必为false，所以需要使用remove_reference_t去除引用
        logAndAddImpl(std::forward<T>(name), 
                      std::is_integral<std::remove_reference_t<T>>()); 
      }
      ```

2. 经由std::enable_if对模板施加限制，就可以将万能引用和重载一起使用，但这种技术控制了万能引用重载版本被调用的条件。

   对于条款26中的万能引用构造函数来说，即使在万能引用版本实现中使用标签分派，也无法解决问题，问题在于编译器生成的函数有时会绕过万能引用版本，或者说并不能保证一定能绕过万能版本。std::enable_if 可以强制编译器表现出来的行为如同特定的模板不存在一般，即只有在其指定的条件满足时，模板才是启用的，使用这种技术，可以将万能引用模板被允许采用的条件砍掉一部分。

    ```cpp
   #include <gtest/gtest.h>
   #include <iostream>
   #include <string>
   #include <type_traits>
   #include <vector>
   
   using namespace std;
   
   // std::decay_t 消除引用和cv饰词，以及将数组和函数退化为指针
   class Person {
    public:
     template <typename T,
               typename = std::enable_if_t<!std::is_base_of<Person, std::decay_t<T>>::value &&
                                           !std::is_integral<std::remove_reference_t<T>>::value>>
     explicit Person(T&& n) : name(std::forward<T>(n)) {
       cout << "T&& version called" << endl;
     }
     explicit Person(int idx) : name(nameFromIdx(idx)) { cout << "int version called" << endl; }
     static vector<string> backNames;
   
     string nameFromIdx(int idx) { return backNames[idx]; }
     string name;
   };
   
   vector<string> Person::backNames = {"a", "b", "c"};
   
   TEST(EnableTest, person) {
     Person p{"mark"}; // T&& version called
     auto cloneOfP(p);
     Person p2{0}; // int version called
     short i = 1;
     Person p3{i}; // int version called
   }
    ```

3. 万能引用通常在性能方面具备优势，但在易用性方面一般会有劣势

   1. 效率优势：比如避免创建临时对象
   2. 易用性不足：比如针对某些型别无法实施完美转发；对于非法实参，错误信息的可理解性差，尤其是参数多层转发时

   std::is_constructible 能够在编译期间判定具备某个型别的对象是否能从另一型别（一组型别）的对象（一组对象）触发完成构造

   ```cpp
   static_assert(std::is_constructible<std::string, T>::value, "Parameter n can't be used to construct a std::string");
   ```

# 条款28 理解引用折叠

1. 引用折叠会在四种语境中发生：模板实例化、auto型别生成，创建和运用typedef和别名声明，以及decltype

2. 当编译器在引用折叠的语境下生成引用的引用时，结果会变成单个引用。如果原始的引用中有任一引用为左值引用，则结果为左值引用。否则结果为右值引用

3. 万能引用就是在型别推导的过程中会区别左值和右值，以及会发生引用折叠的语境中的右值引用。

   前面的条款讲到std::forward需要的左值、右值信息会编码到推导出的结果型别中。这个编码操作只有在实参被用以初始化形参为万能引用时才会发生。编码机制是直接了当的：如果实参是个左值，T的推导结果就是个左值引用；如果实参是个右值，T的推导结果就是个非引用型别。

   下面分析std::forward的实现：

   ```cpp
   template <typename T>
   viod f(T&& fParam) {
     someFunc(std::forward<T>(fParam));
   }
   
   // forward 的可能实现为
   template<typename T>
   T&& forward(remove_reference_t<T>& param) {
     return static_cast<T&&>(param);
   }
   ```

   如果传给f的是个左值，T的型别推导为 T&，带入f 和forward：

   ```cpp
   // f 实例化
   void f(T& && fParam){ someFunc(std::forward<T&>(fParam)); }
   // 形参类型引用折叠后 
   void f(T& fParam){ someFunc(std::forward<T&>(fParam)); }
   
   // forward 实例化
   T& && forward(T& param) { return static_cast<T& &&>(param); }
   // 引用折叠后（模板形参和返回值都会发生引用折叠）
   T& forward(T& param) { return static_cast<T&>(param); }
   ```

   从结果来看，一个左值实参传递给万能引用函数模板f后，forward的结果为左值引用，因此也是一个左值。

   同样的，如果传递的是一个右值， T的型别推导为T， 带入f和forward:

   ```cpp
   // f 实例化,不需要引用折叠
   void f(T&& fParam){ someFunc(std::forward<T>(fParam)); }
   
   // forward 实例化，同样不需要引用折叠
   T&& forward(T& param) { return static_cast<T&&>(param); }
   ```

   从结果来看，一个右值实参传递给万能引用模板f后，forward的结果为右值引用，对于返回值来说，这就是一个右值。

   综上，forward实现了左值返回左值，右值返回右值的功能。

# 条款29 假定移动操作不存在，成本高，未使用

1. 移动语义不会带来好处的三种情况（对应条款标题）：
   1. 没有移动操作，退而使用复制操作：比如不提供显示的移动操作，或者不符合编译器生成移动操作的条件
   2. 移动未能更快：不比复制快，比如string的短字符串优化
   3. 移动不可用：比如标准库容器为了强异常安全的兼容性保证，如果底层类型移动操作没有声明为noexcept，移动退化为复制。
2. 对于那些型别已知、或者移动语义支持情况已知的情况，无需作以上假定。 

# 条款30 熟悉完美转发的失败情形

**完美转发的含义**是我们不仅转发对象，还转发其显著特征：型别、是左值还是右值，以及是否带有const或volatile饰词等。

**完美转发失败**是指：给定目标函数f和转发函数fwd，当以某种特定实参调用f会执行某操作，而用同一实参调用fwd会执行不同的操作，或无法执行该操作。

```cpp
template <typename T>
void fwd(T&& param) {
  f(std::forward<T>(param));
}
f(expression); // 本语句执行了某操作
fwd(expression); // 本语句执行了不同的操作，则称fwd完美转发expression给f失败
```

1. 完美转发失败的情形，是源于模板型别推导失败，或推导结果是错误的型别

2. 会导致完美转发失败的实参种类有大括号初始化物、以值0或NULL表达空指针、仅有声明的整型static const 成员变量、模板或重载的函数名字，以及位域。

   1. 大括号初始化物：

      ```cpp
      void f(const vector<int>& v) {}
      
      template <typename T>
      void fwd(T&& param) {
        f(std::forward<T>(param));
      }
      
      TEST(ForwardTest, forwardFail) {
        // 普通函数调用，编译器会比较实参和形参的类型是否兼容，如有必要，会实施隐式型别转换使得调用成功
        f({1, 2, 3});
        // 函数模板调用，编译器会先根据fwd的实参推导模板形参的类型T，而后才比较T和f声明的形参型别，完美转发失败包含：
        // 1. 无法为一个或多个fwd的形参推导出型别结果
        // 2. 推导出了错误的类型，指推导出的型别无法通过编译或导致f的调用行为不一致。
        //
        // c++规定这是非推导语境，即向未声明为std::initialize_list型别的函数模板传递了大括号初始化，这种情况编译器会禁止类型推导
        // fwd({1, 2, 3});
        // 绕行方法
        auto i1 = {1, 2, 3};
        fwd(i1);
      }
      ```

   2. 0或NULL表达空指针

      0或NULL会被模板推导为整型，而不能用作空指针，应使用nullptr，参考条款8

   3. 仅有声明的整型static const成员变量

      c++规定：不需要给出类中的整型static  const成员变量的定义，仅需声明之。因为编译器会根据这些成员的值实施常数传播，从而不必再为他们保留内存。举例说明，下面的代码可以通过编译：

      ```c++
      class Widget {
      public:
        static const std::size_t MinVals = 28; // 声明
      };
      // 这里未给出定义
      std::vector<int> widgetData;
      widgetData.reserve(Widget::MinVals); // 这里会被替换为28常数。
      ```

      但是如果要取Widget::MinVals的地址，由没有定义，则会在链接期报错。引用的实现与指针类似，从而导致函数模板的调用失败

      ```cpp
      class Widget {
       public:
        static const std::size_t MinVals = 28;  // 声明
      };
      
      void f(std::size_t t) {}
      void f1(const std::size_t* t) {}
      
      template <typename T>
      void fwd(T&& param) {
        f(std::forward<T>(param));
      }
      
      TEST(StaticConst, test1) {
        f(Widget::MinVals);  // OK, 常数替换
        fwd(Widget::MinVals); // MacOS clang OK， 没有定义可以引用
        f1(&Widget::MinVals); // MacOS clang OK， 没有定义可以取地址
      }
      ```

      注：并不是所有编译器或链接期实现都要求按引用传递Minvals需要有定义，比如MacOS clang就不要去

      绕行或是的代码更加可移植：定义static const 成员变量

   4. 模板或重载的函数名字

      ```cpp
      int processVal(int value) { return 0; }
      int processVal(int value, int priority) { return 0; }
      void f(int (*pf)(int)) {}
      template <typename T>
      void fwd(T&& param) {
        f(std::forward<T>(param));
      }
      template <typename T>
      int workOnVal(T param) {
        return 0;
      }
      
      TEST(OverLoadTest, test1) {
        f(processVal);  // OK
        // fwd(processVal);  // error, which processVal? 没有型别，无法推导
        // fwd(workOnVal);   // error, which workOnVal intance?s
        // 绕行办法： 手动指定型别
        using ProcessFuncType = int (*)(int);
        ProcessFuncType fPtr = processVal;  // OK
        fwd(fPtr);
        fwd(static_cast<ProcessFuncType>(workOnVal));  // OK
      }
      ```

   5. 位域

      ```cpp
      struct IPv4Header {
        std::uint32_t version : 4, IHL : 4, DSCP : 6, ECN : 2, totalLenght : 16;
      };
      void f(std::size_t t) {}
      template <typename T>
      void fwd(T&& param) {  // 引用跟指针一样，但是无法获取位域的地址
        f(std::forward<T>(param));
      }
      
      template <typename T>
      void fwd2(const T& param) {
        f(param);
      }
      
      TEST(OverLoadTest, test1) {
        IPv4Header h;
        f(h.totalLenght);  // OK
        // fwd(h.totalLenght); // error: non-const reference cannot bind to bit-field 'totalLenght'
        // 绕行办法：手动制作副本
        auto length = static_cast<std::uint16_t>(h.totalLenght);
        fwd(length);
        fwd2(h.totalLenght);  // OK 常量引用，编译器制作副本。
      }
      ```