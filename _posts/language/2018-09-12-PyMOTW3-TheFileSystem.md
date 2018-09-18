---
title: PyMOTW-3 --- The File System
categories: languages
tags: PyMOTW-3
---

Python的标准库包括大量工具，用于处理文件系统上的文件，构建和解析文件名以及检查文件内容。

处理文件的第一步是确定要处理的文件的名称。Python将文件名表示为简单字符串，但os.path中标准、平台无关的组件提供了构建他们的工具。

pathlib模块提供了一个面向对象的API，用于处理文件系统路径。使用它比os.path更方便，因为它在更高的抽象级别上运行。

使用os中的listdir（）列出目录的内容，或使用glob从模式中构建一个文件名的列表。

glob使用的文件名模式匹配也直接通过fnmatch公开，因此可以在其他上下文中使用。

识别文件名后，可以使用os.stat（）和stat中的常量检查其他特性，例如权限或文件大小。

当应用程序需要随机访问文件时，linecache可以通过行号轻松读取行。文件的内容保存在缓存中，因此请注意内存消耗。

tempfile对于需要创建暂存文件以临时保存数据或将其移动到永久位置之前的情况非常有用。它提供了安全可靠地创建临时文件和目录的类。名称保证是唯一的，并包含随机组件，因此不容易猜到。

程序通常需要整体处理文件，而不考虑其内容。 shutil模块包括高级文件操作，例如复制文件和目录，以及创建或提取文件存档。

filecmp模块比较文件和目录通过查看它们包含的字节，但没有任何关于其格式的特殊知识。

内置file类可用于读取和写入本地文件系统上可见的文件。但是，当程序通过read（）和write（）接口访问大文件时，性能会受到影响，因为它们都涉及在将数据从磁盘移动到“应用程序可以看到的”内存时多次复制数据。使用mmap告诉操作系统使用其虚拟内存子系统将文件的内容直接映射到程序可访问的内存中，从而避免操作系统和文件对象的内部缓冲区之间的复制步骤。

使用ASCII中不可用的字符的文本数据通常以Unicode数据格式保存。由于标准文件句柄假定文本文件的每个字节代表一个字符，因此使用多字节编码读取Unicode文本需要额外的处理。codecs模块自动处理编码和解码，因此在许多情况下可以使用非ASCII文件而无需对程序进行任何其他更改。

io模块提供对“用于实现Python基于文件的输入和输出的”类的访问。为了测试依赖于从文件读取或写入数据的代码，io提供了一个内存中的流对象，其行为类似于文件，但不驻留在磁盘上。

## os.path — Platform-independent Manipulation of Filenames

目的：解析，构建，测试以及以其他方式处理文件名和路径。  
使用os.path模块中包含的函数可以轻松编写代码来处理多个平台上的文件。 即使不打算在平台之间移植的程序也应使用os.path进行可靠的文件名解析。

### Parsing Paths

os.path中的第一组函数可用于将表示文件名的字符串解析为其组成部分。 重要的是要意识到这些功能不依赖于实际存在的路径; 他们只在字符串上操作。  
路径解析依赖于os中定义的一些变量：

+ os.sep - 路径各部分之间的分隔符（例如，`/` 或 `\` ）。
+ os.extsep - 文件名和文件扩展名之间的分隔符（例如 `.` ）。
+ os.pardir - 意味着遍历目录树上一级的路径组件（例如 `..` ）。
+ os.curdir - 引用当前目录的路径组件（例如 `.` ）。

split（）函数将路径分成两个独立的部分，并返回带有结果的元组。 元组的第二个元素是路径的最后一个元素，第一个元素是它之前的所有元素。

```python
#ospath_split.py
import os.path

PATHS = [
    '/one/two/three',
    '/one/two/three/',
    '/',
    '.',
    '',
]

for path in PATHS:
    print('{!r:>17} : {}'.format(path, os.path.split(path)))
```
当输入参数以os.sep结尾时，路径的最后一个元素是一个空字符串。

```
$ python3 ospath_split.py

 '/one/two/three' : ('/one/two', 'three')
'/one/two/three/' : ('/one/two/three', '')
              '/' : ('/', '')
              '.' : ('', '.')
               '' : ('', '')
```
basename（）函数返回值等于split（）的第二部分。

```python
#ospath_basename.py
import os.path

PATHS = [
    '/one/two/three',
    '/one/two/three/',
    '/',
    '.',
    '',
]

for path in PATHS:
    print('{!r:>17} : {!r}'.format(path, os.path.basename(path)))
```
完整路径被剥离到最后一个元素，无论是指向文件还是目录。 如果路径在目录分隔符（os.sep）中结束，则base部分被视为空。

```
$ python3 ospath_basename.py

 '/one/two/three' : 'three'
'/one/two/three/' : ''
              '/' : ''
              '.' : '.'
               '' : ''
```
dirname（）函数返回拆分路径的第一部分：

```python
#ospath_dirname.py
import os.path

PATHS = [
    '/one/two/three',
    '/one/two/three/',
    '/',
    '.',
    '',
]

for path in PATHS:
    print('{!r:>17} : {!r}'.format(path, os.path.dirname(path)))
```
将basename（）的结果与dirname（）组合在一起可以得到原始路径。

```
$ python3 ospath_dirname.py

 '/one/two/three' : '/one/two'
'/one/two/three/' : '/one/two/three'
              '/' : '/'
              '.' : ''
               '' : ''
```
splitext（）的作用类似于split（），但是在扩展分隔符上划分路径，而不是目录分隔符。

```python
#ospath_splitext.py
import os.path

PATHS = [
    'filename.txt',
    'filename',
    '/path/to/filename.txt',
    '/',
    '',
    'my-archive.tar.gz',
    'no-extension.',
]

for path in PATHS:
    print('{!r:>21} : {!r}'.format(path, os.path.splitext(path)))
```
在查找扩展名时，只使用最后一次出现的`os.extsep`，因此如果文件名有多个扩展名，则拆分结果会在前缀上留下部分扩展名。

```
$ python3 ospath_splitext.py

       'filename.txt' : ('filename', '.txt')
           'filename' : ('filename', '')
'/path/to/filename.txt' : ('/path/to/filename', '.txt')
                  '/' : ('/', '')
                   '' : ('', '')
  'my-archive.tar.gz' : ('my-archive.tar', '.gz')
      'no-extension.' : ('no-extension', '.')
```
commonprefix（）将路径列表作为参数，并返回表示所有路径中存在的公共前缀的单个字符串。 该值可能表示实际上不存在的路径，并且路径分隔符未包含在考虑中，因此前缀可能不会在分隔符边界上停止。

```python
#ospath_commonprefix.py
import os.path

paths = ['/one/two/three/four',
         '/one/two/threefold',
         '/one/two/three/',
         ]
for path in paths:
    print('PATH:', path)

print()
print('PREFIX:', os.path.commonprefix(paths))
```
在此示例中，公共前缀字符串是`/one/two/three`，即使其中一个路径不包含名为three的目录。

```
$ python3 ospath_commonprefix.py

PATH: /one/two/three/four
PATH: /one/two/threefold
PATH: /one/two/three/

PREFIX: /one/two/three
```
commonpath（）支持路径分隔符，并返回不包含部分路径值的前缀。

```python
#ospath_commonpath.py
import os.path

paths = ['/one/two/three/four',
         '/one/two/threefold',
         '/one/two/three/',
         ]
for path in paths:
    print('PATH:', path)

print()
print('PREFIX:', os.path.commonpath(paths))
```
因为“threefold”在“three”之后没有路径分隔符，所以公共前缀是`/one/two`。

```
$ python3 ospath_commonpath.py

PATH: /one/two/three/four
PATH: /one/two/threefold
PATH: /one/two/three/

PREFIX: /one/two
```

### Building Paths

除了将现有路径分开之外，经常需要从其他字符串构建路径。 要将多个路径组件组合为单个值，请使用join（）：

```python
#ospath_join.py
import os.path

PATHS = [
    ('one', 'two', 'three'),
    ('/', 'one', 'two', 'three'),
    ('/one', '/two', '/three'),
]

for parts in PATHS:
    print('{} : {!r}'.format(parts, os.path.join(*parts)))
```
如果要连接的任何参数以`os.sep`开头，则所有先前的参数都将被丢弃，新的参数将成为返回值的开头。

```
$ python3 ospath_join.py

('one', 'two', 'three') : 'one/two/three'
('/', 'one', 'two', 'three') : '/one/two/three'
('/one', '/two', '/three') : '/three'
```
也可以使用包含可以自动扩展的“变量”组件的路径。 例如，expanduser（）将波浪号（~）字符转换为用户主目录的名称。

```python
#ospath_expanduser.py
import os.path

for user in ['', 'dhellmann', 'nosuchuser']:
    lookup = '~' + user
    print('{!r:>15} : {!r}'.format(
        lookup, os.path.expanduser(lookup)))
```
如果找不到用户的主目录，则返回字符串不变，就像本例中的~nosuchuser一样。

```
$ python3 ospath_expanduser.py

            '~' : '/Users/dhellmann'
   '~dhellmann' : '/Users/dhellmann'
  '~nosuchuser' : '~nosuchuser'
```
expandvars（）更通用，并扩展路径中存在的任何shell环境变量。

```python
#ospath_expandvars.py
import os.path
import os

os.environ['MYVAR'] = 'VALUE'

print(os.path.expandvars('/path/to/$MYVAR'))
```
不执行由变量值生成的文件名是否存在的验证

```
$ python3 ospath_expandvars.py

/path/to/VALUE
```

### Normalizing Paths

使用join（）从单独的字符串或嵌入变量组装的路径最终可能会带有额外的分隔符或相对路径组件。 使用normpath（）来清理它们：

```python
#ospath_normpath.py
import os.path

PATHS = [
    'one//two//three',
    'one/./two/./three',
    'one/../alt/two/three',
]

for path in PATHS:
    print('{!r:>22} : {!r}'.format(path, os.path.normpath(path)))
```
由`os.curdir`和`os.pardir`组成的路径段将被评估和清理。

```
$ python3 ospath_normpath.py

     'one//two//three' : 'one/two/three'
   'one/./two/./three' : 'one/two/three'
'one/../alt/two/three' : 'alt/two/three'
```
要将相对路径转换为绝对文件名，请使用`abspath（）`。

```python
#ospath_abspath.py
import os
import os.path

os.chdir('/usr')

PATHS = [
    '.',
    '..',
    './one/two/three',
    '../one/two/three',
]

for path in PATHS:
    print('{!r:>21} : {!r}'.format(path, os.path.abspath(path)))
```
结果是一个完整的路径，从文件系统树的顶部开始。

```
$ python3 ospath_abspath.py

                  '.' : '/usr'
                 '..' : '/'
    './one/two/three' : '/usr/one/two/three'
   '../one/two/three' : '/one/two/three'
```

### File Times

除了使用路径之外，os.path还包括用于检索文件属性的函数，类似于os.stat（）返回的函数：

```python
#ospath_properties.py
import os.path
import time

print('File         :', __file__)
print('Access time  :', time.ctime(os.path.getatime(__file__)))
print('Modified time:', time.ctime(os.path.getmtime(__file__)))
print('Change time  :', time.ctime(os.path.getctime(__file__)))
print('Size         :', os.path.getsize(__file__))
```
os.path.getatime（）返回访问时间，os.path.getmtime（）返回修改时间，os.path.getctime（）返回创建时间。 os.path.getsize（）返回文件中的数据量，以字节为单位表示。

```
$ python3 ospath_properties.py

File         : ospath_properties.py
Access time  : Sun Mar 18 16:21:22 2018
Modified time: Fri Nov 11 17:18:44 2016
Change time  : Fri Nov 11 17:18:44 2016
Size         : 481
```

### Testing Files

当程序处理路径名时，通常需要知道路径是指文件、目录还是符号链接以及它是否存在。 os.path包含用于测试所有这些条件的函数。

```python
#ospath_tests.py
import os.path

FILENAMES = [
    __file__,
    os.path.dirname(__file__),
    '/',
    './broken_link',
]

for file in FILENAMES:
    print('File        : {!r}'.format(file))
    print('Absolute    :', os.path.isabs(file))
    print('Is File?    :', os.path.isfile(file))
    print('Is Dir?     :', os.path.isdir(file))
    print('Is Link?    :', os.path.islink(file))
    print('Mountpoint? :', os.path.ismount(file))
    print('Exists?     :', os.path.exists(file))
    print('Link Exists?:', os.path.lexists(file))
    print()
```
所有测试函数都返回布尔值。

```
$ ln -s /does/not/exist broken_link
$ python3 ospath_tests.py

File        : 'ospath_tests.py'
Absolute    : False
Is File?    : True
Is Dir?     : False
Is Link?    : False
Mountpoint? : False
Exists?     : True
Link Exists?: True

File        : ''
Absolute    : False
Is File?    : False
Is Dir?     : False
Is Link?    : False
Mountpoint? : False
Exists?     : False
Link Exists?: False

File        : '/'
Absolute    : True
Is File?    : False
Is Dir?     : True
Is Link?    : False
Mountpoint? : True
Exists?     : True
Link Exists?: True

File        : './broken_link'
Absolute    : False
Is File?    : False
Is Dir?     : False
Is Link?    : True
Mountpoint? : False
Exists?     : False
Link Exists?: True
```

## pathlib — Filesystem Paths as Objects

目的：使用面向对象的API而不是低级字符串操作来解析，构建，测试和以其他方式处理文件名和路径。

### Path Representations

pathlib包括用于管理文件系统路径的类，路径可使用POSIX标准或Microsoft Windows语法格式化。 它包括所谓的“纯”类，它们对字符串进行操作但不与实际文件系统交互，以及“具体”类，它们扩展API以包括反映或修改本地文件系统数据的操作。  
纯类PurePosixPath和PureWindowsPath可以在任何操作系统上实例化和使用，因为它们只处理名称。 要实例化处理真实文件系统的正确类，使用Path获取PosixPath或WindowsPath，具体取决于平台。

### Building Paths

要实例化新路径，请将字符串作为第一个参数。 路径对象的字符串表示形式是此名称值。 要创建的新路径引用的值如果相对于现有路径，请使用 `/` 运算符来扩展路径。 该运算符的参数可以是字符串或其他路径对象。

```python
#pathlib_operator.py
import pathlib

usr = pathlib.PurePosixPath('/usr')
print(usr)

usr_local = usr / 'local'
print(usr_local)

usr_share = usr / pathlib.PurePosixPath('share')
print(usr_share)

root = usr / '..'
print(root)

etc = root / '/etc/'
print(etc)
```
正如示例输出中root的值所示，运算符在给定路径值时将它们组合在一起，并且当它包含父目录引用“..”时不会对结果进行规范化。 但是，如果段以路径分隔符开头，则它将以与os.path.join（）相同的方式解释为新的“根”引用。 从路径值的中间删除额外路径分隔符，如此处的etc示例中所示。

```
$ python3 pathlib_operator.py

/usr
/usr/local
/usr/share
/usr/..
/etc
```
“具体”的路径类包括一个resolve（）方法来规范化路径，它通过在文件系统查看目录和符号链接来生成绝对路径名。

```python
#pathlib_resolve.py
import pathlib

usr_local = pathlib.Path('/usr/local')
share = usr_local / '..' / 'share'
print(share.resolve())
```
这里相对路径转换为绝对路径`/usr/share`。 如果输入路径包含符号链接，那么也会扩展这些符号链接以允许已解析的路径直接引用目标。

```
$ python3 pathlib_resolve.py

/usr/share
```
要在事先不知道段时构建路径，请使用joinpath（），将每个路径段作为单独的参数传递。

```python
#pathlib_joinpath.py
import pathlib

root = pathlib.PurePosixPath('/')
subdirs = ['usr', 'local']
usr_local = root.joinpath(*subdirs)
print(usr_local)
```
与 `/` 运算符一样，调用joinpath（）会创建一个新实例。

```
$ python3 pathlib_joinpath.py

/usr/local
```
给定一个现有的路径对象，很容易构建一个具有微小差异的新对象，例如引用同一目录中的不同文件。 使用`with_name（）`创建一个新路径，用不同的文件名替换路径的名称部分。 使用`with_suffix（）`创建一个新路径，用不同的值替换文件名的扩展名。

```python
#pathlib_from_existing.py
import pathlib

ind = pathlib.PurePosixPath('source/pathlib/index.rst')
print(ind)

py = ind.with_name('pathlib_from_existing.py')
print(py)

pyc = py.with_suffix('.pyc')
print(pyc)
```
两种方法都返回新对象，原始文件保持不变。

```
$ python3 pathlib_from_existing.py

source/pathlib/index.rst
source/pathlib/pathlib_from_existing.py
source/pathlib/pathlib_from_existing.pyc
```

### Parsing Paths

Path对象具有从名称中提取部分值的方法和属性。 例如，parts属性生成一系列基于路径分隔符解析的路径段。

```python
#pathlib_parts.py
import pathlib

p = pathlib.PurePosixPath('/usr/local')
print(p.parts)
```
序列是一个元组，反映了路径实例的不变性。

```
$ python3 pathlib_parts.py

('/', 'usr', 'local')
```
有两种方法可以从给定的路径对象“向上”导航文件系统层次结构。 parent属性引用包含路径的目录的新路径实例，即os.path.dirname（）返回的值。 parents属性是一个可迭代的，它产生父目录引用，不断地“向上”导航路径层次结构，直到到达根目录。

```python
#pathlib_parents.py
import pathlib

p = pathlib.PurePosixPath('/usr/local/lib')

print('parent: {}'.format(p.parent))

print('\nhierarchy:')
for up in p.parents:
    print(up)
```
该示例遍历parents属性并打印成员值。

```
$ python3 pathlib_parents.py

parent: /usr/local

hierarchy:
/usr/local
/usr
/
```
可以通过路径对象的属性访问路径的其他部分。 name属性保存路径的最后一部分，位于最终路径分隔符之后（与os.path.basename（）生成的值相同）。 suffix属性保存扩展分隔符后面的值，而stem（主干）属性保存后缀之前的名称部分。

```python
#pathlib_name.py
import pathlib

p = pathlib.PurePosixPath('./source/pathlib/pathlib_name.py')
print('path  : {}'.format(p))
print('name  : {}'.format(p.name))
print('suffix: {}'.format(p.suffix))
print('stem  : {}'.format(p.stem))
```
虽然suffix和stem与os.path.splitext（）生成的值类似，但这些值仅基于name的值而不是完整路径。

```
$ python3 pathlib_name.py

path  : source/pathlib/pathlib_name.py
name  : pathlib_name.py
suffix: .py
stem  : pathlib_name
```

### Creating Concrete Paths

可以从“引用到文件系统上的文件、目录或符号链接的名称（或潜在名称）的”字符串参数创建具体Path类的实例。 该类还提供了几种方便的方法，用于使用常用的可更改的位置（例如当前工作目录和用户的主目录）构建实例。

```python
#pathlib_convenience.py
import pathlib

home = pathlib.Path.home()
print('home: ', home)

cwd = pathlib.Path.cwd()
print('cwd : ', cwd)
```
两种方法都创建“预先填充了绝对文件系统引用的”Path实例。

```
$ python3 pathlib_convenience.py

home:  /Users/dhellmann
cwd :  /Users/dhellmann/PyMOTW
```

### Directory Contents

有三种方法可以访问目录的项目列表以发现文件系统上可用文件的名称。 iterdir（）是一个生成器，为包含在目录中的每个项生成一个新的Path实例。

```python
#pathlib_iterdir.py
import pathlib

p = pathlib.Path('.')

for f in p.iterdir():
    print(f)
```
如果Path没有引用到目录，则iterdir（）会引发NotADirectoryError。

```
$ python3 pathlib_iterdir.py

example_link
index.rst
pathlib_chmod.py
pathlib_convenience.py
pathlib_from_existing.py
pathlib_glob.py
pathlib_iterdir.py
pathlib_joinpath.py
pathlib_mkdir.py
pathlib_name.py
pathlib_operator.py
pathlib_ownership.py
pathlib_parents.py
pathlib_parts.py
pathlib_read_write.py
pathlib_resolve.py
pathlib_rglob.py
pathlib_rmdir.py
pathlib_stat.py
pathlib_symlink_to.py
pathlib_touch.py
pathlib_types.py
pathlib_unlink.py
```
使用glob（）只查找与模式匹配的文件。

```python
#pathlib_glob.py
import pathlib

p = pathlib.Path('..')

for f in p.glob('*.rst'):
    print(f)
```
此示例显示这个脚本父目录中的所有reStructuredText（rst）输入文件。

```
$ python3 pathlib_glob.py

../about.rst
../algorithm_tools.rst
../book.rst
../compression.rst
../concurrency.rst
../cryptographic.rst
../data_structures.rst
../dates.rst
../dev_tools.rst
../email.rst
../file_access.rst
../frameworks.rst
../i18n.rst
../importing.rst
../index.rst
../internet_protocols.rst
../language.rst
../networking.rst
../numeric.rst
../persistence.rst
../porting_notes.rst
../runtime_services.rst
../text.rst
../third_party.rst
../unix.rst
```
glob处理器支持使用模式前缀`**`或通过调用rglob（）代替glob（）进行递归扫描。

```python
#pathlib_rglob.py
import pathlib

p = pathlib.Path('..')

for f in p.rglob('pathlib_*.py'):
    print(f)
```
因为此示例从父目录开始，所以需要进行递归搜索以查找与`pathlib_*.py`匹配的示例文件。

```
$ python3 pathlib_rglob.py

../pathlib/pathlib_chmod.py
../pathlib/pathlib_convenience.py
../pathlib/pathlib_from_existing.py
../pathlib/pathlib_glob.py
../pathlib/pathlib_iterdir.py
../pathlib/pathlib_joinpath.py
../pathlib/pathlib_mkdir.py
../pathlib/pathlib_name.py
../pathlib/pathlib_operator.py
../pathlib/pathlib_ownership.py
../pathlib/pathlib_parents.py
../pathlib/pathlib_parts.py
../pathlib/pathlib_read_write.py
../pathlib/pathlib_resolve.py
../pathlib/pathlib_rglob.py
../pathlib/pathlib_rmdir.py
../pathlib/pathlib_stat.py
../pathlib/pathlib_symlink_to.py
../pathlib/pathlib_touch.py
../pathlib/pathlib_types.py
../pathlib/pathlib_unlink.py
```

### Reading and Writing Files

每个Path实例都包含用于处理它所引用的文件内容的方法。要立即检索内容，请使用`read_bytes（）`或`read_text（）`。要写入该文件，请使用`write_bytes（）`或`write_text（）`。使用`open（）`方法打开文件并获取文件句柄，代替将名称传递给内置的open（）函数。

```python
#pathlib_read_write.py
import pathlib

f = pathlib.Path('example.txt')

f.write_bytes('This is the content'.encode('utf-8'))

with f.open('r', encoding='utf-8') as handle:
    print('read from open(): {!r}'.format(handle.read()))

print('read_text(): {!r}'.format(f.read_text('utf-8')))
```
便捷方法在打开文件并写入文件之前会进行一些类型检查，但不同的是它们等同于直接执行操作。

```
$ python3 pathlib_read_write.py

read from open(): 'This is the content'
read_text(): 'This is the content'
```

### Manipulating Directories and Symbolic Links

表示不存在的目录或符号链接的路径，可用于创建关联的文件系统条目。

```python
#pathlib_mkdir.py
import pathlib

p = pathlib.Path('example_dir')

print('Creating {}'.format(p))
p.mkdir()
```
如果路径已存在，则mkdir（）会引发FileExistsError。

```
$ python3 pathlib_mkdir.py

Creating example_dir

$ python3 pathlib_mkdir.py

Creating example_dir
Traceback (most recent call last):
  File "pathlib_mkdir.py", line 16, in <module>
    p.mkdir()
  File ".../lib/python3.6/pathlib.py", line 1226, in mkdir
    self._accessor.mkdir(self, mode)
  File ".../lib/python3.6/pathlib.py", line 387, in wrapped
    return strfunc(str(pathobj), *args)
FileExistsError: [Errno 17] File exists: 'example_dir'
```
使用`symlink_to（）`创建符号链接。 链接将基于路径的值命名，并引用到名称，该名称传给`symlink_to（）`作为参数。

```python
#pathlib_symlink_to.py
import pathlib

p = pathlib.Path('example_link')

p.symlink_to('index.rst')

print(p)
print(p.resolve().name)
```
此示例创建一个符号链接，然后使用resolve（）读取链接以查找它指向的内容并打印名称。

```
$ python3 pathlib_symlink_to.py

example_link
index.rst
```

### File Types

Path实例包括几种用于测试路径引用的文件类型的方法。 此示例创建了几个不同类型的文件，并测试这些文件以及本地操作系统上可用的一些其他特定于设备的文件。

```python
#pathlib_types.py
import itertools
import os
import pathlib

root = pathlib.Path('test_files')

# Clean up from previous runs.
if root.exists():
    for f in root.iterdir():
        f.unlink()
else:
    root.mkdir()

# Create test files
(root / 'file').write_text(
    'This is a regular file', encoding='utf-8')
(root / 'symlink').symlink_to('file')
os.mkfifo(str(root / 'fifo'))

# Check the file types
to_scan = itertools.chain(
    root.iterdir(),
    [pathlib.Path('/dev/disk0'),
     pathlib.Path('/dev/console')],
)
hfmt = '{:18s}' + ('  {:>5}' * 6)
print(hfmt.format('Name', 'File', 'Dir', 'Link', 'FIFO', 'Block',
                  'Character'))
print()

fmt = '{:20s}  ' + ('{!r:>5}  ' * 6)
for f in to_scan:
    print(fmt.format(
        str(f),
        f.is_file(),
        f.is_dir(),
        f.is_symlink(),
        f.is_fifo(),
        f.is_block_device(),
        f.is_char_device(),
    ))
```
每个方法，`is_dir（）`，`is_file（）`，`is_symlink（）`，`is_socket（）`，`is_fifo（）`，`is_block_device（）`和`is_char_device（）`都不带参数。

```
$ python3 pathlib_types.py

Name                 File    Dir   Link   FIFO  Block  Character

test_files/fifo       False  False  False   True  False  False
test_files/file        True  False  False  False  False  False
test_files/symlink     True  False   True  False  False  False
/dev/disk0            False  False  False  False   True  False
/dev/console          False  False  False  False  False   True
```

### File Properties

可以使用方法stat（）或lstat（）（用于检查可能是符号链接的状态）访问有关文件的详细信息。 这些方法产生与os.stat（）和os.lstat（）相同的结果。

```python
#pathlib_stat.py
import pathlib
import sys
import time

if len(sys.argv) == 1:
    filename = __file__
else:
    filename = sys.argv[1]

p = pathlib.Path(filename)
stat_info = p.stat()

print('{}:'.format(filename))
print('  Size:', stat_info.st_size)
print('  Permissions:', oct(stat_info.st_mode))
print('  Owner:', stat_info.st_uid)
print('  Device:', stat_info.st_dev)
print('  Created      :', time.ctime(stat_info.st_ctime))
print('  Last modified:', time.ctime(stat_info.st_mtime))
print('  Last accessed:', time.ctime(stat_info.st_atime))
```
输出将根据示例代码的安装方式而有所不同。 尝试在命令行上将不同的文件名传递给pathlib_stat.py。

```
$ python3 pathlib_stat.py

pathlib_stat.py:
  Size: 607
  Permissions: 0o100644
  Owner: 527
  Device: 16777220
  Created      : Thu Dec 29 12:38:23 2016
  Last modified: Thu Dec 29 12:38:23 2016
  Last accessed: Sun Mar 18 16:21:41 2018

$ python3 pathlib_stat.py index.rst

index.rst:
  Size: 19569
  Permissions: 0o100644
  Owner: 527
  Device: 16777220
  Created      : Sun Mar 18 16:11:31 2018
  Last modified: Sun Mar 18 16:11:31 2018
  Last accessed: Sun Mar 18 16:21:40 2018
```
要更简单地访问有关文件所有者的信息，请使用`owner（）`和`group（）`。

```python
#pathlib_ownership.py
import pathlib

p = pathlib.Path(__file__)

print('{} is owned by {}/{}'.format(p, p.owner(), p.group()))
```
当stat（）返回数字系统ID值时，这些方法会查找与ID关联的名称。

```
$ python3 pathlib_ownership.py

pathlib_ownership.py is owned by dhellmann/dhellmann
```
touch（）方法与Unix命令touch一样，用于创建文件或更新现有文件的修改时间和权限。

```python
#pathlib_touch.py
import pathlib
import time

p = pathlib.Path('touched')
if p.exists():
    print('already exists')
else:
    print('creating new')

p.touch()
start = p.stat()

time.sleep(1)

p.touch()
end = p.stat()

print('Start:', time.ctime(start.st_mtime))
print('End  :', time.ctime(end.st_mtime))
```
不止一次运行此示例会在后续运行中更新现有文件。

```
$ python3 pathlib_touch.py

creating new
Start: Sun Mar 18 16:21:41 2018
End  : Sun Mar 18 16:21:42 2018

$ python3 pathlib_touch.py

already exists
Start: Sun Mar 18 16:21:42 2018
End  : Sun Mar 18 16:21:43 2018
```

### Permissions

在类Unix系统上，可以使用chmod（）更改文件权限，将模式作为整数传递。 可以使用stat模块中定义的常量构造模式值。 此示例切换用户的执行权限位。

```python
#pathlib_chmod.py
import os
import pathlib
import stat

# Create a fresh test file.
f = pathlib.Path('pathlib_chmod_example.txt')
if f.exists():
    f.unlink()
f.write_text('contents')

# Determine what permissions are already set using stat.
existing_permissions = stat.S_IMODE(f.stat().st_mode)
print('Before: {:o}'.format(existing_permissions))

# Decide which way to toggle them.
if not (existing_permissions & os.X_OK):
    print('Adding execute permission')
    new_permissions = existing_permissions | stat.S_IXUSR
else:
    print('Removing execute permission')
    # use xor to remove the user execute permission
    new_permissions = existing_permissions ^ stat.S_IXUSR

# Make the change and show the new value.
f.chmod(new_permissions)
after_permissions = stat.S_IMODE(f.stat().st_mode)
print('After: {:o}'.format(after_permissions))
```
该脚本假定它具有在运行时修改文件模式所需的权限。

```
$ python3 pathlib_chmod.py

Before: 644
Adding execute permission
After: 744
```

### Deleting

根据类型，有两种方法可以从文件系统中删除内容。 要删除空目录，请使用rmdir（）。

```python
#pathlib_rmdir.py
import pathlib

p = pathlib.Path('example_dir')

print('Removing {}'.format(p))
p.rmdir()
```
如果已满足后置条件且目录不存在，则引发FileNotFoundError异常。 尝试删除非空目录也是错误的。

```
$ python3 pathlib_rmdir.py

Removing example_dir

$ python3 pathlib_rmdir.py

Removing example_dir
Traceback (most recent call last):
  File "pathlib_rmdir.py", line 16, in <module>
    p.rmdir()
  File ".../lib/python3.6/pathlib.py", line 1270, in rmdir
    self._accessor.rmdir(self)
  File ".../lib/python3.6/pathlib.py", line 387, in wrapped
    return strfunc(str(pathobj), *args)
FileNotFoundError: [Errno 2] No such file or directory:
'example_dir'
```
对于文件，符号链接和大多数其他路径类型使用unlink（）。

```python
#pathlib_unlink.py
import pathlib

p = pathlib.Path('touched')

p.touch()

print('exists before removing:', p.exists())

p.unlink()

print('exists after removing:', p.exists())
```
用户必须具有删除文件，符号链接，套接字或其他文件系统对象的权限。

```
$ python3 pathlib_unlink.py

exists before removing: True
exists after removing: False
```

## glob — Filename Pattern Matching

目的：使用Unix shell规则查找与模式匹配的文件名。  
尽管glob API很小，但该模块具有很强的功能。 在程序需要在文件系统上查找与模式匹配的文件名称列表的任何情况下，它都很有用。 要创建具有特定的扩展名、前缀或中间的任何公共字符串的文件名列表，请使用glob而不是编写自定义代码来扫描目录内容。  
glob的模式规则与re模块使用的正则表达式不同。 相反，它们遵循标准的Unix路径扩展规则。 只有少数特殊字符用于实现两个不同的通配符和字符范围。 模式规则应用于文件名的段（在路径分隔符处停止，/）。 模式中的路径可以是相对的或绝对的。 Shell变量名称和波浪号（~）不会展开。

### Example Data

本节中的示例假定当前工作目录中存在以下测试文件。

```
$ python3 glob_maketestdata.py

dir
dir/file.txt
dir/file1.txt
dir/file2.txt
dir/filea.txt
dir/fileb.txt
dir/file?.txt
dir/file*.txt
dir/file[.txt
dir/subdir
dir/subdir/subfile.txt
```
如果这些文件不存在，请在运行以下示例之前使用示例代码中的glob_maketestdata.py创建它们。

### Wildcards

星号（*）匹配名称段中的零个或多个字符。 例如，`dir/*`。

```python
#glob_asterisk.py
import glob
for name in sorted(glob.glob('dir/*')):
    print(name)
```
该模式匹配目录dir中的每个路径名（文件或目录），而不会进一步递归到子目录中。 glob（）返回的数据没有排序，因此这里的示例对它进行排序，以便更轻松地查看结果。

```
$ python3 glob_asterisk.py

dir/file*.txt
dir/file.txt
dir/file1.txt
dir/file2.txt
dir/file?.txt
dir/file[.txt
dir/filea.txt
dir/fileb.txt
dir/subdir
```
要列出子目录中的文件，子目录必须包含在模式中。

```python
#glob_subdir.py
import glob

print('Named explicitly:')
for name in sorted(glob.glob('dir/subdir/*')):
    print('  {}'.format(name))

print('Named with wildcard:')
for name in sorted(glob.glob('dir/*/*')):
    print('  {}'.format(name))
```
前面显示的第一种情况明确列出了子目录名称，而第二种情况依赖于通配符来查找目录。

```
$ python3 glob_subdir.py

Named explicitly:
  dir/subdir/subfile.txt
Named with wildcard:
  dir/subdir/subfile.txt
```
在这种情况下，结果是相同的。 如果有另一个子目录，则通配符将匹配两个子目录并包含两者的文件名。

### Single Character Wildcard

问号（？）是另一个通配符。 它匹配名称中该位置的任何单个字符。

```python
#glob_question.py
import glob

for name in sorted(glob.glob('dir/file?.txt')):
    print(name)
```
前面的示例匹配以file开头的所有文件名，还有一个任何类型的字符，然后以.txt结尾。

```
$ python3 glob_question.py

dir/file*.txt
dir/file1.txt
dir/file2.txt
dir/file?.txt
dir/file[.txt
dir/filea.txt
dir/fileb.txt
```

### Character Ranges

使用字符范围（[a-z]）而不是问号来匹配多个字符之一。 此示例查找扩展名前名称中带有数字的所有文件。

```python
#glob_charrange.py
import glob
for name in sorted(glob.glob('dir/*[0-9].*')):
    print(name)
```
字符范围[0-9]匹配任何单个数字。 范围根据每个字母或数字的字符代码排序，短划线表示连续字符的连续范围。 可以写出相同的范围值[0123456789]。

```
$ python3 glob_charrange.py

dir/file1.txt
dir/file2.txt
```

### Escaping Meta-characters

有时需要搜索名称中包含特殊的“blob用于他的模式的”元字符的文件。 escape（）函数使用特殊字符“escapeped”构建一个合适的模式，因此它们不会被glob扩展或解释为特殊字符。

```python
#glob_escape.py
import glob

specials = '?*['

for char in specials:
    pattern = 'dir/*' + glob.escape(char) + '.txt'
    print('Searching for: {!r}'.format(pattern))
    for name in sorted(glob.glob(pattern)):
        print(name)
    print()
```
通过构建包含单个元素的字符范围来转义每个特殊字符。

```
$ python3 glob_escape.py

Searching for: 'dir/*[?].txt'
dir/file?.txt

Searching for: 'dir/*[*].txt'
dir/file*.txt

Searching for: 'dir/*[[].txt'
dir/file[.txt
```

## fnmatch — Unix-style Glob Pattern Matching

目的：处理Unix风格的文件名比较。  
fnmatch模块用于将文件名与unix shell使用的glob样式模式进行比较

### Simple Matching

fnmatch（）将单个文件名与模式进行比较，并返回一个布尔值，指示它们是否匹配。 当操作系统使用区分大小写的文件系统时，比较区分大小写。

```python
#fnmatch_fnmatch.py
import fnmatch
import os

pattern = 'fnmatch_*.py'
print('Pattern :', pattern)
print()

files = os.listdir('.')
for name in sorted(files):
    print('Filename: {:<25} {}'.format(
        name, fnmatch.fnmatch(name, pattern)))
```
在此示例中，模式匹配以“fnmatch_”开头并以“.py”结尾的所有文件。

```
$ python3 fnmatch_fnmatch.py

Pattern : fnmatch_*.py

Filename: fnmatch_filter.py         True
Filename: fnmatch_fnmatch.py        True
Filename: fnmatch_fnmatchcase.py    True
Filename: fnmatch_translate.py      True
Filename: index.rst                 False
```
要强制区分大小写的比较，无论文件系统和操作系统设置如何，请使用fnmatchcase（）。

```python
#fnmatch_fnmatchcase.py
import fnmatch
import os

pattern = 'FNMATCH_*.PY'
print('Pattern :', pattern)
print()

files = os.listdir('.')

for name in sorted(files):
    print('Filename: {:<25} {}'.format(
        name, fnmatch.fnmatchcase(name, pattern)))
```
由于用于测试此程序的OS X系统使用区分大小写的文件系统，因此没有文件与修改后的模式匹配。

```
$ python3 fnmatch_fnmatchcase.py

Pattern : FNMATCH_*.PY

Filename: fnmatch_filter.py         False
Filename: fnmatch_fnmatch.py        False
Filename: fnmatch_fnmatchcase.py    False
Filename: fnmatch_translate.py      False
Filename: index.rst                 False
```

### Filtering

要测试文件名序列，请使用filter（），它返回与pattern参数匹配的名称列表。

```python
#fnmatch_filter.py
import fnmatch
import os
import pprint

pattern = 'fnmatch_*.py'
print('Pattern :', pattern)

files = list(sorted(os.listdir('.')))

print('\nFiles   :')
pprint.pprint(files)

print('\nMatches :')
pprint.pprint(fnmatch.filter(files, pattern))
```
在此示例中，filter（）返回与此部分关联的示例源文件的名称列表。

```
$ python3 fnmatch_filter.py

Pattern : fnmatch_*.py

Files   :
['fnmatch_filter.py',
 'fnmatch_fnmatch.py',
 'fnmatch_fnmatchcase.py',
 'fnmatch_translate.py',
 'index.rst']

Matches :
['fnmatch_filter.py',
 'fnmatch_fnmatch.py',
 'fnmatch_fnmatchcase.py',
 'fnmatch_translate.py']
```

### Translating Patterns

在内部，fnmatch将glob模式转换为正则表达式，并使用re模块比较名称和模式。 translate（）函数是用于将glob模式转换为正则表达式的公共API。

```python
#fnmatch_translate.py
import fnmatch

pattern = 'fnmatch_*.py'
print('Pattern :', pattern)
print('Regex   :', fnmatch.translate(pattern))
```
某些字符被转义以生成有效的表达式。

```
$ python3 fnmatch_translate.py

Pattern : fnmatch_*.py
Regex   : (?s:fnmatch_.*\.py)\Z
```

## linecache

目的：从文件或导入的Python模块中检索文本行，保存结果缓存，以便更有效率的从同一文件中读取更多行。  
在处理Python源文件时，linecache模块在Python标准库的其他部分中使用。 缓存的实现将文件的内容保存在内存中，分解为单独的行。 API通过索引到列表中返回所请求的行，并节省了重复读取文件和解析行以找到所需行的时间。 这在查找同一文件中的多行时尤其有用，例如在为错误报告生成traceback时。

### Test Data

由Lorem Ipsum生成器生成的该文本用作样本输入。

```python
#linecache_data.py
import os
import tempfile

lorem = '''Lorem ipsum dolor sit amet, consectetuer
adipiscing elit.  Vivamus eget elit. In posuere mi non
risus. Mauris id quam posuere lectus sollicitudin
varius. Praesent at mi. Nunc eu velit. Sed augue massa,
fermentum id, nonummy a, nonummy sit amet, ligula. Curabitur
eros pede, egestas at, ultricies ac, apellentesque eu,
tellus.

Sed sed odio sed mi luctus mollis. Integer et nulla ac augue
convallis accumsan. Ut felis. Donec lectus sapien, elementum
nec, condimentum ac, interdum non, tellus. Aenean viverra,
mauris vehicula semper porttitor, ipsum odio consectetuer
lorem, ac imperdiet eros odio a sapien. Nulla mauris tellus,
aliquam non, egestas a, nonummy et, erat. Vivamus sagittis
porttitor eros.'''


def make_tempfile():
    fd, temp_file_name = tempfile.mkstemp()
    os.close(fd)
    with open(temp_file_name, 'wt') as f:
        f.write(lorem)
    return temp_file_name


def cleanup(filename):
    os.unlink(filename)
```

### Reading Specific Lines

linecache模块读取的文件行号以1开头，但通常列表从0开始索引数组。

```python
#linecache_getline.py
import linecache
from linecache_data import *

filename = make_tempfile()

# Pick out the same line from source and cache.
# (Notice that linecache counts from 1)
print('SOURCE:')
print('{!r}'.format(lorem.split('\n')[4]))
print()
print('CACHE:')
print('{!r}'.format(linecache.getline(filename, 5)))

cleanup(filename)
```
返回的每一行都包含一个尾随换行符。

```
$ python3 linecache_getline.py

SOURCE:
'fermentum id, nonummy a, nonummy sit amet, ligula. Curabitur'

CACHE:
'fermentum id, nonummy a, nonummy sit amet, ligula. Curabitur\n'
```
### Handling Blank Lines

返回值始终包含行尾的换行符，因此如果该行为空，则返回值只是换行符。

```python
#linecache_empty_line.py
import linecache
from linecache_data import *

filename = make_tempfile()

# Blank lines include the newline
print('BLANK : {!r}'.format(linecache.getline(filename, 8)))

cleanup(filename)
```

输入文件的第八行不包含文本。

```
$ python3 linecache_empty_line.py

BLANK : '\n'
```

### Error Handling

如果请求的行号超出文件中有效行的范围，则getline（）返回空字符串。

```python
#linecache_out_of_range.py
import linecache
from linecache_data import *

filename = make_tempfile()

# The cache always returns a string, and uses
# an empty string to indicate a line which does
# not exist.
not_there = linecache.getline(filename, 500)
print('NOT THERE: {!r} includes {} characters'.format(
    not_there, len(not_there)))

cleanup(filename)
```
输入文件只有15行，因此请求行500就像尝试读取文件末尾一样。

```
$ python3 linecache_out_of_range.py

NOT THERE: '' includes 0 characters
```
从不存在的文件读取以相同的方式处理。

```python
#linecache_missing_file.py
import linecache

# Errors are even hidden if linecache cannot find the file
no_such_file = linecache.getline(
    'this_file_does_not_exist.txt', 1,
)
print('NO FILE: {!r}'.format(no_such_file))
```
当调用者尝试读取数据时，模块永远不会引发异常。

```
$ python3 linecache_missing_file.py

NO FILE: ''
```

### Reading Python Source Files

由于在生成traceback时大量使用了linecache，因此其主要功能之一是能够通过指定模块的基本名称在导入路径中查找Python模块。

```python
#linecache_path_search.py
import linecache
import os

# Look for the linecache module, using
# the built in sys.path search.
module_line = linecache.getline('linecache.py', 3)
print('MODULE:')
print(repr(module_line))

# Look at the linecache module source directly.
file_src = linecache.__file__
if file_src.endswith('.pyc'):
    file_src = file_src[:-1]
print('\nFILE:')
with open(file_src, 'r') as f:
    file_line = f.readlines()[2]
print(repr(file_line))
```

如果在当前目录中找不到具有该名称的文件，则linecache中的缓存填充代码将在sys.path中搜索指定的模块。 此示例查找linecache.py。由于当前目录中没有副本，因此会找到标准库中的文件。

```
$ python3 linecache_path_search.py

MODULE:
'This is intended to read lines from modules imported -- hence if a filename\n'

FILE:
'This is intended to read lines from modules imported -- hence if a filename\n'
```

## tempfile — Temporary File System Objects

目的： 创建临时文件系统对象

安全地创建具有唯一名称的临时文件，因此想要破坏应用程序或窃取数据的人无法猜到这些文件，或具有挑战性。 tempfile模块提供了几种安全地创建临时文件系统资源的功能。 TemporaryFile（）打开并返回一个未命名的文件，NamedTemporaryFile（）打开并返回一个命名文件，SpooledTemporaryFile在写入磁盘之前将其内容保存在内存中，而TemporaryDirectory是一个上下文管理器，它在关闭上下文时删除该目录。

### Temporary Files

需要临时文件来存储数据而不需要与其他程序共享该文件的应用程序应使用TemporaryFile（）函数来创建文件。 该函数创建一个文件，并在可能的平台上立即unlink。 这使得其他程序无法找到或打开该文件，因为文件系统表中没有对它的引用。 TemporaryFile（）创建的文件在关闭时会自动删除，无论是通过调用close（）还是使用上下文管理器API和with语句

```python
#tempfile_TemporaryFile.py
import os
import tempfile

print('Building a filename with PID:')
filename = '/tmp/guess_my_name.{}.txt'.format(os.getpid())
with open(filename, 'w+b') as temp:
    print('temp:')
    print('  {!r}'.format(temp))
    print('temp.name:')
    print('  {!r}'.format(temp.name))

# Clean up the temporary file yourself.
os.remove(filename)

print()
print('TemporaryFile:')
with tempfile.TemporaryFile() as temp:
    print('temp:')
    print('  {!r}'.format(temp))
    print('temp.name:')
    print('  {!r}'.format(temp.name))

# Automatically cleans up the file.
```

此实例说明了使用一般模式构建文件名与使用TemporaryFile()函数创建临时文件的区别。TemporaryFile()返回的文件没有名称。

```
$ python3 tempfile_TemporaryFile.py

Building a filename with PID:
temp:
  <_io.BufferedRandom name='/tmp/guess_my_name.12151.txt'>
temp.name:
  '/tmp/guess_my_name.12151.txt'

TemporaryFile:
temp:
<_io.BufferedRandom name=4>
temp.name:
4
```

默认情况下，文件句柄使用模式'w + b'创建，因此它在所有平台上都表现一致，并且调用者可以写入并从中读取。

```python
#tempfile_TemporaryFile_binary.py

import os
import tempfile

with tempfile.TemporaryFile() as temp:
    temp.write(b'Some data')
    temp.seek(0)
    print(temp.read())
```

写完后，必须使用seek（）“倒回”文件句柄，以便从中读取数据。

```
$ python3 tempfile_TemporaryFile_binary.py

b'Some data'
```

要在文本模式下打开文件，请在创建文件时将模式设置为“w + t”。

```python
#tempfile_TemporaryFile_text.py
import tempfile

with tempfile.TemporaryFile(mode='w+t') as f:
    f.writelines(['first\n', 'second\n'])
    f.seek(0)
    for line in f:
        print(line.rstrip())
```
文件句柄将数据视为文本。

```
$ python3 tempfile_TemporaryFile_text.py
first
second
```

### Named Files

在某些情况下，命名临时文件很重要。 对于跨多个进程甚至主机的应用程序，命名文件是在应用程序的各个部分之间传递它的最简单方法。 NamedTemporaryFile（）函数创建一个文件而不unlink，因此它保留了其名称（使用name属性访问）。

```python
#tempfile_NamedTemporaryFile.py
import os
import pathlib
import tempfile

with tempfile.NamedTemporaryFile() as temp:
    print('temp:')
    print('  {!r}'.format(temp))
    print('temp.name:')
    print('  {!r}'.format(temp.name))

    f = pathlib.Path(temp.name)

print('Exists after close:', f.exists())
```
句柄关闭后删除该文件。

```
$ python3 tempfile_NamedTemporaryFile.py

temp:
  <tempfile._TemporaryFileWrapper object at 0x1011b2d30>
temp.name:
  '/var/folders/5q/8gk0wq888xlggz008k8dr7180000hg/T/tmps4qh5zde'
Exists after close: False
```

### Spooled Files

对于包含相对少量数据的临时文件，使用SpooledTemporaryFile可能更有效，因为它使用io.BytesIO或io.StringIO缓冲区将文件内容保存在内存中，直到它们达到阈值大小。 当数据量超过阈值时，它将“翻转”并写入磁盘，然后缓冲区将替换为普通的TemporaryFile（）。

```python
#tempfile_SpooledTemporaryFile.py
import tempfile

with tempfile.SpooledTemporaryFile(max_size=100,
                                   mode='w+t',
                                   encoding='utf-8') as temp:
    print('temp: {!r}'.format(temp))

    for i in range(3):
        temp.write('This line is repeated over and over.\n')
        print(temp._rolled, temp._file)
```
此示例使用SpooledTemporaryFile的私有属性来确定何时发生滚动到磁盘。 除非调整缓冲区大小，否则通常无需检查此状态。

```shell
$python3 tempfile_SpooledTemporaryFile.py

temp: <tempfile.SpooledTemporaryFile object at 0x1007b2c88>
False <_io.StringIO object at 0x1007a3d38>
False <_io.StringIO object at 0x1007a3d38>
True <_io.TextIOWrapper name=4 mode='w+t' encoding='utf-8'>
```
要显式地将缓冲区写入磁盘，请调用rollover（）或fileno（）方法。

```python
#tempfile_SpooledTemporaryFile_explicit.py
import tempfile

with tempfile.SpooledTemporaryFile(max_size=1000,
                                   mode='w+t',
                                   encoding='utf-8') as temp:
    print('temp: {!r}'.format(temp))

    for i in range(3):
        temp.write('This line is repeated over and over.\n')
        print(temp._rolled, temp._file)
    print('rolling over')
    temp.rollover()
    print(temp._rolled, temp._file)
```
在此示例中，因为缓冲区大小比数据量大得多，所以除了调用rollover（）之外，不会在磁盘上创建任何文件。

```
$ python3 tempfile_SpooledTemporaryFile_explicit.py

temp: <tempfile.SpooledTemporaryFile object at 0x1007b2c88>
False <_io.StringIO object at 0x1007a3d38>
False <_io.StringIO object at 0x1007a3d38>
False <_io.StringIO object at 0x1007a3d38>
rolling over
True <_io.TextIOWrapper name=4 mode='w+t' encoding='utf-8'>
```

### Temporary Directories

当需要多个临时文件时，使用TemporaryDirectory创建单个临时目录并打开该目录中的所有文件可能更方便。

```python
#tempfile_TemporaryDirectory.py
import pathlib
import tempfile

with tempfile.TemporaryDirectory() as directory_name:
    the_dir = pathlib.Path(directory_name)
    print(the_dir)
    a_file = the_dir / 'a_file.txt'
    a_file.write_text('This file is deleted.')

print('Directory exists after?', the_dir.exists())
print('Contents after:', list(the_dir.glob('*')))
```
上下文管理器生成目录的名称，然后可以在上下文块中使用该名称来构建其他文件名。

```
$ python3 tempfile_TemporaryDirectory.py

/var/folders/5q/8gk0wq888xlggz008k8dr7180000hg/T/tmp_urhiioj
Directory exists after? False
Contents after: []
```

### Predicting Names

虽然不如严格的匿名临时文件安全，在名称中包含可预测部分可以找到该文件并检查它以进行调试。 到目前为止所描述的所有函数都有三个参数来控制文件名到某种程度。 名称使用以下公式生成：

dir + prefix + random + suffix

除random之外的所有值都可以作为参数传递给用于创建临时文件或目录的函数。

```python
#tempfile_NamedTemporaryFile_args.py
import tempfile

with tempfile.NamedTemporaryFile(suffix='_suffix',
                                 prefix='prefix_',
                                 dir='/tmp') as temp:
    print('temp:')
    print('  ', temp)
    print('temp.name:')
    print('  ', temp.name)

```
prefix和suffix参数与随机字符串组合以构建文件名，dir参数按原样使用并用作新文件的位置。

```
$ python3 tempfile_NamedTemporaryFile_args.py

temp:
   <tempfile._TemporaryFileWrapper object at 0x1018b2d68>
temp.name:
   /tmp/prefix_q6wd5czl_suffix
```

### Temporary File Location

如果未使用dir参数给出显式目标，则用于临时文件的路径将根据当前平台和设置而有所不同。 tempfile模块包括两个用于查询运行时使用的该设置的函数。

```python
#tempfile_settings.py
import tempfile

print('gettempdir():', tempfile.gettempdir())
print('gettempprefix():', tempfile.gettempprefix())
```

gettempdir（）返回将保存所有临时文件的默认目录，gettempprefix（）返回新文件和目录名称的字符串前缀。

```
$ python3 tempfile_settings.py

gettempdir(): /var/folders/5q/8gk0wq888xlggz008k8dr7180000hg/T
gettempprefix(): tmp
```

gettempdir（）返回的值是基于直观的算法设置的，该算法查看当前进程可以创建文件的第一个位置列表。 搜索列表是：

1. 环境变量TMPDIR
2. 环境变量TEMP
3. 环境变量TMP
4. 基于平台的后备。 （Windows使用第一个可用的C：\ temp，C：\ tmp，\ temp或\ tmp。其他平台使用/ tmp，/ var / tmp或/ usr / tmp。）
5. 如果找不到其他目录，则使用当前工作目录。

```python
#tempfile_tempdir.py
import tempfile

tempfile.tempdir = '/I/changed/this/path'
print('gettempdir():', tempfile.gettempdir())
```

如果程序需要在不使用任何这些环境变量的情况下对所有临时文件使用一个全局位置，应通过直接设置tempfile.tempdir变量的值。

```
$ python3 tempfile_tempdir.py

gettempdir(): /I/changed/this/path
```

## shutil — High-level File Operations

Purpose:	High-level file operations.
shutil模块包括高级文件操作，例如复制和存档。

### Copying Files

copyfile（）将源的内容复制到目标，如果没有写入目标文件的权限则引发IOError。

```python
#shutil_copyfile.py
import glob
import shutil

print('BEFORE:', glob.glob('shutil_copyfile.*'))

shutil.copyfile('shutil_copyfile.py', 'shutil_copyfile.py.copy')

print('AFTER:', glob.glob('shutil_copyfile.*'))
```
由于该函数打开输入文件以进行读取，因此无论其类型如何，无法使用copyfile（）将特殊文件复制为新的特殊文件(如unix设备节点)。

```
$ python3 shutil_copyfile.py

BEFORE: ['shutil_copyfile.py']
AFTER: ['shutil_copyfile.py', 'shutil_copyfile.py.copy']
```
copyfile（）的实现使用较低级别的函数copyfileobj（）。 虽然copyfile（）的参数是文件名，但copyfileobj（）的参数是打开的文件句柄。 可选的第三个参数是用于读取块的缓冲区长度。

```python
#shutil_copyfileobj.py
import io
import os
import shutil
import sys


class VerboseStringIO(io.StringIO):

    def read(self, n=-1):
        next = io.StringIO.read(self, n)
        print('read({}) got {} bytes'.format(n, len(next)))
        return next


lorem_ipsum = '''Lorem ipsum dolor sit amet, consectetuer
adipiscing elit.  Vestibulum aliquam mollis dolor. Donec
vulputate nunc ut diam. Ut rutrum mi vel sem. Vestibulum
ante ipsum.'''

print('Default:')
input = VerboseStringIO(lorem_ipsum)
output = io.StringIO()
shutil.copyfileobj(input, output)

print()

print('All at once:')
input = VerboseStringIO(lorem_ipsum)
output = io.StringIO()
shutil.copyfileobj(input, output, -1)

print()

print('Blocks of 256:')
input = VerboseStringIO(lorem_ipsum)
output = io.StringIO()
shutil.copyfileobj(input, output, 256)
```
默认行为是使用大块读取。 使用-1一次读取所有输入或另一个正整数来设置特定的块大小。 此示例使用几种不同的块大小来显示效果。

```
$ python3 shutil_copyfileobj.py

Default:
read(16384) got 166 bytes
read(16384) got 0 bytes

All at once:
read(-1) got 166 bytes
read(-1) got 0 bytes

Blocks of 256:
read(256) got 166 bytes
read(256) got 0 bytes
```
copy（）函数解释输出名称，与Unix命令行工具cp一样。 如果指定的目标引用目录而不是文件，则使用源的基本名称在该目录中创建新文件。

```python
#shutil_copy.py
import glob
import os
import shutil

os.mkdir('example')
print('BEFORE:', glob.glob('example/*'))

shutil.copy('shutil_copy.py', 'example')

print('AFTER :', glob.glob('example/*'))
```
文件的权限将与内容一起复制。

```
$ python3 shutil_copy.py

BEFORE: []
AFTER : ['example/shutil_copy.py']
```
copy2（）的工作方式与copy（）类似，但包括复制元数据中的访问和修改时间到新文件。

```python
#shutil_copy2.py
import os
import shutil
import time


def show_file_info(filename):
    stat_info = os.stat(filename)
    print('  Mode    :', oct(stat_info.st_mode))
    print('  Created :', time.ctime(stat_info.st_ctime))
    print('  Accessed:', time.ctime(stat_info.st_atime))
    print('  Modified:', time.ctime(stat_info.st_mtime))


os.mkdir('example')
print('SOURCE:')
show_file_info('shutil_copy2.py')

shutil.copy2('shutil_copy2.py', 'example')

print('DEST:')
show_file_info('example/shutil_copy2.py')
```
新文件具有与旧版本相同的所有特征。

```
$ python3 shutil_copy2.py

SOURCE:
  Mode    : 0o100644
  Created : Wed Dec 28 19:03:12 2016
  Accessed: Wed Dec 28 19:03:49 2016
  Modified: Wed Dec 28 19:03:12 2016
DEST:
  Mode    : 0o100644
  Created : Wed Dec 28 19:03:49 2016
  Accessed: Wed Dec 28 19:03:49 2016
  Modified: Wed Dec 28 19:03:12 2016
```

### Copying File Metadata

默认情况下，在Unix下创建新文件时，它会根据当前用户的umask接收权限。 要将权限从一个文件复制到另一个文件，请使用copymode（）。

```python
#shutil_copymode.py
import os
import shutil
import subprocess

with open('file_to_change.txt', 'wt') as f:
    f.write('content')
os.chmod('file_to_change.txt', 0o444)

print('BEFORE:', oct(os.stat('file_to_change.txt').st_mode))

shutil.copymode('shutil_copymode.py', 'file_to_change.txt')

print('AFTER :', oct(os.stat('file_to_change.txt').st_mode))
```
此示例脚本创建要修改的文件，然后使用copymode（）将脚本的权限复制到示例文件。

```
$ python3 shutil_copymode.py

BEFORE: 0o100444
AFTER : 0o100644
```
要复制有关该文件的其他元数据，请使用copystat（）。

```python
#shutil_copystat.py
import os
import shutil
import time


def show_file_info(filename):
    stat_info = os.stat(filename)
    print('  Mode    :', oct(stat_info.st_mode))
    print('  Created :', time.ctime(stat_info.st_ctime))
    print('  Accessed:', time.ctime(stat_info.st_atime))
    print('  Modified:', time.ctime(stat_info.st_mtime))


with open('file_to_change.txt', 'wt') as f:
    f.write('content')
os.chmod('file_to_change.txt', 0o444)

print('BEFORE:')
show_file_info('file_to_change.txt')

shutil.copystat('shutil_copystat.py', 'file_to_change.txt')

print('AFTER:')
show_file_info('file_to_change.txt')
```
只有与该文件关联的权限和日期与copystat（）重复。

```
$ python3 shutil_copystat.py

BEFORE:
  Mode    : 0o100444
  Created : Wed Dec 28 19:03:49 2016
  Accessed: Wed Dec 28 19:03:49 2016
  Modified: Wed Dec 28 19:03:49 2016
AFTER:
  Mode    : 0o100644
  Created : Wed Dec 28 19:03:49 2016
  Accessed: Wed Dec 28 19:03:49 2016
  Modified: Wed Dec 28 19:03:46 2016
```

### Working With Directory Trees

shutil包含三个用于处理目录树的函数。 要将目录从一个位置复制到另一个位置，请使用copytree（）。 它通过源目录树递归，将文件复制到目标。 目标目录不得提前存在。

```python
#shutil_copytree.py
import glob
import pprint
import shutil

print('BEFORE:')
pprint.pprint(glob.glob('/tmp/example/*'))

shutil.copytree('../shutil', '/tmp/example')

print('\nAFTER:')
pprint.pprint(glob.glob('/tmp/example/*'))
```
symlinks参数控制是将符号链接复制为链接还是文件。 默认设置是将内容复制到新文件。 如果该选项为true，则在目标树中创建新的符号链接。

```
$ python3 shutil_copytree.py

BEFORE:
[]

AFTER:
['/tmp/example/example',
 '/tmp/example/example.out',
 '/tmp/example/file_to_change.txt',
 '/tmp/example/index.rst',
 '/tmp/example/shutil_copy.py',
 '/tmp/example/shutil_copy2.py',
 '/tmp/example/shutil_copyfile.py',
 '/tmp/example/shutil_copyfile.py.copy',
 '/tmp/example/shutil_copyfileobj.py',
 '/tmp/example/shutil_copymode.py',
 '/tmp/example/shutil_copystat.py',
 '/tmp/example/shutil_copytree.py',
 '/tmp/example/shutil_copytree_verbose.py',
 '/tmp/example/shutil_disk_usage.py',
 '/tmp/example/shutil_get_archive_formats.py',
 '/tmp/example/shutil_get_unpack_formats.py',
 '/tmp/example/shutil_make_archive.py',
 '/tmp/example/shutil_move.py',
 '/tmp/example/shutil_rmtree.py',
 '/tmp/example/shutil_unpack_archive.py',
 '/tmp/example/shutil_which.py',
 '/tmp/example/shutil_which_regular_file.py']
```
copytree（）接受两个可调用的参数来控制其行为。 调用ignore参数，并将每个目录或子目录的名称与目录内容列表一起复制。 它应该返回应该复制的项目列表。 调用copy_function参数以实际复制文件。

```python
#shutil_copytree_verbose.py
import glob
import pprint
import shutil


def verbose_copy(src, dst):
    print('copying\n {!r}\n to {!r}'.format(src, dst))
    return shutil.copy2(src, dst)


print('BEFORE:')
pprint.pprint(glob.glob('/tmp/example/*'))
print()

shutil.copytree(
    '../shutil', '/tmp/example',
    copy_function=verbose_copy,
    ignore=shutil.ignore_patterns('*.py'),
)

print('\nAFTER:')
pprint.pprint(glob.glob('/tmp/example/*'))
```
在该示例中，`ignore_patterns（）`用于创建忽略函数以跳过复制Python源文件。 verbose_copy（）在复制文件时打印文件的名称，然后使用copy2（）（默认复制函数）来制作副本。

```
$ python3 shutil_copytree_verbose.py

BEFORE:
[]

copying
 '../shutil/example.out'
 to '/tmp/example/example.out'
copying
 '../shutil/file_to_change.txt'
 to '/tmp/example/file_to_change.txt'
copying
 '../shutil/index.rst'
 to '/tmp/example/index.rst'

AFTER:
['/tmp/example/example',
 '/tmp/example/example.out',
 '/tmp/example/file_to_change.txt',
 '/tmp/example/index.rst']
```
要删除目录及其内容，请使用rmtree（）。

```python

#shutil_rmtree.py
import glob
import pprint
import shutil

print('BEFORE:')
pprint.pprint(glob.glob('/tmp/example/*'))

shutil.rmtree('/tmp/example')

print('\nAFTER:')
pprint.pprint(glob.glob('/tmp/example/*'))
```
默认情况下，错误会作为异常引发，但如果第二个参数为true，则可以忽略错误，并且可以在第三个参数中提供特殊的错误处理函数。

```
$ python3 shutil_rmtree.py

BEFORE:
['/tmp/example/example',
 '/tmp/example/example.out',
 '/tmp/example/file_to_change.txt',
 '/tmp/example/index.rst']

AFTER:
[]
```
要将文件或目录从一个位置移动到另一个位置，请使用move（）。

```python
#shutil_move.py
import glob
import shutil

with open('example.txt', 'wt') as f:
    f.write('contents')

print('BEFORE: ', glob.glob('example*'))

shutil.move('example.txt', 'example.out')

print('AFTER : ', glob.glob('example*'))
```
语义类似于Unix命令mv。 如果源和目标位于同一文件系统中，则重命名源。 否则，将源复制到目标，然后删除源。

```
$ python3 shutil_move.py

BEFORE:  ['example.txt']
AFTER :  ['example.out']
```

### Finding Files

which（）函数扫描搜索路径以查找命名文件。 典型的用例是在shell的环境变量PATH中定义的搜索路径上查找可执行程序。

```python
#shutil_which.py
import shutil

print(shutil.which('virtualenv'))
print(shutil.which('tox'))
print(shutil.which('no-such-program'))
```
如果找不到与搜索参数匹配的文件，则which（）返回None。

```
$ python3 shutil_which.py

/Users/dhellmann/Library/Python/3.5/bin/virtualenv
/Users/dhellmann/Library/Python/3.5/bin/tox
None
```
which（）根据文件具有的权限和要检查的搜索路径来过滤参数。 path参数默认为os.environ（'PATH'），但可以是包含由os.pathsep分隔的目录名的任何字符串。 mode参数应该是与文件权限匹配的位掩码。 默认情况下，掩码查找可执行文件，但以下示例使用可读位掩码和备用搜索路径来查找配置文件。

```python
#shutil_which_regular_file.py
import os
import shutil

path = os.pathsep.join([
    '.',
    os.path.expanduser('~/pymotw'),
])

mode = os.F_OK | os.R_OK

filename = shutil.which(
    'config.ini',
    mode=mode,
    path=path,
)

print(filename)
```
仍然存在以这种方式搜索可读文件的竞争条件，因为在查找文件和实际尝试使用文件之间的时间内，可以删除文件或者可以更改其权限。

```
$ touch config.ini
$ python3 shutil_which_regular_file.py

./config.ini
```

### Archives

Python的标准库包含许多用于管理存档文件的模块，例如tarfile和zipfile。 还有一些用于在shutil中创建和提取存档的高级函数。 `get_archive_formats（）`返回当前系统支持的格式的名称和描述序列。

```python
#shutil_get_archive_formats.py
import shutil

for format, description in shutil.get_archive_formats():
    print('{:<5}: {}'.format(format, description))
```
支持的格式取决于可用的模块和底层库，因此此示例的输出可能会根据其运行位置而更改。

```
$ python3 shutil_get_archive_formats.py

bztar: bzip2'ed tar-file
gztar: gzip'ed tar-file
tar  : uncompressed tar file
xztar: xz'ed tar-file
zip  : ZIP file
```
使用`make_archive（）`创建新的存档文件。 它的输入旨在最好地支持递归地归档整个目录及其所有内容。 默认情况下，它使用当前工作目录，以便所有文件和子目录显示在归档的顶层。 要更改该行为，请使用`root_dir`参数移动到文件系统上的新相对位置，并使用`base_dir`参数指定要添加到存档的目录。

```python
#shutil_make_archive.py
import logging
import shutil
import sys
import tarfile

logging.basicConfig(
    format='%(message)s',
    stream=sys.stdout,
    level=logging.DEBUG,
)
logger = logging.getLogger('pymotw')

print('Creating archive:')
shutil.make_archive(
    'example', 'gztar',
    root_dir='..',
    base_dir='shutil',
    logger=logger,
)

print('\nArchive contents:')
with tarfile.open('example.tar.gz', 'r') as t:
    for n in t.getnames():
        print(n)
```
此示例在shutil示例的源目录中启动，并在文件系统中向上移动一级，然后将shutil目录添加到使用gzip压缩的tar存档中。 日志记录模块配置为显示来自make_archive（）的消息，告知其正在执行的操作。

```
$ python3 shutil_make_archive.py

Creating archive:
changing into '..'
Creating tar archive
changing back to '...'

Archive contents:
shutil
shutil/config.ini
shutil/example.out
shutil/file_to_change.txt
shutil/index.rst
shutil/shutil_copy.py
shutil/shutil_copy2.py
shutil/shutil_copyfile.py
shutil/shutil_copyfileobj.py
shutil/shutil_copymode.py
shutil/shutil_copystat.py
shutil/shutil_copytree.py
shutil/shutil_copytree_verbose.py
shutil/shutil_disk_usage.py
shutil/shutil_get_archive_formats.py
shutil/shutil_get_unpack_formats.py
shutil/shutil_make_archive.py
shutil/shutil_move.py
shutil/shutil_rmtree.py
shutil/shutil_unpack_archive.py
shutil/shutil_which.py
shutil/shutil_which_regular_file.py
```
shutil维护一个可以在当前系统上解压缩的格式注册表，可以通过`get_unpack_formats（）`访问。

```python
#shutil_get_unpack_formats.py
import shutil

for format, exts, description in shutil.get_unpack_formats():
    print('{:<5}: {}, names ending in {}'.format(
        format, description, exts))
```
此注册表与用于创建存档的注册表不同，因为它还包括用于每种格式的公共文件扩展名，以便提取存档的功能可以根据文件扩展名猜出要使用的格式。

```
$ python3 shutil_get_unpack_formats.py

bztar: bzip2'ed tar-file, names ending in ['.tar.bz2', '.tbz2']
gztar: gzip'ed tar-file, names ending in ['.tar.gz', '.tgz']
tar  : uncompressed tar file, names ending in ['.tar']
xztar: xz'ed tar-file, names ending in ['.tar.xz', '.txz']
zip  : ZIP file, names ending in ['.zip']
```
使用unpack_archive（）提取存档，传递存档文件名，并可选择传递应该提取的目录。 如果没有给出目录，则使用当前目录。

```python
#shutil_unpack_archive.py
import pathlib
import shutil
import sys
import tempfile

with tempfile.TemporaryDirectory() as d:
    print('Unpacking archive:')
    shutil.unpack_archive(
        'example.tar.gz',
        extract_dir=d,
    )

    print('\nCreated:')
    prefix_len = len(d) + 1
    for extracted in pathlib.Path(d).rglob('*'):
        print(str(extracted)[prefix_len:])
```
在此示例中，unpack_archive（）能够确定存档的格式，因为文件名以tar.gz结尾，并且该值与解包格式注册表中的gztar格式相关联。

```
$ python3 shutil_unpack_archive.py

Unpacking archive:

Created:
shutil
shutil/config.ini
shutil/example.out
shutil/file_to_change.txt
shutil/index.rst
shutil/shutil_copy.py
shutil/shutil_copy2.py
shutil/shutil_copyfile.py
shutil/shutil_copyfileobj.py
shutil/shutil_copymode.py
shutil/shutil_copystat.py
shutil/shutil_copytree.py
shutil/shutil_copytree_verbose.py
shutil/shutil_disk_usage.py
shutil/shutil_get_archive_formats.py
shutil/shutil_get_unpack_formats.py
shutil/shutil_make_archive.py
shutil/shutil_move.py
shutil/shutil_rmtree.py
shutil/shutil_unpack_archive.py
shutil/shutil_which.py
shutil/shutil_which_regular_file.py
```

### File System Space

在执行可能耗尽该空间的长时间运行操作之前，检查本地文件系统以查看有多少可用空间可能很有用。 disk_usage（）返回一个元组，其中包含总空间，当前使用的数量以及剩余空闲量。

```python
#shutil_disk_usage.py
import shutil

total_b, used_b, free_b = shutil.disk_usage('.')

gib = 2 ** 30  # GiB == gibibyte
gb = 10 ** 9   # GB == gigabyte

print('Total: {:6.2f} GB  {:6.2f} GiB'.format(
    total_b / gb, total_b / gib))
print('Used : {:6.2f} GB  {:6.2f} GiB'.format(
    used_b / gb, used_b / gib))
print('Free : {:6.2f} GB  {:6.2f} GiB'.format(
    free_b / gb, free_b / gib))
```
disk_usage（）返回的值是字节数，因此示例程序在打印它们之前将它们转换为更易读的单元。

```
$ python3 shutil_disk_usage.py

Total: 499.42 GB  465.12 GiB
Used : 246.68 GB  229.73 GiB
Free : 252.48 GB  235.14 GiB
```

## filecmp — 比较文件

目的：比较文件系统上的文件和目录。
filecmp模块包括用于比较文件系统上的文件和目录的函数和类。

### 示例数据

本讨论中的示例使用由filecmp_mkexamples.py创建的一组测试文件。

```python
#filecmp_mkexamples.py
import os


def mkfile(filename, body=None):
    with open(filename, 'w') as f:
        f.write(body or filename)
    return


def make_example_dir(top):
    if not os.path.exists(top):
        os.mkdir(top)
    curdir = os.getcwd()
    os.chdir(top)

    os.mkdir('dir1')
    os.mkdir('dir2')

    mkfile('dir1/file_only_in_dir1')
    mkfile('dir2/file_only_in_dir2')

    os.mkdir('dir1/dir_only_in_dir1')
    os.mkdir('dir2/dir_only_in_dir2')

    os.mkdir('dir1/common_dir')
    os.mkdir('dir2/common_dir')

    mkfile('dir1/common_file', 'this file is the same')
    mkfile('dir2/common_file', 'this file is the same')

    mkfile('dir1/not_the_same')
    mkfile('dir2/not_the_same')

    mkfile('dir1/file_in_dir1', 'This is a file in dir1')
    os.mkdir('dir2/file_in_dir1')

    os.chdir(curdir)
    return


if __name__ == '__main__':
    os.chdir(os.path.dirname(__file__) or os.getcwd())
    make_example_dir('example')
    make_example_dir('example/dir1/common_dir')
    make_example_dir('example/dir2/common_dir')
```
运行该脚本会在example目录下生成一个文件树：


```
$ find example | sort

example
example/dir1
example/dir1/common_dir
example/dir1/common_dir/dir1
example/dir1/common_dir/dir1/common_dir
example/dir1/common_dir/dir1/common_file
example/dir1/common_dir/dir1/dir_only_in_dir1
example/dir1/common_dir/dir1/file_in_dir1
example/dir1/common_dir/dir1/file_only_in_dir1
example/dir1/common_dir/dir1/not_the_same
example/dir1/common_dir/dir2
example/dir1/common_dir/dir2/common_dir
example/dir1/common_dir/dir2/common_file
example/dir1/common_dir/dir2/dir_only_in_dir2
example/dir1/common_dir/dir2/file_in_dir1
example/dir1/common_dir/dir2/file_only_in_dir2
example/dir1/common_dir/dir2/not_the_same
example/dir1/common_file
example/dir1/dir_only_in_dir1
example/dir1/file_in_dir1
example/dir1/file_only_in_dir1
example/dir1/not_the_same
example/dir2
example/dir2/common_dir
example/dir2/common_dir/dir1
example/dir2/common_dir/dir1/common_dir
example/dir2/common_dir/dir1/common_file
example/dir2/common_dir/dir1/dir_only_in_dir1
example/dir2/common_dir/dir1/file_in_dir1
example/dir2/common_dir/dir1/file_only_in_dir1
example/dir2/common_dir/dir1/not_the_same
example/dir2/common_dir/dir2
example/dir2/common_dir/dir2/common_dir
example/dir2/common_dir/dir2/common_file
example/dir2/common_dir/dir2/dir_only_in_dir2
example/dir2/common_dir/dir2/file_in_dir1
example/dir2/common_dir/dir2/file_only_in_dir2
example/dir2/common_dir/dir2/not_the_same
example/dir2/common_file
example/dir2/dir_only_in_dir2
example/dir2/file_in_dir1
example/dir2/file_only_in_dir2
example/dir2/not_the_same
```
在“common_dir”目录下重复相同的目录结构一次，以提供有趣的递归比较选项。

### 文件比较

cmp（）比较文件系统上的两个文件。

```python
#filecmp_cmp.py
import filecmp

print('common_file :', end=' ')
print(filecmp.cmp('example/dir1/common_file',
                  'example/dir2/common_file'),
      end=' ')
print(filecmp.cmp('example/dir1/common_file',
                  'example/dir2/common_file',
                  shallow=False))

print('not_the_same:', end=' ')
print(filecmp.cmp('example/dir1/not_the_same',
                  'example/dir2/not_the_same'),
      end=' ')
print(filecmp.cmp('example/dir1/not_the_same',
                  'example/dir2/not_the_same',
                  shallow=False))

print('identical   :', end=' ')
print(filecmp.cmp('example/dir1/file_only_in_dir1',
                  'example/dir1/file_only_in_dir1'),
      end=' ')
print(filecmp.cmp('example/dir1/file_only_in_dir1',
                  'example/dir1/file_only_in_dir1',
                  shallow=False))
```
shallow参数告诉cmp（）除了元数据之外是否要查看文件的内容。 默认设置是使用os.stat（）中提供的信息执行shallow比较。 如果stat结果相同，则认为文件相同，因此同时创建的相同大小的文件是相同的，即使它们的内容不同。 当shallow为False时，始终会比较文件的内容。


```
$ python3 filecmp_cmp.py

common_file : True True
not_the_same: True False
identical   : True True
```
要比较两个目录中的一组文件而不进行递归，请使用cmpfiles（）。 参数是目录的名称和要在两个位置检查的文件列表。 传入的公共文件列表应仅包含文件名（目录始终导致不匹配），并且文件必须存在于两个位置。 下一个示例显示了构建公共列表的简单方法。 比较也采用shallow 标志，就像cmp（）一样。

```python
#filecmp_cmpfiles.py
import filecmp
import os

# Determine the items that exist in both directories
d1_contents = set(os.listdir('example/dir1'))
d2_contents = set(os.listdir('example/dir2'))
common = list(d1_contents & d2_contents)
common_files = [
    f
    for f in common
    if os.path.isfile(os.path.join('example/dir1', f))
]
print('Common files:', common_files)

# Compare the directories
match, mismatch, errors = filecmp.cmpfiles(
    'example/dir1',
    'example/dir2',
    common_files,
)
print('Match       :', match)
print('Mismatch    :', mismatch)
print('Errors      :', errors)
```
cmpfiles（）返回三个文件名列表，其中包含匹配的文件，不匹配的文件以及无法比较的文件（由于权限问题或任何其他原因）。

```
$ python3 filecmp_cmpfiles.py

Common files: ['not_the_same', 'file_in_dir1', 'common_file']
Match       : ['not_the_same', 'common_file']
Mismatch    : ['file_in_dir1']
Errors      : []
```

### 目录比较

前面描述的功能适用于相对简单的比较。 对于大型目录树的递归比较或更完整的分析，dircmp类更有用。 在最简单的用例中，report（）打印一个两个目录的比较报告。

```python
#filecmp_dircmp_report.py
import filecmp

dc = filecmp.dircmp('example/dir1', 'example/dir2')
dc.report()
```
输出是一个纯文本报告，显示给定目录内容的结果，而不进行递归。 在这种情况下，文件`not_the_same`被认为是相同的，因为没有比较内容。 没有办法让dircmp像cmp（）那样的比较文件内容。

```
$ python3 filecmp_dircmp_report.py

diff example/dir1 example/dir2
Only in example/dir1 : ['dir_only_in_dir1', 'file_only_in_dir1']
Only in example/dir2 : ['dir_only_in_dir2', 'file_only_in_dir2']
Identical files : ['common_file', 'not_the_same']
Common subdirectories : ['common_dir']
Common funny cases : ['file_in_dir1']
```
有关更多详细信息和递归比较，请使用`report_full_closure（）`：

```python
#filecmp_dircmp_report_full_closure.py
import filecmp

dc = filecmp.dircmp('example/dir1', 'example/dir2')
dc.report_full_closure()
```
输出包括所有并行子目录的比较。

```
$ python3 filecmp_dircmp_report_full_closure.py

diff example/dir1 example/dir2
Only in example/dir1 : ['dir_only_in_dir1', 'file_only_in_dir1']
Only in example/dir2 : ['dir_only_in_dir2', 'file_only_in_dir2']
Identical files : ['common_file', 'not_the_same']
Common subdirectories : ['common_dir']
Common funny cases : ['file_in_dir1']

diff example/dir1/common_dir example/dir2/common_dir
Common subdirectories : ['dir1', 'dir2']

diff example/dir1/common_dir/dir1 example/dir2/common_dir/dir1
Identical files : ['common_file', 'file_in_dir1',
'file_only_in_dir1', 'not_the_same']
Common subdirectories : ['common_dir', 'dir_only_in_dir1']

diff example/dir1/common_dir/dir1/dir_only_in_dir1
example/dir2/common_dir/dir1/dir_only_in_dir1

diff example/dir1/common_dir/dir1/common_dir
example/dir2/common_dir/dir1/common_dir

diff example/dir1/common_dir/dir2 example/dir2/common_dir/dir2
Identical files : ['common_file', 'file_only_in_dir2',
'not_the_same']
Common subdirectories : ['common_dir', 'dir_only_in_dir2',
'file_in_dir1']

diff example/dir1/common_dir/dir2/common_dir
example/dir2/common_dir/dir2/common_dir

diff example/dir1/common_dir/dir2/file_in_dir1
example/dir2/common_dir/dir2/file_in_dir1

diff example/dir1/common_dir/dir2/dir_only_in_dir2
example/dir2/common_dir/dir2/dir_only_in_dir2
```

### 在程序中使用差异

除了生成打印报告外，dircmp还计算可直接在程序中使用的文件列表。 仅在请求时计算以下每个属性，因此创建dircmp实例不会产生未使用数据的开销。

```python
#filecmp_dircmp_list.py
import filecmp
import pprint

dc = filecmp.dircmp('example/dir1', 'example/dir2')
print('Left:')
pprint.pprint(dc.left_list)

print('\nRight:')
pprint.pprint(dc.right_list)
```
所比较的目录中包含的文件和子目录列在`left_list`和`right_list`中。

```
$ python3 filecmp_dircmp_list.py

Left:
['common_dir',
 'common_file',
 'dir_only_in_dir1',
 'file_in_dir1',
 'file_only_in_dir1',
 'not_the_same']

Right:
['common_dir',
 'common_file',
 'dir_only_in_dir2',
 'file_in_dir1',
 'file_only_in_dir2',
 'not_the_same']
```

可以通过将要忽略的名称列表传递给构造函数来过滤输入。 默认情况下，将忽略名称RCS，CVS和tags。

```python
#filecmp_dircmp_list_filter.py
import filecmp
import pprint

dc = filecmp.dircmp('example/dir1', 'example/dir2',
                    ignore=['common_file'])

print('Left:')
pprint.pprint(dc.left_list)

print('\nRight:')
pprint.pprint(dc.right_list)
```
在这种情况下，“common_file”被排除在要比较的文件列表之外。

```
$ python3 filecmp_dircmp_list_filter.py

Left:
['common_dir',
 'dir_only_in_dir1',
 'file_in_dir1',
 'file_only_in_dir1',
 'not_the_same']

Right:
['common_dir',
 'dir_only_in_dir2',
 'file_in_dir1',
 'file_only_in_dir2',
 'not_the_same']
```
两个输入目录共有的文件名称保存在common中，每个目录唯一的文件列在`left_only`和`right_only`中。

```python
#filecmp_dircmp_membership.py
import filecmp
import pprint

dc = filecmp.dircmp('example/dir1', 'example/dir2')
print('Common:')
pprint.pprint(dc.common)

print('\nLeft:')
pprint.pprint(dc.left_only)

print('\nRight:')
pprint.pprint(dc.right_only)
```

“left”目录是dircmp（）的第一个参数，“right”目录是第二个。

```
$ python3 filecmp_dircmp_membership.py

Common:
['file_in_dir1', 'common_file', 'common_dir', 'not_the_same']

Left:
['dir_only_in_dir1', 'file_only_in_dir1']

Right:
['file_only_in_dir2', 'dir_only_in_dir2']
```
公共成员可以进一步细分为文件，目录和“有趣”项（两个目录中具有不同类型的任何内容或os.stat（）中存在错误的任何内容）。

```python
#filecmp_dircmp_common.py
import filecmp
import pprint

dc = filecmp.dircmp('example/dir1', 'example/dir2')
print('Common:')
pprint.pprint(dc.common)

print('\nDirectories:')
pprint.pprint(dc.common_dirs)

print('\nFiles:')
pprint.pprint(dc.common_files)

print('\nFunny:')
pprint.pprint(dc.common_funny)
```
在示例数据中，名为`“file_in_dir1”`的项目是一个目录中的文件，另一个目录中是子目录，因此它显示在“有趣”列表中。

```
$ python3 filecmp_dircmp_common.py

Common:
['file_in_dir1', 'common_file', 'common_dir', 'not_the_same']

Directories:
['common_dir']

Files:
['common_file', 'not_the_same']

Funny:
['file_in_dir1']
```
文件之间的差异类似地细分。

```python
#filecmp_dircmp_diff.py
import filecmp

dc = filecmp.dircmp('example/dir1', 'example/dir2')
print('Same      :', dc.same_files)
print('Different :', dc.diff_files)
print('Funny     :', dc.funny_files)
```
文件`not_the_same`仅通过os.stat（）进行比较，并且不检查内容，因此它包含在same_files列表中。

```
$ python3 filecmp_dircmp_diff.py

Same      : ['common_file', 'not_the_same']
Different : []
Funny     : []
```
最后，还保存子目录以便于递归比较。

```python
#filecmp_dircmp_subdirs.py
import filecmp

dc = filecmp.dircmp('example/dir1', 'example/dir2')
print('Subdirectories:')
print(dc.subdirs)
```
subdirs属性是一个字典，将目录名称映射到新的dircmp对象。

```
$ python3 filecmp_dircmp_subdirs.py

Subdirectories:
{'common_dir': <filecmp.dircmp object at 0x1019b2be0>}
```

## mmap — Memory-map Files

目的：内存映射文件，而不是直接读取内容。  
内存映射文件使用操作系统虚拟内存系统直接访问文件系统上的数据，而不是使用普通的I/O函数。 内存映射通常可以提高I / O性能，因为它不需要为每次访问分别进行系统调用，也不需要在缓冲区之间拷贝数据 - 内核和用户应用程序都直接访问内存。

内存映射文件可视为可变字符串或类文件对象，具体取决于需要。 映射文件支持预期的文件API方法，例如close（），flush（），read（），readline（），seek（），tell（）和write（）。 它还支持字符串API，具有切片和find（）等功能。

所有示例都使用文本文件lorem.txt，其中包含一些Lorem Ipsum。 作为参考，该文件的文本是

>lorem.txt  
Lorem ipsum dolor sit amet, consectetuer adipiscing elit.  
Donec egestas, enim et consectetuer ullamcorper, lectus ligula rutrum leo,  
a elementum elit tortor eu quam. Duis tincidunt nisi ut ante. Nulla  
facilisi. Sed tristique eros eu libero. Pellentesque vel  
arcu. Vivamus purus orci, iaculis ac, suscipit sit amet, pulvinar eu,  
lacus. Praesent placerat tortor sed nisl. Nunc blandit diam egestas  
dui. Pellentesque habitant morbi tristique senectus et netus et  
malesuada fames ac turpis egestas. Aliquam viverra fringilla  
leo. Nulla feugiat augue eleifend nulla. Vivamus mauris. Vivamus sed  
mauris in nibh placerat egestas. Suspendisse potenti. Mauris  
massa. Ut eget velit auctor tortor blandit sollicitudin. Suspendisse  
imperdiet justo.

>Note  
Unix和Windows之间mmap（）的参数和行为存在差异，这里没有详细讨论。 有关更多详细信息，请参阅标准库文档。

### Reading

使用mmap（）函数创建内存映射文件。 第一个参数是文件描述符，可以使用file对象的fileno（）方法，也可以使用os.open（）。 调用者负责在调用mmap（）之前打开文件，并在不再需要它之后关闭它。  
mmap（）的第二个参数是要映射的文件部分的字节大小。 如果值为0，则映射整个文件。 如果大小大于文件的当前大小，则扩展文件。
>Note:Windows不支持创建零长度映射。

两个平台都支持可选的关键字参数access。 使用`ACCESS_READ`进行只读访问，使用`ACCESS_WRITE`进行直写（对内存的赋值直接进入文件），或使用`ACCESS_COPY`进行写时复制（不将内存赋值写入文件）。

```python
#mmap_read.py
import mmap

with open('lorem.txt', 'r') as f:
    with mmap.mmap(f.fileno(), 0,
                   access=mmap.ACCESS_READ) as m:
        print('First 10 bytes via read :', m.read(10))
        print('First 10 bytes via slice:', m[:10])
        print('2nd   10 bytes via read :', m.read(10))
```
文件指针跟踪通过切片操作访问的最后一个字节。 在此示例中，指针在第一次读取后向前移动10个字节。 然后通过切片操作将其重置到文件的开头，并通过切片再次向前移动10个字节。 在切片操作之后，再次调用read（）会在文件中提供字节11-20。

```
$ python3 mmap_read.py

First 10 bytes via read : b'Lorem ipsu'
First 10 bytes via slice: b'Lorem ipsu'
2nd   10 bytes via read : b'm dolor si'
```

### Writing

要设置内存映射文件以接收更新，首先在映射之前以附加模式`'r+'`打开它（而不是`'w'`）。 然后使用任何更改数据的API方法（write（），赋值给切片等）。  
下一个示例使用`ACCESS_WRITE`的默认访问模式并给切片赋值以修改行的一部分。

```python
#mmap_write_slice.py
import mmap
import shutil

# Copy the example file
shutil.copyfile('lorem.txt', 'lorem_copy.txt')

word = b'consectetuer'
reversed = word[::-1]
print('Looking for    :', word)
print('Replacing with :', reversed)

with open('lorem_copy.txt', 'r+') as f:
    with mmap.mmap(f.fileno(), 0) as m:
        print('Before:\n{}'.format(m.readline().rstrip()))
        m.seek(0)  # rewind

        loc = m.find(word)
        m[loc:loc + len(word)] = reversed
        m.flush()

        m.seek(0)  # rewind
        print('After :\n{}'.format(m.readline().rstrip()))

        f.seek(0)  # rewind
        print('File  :\n{}'.format(f.readline().rstrip()))
```
“consectetuer”一词在内存和文件的第一行中间被替换。

```
$ python3 mmap_write_slice.py

Looking for    : b'consectetuer'
Replacing with : b'reutetcesnoc'
Before:
b'Lorem ipsum dolor sit amet, consectetuer adipiscing elit.'
After :
b'Lorem ipsum dolor sit amet, reutetcesnoc adipiscing elit.'
File  :
Lorem ipsum dolor sit amet, reutetcesnoc adipiscing elit.
```

### Copy Mode

使用访问设置ACCESS_COPY不会将更改写入磁盘上的文件。

```python
#mmap_write_copy.py
import mmap
import shutil

# Copy the example file
shutil.copyfile('lorem.txt', 'lorem_copy.txt')

word = b'consectetuer'
reversed = word[::-1]

with open('lorem_copy.txt', 'r+') as f:
    with mmap.mmap(f.fileno(), 0,
                   access=mmap.ACCESS_COPY) as m:
        print('Memory Before:\n{}'.format(
            m.readline().rstrip()))
        print('File Before  :\n{}\n'.format(
            f.readline().rstrip()))

        m.seek(0)  # rewind
        loc = m.find(word)
        m[loc:loc + len(word)] = reversed

        m.seek(0)  # rewind
        print('Memory After :\n{}'.format(
            m.readline().rstrip()))

        f.seek(0)
        print('File After   :\n{}'.format(
            f.readline().rstrip()))
```
有必要在此示例中将文件句柄与mmap句柄分开回滚，因为两个对象的内部状态是分开维护的。

```
$ python3 mmap_write_copy.py

Memory Before:
b'Lorem ipsum dolor sit amet, consectetuer adipiscing elit.'
File Before  :
Lorem ipsum dolor sit amet, consectetuer adipiscing elit.

Memory After :
b'Lorem ipsum dolor sit amet, reutetcesnoc adipiscing elit.'
File After   :
Lorem ipsum dolor sit amet, consectetuer adipiscing elit.
```

### Regular Expressions

由于内存映射文件可以像字符串一样工作，因此它可以与对字符串操作的其他模块一起使用，例如正则表达式。 这个例子找到了所有带有“nulla”的句子。

```python
#mmap_regex.py
import mmap
import re

pattern = re.compile(rb'(\.\W+)?([^.]?nulla[^.]*?\.)',
                     re.DOTALL | re.IGNORECASE | re.MULTILINE)

with open('lorem.txt', 'r') as f:
    with mmap.mmap(f.fileno(), 0,
                   access=mmap.ACCESS_READ) as m:
        for match in pattern.findall(m):
            print(match[1].replace(b'\n', b' '))
```
因为模式包含两个组，所以findall（）的返回值是一系列元组。 print语句拉出匹配的句子并用空格替换换行符，以便每个结果打印在一行上。

```
$ python3 mmap_regex.py

b'Nulla facilisi.'
b'Nulla feugiat augue eleifend nulla.'
```

## io — Text, Binary, and Raw Stream I/O Tools

目的：实现文件I/O并提供使用类文件API处理缓冲区的类。  
io模块实现了解释器内置的open（）后面的类，用于基于文件的输入和输出操作。 这些类以这样的方式分解，使它们可以重新组合以用于其他目的，例如以便能够将Unicode数据写入网络套接字。

### In-memory Streams

StringIO提供了一种使用文件API（read（），write（）等）在内存中处理文本的便捷方法。 在某些情况下，使用StringIO构建大型字符串可以比其他一些字符串连接技术性能更好。 内存中的流缓冲区对于测试也很有用，其中写入磁盘上的真实文件可能会降低测试套件的速度。

以下是使用StringIO缓冲区的一些标准示例：

```python
#io_stringio.py
import io

# Writing to a buffer
output = io.StringIO()
output.write('This goes into the buffer. ')
print('And so does this.', file=output)

# Retrieve the value written
print(output.getvalue())

output.close()  # discard buffer memory

# Initialize a read buffer
input = io.StringIO('Inital value for read buffer')

# Read from the buffer
print(input.read())
```
此示例使用read（），但readline（）和readlines（）方法也可用。 StringIO类还提供了一个seek（）方法，用于读取时在缓冲区中跳转，对于rewind很有用如果正在使用预测性解析算法。

```
$ python3 io_stringio.py

This goes into the buffer. And so does this.

Inital value for read buffer
```

要使用原始字节而不是Unicode文本，请使用BytesIO。

```python
#io_bytesio.py
import io

# Writing to a buffer
output = io.BytesIO()
output.write('This goes into the buffer. '.encode('utf-8'))
output.write('ÁÇÊ'.encode('utf-8'))

# Retrieve the value written
print(output.getvalue())

output.close()  # discard buffer memory

# Initialize a read buffer
input = io.BytesIO(b'Inital value for read buffer')

# Read from the buffer
print(input.read())
```
写入BytesIO的值必须是bytes而不是str。

```
$ python3 io_bytesio.py

b'This goes into the buffer. \xc3\x81\xc3\x87\xc3\x8a'
b'Inital value for read buffer'
```

### 包装文本数据的字节流

诸如套接字之类的原始字节流可以包装一层，以处理字符串编码和解码，从而更容易将它们与文本数据一起使用。 TextIOWrapper类支持写入和读取。 write_through参数禁用缓冲，并立即将写入包装器的所有数据刷新到底层缓冲区。

```python
#io_textiowrapper.py
import io

# Writing to a buffer
output = io.BytesIO()
wrapper = io.TextIOWrapper(
    output,
    encoding='utf-8',
    write_through=True,
)
wrapper.write('This goes into the buffer. ')
wrapper.write('ÁÇÊ')

# Retrieve the value written
print(output.getvalue())

output.close()  # discard buffer memory

# Initialize a read buffer
input = io.BytesIO(
    b'Inital value for read buffer with unicode characters ' +
    'ÁÇÊ'.encode('utf-8')
)
wrapper = io.TextIOWrapper(input, encoding='utf-8')

# Read from the buffer
print(wrapper.read())
```
此示例使用BytesIO实例作为流。 [bz2](https://pymotw.com/3/bz2/index.html#module-bz2)，[http.server](https://pymotw.com/3/http.server/index.html#module-http.server)和[subprocess](https://pymotw.com/3/subprocess/index.html#module-subprocess)的示例演示了如何将TextIOWrapper与其他类型的类文件对象一起使用。

```
$ python3 io_textiowrapper.py

b'This goes into the buffer. \xc3\x81\xc3\x87\xc3\x8a'
Inital value for read buffer with unicode characters ÁÇÊ
```

## codecs — String Encoding and Decoding

目的：用于在不同表示之间转换文本的编码器和解码器。  
codecs模块提供用于转码数据的流和文件接口。 它最常用于处理Unicode文本，但其他编码也可用于其他目的。

### Unicode 入门

CPython 3.x区分文本和字节字符串。 bytes实例使用一系列8位字节值。 相反，str字符串在内部作为一系列Unicode code point进行管理。 code point值保存为每个2或4个字节的序列，具体取决于编译Python时给出的选项。  
当输出str值时，使用几种标准方案之一对它们进行编码，以便稍后可以将字节序列重建为相同的文本字符串。 encoded值与code point值字节不一定相同，并且编码定义了在两组值之间进行转换的方式。 读取Unicode数据还需要知道编码方式，以便输入的字节可以转换为unicode类使用的内部表示。  
西方语言最常见的编码是UTF-8和UTF-16，它们分别使用一个和两个字节值的序列来表示每个code point。 另外该编码可以更有效地存储大多数字符不适合用两个字节的code point来表示的语言。
>有关Unicode的更多介绍性信息，请参阅本节末尾的参考列表。 Python Unicode HOWTO特别有用。

### Encodings

理解编码的最佳方法是查看通过以不同方式编码相同字符串而产生的不同字节序列。 以下示例使用此函数格式化字节字符串以使其更易于阅读。

```python
#codecs_to_hex.py
import binascii


def to_hex(t, nbytes):
    """Format text t as a sequence of nbyte long values
    separated by spaces.
    """
    chars_per_item = nbytes * 2
    hex_version = binascii.hexlify(t)
    return b' '.join(
        hex_version[start:start + chars_per_item]
        for start in range(0, len(hex_version), chars_per_item)
    )


if __name__ == '__main__':
    print(to_hex(b'abcdef', 1))
    print(to_hex(b'abcdef', 2))
```
该函数使用binascii获取输入字节字符串的十六进制表示，然后在返回值之前在每个nbytes字节之间插入一个空格。

```
$ python3 codecs_to_hex.py

b'61 62 63 64 65 66'
b'6162 6364 6566'
```
第一个编码示例首先使用unicode类的原始表示形式打印文本'français'，然后使用Unicode数据库中每个字符的名称。 接下来的两行分别将字符串编码为UTF-8和UTF-16，并显示编码产生的十六进制值。

```python
#codecs_encodings.py
import unicodedata
from codecs_to_hex import to_hex

text = 'français'

print('Raw   : {!r}'.format(text))
for c in text:
    print('  {!r}: {}'.format(c, unicodedata.name(c, c)))
print('UTF-8 : {!r}'.format(to_hex(text.encode('utf-8'), 1)))
print('UTF-16: {!r}'.format(to_hex(text.encode('utf-16'), 2)))
```
编码str的结果是bytes对象。

```
$ python3 codecs_encodings.py

Raw   : 'français'
  'f': LATIN SMALL LETTER F
  'r': LATIN SMALL LETTER R
  'a': LATIN SMALL LETTER A
  'n': LATIN SMALL LETTER N
  'ç': LATIN SMALL LETTER C WITH CEDILLA
  'a': LATIN SMALL LETTER A
  'i': LATIN SMALL LETTER I
  's': LATIN SMALL LETTER S
UTF-8 : b'66 72 61 6e c3 a7 61 69 73'
UTF-16: b'fffe 6600 7200 6100 6e00 e700 6100 6900 7300'
```
给定一系列已编码字节作为bytes实例，decode（）方法将它们转换为code point并将序列作为str实例返回。

```python
#codecs_decode.py
from codecs_to_hex import to_hex

text = 'français'
encoded = text.encode('utf-8')
decoded = encoded.decode('utf-8')

print('Original :', repr(text))
print('Encoded  :', to_hex(encoded, 1), type(encoded))
print('Decoded  :', repr(decoded), type(decoded))
```
使用的编码选择不会更改输出类型。

```
$ python3 codecs_decode.py

Original : 'français'
Encoded  : b'66 72 61 6e c3 a7 61 69 73' <class 'bytes'>
Decoded  : 'français' <class 'str'>
```
>Note  
在加载站点时，在解释器启动进程时设置默认编码。 有关默认编码设置的说明，请参阅sys讨论中的Unicode默认值部分。

### Working with Files

在处理I/O操作时，对字符串进行编码和解码尤为重要。 无论是写入文件，套接字还是其他流，数据都必须使用正确的编码。 通常，所有文本数据需要在读取时从其字节表示中解码，并在写入时从内部值编码到特定表示。 程序可以显式地编码和解码数据，但是根据所使用的编码，确定是否已经读取了足够的字节以完全解码数据可能是非常重要的。 codecs提供管理数据编码和解码的类，因此应用程序不必执行此操作。  
codecs提供的最简单的接口是内置open（）函数的替代方法。 新版本的工作方式与内置版本类似，但添加了两个新参数来指定编码和所需的错误处理技术。

```python
#codecs_open_write.py
from codecs_to_hex import to_hex

import codecs
import sys

encoding = sys.argv[1]
filename = encoding + '.txt'

print('Writing to', filename)
with codecs.open(filename, mode='w', encoding=encoding) as f:
    f.write('français')

# Determine the byte grouping to use for to_hex()
nbytes = {
    'utf-8': 1,
    'utf-16': 2,
    'utf-32': 4,
}.get(encoding, 1)

# Show the raw bytes in the file
print('File contents:')
with open(filename, mode='rb') as f:
    print(to_hex(f.read(), nbytes))
```

此示例以带有“ç”的unicode字符串开头，并使用命令行中指定的编码将文本保存到文件中。

```
$ python3 codecs_open_write.py utf-8

Writing to utf-8.txt
File contents:
b'66 72 61 6e c3 a7 61 69 73'

$ python3 codecs_open_write.py utf-16

Writing to utf-16.txt
File contents:
b'fffe 6600 7200 6100 6e00 e700 6100 6900 7300'

$ python3 codecs_open_write.py utf-32

Writing to utf-32.txt
File contents:
b'fffe0000 66000000 72000000 61000000 6e000000 e7000000 61000000 69000000 73000000'
```
使用open（）读取数据很简单，有一个问题：必须事先知道编码，才能正确设置解码器。 某些数据格式（如XML）将编码指定为文件的一部分，但通常由应用程序来管理。 codecs只是将编码作为参数并假设它是正确的。

```python
#codecs_open_read.py
import codecs
import sys

encoding = sys.argv[1]
filename = encoding + '.txt'

print('Reading from', filename)
with codecs.open(filename, mode='r', encoding=encoding) as f:
    print(repr(f.read()))
```

此示例读取前一个程序创建的文件，并将生成的unicode对象的表示形式打印到控制台。

```
$ python3 codecs_open_read.py utf-8

Reading from utf-8.txt
'français'

$ python3 codecs_open_read.py utf-16

Reading from utf-16.txt
'français'

$ python3 codecs_open_read.py utf-32

Reading from utf-32.txt
'français'
```

### Byte Order

诸如UTF-16和UTF-32之类的多字节编码在通过直接复制文件或通过网络通信在不同计算机系统之间传输数据时会产生问题。 不同的系统使用不同字节排序。 数据的这种特性（称为字节序）取决于诸如硬件架构和操作系统和应用程序开发人员所做出的选择等因素。 事先并不总能知道用于给定数据集的字节顺序，因此多字节编码包括字节序标记（BOM）作为编码输出的前几个字节。 例如，UTF-16的定义方式是0xFFFE，可用于指示字节顺序。 codecs定义UTF-16和UTF-32使用的字节顺序标记的常量。

```python
#codecs_bom.py
import codecs
from codecs_to_hex import to_hex

BOM_TYPES = [
    'BOM', 'BOM_BE', 'BOM_LE',
    'BOM_UTF8',
    'BOM_UTF16', 'BOM_UTF16_BE', 'BOM_UTF16_LE',
    'BOM_UTF32', 'BOM_UTF32_BE', 'BOM_UTF32_LE',
]

for name in BOM_TYPES:
    print('{:12} : {}'.format(
        name, to_hex(getattr(codecs, name), 2)))
```
`BOM`，`BOM_UTF16`和`BOM_UTF32`会自动设置为适当的big-endian或little-endian值，具体取决于当前系统的本机字节顺序。

```
$ python3 codecs_bom.py

BOM          : b'fffe'
BOM_BE       : b'feff'
BOM_LE       : b'fffe'
BOM_UTF8     : b'efbb bf'
BOM_UTF16    : b'fffe'
BOM_UTF16_BE : b'feff'
BOM_UTF16_LE : b'fffe'
BOM_UTF32    : b'fffe 0000'
BOM_UTF32_BE : b'0000 feff'
BOM_UTF32_LE : b'fffe 0000'
```
字节序由codecs中的解码器自动检测和处理，但是在编码时可以显式指定字节序。

```python
#codecs_bom_create_file.py
import codecs
from codecs_to_hex import to_hex

# Pick the nonnative version of UTF-16 encoding
if codecs.BOM_UTF16 == codecs.BOM_UTF16_BE:
    bom = codecs.BOM_UTF16_LE
    encoding = 'utf_16_le'
else:
    bom = codecs.BOM_UTF16_BE
    encoding = 'utf_16_be'

print('Native order  :', to_hex(codecs.BOM_UTF16, 2))
print('Selected order:', to_hex(bom, 2))

# Encode the text.
encoded_text = 'français'.encode(encoding)
print('{:14}: {}'.format(encoding, to_hex(encoded_text, 2)))

with open('nonnative-encoded.txt', mode='wb') as f:
    # Write the selected byte-order marker.  It is not included
    # in the encoded text because the byte order was given
    # explicitly when selecting the encoding.
    f.write(bom)
    # Write the byte string for the encoded text.
    f.write(encoded_text)
```
`codecs_bom_create_file.py`计算出本机字节排序，然后明确使用相反字节序，以便下一个示例可以在读取时演示自动检测。

```
$ python3 codecs_bom_create_file.py

Native order  : b'fffe'
Selected order: b'feff'
utf_16_be     : b'0066 0072 0061 006e 00e7 0061 0069 0073'
```
打开文件时，`codecs_bom_detection.py`没有指定字节顺序，因此解码器使用文件前两个字节中的BOM值来确定它。

```python
#codecs_bom_detection.py
import codecs
from codecs_to_hex import to_hex

# Look at the raw data
with open('nonnative-encoded.txt', mode='rb') as f:
    raw_bytes = f.read()

print('Raw    :', to_hex(raw_bytes, 2))

# Re-open the file and let codecs detect the BOM
with codecs.open('nonnative-encoded.txt',
                 mode='r',
                 encoding='utf-16',
                 ) as f:
    decoded_text = f.read()

print('Decoded:', repr(decoded_text))
```
由于文件的前两个字节用于字节顺序检测，因此它们不包含在read（）返回的数据中。

```
$ python3 codecs_bom_detection.py

Raw    : b'feff 0066 0072 0061 006e 00e7 0061 0069 0073'
Decoded: 'français'
```

### Error Handling

前面的部分指出需要知道在读取和写入Unicode文件时使用的编码。 正确设置编码很重要，原因有两个。 如果在从文件读取时编码配置不正确，则数据将被解释错误并且可能崩溃或无法解码。 并非所有Unicode字符都可以在所有编码中表示，因此如果在写入时使用了错误的编码，则会生成错误并且数据可能会丢失。

codecs使用由str的encode（）方法和字节的decode（）方法提供的相同的五个错误处理选项，如下表所示。

Codec Error Handling Modes

Error Mode |	Description
:--- | :---
strict	| Raises an exception if the data cannot be converted.
replace	| Substitutes a special marker character for data that cannot be encoded.
ignore	| Skips the data.
xmlcharrefreplace	|XML character (encoding only)
backslashreplace	|escape sequence (encoding only)

### Encoding Errors

最常见的错误情况是收到UnicodeEncodeError，当把Unicode数据写入ASCII输出流时，例如没有设置更健壮的编码的常规文件或sys.stdout。 此示例程序可用于试验不同的错误处理模式。

```python
#codecs_encode_error.py
import codecs
import sys

error_handling = sys.argv[1]

text = 'français'

try:
    # Save the data, encoded as ASCII, using the error
    # handling mode specified on the command line.
    with codecs.open('encode_error.txt', 'w',
                     encoding='ascii',
                     errors=error_handling) as f:
        f.write(text)

except UnicodeEncodeError as err:
    print('ERROR:', err)

else:
    # If there was no error writing to the file,
    # show what it contains.
    with open('encode_error.txt', 'rb') as f:
        print('File contents: {!r}'.format(f.read()))
```
虽然strict模式对于确保应用程序显式为所有I / O操作设置正确的编码是最安全的，但是当引发异常时，它可能导致程序崩溃。

```
$ python3 codecs_encode_error.py strict

ERROR: 'ascii' codec can't encode character '\xe7' in position
4: ordinal not in range(128)
```
一些其他错误模式更灵活。 例如，replace确保不会引发错误，代价是可能丢失无法转换为请求编码的数据。 pi的Unicode字符仍然不能用ASCII编码，但是不是引发异常，而是将字符替换为？ 在输出中。

```
$ python3 codecs_encode_error.py replace

File contents: b'fran?ais'
```
要完全跳过问题数据，请使用ignore。 任何无法编码的数据都将被丢弃。

```
$ python3 codecs_encode_error.py ignore

File contents: b'franais'
```
有两个无损错误处理选项，这两个选项都使用由 与编码分开的标准定义的替代表示 替换字符。 xmlcharrefreplace使用XML字符引用作为替代（字符引用列表指定为W3C文档XML字符实体定义）。

```
$ python3 codecs_encode_error.py xmlcharrefreplace

File contents: b'fran&#231;ais'
```
另一个无损错误处理方案是backslashreplace，它产生的输出格式类似于打印unicode对象的repr（）时返回的值。 Unicode字符替换为\u，后跟code point的十六进制值。

```
$ python3 codecs_encode_error.py backslashreplace

File contents: b'fran\\xe7ais'
```

### Decoding Errors

在解码数据时也可能会看到错误，尤其是在使用错误编码的情况下。

```python
#codecs_decode_error.py
import codecs
import sys

from codecs_to_hex import to_hex

error_handling = sys.argv[1]

text = 'français'
print('Original     :', repr(text))

# Save the data with one encoding
with codecs.open('decode_error.txt', 'w',
                 encoding='utf-16') as f:
    f.write(text)

# Dump the bytes from the file
with open('decode_error.txt', 'rb') as f:
    print('File contents:', to_hex(f.read(), 1))

# Try to read the data with the wrong encoding
with codecs.open('decode_error.txt', 'r',
                 encoding='utf-8',
                 errors=error_handling) as f:
    try:
        data = f.read()
    except UnicodeDecodeError as err:
        print('ERROR:', err)
    else:
        print('Read         :', repr(data))
```
与编码一样，如果无法正确解码字节流，则strict的错误处理模式会引发异常。 在这种情况下，通过尝试使用UTF-8解码器将部分UTF-16 BOM转换为字符而产生UnicodeDecodeError。

```
$ python3 codecs_decode_error.py strict

Original     : 'français'
File contents: b'ff fe 66 00 72 00 61 00 6e 00 e7 00 61 00 69 00
73 00'
ERROR: 'utf-8' codec can't decode byte 0xff in position 0:
invalid start byte
```
切换到忽略会导致解码器跳过无效字节。 但结果仍然不尽如人意，因为它包含嵌入的空字节。

```
$ python3 codecs_decode_error.py ignore

Original     : 'français'
File contents: b'ff fe 66 00 72 00 61 00 6e 00 e7 00 61 00 69 00
73 00'
Read         : 'f\x00r\x00a\x00n\x00\x00a\x00i\x00s\x00'
```
在replace模式中，无效字节被替换为\uFFFD，这是官方的Unicode替换字符，看起来像一个黑色背景包含白色问号的钻石。

```
$ python3 codecs_decode_error.py replace

Original     : 'français'
File contents: b'ff fe 66 00 72 00 61 00 6e 00 e7 00 61 00 69 00
73 00'
Read         : '��f\x00r\x00a\x00n\x00�\x00a\x00i\x00s\x00'
```

### Encoding Translation

虽然大多数应用程序将在内部使用str数据，将解码或编码作为I/O操作的一部分，但有时更改文件的编码而不保留该中间数据格式是有用的。 EncodedFile（）使用一个编码获取一个打开的文件句柄，并用一个类将其包装，该类在发生I/O时将数据转换为另一个编码。

```python
#codecs_encodedfile.py
from codecs_to_hex import to_hex

import codecs
import io

# Raw version of the original data.
data = 'français'

# Manually encode it as UTF-8.
utf8 = data.encode('utf-8')
print('Start as UTF-8   :', to_hex(utf8, 1))

# Set up an output buffer, then wrap it as an EncodedFile.
output = io.BytesIO()
encoded_file = codecs.EncodedFile(output, data_encoding='utf-8',
                                  file_encoding='utf-16')
encoded_file.write(utf8)

# Fetch the buffer contents as a UTF-16 encoded byte string
utf16 = output.getvalue()
print('Encoded to UTF-16:', to_hex(utf16, 2))

# Set up another buffer with the UTF-16 data for reading,
# and wrap it with another EncodedFile.
buffer = io.BytesIO(utf16)
encoded_file = codecs.EncodedFile(buffer, data_encoding='utf-8',
                                  file_encoding='utf-16')

# Read the UTF-8 encoded version of the data.
recoded = encoded_file.read()
print('Back to UTF-8    :', to_hex(recoded, 1))
```
此示例显示读取和写入EncodedFile（）返回的单独句柄。 无论句柄是用于读取还是写入，`file_encoding`始终引用作为第一个参数传递的打开文件句柄使用的编码，而`data_encoding`值指的是通过read（）、write（）传递的数据使用的编码。

```
$ python3 codecs_encodedfile.py

Start as UTF-8   : b'66 72 61 6e c3 a7 61 69 73'
Encoded to UTF-16: b'fffe 6600 7200 6100 6e00 e700 6100 6900
7300'
Back to UTF-8    : b'66 72 61 6e c3 a7 61 69 73'
```

### Non-Unicode Encodings

虽然大多数早期示例使用Unicode编码，但codecs可用于许多其他数据转换。 例如，Python包含用于处理base-64，bzip2，ROT-13，ZIP和其他数据格式的codecs。

```python
#codecs_rot13.py
import codecs
import io

buffer = io.StringIO()
stream = codecs.getwriter('rot_13')(buffer)

text = 'abcdefghijklmnopqrstuvwxyz'

stream.write(text)
stream.flush()

print('Original:', text)
print('ROT-13  :', buffer.getvalue())
```

任何转换，如果可以表示为采用单个输入参数并返回字节或Unicode字符串的函数，都可以注册为codec。 对于'rot_13' codec，输入应该是Unicode字符串，输出也是Unicode字符串。

```
$ python3 codecs_rot13.py

Original: abcdefghijklmnopqrstuvwxyz
ROT-13  : nopqrstuvwxyzabcdefghijklm
```
使用codecs包装数据流提供了比直接使用zlib更简单的接口。

```python
#codecs_zlib.py
import codecs
import io

from codecs_to_hex import to_hex

buffer = io.BytesIO()
stream = codecs.getwriter('zlib')(buffer)

text = b'abcdefghijklmnopqrstuvwxyz\n' * 50

stream.write(text)
stream.flush()

print('Original length :', len(text))
compressed_data = buffer.getvalue()
print('ZIP compressed  :', len(compressed_data))

buffer = io.BytesIO(compressed_data)
stream = codecs.getreader('zlib')(buffer)

first_line = stream.readline()
print('Read first line :', repr(first_line))

uncompressed_data = first_line + stream.read()
print('Uncompressed    :', len(uncompressed_data))
print('Same            :', text == uncompressed_data)
```
并非所有压缩或编码系统都支持使用readline（）或read（）通过流接口读取部分数据，因为它们需要找到压缩段的末尾以扩展它。 如果程序无法将整个未压缩数据集保存在内存中，请使用压缩库的增量访问功能，而不是codecs。

```
$ python3 codecs_zlib.py

Original length : 1350
ZIP compressed  : 48
Read first line : b'abcdefghijklmnopqrstuvwxyz\n'
Uncompressed    : 1350
Same            : True
```

### Incremental Encoding

提供的一些编码，尤其是bz2和zlib，可能会在数据流处理时显着改变数据流的长度。 对于大型数据集，这些编码以递增方式运行，一次处理一小块数据。 IncrementalEncoder和IncrementalDecoder API就是为此目的而设计的。

```python
#codecs_incremental_bz2.py
import codecs
import sys

from codecs_to_hex import to_hex

text = b'abcdefghijklmnopqrstuvwxyz\n'
repetitions = 50

print('Text length :', len(text))
print('Repetitions :', repetitions)
print('Expected len:', len(text) * repetitions)

# Encode the text several times to build up a
# large amount of data
encoder = codecs.getincrementalencoder('bz2')()
encoded = []

print()
print('Encoding:', end=' ')
last = repetitions - 1
for i in range(repetitions):
    en_c = encoder.encode(text, final=(i == last))
    if en_c:
        print('\nEncoded : {} bytes'.format(len(en_c)))
        encoded.append(en_c)
    else:
        sys.stdout.write('.')

all_encoded = b''.join(encoded)
print()
print('Total encoded length:', len(all_encoded))
print()

# Decode the byte string one byte at a time
decoder = codecs.getincrementaldecoder('bz2')()
decoded = []

print('Decoding:', end=' ')
for i, b in enumerate(all_encoded):
    final = (i + 1) == len(text)
    c = decoder.decode(bytes([b]), final)
    if c:
        print('\nDecoded : {} characters'.format(len(c)))
        print('Decoding:', end=' ')
        decoded.append(c)
    else:
        sys.stdout.write('.')
print()

restored = b''.join(decoded)

print()
print('Total uncompressed length:', len(restored))
```
每次将数据传递给编码器或解码器时，其内部状态都会更新。 当状态一致时（由编解码器定义），返回数据并重置状态。 在此之前，对encode（）或decode（）的调用不会返回任何数据。 传入最后一位数据时，参数final应设置为True，以便编解码器知道刷新任何剩余的缓冲数据。

```
$ python3 codecs_incremental_bz2.py

Text length : 27
Repetitions : 50
Expected len: 1350

Encoding: .................................................
Encoded : 99 bytes

Total encoded length: 99

Decoding: ......................................................
..................................
Decoded : 1350 characters
Decoding: ..........

Total uncompressed length: 1350
```

### Unicode Data and Network Communication

网络套接字是字节流，与标准输入和输出流不同，它们默认不支持编码。 这意味着想要通过网络发送或接收Unicode数据的程序必须在写入套接字之前编码为字节。 此服务器将其收到的数据回送给发件人。

```python
#codecs_socket_fail.py
import sys
import socketserver


class Echo(socketserver.BaseRequestHandler):

    def handle(self):
        # Get some bytes and echo them back to the client.
        data = self.request.recv(1024)
        self.request.send(data)
        return


if __name__ == '__main__':
    import codecs
    import socket
    import threading

    address = ('localhost', 0)  # let the kernel assign a port
    server = socketserver.TCPServer(address, Echo)
    ip, port = server.server_address  # what port was assigned?

    t = threading.Thread(target=server.serve_forever)
    t.setDaemon(True)  # don't hang on exit
    t.start()

    # Connect to the server
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((ip, port))

    # Send the data
    # WRONG: Not encoded first!
    text = 'français'
    len_sent = s.send(text)

    # Receive a response
    response = s.recv(len_sent)
    print(repr(response))

    # Clean up
    s.close()
    server.socket.close()
```

可以在每次调用send（）之前显式编码数据，但缺少对send（）的一次调用会导致编码错误。

```
$ python3 codecs_socket_fail.py

Traceback (most recent call last):
  File "codecs_socket_fail.py", line 43, in <module>
    len_sent = s.send(text)
TypeError: a bytes-like object is required, not 'str'
```
使用makefile（）获取套接字的类文件句柄，然后使用基于流的reader或writer包装它，意味着Unicode字符串将在进出套接字的路上进行编码。

```python
#codecs_socket.py
import sys
import socketserver


class Echo(socketserver.BaseRequestHandler):

    def handle(self):
        """Get some bytes and echo them back to the client.

        There is no need to decode them, since they are not used.

        """
        data = self.request.recv(1024)
        self.request.send(data)


class PassThrough:

    def __init__(self, other):
        self.other = other

    def write(self, data):
        print('Writing :', repr(data))
        return self.other.write(data)

    def read(self, size=-1):
        print('Reading :', end=' ')
        data = self.other.read(size)
        print(repr(data))
        return data

    def flush(self):
        return self.other.flush()

    def close(self):
        return self.other.close()


if __name__ == '__main__':
    import codecs
    import socket
    import threading

    address = ('localhost', 0)  # let the kernel assign a port
    server = socketserver.TCPServer(address, Echo)
    ip, port = server.server_address  # what port was assigned?

    t = threading.Thread(target=server.serve_forever)
    t.setDaemon(True)  # don't hang on exit
    t.start()

    # Connect to the server
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((ip, port))

    # Wrap the socket with a reader and writer.
    read_file = s.makefile('rb')
    incoming = codecs.getreader('utf-8')(PassThrough(read_file))
    write_file = s.makefile('wb')
    outgoing = codecs.getwriter('utf-8')(PassThrough(write_file))

    # Send the data
    text = 'français'
    print('Sending :', repr(text))
    outgoing.write(text)
    outgoing.flush()

    # Receive a response
    response = incoming.read()
    print('Received:', repr(response))

    # Clean up
    s.close()
    server.socket.close()
```
此示例使用PassThrough展示：在客户端，数据在发送之前被编码，并且响应在接收后被解码。

```
$ python3 codecs_socket.py

Sending : 'français'
Writing : b'fran\xc3\xa7ais'
Reading : b'fran\xc3\xa7ais'
Reading : b''
Received: 'français'
```

### Defining a Custom Encoding

由于Python已经提供了大量标准编解码器，因此应用程序不太可能需要定义自定义编码器或解码器。 但是，在必要时，codecs中有几个基类可以使自定义编解码器更容易。  
第一步是理解编码描述的转换的本质。 这些示例将使用“invertcaps”编码，该编码将大写字母转换为小写，将小写字母转换为大写。下面是对输入字符串执行此转换的编码函数的简单定义。

```python
#codecs_invertcaps.py
import string


def invertcaps(text):
    """Return new string with the case of all letters switched.
    """
    return ''.join(
        c.upper() if c in string.ascii_lowercase
        else c.lower() if c in string.ascii_uppercase
        else c
        for c in text
    )


if __name__ == '__main__':
    print(invertcaps('ABCdef'))
    print(invertcaps('abcDEF'))
```
在这种情况下，编码器和解码器具有相同的功能（ROT-13的情况也是如此）。

```
$ python3 codecs_invertcaps.py

abcDEF
ABCdef
```
虽然很容易理解，但这种实现效率不高，特别是对于非常大的文本字符串。 幸运的是，codecs包括一些辅助函数，用于创建基于字符映射的编解码器，例如invertcaps。 字符映射编码由两个字典组成。 编码映射将输入字符串中的字符值转换为输出中的字节值，而解码映射则转换为另一种方式。 首先创建解码映射，然后使用`make_encoding_map（）`将其转换为编码映射。 C函数`charmap_encode（）`和`charmap_decode（）`使用映射有效地转换其输入数据。

```python
#codecs_invertcaps_charmap.py
import codecs
import string

# Map every character to itself
decoding_map = codecs.make_identity_dict(range(256))

# Make a list of pairs of ordinal values for the lower
# and uppercase letters
pairs = list(zip(
    [ord(c) for c in string.ascii_lowercase],
    [ord(c) for c in string.ascii_uppercase],
))

# Modify the mapping to convert upper to lower and
# lower to upper.
decoding_map.update({
    upper: lower
    for (lower, upper)
    in pairs
})
decoding_map.update({
    lower: upper
    for (lower, upper)
    in pairs
})

# Create a separate encoding map.
encoding_map = codecs.make_encoding_map(decoding_map)

if __name__ == '__main__':
    print(codecs.charmap_encode('abcDEF', 'strict',
                                encoding_map))
    print(codecs.charmap_decode(b'abcDEF', 'strict',
                                decoding_map))
    print(encoding_map == decoding_map)
```
虽然“invertcaps”编码和解码映射是相同的，但情况并非总是如此。 `make_encoding_map（）`检测将多个输入字符编码到同一输出字节并将编码值替换为None以将编码标记为未定义的情况。

```
$ python3 codecs_invertcaps_charmap.py

(b'ABCdef', 6)
('ABCdef', 6)
True
```
字符映射编码器和解码器支持前面描述的所有标准错误处理方法，因此不需要额外的工作来遵守API的那一部分。

```python
#codecs_invertcaps_error.py
import codecs
from codecs_invertcaps_charmap import encoding_map

text = 'pi: \u03c0'

for error in ['ignore', 'replace', 'strict']:
    try:
        encoded = codecs.charmap_encode(
            text, error, encoding_map)
    except UnicodeEncodeError as err:
        encoded = str(err)
    print('{:7}: {}'.format(error, encoded))
```
由于π的Unicode代码点不在编码映射中，因此严格的错误处理模式会引发异常。

```
$ python3 codecs_invertcaps_error.py

ignore : (b'PI: ', 5)
replace: (b'PI: ?', 5)
strict : 'charmap' codec can't encode character '\u03c0' in
position 4: character maps to <undefined>
```
在定义了编码和解码映射之后，需要设置一些额外的类，并且该编码应该被注册。 register（）向注册表添加搜索函数，以便当用户想要使用编码时，codecs可以找到它。 搜索函数必须使用带有编码名称的单个字符串参数，并在知道编码时返回CodecInfo对象，否则返回None。

```python
#codecs_register.py
import codecs
import encodings

def search1(encoding):
    print('search1: Searching for:', encoding)
    return None

def search2(encoding):
    print('search2: Searching for:', encoding)
    return None

codecs.register(search1)
codecs.register(search2)

utf8 = codecs.lookup('utf-8')
print('UTF-8:', utf8)

try:
    unknown = codecs.lookup('no-such-encoding')
except LookupError as err:
    print('ERROR:', err)
```
可以注册多个搜索函数，并且每个搜索函数将依次被调用，直到返回CodecInfo或列表用尽为止。 codecs注册的内部搜索函数知道如何从编码中加载标准编解码器（如UTF-8），因此这些名称永远不会传递给自定义搜索函数。

```
$ python3 codecs_register.py

UTF-8: <codecs.CodecInfo object for encoding utf-8 at
0x1007773a8>
search1: Searching for: no-such-encoding
search2: Searching for: no-such-encoding
ERROR: unknown encoding: no-such-encoding
```
搜索函数返回的CodecInfo实例告诉codecs如何使用所支持的所有不同机制进行编码和解码：无状态，增量和流。 codecs包括有助于设置字符映射编码的基类。 此示例将所有部分放在一起以注册搜索函数，该函数返回为invertcaps编解码器配置的CodecInfo实例。

```python
#codecs_invertcaps_register.py
import codecs

from codecs_invertcaps_charmap import encoding_map, decoding_map

class InvertCapsCodec(codecs.Codec):
    "Stateless encoder/decoder"

    def encode(self, input, errors='strict'):
        return codecs.charmap_encode(input, errors, encoding_map)

    def decode(self, input, errors='strict'):
        return codecs.charmap_decode(input, errors, decoding_map)

class InvertCapsIncrementalEncoder(codecs.IncrementalEncoder):
    def encode(self, input, final=False):
        data, nbytes = codecs.charmap_encode(input,
                                             self.errors,
                                             encoding_map)
        return data

class InvertCapsIncrementalDecoder(codecs.IncrementalDecoder):
    def decode(self, input, final=False):
        data, nbytes = codecs.charmap_decode(input,
                                             self.errors,
                                             decoding_map)
        return data

class InvertCapsStreamReader(InvertCapsCodec,
                             codecs.StreamReader):
    pass

class InvertCapsStreamWriter(InvertCapsCodec,
                             codecs.StreamWriter):
    pass

def find_invertcaps(encoding):
    """Return the codec for 'invertcaps'.
    """
    if encoding == 'invertcaps':
        return codecs.CodecInfo(
            name='invertcaps',
            encode=InvertCapsCodec().encode,
            decode=InvertCapsCodec().decode,
            incrementalencoder=InvertCapsIncrementalEncoder,
            incrementaldecoder=InvertCapsIncrementalDecoder,
            streamreader=InvertCapsStreamReader,
            streamwriter=InvertCapsStreamWriter,
        )
    return None

codecs.register(find_invertcaps)

if __name__ == '__main__':

    # Stateless encoder/decoder
    encoder = codecs.getencoder('invertcaps')
    text = 'abcDEF'
    encoded_text, consumed = encoder(text)
    print('Encoded "{}" to "{}", consuming {} characters'.format(
        text, encoded_text, consumed))

    # Stream writer
    import io
    buffer = io.BytesIO()
    writer = codecs.getwriter('invertcaps')(buffer)
    print('StreamWriter for io buffer: ')
    print('  writing "abcDEF"')
    writer.write('abcDEF')
    print('  buffer contents: ', buffer.getvalue())

    # Incremental decoder
    decoder_factory = codecs.getincrementaldecoder('invertcaps')
    decoder = decoder_factory()
    decoded_text_parts = []
    for c in encoded_text:
        decoded_text_parts.append(
            decoder.decode(bytes([c]), final=False)
        )
    decoded_text_parts.append(decoder.decode(b'', final=True))
    decoded_text = ''.join(decoded_text_parts)
    print('IncrementalDecoder converted {!r} to {!r}'.format(
        encoded_text, decoded_text))
```

无状态编码器/解码器基类是Codec。使用新实现覆盖encode（）和decode（）（在本例中，分别调用`charmap_encode（）`和`charmap_decode（）`）。每个方法都必须返回一个元组，其中包含已转换的数据以及所消耗的输入字节数或字符数。方便的是，`charmap_encode（）`和`charmap_decode（）`已经返回了该信息。

IncrementalEncoder和IncrementalDecoder充当增量接口的基类。增量类的encode（）和decode（）方法以这样的方式定义，即它们只返回实际的转换数据。有关缓冲的任何信息都保持为内部状态。 invertcaps编码不需要缓冲数据（它使用一对一映射）。对于根据正在处理的数据产生不同输出量的编码，例如压缩算法，BufferedIncrementalEncoder和BufferedIncrementalDecoder是更合适的基类，因为它们管理输入的未处理部分。

StreamReader和StreamWriter也需要encode（）和decode（）方法，并且因为它们需要返回与Codec版本相同的值，所以可以使用多重继承来实现。

```
$ python3 codecs_invertcaps_register.py

Encoded "abcDEF" to "b'ABCdef'", consuming 6 characters
StreamWriter for io buffer:
  writing "abcDEF"
  buffer contents:  b'ABCdef'
IncrementalDecoder converted b'ABCdef' to 'abcDEF'
```













