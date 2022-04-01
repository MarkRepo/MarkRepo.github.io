---
title: effective modern c++ 之 微调
descriptions: 高效使用 c++11 特性
categories: language
tags: c++11
---

# 条款41 针对可复制的形参，在移动成本低并且一定会被复制的前提下，考虑将其按值传递

考虑addnames的三种实现方式（重载，万能引用，按值传递）：

```cpp
// 途径一： 针对左值和右值重载
// 缺点：维护两份函数
class Widget {
public:
  void addNames(const std::string& newName){
    names.push_back(newName);
  }
  void addNames(std::string&& newName){
    names.push_back(std::move(newName));
  }
private:
  std::vector<std::string> names;
};

// 途径二： 使用万能引用
// 缺点：addNames的实现一般要在头文件中，导致头文件膨胀；
// 可能在目标代码中产生好几个函数(条款25)；怪异的失败情形，即有些型别不能按引用传递（条款30）；令人费解的错误信息（条款27）
class Widget {
public: 
  template<typename T>
  void addNames(T&& newName) {
    names.push_back(std::forward<T>(newName));
  }
};

// 途径三： 按值传递
// 优点：只有一个函数；没有万能引用的怪癖；针对左值实施复制，针对右值实施移动。
class Widget {
public:
  // 按值传递使得newName形参对实参没有依赖，因此使用std::move没有问题
  void addNames(std::string newName){
    names.push_back(std::move(newName));
  }
};

// 使用左值和右值调用
Widget w;
std::string name("mark"); 
w.addNames(name); // 左值
w.addNames(name+"niubi"); // 右值
```

**在c++11 中，newName仅在传入左值实参时才会被复制构造，如果传入的是右值实参，会被移动构造**

因此上面的三种实现方式在实参为左值和右值时的性能消耗如下：

1. 重载：对于左值是一次复制，对于右值是一次移动
2. 万能引用：对于左值时一次复制，对于右值是一次移动
3. 按值传递：对于左值时一次复制加一次移动，对于右值是两次移动

对于本条款标题：针对可复制的形参，在移动成本低并且一定会被复制的前提下，考虑将其按值传递。

使用这样的描述有4个理由：

1. 你只是“考虑”按值传递，因为还得考虑使用按值传递带来的成本

2. 仅对于可复制的形参，才考虑按值传递。不符合这个要求必然是只移型别，其复制构造函数被禁止，因此无需提供左值版本，因为复制左值需要调用复制构造函数。考虑下面的例子

   ```cpp
   // 实现一
   class Widget{
   public:
     void setPtr(std::unique_ptr<std::string>&& ptr) {
       p = std::move(ptr);
     }
   private:
     std::unique_ptr<std::string> p;
   };
   
   // 实现二
   class Widget {
   public:
     void setPtr(std::unique_ptr<std::string> ptr){
       p = std::move(ptr);
     }
   };
   
   Widget w;
   // 实现一的成本是一次移动，实现二的成本是两次移动
   w.setPtr(std::make_unique<std::string>("Modern c++"));
   ```

3. 按值传递仅在形参移动成本低廉的前提下，才值得考虑。

4. 应该只针对一定会被复制的形参才考虑按值传递

   ```cpp
   class Widget {
   public:
     void addName(std::string newName) {
       if ((newName.length() >= minLen)) {
         names.push_back(std::move(newName));
       }
     }
   };
   ```

   即使没有向names添加任何内容，该函数也会招致构造和析构newName的成本。如果采用的是按引用途径，就不会有这些成本。

前面的讨论是针对使用构造来实施形参复制的函数（push_back调用），这种情况，改为按值传递，会增加一次额外的移动所带来的成本。但是如果函数使用赋值来实施形参复制的话，成本分析就复杂的多了。这时通常情况，总是采用重载或万能引用而非按值传递，除非已确凿的证明按值传递能够为所需的形参型别生成可接受效率的代码

按值传递会导致切片问题，因此基类型别特别不适用于按值传递。

# 条款42 考虑置入而非插入

1. 插入函数接受的是待插入对象，而置入函数接受的则是待插入对象的构造函数参数。这一区别就让置入函数得以避免临时对象的创建和析构，但插入函数无法避免。

2. 启发式的方法判断置入是否比插入更高效：

   1. 欲添加的值是以构造而非赋值方式加入容器
   2. 传递的实参型别与容器持有之物的型别不同
   3. 容器不太可能由于出现重复情况而拒绝待添加的值。因为，想检测某值是否已经在容器中，置入的实现通常会使用该新值创建一个节点以便将该节点的值与容器的现有节点进行比较。如果该值已存在，置入被中止，节点被析构，浪费了构造和析构成本。

3. 是否选用置入函数，还需要考虑两个问题

   1. 和资源管理有关。在调用持有资源管理对象的容器（如`std::list<std::shared_ptr<Widget>>`）置入函数时，完美转发会推迟资源管理对象的创建，直到他们能够在容器的内存中构造为止。这就有可能因为异常而导致资源泄漏。而插入函数通常能确保在资源的获取和对资源管理对象的创建之间不再有任何其他动作。

   2. 与带有explicit声明饰词的构造函数之间的互动。这涉及到直接初始化和复制初始化的区别

      ```cpp
      std::regex r1 = nullptr; // 错误，不能通过编译（复制初始化）
      std::regex r1(nullptr); // 能编译（直接初始化）
      ```

      复制初始化不允许调用带有explicit声明饰词的构造函数，但直接初始化就允许。

      置入函数使用的是直接初始化，所以他们能够调用带有explicit声明饰词的构造函数。而插入函数是复制初始化，不能调用explicit构造函数。因此使用置入函数时，要特别小心去保证传递了正确的实参。