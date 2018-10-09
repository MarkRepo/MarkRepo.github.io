---
title: PyMOTW-3 --- Modules and Packages
categories: language
tags: PyMOTW-3
---

Python的主要扩展机制使用保存到模块的源代码，并通过import语句合并到程序中。大多数开发人员认为的“Python”功能实际上是通过模块集合实现的，即标准库，这是本书的主题。虽然导入功能内置于解释器本身，但库中有几个与导入过程相关的模块。

importlib模块公开了解释器使用的导入机制的底层实现。它可用于在运行时动态导入模块，而不是在启动期间使用import语句加载它们。当事先不知道需要导入的模块的名称时，动态加载模块很有用，例如插件或应用程序的扩展。

Python包可以包括辅助资源文件，例如模板，默认配置文件，图像和其他数据以及源代码。以可移植方式访问资源文件的接口在pkgutil模块中实现。它还支持修改包的导入路径，以便内容可以安装到多个目录中，但显示为同一个包的一部分。

zipimport为保存到ZIP存档的模块和包提供自定义导入程序。例如，它用于加载Python EGG文件，也可以用作打包和分发应用程序的便捷方式。

## importlib — Python’s Import Mechanism

目的：importlib模块公开Python的import语句的实现。  
importlib模块包含实现Python导入机制的函数，用于加载在包和模块中的代码。它是动态导入模块的一个访问点，在编写代码时需要导入的模块名称未知的某些情况下很有用（例如，对于插件或应用程序的扩展）。

### Example Package

本节中的示例使用包含`__init__.py`的example包。

```python
#example/__init__.py
print('Importing example package')
```
这个包还包含`submodule.py`.

```python
#example/submodule.py
print('Importing submodule')
```
导入包或模块时，请注意示例输出中print（）调用的文本。

### Module Types

Python支持多种样式的模块。打开模块并将其添加到命名空间时，每个都需要自己的处理，并且格式的支持因平台而异。例如，在Microsoft Windows下，共享库是从扩展名为.dll或.pyd的文件加载而不是.so。当使用解释器的调试版本而不是正常版本构建时，C模块的扩展也可能会发生变化，因为它们编译时也可以包含调试信息。如果未按预期加载C扩展库或其他模块，请使用importlib.machinery中定义的常量来查找当前平台支持的类型以及用于加载它们的参数。

```python
#importlib_suffixes.py
import importlib.machinery

SUFFIXES = [
    ('Source:', importlib.machinery.SOURCE_SUFFIXES),
    ('Debug:', importlib.machinery.DEBUG_BYTECODE_SUFFIXES),
    ('Optimized:', importlib.machinery.OPTIMIZED_BYTECODE_SUFFIXES),
    ('Bytecode:', importlib.machinery.BYTECODE_SUFFIXES),
    ('Extension:', importlib.machinery.EXTENSION_SUFFIXES),
]

def main():
    tmpl = '{:<10}  {}'
    for name, value in SUFFIXES:
        print(tmpl.format(name, value))

if __name__ == '__main__':
    main()
```
返回值是元组序列，元祖包含文件扩展名、打开模块文件使用的模式，以及定义在模块中的类型码常量。此表不完整，因为某些可导入的模块或包类型与单个文件不对应。

```
$ python3 importlib_suffixes.py

Source:     ['.py']
Debug:      ['.pyc']
Optimized:  ['.pyc']
Bytecode:   ['.pyc']
Extension:  ['.cpython-36m-darwin.so', '.abi3.so', '.so']
```

### Importing Modules

importlib中的高级API使得在给定绝对或相对名称的情况下导入模块变得简单。使用相对模块名称时，请将包含模块的包指定为单独的参数。

```python
#importlib_import_module.py
import importlib

m1 = importlib.import_module('example.submodule')
print(m1)

m2 = importlib.import_module('.submodule', package='example')
print(m2)

print(m1 is m2)
```
import_module（）的返回值是导入创建的模块对象。

```
$ python3 importlib_import_module.py

Importing example package
Importing submodule
<module 'example.submodule' from '.../example/submodule.py'>
<module 'example.submodule' from '.../example/submodule.py'>
True
```
如果无法导入模块，import_module（）会引发ImportError。

```python
#importlib_import_module_error.py
import importlib

try:
    importlib.import_module('example.nosuchmodule')
except ImportError as err:
    print('Error:', err)
```
错误消息包含缺少的模块的名称。

```
$ python3 importlib_import_module_error.py

Importing example package
Error: No module named 'example.nosuchmodule'
```
要重新加载现有模块，请使用reload（）。

```python
#importlib_reload.py
import importlib

m1 = importlib.import_module('example.submodule')
print(m1)

m2 = importlib.reload(m1)
print(m1 is m2)
```
reload（）的返回值是新模块。根据使用的加载器类型，它可能是相同的模块实例。

```
$ python3 importlib_reload.py

Importing example package
Importing submodule
<module 'example.submodule' from '.../example/submodule.py'>
Importing submodule
True
```

### Loaders

importlib中的低级API提供对加载器对象的访问，如sys模块一节中的`Modules and Imports`中所述。要获取模块的加载器，请使用`find_loader（）`。然后检索该模块，请使用loader的`load_module（）`方法。

```python
#importlib_find_loader.py
import importlib

loader = importlib.find_loader('example')
print('Loader:', loader)

m = loader.load_module()
print('Module:', m)
```
此示例加载example包的顶级。

```
$ python3 importlib_find_loader.py

Loader: <_frozen_importlib_external.SourceFileLoader object at 0x101fe1828>
Importing example package
Module: <module 'example' from '.../example/__init__.py'>
```
包中的子模块需要使用包中的路径单独加载。在下面的示例中，首先加载包，然后将其路径传递给`find_loader（）`以创建能够加载子模块的加载器。

```python
#importlib_submodule.py
import importlib

pkg_loader = importlib.find_loader('example')
pkg = pkg_loader.load_module()

loader = importlib.find_loader('submodule', pkg.__path__)
print('Loader:', loader)

m = loader.load_module()
print('Module:', m)
```
与import_module（）不同，子模块的名称没有任何相对路径前缀，因为加载器已经受到包路径的约束。

```
$ python3 importlib_submodule.py

Importing example package
Loader: <_frozen_importlib_external.SourceFileLoader object at 0x101fe1f28>
Importing submodule
Module: <module 'submodule' from '.../example/submodule.py'>
```

## pkgutil — Package Utilities

目的：为特定包添加模块搜索路径，并使用包中包含的资源。  
pkgutil模块包括用于更改Python包的导入规则以及从包中分发的文件加载非代码资源的功能。

### Package Import Paths

extend_path（）函数用于修改搜索路径并更改从包中导入子模块的方式，以便可以组合多个不同的目录，就好像它们是一个一样。这可用于用开发版本的包覆盖已安装的版本，或将特定于平台的模块和共享模块组合到单个包命名空间中。  
调用extend_path（）的最常用方法是在包内的`__init__.py`中添加两行。

```python
import pkgutil
__path__ = pkgutil.extend_path(__path__, __name__)
```
extend_path（）扫描sys.path以查找包含“用于为第二个参数指定的包命名的”子目录的目录。目录列表与作为第一个参数传递的路径值组合在一起，并作为单个列表返回，适合用作包导入路径。  
名为demopkg的示例包包含两个文件`__init__.py`和`shared.py`。demopkg1中的`__init__.py`文件包含用于显示修改前后搜索路径的print语句，以突出显示差异。

```python
#demopkg1/__init__.py
import pkgutil
import pprint

print('demopkg1.__path__ before:')
pprint.pprint(__path__)
print()

__path__ = pkgutil.extend_path(__path__, __name__)

print('demopkg1.__path__ after:')
pprint.pprint(__path__)
print()
```
extension目录具有demopkg的附加功能，包含另外三个源文件。每个目录级别都有一个`__init__.py`，和一个`not_shared.py`。

```
$ find extension -name '*.py'

extension/__init__.py
extension/demopkg1/__init__.py
extension/demopkg1/not_shared.py
```
这个简单的测试程序导入demopkg1包。

```python
#pkgutil_extend_path.py
import demopkg1
print('demopkg1           :', demopkg1.__file__)

try:
    import demopkg1.shared
except Exception as err:
    print('demopkg1.shared    : Not found ({})'.format(err))
else:
    print('demopkg1.shared    :', demopkg1.shared.__file__)

try:
    import demopkg1.not_shared
except Exception as err:
    print('demopkg1.not_shared: Not found ({})'.format(err))
else:
    print('demopkg1.not_shared:', demopkg1.not_shared.__file__)
```
当直接从命令行运行此测试程序时，找不到`not_shared`模块。

>注意  
这些示例中的完整文件系统路径已缩短，以强调更改的部分。

```
$ python3 pkgutil_extend_path.py

demopkg1.__path__ before:
['.../demopkg1']

demopkg1.__path__ after:
['.../demopkg1']

demopkg1           : .../demopkg1/__init__.py
demopkg1.shared    : .../demopkg1/shared.py
demopkg1.not_shared: Not found (No module named 'demopkg1.not_shared')
```
但是，如果将extension目录添加到PYTHONPATH并再次运行程序，则会生成不同的结果。

```
$ PYTHONPATH=extension python3 pkgutil_extend_path.py

demopkg1.__path__ before:
['.../demopkg1']

demopkg1.__path__ after:
['.../demopkg1', '.../extension/demopkg1']

demopkg1           : .../demopkg1/__init__.py
demopkg1.shared    : .../demopkg1/shared.py
demopkg1.not_shared: .../extension/demopkg1/not_shared.py
```
extension目录中的demopkg1版本已添加到搜索路径中，因此可以在那里找到`not_shared`模块。  
以这种方式扩展路径对于将特定于平台的软件包版本与通用软件包相结合非常有用，尤其是在特定于平台的版本包含C扩展模块的情况下。

### Development Versions of Packages

在开发项目增强功能时，通常需要测试已安装软件包的更改。用开发版本替换已安装的副本可能是个坏主意，因为它不一定正确，系统上的其他工具可能依赖于已安装的软件包。  
可以使用virtualenv或venv在开发环境中配置完全独立的包副本，但是对于小的修改，设置具有所有依赖性的虚拟环境的开销可能过高。  
另一个选择是使用pkgutil修改“属于正在开发的包的”模块的模块搜索路径。但是，在这种情况下，必须反转路径，以便使开发版本覆盖已安装的版本。  
给定一个包含`__init__.py`和`overloaded.py`的包demopkg2，其中正在开发的函数位于`demopkg2/overloaded.py`中。已安装的版本包含

```python
#demopkg2/overloaded.py

def func():
    print('This is the installed version of func().')
```
并且`demopkg2/__init__.py`包含

```python
#demopkg2/__init__.py
import pkgutil

__path__ = pkgutil.extend_path(__path__, __name__)
__path__.reverse()
```
reverse（）用于确保在默认位置之前扫描由pkgutil添加到搜索路径的任何目录以进行导入。  
该程序导入demopkg2.overloaded并调用func（）。

```python
#pkgutil_devel.py
import demopkg2
print('demopkg2           :', demopkg2.__file__)

import demopkg2.overloaded
print('demopkg2.overloaded:', demopkg2.overloaded.__file__)

print()
demopkg2.overloaded.func()
```
在没有任何特殊路径处理的情况下运行它会从已安装的func（）版本生成输出。

```
$ python3 pkgutil_devel.py

demopkg2           : .../demopkg2/__init__.py
demopkg2.overloaded: .../demopkg2/overloaded.py

This is the installed version of func().
```
开发目录包含

```
$ find develop/demopkg2 -name '*.py'

develop/demopkg2/__init__.py
develop/demopkg2/overloaded.py
```
和overloaded的修改版本

```python
#develop/demopkg2/overloaded.py

def func():
    print('This is the development version of func().')
```
如果develop目录在搜索路径中，当测试程序运行时，它将被加载：

```
$ PYTHONPATH=develop python3 pkgutil_devel.py

demopkg2           : .../demopkg2/__init__.py
demopkg2.overloaded: .../develop/demopkg2/overloaded.py

This is the development version of func().
```

### Managing Paths with PKG Files

第一个示例说明了如何使用PYTHONPATH中包含的额外目录扩展搜索路径。也可以使用包含目录名的*.pkg文件添加到搜索路径。PKG文件类似于site模块使用的PTH文件。它们可以包含目录名，每行一个，以添加到包的搜索路径中。  
从第一个示例构建应用程序的特定于平台的部分的另一种方法是为每个操作系统使用单独的目录，并包括.pkg文件以扩展搜索路径。此示例使用相同的demopkg1文件，还包括以下文件。

```
$ find os_* -type f

os_one/demopkg1/__init__.py
os_one/demopkg1/not_shared.py
os_one/demopkg1.pkg
os_two/demopkg1/__init__.py
os_two/demopkg1/not_shared.py
os_two/demopkg1.pkg
```
PKG文件名为demopkg1.pkg以匹配要扩展的包。它们都包含一行。

```
demopkg
```
此演示程序显示正在导入的模块的版本。

```python
#pkgutil_os_specific.py
import demopkg1
print('demopkg1:', demopkg1.__file__)

import demopkg1.shared
print('demopkg1.shared:', demopkg1.shared.__file__)

import demopkg1.not_shared
print('demopkg1.not_shared:', demopkg1.not_shared.__file__)
```
可以使用简单的包装器脚本在两个包之间切换。

```shell
#with_os.sh

#!/bin/sh

export PYTHONPATH=os_${1}
echo "PYTHONPATH=$PYTHONPATH"
echo

python3 pkgutil_os_specific.py
```
以“one”或“two”作为参数运行时，将调整路径。

```
$ ./with_os.sh one

PYTHONPATH=os_one

demopkg1.__path__ before:
['.../demopkg1']

demopkg1.__path__ after:
['.../demopkg1', '.../os_one/demopkg1', 'demopkg']

demopkg1: .../demopkg1/__init__.py
demopkg1.shared: .../demopkg1/shared.py
demopkg1.not_shared: .../os_one/demopkg1/not_shared.py

$ ./with_os.sh two

PYTHONPATH=os_two

demopkg1.__path__ before:
['.../demopkg1']

demopkg1.__path__ after:
['.../demopkg1', '.../os_two/demopkg1', 'demopkg']

demopkg1: .../demopkg1/__init__.py
demopkg1.shared: .../demopkg1/shared.py
demopkg1.not_shared: .../os_two/demopkg1/not_shared.py
```
PKG文件可以出现在普通搜索路径中的任何位置，因此当前工作目录中的单个PKG文件也可用于包含开发树。

### Nested Packages

对于嵌套包，只需要修改顶级包的路径。例如，有这个目录结构

```
$ find nested -name '*.py'

nested/__init__.py
nested/second/__init__.py
nested/second/deep.py
nested/shallow.py
```
`nested/__init__.py` 包含

```python
#nested/__init__.py
import pkgutil

__path__ = pkgutil.extend_path(__path__, __name__)
__path__.reverse()
```
并且一个开发树如下所示

```
$ find develop/nested -name '*.py'

develop/nested/__init__.py
develop/nested/second/__init__.py
develop/nested/second/deep.py
develop/nested/shallow.py
```
shallow和deep模块都包含一个简单的函数，用于打印一条消息，指示它们是否来自已安装或开发版本。  
该测试程序会运行新的包。

```python
#pkgutil_nested.py
import nested

import nested.shallow
print('nested.shallow:', nested.shallow.__file__)
nested.shallow.func()

print()
import nested.second.deep
print('nested.second.deep:', nested.second.deep.__file__)
nested.second.deep.func()
```
在没有任何路径操作的情况下运行`pkgutil_nested.py`时，将使用两个模块的已安装版本。

```
$ python3 pkgutil_nested.py

nested.shallow: .../nested/shallow.py
This func() comes from the installed version of nested.shallow

nested.second.deep: .../nested/second/deep.py
This func() comes from the installed version of nested.second.deep
```
将开发目录添加到路径时，两个函数的开发版本将覆盖已安装的版本。

```
$ PYTHONPATH=develop python3 pkgutil_nested.py

nested.shallow: .../develop/nested/shallow.py
This func() comes from the development version of nested.shallow

nested.second.deep: .../develop/nested/second/deep.py
This func() comes from the development version of nested.second.deep
```

### Package Data

除了代码之外，Python包还可以包含数据文件，例如模板，默认配置文件，图像以及包中代码使用的其他辅助文件。get_data（）函数以格式无关的方式访问文件中的数据，因此无论软件包是作为EGG，冻结二进制文件的一部分还是文件系统上的常规文件分发都无关紧要。  
使用包含templates目录的包pkgwithdata

```
$ find pkgwithdata -type f

pkgwithdata/__init__.py
pkgwithdata/templates/base.html
```
文件`pkgwithdata/templates/base.html`包含一个简单的HTML模板。

```html
#pkgwithdata/templates/base.html

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML//EN">
<html> <head>
<title>PyMOTW Template</title>
</head>

<body>
<h1>Example Template</h1>

<p>This is a sample data file.</p>

</body>
</html>
```
该程序使用get_data（）来检索模板内容并将其打印出来。

```python
#pkgutil_get_data.py
import pkgutil

template = pkgutil.get_data('pkgwithdata', 'templates/base.html')
print(template.decode('utf-8'))
```
get_data（）的参数是包的虚线名称，以及相对于包顶部的文件名。返回值是一个字节序列，因此在打印之前从UTF-8解码。

```
$ python3 pkgutil_get_data.py

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML//EN">
<html> <head>
<title>PyMOTW Template</title>
</head>

<body>
<h1>Example Template</h1>

<p>This is a sample data file.</p>

</body>
</html>
```
get_data（）与分发格式无关，因为它使用PEP302中定义的导入钩子来访问包内容。可以使用任何提供钩子的加载器，包括zipfile中的ZIP存档导入器。

```python
#pkgutil_get_data_zip.py
import pkgutil
import zipfile
import sys

# Create a ZIP file with code from the current directory
# and the template using a name that does not appear on the
# local filesystem.
with zipfile.PyZipFile('pkgwithdatainzip.zip', mode='w') as zf:
    zf.writepy('.')
    zf.write('pkgwithdata/templates/base.html',
             'pkgwithdata/templates/fromzip.html',
             )

# Add the ZIP file to the import path.
sys.path.insert(0, 'pkgwithdatainzip.zip')

# Import pkgwithdata to show that it comes from the ZIP archive.
import pkgwithdata
print('Loading pkgwithdata from', pkgwithdata.__file__)

# Print the template body
print('\nTemplate:')
data = pkgutil.get_data('pkgwithdata', 'templates/fromzip.html')
print(data.decode('utf-8'))
```
此示例使用PyZipFile.writepy（）创建包含pkgwithdata包副本的ZIP存档，包括模板文件的重命名版本。然后，在使用pkgutil加载模板并打印之前，它会将ZIP存档添加到导入路径。有关使用writepy（）的更多详细信息，请参阅zipfile的讨论。

```
$ python3 pkgutil_get_data_zip.py

Loading pkgwithdata from
pkgwithdatainzip.zip/pkgwithdata/__init__.pyc

Template:
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML//EN">
<html> <head>
<title>PyMOTW Template</title>
</head>

<body>
<h1>Example Template</h1>

<p>This is a sample data file.</p>

</body>
</html>
```

## zipimport — Load Python Code from ZIP Archives

目的：导入保存为ZIP存档成员的Python模块。  
zipimport模块实现了zipimporter类，可用于在ZIP存档中查找和加载Python模块。zipimporter支持PEP302中指定的导入钩子API; 这是Python Eggs的工作方式。  
通常不需要直接使用zipimport模块，因为只要该存档出现在sys.path中，就可以直接从ZIP存档导入。但是，研究如何使用导入器API，了解可用的功能以及理解模块导入的工作原理是有益的。了解ZIP导入器的工作方式也有助于调试在“使用zipfile.PyZipFile创建ZIP存档的包以分发应用程序”时可能出现的问题。

### Example

这些示例重用了zipfile讨论中的一些代码来创建包含一些Python模块的示例ZIP存档。

```python
#zipimport_make_example.py
import sys
import zipfile

if __name__ == '__main__':
    zf = zipfile.PyZipFile('zipimport_example.zip', mode='w')
    try:
        zf.writepy('.')
        zf.write('zipimport_get_source.py')
        zf.write('example_package/README.txt')
    finally:
        zf.close()
    for name in zf.namelist():
        print(name)
```
在任何其他示例之前运行`zipimport_make_example.py`，以创建包含示例目录中所有模块以及本节中示例所需的一些测试数据的ZIP存档。

```
$ python3 zipimport_make_example.py

__init__.pyc
example_package/__init__.pyc
zipimport_find_module.pyc
zipimport_get_code.pyc
zipimport_get_data.pyc
zipimport_get_data_nozip.pyc
zipimport_get_data_zip.pyc
zipimport_get_source.pyc
zipimport_is_package.pyc
zipimport_load_module.pyc
zipimport_make_example.pyc
zipimport_get_source.py
example_package/README.txt
```

### Finding a Module

给定模块的全名，find_module（）将尝试在ZIP存档中找到该模块。

```python
#zipimport_find_module.py
import zipimport

importer = zipimport.zipimporter('zipimport_example.zip')

for module_name in ['zipimport_find_module', 'not_there']:
    print(module_name, ':', importer.find_module(module_name))
```
如果找到该模块，则返回zipimporter实例。 否则，返回None。

```
$ python3 zipimport_find_module.py

zipimport_find_module : <zipimporter object "zipimport_example.zip">
not_there : None
```

### Accessing Code

get_code（）方法从归档中加载模块的代码对象。

```python
#zipimport_get_code.py
import zipimport

importer = zipimport.zipimporter('zipimport_example.zip')
code = importer.get_code('zipimport_get_code')
print(code)
```
代码对象与模块对象不同，但代码对象用于创建一个模块对象。

```
$ python3 zipimport_get_code.py

<code object <module> at 0x1012b4ae0, file "./zipimport_get_code.py", line 6>
```
要将代码加载为可用模块，请改用load_module（）。

```python
#zipimport_load_module.py
import zipimport

importer = zipimport.zipimporter('zipimport_example.zip')
module = importer.load_module('zipimport_get_code')
print('Name   :', module.__name__)
print('Loader :', module.__loader__)
print('Code   :', module.code)
```
结果是模块对象配置为好像代码已从常规导入加载。

```
$ python3 zipimport_load_module.py

<code object <module> at 0x1007b4c00, file "./zipimport_get_code.py", line 6>
Name   : zipimport_get_code
Loader : <zipimporter object "zipimport_example.zip">
Code   : <code object <module> at 0x1007b4c00, file "./zipimport_get_code.py", line 6>
```

### Source

与inspect模块一样，如果存档包含源，则可以从ZIP存档中检索模块的源代码。在示例中，只有`zipimport_get_source.py`被添加到zipimport_example.zip（其余模块只是作为.pyc文件添加）。

```python
#zipimport_get_source.py
import zipimport

modules = [
    'zipimport_get_code',
    'zipimport_get_source',
]

importer = zipimport.zipimporter('zipimport_example.zip')
for module_name in modules:
    source = importer.get_source(module_name)
    print('=' * 80)
    print(module_name)
    print('=' * 80)
    print(source)
    print()
```
如果模块的源不可用，则get_source（）返回None。

```
$ python3 zipimport_get_source.py

================================================================
zipimport_get_code
================================================================
None

================================================================
zipimport_get_source
================================================================
#!/usr/bin/env python3
#
# Copyright 2007 Doug Hellmann.
#
"""Retrieving the source code for a module within a zip archive.
"""

#end_pymotw_header
import zipimport

modules = [
    'zipimport_get_code',
    'zipimport_get_source',
]

importer = zipimport.zipimporter('zipimport_example.zip')
for module_name in modules:
    source = importer.get_source(module_name)
    print('=' * 80)
    print(module_name)
    print('=' * 80)
    print(source)
    print()
```

### Packages

要确定一个名称是否引用包而不是常规模块，请使用is_package（）。

```python
#zipimport_is_package.py
import zipimport

importer = zipimport.zipimporter('zipimport_example.zip')
for name in ['zipimport_is_package', 'example_package']:
    print(name, importer.is_package(name))
```
在此例中，`zipimport_is_package`来自一个模块，example_package是一个包。

```
$ python3 zipimport_is_package.py

zipimport_is_package False
example_package True
```

### Data

有时源码模块或包需要和非代码数据一起分发。图像，配置文件，默认数据和测试夹具只是其中的几个例子。通常，模块的`__path__`或`__file__`属性用于查找这些数据文件相对于代码安装位置的路径。  
例如，对于“普通”模块，可以从导入的包的`__file__`属性构造文件系统路径，如下所示：

```python
#zipimport_get_data_nozip.py
import os
import example_package

# Find the directory containing the imported
# package and build the data filename from it.
pkg_dir = os.path.dirname(example_package.__file__)
data_filename = os.path.join(pkg_dir, 'README.txt')

# Read the file and show its contents.
print(data_filename, ':')
print(open(data_filename, 'r').read())
```
输出将取决于示例代码在文件系统上的位置。

```
$ python3 zipimport_get_data_nozip.py

.../example_package/README.txt :
This file represents sample data which could be embedded in the
ZIP archive.  You could include a configuration file, images, or
any other sort of noncode data.
```
如果从ZIP存档而不是文件系统导入example_package，则使用`__file__`不起作用。

```python
#zipimport_get_data_zip.py
import sys
sys.path.insert(0, 'zipimport_example.zip')

import os
import example_package
print(example_package.__file__)
data_filename = os.path.join(os.path.dirname(example_package.__file__), 'README.txt', )
print(data_filename, ':')
print(open(data_filename, 'rt').read())
```
包的`__file__`引用ZIP存档，并且不是一个目录，因此构建README.txt文件的路径会给出错误的值。

```
$ python3 zipimport_get_data_zip.py

zipimport_example.zip/example_package/__init__.pyc
zipimport_example.zip/example_package/README.txt :
Traceback (most recent call last):
  File "zipimport_get_data_zip.py", line 20, in <module>
    print(open(data_filename, 'rt').read())
NotADirectoryError: [Errno 20] Not a directory:
'zipimport_example.zip/example_package/README.txt'
```
检索文件的更可靠方法是使用get_data（）方法。可以通过被导入模块的`__loader__`属性访问加载模块的zipimporter实例：

```python
#zipimport_get_data.py
import sys
sys.path.insert(0, 'zipimport_example.zip')

import os
import example_package
print(example_package.__file__)
data = example_package.__loader__.get_data('example_package/README.txt')
print(data.decode('utf-8'))
```
pkgutil.get_data（）使用此接口访问包中的数据。返回的值是一个字节字符串，需要在打印前解码为unicode字符串。

```
$ python3 zipimport_get_data.py

zipimport_example.zip/example_package/__init__.pyc
This file represents sample data which could be embedded in the
ZIP archive.  You could include a configuration file, images, or
any other sort of noncode data.
```
不是通过zipimport导入的模块没有设置`__loader__`。
