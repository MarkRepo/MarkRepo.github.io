---
title: Python之generator，yield，yield from 详解
description:
categories: language
tags:
  - python
  - yield
---

## Generator Expressions

生成器表达式是用小括号表示的简单生成器标记法：

	generator_expression ::= "(" expression comp_for ")"

生成器表达式产生一个生成器对象，它的语法和for类似，除了它是被“（）”包含，而不是[]或{}；  
生成器表达式中变量的计算被延迟到`__next__()`函数的调用，然而最左边for循环子句被立即计算，这样，如果他有错误的话可以被立即看到。后面的for循环子句不能被立即计算，因为他们可能依赖于前面的for循环，例如`（x*y for x in range(10) for y in bar(x)）`

python 3.6以后，如果生成器出现在`async def function`中，那么`async for`子句和await表达式可以被理解为是异步的。如果生成器表达式包含`async for`子句或者await表达式，就叫做异步生成器表达式。异步生成器表达式产生一个新的异步生成器对象，它是一个异步迭代器。

## Yield Expressions

	yield_atom       ::=  "(" yield_expression ")"
	yield_expression ::=  "yield" [expression_list | "from" expression]

Yield表达式用于定义一个生成器函数或异步生成器函数，因此只能被用于一个函数定义体内。在一个函数定义体中使用yield表达式使其成为生成器，在一个async def函数体内使用yield表达式使协程函数成为一个异步生成器。例如：

```python
def gen（）：   #定义一个生成器函数
　　yield 123
async def agen（）： #定义一个异步生成器函数
　　yield 123
```
当一个生成器函数被调用，它返回一个迭代器，也叫生成器。这个生成器控制生成器函数的执行。当生成器的一个函数被调用的时候，生成器函数开始执行。

执行到第一个yield表达式时，挂起，返回`expression_list`的值给调用者。对于挂起，我们指的是所有局部状态被保留，包含当前局部变量的绑定，指令指针，内部调用栈，任何异常处理状态。当通过调用生成器的一个函数恢复执行时，函数的执行就好像是从外部再次调用yield表达式一样。恢复执行后，yield表达式的值取决调用的方法。如果是`__next_()`被调用（一般通过for循环或者内置的next（）函数），那么值是None。如果是send（）被调用，值是传给send的参数的值。

所有的这些使得生成器函数非常像协程；它产生多次值，有多个入口点并且执行可以被挂起。唯一的不同是，生成器函数yield后不能控制程序从哪里继续执行，控制权总是传回给生成器的调用者。

yield表达式可以在try块的任何地方，如果生成器在被结束（到达零引用或者因为垃圾回收机制）之前没有被恢复执行，生成器的close函数被调用，因此finally子句被执行。

当`yield from <expr>`被使用，它把附加的表达式当成一个子迭代器，所有子迭代器产生的值被直接传回给当前生成器函数的调用者。当前生成器调用send的参数值和调用throw的异常参数 都将被传给底层迭代器（子迭代器），如果他有对应的方法的话。否则，send导致AttributeError或者TypeError，而throw立即raise传给他的异常。

当底层迭代器执行完成，StopIteration对象的value值变成这个yield from表达式的值。这个值可以被显式的设置当产生StopIteration时，或者自动设置，如果子迭代器是一个生成器（子生成器返回一个值）

## 生成器-迭代器 方法

这部分介绍生成器迭代器的方法，他们可以被用来控制生成器函数的执行。当生成器正在执行时调用这些函数将导致ValueError。

+ `generator.__next__()`  
开始生成器函数的执行或者从最后被执行的yield表达式中恢复执行，如果是恢复执行，yield表达式的值是None，继续执行到下一个yield表达式，挂起，返回expression_list的值给`__next__()`的调用者。如果生成器没有再yield一个值，则产生一个StopIteration异常
这个方法一般被隐式调用，例如for循环，next()
+ `generator.send(value)`  
恢复执行并且发送一个值到生成器函数。这个value参数就是当前yield表达式的结果。send方法返回下一个生成器yield的值
如果没有yield，返回StopIteration。当用send来启动生成器，他的参数必须是None，因为没有yield表达式接收值。
+ `generator.throw(type[,value[,traceback]])`  
在生成器挂起的地方产生一个type类型的异常，并且返回下一个生成器函数yield的值，如果没有yield，返回StopIteration。
如果生成器函数没有捕获这个传进去的异常 ，或者产生了另一个不同的异常，那么将这个异常传递给调用者。
+ `generator.close()`  
在生成器挂起的地方产生一个GeneratorExit()，如果生成器之后优雅的退出，已经关闭，或者产生了一个GeneartorExit（不捕获该异常），close将返回到它的调用者。如果生成器yield一个值，那么产生一个RuntimeError。如果生成器产生了任何其他异常，将传递给调用者。如果
生成器因为一个异常或者正常退出，那么close不做任何事情。 

## 实例：

```python
>>> def echo(value=None):
...     print("Execution starts when 'next()' is called for the first time.")
...     try:
...         while True:
...             try:
...                 value = (yield value)
...             except Exception as e:
...                 value = e
...     finally:
...         print("Don't forget to clean up when 'close()' is called.")
...
>>> generator = echo(1)
>>> print(next(generator))
Execution starts when 'next()' is called for the first time.
1
>>> print(next(generator))
None
>>> print(generator.send(2))
2
>>> generator.throw(TypeError, "spam")
TypeError('spam',)
>>> generator.close()
Don't forget to clean up when 'close()' is called.
```

PEP380 加入了`yield from`表达式，允许一个生成器委派部分操作给另一个生成器。这可以剔除一部分包含yield的代码放到另一个生成器。另外，子生成器可以返回一个值，这个值对于委托生成器也是可用的。虽然主要涉及用来委派一个子生成器，但是`yield from`表达式事实上可以委派任何的子迭代器。至于简单的迭代器， `yield from iterable` 本质上就是一个简短的形式：`for item in iterable: yield item`,例如：

```python
>>> def g(x):
...     yield from range(x, 0, -1)
...     yield from range(x)
...
>>> list(g(5))
[5, 4, 3, 2, 1, 0, 1, 2, 3, 4]
```
然而，不像普通的循环，yield from 允许子生成器直接从调用区域接收send和throw的值，并且返回一个最后的值给外层生成器。示例如下：

```python
>>> def accumulate():
...     tally = 0
...     while 1:
...         next = yield
...         if next is None:
...             return tally
...         tally += next
...
>>> def gather_tallies(tallies):
...     while 1:
...         tally = yield from accumulate()
...         tallies.append(tally)
...
>>> tallies = []
>>> acc = gather_tallies(tallies)
>>> next(acc)  # Ensure the accumulator is ready to accept values
>>> for i in range(4):
...     acc.send(i)
...
>>> acc.send(None)  # Finish the first tally
>>> for i in range(5):
...     acc.send(i)
...
>>> acc.send(None)  # Finish the second tally
>>> tallies
[6, 10]
```
