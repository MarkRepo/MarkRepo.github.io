---
title: effective modern c++ 之 auto
descriptions: 高效使用 c++11 特性
categories: language
tags: c++11
---

#  二、auto

## 条款5 优先使用auto，而非显示型别声明

auto 的优点：

1. 可以少打一些字 
2. 还能阻止那些由于手工指定型别带来的潜在错误和性能问题 
3. 用auto声明变量，其型别都推导自其初始化物，所以他们必须初始化。
4. 使用auto 并不会导致可读性问题，对于型别的抽象理解和了解他的精确型别同等有用
5. 使用auto有利于代码重构

 举例1：关于如何声明变量来持有lambda闭包，使用std::function和auto的比较：

1. std::function对象一般都会比使用auto声明的变量使用更多的内存

   使用auto声明的、存储着一个闭包的变量和该闭包是同一型别，从而他要求的内存量也一样； 而使用std::function声明、存储着一个闭包的变量是std::function的一个实例，不管给定的签名如何，它拥有固定的内存，如果这个内存不足以包含其存储的闭包的话，std::function的构造函数会分配堆上的内存来存储闭包。

2. 使用std::function来调用闭包会比通过使用auto声明的变量来调用同一闭包来的慢

举例2：使用auto作为目标变量的型别，就没必要担心在用以声明变量的型别和他的初始化表达式的型别之间的不匹配了：

1. `std::vector<int>::size_type ` 声明成了 `unsigned`，导致的一些移植性问题。

2. `std::pair<const std::string, int>`  声明成了 `std::pair<std::string, int>`， 导致的临时对象和性能问题

## 条款6 如果auto推导的型别不符合要求时，使用带显示型别的初始化物习惯用法

1. ”隐形“ 的代理型别可以导致auto根据初始化表达式推导出”错误的“型别
2. 带显示型别的初始化物习惯用法强制auto推导出你想要的型别

举例：

```cpp
std::vector<bool> features(const Widget& w);
Widget w;
//1. std::vector<bool>::operator[] 返回std::vector<bool>::reference这个代理类型对象
//2. 代理对象隐式转化为bool
bool highPriority = feature(w)[5]; 
processWidget(w, highPriority); // ok

// 使用auto
//1. features(w)返回一个临时的std::vector<bool>对象
//2. highPriority是一个std::vector<bool>::reference 代理对象，它的资源已被临时对象释放，导致未定义行为
auto highPriority = features(w)[5];
processWidget(w, highPriority); // 未定义行为

// 使用带显示型别的初始化物习惯用法：
// 要求使用auto声明，但针对初始化表达式进行强制类型转换，转换成你想要auto推导出来的类型。
auto highPriority = static_cast<bool>(features(w)[5]);
```

一个普遍规律是，”隐形“的代理类无法和auto和平共处。这种对象往往设计成仅仅维持到单个语句之内存在，所以如果要创建这种类的变量，往往就是违反了基本的库设计的假定前提。

使用带显示型别的初始化物习惯用法，它同样可以应用于你想要强调你意在创建一个型别不同于初始化表达式型别的变量的场合。