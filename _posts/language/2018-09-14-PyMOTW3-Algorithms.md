---
title: PyMOTW-3 --- Algorithms
categories: language
tags: PyMOTW-3
---

Python包含几个模块，用于使用最适合任务的任何风格优雅简洁地实现算法。它支持纯粹的过程、面向对象和函数式风格，并且这三种风格经常混合在同一程序的不同部分中。

functools包括创建函数装饰器、支持面向方面（aspect oriented）的编程和“超出传统的面向对象方法所支持的”代码重用等功能。它还提供了一个类装饰器，用于使用快捷方式实现所有丰富的比较API，以及用于创建“包含其参数的”函数的引用的partial对象，。

itertools模块包括用于创建和使用“函数式编程中使用的”迭代器和生成器的函数。通过为内置操作（如算术或项查找）提供基于函数的接口，operator模块在使用函数式编程风格时不需要许多普通的lambda函数。

无论程序使用何种样式，contextlib都可以使资源管理更简单，更可靠，更简洁。结合上下文管理器和with语句减少了`try：finally`块和缩进级别所需的数量，同时确保在正确的时间关闭和释放文件、套接字、数据库事务和其他资源。

## functools — Tools for Manipulating Functions

目的：操作其他函数的函数。   
functools模块提供了用于调整或扩展函数和其他可调用对象的工具，而无需完全重写它们。

### Decorators

functools模块提供的主要工具是partial类，可用于使用默认参数“包装”可调用对象。 生成的对象本身是可调用的，可以将其视为原始函数。 它采用与原始函数相同的所有参数，并且可以使用额外的位置或命名参数进行调用。 可以使用partial代替lambda为函数提供默认参数，同时保留一些未指定的参数。

#### Partial Objects

此示例显示函数myfunc（）的两个简单partial对象。 show_details（）的输出包括partial对象的func，args和keywords属性。

```python
#functools_partial.py
import functools

def myfunc(a, b=2):
    "Docstring for myfunc()."
    print('  called myfunc with:', (a, b))

def show_details(name, f, is_partial=False):
    "Show details of a callable object."
    print('{}:'.format(name))
    print('  object:', f)
    if not is_partial:
        print('  __name__:', f.__name__)
    if is_partial:
        print('  func:', f.func)
        print('  args:', f.args)
        print('  keywords:', f.keywords)
    return

show_details('myfunc', myfunc)
myfunc('a', 3)
print()

# Set a different default value for 'b', but require the caller to provide 'a'.
p1 = functools.partial(myfunc, b=4)
show_details('partial with named default', p1, True)
p1('passing a')
p1('override b', b=5)
print()

# Set default values for both 'a' and 'b'.
p2 = functools.partial(myfunc, 'default a', b=99)
show_details('partial with defaults', p2, True)
p2()
p2(b='override b')
print()

print('Insufficient arguments:')
p1()
```
在示例的末尾，调用第一个创建的partial对象而不传递a的值，从而导致异常。

```
$ python3 functools_partial.py

myfunc:
  object: <function myfunc at 0x1007a6a60>
  __name__: myfunc
  called myfunc with: ('a', 3)

partial with named default:
  object: functools.partial(<function myfunc at 0x1007a6a60>, b=4)
  func: <function myfunc at 0x1007a6a60>
  args: ()
  keywords: {'b': 4}
  called myfunc with: ('passing a', 4)
  called myfunc with: ('override b', 5)

partial with defaults:
  object: functools.partial(<function myfunc at 0x1007a6a60>, 'default a', b=99)
  func: <function myfunc at 0x1007a6a60>
  args: ('default a',)
  keywords: {'b': 99}
  called myfunc with: ('default a', 99)
  called myfunc with: ('default a', 'override b')

Insufficient arguments:
Traceback (most recent call last):
  File "functools_partial.py", line 51, in <module>
    p1()
TypeError: myfunc() missing 1 required positional argument: 'a'
```

#### Acquiring Function Properties

默认情况下，partial对象没有`__name__`或`__doc__`属性，如果没有这些属性，则被修饰的函数更难调试。 使用`update_wrapper（）`，将原始函数的属性复制或添加到partial对象。

```python
#functools_update_wrapper.py
import functools

def myfunc(a, b=2):
    "Docstring for myfunc()."
    print('  called myfunc with:', (a, b))

def show_details(name, f):
    "Show details of a callable object."
    print('{}:'.format(name))
    print('  object:', f)
    print('  __name__:', end=' ')
    try:
        print(f.__name__)
    except AttributeError:
        print('(no __name__)')
    print('  __doc__', repr(f.__doc__))
    print()

show_details('myfunc', myfunc)

p1 = functools.partial(myfunc, b=4)
show_details('raw wrapper', p1)

print('Updating wrapper:')
print('  assign:', functools.WRAPPER_ASSIGNMENTS)
print('  update:', functools.WRAPPER_UPDATES)
print()

functools.update_wrapper(p1, myfunc)
show_details('updated wrapper', p1)
```
添加到包装器的属性在`WRAPPER_ASSIGNMENTS`中定义，而`WRAPPER_UPDATES`列出要修改的值。

```
$ python3 functools_update_wrapper.py

myfunc:
  object: <function myfunc at 0x1018a6a60>
  __name__: myfunc
  __doc__ 'Docstring for myfunc().'

raw wrapper:
  object: functools.partial(<function myfunc at 0x1018a6a60>, b=4)
  __name__: (no __name__)
  __doc__ 'partial(func, *args, **keywords) - new function with
partial application\n    of the given arguments and keywords.\n'

Updating wrapper:
  assign: ('__module__', '__name__', '__qualname__', '__doc__', '__annotations__')
  update: ('__dict__',)

updated wrapper:
  object: functools.partial(<function myfunc at 0x1018a6a60>, b=4)
  __name__: myfunc
  __doc__ 'Docstring for myfunc().'
```

#### Other Callables

Partial可以与任何可调用对象一起使用，而不仅仅是独立函数。

```python
#functools_callable.py
import functools

class MyClass:
    "Demonstration class for functools"

    def __call__(self, e, f=6):
        "Docstring for MyClass.__call__"
        print('  called object with:', (self, e, f))

def show_details(name, f):
    "Show details of a callable object."
    print('{}:'.format(name))
    print('  object:', f)
    print('  __name__:', end=' ')
    try:
        print(f.__name__)
    except AttributeError:
        print('(no __name__)')
    print('  __doc__', repr(f.__doc__))
    return

o = MyClass()

show_details('instance', o)
o('e goes here')
print()

p = functools.partial(o, e='default for e', f=8)
functools.update_wrapper(p, o)
show_details('instance wrapper', p)
p()
```
此示例使用带有`__call __（）`方法的类的实例创建partials。

```
$ python3 functools_callable.py

instance:
  object: <__main__.MyClass object at 0x1011b1cf8>
  __name__: (no __name__)
  __doc__ 'Demonstration class for functools'
  called object with: (<__main__.MyClass object at 0x1011b1cf8>, 'e goes here', 6)

instance wrapper:
  object: functools.partial(<__main__.MyClass object at 0x1011b1cf8>, f=8, e='default for e')
  __name__: (no __name__)
  __doc__ 'Demonstration class for functools'
  called object with: (<__main__.MyClass object at 0x1011b1cf8>, 'default for e', 8)
```

#### Methods and Functions

虽然partial（）返回一个可以直接使用的callable，但partialmethod（）返回的可使用的callable，可用作对象的未绑定方法。 在下面的示例中，同一个standalone函数作为MyClass的属性添加两次，一次使用partialmethod（）作为method1（），再次使用partial（）作为method2（）。

```python
#functools_partialmethod.py
import functools

def standalone(self, a=1, b=2):
    "Standalone function"
    print('  called standalone with:', (self, a, b))
    if self is not None:
        print('  self.attr =', self.attr)

class MyClass:
    "Demonstration class for functools"

    def __init__(self):
        self.attr = 'instance attribute'

    method1 = functools.partialmethod(standalone)
    method2 = functools.partial(standalone)

o = MyClass()

print('standalone')
standalone(None)
print()

print('method1 as partialmethod')
o.method1()
print()

print('method2 as partial')
try:
    o.method2()
except TypeError as err:
    print('ERROR: {}'.format(err))
```
可以从MyClass的实例调用method1（），并将实例（self）作为第一个参数传递，就像通常定义的方法一样。 method2（）未设置为绑定方法，因此必须显式传递self参数，否则调用将导致TypeError。

```
$ python3 functools_partialmethod.py

standalone
  called standalone with: (None, 1, 2)

method1 as partialmethod
  called standalone with: (<__main__.MyClass object at 0x1007b1d30>, 1, 2)
  self.attr = instance attribute

method2 as partial
ERROR: standalone() missing 1 required positional argument:
'self'
```

#### Acquiring Function Properties for Decorators

在装饰器中使用时，更新被包装的可调用对象的属性特别有用，因为已转换的函数最终具有原始“裸”函数的属性。

```python
#functools_wraps.py
import functools

def show_details(name, f):
    "Show details of a callable object."
    print('{}:'.format(name))
    print('  object:', f)
    print('  __name__:', end=' ')
    try:
        print(f.__name__)
    except AttributeError:
        print('(no __name__)')
    print('  __doc__', repr(f.__doc__))
    print()

def simple_decorator(f):
    @functools.wraps(f)
    def decorated(a='decorated defaults', b=1):
        print('  decorated:', (a, b))
        print('  ', end=' ')
        return f(a, b=b)
    return decorated

def myfunc(a, b=2):
    "myfunc() is not complicated"
    print('  myfunc:', (a, b))
    return

# The raw function
show_details('myfunc', myfunc)
myfunc('unwrapped, default b')
myfunc('unwrapped, passing b', 3)
print()

# Wrap explicitly
wrapped_myfunc = simple_decorator(myfunc)
show_details('wrapped_myfunc', wrapped_myfunc)
wrapped_myfunc()
wrapped_myfunc('args to wrapped', 4)
print()

# Wrap with decorator syntax
@simple_decorator
def decorated_myfunc(a, b):
    myfunc(a, b)
    return

show_details('decorated_myfunc', decorated_myfunc)
decorated_myfunc()
decorated_myfunc('args to decorated', 4)
```
functools提供了一个装饰器，wrapps（），它将update_wrapper（）应用于修饰函数。

```
$ python3 functools_wraps.py

myfunc:
  object: <function myfunc at 0x101241b70>
  __name__: myfunc
  __doc__ 'myfunc() is not complicated'

  myfunc: ('unwrapped, default b', 2)
  myfunc: ('unwrapped, passing b', 3)

wrapped_myfunc:
  object: <function myfunc at 0x1012e62f0>
  __name__: myfunc
  __doc__ 'myfunc() is not complicated'

  decorated: ('decorated defaults', 1)
     myfunc: ('decorated defaults', 1)
  decorated: ('args to wrapped', 4)
     myfunc: ('args to wrapped', 4)

decorated_myfunc:
  object: <function decorated_myfunc at 0x1012e6400>
  __name__: decorated_myfunc
  __doc__ None

  decorated: ('decorated defaults', 1)
     myfunc: ('decorated defaults', 1)
  decorated: ('args to decorated', 4)
     myfunc: ('args to decorated', 4)
```

### Comparison

在Python 2下，类可以定义一个`__cmp __（）`方法，该方法根据对象是否小于，等于或大于要比较的项而返回-1,0或1。 Python 2.1引入了丰富的比较方法API（`__lt__（）`，`__le__（）`，`__eq__（）`，`__ne__（）`，`__gt__（）`和`__ge__（）`），它们执行单个比较操作并返回一个布尔值。 Python 3中弃用`__cmp__（）`，来赞成这些新方法，而且functools提供了一些工具，可以更容易地编写符合Python 3中新的比较要求的类。

#### 丰富的比较

丰富的比较API旨在允许具有复杂比较的类以尽可能最有效的方式实现每个测试。 但是，对于比较相对简单的类，手动创建每个丰富的比较方法是没有意义的。 total_ordering（）类装饰器接受一个提供某些方法的类，并添加其余的方法。

```python
#functools_total_ordering.py
import functools
import inspect
from pprint import pprint

@functools.total_ordering
class MyObject:

    def __init__(self, val):
        self.val = val

    def __eq__(self, other):
        print('  testing __eq__({}, {})'.format(
            self.val, other.val))
        return self.val == other.val

    def __gt__(self, other):
        print('  testing __gt__({}, {})'.format(
            self.val, other.val))
        return self.val > other.val

print('Methods:\n')
pprint(inspect.getmembers(MyObject, inspect.isfunction))

a = MyObject(1)
b = MyObject(2)

print('\nComparisons:')
for expr in ['a < b', 'a <= b', 'a == b', 'a >= b', 'a > b']:
    print('\n{:<6}:'.format(expr))
    result = eval(expr)
    print('  result of {}: {}'.format(expr, result))
```
该类必须提供`__eq__（）`和另一个丰富的比较方法的实现。 装饰器通过使用提供的比较方法添加其他方法的实现。 如果无法进行比较，则该方法应返回NotImplemented，以便在完全失败之前使用另一个对象上的反向比较运算符尝试比较。

```
$ python3 functools_total_ordering.py

Methods:

[('__eq__', <function MyObject.__eq__ at 0x10139a488>),
 ('__ge__', <function _ge_from_gt at 0x1012e2510>),
 ('__gt__', <function MyObject.__gt__ at 0x10139a510>),
 ('__init__', <function MyObject.__init__ at 0x10139a400>),
 ('__le__', <function _le_from_gt at 0x1012e2598>),
 ('__lt__', <function _lt_from_gt at 0x1012e2488>)]

Comparisons:

a < b :
  testing __gt__(1, 2)
  testing __eq__(1, 2)
  result of a < b: True

a <= b:
  testing __gt__(1, 2)
  result of a <= b: True

a == b:
  testing __eq__(1, 2)
  result of a == b: False

a >= b:
  testing __gt__(1, 2)
  testing __eq__(1, 2)
  result of a >= b: False

a > b :
  testing __gt__(1, 2)
  result of a > b: False
```

#### Collation Order

由于旧式比较函数在Python 3中已弃用，因此不再支持sort（）等函数的cmp参数。 使用比较函数的旧程序可以使用`cmp_to_key（）`将它们转换为返回归类键(collation key)的函数，该归类键用于确定在最终序列中的位置。

```python
#functools_cmp_to_key.py
import functools

class MyObject:

    def __init__(self, val):
        self.val = val

    def __str__(self):
        return 'MyObject({})'.format(self.val)

def compare_obj(a, b):
    """Old-style comparison function.
    """
    print('comparing {} and {}'.format(a, b))
    if a.val < b.val:
        return -1
    elif a.val > b.val:
        return 1
    return 0

# Make a key function using cmp_to_key()
get_key = functools.cmp_to_key(compare_obj)

def get_key_wrapper(o):
    "Wrapper function for get_key to allow for print statements."
    new_key = get_key(o)
    print('key_wrapper({}) -> {!r}'.format(o, new_key))
    return new_key

objs = [MyObject(x) for x in range(5, 0, -1)]

for o in sorted(objs, key=get_key_wrapper):
    print(o)
```
通常，`cmp_to_key（）`将直接使用，但在此示例中，引入了额外的包装函数，以便在调用key函数时打印出更多信息。
输出显示sorted（）通过为序列中的每个项调用`get_key_wrapper（）`来生成key。 `cmp_to_key（）`返回的key是functools中定义的类的实例，它使用传入的旧式比较函数实现丰富的比较API。创建所有key后，通过比较key对序列进行排序。


```
$ python3 functools_cmp_to_key.py

key_wrapper(MyObject(5)) -> <functools.KeyWrapper object at 0x1011c5530>
key_wrapper(MyObject(4)) -> <functools.KeyWrapper object at 0x1011c5510>
key_wrapper(MyObject(3)) -> <functools.KeyWrapper object at 0x1011c54f0>
key_wrapper(MyObject(2)) -> <functools.KeyWrapper object at 0x1011c5390>
key_wrapper(MyObject(1)) -> <functools.KeyWrapper object at 0x1011c5710>
comparing MyObject(4) and MyObject(5)
comparing MyObject(3) and MyObject(4)
comparing MyObject(2) and MyObject(3)
comparing MyObject(1) and MyObject(2)
MyObject(1)
MyObject(2)
MyObject(3)
MyObject(4)
MyObject(5)
```

### Caching

`lru_cache（）`装饰器将函数包装在最近最少使用的缓存中。 这个函数的参数用于构建散列键，然后映射到结果。 具有相同参数的后续调用将从缓存中获取值而不是调用该函数。 装饰器还向函数添加方法以检查缓存的状态（`cache_info（）`）并且清空缓存（`cache_clear（）`）。

```python
#functools_lru_cache.py
import functools

@functools.lru_cache()
def expensive(a, b):
    print('expensive({}, {})'.format(a, b))
    return a * b

MAX = 2

print('First set of calls:')
for i in range(MAX):
    for j in range(MAX):
        expensive(i, j)
print(expensive.cache_info())

print('\nSecond set of calls:')
for i in range(MAX + 1):
    for j in range(MAX + 1):
        expensive(i, j)
print(expensive.cache_info())

print('\nClearing cache:')
expensive.cache_clear()
print(expensive.cache_info())

print('\nThird set of calls:')
for i in range(MAX):
    for j in range(MAX):
        expensive(i, j)
print(expensive.cache_info())
```
这个例子在一组嵌套循环中多次调用expensive（）。 第二次使用相同的值进行调用时，结果将出现在缓存中。 清除缓存并再次运行循环时，必须重新计算值。

```
$ python3 functools_lru_cache.py

First set of calls:
expensive(0, 0)
expensive(0, 1)
expensive(1, 0)
expensive(1, 1)
CacheInfo(hits=0, misses=4, maxsize=128, currsize=4)

Second set of calls:
expensive(0, 2)
expensive(1, 2)
expensive(2, 0)
expensive(2, 1)
expensive(2, 2)
CacheInfo(hits=4, misses=9, maxsize=128, currsize=9)

Clearing cache:
CacheInfo(hits=0, misses=0, maxsize=128, currsize=0)

Third set of calls:
expensive(0, 0)
expensive(0, 1)
expensive(1, 0)
expensive(1, 1)
CacheInfo(hits=0, misses=4, maxsize=128, currsize=4)
```
为了防止缓存在长时间运行的进程中无限制地增长，它被赋予最大大小。 默认值为128个条目，但可以为每个缓存使用maxsize参数更改。

```python
#functools_lru_cache_expire.py
import functools

@functools.lru_cache(maxsize=2)
def expensive(a, b):
    print('called expensive({}, {})'.format(a, b))
    return a * b

def make_call(a, b):
    print('({}, {})'.format(a, b), end=' ')
    pre_hits = expensive.cache_info().hits
    expensive(a, b)
    post_hits = expensive.cache_info().hits
    if post_hits > pre_hits:
        print('cache hit')

print('Establish the cache')
make_call(1, 2)
make_call(2, 3)

print('\nUse cached items')
make_call(1, 2)
make_call(2, 3)

print('\nCompute a new value, triggering cache expiration')
make_call(3, 4)

print('\nCache still contains one old item')
make_call(2, 3)

print('\nOldest item needs to be recomputed')
make_call(1, 2)
```
在此示例中，高速缓存大小设置为2个条目。 当使用第三组唯一参数（3,4）时，缓存中最旧的项将被删除并替换为新结果。

```
$ python3 functools_lru_cache_expire.py

Establish the cache
(1, 2) called expensive(1, 2)
(2, 3) called expensive(2, 3)

Use cached items
(1, 2) cache hit
(2, 3) cache hit

Compute a new value, triggering cache expiration
(3, 4) called expensive(3, 4)

Cache still contains one old item
(2, 3) cache hit

Oldest item needs to be recomputed
(1, 2) called expensive(1, 2)
```
由lru_cache（）管理的缓存的键必须是可清除的，因此用于缓存查找的函数的所有参数都必须是可清除的。

```python
#functools_lru_cache_arguments.py
import functools

@functools.lru_cache(maxsize=2)
def expensive(a, b):
    print('called expensive({}, {})'.format(a, b))
    return a * b

def make_call(a, b):
    print('({}, {})'.format(a, b), end=' ')
    pre_hits = expensive.cache_info().hits
    expensive(a, b)
    post_hits = expensive.cache_info().hits
    if post_hits > pre_hits:
        print('cache hit')

make_call(1, 2)

try:
    make_call([1], 2)
except TypeError as err:
    print('ERROR: {}'.format(err))

try:
    make_call(1, {'2': 'two'})
except TypeError as err:
    print('ERROR: {}'.format(err))
```
如果将任何无法散列的对象传入函数，则会引发TypeError。

```
$ python3 functools_lru_cache_arguments.py

(1, 2) called expensive(1, 2)
([1], 2) ERROR: unhashable type: 'list'
(1, {'2': 'two'}) ERROR: unhashable type: 'dict'
```

### Reducing a Data Set

reduce（）函数将可调用对象和数据序列作为输入并生成单个值作为输出，它使用序列中的值调用callable并累积结果输出。

```python
#functools_reduce.py
import functools

def do_reduce(a, b):
    print('do_reduce({}, {})'.format(a, b))
    return a + b

data = range(1, 5)
print(data)
result = functools.reduce(do_reduce, data)
print('result: {}'.format(result))
```
此示例将输入序列中的数字相加。

```
$ python3 functools_reduce.py

range(1, 5)
do_reduce(1, 2)
do_reduce(3, 3)
do_reduce(6, 4)
result: 10
```
可选的initializer参数被放在序列的前面，并与其他项一起处理。 这可用于使用新输入更新先前计算的值。

```python
#functools_reduce_initializer.py
import functools

def do_reduce(a, b):
    print('do_reduce({}, {})'.format(a, b))
    return a + b

data = range(1, 5)
print(data)
result = functools.reduce(do_reduce, data, 99)
print('result: {}'.format(result))
```
在此示例中，先前的99之和用于初始化reduce（）计算的值。

```
$ python3 functools_reduce_initializer.py

range(1, 5)
do_reduce(99, 1)
do_reduce(100, 2)
do_reduce(102, 3)
do_reduce(105, 4)
result: 109
```
当没有initializer时，具有单个值的序列会自动reduce到该值。 除非提供initializer，否则空列表会生成错误。

```python
#functools_reduce_short_sequences.py
import functools

def do_reduce(a, b):
    print('do_reduce({}, {})'.format(a, b))
    return a + b

print('Single item in sequence:',
      functools.reduce(do_reduce, [1]))

print('Single item in sequence with initializer:',
      functools.reduce(do_reduce, [1], 99))

print('Empty sequence with initializer:',
      functools.reduce(do_reduce, [], 99))

try:
    print('Empty sequence:', functools.reduce(do_reduce, []))
except TypeError as err:
    print('ERROR: {}'.format(err))
```
因为initializer参数用作默认值，但如果输入序列不为空，也会与新值组合，因此请务必仔细考虑是否使用它。 如果将默认值与新值组合起来没有意义，最好捕获TypeError而不是传递初始值设定项。

```
$ python3 functools_reduce_short_sequences.py

Single item in sequence: 1
do_reduce(99, 1)
Single item in sequence with initializer: 100
Empty sequence with initializer: 99
ERROR: reduce() of empty sequence with no initial value
```

### Generic Functions

在像Python这样的动态类型语言中，通常需要根据参数的类型执行稍微不同的操作，尤其是在处理项目列表和单个项目之间的差异时。 直接检查参数的类型很简单，但是在“行为差异可以被分离成单独的函数”的情况下，functools提供singledispatch（）装饰器来注册一组generic functions，用于根据第一个参数的类型自动切换函数。

```python
#functools_singledispatch.py
import functools

@functools.singledispatch
def myfunc(arg):
    print('default myfunc({!r})'.format(arg))

@myfunc.register(int)
def myfunc_int(arg):
    print('myfunc_int({})'.format(arg))

@myfunc.register(list)
def myfunc_list(arg):
    print('myfunc_list()')
    for item in arg:
        print('  {}'.format(item))

myfunc('string argument')
myfunc(1)
myfunc(2.3)
myfunc(['a', 'b', 'c'])
```
新函数的register（）属性用作注册替代实现的另一个装饰器。 如果没有找到其他特定于类型的函数，则使用singledispatch（）包装的第一个函数是默认实现，如本示例中的float情况。

```
$ python3 functools_singledispatch.py

default myfunc('string argument')
myfunc_int(1)
default myfunc(2.3)
myfunc_list()
  a
  b
  c
```
如果未找到类型的完全匹配，则评估继承顺序并使用最接近的匹配类型。

```python
#functools_singledispatch_mro.py
import functools

class A:
    pass

class B(A):
    pass

class C(A):
    pass

class D(B):
    pass

class E(C, D):
    pass

@functools.singledispatch
def myfunc(arg):
    print('default myfunc({})'.format(arg.__class__.__name__))

@myfunc.register(A)
def myfunc_A(arg):
    print('myfunc_A({})'.format(arg.__class__.__name__))

@myfunc.register(B)
def myfunc_B(arg):
    print('myfunc_B({})'.format(arg.__class__.__name__))

@myfunc.register(C)
def myfunc_C(arg):
    print('myfunc_C({})'.format(arg.__class__.__name__))

myfunc(A())
myfunc(B())
myfunc(C())
myfunc(D())
myfunc(E())
```
在此示例中，类D和E与任何已注册的泛型函数都不完全匹配，所选的函数取决于类层次结构。

```
$ python3 functools_singledispatch_mro.py

myfunc_A(A)
myfunc_B(B)
myfunc_C(C)
myfunc_B(D)
myfunc_C(E)
```

## itertools — Iterator Functions

目的：itertools模块包括一组用于处理序列数据集的函数。  
itertools提供的功能受到Clojure，Haskell，APL和SML等函数式编程语言的类似特性的启发。它们旨在快速并有效地使用内存，并且还要连接在一起以表达更复杂的基于迭代的算法。  
与使用列表的代码相比，基于迭代器的代码提供了更好的内存消耗特性。由于在需要之前不会从迭代器生成数据，因此不需要将所有数据同时存储在内存中。 这种“懒惰”处理模型可以减少大数据集的交换和其他副作用，从而提高性能。  
除了itertools中定义的函数之外，本节中的示例还依赖于一些用于迭代的内置函数。

### Merging and Splitting Iterators

chain（）函数将几个迭代器作为参数，并返回一个迭代器，它生成所有输入的内容，就像来自单个迭代器一样。

```python
#itertools_chain.py
from itertools import *

for i in chain([1, 2, 3], ['a', 'b', 'c']):
    print(i, end=' ')
print()
```
chain（）可以轻松处理多个序列而无需构建一个大型列表。

```
$ python3 itertools_chain.py

1 2 3 a b c
```
如果要组合的可迭代对象不是事先都知道，或者需要懒惰地进行评估，则可以使用chain.from_iterable（）来构造链。

```python
#itertools_chain_from_iterable.py
from itertools import *

def make_iterables_to_chain():
    yield [1, 2, 3]
    yield ['a', 'b', 'c']

for i in chain.from_iterable(make_iterables_to_chain()):
    print(i, end=' ')
print()
```
```
$ python3 itertools_chain_from_iterable.py

1 2 3 a b c
```
内置函数zip（）返回一个迭代器，它将几个迭代器的元素组合成元组。

```python
#itertools_zip.py
for i in zip([1, 2, 3], ['a', 'b', 'c']):
    print(i)
```
与此模块中的其他函数一样，返回值是一个可迭代的对象，它一次生成一个值。

```
$ python3 itertools_zip.py

(1, 'a')
(2, 'b')
(3, 'c')
```
zip（）在第一个输入迭代器耗尽时停止。 要处理所有输入，即使迭代器产生不同数量的值，请使用zip_longest（）。

```python
#itertools_zip_longest.py
from itertools import *

r1 = range(3)
r2 = range(2)

print('zip stops early:')
print(list(zip(r1, r2)))

r1 = range(3)
r2 = range(2)

print('\nzip_longest processes all of the values:')
print(list(zip_longest(r1, r2)))
```
默认情况下，zip_longest（）将None替换为任何缺失值。 使用fillvalue参数可以使用不同的替换值。

```
$ python3 itertools_zip_longest.py

zip stops early:
[(0, 0), (1, 1)]

zip_longest processes all of the values:
[(0, 0), (1, 1), (2, None)]
```
islice（）函数返回一个迭代器，它通过索引从输入迭代器返回所选项。

```python
#itertools_islice.py
from itertools import *

print('Stop at 5:')
for i in islice(range(100), 5):
    print(i, end=' ')
print('\n')

print('Start at 5, Stop at 10:')
for i in islice(range(100), 5, 10):
    print(i, end=' ')
print('\n')

print('By tens to 100:')
for i in islice(range(100), 0, 100, 10):
    print(i, end=' ')
print('\n')
```
islice（）采用与列表的切片运算符相同的参数：start，stop和step。 start和step参数是可选的。

```
$ python3 itertools_islice.py

Stop at 5:
0 1 2 3 4

Start at 5, Stop at 10:
5 6 7 8 9

By tens to 100:
0 10 20 30 40 50 60 70 80 90
```
tee（）函数根据单个原始输入返回几个独立的迭代器（默认为2）。

```python
#itertools_tee.py
from itertools import *

r = islice(count(), 5)
i1, i2 = tee(r)

print('i1:', list(i1))
print('i2:', list(i2))
```
tee（）具有类似于Unix tee实用程序的语义，它重复从输入读取的值并将它们写入命名文件和标准输出。 tee（）返回的迭代器可用于将同一组数据提供给多个算法，以便并行处理。

```
$ python3 itertools_tee.py

i1: [0, 1, 2, 3, 4]
i2: [0, 1, 2, 3, 4]
```
由tee（）创建的新迭代器共享它们的输入，因此在创建新迭代器之后不应使用原始迭代器。

```python
#itertools_tee_error.py
from itertools import *

r = islice(count(), 5)
i1, i2 = tee(r)

print('r:', end=' ')
for i in r:
    print(i, end=' ')
    if i > 1:
        break
print()

print('i1:', list(i1))
print('i2:', list(i2))
```
如果从原始输入中消耗了值，则新迭代器将不会生成这些值：

```
$ python3 itertools_tee_error.py

r: 0 1 2
i1: [3, 4]
i2: [3, 4]
```

### Converting Inputs

内置的map（）函数返回一个迭代器，它使用输入迭代器中的值调用一个函数，并返回结果。 当任何输入迭代器耗尽时它会停止。

```python
#itertools_map.py

def times_two(x):
    return 2 * x

def multiply(x, y):
    return (x, y, x * y)

print('Doubles:')
for i in map(times_two, range(5)):
    print(i)

print('\nMultiples:')
r1 = range(5)
r2 = range(5, 10)
for i in map(multiply, r1, r2):
    print('{:d} * {:d} = {:d}'.format(*i))

print('\nStopping:')
r1 = range(5)
r2 = range(2)
for i in map(multiply, r1, r2):
    print(i)
```
在第一个示例中，lambda函数将输入值乘以2.在第二个示例中，lambda函数将两个参数相乘，取自单独的迭代器，并返回具有原始参数和计算值的元组。 第三个示例在生成两个元组后停止，因为第二个范围已用完。

```
$ python3 itertools_map.py

Doubles:
0
2
4
6
8

Multiples:
0 * 5 = 0
1 * 6 = 6
2 * 7 = 14
3 * 8 = 24
4 * 9 = 36

Stopping:
(0, 0, 0)
(1, 1, 1)
```
starmap（）函数类似于map（），但它不是从多个迭代器构造元组，而是使用 `*` 语法将单个迭代器中的项目拆分为映射函数的参数。

```python
#itertools_starmap.py
from itertools import *

values = [(0, 5), (1, 6), (2, 7), (3, 8), (4, 9)]

for i in starmap(lambda x, y: (x, y, x * y), values):
    print('{} * {} = {}'.format(*i))
```
在`map（）`的映射函数被称为`f（i1，i2）`的情况下，传递给`starmap（）`的映射函数称为`f（* i）`。

```
$ python3 itertools_starmap.py

0 * 5 = 0
1 * 6 = 6
2 * 7 = 14
3 * 8 = 24
4 * 9 = 36
```

### Producing New Values

count（）函数返回一个无限期生成连续整数的迭代器。 第一个数字可以作为参数传递（默认值为零）。 没有上限参数（有关对结果集的更多控制，请参阅内置range（））。

```python
#itertools_count.py
from itertools import *

for i in zip(count(1), ['a', 'b', 'c']):
    print(i)
```
此示例因为消耗了list参数而停止。

```
$ python3 itertools_count.py

(1, 'a')
(2, 'b')
(3, 'c')
```
count（）的start和step参数可以是可以一起添加的任何数值。

```python
#itertools_count_step.py
import fractions
from itertools import *

start = fractions.Fraction(1, 3)
step = fractions.Fraction(1, 3)

for i in zip(count(start, step), ['a', 'b', 'c']):
    print('{}: {}'.format(*i))
```
在此示例中，start和step是fraction模块中的Fraction对象。

```
$ python3 itertools_count_step.py

1/3: a
2/3: b
1: c
```
cycle（）函数返回一个迭代器，它无限期重复参数内容。 由于它必须记住输入迭代器的全部内容，如果迭代器很长，它可能会消耗相当多的内存。

```python
#itertools_cycle.py
from itertools import *

for i in zip(range(7), cycle(['a', 'b', 'c'])):
    print(i)
```
在此示例中，计数器变量用于在几个循环之后中断循环。

```
$ python3 itertools_cycle.py

(0, 'a')
(1, 'b')
(2, 'c')
(3, 'a')
(4, 'b')
(5, 'c')
(6, 'a')
```
repeat（）函数返回一个迭代器，每次访问它时都会产生相同的值。

```python
#itertools_repeat.py
from itertools import *

for i in repeat('over-and-over', 5):
    print(i)
```
repeat（）返回的迭代器会永远保持返回数据，除非提供可选的times参数来限制它。

```
$ python3 itertools_repeat.py

over-and-over
over-and-over
over-and-over
over-and-over
over-and-over
```
当常量值与其他迭代器的值一起包含时，将repeat（）与zip（）或map（）结合起来很有用。

```python
#itertools_repeat_zip.py
from itertools import *

for i, s in zip(count(), repeat('over-and-over', 5)):
    print(i, s)
```
此示例中计数器值与repeat（）返回的常量组合。

```
$ python3 itertools_repeat_zip.py

0 over-and-over
1 over-and-over
2 over-and-over
3 over-and-over
4 over-and-over
```
此示例使用map（）将0到4范围内的数字乘以2。

```python
#itertools_repeat_map.py
from itertools import *

for i in map(lambda x, y: (x, y, x * y), repeat(2), range(5)):
    print('{:d} * {:d} = {:d}'.format(*i))
```
repeat（）迭代器不需要显式限制，因为map（）在任何输入结束时停止处理，而range（）只返回五个元素。

```
$ python3 itertools_repeat_map.py

2 * 0 = 0
2 * 1 = 2
2 * 2 = 4
2 * 3 = 6
2 * 4 = 8
```

### Filtering

dropwhile（）函数返回一个迭代器，它在条件第一次变为false后生成输入迭代器的元素。

```python
#itertools_dropwhile.py
from itertools import *

def should_drop(x):
    print('Testing:', x)
    return x < 1

for i in dropwhile(should_drop, [-1, 0, 1, 2, -2]):
    print('Yielding:', i)
```
dropwhile（）不会过滤输入的每个元素; 在第一次条件为假之后，返回输入中的所有剩余元素。

```
$ python3 itertools_dropwhile.py

Testing: -1
Testing: 0
Testing: 1
Yielding: 1
Yielding: 2
Yielding: -2
```
dropwhile（）的反义词是takewhile（）。 它返回一个迭代器，只要测试函数返回true，它就会从输入迭代器返回项。

```python
#itertools_takewhile.py
from itertools import *

def should_take(x):
    print('Testing:', x)
    return x < 2

for i in takewhile(should_take, [-1, 0, 1, 2, -2]):
    print('Yielding:', i)
```
只要should_take（）返回False，takewhile（）就会停止处理输入。

```
$ python3 itertools_takewhile.py

Testing: -1
Yielding: -1
Testing: 0
Yielding: 0
Testing: 1
Yielding: 1
Testing: 2
```
内置函数filter（）返回一个迭代器，该迭代器仅包含测试函数返回true的项。

```python
#itertools_filter.py
from itertools import *

def check_item(x):
    print('Testing:', x)
    return x < 1

for i in filter(check_item, [-1, 0, 1, 2, -2]):
    print('Yielding:', i)
```
filter（）与dropwhile（）和takewhile（）的不同之处在于每个项目在返回之前都经过测试。

```
$ python3 itertools_filter.py

Testing: -1
Yielding: -1
Testing: 0
Yielding: 0
Testing: 1
Testing: 2
Testing: -2
Yielding: -2
```
filterfalse（）返回一个迭代器，该迭代器仅包含test函数返回false的项。

```python
#itertools_filterfalse.py
from itertools import *

def check_item(x):
    print('Testing:', x)
    return x < 1

for i in filterfalse(check_item, [-1, 0, 1, 2, -2]):
    print('Yielding:', i)
```
check_item（）中的测试表达式是相同的，因此使用filterfalse（）的此示例中的结果与前一示例的结果相反。

```
$ python3 itertools_filterfalse.py

Testing: -1
Testing: 0
Testing: 1
Yielding: 1
Testing: 2
Yielding: 2
Testing: -2
```
compress（）提供了另一种过滤迭代内容的方法。 它不是调用函数，而是使用另一个iterable中的值来指示何时接受值以及何时忽略它。

```python
#itertools_compress.py
from itertools import *

every_third = cycle([False, False, True])
data = range(1, 10)

for i in compress(data, every_third):
    print(i, end=' ')
print()
```
第一个参数是可迭代处理的数据，第二个参数是可迭代的选择器，它产生布尔值，指示从数据输入中取出哪些元素（true值导致生成值，false值导致忽略它）。

```
$ python3 itertools_compress.py

3 6 9
```

### Grouping Data

groupby（）函数返回一个迭代器，该迭代器生成由公共key组织的值集。 此示例说明了基于一个属性对相关值进行分组。

```python
#itertools_groupby_seq.py
import functools
from itertools import *
import operator
import pprint

@functools.total_ordering
class Point:

    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __repr__(self):
        return '({}, {})'.format(self.x, self.y)

    def __eq__(self, other):
        return (self.x, self.y) == (other.x, other.y)

    def __gt__(self, other):
        return (self.x, self.y) > (other.x, other.y)

# Create a dataset of Point instances
data = list(map(Point,
                cycle(islice(count(), 3)),
                islice(count(), 7)))
print('Data:')
pprint.pprint(data, width=35)
print()

# Try to group the unsorted data based on X values
print('Grouped, unsorted:')
for k, g in groupby(data, operator.attrgetter('x')):
    print(k, list(g))
print()

# Sort the data
data.sort()
print('Sorted:')
pprint.pprint(data, width=35)
print()

# Group the sorted data based on X values
print('Grouped, sorted:')
for k, g in groupby(data, operator.attrgetter('x')):
    print(k, list(g))
print()
```
输入序列需要按键值排序，以便分组按预期工作。

```
$ python3 itertools_groupby_seq.py

Data:
[(0, 0),
 (1, 1),
 (2, 2),
 (0, 3),
 (1, 4),
 (2, 5),
 (0, 6)]

Grouped, unsorted:
0 [(0, 0)]
1 [(1, 1)]
2 [(2, 2)]
0 [(0, 3)]
1 [(1, 4)]
2 [(2, 5)]
0 [(0, 6)]

Sorted:
[(0, 0),
 (0, 3),
 (0, 6),
 (1, 1),
 (1, 4),
 (2, 2),
 (2, 5)]

Grouped, sorted:
0 [(0, 0), (0, 3), (0, 6)]
1 [(1, 1), (1, 4)]
2 [(2, 2), (2, 5)]
```

### Combining Inputs

accumulate（）函数处理输入iterable，将第n和第n + 1项传递给函数并生成返回值代替其中一个输入。 用于组合这两个值的默认函数会add他们，因此accumulate（）可用于生成一系列数字输入的累积和。

```python
#itertools_accumulate.py
from itertools import *

print(list(accumulate(range(5))))
print(list(accumulate('abcde')))
```
当与一系列非整数值一起使用时，结果取决于将两个项目“add”在一起的含义。 此脚本中的第二个示例显示，当accumulate（）接收到字符串输入时，每个响应都是该字符串的逐渐加长的前缀。

```
$ python3 itertools_accumulate.py

[0, 1, 3, 6, 10]
['a', 'ab', 'abc', 'abcd', 'abcde']
```
可以将accumulate（）与采用两个输入值的任何其他函数组合以实现不同的结果。

```python
#itertools_accumulate_custom.py
from itertools import *

def f(a, b):
    print(a, b)
    return b + a + b

print(list(accumulate('abcde', f)))
```
此示例以一系列（无意义的）回文结合的方式组合字符串值。 调用f（）时的每一步，它都会打印accumulate（）传递给它的输入值。

```
$ python3 itertools_accumulate_custom.py

a b
bab c
cbabc d
dcbabcd e
['a', 'bab', 'cbabc', 'dcbabcd', 'edcbabcde']
```
迭代多个序列的嵌套for循环通常可以用product（）替换，product（）产生一个可迭代对象，其值是输入值集的笛卡尔乘积。

```python
#itertools_product.py
from itertools import *
import pprint

FACE_CARDS = ('J', 'Q', 'K', 'A')
SUITS = ('H', 'D', 'C', 'S')

DECK = list(
    product(
        chain(range(2, 11), FACE_CARDS),
        SUITS,
    )
)

for card in DECK:
    print('{:>2}{}'.format(*card), end=' ')
    if card[1] == SUITS[-1]:
        print()
```
product（）生成的值是元组，其成员按传入顺序从每个作为参数传入的迭代中获取。 返回的第一个元组包含每个迭代的第一个值。 传递给product（）的最后一个iterable首先处理，然后是倒数第二个，依此类推。 结果是返回值基于第一个迭代，然后是下一个迭代，等等。

在此示例中，cards按值排序，然后按suit排序。

```
$ python3 itertools_product.py

 2H  2D  2C  2S
 3H  3D  3C  3S
 4H  4D  4C  4S
 5H  5D  5C  5S
 6H  6D  6C  6S
 7H  7D  7C  7S
 8H  8D  8C  8S
 9H  9D  9C  9S
10H 10D 10C 10S
 JH  JD  JC  JS
 QH  QD  QC  QS
 KH  KD  KC  KS
 AH  AD  AC  AS
```
要更改cards的顺序，请更改product（）参数的顺序。

```python
#itertools_product_ordering.py
from itertools import *
import pprint

FACE_CARDS = ('J', 'Q', 'K', 'A')
SUITS = ('H', 'D', 'C', 'S')

DECK = list(
    product(
        SUITS,
        chain(range(2, 11), FACE_CARDS),
    )
)

for card in DECK:
    print('{:>2}{}'.format(card[1], card[0]), end=' ')
    if card[1] == FACE_CARDS[-1]:
        print()
```
此示例中的打印循环查找Ace卡而不是spade suit，然后添加换行符以分解输出。

```
$ python3 itertools_product_ordering.py

 2H  3H  4H  5H  6H  7H  8H  9H 10H  JH  QH  KH  AH
 2D  3D  4D  5D  6D  7D  8D  9D 10D  JD  QD  KD  AD
 2C  3C  4C  5C  6C  7C  8C  9C 10C  JC  QC  KC  AC
 2S  3S  4S  5S  6S  7S  8S  9S 10S  JS  QS  KS  AS
```
要计算序列与自身的product，请指定输入应重复的次数。

```python
#itertools_product_repeat.py
from itertools import *

def show(iterable):
    for i, item in enumerate(iterable, 1):
        print(item, end=' ')
        if (i % 3) == 0:
            print()
    print()

print('Repeat 2:\n')
show(list(product(range(3), repeat=2)))

print('Repeat 3:\n')
show(list(product(range(3), repeat=3)))
```
由于重复单个iterable就像多次传递相同的iterable一样，product（）生成的每个元组将包含一些等于repeat计数器的项。

```
$ python3 itertools_product_repeat.py

Repeat 2:

(0, 0) (0, 1) (0, 2)
(1, 0) (1, 1) (1, 2)
(2, 0) (2, 1) (2, 2)

Repeat 3:

(0, 0, 0) (0, 0, 1) (0, 0, 2)
(0, 1, 0) (0, 1, 1) (0, 1, 2)
(0, 2, 0) (0, 2, 1) (0, 2, 2)
(1, 0, 0) (1, 0, 1) (1, 0, 2)
(1, 1, 0) (1, 1, 1) (1, 1, 2)
(1, 2, 0) (1, 2, 1) (1, 2, 2)
(2, 0, 0) (2, 0, 1) (2, 0, 2)
(2, 1, 0) (2, 1, 1) (2, 1, 2)
(2, 2, 0) (2, 2, 1) (2, 2, 2)
```
permutations（）函数从输入iterable中产生以给定长度组合的可能的排列项。 它默认生成所有排列的完整集合。

```python
#itertools_permutations.py
from itertools import *

def show(iterable):
    first = None
    for i, item in enumerate(iterable, 1):
        if first != item[0]:
            if first is not None:
                print()
            first = item[0]
        print(''.join(item), end=' ')
    print()

print('All permutations:\n')
show(permutations('abcd'))

print('\nPairs:\n')
show(permutations('abcd', r=2))
```
使用r参数来限制返回的各个排列的长度和数量。

```
$ python3 itertools_permutations.py

All permutations:

abcd abdc acbd acdb adbc adcb
bacd badc bcad bcda bdac bdca
cabd cadb cbad cbda cdab cdba
dabc dacb dbac dbca dcab dcba

Pairs:

ab ac ad
ba bc bd
ca cb cd
da db dc
```
要将值限制为唯一组合而不是排列，请使用combinations（）。 只要输入的成员是唯一的，输出就不会包含任何重复的值。

```python
#itertools_combinations.py
from itertools import *

def show(iterable):
    first = None
    for i, item in enumerate(iterable, 1):
        if first != item[0]:
            if first is not None:
                print()
            first = item[0]
        print(''.join(item), end=' ')
    print()

print('Unique pairs:\n')
show(combinations('abcd', r=2))
```
与排列不同，组合（）的r参数是必需的。

```
$ python3 itertools_combinations.py

Unique pairs:

ab ac ad
bc bd
cd
```
虽然`combination（）`不重复单个输入元素，但有时考虑包含重复元素的组合是有用的。 对于这些情况，请使用`combination_with_replacement（）`。

```python
#itertools_combinations_with_replacement.py
from itertools import *

def show(iterable):
    first = None
    for i, item in enumerate(iterable, 1):
        if first != item[0]:
            if first is not None:
                print()
            first = item[0]
        print(''.join(item), end=' ')
    print()

print('Unique pairs:\n')
show(combinations_with_replacement('abcd', r=2))
```
在此输出中，每个输入项与其自身以及输入序列的所有其他成员配对。

```
$ python3 itertools_combinations_with_replacement.py

Unique pairs:

aa ab ac ad
bb bc bd
cc cd
dd
```

## operator — Functional Interface to Built-in Operators

目的：内置运算符的功能接口。  
使用迭代器进行编程有时需要为简单表达式创建小函数。 有时，这些可以作为lambda函数实现，但对于某些操作，根本不需要新函数。 operator模块定义“与算术、比较的内置操作相对应的”函数，和“对应于标准对象API的其他操作”相对应的函数。

### Logical Operations

有一些函数可以确定一个值的布尔等价值，否定它来创建相反的布尔值，并比较对象以查看它们是否相同。

```python
#operator_boolean.py
from operator import *

a = -1
b = 5

print('a =', a)
print('b =', b)
print()

print('not_(a)     :', not_(a))
print('truth(a)    :', truth(a))
print('is_(a, b)   :', is_(a, b))
print('is_not(a, b):', is_not(a, b))
```
`not_（）`包括尾随下划线，因为not是Python关键字。 `truth（）`应用的逻辑“与在if语句中测试一个表达式或将表达式转换为bool时使用的”相同。 `is_（）`实现的检查与is关键字使用的相同，`is_not（）`执行相同的测试并返回相反的答案。

```
$ python3 operator_boolean.py

a = -1
b = 5

not_(a)     : False
truth(a)    : True
is_(a, b)   : False
is_not(a, b): True
```

### Comparison Operators

支持所有丰富的比较运算符。

```python
#operator_comparisons.py
from operator import *

a = 1
b = 5.0

print('a =', a)
print('b =', b)
for func in (lt, le, eq, ne, ge, gt):
    print('{}(a, b): {}'.format(func.__name__, func(a, b)))
```
这些函数等效于使用`<，<=，==，>=和>`的表达式语法。

```
$ python3 operator_comparisons.py

a = 1
b = 5.0
lt(a, b): True
le(a, b): True
eq(a, b): False
ne(a, b): True
ge(a, b): False
gt(a, b): False
```

### Arithmetic Operators

还支持用于操纵数值的算术运算符。

```python
#operator_math.py
from operator import *

a = -1
b = 5.0
c = 2
d = 6

print('a =', a)
print('b =', b)
print('c =', c)
print('d =', d)

print('\nPositive/Negative:')
print('abs(a):', abs(a))
print('neg(a):', neg(a))
print('neg(b):', neg(b))
print('pos(a):', pos(a))
print('pos(b):', pos(b))

print('\nArithmetic:')
print('add(a, b)     :', add(a, b))
print('floordiv(a, b):', floordiv(a, b))
print('floordiv(d, c):', floordiv(d, c))
print('mod(a, b)     :', mod(a, b))
print('mul(a, b)     :', mul(a, b))
print('pow(c, d)     :', pow(c, d))
print('sub(b, a)     :', sub(b, a))
print('truediv(a, b) :', truediv(a, b))
print('truediv(d, c) :', truediv(d, c))

print('\nBitwise:')
print('and_(c, d)  :', and_(c, d))
print('invert(c)   :', invert(c))
print('lshift(c, d):', lshift(c, d))
print('or_(c, d)   :', or_(c, d))
print('rshift(d, c):', rshift(d, c))
print('xor(c, d)   :', xor(c, d))
```
有两个独立的除法运算符：floordiv（）（在3.0之前的Python中实现的整数除法）和truediv（）（浮点除法）。

```
$ python3 operator_math.py

a = -1
b = 5.0
c = 2
d = 6

Positive/Negative:
abs(a): 1
neg(a): 1
neg(b): -5.0
pos(a): -1
pos(b): 5.0

Arithmetic:
add(a, b)     : 4.0
floordiv(a, b): -1.0
floordiv(d, c): 3
mod(a, b)     : 4.0
mul(a, b)     : -5.0
pow(c, d)     : 64
sub(b, a)     : 6.0
truediv(a, b) : -0.2
truediv(d, c) : 3.0

Bitwise:
and_(c, d)  : 2
invert(c)   : -3
lshift(c, d): 128
or_(c, d)   : 6
rshift(d, c): 1
xor(c, d)   : 4
```

### Sequence Operators

处理序列的运算符可以分为四组：构建序列，搜索项目，访问内容以及从序列中删除项目。

```python
#operator_sequences.py
from operator import *

a = [1, 2, 3]
b = ['a', 'b', 'c']

print('a =', a)
print('b =', b)

print('\nConstructive:')
print('  concat(a, b):', concat(a, b))

print('\nSearching:')
print('  contains(a, 1)  :', contains(a, 1))
print('  contains(b, "d"):', contains(b, "d"))
print('  countOf(a, 1)   :', countOf(a, 1))
print('  countOf(b, "d") :', countOf(b, "d"))
print('  indexOf(a, 5)   :', indexOf(a, 1))

print('\nAccess Items:')
print('  getitem(b, 1)                  :', getitem(b, 1))
print('  getitem(b, slice(1, 3))        :', getitem(b, slice(1, 3)))
print('  setitem(b, 1, "d")             :', end=' ')
setitem(b, 1, "d")
print(b)
print('  setitem(a, slice(1, 3), [4, 5]):', end=' ')
setitem(a, slice(1, 3), [4, 5])
print(a)

print('\nDestructive:')
print('  delitem(b, 1)          :', end=' ')
delitem(b, 1)
print(b)
print('  delitem(a, slice(1, 3)):', end=' ')
delitem(a, slice(1, 3))
print(a)
```
其中一些操作（如setitem（）和delitem（））会就地修改序列并且不返回值。

```
$ python3 operator_sequences.py

a = [1, 2, 3]
b = ['a', 'b', 'c']

Constructive:
  concat(a, b): [1, 2, 3, 'a', 'b', 'c']

Searching:
  contains(a, 1)  : True
  contains(b, "d"): False
  countOf(a, 1)   : 1
  countOf(b, "d") : 0
  indexOf(a, 5)   : 0

Access Items:
  getitem(b, 1)                  : b
  getitem(b, slice(1, 3))        : ['b', 'c']
  setitem(b, 1, "d")             : ['a', 'd', 'c']
  setitem(a, slice(1, 3), [4, 5]): [1, 4, 5]

Destructive:
  delitem(b, 1)          : ['a', 'c']
  delitem(a, slice(1, 3)): [1]
```

### In-place Operators

除了标准运算符之外，许多类型的对象还通过特殊运算符（如`+=`）支持“就地”修改。 也具有与就地修改一样功能的函数：

```python
#operator_inplace.py
from operator import *

a = -1
b = 5.0
c = [1, 2, 3]
d = ['a', 'b', 'c']
print('a =', a)
print('b =', b)
print('c =', c)
print('d =', d)
print()

a = iadd(a, b)
print('a = iadd(a, b) =>', a)
print()

c = iconcat(c, d)
print('c = iconcat(c, d) =>', c)
```
这些示例仅演示了一些功能。 有关完整的详细信息，请参阅标准库文档。

```
$ python3 operator_inplace.py

a = -1
b = 5.0
c = [1, 2, 3]
d = ['a', 'b', 'c']

a = iadd(a, b) => 4.0

c = iconcat(c, d) => [1, 2, 3, 'a', 'b', 'c']
```

### Attribute and Item “Getters”

operator模块最不寻常的特征之一是getters的概念。 他们是在运行时构造的可调用对象，用于检索对象的属性或序列中的内容。 在使用迭代器或生成器序列时，getters特别有用，它们的开销比lambda或Python函数少。

```python
#operator_attrgetter.py
from operator import *

class MyObj:
    """example class for attrgetter"""

    def __init__(self, arg):
        super().__init__()
        self.arg = arg

    def __repr__(self):
        return 'MyObj({})'.format(self.arg)

l = [MyObj(i) for i in range(5)]
print('objects   :', l)

# Extract the 'arg' value from each object
g = attrgetter('arg')
vals = [g(i) for i in l]
print('arg values:', vals)

# Sort using arg
l.reverse()
print('reversed  :', l)
print('sorted    :', sorted(l, key=g))
```
attribute getter的工作方式类似于`lambda x，n ='attrname'：getattr（x，n）`：

```
$ python3 operator_attrgetter.py

objects   : [MyObj(0), MyObj(1), MyObj(2), MyObj(3), MyObj(4)]
arg values: [0, 1, 2, 3, 4]
reversed  : [MyObj(4), MyObj(3), MyObj(2), MyObj(1), MyObj(0)]
sorted    : [MyObj(0), MyObj(1), MyObj(2), MyObj(3), MyObj(4)]
```
item getter的工作方式类似于`lambda x，y=5： x[y]`：


```python
#operator_itemgetter.py
from operator import *

l = [dict(val=-1 * i) for i in range(4)]
print('Dictionaries:')
print(' original:', l)
g = itemgetter('val')
vals = [g(i) for i in l]
print('   values:', vals)
print('   sorted:', sorted(l, key=g))

print()
l = [(i, i * -2) for i in range(4)]
print('\nTuples:')
print(' original:', l)
g = itemgetter(1)
vals = [g(i) for i in l]
print('   values:', vals)
print('   sorted:', sorted(l, key=g))
```
item getter 用于映射和序列。

```
$ python3 operator_itemgetter.py

Dictionaries:
 original: [{'val': 0}, {'val': -1}, {'val': -2}, {'val': -3}]
   values: [0, -1, -2, -3]
   sorted: [{'val': -3}, {'val': -2}, {'val': -1}, {'val': 0}]

Tuples:
 original: [(0, 0), (1, -2), (2, -4), (3, -6)]
   values: [0, -2, -4, -6]
   sorted: [(3, -6), (2, -4), (1, -2), (0, 0)]
```

### Combining Operators and Custom Classes

operator模块中的函数通过标准Python接口进行操作，因此它们可以使用用户定义的类以及内置类型。

```python
#operator_classes.py
from operator import *

class MyObj:
    """Example for operator overloading"""

    def __init__(self, val):
        super(MyObj, self).__init__()
        self.val = val

    def __str__(self):
        return 'MyObj({})'.format(self.val)

    def __lt__(self, other):
        """compare for less-than"""
        print('Testing {} < {}'.format(self, other))
        return self.val < other.val

    def __add__(self, other):
        """add values"""
        print('Adding {} + {}'.format(self, other))
        return MyObj(self.val + other.val)

a = MyObj(1)
b = MyObj(2)

print('Comparison:')
print(lt(a, b))

print('\nArithmetic:')
print(add(a, b))
```
有关每个运算符使用的特殊方法的完整列表，请参阅Python参考指南。

```
$ python3 operator_classes.py

Comparison:
Testing MyObj(1) < MyObj(2)
True

Arithmetic:
Adding MyObj(1) + MyObj(2)
MyObj(3)
```

## contextlib — Context Manager Utilities

目的：用于创建和使用上下文管理器的实用程序(Utilities)。  
contextlib模块包含用于处理上下文管理器和with语句的实用程序。

### Context Manager API

上下文管理器负责代码块中的资源，可能在进入块时创建它，然后在退出块后清理它。 例如，文件支持上下文管理器API，以便在完成所有读取或写入操作后轻松确保它们被关闭。

```python
#contextlib_file.py
with open('/tmp/pymotw.txt', 'wt') as f:
    f.write('contents go here')
# file is automatically closed
```
上下文管理器由with语句启用，API涉及两个方法。 当执行流进入with内的代码块时运行`__enter__（）`方法。 它返回一个在上下文中使用的对象。 当执行流离开with块时，将调用上下文管理器的`__exit__（）`方法来清理正在被使用的任何资源。

```python
#contextlib_api.py
class Context:

    def __init__(self):
        print('__init__()')

    def __enter__(self):
        print('__enter__()')
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        print('__exit__()')

with Context():
    print('Doing work in the context')
```
组合上下文管理器和with语句是比编写一个`try：finally`块更紧凑的方式，因为即使引发了异常，上下文管理器的`__exit__（）`方法也总是被调用。

```
$ python3 contextlib_api.py

__init__()
__enter__()
Doing work in the context
__exit__()
```
`__enter__（）`方法可以返回与with语句的as子句中指定的名称关联的任何对象。 在此示例中，Context返回“使用开放上下文（open context）”的对象。

```python
#contextlib_api_other_object.py
class WithinContext:

    def __init__(self, context):
        print('WithinContext.__init__({})'.format(context))

    def do_something(self):
        print('WithinContext.do_something()')

    def __del__(self):
        print('WithinContext.__del__')

class Context:

    def __init__(self):
        print('Context.__init__()')

    def __enter__(self):
        print('Context.__enter__()')
        return WithinContext(self)

    def __exit__(self, exc_type, exc_val, exc_tb):
        print('Context.__exit__()')

with Context() as c:
    c.do_something()
```
与变量c关联的值是`__enter__（）`返回的对象，该对象不一定是在with语句中创建的Context实例。

```
$ python3 contextlib_api_other_object.py

Context.__init__()
Context.__enter__()
WithinContext.__init__(<__main__.Context object at 0x101e9c080>)
WithinContext.do_something()
Context.__exit__()
WithinContext.__del__
```
`__exit__（）`方法接收包含with块中引发的任何异常的详细信息的参数。

```python
#contextlib_api_error.py
class Context:

    def __init__(self, handle_error):
        print('__init__({})'.format(handle_error))
        self.handle_error = handle_error

    def __enter__(self):
        print('__enter__()')
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        print('__exit__()')
        print('  exc_type =', exc_type)
        print('  exc_val  =', exc_val)
        print('  exc_tb   =', exc_tb)
        return self.handle_error

with Context(True):
    raise RuntimeError('error message handled')

print()

with Context(False):
    raise RuntimeError('error message propagated')
```
如果上下文管理器可以处理异常，`__exit__（）`应该返回一个true值，以指示不需要传播该异常。 返回false会导致在`__exit__（）`返回后重新引发异常。

```
$ python3 contextlib_api_error.py

__init__(True)
__enter__()
__exit__()
  exc_type = <class 'RuntimeError'>
  exc_val  = error message handled
  exc_tb   = <traceback object at 0x1044ea648>

__init__(False)
__enter__()
__exit__()
  exc_type = <class 'RuntimeError'>
  exc_val  = error message propagated
  exc_tb   = <traceback object at 0x1044ea648>
Traceback (most recent call last):
  File "contextlib_api_error.py", line 34, in <module>
    raise RuntimeError('error message propagated')
RuntimeError: error message propagated
```

### Context Managers as Function Decorators

ContextDecorator类增加了对常规上下文管理器类的支持，使它们可以用作函数修饰器和上下文管理器。

```python
#contextlib_decorator.py
import contextlib

class Context(contextlib.ContextDecorator):

    def __init__(self, how_used):
        self.how_used = how_used
        print('__init__({})'.format(how_used))

    def __enter__(self):
        print('__enter__({})'.format(self.how_used))
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        print('__exit__({})'.format(self.how_used))

@Context('as decorator')
def func(message):
    print(message)

print()
with Context('as context manager'):
    print('Doing work in the context')

print()
func('Doing work in the wrapped function')
```
使用上下文管理器作为装饰器的一个区别是`__enter__（）`返回的值在被装饰的函数内部不可用，这与使用with和as时不同。 传递给装饰函数的参数以通常的方式提供。

```
$ python3 contextlib_decorator.py

__init__(as decorator)

__init__(as context manager)
__enter__(as context manager)
Doing work in the context
__exit__(as context manager)

__enter__(as decorator)
Doing work in the wrapped function
__exit__(as decorator)
```

### From Generator to Context Manager

通过使用`__enter__（）`和`__exit__（）`方法编写类，以传统方式创建上下文管理器并不困难。 但有时候完全写出所有的东西对于一些琐碎的上下文来说是额外的开销。 在这些情况下，使用contextmanager（）装饰器将生成器函数转换为上下文管理器。

```python
#contextlib_contextmanager.py
import contextlib

@contextlib.contextmanager
def make_context():
    print('  entering')
    try:
        yield {}
    except RuntimeError as err:
        print('  ERROR:', err)
    finally:
        print('  exiting')

print('Normal:')
with make_context() as value:
    print('  inside with statement:', value)

print('\nHandled error:')
with make_context() as value:
    raise RuntimeError('showing example of handling an error')

print('\nUnhandled error:')
with make_context() as value:
    raise ValueError('this exception is not handled')
```
生成器应该初始化上下文，只产生一次，然后清理上下文。 如果有的话，产生的值绑定到with语句的as子句中的变量。 with块内的异常会在生成器内重新引发，因此可以在那里处理它们。

```
$ python3 contextlib_contextmanager.py

Normal:
  entering
  inside with statement: {}
  exiting

Handled error:
  entering
  ERROR: showing example of handling an error
  exiting

Unhandled error:
  entering
  exiting
Traceback (most recent call last):
  File "contextlib_contextmanager.py", line 33, in <module>
    raise ValueError('this exception is not handled')
ValueError: this exception is not handled
```
contextmanager（）返回的上下文管理器派生自ContextDecorator，因此它也可以作为函数装饰器使用。

```python
#contextlib_contextmanager_decorator.py
import contextlib

@contextlib.contextmanager
def make_context():
    print('  entering')
    try:
        # Yield control, but not a value, because any value
        # yielded is not available when the context manager
        # is used as a decorator.
        yield
    except RuntimeError as err:
        print('  ERROR:', err)
    finally:
        print('  exiting')

@make_context()
def normal():
    print('  inside with statement')

@make_context()
def throw_error(err):
    raise err

print('Normal:')
normal()

print('\nHandled error:')
throw_error(RuntimeError('showing example of handling an error'))

print('\nUnhandled error:')
throw_error(ValueError('this exception is not handled'))
```
与上面的ContextDecorator示例一样，当上下文管理器用作装饰器时，生成器生成的值在正在装饰的函数内不可用。 传递给修饰函数的参数仍然可用，如本例中的throw_error（）所示。

```
$ python3 contextlib_contextmanager_decorator.py

Normal:
  entering
  inside with statement
  exiting

Handled error:
  entering
  ERROR: showing example of handling an error
  exiting

Unhandled error:
  entering
  exiting
Traceback (most recent call last):
  File "contextlib_contextmanager_decorator.py", line 43, in
<module>
    throw_error(ValueError('this exception is not handled'))
  File ".../lib/python3.6/contextlib.py", line 52, in inner
    return func(*args, **kwds)
  File "contextlib_contextmanager_decorator.py", line 33, in
throw_error
    raise err
ValueError: this exception is not handled
```

### Closing Open Handles

文件类直接支持上下文管理器API，但是其他一些表示打开句柄的对象则不支持。 contextlib的标准库文档中给出的示例是从urllib.urlopen（）返回的对象。 还有其他遗留类使用close（）方法但不支持上下文管理器API。 要确保句柄已关闭，请使用closing（）为其创建上下文管理器。

```python
#contextlib_closing.py
import contextlib

class Door:

    def __init__(self):
        print('  __init__()')
        self.status = 'open'

    def close(self):
        print('  close()')
        self.status = 'closed'

print('Normal Example:')
with contextlib.closing(Door()) as door:
    print('  inside with statement: {}'.format(door.status))
print('  outside with statement: {}'.format(door.status))

print('\nError handling example:')
try:
    with contextlib.closing(Door()) as door:
        print('  raising from inside with statement')
        raise RuntimeError('error message')
except Exception as err:
    print('  Had an error:', err)
```
无论with块中是否有错误，句柄都会关闭。

```
$ python3 contextlib_closing.py

Normal Example:
  __init__()
  inside with statement: open
  close()
  outside with statement: closed

Error handling example:
  __init__()
  raising from inside with statement
  close()
  Had an error: error message
```

### Ignoring Exceptions

忽略库引发的异常通常很有用，因为错误表明已经实现了所需的状态，否则可以忽略它。 忽略异常的最常见方法是使用`try：except`语句，在except块中只有一个pass语句。

```python
#contextlib_ignore_error.py
import contextlib

class NonFatalError(Exception):
    pass

def non_idempotent_operation():
    raise NonFatalError(
        'The operation failed because of existing state'
    )

try:
    print('trying non-idempotent operation')
    non_idempotent_operation()
    print('succeeded!')
except NonFatalError:
    pass

print('done')
```
在这种情况下，操作失败并忽略错误。

```
$ python3 contextlib_ignore_error.py

trying non-idempotent operation
done
```
`try：except`表单可以替换为contextlib.suppress（），以更明确地抑制with块中任何位置某类异常的发生。

```python
#contextlib_suppress.py
import contextlib

class NonFatalError(Exception):
    pass

def non_idempotent_operation():
    raise NonFatalError(
        'The operation failed because of existing state'
    )

with contextlib.suppress(NonFatalError):
    print('trying non-idempotent operation')
    non_idempotent_operation()
    print('succeeded!')

print('done')
```
在此更新版本中，异常将完全丢弃。

```
$ python3 contextlib_suppress.py

trying non-idempotent operation
done
```

### Redirecting Output Streams

设计不良的库代码可能直接写入sys.stdout或sys.stderr，而不提供参数用来配置不同的输出目标。 `redirect_stdout（）`和`redirect_stderr（）`上下文管理器可用于捕获此类函数的输出，这类函数无法更改源码以接受新的输出参数。

```python
#contextlib_redirect.py
from contextlib import redirect_stdout, redirect_stderr
import io
import sys

def misbehaving_function(a):
    sys.stdout.write('(stdout) A: {!r}\n'.format(a))
    sys.stderr.write('(stderr) A: {!r}\n'.format(a))

capture = io.StringIO()
with redirect_stdout(capture), redirect_stderr(capture):
    misbehaving_function(5)

print(capture.getvalue())
```
在此示例中，misbehaving_function（）写入stdout和stderr，但是两个上下文管理器将该输出发送到同一个io.StringIO实例，在该实例中保存它以便稍后使用。

```
$ python3 contextlib_redirect.py

(stdout) A: 5
(stderr) A: 5
```
>注意  
`redirect_stdout（）`和`redirect_stderr（）`都通过替换sys模块中的对象来修改全局状态，因此应谨慎使用。 这些函数不是线程安全的，并且可能会干扰期望将标准输出流附加到终端设备的其他操作。

### Dynamic Context Manager Stacks

大多数上下文管理器一次操作一个对象，例如单个文件或数据库句柄。在这些情况下，该对象是事先已知的，并且使用上下文管理器的代码可以围绕那个对象构建。 在其他情况下，程序可能需要在上下文中创建未知数量的对象，同时希望在控制流退出上下文时清除所有对象。 创建ExitStack是为了处理这些更具动态性的案例。  
ExitStack实例维护清理回调的堆栈数据结构。回调在上下文中显式填充，并且当控制流退出上下文时，以相反的顺序调用任何已注册的回调。 结果就像多个嵌套的with语句一样，除了它们是动态建立的。

#### Stacking Context Managers

有几种方法可以填充ExitStack。 此示例使用`enter_context（）`将新的上下文管理器添加到堆栈。

```python
#contextlib_exitstack_enter_context.py
import contextlib

@contextlib.contextmanager
def make_context(i):
    print('{} entering'.format(i))
    yield {}
    print('{} exiting'.format(i))

def variable_stack(n, msg):
    with contextlib.ExitStack() as stack:
        for i in range(n):
            stack.enter_context(make_context(i))
        print(msg)

variable_stack(2, 'inside context')
```
`enter_context（）`首先在上下文管理器上调用`__enter__（）`，然后将其`__exit__（）`方法注册为要在撤消堆栈时调用的回调。

```
$ python3 contextlib_exitstack_enter_context.py

0 entering
1 entering
inside context
1 exiting
0 exiting
```
给予ExitStack的上下文管理器被视为它们是一系列嵌套的with语句。在上下文中的任何地方发生的错误都会通过上下文管理器的正常错误处理进行传播。 这些上下文管理器类说明了错误传播的方式。

```python
#contextlib_context_managers.py
import contextlib

class Tracker:
    "Base class for noisy context managers."

    def __init__(self, i):
        self.i = i

    def msg(self, s):
        print('  {}({}): {}'.format(self.__class__.__name__, self.i, s))

    def __enter__(self):
        self.msg('entering')

class HandleError(Tracker):
    "If an exception is received, treat it as handled."

    def __exit__(self, *exc_details):
        received_exc = exc_details[1] is not None
        if received_exc:
            self.msg('handling exception {!r}'.format(
                exc_details[1]))
        self.msg('exiting {}'.format(received_exc))
        # Return Boolean value indicating whether the exception
        # was handled.
        return received_exc

class PassError(Tracker):
    "If an exception is received, propagate it."

    def __exit__(self, *exc_details):
        received_exc = exc_details[1] is not None
        if received_exc:
            self.msg('passing exception {!r}'.format(
                exc_details[1]))
        self.msg('exiting')
        # Return False, indicating any exception was not handled.
        return False

class ErrorOnExit(Tracker):
    "Cause an exception."

    def __exit__(self, *exc_details):
        self.msg('throwing error')
        raise RuntimeError('from {}'.format(self.i))

class ErrorOnEnter(Tracker):
    "Cause an exception."

    def __enter__(self):
        self.msg('throwing error on enter')
        raise RuntimeError('from {}'.format(self.i))

    def __exit__(self, *exc_info):
        self.msg('exiting')
```
使用这些类的示例基于`variable_stack（）`，它使用传递的上下文管理器构造一个ExitStack，逐个构建整体上下文。 下面的示例通过传递不同的上下文管理器来探索错误处理行为。 首先，是正常没有异常的情况。

```python
print('No errors:')
variable_stack([
    HandleError(1),
    PassError(2),
])
```
然后，是在堆栈末尾的上下文管理器中处理异常的示例，堆栈中所有打开的上下文在堆栈展开时关闭。

```python
print('\nError at the end of the context stack:')
variable_stack([
    HandleError(1),
    HandleError(2),
    ErrorOnExit(3),
])
```
接下来，是一个在堆栈中间的上下文管理器中处理异常的示例，其中错误不会发生直到某些上下文已经关闭，因此这些上下文不会看到错误。

```python
print('\nError in the middle of the context stack:')
variable_stack([
    HandleError(1),
    PassError(2),
    ErrorOnExit(3),
    HandleError(4),
])
```
最后，是一个异常没有被处理并被传播到调用代码的示例。

```python
try:
    print('\nError ignored:')
    variable_stack([
        PassError(1),
        ErrorOnExit(2),
    ])
except RuntimeError:
    print('error handled outside of context')
```
如果堆栈中的任何上下文管理器收到异常并返回True值，则会阻止该异常传播到任何其他上下文管理器。

```
$ python3 contextlib_exitstack_enter_context_errors.py

No errors:
  HandleError(1): entering
  PassError(2): entering
  PassError(2): exiting
  HandleError(1): exiting False
  outside of stack, any errors were handled

Error at the end of the context stack:
  HandleError(1): entering
  HandleError(2): entering
  ErrorOnExit(3): entering
  ErrorOnExit(3): throwing error
  HandleError(2): handling exception RuntimeError('from 3',)
  HandleError(2): exiting True
  HandleError(1): exiting False
  outside of stack, any errors were handled

Error in the middle of the context stack:
  HandleError(1): entering
  PassError(2): entering
  ErrorOnExit(3): entering
  HandleError(4): entering
  HandleError(4): exiting False
  ErrorOnExit(3): throwing error
  PassError(2): passing exception RuntimeError('from 3',)
  PassError(2): exiting
  HandleError(1): handling exception RuntimeError('from 3',)
  HandleError(1): exiting True
  outside of stack, any errors were handled

Error ignored:
  PassError(1): entering
  ErrorOnExit(2): entering
  ErrorOnExit(2): throwing error
  PassError(1): passing exception RuntimeError('from 2',)
  PassError(1): exiting
error handled outside of context
```

#### Arbitrary Context Callbacks

ExitStack还支持任意的上下文关闭回调，从而可以轻松清理不通过上下文管理器控制的资源。

```python
#contextlib_exitstack_callbacks.py
import contextlib

def callback(*args, **kwds):
    print('closing callback({}, {})'.format(args, kwds))

with contextlib.ExitStack() as stack:
    stack.callback(callback, 'arg1', 'arg2')
    stack.callback(callback, arg3='val3')
```
与完整上下文管理器的`__exit__（）`方法一样，回调的调用顺序与它们的注册顺序相反。

```
$ python3 contextlib_exitstack_callbacks.py

closing callback((), {'arg3': 'val3'})
closing callback(('arg1', 'arg2'), {})
```
无论是否发生错误，都会调用回调，并且不会给出有关是否发生错误的任何信息。 它们的返回值被忽略。

```python
#contextlib_exitstack_callbacks_error.py
import contextlib

def callback(*args, **kwds):
    print('closing callback({}, {})'.format(args, kwds))

try:
    with contextlib.ExitStack() as stack:
        stack.callback(callback, 'arg1', 'arg2')
        stack.callback(callback, arg3='val3')
        raise RuntimeError('thrown error')
except RuntimeError as err:
    print('ERROR: {}'.format(err))
```
因为它们无法访问错误，所以回调无法抑制异常通过其余的上下文管理器堆栈传播。

```
$ python3 contextlib_exitstack_callbacks_error.py

closing callback((), {'arg3': 'val3'})
closing callback(('arg1', 'arg2'), {})
ERROR: thrown error
```
回调可以方便地清楚地定义清理逻辑，而无需创建新的上下文管理器类。 为了提高代码可读性，可以将该逻辑封装在内联函数中，并将callback（）用作装饰器。

```python
#contextlib_exitstack_callbacks_decorator.py
import contextlib

with contextlib.ExitStack() as stack:

    @stack.callback
    def inline_cleanup():
        print('inline_cleanup()')
        print('local_resource = {!r}'.format(local_resource))

    local_resource = 'resource created in context'
    print('within the context')
```
无法为“使用装饰器形式的callback（）注册的”函数指定参数。
但是，如果清理回调是内联定义的，则范围规则允许它访问调用代码中定义的变量。

```
$ python3 contextlib_exitstack_callbacks_decorator.py

within the context
inline_cleanup()
local_resource = 'resource created in context'
```

#### Partial Stacks

有时，在构建复杂的上下文时，如果上下文无法完全构建，则能够中止一个操作是有用的，但是如果能够正确设置所有资源，则延迟一段时间清除所有资源。 例如，如果操作需要多个长期网络连接，则最好不要在一个连接失败时启动操作。但是，如果可以打开所有连接，则需要保持打开的时间长于单个上下文管理器的持续时间。 可以在此场景中可以使用ExitStack的pop_all（）方法。  
pop_all（）清除调用它的堆栈中的所有上下文管理器和回调，并返回一个预先填充了相同上下文管理器和回调的新堆栈。 在原始堆栈消失之后，可以稍后调用新堆栈的close（）方法来清理资源。

```python
#contextlib_exitstack_pop_all.py
import contextlib

from contextlib_context_managers import *

def variable_stack(contexts):
    with contextlib.ExitStack() as stack:
        for c in contexts:
            stack.enter_context(c)
        # Return the close() method of a new stack as a clean-up
        # function.
        return stack.pop_all().close
    # Explicitly return None, indicating that the ExitStack could
    # not be initialized cleanly but that cleanup has already
    # occurred.
    return None

print('No errors:')
cleaner = variable_stack([
    HandleError(1),
    HandleError(2),
])
cleaner()

print('\nHandled error building context manager stack:')
try:
    cleaner = variable_stack([
        HandleError(1),
        ErrorOnEnter(2),
    ])
except RuntimeError as err:
    print('caught error {}'.format(err))
else:
    if cleaner is not None:
        cleaner()
    else:
        print('no cleaner returned')

print('\nUnhandled error building context manager stack:')
try:
    cleaner = variable_stack([
        PassError(1),
        ErrorOnEnter(2),
    ])
except RuntimeError as err:
    print('caught error {}'.format(err))
else:
    if cleaner is not None:
        cleaner()
    else:
        print('no cleaner returned')
```
此示例使用前面定义的相同上下文管理器类，区别在于ErrorOnEnter在`__enter__（）`而不是`__exit__（）`上生成错误。 在`variable_stack（）`内部，如果输入的所有上下文都没有错误，则返回新的ExitStack的close（）方法。 如果发生处理错误，则`variable_stack（）`返回None以指示已完成清理工作。 如果发生未处理的错误，则清除部分堆栈并传播错误。

```
$ python3 contextlib_exitstack_pop_all.py

No errors:
  HandleError(1): entering
  HandleError(2): entering
  HandleError(2): exiting False
  HandleError(1): exiting False

Handled error building context manager stack:
  HandleError(1): entering
  ErrorOnEnter(2): throwing error on enter
  HandleError(1): handling exception RuntimeError('from 2',)
  HandleError(1): exiting True
no cleaner returned

Unhandled error building context manager stack:
  PassError(1): entering
  ErrorOnEnter(2): throwing error on enter
  PassError(1): passing exception RuntimeError('from 2',)
  PassError(1): exiting
caught error from 2
```
