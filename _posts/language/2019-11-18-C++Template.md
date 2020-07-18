## 二十三章 模版

1. 



## 二十五章 特例化

1. 模版参数包括：类型参数，值参数，模版参数

2. 类型实参是无约束的，完全依赖于模版如何使用它，这提供了鸭子类型的一种形式。

3. 一个类型必须在作用域内可访问才可作为模版实参

4. 模版值参数的实参可以是：

   1. 整型常量表达式

   2. 外部链接的对象或函数的指针或引用

   3. 指向非重载成员的指针

   4. 空指针

      一个指针必须具有&of或f的形式，才能作为模版实参，其中of是对象或函数的名字，f是函数名。指向成员的指针必须具有&X::of的形式。字符串字面值常量不能作为模版实参。理解值参数最好的方式是将其看作向函数传递整数和指针的一种机制

5. 操作作为实参，排序准则概念可以表示为

   1. 一个模版值实参

      template<typename Key, typename V, bool(*cmp)(const Key&, const Key&)>

      class map{};

   2. map模版的一个类型实参(常用，灵活)

      template<typename Key, typename V, typename Compare = std::less<Key>>

      class map{};

      可以使用函数对象，函数指针和可以转化为函数指针的lambda表达式：

      map<string,int,std::greater < string>>; //函数对象

      using Cmp = bool(*)(const string&, const string&);

      map<string,int, Cmp> m{insentive} //函数指针

      map<string, int, Cmp> m{ [] (const string& a, const string&b){return a>b;}};//lambda

      lambda不能转化为函数对象类型,Compare是函数对象类型:

      map<string, int, Compare> c3{[] (const string&a, const string& b){return a<b;}};//错误

      可以命名lambda，然后使用其名字：

      Auto cmp = [] (const string& a, const string& b){return a<b;};

      map<string, int, decltype(cmp)> c4{cmp};

6. 模版作为实参：

   template<typename T, template<typename>class C>

   class Xrefd{

   ​	C<T> mems;

   ​	C<T*> refs;

   }

   Xrefd<int, vector>;

   为了将一个模版声明为模版参数，必须指定其所需的参数。

   一个模版作为另一个模版的参数，通常是希望用多种实参类型对其进行实例化，即用这个模版来声明另一个模版的成员，且希望这个模版是一个参数，从而可以让用户指定不同类型。

   只有类模版可以作为模版实参。

7. 默认模版参数

   只有当我们真正使用它时，编译器才会对默认实参进行语义检查

   类似默认函数实参，只能对尾部模版参数指定和提供默认值

   如果所有函数模版实参都有默认参数，则<>可省略。

8. 特例化

   1. 完全特例化

      template<>

      class Vector<void*>{};

      前缀template<>表示这是一个不必指明模版参数的特例化版本，特例化版本所用的模版实参由模版名后面的括号<>中的内容指定。这种不用再指定或推断任何模版参数的特例化也叫完全特例化。

   2. 部分特例化

      定义所有指针的vector版本：

      template< typename T>

      class Vector<T\*>: private Vector<void\*>{

      

      }

   3. 

9. 