---
title: effective modern c++ 之 智能指针
descriptions: 高效使用 c++11 特性
categories: language
tags: c++11, shared_ptr
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

2. 默认的，析构资源使用delete，但可以指定自定义删除器。有状态的删除器和采用函数指针实现的删除器会增加unique_ptr 型别的对象尺寸（无状态的函数对象和无捕获的lambda表达式不会增加尺寸，即与裸指针一样  ）

   ```cpp
   // 注意删除器的参数类型
   auto delInvmt = [](Investment* pInvest) {
     makeLogEntry(pInvest);
     delte pInvest;
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

## 条款21

## 条款22

