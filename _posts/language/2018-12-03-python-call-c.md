---
title: 在python里调用C函数的三种方式[转]
categories: language
tags: python
---

[原文链接](https://www.yanxurui.cc/posts/python/2017-06-18-3-ways-of-calling-c-functions-from-python/)

一个python项目快速开发完以后，常常针对瓶颈进行优化，其中一种方式就是对于性能至关重要的部分，使用C重写，这已经是一种最佳实践。如果整个项目完全使用C，开发效率就没有保障。python运行环境(CPython)是用C开发的，因此python与C结合起来很容易，而且方式多种多样。使用C重写了关键部分后，需要在python中调用，本文介绍三种最常用的调用C函数的方式，分别是c extension，Cython和ctypes。

举个例子，假设我们用C重写了add函数，它接受两个整数，计算他们的和并返回。

```c
#include <stdio.h>

int add(int a, int b)
{
    return a + b;
}

int main()
{
    int a=1,b=1;
    printf("%d\n", add(a,b));
}
```

那么如何在python中调用add？

## c extension

### 介绍

python标准库包含了很多使用C开发的扩展模块，比如对性能要求很高的json库。开发者同样可以使用C开发扩展，这是最原始也是最底层的扩展python的方式。

### 示例

#### demomodule.c

python的扩展模块由以下几部分组成：

+ Python.h
+ C函数
+ 接口函数（python代码调用的函数）到C函数的映射表
+ 初始化函数

```c
// pulls in the Python API 
#include <Python.h>

// C function always has two arguments, conventionally named self and args
// The args argument will be a pointer to a Python tuple object containing the arguments.
// Each item of the tuple corresponds to an argument in the call’s argument list.
static PyObject *
demo_add(PyObject *self, PyObject *args)
{
    const int a, b;
    // convert PyObject to C values
    if (!PyArg_ParseTuple(args, "ii", &a, &b))
        return NULL;
    return Py_BuildValue("i", a+b);
}

// module's method table
static PyMethodDef DemoMethods[] = {
    {"add", demo_add, METH_VARARGS, "Add two integers"},
    {NULL, NULL, 0, NULL} 
};

// module’s initialization function
PyMODINIT_FUNC
initdemo(void)
{
    (void)Py_InitModule("demo", DemoMethods);
}
```

#### setup.py

编译扩展模块通常使用distutils或setuptools，它会自动调用gcc完成编译和链接。

```python
from distutils.core import setup, Extension

module1 = Extension('demo',
                    sources = ['demomodule.c']
                    )

setup (name = 'a demo extension module',
       version = '1.0',
       description = 'This is a demo package',
       ext_modules = [module1])
```
执行

```
python setup.py build_ext --inplace
```

会在当前目录生成一个demo.so。一个python扩展模块其实就是一个共享库(`.so`)，它可以直接在python解释器中import。
`--inplace`表示将生成的扩展放到源码所在的目录，即当前目录，这样就可以直接import而不需要安装到`site-packages`目录。

#### 测试

```
>>> from demo import add
>>> add(1,1)
2
>>> add(1,2)
3
>>> add(1)
Traceback (most recent call last):
  ...
TypeError: function takes exactly 2 arguments (1 given)
>>> add(1,'2')
Traceback (most recent call last):
  ...
TypeError: an integer is required
```

## Cython

### 介绍

Cython听起来像是一种语言，c与python的结合，这么说其实没有错。python是一种动态类型的解释型语言，执行效率低，Cython在python的基础上增加了可选的静态类型申明的语法，代码在使用前先被转换成优化过的C代码，然后编译成python扩展库，大大提升了执行效率。因此从语言的角度来讲，Cython是python的超集，即扩展了的python。

注意不要和CPython混淆，CPython是用c实现的python解释器，由官方提供，我们平时使用的python就是CPython。另外，pypy是python自己实现的python解释器。Cython是cpython标准库的一部分，不需要额外安装。

用官网的一句话介绍Cython的作用：

extending the CPython interpreter with fast binary modules, and interfacing Python code with external C libraries.

简单的说，Cython的两个主要作用是：

1. 将python代码编译成二进制的扩展模块，以获得加速；同时可以在python中使用类型声明，进一步提升性能；这就意味着可以使用python代替c编写python扩展
2. 在python代码里调用外部的c库

### 示例

现在使用Cython重新实现上面的例子——编写C函数的包装器。

最终的目录结构如下

```
.
├── add_wrapper.c
├── add_wrapper.pyx
├── add_wrapper.so
├── build
│   └── temp.linux-x86_64-2.7
│       └── add_wrapper.o
├── libadd.a
├── libadd.c
├── libadd.h
├── libadd.o
└── setup.py
```

#### 编译C程序

libadd.h

```c
int add(int a, int b);
```

libadd.c

```
int add(int a, int b)
{
    return a + b;
}
```
一般都是通过python调用动态链接库，需要将生成的库文件(.so)安装到标准路径下（比如/usr/lib)下，链接和运行的时候才能找到该文件，为了方便这里以静态链接库为例。

首先将c文件编译成静态链接库：

```
gcc -c libadd.c
ar rcs libadd.a libadd.o
```
第一步会在当前目录下生成libadd.o，第二步创建静态链接库libadd.a。

#### 使用Cython包装C函数

使用Cython调用c函数很简单，只需要在Cython中声明函数的签名，然后编译的时候正确地链接外部的动态或静态库。

下面就是一个add函数的python包装器： add_wrapper.pyx

```python
cdef extern from "libadd.h":
    cpdef int add(int a, int b)
```
第一行表示引入头文件libadd.h。第二行声明该头文件中的add函数，直接从libadd.h拷贝过来即可，此时只有在Cython模块内部能调用该C函数，还需要在前面加cpdef声明，表示暴露出接口给python调用。

#### 编译Cython代码

Cython是需要编译成二进制模块才能使用的，编译过程包含两步：

1. Cython将Cython文件(.pyx)编译成c代码(.c)
2. gcc将c代码编译成共享库(.so)

怎么编译呢？最常用的方式是编写一个setup.py文件：

```python
from distutils.core import setup, Extension
from Cython.Build import Cythonize

ext_modules=[
    Extension("add_wrapper",
              sources=["add_wrapper.pyx"],
              extra_objects=['libadd.a']
    )
]

setup(
  name = 'wrapper for libadd',
  ext_modules = Cythonize(ext_modules),
)
```
`extra_objects`表示需要链接的静态库文件，也可以替换成`libraries=["add"],library_dirs=["."]`，连接器会自动搜索`libadd.so`和`libadd.a`，动态链接库优先。  
执行

```
python setup.py build_ext --inplace
```
在当前目录下会生成`add_wrapper.c`和`add_wrapper.so`，`add_wrapper.c`是第一步编译生成的中间文件，内容比较长。add_wrapper.so是最终的python二进制模块，将它放到PYTHONPATH的某个路径下，就可以直接import。

如果需要重新build，你可能需要加上--force选项，否则可能不会生效。

#### 测试

```
>>> from add_wrapper import add
>>> add(1,1)
2
>>> add(2,3)
5
>>> add(-1,1)
0
>>> add(1,False)                                                                                                  
1
>>> add(1)
Traceback (most recent call last):
  ...
TypeError: wrap() takes exactly 2 positional arguments (1 given)
>>>
>>> add(1,'1')
Traceback (most recent call last):
  ...
TypeError: an integer is required
```
由此可见，Cython会自动检查参数类型并完成python对象到C类型的转换。

## ctypes

### 介绍

ctypes的主要作用就是在python中调用C动态链接库（shared library）中的函数。

### 示例

#### 编译成动态链接库

libadd.c

```c
int add(int a, int b)
{
    return a + b;
}
```
```
gcc -shared -o libadd.so libadd.c
```

#### 加载共享库

使用CDLL动态加载共享库，一个共享库对应一个cdll对象。调用cdll的LoadLibrary()方法或直接调用CDLL的构造函数创建一个CDLL对象。

```
>>> from ctypes import *
>>> mylib = CDLL('/home/yanxurui/test/keepcoding/python/extension/ctypes/libadd.so')
```
第二行的CDLL等价于cdll.LoadLibrary。

如果共享库不在标准路径/usr/lib下则需要使用完整的路径。 ctypes提供了find_library用来找到共享库的位置，但是find_library会查找/usr/local/lib，因此搜索成功不代表也能加载成功。有人也反映了这个bug：

CDLL does not use the same paths as find_library and thus you can find a library, but you can't necessarily use it.

#### 调用共享库里的函数

通过访问dll对象的属性来调用相应的函数，就像调用python的函数对象一样：

```
>>> mylib.add
<_FuncPtr object at 0x7ff6864b7bb0>
>>> add = mylib.add
>>> add(1,2)
3
>>> add()
1
>>> add(1)
-2044290911
>>> add(1,'a')                                                                                                    
-2042137139
```

#### 指定函数类型

ctypes并不会校验参数的数量和类型，通过设置函数的argtypes的属性可以指定函数参数的类型：

```
>>> add.argtypes = [c_int, c_int]
>>> add(1, 2)
3
>>> add(1)
Traceback (most recent call last):
  ...
TypeError: this function takes at least 2 arguments (1 given)
>>> add(1, '2')
Traceback (most recent call last):
  ...
ctypes.ArgumentError: argument 2: <type 'exceptions.TypeError'>: wrong type
```
另外，原生的python类型中只允许传入None, 整数, 字符串作为函数的参数。如果需要传递其他的类型，则需要使用ctypes定义的类型，比如c_double表示double。

## benchmark

从上面看出，c扩展虽然复杂，但更接地气，性能必然也是最好的，而Cython和ctypes开发效率奇高。

调用C库的一个主要目的是优化性能，因此我们更关心三种方式对性能的影响。 下面通过一个简单的benchmark来比较，即使10000000次加法操作也很快，很难看出调用C函数对性能带来的提升，但这无所谓，因为我们的主要目的是对比不同调用方式在调用共享库时的性能开销。

测试的代码如下，由于模块名以及import的方式不同，所以每次测试需要稍微修改一下注释的地方。

```python
from time import time
# c ext
# from demo import add

# Cython
# from add_wrapper import add

# ctypes
# mylib = CDLL('/home/yanxurui/test/keepcoding/python/extension/ctypes/libadd.so')
# add = mylib.add
# add.argtypes = [c_int, c_int]

# python
# def add(a,b):
#    return a+b

s=time()
for i in range(10000000):
    r = add(i, i)
print(time()-s)
10000000 loops, best of 3:

method	cost(s)
c ext	2.522
Cython	1.723
ctypes	8.896
python	1.879
```
测试的结果让人惊讶：

1. 纯python比c扩展快？
2. Cython的调用开销居然比C模块还低，这个是为何？？？
3. 使用ctypes调用C库居然比纯python慢这么多

对于这个测试的结果，我无法盲目的相信，还需要进一步的探究。

### 总结

如果已经有一个现成的库，我会选择使用Cython或ctypes作为包装器，如果还需要考虑性能的话，当然就是Cython了。 如果需要从零开始开发一个功能模块，并且对性能要求极严，我会编写C扩展。

## 参考

+ [Stackoverflow: Wrapping a C library in Python: C, Cython or ctypes?](https://stackoverflow.com/questions/1942298/wrapping-a-c-library-in-python-c-Cython-or-ctypes)
+ [Extending Python with C or C++](https://docs.python.org/2/extending/extending.html)
+ [Python Extension Programming with C](https://www.tutorialspoint.com/python/python_further_extensions.htm)
+ [Cython: Calling C functions](http://docs.cython.org/en/latest/src/tutorial/external.html)
+ [ctypes — A foreign function library for Python](https://docs.python.org/2.7/library/ctypes.html)
+ [distutils.core — Core Distutils functionality](https://docs.python.org/2/distutils/apiref.html)