---
title: PyMOTW-3 --- Internationalization and Localization
categories: language
tags: PyMOTW-3
---

Python附带两个模块，用于准备让应用程序使用多种自然语言和文化设置。gettext用于创建不同语言的message catalog，以便可以用用户可以理解的语言显示提示和错误消息。locale更改数字、货币、日期和时间的格式，以考虑文化差异，例如如何显示负值以及本地货币符号是什么。 两个模块都与其他工具和操作环境相连接，以使Python应用程序适合系统上的所有其他程序。

## gettext — Message Catalogs

目的：用于国际化的message catalog API。  
gettext模块提供了与GNU gettext库兼容的纯Python实现，用于message翻译和catalog管理。 Python源代码分发提供的工具使您能够从一组源文件中提取消息，构建包含翻译的message catalog，并使用该message catalog在运行时为用户显示适当的消息。
message catalog可用于为程序提供国际化接口，以适合用户的语言显示消息。它们还可以用于其他消息自定义，包括为不同的包装器或搭档（partners）“蒙上（skinning）”一层接口。

>注意  
尽管标准库文档说Python中包含了所有必需的工具，但pygettext.py无法提取用ngettext调用包装的消息，即使使用适当的命令行选项也是如此。这些示例使用GNU gettext工具集中的xgettext代替。

### Translation Workflow Overview

设置和使用翻译的过程包括五个步骤。

1. 识别并标记源码中包含要翻译的消息的文字字符串。  
首先识别程序源码中需要翻译的消息，然后标记文字字符串，以便提取程序可以找到它们。
2. 提取消息。  
识别源码中的可翻译字符串后，使用xgettext提取它们并创建`.pot`文件或翻译模板。该模板是一个文本文件，其中包含所有已识别字符串的副本以及用于其翻译的占位符。
3. 翻译消息。  
将`.pot`文件的副本提供给翻译人员，并将扩展名更改为`.po`。`.po`文件是一个可编辑的源文件，用作编译步骤的输入。翻译人员应更新文件中的标题文本，并提供所有字符串的翻译。
4. 从翻译中“编译” message catalog。  
当翻译人员发回完成的`.po`文件时，使用msgfmt将文本文件编译为二进制目录格式。二进制格式由运行时catalog查找代码使用。
5. 在运行时加载并激活相应的message catalog。  
最后一步是向应用程序添加几行以配置和加载message catalog并安装翻译功能。有几种方法可以做到这一点，以进行相关的权衡。

本节的其余部分将更详细地检查这些步骤，从所需的代码修改开始。

### Creating Message Catalogs from Source Code

gettext的工作原理是在翻译数据库中查找文字字符串，并提取相应的翻译字符串。 通常的模式是将适当的查找函数绑定到名称“_”（单个下划线字符），这样代码就不会因为对名称较长的函数的大量调用而混乱。  
消息提取程序xgettext查找嵌入在目录查找函数调用中的消息。 它理解不同的源码语言，并为每个语言使用适当的解析器。 如果查找函数具有别名或添加了额外函数，请为xgettext提供“在提取消息时要考虑的”附加符号的名称。

此脚本有一条准备好要翻译的消息。

```python
#gettext_example.py
import gettext

# Set up message catalog access
t = gettext.translation(
    'example_domain', 'locale',
    fallback=True,
)
_ = t.gettext

print(_('This message is in the script.'))
```
文本“This message is in the script.”是要从catalog中替换的消息。 启用了fallback模式，因此如果在没有message catalog的情况下运行脚本，则会打印源码行中的消息。

```
$ python3 gettext_example.py

This message is in the script.
```
下一步是使用pygettext.py或xgettext提取消息并创建`.pot`文件。

```
$ xgettext -o example.pot gettext_example.py
```
生成的输出文件包含以下内容。

```python
#example.pot
# SOME DESCRIPTIVE TITLE.
# Copyright (C) YEAR THE PACKAGE'S COPYRIGHT HOLDER
# This file is distributed under the same license as the PACKAGE package.
# FIRST AUTHOR <EMAIL@ADDRESS>, YEAR.
#
#, fuzzy
msgid ""
msgstr ""
"Project-Id-Version: PACKAGE VERSION\n"
"Report-Msgid-Bugs-To: \n"
"POT-Creation-Date: 2018-03-18 16:20-0400\n"
"PO-Revision-Date: YEAR-MO-DA HO:MI+ZONE\n"
"Last-Translator: FULL NAME <EMAIL@ADDRESS>\n"
"Language-Team: LANGUAGE <LL@li.org>\n"
"Language: \n"
"MIME-Version: 1.0\n"
"Content-Type: text/plain; charset=CHARSET\n"
"Content-Transfer-Encoding: 8bit\n"

#: gettext_example.py:19
msgid "This message is in the script."
msgstr ""
```
message catalog安装在按domain和language组织的目录中。域由应用程序或库提供，通常是像应用程序名称一样的唯一值。在这个例子中，`gettext_example.py`中的域是`example_domain`。语言值由用户的环境在运行时通过其中一个环境变量`LANGUAGE`，`LC_ALL`，`LC_MESSAGES`或`LANG`提供，具体取决于其配置和平台。这些示例都使用设置为`en_US`的语言运行。

现在模板已准备就绪，下一步是创建所需的目录结构并将模板复制到正确的位置。 PyMOTW源代码树内的locale目录将作为这些示例的message catalog目录(message catalog directory)的根目录，但通常最好使用系统范围内可访问的目录，以便所有用户都可以访问message catalog。目录输入源(catalog input source)的完整路径是`$localedir/$language/LC_MESSAGES/$domain.po`，实际catalog的文件扩展名为`.mo`。

通过将`example.pot`复制到`locale/en_US/LC_MESSAGES/example.po`并对其进行编辑以更改标头中的值并设置替换消息来创建catalog。结果如下所示。

```python
#locale/en_US/LC_MESSAGES/example.po
# Messages from gettext_example.py.
# Copyright (C) 2009 Doug Hellmann
# Doug Hellmann <doug@doughellmann.com>, 2016.
#
msgid ""
msgstr ""
"Project-Id-Version: PyMOTW-3\n"
"Report-Msgid-Bugs-To: Doug Hellmann <doug@doughellmann.com>\n"
"POT-Creation-Date: 2016-01-24 13:04-0500\n"
"PO-Revision-Date: 2016-01-24 13:04-0500\n"
"Last-Translator: Doug Hellmann <doug@doughellmann.com>\n"
"Language-Team: US English <doug@doughellmann.com>\n"
"MIME-Version: 1.0\n"
"Content-Type: text/plain; charset=UTF-8\n"
"Content-Transfer-Encoding: 8bit\n"


#: gettext_example.py:16
msgid "This message is in the script."
msgstr "This message is in the en_US catalog."
```
catalog是使用msgformat从`.po`文件构建的。

```
$ cd locale/en_US/LC_MESSAGES; msgfmt -o example.mo example.po
```
`gettext_example.py`中的域是`example_domain`，但该文件名为`example.pot`。 要让gettext找到正确的翻译文件，名称需要匹配。

```python
#gettext_example_corrected.py
t = gettext.translation(
    'example', 'locale',
    fallback=True,
)
```
现在，当脚本运行时，将打印catalog中的消息而不是源码中的字符串。

```
$ python3 gettext_example_corrected.py

This message is in the en_US catalog.
```

### Finding Message Catalogs at Runtime

如前所述，包含message catalog的locale目录是根据“用程序的域命名的”catalog使用的语言进行组织的。 不同的操作系统定义了自己的默认值，但gettext并不知道所有这些默认值。 它使用`sys.prefix + '/share/locale'`形式的默认locale目录，但大多数情况下始终显式提供localedir值比依赖“此默认值有效”更安全。 find（）函数负责在运行时定位恰当的message catalog。

```python
#gettext_find.py
import gettext

catalogs = gettext.find('example', 'locale', all=True)
print('Catalogs:', catalogs)
```
路径的语言部分取自可用于配置本地化功能的几个环境变量之一（LANGUAGE，`LC_ALL`，`LC_MESSAGES`和LANG）。 使用找到的第一个变量。 可以通过用冒号（:)分隔值来选择多种语言。 要查看其工作原理，请使用第二个message catalog运行一些实验。

```shell
$ cd locale/en_CA/LC_MESSAGES; msgfmt -o example.mo example.po
$ cd ../../..
$ python3 gettext_find.py

Catalogs: ['locale/en_US/LC_MESSAGES/example.mo']

$ LANGUAGE=en_CA python3 gettext_find.py

Catalogs: ['locale/en_CA/LC_MESSAGES/example.mo']

$ LANGUAGE=en_CA:en_US python3 gettext_find.py

Catalogs: ['locale/en_CA/LC_MESSAGES/example.mo', 'locale/en_US/LC_MESSAGES/example.mo']

$ LANGUAGE=en_US:en_CA python3 gettext_find.py

Catalogs: ['locale/en_US/LC_MESSAGES/example.mo', 'locale/en_CA/LC_MESSAGES/example.mo']
```
尽管find（）显示了完整的catalog列表，但实际上只会加载序列中的第一个以进行消息查找。

```
$ python3 gettext_example_corrected.py

This message is in the en_US catalog.

$ LANGUAGE=en_CA python3 gettext_example_corrected.py

This message is in the en_CA catalog.

$ LANGUAGE=en_CA:en_US python3 gettext_example_corrected.py

This message is in the en_CA catalog.

$ LANGUAGE=en_US:en_CA python3 gettext_example_corrected.py

This message is in the en_US catalog.
```

### Plural Values 复数值

虽然简单的消息替换可以处理大多数翻译需求，但gettext将复数视为一种特殊情况。根据语言，消息的单数形式和复数形式之间的差异可能仅仅由单个单词的结尾变化，或者整个句子结构也可能不同。 根据复数的级别，也可以有不同的形式。为了使复数管理更容易（或possible，在某些情况下），有一组单独的函数用于请求复数形式的消息。

```python
#gettext_plural.py
from gettext import translation
import sys

t = translation('plural', 'locale', fallback=False)
num = int(sys.argv[1])
msg = t.ngettext('{num} means singular.',
                 '{num} means plural.',
                 num)

# Still need to add the values to the message ourself.
print(msg.format(num=num))
```
使用ngettext（）访问消息的复数替换。 参数是要翻译的消息和项目数量。

```
$ xgettext -L Python -o plural.pot gettext_plural.py
```
由于存在要翻译的替换表单，因此代替者列在数组中。 使用数组允许翻译具有多个复数形式的语言（例如，波兰语具有不同的形式用来指示相应的数量）。

```python
#plural.pot
# SOME DESCRIPTIVE TITLE.
# Copyright (C) YEAR THE PACKAGE'S COPYRIGHT HOLDER
# This file is distributed under the same license as the PACKAGE package.
# FIRST AUTHOR <EMAIL@ADDRESS>, YEAR.
#
#, fuzzy
msgid ""
msgstr ""
"Project-Id-Version: PACKAGE VERSION\n"
"Report-Msgid-Bugs-To: \n"
"POT-Creation-Date: 2018-03-18 16:20-0400\n"
"PO-Revision-Date: YEAR-MO-DA HO:MI+ZONE\n"
"Last-Translator: FULL NAME <EMAIL@ADDRESS>\n"
"Language-Team: LANGUAGE <LL@li.org>\n"
"Language: \n"
"MIME-Version: 1.0\n"
"Content-Type: text/plain; charset=CHARSET\n"
"Content-Transfer-Encoding: 8bit\n"
"Plural-Forms: nplurals=INTEGER; plural=EXPRESSION;\n"

#: gettext_plural.py:15
#, python-brace-format
msgid "{num} means singular."
msgid_plural "{num} means plural."
msgstr[0] ""
msgstr[1] ""
```
除了填充翻译字符串之外，还需要告知库复数形成的方式，以便于它知道如何为任何给定的计数值索引数组。 行`“Plural-Forms：nplurals = INTEGER; plural = EXPRESSION; \ n”`包含两个要手动替换的值。 nplurals是一个整数，表示数组的大小（使用的翻译数），plural是一个C语言表达式，用于在查找翻译时将传入的数量转换为数组中的索引。 文字字符串n将替换为传递给ungettext（）的数量。

例如，英语包括两种复数形式。 数量为0被视为复数（“0 bananas”）。 Plural-Forms条目是：

```
Plural-Forms: nplurals=2; plural=n != 1;
```
然后，单数转换将进入位置0，并且复数转换将进入位置1。

```python
#locale/en_US/LC_MESSAGES/plural.po
# Messages from gettext_plural.py
# Copyright (C) 2009 Doug Hellmann
# This file is distributed under the same license 
# as the PyMOTW package.
# Doug Hellmann <doug@doughellmann.com>, 2016.
#
#, fuzzy
msgid ""
msgstr ""
"Project-Id-Version: PyMOTW-3\n"
"Report-Msgid-Bugs-To: Doug Hellmann <doug@doughellmann.com>\n"
"POT-Creation-Date: 2016-01-24 13:04-0500\n"
"PO-Revision-Date: 2016-01-24 13:04-0500\n"
"Last-Translator: Doug Hellmann <doug@doughellmann.com>\n"
"Language-Team: en_US <doug@doughellmann.com>\n"
"MIME-Version: 1.0\n"
"Content-Type: text/plain; charset=UTF-8\n"
"Content-Transfer-Encoding: 8bit\n"
"Plural-Forms: nplurals=2; plural=n != 1;"

#: gettext_plural.py:15
#, python-format
msgid "{num} means singular."
msgid_plural "{num} means plural."
msgstr[0] "In en_US, {num} is singular."
msgstr[1] "In en_US, {num} is plural."
```
编译目录后运行测试脚本几次将演示如何将不同的N值转换为翻译字符串的索引。

```
$ cd locale/en_US/LC_MESSAGES/; msgfmt -o plural.mo plural.po
$ cd ../../..
$ python3 gettext_plural.py 0

In en_US, 0 is plural.

$ python3 gettext_plural.py 1

In en_US, 1 is singular.

$ python3 gettext_plural.py 2

In en_US, 2 is plural.
```

### Application vs. Module Localization

定义gettext如何安装和与代码体一起使用属于翻译工作的范畴。

#### Application Localization

对于应用程序范围的翻译，作者可以使用`__builtins__`命名空间全局安装像ngettext（）这样的函数，因为它们可以控制应用程序代码的顶层。

```python
#gettext_app_builtin.py
import gettext

gettext.install(
    'example',
    'locale',
    names=['ngettext'],
)

print(_('This message is in the script.'))
```
`install（）`函数将`gettext（）`绑定到`__builtins__`名称空间中的名称`_（）`。 它还添加了ngettext（）和names中列出的其他函数。

#### Module Localization

对于库或单个模块，修改`__builtins__`不是一个好主意，因为它可能会引入与应用程序全局值的冲突。 更好的方法是，在模块顶部手动导入或重新绑定翻译函数的名称。

```python
#gettext_module_global.py
import gettext

t = gettext.translation(
    'example',
    'locale',
    fallback=False,
)
_ = t.gettext
ngettext = t.ngettext

print(_('This message is in the script.'))
```

### Switching Translations

前面的示例都在程序的持续时间内使用单个翻译。 某些情况，尤其是Web应用程序，需要在不同时间使用不同的message catalog，而无需退出和重置环境。 对于这些情况，gettext中提供的基于类的API将更加方便。 API调用与本节中描述的全局调用基本相同，但message catalog对象是公开的，可以直接操作，因此可以使用多个catalog。

## locale — Cultural Localization API

目的：格式化和解析依赖于地域（location）或语言的值。  
locale模块是Python国际化和本地化支持库的一部分。它提供了一种处理可能取决于用户语言或区域的操作的标准方法。例如，它处理将数字格式化为货币，比较字符串以进行排序和处理日期。它不包括翻译（请参阅gettext模块）或Unicode编码（请参阅codecs模块）。

>注意  
更改区域设置可能会产生应用程序范围的分歧，因此建议的做法是避免更改库中的值并让应用程序只设置一次。在本节的示例中，区域设置在短程序中多次更改，以突出显示各种区域设置的差异。应用程序更有可能在启动时或收到Web请求时设置区域一次，并且不更改它。

本节介绍了locale模块中的一些高级功能。还有一些是较低级别（format_string（））或与管理应用程序的区域设置（resetlocale（））相关的。

### Probing the Current Locale

让用户更改应用程序的区域设置的最常用方法是通过环境变量（`LC_ALL`，`LC_CTYPE`，LANG或LANGUAGE，具体取决于平台）。 然后，应用程序调用setlocale（）而不使用硬编码值，而是使用环境变量。

```python
#locale_env.py
import locale
import os
import pprint

# Default settings based on the user's environment.
locale.setlocale(locale.LC_ALL, '')

print('Environment settings:')
for env_name in ['LC_ALL', 'LC_CTYPE', 'LANG', 'LANGUAGE']:
    print('  {} = {}'.format(
        env_name, os.environ.get(env_name, ''))
    )

# What is the locale?
print('\nLocale from environment:', locale.getlocale())

template = """
Numeric formatting:

  Decimal point      : "{decimal_point}"
  Grouping positions : {grouping}
  Thousands separator: "{thousands_sep}"

Monetary formatting:

  International currency symbol   : "{int_curr_symbol!r}"
  Local currency symbol           : {currency_symbol!r}
  Symbol precedes positive value  : {p_cs_precedes}
  Symbol precedes negative value  : {n_cs_precedes}
  Decimal point                   : "{mon_decimal_point}"
  Digits in fractional values     : {frac_digits}
  Digits in fractional values,
                   international  : {int_frac_digits}
  Grouping positions              : {mon_grouping}
  Thousands separator             : "{mon_thousands_sep}"
  Positive sign                   : "{positive_sign}"
  Positive sign position          : {p_sign_posn}
  Negative sign                   : "{negative_sign}"
  Negative sign position          : {n_sign_posn}

"""

sign_positions = {
    0: 'Surrounded by parentheses',
    1: 'Before value and symbol',
    2: 'After value and symbol',
    3: 'Before value',
    4: 'After value',
    locale.CHAR_MAX: 'Unspecified',
}

info = {}
info.update(locale.localeconv())
info['p_sign_posn'] = sign_positions[info['p_sign_posn']]
info['n_sign_posn'] = sign_positions[info['n_sign_posn']]

print(template.format(**info))
```
localeconv（）方法返回包含locale的约定的字典。 标准库文档中介绍了值名称和定义的完整列表。  
运行OS X 10.11.6且未设置所有变量的Mac会生成此输出：

```
$ export LANG=; export LC_CTYPE=; python3 locale_env.py

Environment settings:
  LC_ALL =
  LC_CTYPE =
  LANG =
  LANGUAGE =

Locale from environment: (None, None)

Numeric formatting:

  Decimal point      : "."
  Grouping positions : []
  Thousands separator: ""

Monetary formatting:

  International currency symbol   : "''"
  Local currency symbol           : ''
  Symbol precedes positive value  : 127
  Symbol precedes negative value  : 127
  Decimal point                   : ""
  Digits in fractional values     : 127
  Digits in fractional values,
                   international  : 127
  Grouping positions              : []
  Thousands separator             : ""
  Positive sign                   : ""
  Positive sign position          : Unspecified
  Negative sign                   : ""
  Negative sign position          : Unspecified
```
设置LANG变量运行相同的脚本会显示locale和默认编码的更改方式。

The United States（en_US）：

```
$ LANG=en_US LC_CTYPE=en_US LC_ALL=en_US python3 locale_env.py

Environment settings:
  LC_ALL = en_US
  LC_CTYPE = en_US
  LANG = en_US
  LANGUAGE =

Locale from environment: ('en_US', 'ISO8859-1')

Numeric formatting:

  Decimal point      : "."
  Grouping positions : [3, 3, 0]
  Thousands separator: ","

Monetary formatting:

  International currency symbol   : "'USD '"
  Local currency symbol           : '$'
  Symbol precedes positive value  : 1
  Symbol precedes negative value  : 1
  Decimal point                   : "."
  Digits in fractional values     : 2
  Digits in fractional values,
                   international  : 2
  Grouping positions              : [3, 3, 0]
  Thousands separator             : ","
  Positive sign                   : ""
  Positive sign position          : Before value and symbol
  Negative sign                   : "-"
  Negative sign position          : Before value and symbol
```
France (fr_FR):

```
$ LANG=fr_FR LC_CTYPE=fr_FR LC_ALL=fr_FR python3 locale_env.py

Environment settings:
  LC_ALL = fr_FR
  LC_CTYPE = fr_FR
  LANG = fr_FR
  LANGUAGE =

Locale from environment: ('fr_FR', 'ISO8859-1')

Numeric formatting:

  Decimal point      : ","
  Grouping positions : [127]
  Thousands separator: ""

Monetary formatting:

  International currency symbol   : "'EUR '"
  Local currency symbol           : 'Eu'
  Symbol precedes positive value  : 0
  Symbol precedes negative value  : 0
  Decimal point                   : ","
  Digits in fractional values     : 2
  Digits in fractional values,
                   international  : 2
  Grouping positions              : [3, 3, 0]
  Thousands separator             : " "
  Positive sign                   : ""
  Positive sign position          : Before value and symbol
  Negative sign                   : "-"
  Negative sign position          : After value and symbol
```
Spain (es_ES):

```
$ LANG=es_ES LC_CTYPE=es_ES LC_ALL=es_ES python3 locale_env.py

Environment settings:
  LC_ALL = es_ES
  LC_CTYPE = es_ES
  LANG = es_ES
  LANGUAGE =

Locale from environment: ('es_ES', 'ISO8859-1')

Numeric formatting:

  Decimal point      : ","
  Grouping positions : [127]
  Thousands separator: ""

Monetary formatting:

  International currency symbol   : "'EUR '"
  Local currency symbol           : 'Eu'
  Symbol precedes positive value  : 0
  Symbol precedes negative value  : 0
  Decimal point                   : ","
  Digits in fractional values     : 2
  Digits in fractional values,
                   international  : 2
  Grouping positions              : [3, 3, 0]
  Thousands separator             : "."
  Positive sign                   : ""
  Positive sign position          : Before value and symbol
  Negative sign                   : "-"
  Negative sign position          : Before value and symbol
```
Portugal (pt_PT):

```
$ LANG=pt_PT LC_CTYPE=pt_PT LC_ALL=pt_PT python3 locale_env.py

Environment settings:
  LC_ALL = pt_PT
  LC_CTYPE = pt_PT
  LANG = pt_PT
  LANGUAGE =

Locale from environment: ('pt_PT', 'ISO8859-1')

Numeric formatting:

  Decimal point      : ","
  Grouping positions : []
  Thousands separator: " "

Monetary formatting:

  International currency symbol   : "'EUR '"
  Local currency symbol           : 'Eu'
  Symbol precedes positive value  : 0
  Symbol precedes negative value  : 0
  Decimal point                   : "."
  Digits in fractional values     : 2
  Digits in fractional values,
                   international  : 2
  Grouping positions              : [3, 3, 0]
  Thousands separator             : "."
  Positive sign                   : ""
  Positive sign position          : Before value and symbol
  Negative sign                   : "-"
  Negative sign position          : Before value and symbol
```
Poland (pl_PL):

```
$ LANG=pl_PL LC_CTYPE=pl_PL LC_ALL=pl_PL python3 locale_env.py

Environment settings:
  LC_ALL = pl_PL
  LC_CTYPE = pl_PL
  LANG = pl_PL
  LANGUAGE =

Locale from environment: ('pl_PL', 'ISO8859-2')

Numeric formatting:

  Decimal point      : ","
  Grouping positions : [3, 3, 0]
  Thousands separator: " "

Monetary formatting:

  International currency symbol   : "'PLN '"
  Local currency symbol           : 'zł'
  Symbol precedes positive value  : 1
  Symbol precedes negative value  : 1
  Decimal point                   : ","
  Digits in fractional values     : 2
  Digits in fractional values,
                   international  : 2
  Grouping positions              : [3, 3, 0]
  Thousands separator             : " "
  Positive sign                   : ""
  Positive sign position          : After value
  Negative sign                   : "-"
  Negative sign position          : After value
```

### Currency

前面的示例输出显示更改区域设置会更新货币符号设置，而字符会将整数与小数分开。 此示例循环遍历多个不同的区域设置，以打印为每个区域设置格式化的正负货币值。

```python
#locale_currency.py
import locale

sample_locales = [
    ('USA', 'en_US'),
    ('France', 'fr_FR'),
    ('Spain', 'es_ES'),
    ('Portugal', 'pt_PT'),
    ('Poland', 'pl_PL'),
]

for name, loc in sample_locales:
    locale.setlocale(locale.LC_ALL, loc)
    print('{:>10}: {:>10}  {:>10}'.format(
        name,
        locale.currency(1234.56),
        locale.currency(-1234.56),
    ))
```
输出是这个小表：

```
$ python3 locale_currency.py

       USA:   $1234.56   -$1234.56
    France: 1234,56 Eu  1234,56 Eu-
     Spain: 1234,56 Eu  -1234,56 Eu
  Portugal: 1234.56 Eu  -1234.56 Eu
    Poland: zł 1234,56  zł 1234,56-
```

### Formatting Numbers

与货币无关的数字的格式也不同，具体取决于区域设置。 特别是，用于将大数字分成可读块的分组（grouping）字符会发生变化。

```python
#locale_grouping.py
import locale

sample_locales = [
    ('USA', 'en_US'),
    ('France', 'fr_FR'),
    ('Spain', 'es_ES'),
    ('Portugal', 'pt_PT'),
    ('Poland', 'pl_PL'),
]

print('{:>10} {:>10} {:>15}'.format(
    'Locale', 'Integer', 'Float')
)
for name, loc in sample_locales:
    locale.setlocale(locale.LC_ALL, loc)

    print('{:>10}'.format(name), end=' ')
    print(locale.format('%10d', 123456, grouping=True), end=' ')
    print(locale.format('%15.2f', 123456.78, grouping=True))
```
要格式化没有货币符号的数字，请使用format（）而不是currency（）。

```
$ python3 locale_grouping.py

    Locale    Integer           Float
       USA    123,456      123,456.78
    France     123456       123456,78
     Spain     123456       123456,78
  Portugal     123456       123456,78
    Poland    123 456      123 456,78
```
要将区域设置格式的数字转换为标准化的区域设置无关格式，请使用delocalize（）。

```python
#locale_delocalize.py
import locale

sample_locales = [
    ('USA', 'en_US'),
    ('France', 'fr_FR'),
    ('Spain', 'es_ES'),
    ('Portugal', 'pt_PT'),
    ('Poland', 'pl_PL'),
]

for name, loc in sample_locales:
    locale.setlocale(locale.LC_ALL, loc)
    localized = locale.format('%0.2f', 123456.78, grouping=True)
    delocalized = locale.delocalize(localized)
    print('{:>10}: {:>10}  {:>10}'.format(
        name,
        localized,
        delocalized,
    ))
```
删除分组标点符号并将小数分隔符转换为`.`。

```
$ python3 locale_delocalize.py

       USA: 123,456.78   123456.78
    France:  123456,78   123456.78
     Spain:  123456,78   123456.78
  Portugal:  123456,78   123456.78
    Poland: 123 456,78   123456.78
```

### Parsing Numbers

除了以不同格式生成输出外，locale模块还有助于解析输入。 它包括atoi（）和atof（）函数，用于基于locale的数字格式约定将字符串转换为整数和浮点值。

```python
#locale_atof.py
import locale

sample_data = [
    ('USA', 'en_US', '1,234.56'),
    ('France', 'fr_FR', '1234,56'),
    ('Spain', 'es_ES', '1234,56'),
    ('Portugal', 'pt_PT', '1234.56'),
    ('Poland', 'pl_PL', '1 234,56'),
]

for name, loc, a in sample_data:
    locale.setlocale(locale.LC_ALL, loc)
    print('{:>10}: {:>9} => {:f}'.format(
        name,
        a,
        locale.atof(a),
    ))
```
解析器可识别locale的分组和小数分隔符值。

```
$ python3 locale_atof.py

       USA:  1,234.56 => 1234.560000
    France:   1234,56 => 1234.560000
     Spain:   1234,56 => 1234.560000
  Portugal:   1234.56 => 1234.560000
    Poland:  1 234,56 => 1234.560000
```

### Dates and Times

本地化的另一个重要方面是日期和时间格式。

```python
#locale_date.py
import locale
import time

sample_locales = [
    ('USA', 'en_US'),
    ('France', 'fr_FR'),
    ('Spain', 'es_ES'),
    ('Portugal', 'pt_PT'),
    ('Poland', 'pl_PL'),
]

for name, loc in sample_locales:
    locale.setlocale(locale.LC_ALL, loc)
    format = locale.nl_langinfo(locale.D_T_FMT)
    print('{:>10}: {}'.format(name, time.strftime(format)))
```
此示例使用区域设置的日期格式字符串来打印当前日期和时间。

```
$ python3 locale_date.py

       USA: Sun Mar 18 16:20:59 2018
    France: Dim 18 mar 16:20:59 2018
     Spain: dom 18 mar 16:20:59 2018
  Portugal: Dom 18 Mar 16:20:59 2018
    Poland: ndz 18 mar 16:20:59 2018
```
