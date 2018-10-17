---
title: PyMOTW-3 --- Text
categories: language
tags: PyMOTW-3
---

str类是Python程序员可用的最明显的文本处理工具，但是标准库中还有许多其他工具可以使高级文本操作变得简单。

程序可以使用string.Template作为str对象特征之外的参数化字符串的简单方法。虽然不像Python包索引中提供的许多Web框架或扩展模块定义的模板那样功能丰富，但string.Template是用户可修改模板的良好中间环节，其中动态值需要插入其他静态文本中。

textwrap模块包括通过限制输出宽度，添加缩进以及插入换行符以一致地包装行来设置段落文本格式的工具。

标准库包括两个模块，用于比较超出字符串对象支持的内置相等和排序比较的文本值。re提供了一个完整的正则表达式库，用C语言实现了速度。正则表达式非常适合在较大的数据集中查找子字符串，将字符串与比另一个固定字符串更复杂的模式进行比较，以及温和的解析。

相反，difflib根据添加，删除或更改的部分计算文本序列之间的实际差异。difflib中比较函数的输出可用于向用户提供有关两个输入中发生更改的位置，文档如何随时间变化等的更详细反馈。

## string — Text Constants and Templates

目的：包含用于处理文本的常量和类。  
string模块可以追溯到最早的Python版本。先前在此模块中实现的许多函数已移至str对象的方法。string模块保留了几个用于处理str对象的有用常量和类。这个讨论将集中在他们身上。

### Functions

函数capwords（）将字符串中的所有单词大写。

```python
#string_capwords.py
import string

s = 'The quick brown fox jumped over the lazy dog.'

print(s)
print(string.capwords(s))
```
结果与通过调用split（）、将结果列表中的单词大写，然后调用join（）来组合结果所获得的结果相同。

```
$ python3 string_capwords.py

The quick brown fox jumped over the lazy dog.
The Quick Brown Fox Jumped Over The Lazy Dog.
```

### Templates

字符串模板作为PEP292的一部分添加，旨在替代内置插值语法。使用string.Template插值，通过在名称前加上`$`（例如`$var`）来标识变量。或者，如果需要将它们与周围文本加以区分，它们也可以用花括号包裹（例如，`${var}`）。  
此示例将简单模板与“使用％运算符和使用str.format（）的新格式字符串语法”的类似的字符串插值进行比较。

```python
#string_template.py
import string

values = {'var': 'foo'}

t = string.Template("""
Variable        : $var
Escape          : $$
Variable in text: ${var}iable
""")

print('TEMPLATE:', t.substitute(values))

s = """
Variable        : %(var)s
Escape          : %%
Variable in text: %(var)siable
"""

print('INTERPOLATION:', s % values)

#由于jekyll中使用ruby的正则表达式语法，用[，]代替{，}显示
s = """
Variable        : {var}
Escape          : [[]]
Variable in text: {var}iable
"""

print('FORMAT:', s.format(**values))
```
在前两种情况下，触发字符（$或％）通过重复两次来转义。对于格式语法，{和}都需要通过重复来转义。

```
$ python3 string_template.py

TEMPLATE:
Variable        : foo
Escape          : $
Variable in text: fooiable

INTERPOLATION:
Variable        : foo
Escape          : %
Variable in text: fooiable

FORMAT:
Variable        : foo
Escape          : {}
Variable in text: fooiable
```
模板和字符串插值或格式化之间的一个关键区别是不考虑参数的类型。这些值将转换为字符串，并将字符串插入到结果中。没有可用的格式化选项。例如，无法控制用于表示浮点值的位数。  
但是，一个好处是，如果并非模板所需的所有值都作为参数提供，则使用safe_substitute（）方法可以避免异常。

```python
#string_template_missing.py
import string

values = {'var': 'foo'}

t = string.Template("$var is here but $missing is not provided")

try:
    print('substitute()     :', t.substitute(values))
except KeyError as err:
    print('ERROR:', str(err))

print('safe_substitute():', t.safe_substitute(values))
```
由于值字典中没有missing值，因此由substitute（）引发KeyError。safe_substitute（）不会引发错误，而是捕获它并在文本中保留变量表达式。

```
$ python3 string_template_missing.py

ERROR: 'missing'
safe_substitute(): foo is here but $missing is not provided
```

### Advanced Templates

可以通过调整用于在模板主体中查找变量名称的正则表达式模式来更改string.Template的默认语法。一种简单的方法是更改​​delimiter和idpattern类属性。

```python
#string_template_advanced.py
import string

class MyTemplate(string.Template):
    delimiter = '%'
    idpattern = '[a-z]+_[a-z]+'

template_text = '''
  Delimiter : %%
  Replaced  : %with_underscore
  Ignored   : %notunderscored
'''

d = {
    'with_underscore': 'replaced',
    'notunderscored': 'not replaced',
}

t = MyTemplate(template_text)
print('Modified ID pattern:')
print(t.safe_substitute(d))
```
在此示例中，替换规则已更改，因此分隔符为％而不是$，变量名称必须在中间某处包含下划线。模式％notunderscored不会被任何内容替换，因为它不包含下划线字符。

```
$ python3 string_template_advanced.py

Modified ID pattern:

  Delimiter : %
  Replaced  : replaced
  Ignored   : %notunderscored
```
对于更复杂的更改，可以覆盖pattern属性并定义全新的正则表达式。提供的模式必须包含四个命名组，用于捕获转义分隔符，命名变量，变量名称的大括号版本以及无效分隔符模式。

```python
#string_template_defaultpattern.py
import string

t = string.Template('$var')
print(t.pattern.pattern)
```
t.pattern的值是已编译的正则表达式，但原始字符串可通过其pattern属性获得。

```
\$(?:
  (?P<escaped>\$) |                # two delimiters
  (?P<named>[_a-z][_a-z0-9]*)    | # identifier
  {(?P<braced>[_a-z][_a-z0-9]*)} | # braced identifier
  (?P<invalid>)                    # ill-formed delimiter exprs
)
```
此示例使用{{var}}作为变量语法定义新模式以创建新类型的模板。

```python
#string_template_newsyntax.py
import re
import string

#由于jekyll中使用的ruby的正则表达式语法问题，用[，]代替{，}显示
class MyTemplate(string.Template):
    delimiter = '[['

    pattern = r'''
    \[\[(?:
    (?P<escaped>\[\[)|
    (?P<named>[_a-z][_a-z0-9]*)\]\]|
    (?P<braced>[_a-z][_a-z0-9]*)\]\]|
    (?P<invalid>)
    )
    '''

t = MyTemplate('''
[[[[
[[var]]
''')

print('MATCHES:', t.pattern.findall(t.template))
print('SUBSTITUTED:', t.safe_substitute(var='replacement'))
```
named和braced模式都必须单独提供，即使它们是相同的。运行示例程序会生成以下输出：

```
$ python3 string_template_newsyntax.py

MATCHES: [('[[', '', '', ''), ('', 'var', '', '')]
SUBSTITUTED:
[[
replacement
```

### Formatter

Formatter类实现与str的format（）方法相同的布局规范语言。其功能包括类型coersion，对齐，属性和字段引用，命名和位置模板参数以及特定于类型的格式选项。大多数情况下，format（）方法是这些功能的更方便的接口，但是对于需要变化的情况，提供Formatter作为一种构建子类的方法。

### Constants

string模块包括许多与ASCII和数字字符集相关的常量。

```python
#string_constants.py
import inspect
import string

def is_str(value):
    return isinstance(value, str)

for name, value in inspect.getmembers(string, is_str):
    if name.startswith('_'):
        continue
    print('%s=%r\n' % (name, value))
```
这些常量在处理ASCII数据时很有用，但由于在某写Unicode形式中遇到非ASCII文本越来越常见，因此它们的应用受到限制。

```
$ python3 string_constants.py

ascii_letters='abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ'

ascii_lowercase='abcdefghijklmnopqrstuvwxyz'

ascii_uppercase='ABCDEFGHIJKLMNOPQRSTUVWXYZ'

digits='0123456789'

hexdigits='0123456789abcdefABCDEF'

octdigits='01234567'

printable='0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQ
RSTUVWXYZ!"#$%&\'()*+,-./:;<=>?@[\\]^_`{|}~ \t\n\r\x0b\x0c'

punctuation='!"#$%&\'()*+,-./:;<=>?@[\\]^_`{|}~'

whitespace=' \t\n\r\x0b\x0c'
```

## textwrap — Formatting Text Paragraphs

目的：通过调整段落中换行符出现的位置来格式化文本。  
textwrap模块可用于在需要漂亮打印的情况下格式化文本以进行输出。它提供类似于许多文本编辑器和文字处理器中的段落包装或填充功能的编程性功能。

### Example Data

本节中的示例使用模块`textwrap_example.py`，其中包含字符串sample_text。

```python
#textwrap_example.py
sample_text = '''
    The textwrap module can be used to format text for output in
    situations where pretty-printing is desired.  It offers
    programmatic functionality similar to the paragraph wrapping
    or filling features found in many text editors.
    '''
```

### Filling Paragraphs

fill（）函数将文本作为输入，并生成格式化文本作为输出。

```python
#textwrap_fill.py
import textwrap
from textwrap_example import sample_text

print(textwrap.fill(sample_text, width=50))
```
结果不太理想。文本现在左对齐，但第一行保留其缩进，并且每个后续行的前面的空格都嵌入到段落中。

```
$ python3 textwrap_fill.py

     The textwrap module can be used to format
text for output in     situations where pretty-
printing is desired.  It offers     programmatic
functionality similar to the paragraph wrapping
or filling features found in many text editors.
```

### Removing Existing Indentation

前面的示例已将嵌入的tabs和额外的空格混合到输出的中间，因此它的格式不是非常干净。使用dedent（）从示例文本中的所有行中删除公共空白前缀会产生更好的结果，并且当删除代码本身的格式时，允许直接从Python代码中使用文档字符串或嵌入的多行字符串。示例字符串引入的人工缩进级别，用于说明此功能。

```python
#textwrap_dedent.py
import textwrap
from textwrap_example import sample_text

dedented_text = textwrap.dedent(sample_text)
print('Dedented:')
print(dedented_text)
```
结果开始变得更好。

```
$ python3 textwrap_dedent.py

Dedented:

The textwrap module can be used to format text for output in
situations where pretty-printing is desired.  It offers
programmatic functionality similar to the paragraph wrapping
or filling features found in many text editors.
```
由于“dedent”与“indent”相反，因此结果是一个文本块，每行删除了共同的初始空格。如果一行已经比另一行缩进更多，则不会删除某些空格。

输入像这样：

```
␣Line one.
␣␣␣Line two.
␣Line three.
```
变成：

```
Line one.
␣␣Line two.
Line three.
```

### Combining Dedent and Fill

接下来，dedented文本可以和几个不同的width值一起传递给fill（）。

```python
#textwrap_fill_width.py
import textwrap
from textwrap_example import sample_text

dedented_text = textwrap.dedent(sample_text).strip()
for width in [45, 60]:
    print('{} Columns:\n'.format(width))
    print(textwrap.fill(dedented_text, width=width))
    print()
```
这将生成指定宽度的输出。

```
$ python3 textwrap_fill_width.py

45 Columns:

The textwrap module can be used to format
text for output in situations where pretty-
printing is desired.  It offers programmatic
functionality similar to the paragraph
wrapping or filling features found in many
text editors.

60 Columns:

The textwrap module can be used to format text for output in
situations where pretty-printing is desired.  It offers
programmatic functionality similar to the paragraph wrapping
or filling features found in many text editors.
```

### Indenting Blocks

使用indent（）函数将一致的前缀文本添加到字符串中的所有行。此示例将相同的示例文本格式化为好像它是回复中引用的电子邮件消息的一部分，使用>作为每行的前缀。

```python
#textwrap_indent.py
import textwrap
from textwrap_example import sample_text

dedented_text = textwrap.dedent(sample_text)
wrapped = textwrap.fill(dedented_text, width=50)
wrapped += '\n\nSecond paragraph after a blank line.'
final = textwrap.indent(wrapped, '> ')

print('Quoted block:\n')
print(final)
```
文本块在换行符上拆分，前缀添加到包含文本的每一行，然后将这些行组合回一个新的字符串并返回。

```
$ python3 textwrap_indent.py

Quoted block:

>  The textwrap module can be used to format text
> for output in situations where pretty-printing is
> desired.  It offers programmatic functionality
> similar to the paragraph wrapping or filling
> features found in many text editors.

> Second paragraph after a blank line.
```
要控制哪些行接收新前缀，传递一个callable作为predicate参数给indent（）。将依次为每行文本调用callable，并为返回值为true的行添加前缀。

```python
#textwrap_indent_predicate.py
import textwrap
from textwrap_example import sample_text

def should_indent(line):
    print('Indent {!r}?'.format(line))
    return len(line.strip()) % 2 == 0

dedented_text = textwrap.dedent(sample_text)
wrapped = textwrap.fill(dedented_text, width=50)
final = textwrap.indent(wrapped, 'EVEN ', predicate=should_indent)

print('\nQuoted block:\n')
print(final)
```
此示例将前缀EVEN添加到包含偶数个字符的行。

```
$ python3 textwrap_indent_predicate.py

Indent ' The textwrap module can be used to format text\n'?
Indent 'for output in situations where pretty-printing is\n'?
Indent 'desired.  It offers programmatic functionality\n'?
Indent 'similar to the paragraph wrapping or filling\n'?
Indent 'features found in many text editors.'?

Quoted block:

EVEN  The textwrap module can be used to format text
for output in situations where pretty-printing is
EVEN desired.  It offers programmatic functionality
EVEN similar to the paragraph wrapping or filling
EVEN features found in many text editors.
```

### Hanging Indents

以设置输出宽度相同的方式，可以独立控制第一行和后续行的缩进。

```python
#textwrap_hanging_indent.py
import textwrap
from textwrap_example import sample_text

dedented_text = textwrap.dedent(sample_text).strip()
print(textwrap.fill(dedented_text,
                    initial_indent='',
                    subsequent_indent=' ' * 4,
                    width=50,
                    ))
```
这使得可以产生悬挂缩进，其中第一行缩进小于其他行。

```
$ python3 textwrap_hanging_indent.py

The textwrap module can be used to format text for
    output in situations where pretty-printing is
    desired.  It offers programmatic functionality
    similar to the paragraph wrapping or filling
    features found in many text editors.
```
缩进值也可以包括非空白字符。例如，悬挂缩进可以以*为前缀以产生子弹点。

### Truncating Long Text

要截断文本以创建摘要或预览，请使用shorten（）。所有现有的空格，例如制表符，换行符和多个空格系列，将标准化为单个空格。然后，在文字边界之间将文本截断为小于或等于所请求的长度，以便不包括单词的部分。

```python
#textwrap_shorten.py
import textwrap
from textwrap_example import sample_text

dedented_text = textwrap.dedent(sample_text)
original = textwrap.fill(dedented_text, width=50)

print('Original:\n')
print(original)

shortened = textwrap.shorten(original, 100)
shortened_wrapped = textwrap.fill(shortened, width=50)

print('\nShortened:\n')
print(shortened_wrapped)
```
如果非空白文本作为截断的一部分从原始文本中删除，则将其替换为占位符值。可以通过为shorten（）提供placeholder参数来替换默认值[...]。

```
$ python3 textwrap_shorten.py

Original:

 The textwrap module can be used to format text
for output in situations where pretty-printing is
desired.  It offers programmatic functionality
similar to the paragraph wrapping or filling
features found in many text editors.

Shortened:

The textwrap module can be used to format text for
output in situations where pretty-printing [...]
```

## re — Regular Expressions

目的：使用正式模式在内部搜索和更改文本。  
正则表达式是使用正式语法描述的文本匹配模式。模式被解释为一组指令，然后使用字符串作为输入来执行，以产生匹配子集或原始的修改版本。在会话中，术语“regular expressions”经常缩写为“regex”或“regexp”。表达式可以包括字面文本匹配，重复，模式组合，分支和其他复杂规则。使用正则表达式比使用专用词法分析器和解析器更容易解决大量解析问题。

正则表达式通常用于涉及大量文本处理的应用程序。例如，它们通常用作开发人员使用的文本编辑程序中的搜索模式，包括vi，emacs和现代IDE。它们也是Unix命令行实用程序（如sed，grep和awk）的组成部分。许多编程语言都支持语言语法中的正则表达式（Perl，Ruby，Awk和Tcl）。其他语言（如C，C ++和Python）通过扩展库支持正则表达式。

存在多个正则表达式的开源实现，每个实现共享一个共同的核心语法，但对其高级功能具有不同的扩展或修改。Python的re模块中使用的语法基于Perl中用于正则表达式的语法，并带有一些特定于Python的增强功能。

>Note  
虽然“正则表达式”的正式定义仅限于描述常规语言的表达式，但是受支持的一些扩展不仅仅是描述常规语言。这里使用术语“regular expression”在更一般意义上表示可以由Python的re模块评估的任何表达式。

### Finding Patterns in Text

re的最常见用途是在文本中搜索模式。search（）函数采用模式和文本进行扫描，并在找到模式时返回Match对象。如果未找到模式，则search（）返回None。每个Match对象都包含有关匹配性质的信息，包括原始输入字符串，使用的正则表达式以及模式出现在原始字符串中的位置。

```python
#re_simple_match.py
import re

pattern = 'this'
text = 'Does this text match the pattern?'

match = re.search(pattern, text)

s = match.start()
e = match.end()

print('Found "{}"\nin "{}"\nfrom {} to {} ("{}")'.format(
    match.re.pattern, match.string, s, e, text[s:e]))
```
start（）和end（）方法将索引赋予字符串，显示模式匹配的文本的位置。

```
$ python3 re_simple_match.py

Found "this"
in "Does this text match the pattern?"
from 5 to 9 ("this")
```

### Compiling Expressions

虽然re包含将正则表达式作为文本字符串处理的模块级函数，但程序经常使用的将表达式compile更有效。
compile（）函数将表达式字符串转换为RegexObject。

```python
#re_simple_compiled.py
import re

# Precompile the patterns
regexes = [ re.compile(p) for p in ['this', 'that'] ]
text = 'Does this text match the pattern?'

print('Text: {!r}\n'.format(text))

for regex in regexes:
    print('Seeking "{}" ->'.format(regex.pattern),
          end=' ')

    if regex.search(text):
        print('match!')
    else:
        print('no match')
```
模块级函数维护已编译表达式的缓存，但缓存的大小是有限的，并且使用编译表达式直接避免了与缓存查找相关的开销。使用编译表达式的另一个优点是，通过在加载模块时预编译所有表达式，编译工作将转移到应用程序启动时间，而不是发生在程序可能响应用户操作的位置。

```
$ python3 re_simple_compiled.py

Text: 'Does this text match the pattern?'

Seeking "this" -> match!
Seeking "that" -> no match
```

### Multiple Matches

到目前为止，示例模式都使用search（）来查找字面文本字符串的单个实例。
findall（）函数返回与模式匹配但不重叠的输入的所有子字符串。

```python
#re_findall.py
import re

text = 'abbaaabbbbaaaaa'

pattern = 'ab'

for match in re.findall(pattern, text):
    print('Found {!r}'.format(match))
```
此示例输入字符串包括两个ab实例。

```
$ python3 re_findall.py

Found 'ab'
Found 'ab'
```
finditer（）函数返回一个迭代器，它生成Match实例而不是findall（）返回的字符串。

```python
#re_finditer.py
import re

text = 'abbaaabbbbaaaaa'

pattern = 'ab'

for match in re.finditer(pattern, text):
    s = match.start()
    e = match.end()
    print('Found {!r} at {:d}:{:d}'.format(text[s:e], s, e))
```
此示例查找ab相同的两次出现，并且Match实例显示在原始输入中它们被找到的位置。

```
$ python3 re_finditer.py

Found 'ab' at 0:2
Found 'ab' at 5:7
```

### Pattern Syntax

正则表达式支持比简单字面文本字符串更强大的模式。模式可以重复，可以锚定到输入中的不同逻辑位置，并且可以以紧凑形式表示，这种表示形式不需要在模式中存在每个字面字符。所有这些功能都通过将字面文本值与元字符组合使用，元字符是re实现的正则表达式模式语法的一部分。

```python
#re_test_patterns.py
import re

def test_patterns(text, patterns):
    """Given source text and a list of patterns, look for
    matches for each pattern within the text and print
    them to stdout.
    """
    # Look for each pattern in the text and print the results
    for pattern, desc in patterns:
        print("'{}' ({})\n".format(pattern, desc))
        print("  '{}'".format(text))
        for match in re.finditer(pattern, text):
            s = match.start()
            e = match.end()
            substr = text[s:e]
            n_backslashes = text[:s].count('\\')
            prefix = '.' * (s + n_backslashes)
            print("  {}'{}'".format(prefix, substr))
        print()
    return

if __name__ == '__main__':
    test_patterns('abbaaabbbbaaaaa',
                  [('ab', "'a' followed by 'b'"), ])
```
以下示例将使用test_patterns（）来探索“模式的变化如何改变它们匹配相同输入文本的方式”。输出显示输入文本以及与模式匹配的输入的每个部分的子字符串范围。

```
$ python3 re_test_patterns.py

'ab' ('a' followed by 'b')

  'abbaaabbbbaaaaa'
  'ab'
  .....'ab'
```

#### Repetition

在模式中有五种表达重复的方法。模式后跟着元字符`*`表示模式重复零次或多次（允许模式重复零次意味着它根本不需要出现以匹配）。如果`*`替换为`+`，则模式必须至少出现一次。用`？`表示模式出现零或一次。对于特定的出现次数，在模式后使用`{m}`，其中m是模式应重复的次数。最后，为了允许可变但有限数量的重复，使用`{m，n}`，其中`m`是最小重复次数，`n`是最大值。省略`n（{m，}）`意味着该值必须至少出现m次，没有最大值。

```python
#re_repetition.py
from re_test_patterns import test_patterns

test_patterns(
    'abbaabbba',
    [('ab*', 'a followed by zero or more b'),
     ('ab+', 'a followed by one or more b'),
     ('ab?', 'a followed by zero or one b'),
     ('ab{3}', 'a followed by three b'),
     ('ab{2,3}', 'a followed by two to three b')],
)
```
ab*和ab?比ab+有更多匹配。

```
$ python3 re_repetition.py

'ab*' (a followed by zero or more b)

  'abbaabbba'
  'abb'
  ...'a'
  ....'abbb'
  ........'a'

'ab+' (a followed by one or more b)

  'abbaabbba'
  'abb'
  ....'abbb'

'ab?' (a followed by zero or one b)

  'abbaabbba'
  'ab'
  ...'a'
  ....'ab'
  ........'a'

'ab{3}' (a followed by three b)

  'abbaabbba'
  ....'abbb'

'ab{2,3}' (a followed by two to three b)

  'abbaabbba'
  'abb'
  ....'abbb'
```
处理重复指令时，re通常会在匹配模式时消耗尽可能多的输入。这种所谓的贪婪行为可能导致更少的独立匹配，或者匹配可能包括比预期更多的输入文本。通过使用重复指令后跟`?`可以关闭贪婪。

```python
#re_repetition_non_greedy.py
from re_test_patterns import test_patterns

test_patterns(
    'abbaabbba',
    [('ab*?', 'a followed by zero or more b'),
     ('ab+?', 'a followed by one or more b'),
     ('ab??', 'a followed by zero or one b'),
     ('ab{3}?', 'a followed by three b'),
     ('ab{2,3}?', 'a followed by two to three b')],
)
```
对于允许零次出现`b`的任何模式“禁用贪婪消耗输入”意味着匹配的子字符串不包含任何`b`字符。

```
$ python3 re_repetition_non_greedy.py

'ab*?' (a followed by zero or more b)

  'abbaabbba'
  'a'
  ...'a'
  ....'a'
  ........'a'

'ab+?' (a followed by one or more b)

  'abbaabbba'
  'ab'
  ....'ab'

'ab??' (a followed by zero or one b)

  'abbaabbba'
  'a'
  ...'a'
  ....'a'
  ........'a'

'ab{3}?' (a followed by three b)

  'abbaabbba'
  ....'abbb'

'ab{2,3}?' (a followed by two to three b)

  'abbaabbba'
  'abb'
  ....'abb'
```

#### Character Sets

字符集是一组字符，其中任何一个字符都可以匹配模式中的该点。例如，`[ab]`将匹配`a`或`b`。

```python
#re_charset.py
from re_test_patterns import test_patterns

test_patterns(
    'abbaabbba',
    [('[ab]', 'either a or b'),
     ('a[ab]+', 'a followed by 1 or more a or b'),
     ('a[ab]+?', 'a followed by 1 or more a or b, not greedy')],
)
```
表达式的贪婪形式`（a[ab]+）`消耗整个字符串，因为第一个字母是a，后面的每个字符都是a或b。

```
$ python3 re_charset.py

'[ab]' (either a or b)

  'abbaabbba'
  'a'
  .'b'
  ..'b'
  ...'a'
  ....'a'
  .....'b'
  ......'b'
  .......'b'
  ........'a'

'a[ab]+' (a followed by 1 or more a or b)

  'abbaabbba'
  'abbaabbba'

'a[ab]+?' (a followed by 1 or more a or b, not greedy)

  'abbaabbba'
  'ab'
  ...'aa'
```
字符集也可用于排除特定字符。克拉`（^）`表示查找不在克拉后面的集合中的字符。

```python
#re_charset_exclude.py
from re_test_patterns import test_patterns

test_patterns(
    'This is some text -- with punctuation.',
    [('[^-. ]+', 'sequences without -, ., or space')],
)
```
此模式查找不包含字符 - ，。或空格的所有子字符串。

```
$ python3 re_charset_exclude.py

'[^-. ]+' (sequences without -, ., or space)

  'This is some text -- with punctuation.'
  'This'
  .....'is'
  ........'some'
  .............'text'
  .....................'with'
  ..........................'punctuation'
```
随着字符集变大，键入应该（或不应该）匹配的每个字符变得乏味。使用字符范围的更紧凑的格式可用于定义字符集以包括指定的起点和终点之间的所有连续字符。

```python
#re_charset_ranges.py
from re_test_patterns import test_patterns

test_patterns(
    'This is some text -- with punctuation.',
    [('[a-z]+', 'sequences of lowercase letters'),
     ('[A-Z]+', 'sequences of uppercase letters'),
     ('[a-zA-Z]+', 'sequences of letters of either case'),
     ('[A-Z][a-z]+', 'one uppercase followed by lowercase')],
)
```
这里范围a-z包括小写ASCII字母，范围A-Z包括大写ASCII字母。范围也可以组合成单个字符集。

```
$ python3 re_charset_ranges.py

'[a-z]+' (sequences of lowercase letters)

  'This is some text -- with punctuation.'
  .'his'
  .....'is'
  ........'some'
  .............'text'
  .....................'with'
  ..........................'punctuation'

'[A-Z]+' (sequences of uppercase letters)

  'This is some text -- with punctuation.'
  'T'

'[a-zA-Z]+' (sequences of letters of either case)

  'This is some text -- with punctuation.'
  'This'
  .....'is'
  ........'some'
  .............'text'
  .....................'with'
  ..........................'punctuation'

'[A-Z][a-z]+' (one uppercase followed by lowercase)

  'This is some text -- with punctuation.'
  'This'
```
作为字符集的特殊情况，元字符点或句点`（.）`表示该模式应匹配该位置中的任何单个字符。

```python
#re_charset_dot.py
from re_test_patterns import test_patterns

test_patterns(
    'abbaabbba',
    [('a.', 'a followed by any one character'),
     ('b.', 'b followed by any one character'),
     ('a.*b', 'a followed by anything, ending in b'),
     ('a.*?b', 'a followed by anything, ending in b')],
)
```
除非使用非贪婪形式，否则将点与重复组合可以导致非常长的匹配。

```
$ python3 re_charset_dot.py

'a.' (a followed by any one character)

  'abbaabbba'
  'ab'
  ...'aa'

'b.' (b followed by any one character)

  'abbaabbba'
  .'bb'
  .....'bb'
  .......'ba'

'a.*b' (a followed by anything, ending in b)

  'abbaabbba'
  'abbaabbb'

'a.*?b' (a followed by anything, ending in b)

  'abbaabbba'
  'ab'
  ...'aab'
```

#### Escape Codes

更紧凑的表示使用转义码来表示几个预定义的字符集。re识别的转义码列于下表中。

Regular Expression Escape Codes

Code	| Meaning
:---  | :---
\d	  | a digit
\D	  | a non-digit
\s	  | whitespace (tab, space, newline, etc.)
\S	  | non-whitespace
\w	  | alphanumeric
\W	  | non-alphanumeric

>Note  
通过在字符前面加上反斜杠（\）来表示转义。不幸的是，反斜杠本身必须在普通的Python字符串中进行转义，这会导致难以阅读的表达式。使用通过在文字值前加上r来创建的原始字符串可以消除此问题并保持可读性。

```python
#re_escape_codes.py
from re_test_patterns import test_patterns

test_patterns(
    'A prime #1 example!',
    [(r'\d+', 'sequence of digits'),
     (r'\D+', 'sequence of non-digits'),
     (r'\s+', 'sequence of whitespace'),
     (r'\S+', 'sequence of non-whitespace'),
     (r'\w+', 'alphanumeric characters'),
     (r'\W+', 'non-alphanumeric')],
)
```
这些样本表达式将转义码与重复相结合，以在输入字符串中查找相似字符的序列。

```
$ python3 re_escape_codes.py

'\d+' (sequence of digits)

  'A prime #1 example!'
  .........'1'

'\D+' (sequence of non-digits)

  'A prime #1 example!'
  'A prime #'
  ..........' example!'

'\s+' (sequence of whitespace)

  'A prime #1 example!'
  .' '
  .......' '
  ..........' '

'\S+' (sequence of non-whitespace)

  'A prime #1 example!'
  'A'
  ..'prime'
  ........'#1'
  ...........'example!'

'\w+' (alphanumeric characters)

  'A prime #1 example!'
  'A'
  ..'prime'
  .........'1'
  ...........'example'

'\W+' (non-alphanumeric)

  'A prime #1 example!'
  .' '
  .......' #'
  ..........' '
  ..................'!'
```
要匹配作为正则表达式语法一部分的字符，请转义搜索模式中的字符。

```python
#re_escape_escapes.py
from re_test_patterns import test_patterns

test_patterns(
    r'\d+ \D+ \s+',
    [(r'\\.\+', 'escape code')],
)
```
此示例中的模式转义反斜杠和加号字符，因为它们都是元字符并且在正则表达式中具有特殊含义。

```
$ python3 re_escape_escapes.py

'\\.\+' (escape code)

  '\d+ \D+ \s+'
  '\d+'
  .....'\D+'
  ..........'\s+'
```

#### Anchoring

除了描述要匹配的模式的内容之外，通过使用锚定指令还可以指定模式应该出现在输入文本中的相对位置。下表列出了有效的锚定代码。

Regular Expression Anchoring Codes

Code	| Meaning
:---  | :---
^	    | start of string, or line
$	    | end of string, or line
\A	  | start of string
\Z	  | end of string
\b	  | empty string at the beginning or end of a word
\B	  | empty string not at the beginning or end of a word

```python
#re_anchoring.py
from re_test_patterns import test_patterns

test_patterns(
    'This is some text -- with punctuation.',
    [(r'^\w+', 'word at start of string'),
     (r'\A\w+', 'word at start of string'),
     (r'\w+\S*$', 'word near end of string'),
     (r'\w+\S*\Z', 'word near end of string'),
     (r'\w*t\w*', 'word containing t'),
     (r'\bt\w+', 't at start of word'),
     (r'\w+t\b', 't at end of word'),
     (r'\Bt\B', 't, not start or end of word')],
)
```
用于匹配字符串开头和结尾处的单词的示例中的模式是不同的，因为字符串末尾的单词后跟标点符号以终止句子。因为，模式\ w + $不匹配。不被视为字母数字字符。

```
$ python3 re_anchoring.py

'^\w+' (word at start of string)

  'This is some text -- with punctuation.'
  'This'

'\A\w+' (word at start of string)

  'This is some text -- with punctuation.'
  'This'

'\w+\S*$' (word near end of string)

  'This is some text -- with punctuation.'
  ..........................'punctuation.'

'\w+\S*\Z' (word near end of string)

  'This is some text -- with punctuation.'
  ..........................'punctuation.'

'\w*t\w*' (word containing t)

  'This is some text -- with punctuation.'
  .............'text'
  .....................'with'
  ..........................'punctuation'

'\bt\w+' (t at start of word)

  'This is some text -- with punctuation.'
  .............'text'

'\w+t\b' (t at end of word)

  'This is some text -- with punctuation.'
  .............'text'

'\Bt\B' (t, not start or end of word)

  'This is some text -- with punctuation.'
  .......................'t'
  ..............................'t'
  .................................'t'
```

### Constraining the Search

在预先知道应该仅搜索完整输入的子集的情况下，可以通过告知re限制搜索范围来进一步约束正则表达式匹配。例如，如果模式必须出现在输入的前面，那么使用match（）而不是search（）将锚定搜索，而不必在搜索模式中明确包含锚点。

```python
#re_match.py
import re

text = 'This is some text -- with punctuation.'
pattern = 'is'

print('Text   :', text)
print('Pattern:', pattern)

m = re.match(pattern, text)
print('Match  :', m)
s = re.search(pattern, text)
print('Search :', s)
```
由于字面文本`is`没有出现在输入文本的开头，因此使用match（）找不到。但是，序列在文本中出现另外两次，因此search（）会找到它。

```
$ python3 re_match.py

Text   : This is some text -- with punctuation.
Pattern: is
Match  : None
Search : <_sre.SRE_Match object; span=(2, 4), match='is'>
```
fullmatch（）方法要求整个输入字符串与模式匹配。

```python
#re_fullmatch.py
import re

text = 'This is some text -- with punctuation.'
pattern = 'is'

print('Text       :', text)
print('Pattern    :', pattern)

m = re.search(pattern, text)
print('Search     :', m)
s = re.fullmatch(pattern, text)
print('Full match :', s)
```
这里search（）显示模式确实出现在输入中，但它不消耗所有输入，因此fullmatch（）不报告一个匹配。

```
$ python3 re_fullmatch.py

Text       : This is some text -- with punctuation.
Pattern    : is
Search     : <_sre.SRE_Match object; span=(2, 4), match='is'>
Full match : None
```
已编译正则表达式的search（）方法接受可选的start和end位置参数，以将搜索限制为输入的子字符串。

```python
#re_search_substring.py
import re

text = 'This is some text -- with punctuation.'
pattern = re.compile(r'\b\w*is\w*\b')

print('Text:', text)
print()

pos = 0
while True:
    match = pattern.search(text, pos)
    if not match:
        break
    s = match.start()
    e = match.end()
    print('  {:>2d} : {:>2d} = "{}"'.format(
        s, e - 1, text[s:e]))
    # Move forward in text for the next search
    pos = e
```
此示例实现了一种效率较低的iterall（）形式。每次找到匹配项时，该匹配项的结束位置将用于下一次搜索。

```
$ python3 re_search_substring.py

Text: This is some text -- with punctuation.

   0 :  3 = "This"
   5 :  6 = "is"
```

### Dissecting Matches with Groups

搜索模式的匹配是正则表达式提供的强大功能的基础。将组添加到模式会隔离匹配文本的一部分，扩展这些功能以创建解析器。通过将模式括在括号中来定义组。

```python
#re_groups.py
from re_test_patterns import test_patterns

test_patterns(
    'abbaaabbbbaaaaa',
    [('a(ab)', 'a followed by literal ab'),
     ('a(a*b*)', 'a followed by 0-n a and 0-n b'),
     ('a(ab)*', 'a followed by 0-n ab'),
     ('a(ab)+', 'a followed by 1-n ab')],
)
```
任何完整的正则表达式都可以转换为一个组并嵌套在一个更大的表达式中。所有重复修饰符可以作为整体应用于组，要求重复整个组模式。

```
$ python3 re_groups.py

'a(ab)' (a followed by literal ab)

  'abbaaabbbbaaaaa'
  ....'aab'

'a(a*b*)' (a followed by 0-n a and 0-n b)

  'abbaaabbbbaaaaa'
  'abb'
  ...'aaabbbb'
  ..........'aaaaa'

'a(ab)*' (a followed by 0-n ab)

  'abbaaabbbbaaaaa'
  'a'
  ...'a'
  ....'aab'
  ..........'a'
  ...........'a'
  ............'a'
  .............'a'
  ..............'a'

'a(ab)+' (a followed by 1-n ab)

  'abbaaabbbbaaaaa'
  ....'aab'
```
要访问模式中各个组匹配的子字符串，请使用Match对象的groups（）方法.

```python
#re_groups_match.py
import re

text = 'This is some text -- with punctuation.'

print(text)
print()

patterns = [
    (r'^(\w+)', 'word at start of string'),
    (r'(\w+)\S*$', 'word at end, with optional punctuation'),
    (r'(\bt\w+)\W+(\w+)', 'word starting with t, another word'),
    (r'(\w+t)\b', 'word ending with t'),
]

for pattern, desc in patterns:
    regex = re.compile(pattern)
    match = regex.search(text)
    print("'{}' ({})\n".format(pattern, desc))
    print('  ', match.groups())
    print()
```
Match.groups（）按照表达式中与字符串匹配的组的顺序返回字符串序列。

```
$ python3 re_groups_match.py

This is some text -- with punctuation.

'^(\w+)' (word at start of string)

   ('This',)

'(\w+)\S*$' (word at end, with optional punctuation)

   ('punctuation',)

'(\bt\w+)\W+(\w+)' (word starting with t, another word)

   ('text', 'with')

'(\w+t)\b' (word ending with t)

   ('text',)
```
要获取单个组的匹配，请使用group（）方法。当使用分组来查找字符串的某些部分但结果中不需要某些与组匹配的部分时，这很有用。

```python
#re_groups_individual.py
import re

text = 'This is some text -- with punctuation.'

print('Input text            :', text)

# word starting with 't' then another word
regex = re.compile(r'(\bt\w+)\W+(\w+)')
print('Pattern               :', regex.pattern)

match = regex.search(text)
print('Entire match          :', match.group(0))
print('Word starting with "t":', match.group(1))
print('Word after "t" word   :', match.group(2))
```
组0表示整个表达式匹配的字符串，子组按照左括号出现在表达式中的顺序从1开始编号。

```
$ python3 re_groups_individual.py

Input text            : This is some text -- with punctuation.
Pattern               : (\bt\w+)\W+(\w+)
Entire match          : text -- with
Word starting with "t": text
Word after "t" word   : with
```
Python扩展了基本分组语法以添加命名组。使用名称来引用组可以更容易地随着时间的推移修改模式，而不必使用匹配结果修改代码。要设置组的名称，请使用语法`(?P<name>pattern)`。

```python
#re_groups_named.py
import re

text = 'This is some text -- with punctuation.'

print(text)
print()

patterns = [
    r'^(?P<first_word>\w+)',
    r'(?P<last_word>\w+)\S*$',
    r'(?P<t_word>\bt\w+)\W+(?P<other_word>\w+)',
    r'(?P<ends_with_t>\w+t)\b',
]

for pattern in patterns:
    regex = re.compile(pattern)
    match = regex.search(text)
    print("'{}'".format(pattern))
    print('  ', match.groups())
    print('  ', match.groupdict())
    print()
```
使用groupdict（）检索从匹配中映射组名到子字符串的字典。命名模式也包含在groups（）返回的有序序列中。

```
$ python3 re_groups_named.py

This is some text -- with punctuation.

'^(?P<first_word>\w+)'
   ('This',)
   {'first_word': 'This'}

'(?P<last_word>\w+)\S*$'
   ('punctuation',)
   {'last_word': 'punctuation'}

'(?P<t_word>\bt\w+)\W+(?P<other_word>\w+)'
   ('text', 'with')
   {'t_word': 'text', 'other_word': 'with'}

'(?P<ends_with_t>\w+t)\b'
   ('text',)
   {'ends_with_t': 'text'}
```

test_patterns（）的更新版本显示了被模式匹配的数字编号组和命名组，这将使以下示例更容易理解。

```python
#re_test_patterns_groups.py
import re

def test_patterns(text, patterns):
    """Given source text and a list of patterns, look for
    matches for each pattern within the text and print
    them to stdout.
    """
    # Look for each pattern in the text and print the results
    for pattern, desc in patterns:
        print('{!r} ({})\n'.format(pattern, desc))
        print('  {!r}'.format(text))
        for match in re.finditer(pattern, text):
            s = match.start()
            e = match.end()
            prefix = ' ' * (s)
            print(
                '  {}{!r}{} '.format(prefix,
                                     text[s:e],
                                     ' ' * (len(text) - e)),
                end=' ',
            )
            print(match.groups())
            if match.groupdict():
                print('{}{}'.format(
                    ' ' * (len(text) - s),
                    match.groupdict()),
                )
        print()
    return
```
由于组本身就是一个完整的正则表达式，因此可以将组嵌套在其他组中以构建更复杂的表达式。

```python
#re_groups_nested.py
from re_test_patterns_groups import test_patterns

test_patterns(
    'abbaabbba',
    [(r'a((a*)(b*))', 'a followed by 0-n a and 0-n b')],
)
```
在这种情况下，组（a*）匹配空字符串，因此groups（）的返回值包含该空字符串作为匹配值。

```
$ python3 re_groups_nested.py

'a((a*)(b*))' (a followed by 0-n a and 0-n b)

  'abbaabbba'
  'abb'        ('bb', '', 'bb')
     'aabbb'   ('abbb', 'a', 'bbb')
          'a'  ('', '', '')
```
组也可用于指定可选择的模式。使用管道符号( `|` )分隔两个模式，并指示其中一个模式应匹配。不过，请仔细考虑管道的位置。此示例中的第一个表达式匹配a，后跟一个完全由单个字母a或b组成的序列。第二个模式匹配a后跟一个可能包含a或b的序列。模式类似，但匹配结果完全不同。

```python
#re_groups_alternative.py
from re_test_patterns_groups import test_patterns

test_patterns(
    'abbaabbba',
    [(r'a((a+)|(b+))', 'a then seq. of a or seq. of b'),
     (r'a((a|b)+)', 'a then seq. of [ab]')],
)
```
如果替代组不匹配，但整个模式匹配，则groups（）的返回值在序列中应出现替代组的点处包含None值。

```
$ python3 re_groups_alternative.py

'a((a+)|(b+))' (a then seq. of a or seq. of b)

  'abbaabbba'
  'abb'        ('bb', None, 'bb')
     'aa'      ('a', 'a', None)

'a((a|b)+)' (a then seq. of [ab])

  'abbaabbba'
  'abbaabbba'  ('bbaabbba', 'a')
```
在“与子模式匹配的字符串”不是应从全文中提取的内容的一部分的情况下，定义包含子模式的组也很有用。这类组称为非捕获组。非捕获组可用于描述重复模式或可选择模式，而不会在返回的值中隔离字符串的匹配部分。要创建非捕获组，请使用语法`(?:pattern)`。

```python
#re_groups_noncapturing.py
from re_test_patterns_groups import test_patterns

test_patterns(
    'abbaabbba',
    [(r'a((a+)|(b+))', 'capturing form'),
     (r'a((?:a+)|(?:b+))', 'noncapturing')],
)
```
在以下示例中，比较模式的捕获和非捕获形式返回的组, 匹配相同的结果。

```
$ python3 re_groups_noncapturing.py

'a((a+)|(b+))' (capturing form)

  'abbaabbba'
  'abb'        ('bb', None, 'bb')
     'aa'      ('a', 'a', None)

'a((?:a+)|(?:b+))' (noncapturing)

  'abbaabbba'
  'abb'        ('bb',)
     'aa'      ('a',)
```

### Search Options

选项标志用于更改匹配引擎处理表达式的方式。可以使用按位OR运算组合标志，然后传递给compile（），search（），match（）和其他接受搜索模式的函数。

#### Case-insensitive Matching

IGNORECASE使模式中的字面字符和字符范围匹配大写和小写字符.

```python
#re_flags_ignorecase.py
import re

text = 'This is some text -- with punctuation.'
pattern = r'\bT\w+'
with_case = re.compile(pattern)
without_case = re.compile(pattern, re.IGNORECASE)

print('Text:\n  {!r}'.format(text))
print('Pattern:\n  {}'.format(pattern))
print('Case-sensitive:')
for match in with_case.findall(text):
    print('  {!r}'.format(match))
print('Case-insensitive:')
for match in without_case.findall(text):
    print('  {!r}'.format(match))
```
由于模式包含文字T，如果未设置IGNORECASE，则唯一匹配是单词This。如果忽略大小写，则text也匹配。

```
$ python3 re_flags_ignorecase.py

Text:
  'This is some text -- with punctuation.'
Pattern:
  \bT\w+
Case-sensitive:
  'This'
Case-insensitive:
  'This'
  'text'
```

#### Input with Multiple Lines

两个标志会影响多行输入的搜索方式：MULTILINE和DOTALL。MULTILINE标志控制模式匹配代码如何处理包含换行符的文本的锚定指令。打开多行模式时，除了整个字符串之外，`^`和`$`的锚点规则适用于每行的开头和结尾。

```python
#re_flags_multiline.py
import re

text = 'This is some text -- with punctuation.\nA second line.'
pattern = r'(^\w+)|(\w+\S*$)'
single_line = re.compile(pattern)
multiline = re.compile(pattern, re.MULTILINE)

print('Text:\n  {!r}'.format(text))
print('Pattern:\n  {}'.format(pattern))
print('Single Line :')
for match in single_line.findall(text):
    print('  {!r}'.format(match))
print('Multline    :')
for match in multiline.findall(text):
    print('  {!r}'.format(match))
```
示例中的模式匹配输入的第一个或最后一个单词。它匹配线。在字符串的末尾，即使没有换行符。

```
$ python3 re_flags_multiline.py

Text:
  'This is some text -- with punctuation.\nA second line.'
Pattern:
  (^\w+)|(\w+\S*$)
Single Line :
  ('This', '')
  ('', 'line.')
Multline    :
  ('This', '')
  ('', 'punctuation.')
  ('A', '')
  ('', 'line.')
```
DOTALL是与多行文本相关的另一个标志。通常，点字符`（.）`匹配输入文本中除换行符之外的所有内容。该标志还允许点匹配换行符。

```python
#re_flags_dotall.py
import re

text = 'This is some text -- with punctuation.\nA second line.'
pattern = r'.+'
no_newlines = re.compile(pattern)
dotall = re.compile(pattern, re.DOTALL)

print('Text:\n  {!r}'.format(text))
print('Pattern:\n  {}'.format(pattern))
print('No newlines :')
for match in no_newlines.findall(text):
    print('  {!r}'.format(match))
print('Dotall      :')
for match in dotall.findall(text):
    print('  {!r}'.format(match))
```
如果没有标志，输入文本的每一行将分别与模式匹配。添加标志会导致整个字符串被占用。

```
$ python3 re_flags_dotall.py

Text:
  'This is some text -- with punctuation.\nA second line.'
Pattern:
  .+
No newlines :
  'This is some text -- with punctuation.'
  'A second line.'
Dotall      :
  'This is some text -- with punctuation.\nA second line.'
```

#### Unicode

在Python 3下，str对象使用完整的Unicode字符集，str上的正则表达式处理假定模式和输入文本都是Unicode。前面描述的转义码默认使用Unicode定义。这些假设意味着模式`\w+`将匹配单词“French”和“Français”。要将转义码限制为ASCII字符集（也是Python 2中的默认值），请在编译模式时或在调用模块级函数search（）和match（）时使用ASCII标志。

```python
#re_flags_ascii.py
import re

text = u'Français złoty Österreich'
pattern = r'\w+'
ascii_pattern = re.compile(pattern, re.ASCII)
unicode_pattern = re.compile(pattern)

print('Text    :', text)
print('Pattern :', pattern)
print('ASCII   :', list(ascii_pattern.findall(text)))
print('Unicode :', list(unicode_pattern.findall(text)))
```
对于ASCII文本，其他转义序列（\W，\b，\B，\d，\D，\s和\S）的处理方式也不同。re使用字符集的ASCII定义而不是通过Unicode数据库来查找每个字符的属性。

```
$ python3 re_flags_ascii.py

Text    : Français złoty Österreich
Pattern : \w+
ASCII   : ['Fran', 'ais', 'z', 'oty', 'sterreich']
Unicode : ['Français', 'złoty', 'Österreich']
```

#### Verbose Expression Syntax

随着表达式变得更加复杂，正则表达式的紧凑格式语法可能成为障碍。随着表达式中组的数量的增加，将需要做更多的事情以跟踪为什么每个元素被需要以及表达式的各个部分如何正确的交互。使用命名组有助于缓解这些问题，但更好的解决方案是使用详细模式（verbose mode）表达式，它允许将注释和额外的空白嵌入到模式中。

验证电子邮件地址的模式将说明详细模式如何使正则表达式更容易处理。第一个版本识别以三个顶级域之一结尾的地址：.com，.org或.edu。

```python
#re_email_compact.py
import re

address = re.compile('[\w\d.+-]+@([\w\d.]+\.)+(com|org|edu)')

candidates = [
    u'first.last@example.com',
    u'first.last+category@gmail.com',
    u'valid-address@mail.example.com',
    u'not-valid@example.foo',
]

for candidate in candidates:
    match = address.search(candidate)
    print('{:<30}  {}'.format(
        candidate, 'Matches' if match else 'No match')
    )
```
这个表达式已经很复杂了。有几个字符类，组和重复表达式。

```
$ python3 re_email_compact.py

first.last@example.com          Matches
first.last+category@gmail.com   Matches
valid-address@mail.example.com  Matches
not-valid@example.foo           No match
```
将表达式转换为更详细的格式将使其更容易扩展。

```python
#re_email_verbose.py
import re

address = re.compile(
    '''
    [\w\d.+-]+       # username
    @
    ([\w\d.]+\.)+    # domain name prefix
    (com|org|edu)    # TODO: support more top-level domains
    ''',
    re.VERBOSE)

candidates = [
    u'first.last@example.com',
    u'first.last+category@gmail.com',
    u'valid-address@mail.example.com',
    u'not-valid@example.foo',
]

for candidate in candidates:
    match = address.search(candidate)
    print('{:<30}  {}'.format(
        candidate, 'Matches' if match else 'No match'),
    )
```
表达式匹配相同的输入，但在这种扩展格式中，它更容易阅读。注释还有助于识别模式的不同部分，以便可以扩展以匹配更多输入。

```
$ python3 re_email_verbose.py

first.last@example.com          Matches
first.last+category@gmail.com   Matches
valid-address@mail.example.com  Matches
not-valid@example.foo           No match
```
此扩展版本会解析包含人名和电子邮件地址的输入，这些输入可能会显示在电子邮件标题中。该名称首先出现并且独立存在，接着是电子邮件地址，用尖括号（<和>）包围。

```python
#re_email_with_name.py
import re

address = re.compile(
    '''

    # A name is made up of letters, and may include "."
    # for title abbreviations and middle initials.
    ((?P<name>
       ([\w.,]+\s+)*[\w.,]+)
       \s*
       # Email addresses are wrapped in angle
       # brackets < >, but only if a name is
       # found, so keep the start bracket in this
       # group.
       <
    )? # the entire name is optional

    # The address itself: username@domain.tld
    (?P<email>
      [\w\d.+-]+       # username
      @
      ([\w\d.]+\.)+    # domain name prefix
      (com|org|edu)    # limit the allowed top-level domains
    )

    >? # optional closing angle bracket
    ''',
    re.VERBOSE)

candidates = [
    u'first.last@example.com',
    u'first.last+category@gmail.com',
    u'valid-address@mail.example.com',
    u'not-valid@example.foo',
    u'First Last <first.last@example.com>',
    u'No Brackets first.last@example.com',
    u'First Last',
    u'First Middle Last <first.last@example.com>',
    u'First M. Last <first.last@example.com>',
    u'<first.last@example.com>',
]

for candidate in candidates:
    print('Candidate:', candidate)
    match = address.search(candidate)
    if match:
        print('  Name :', match.groupdict()['name'])
        print('  Email:', match.groupdict()['email'])
    else:
        print('  No match')
```
与其他编程语言一样，将注释插入到详细正则表达式中的能力有助于其可维护性。最终版本包括对未来维护者的实现说明和将这些组彼此分开的空白并突出显示其嵌套级别。

```
$ python3 re_email_with_name.py

Candidate: first.last@example.com
  Name : None
  Email: first.last@example.com
Candidate: first.last+category@gmail.com
  Name : None
  Email: first.last+category@gmail.com
Candidate: valid-address@mail.example.com
  Name : None
  Email: valid-address@mail.example.com
Candidate: not-valid@example.foo
  No match
Candidate: First Last <first.last@example.com>
  Name : First Last
  Email: first.last@example.com
Candidate: No Brackets first.last@example.com
  Name : None
  Email: first.last@example.com
Candidate: First Last
  No match
Candidate: First Middle Last <first.last@example.com>
  Name : First Middle Last
  Email: first.last@example.com
Candidate: First M. Last <first.last@example.com>
  Name : First M. Last
  Email: first.last@example.com
Candidate: <first.last@example.com>
  Name : None
  Email: first.last@example.com
```

#### Embedding Flags in Patterns

在编译表达式时无法添加标志的情况下，例如当模式作为参数传递给稍后将编译它的库函数时，标志可以嵌入到表达式字符串本身中。例如，要打开不区分大小写的匹配，请将`(?i)`添加到表达式的开头。

```python
#re_flags_embedded.py
import re

text = 'This is some text -- with punctuation.'
pattern = r'(?i)\bT\w+'
regex = re.compile(pattern)

print('Text      :', text)
print('Pattern   :', pattern)
print('Matches   :', regex.findall(text))
```
因为这些选项控制着评估或解析整个表达式的方式，所以它们应始终出现在表达式的开头。

```
$ python3 re_flags_embedded.py

Text      : This is some text -- with punctuation.
Pattern   : (?i)\bT\w+
Matches   : ['This', 'text']
```
所有标志的缩写都列在下表中。  
Regular Expression Flag Abbreviations

Flag	      | Abbreviation
:---        | :---
ASCII	      | a
IGNORECASE	| i
MULTILINE	  | m
DOTALL	    | s
VERBOSE	    | x

嵌入式标志可以通过将它们放在同一组中来组合。例如，`(?im)`打开多行字符串的不区分大小写的匹配。

### Looking Ahead or Behind

在许多情况下，仅当某个其他部分也匹配时才匹配模式的一部分是有用的。例如，在电子邮件解析表达式中，尖括号被标记为可选。实际上，括号应该是配对的，只有当两者都存在或都不存在时，表达式才应该匹配。该表达式的修改版本使用正向前置断言来匹配该对括号。前瞻断言语法是`(?=pattern)`。
>(?=pattern) 当模式匹配时成功，但不消耗字符  
(?=exp)也叫“零宽度正预测先行断言”，它断言自身出现的位置的后面能匹配表达式exp，比如\b\w+(?=ing\b)，匹配以ing结尾的单词的前面部分(除了ing以外的部分)，如查找I'm singing while you're dancing.时，它会匹配sing和danc

```python
#re_look_ahead.py
import re

address = re.compile(
    '''
    # A name is made up of letters, and may include "."
    # for title abbreviations and middle initials.
    ((?P<name>
       ([\w.,]+\s+)*[\w.,]+
     )
     \s+
    ) # name is no longer optional

    # LOOKAHEAD
    # Email addresses are wrapped in angle brackets, but only
    # if both are present or neither is.
    (?= (<.*>$)       # remainder wrapped in angle brackets
        |
        ([^<].*[^>]$) # remainder *not* wrapped in angle brackets
      )

    <? # optional opening angle bracket

    # The address itself: username@domain.tld
    (?P<email>
      [\w\d.+-]+       # username
      @
      ([\w\d.]+\.)+    # domain name prefix
      (com|org|edu)    # limit the allowed top-level domains
    )

    >? # optional closing angle bracket
    ''',
    re.VERBOSE)

candidates = [
    u'First Last <first.last@example.com>',
    u'No Brackets first.last@example.com',
    u'Open Bracket <first.last@example.com',
    u'Close Bracket first.last@example.com>',
]

for candidate in candidates:
    print('Candidate:', candidate)
    match = address.search(candidate)
    if match:
        print('  Name :', match.groupdict()['name'])
        print('  Email:', match.groupdict()['email'])
    else:
        print('  No match')
```
此版本的表达式有几个重要的更改。首先，名称部分不再是可选的。这意味着独立地址不匹配，但它也可以防止匹配格式不正确的名称/地址组合。 “name”组之后的正向预测规则声明字符串的其余部分用一对尖括号包裹，或者没有不匹配的括号;括号中的两个都存在或两个都不存在。前瞻表示为一个组，但是前瞻组的匹配不会消耗任何输入文本，因此在前瞻匹配之后，模式的其余部分从同一位置提取。

```
$ python3 re_look_ahead.py

Candidate: First Last <first.last@example.com>
  Name : First Last
  Email: first.last@example.com
Candidate: No Brackets first.last@example.com
  Name : No Brackets
  Email: first.last@example.com
Candidate: Open Bracket <first.last@example.com
  No match
Candidate: Close Bracket first.last@example.com>
  No match
```
负向前置断言（`(?!pattern)`）表示该模式与当前点之后的文本不匹配。
例如，可以修改电子邮件识别模式以忽略自动化系统常用的noreply邮件地址。
>(?!pattern) 当模式不匹配时成功，但不消耗字符。 “零宽度负预测先行断言”(?!exp)，断言此位置的后面不能匹配表达式exp。例如： \d{3}(?!\d)匹配三位数字，而且这三位数字的后面不能是数字；

```python
#re_negative_look_ahead.py
import re

address = re.compile(
    '''
    ^

    # An address: username@domain.tld

    # Ignore noreply addresses
    (?!noreply@.*$)

    [\w\d.+-]+       # username
    @
    ([\w\d.]+\.)+    # domain name prefix
    (com|org|edu)    # limit the allowed top-level domains

    $
    ''',
    re.VERBOSE)

candidates = [
    u'first.last@example.com',
    u'noreply@example.com',
]

for candidate in candidates:
    print('Candidate:', candidate)
    match = address.search(candidate)
    if match:
        print('  Match:', candidate[match.start():match.end()])
    else:
        print('  No match')
```
以noreply开头的地址与模式不匹配，因为前瞻断言失败。

```
$ python3 re_negative_look_ahead.py

Candidate: first.last@example.com
  Match: first.last@example.com
Candidate: noreply@example.com
  No match
```

使用语法`(?<!pattern)`，在匹配用户名之后，使用负向后置断言来编写模式，而不是在电子邮件地址的用户名部分中前置搜索noreply。
>(?<!pattern) 只有当模式不能够被当前位置之前的字符匹配时才成功(模式必须是固定长度)，但并不消耗字符。 “零宽度负回顾后发断言” (?<!pattern)，来断言此位置的前面不能匹配表达式exp：(?<![a-z])\d{7}匹配前面不是小写字母的七位数字

```python
#re_negative_look_behind.py
import re

address = re.compile(
    '''
    ^

    # An address: username@domain.tld

    [\w\d.+-]+       # username

    # Ignore noreply addresses
    (?<!noreply)

    @
    ([\w\d.]+\.)+    # domain name prefix
    (com|org|edu)    # limit the allowed top-level domains

    $
    ''',
    re.VERBOSE)

candidates = [
    u'first.last@example.com',
    u'noreply@example.com',
]

for candidate in candidates:
    print('Candidate:', candidate)
    match = address.search(candidate)
    if match:
        print('  Match:', candidate[match.start():match.end()])
    else:
        print('  No match')
```
前置与后置略有不同，因为表达式必须使用固定长度的模式。允许重复，只要它们有固定数量（没有通配符或范围）。

```
$ python3 re_negative_look_behind.py

Candidate: first.last@example.com
  Match: first.last@example.com
Candidate: noreply@example.com
  No match
```

正向后置断言使用语法`(?<=pattern)`用于查找模式后面的文本。在以下示例中，表达式查找Twitter句柄。
>(?<=pattern) 只有当模式能够被当前位置之前的字符匹配时才成功(模式必须是固定长度)，但并不消耗字符。 (?<=exp)也叫“零宽度正回顾后发断言”，它断言自身出现的位置的前面能匹配表达式exp。比如(?<=\bre)\w+\b会匹配以re开头的单词的后半部分(除了re以外的部分)，例如在查找reading a book时，它匹配ading

```python
#re_look_behind.py
import re

twitter = re.compile(
    '''
    # A twitter handle: @username
    (?<=@)
    ([\w\d_]+)       # username
    ''',
    re.VERBOSE)

text = '''This text includes two Twitter handles.
One for @ThePSF, and one for the author, @doughellmann.
'''

print(text)
for match in twitter.findall(text):
    print('Handle:', match)
```
该模式匹配可构成Twitter句柄的字符序列，只要它们前面有@。

```
$ python3 re_look_behind.py

This text includes two Twitter handles.
One for @ThePSF, and one for the author, @doughellmann.

Handle: ThePSF
Handle: doughellmann
```

### Self-referencing Expressions

匹配值可以在表达式的后续部分中使用。例如，可以通过包括对这些组的反向引用来更新电子邮件示例以仅匹配由该人的名字和姓氏组成的地址。实现此目的的最简单方法是使用`\num`，通过ID号引用先前匹配的组。

```python
#re_refer_to_group.py
import re

address = re.compile(
    r'''

    # The regular name
    (\w+)               # first name
    \s+
    (([\w.]+)\s+)?      # optional middle name or initial
    (\w+)               # last name

    \s+

    <

    # The address: first_name.last_name@domain.tld
    (?P<email>
      \1               # first name
      \.
      \4               # last name
      @
      ([\w\d.]+\.)+    # domain name prefix
      (com|org|edu)    # limit the allowed top-level domains
    )

    >
    ''',
    re.VERBOSE | re.IGNORECASE)

candidates = [
    u'First Last <first.last@example.com>',
    u'Different Name <first.last@example.com>',
    u'First Middle Last <first.last@example.com>',
    u'First M. Last <first.last@example.com>',
]

for candidate in candidates:
    print('Candidate:', candidate)
    match = address.search(candidate)
    if match:
        print('  Match name :', match.group(1), match.group(4))
        print('  Match email:', match.group(5))
    else:
        print('  No match')
```
虽然语法很简单，但是通过数字ID创建反向引用有一些缺点。从实际角度来看，随着表达式的变化，必须再次对这些组进行计数，并且可能需要更新每个引用。另一个缺点是只能使用标准反向引用语法`\n`进行99次引用，因为如果ID号是三位数长，它将被解释为八进制字符值而不是组引用。当然，如果表达式中有超过99个组，则会出现更严重的维护挑战，而不仅仅是无法引用所有这些组。

```
$ python3 re_refer_to_group.py

Candidate: First Last <first.last@example.com>
  Match name : First Last
  Match email: first.last@example.com
Candidate: Different Name <first.last@example.com>
  No match
Candidate: First Middle Last <first.last@example.com>
  Match name : First Last
  Match email: first.last@example.com
Candidate: First M. Last <first.last@example.com>
  Match name : First Last
  Match email: first.last@example.com
```

Python的表达式解析器包含一个扩展，它使用`(?P=name)`来引用表达式中先前匹配的命名组的值。

```python
#re_refer_to_named_group.py
import re

address = re.compile(
    '''

    # The regular name
    (?P<first_name>\w+)
    \s+
    (([\w.]+)\s+)?      # optional middle name or initial
    (?P<last_name>\w+)

    \s+

    <

    # The address: first_name.last_name@domain.tld
    (?P<email>
      (?P=first_name)
      \.
      (?P=last_name)
      @
      ([\w\d.]+\.)+    # domain name prefix
      (com|org|edu)    # limit the allowed top-level domains
    )

    >
    ''',
    re.VERBOSE | re.IGNORECASE)

candidates = [
    u'First Last <first.last@example.com>',
    u'Different Name <first.last@example.com>',
    u'First Middle Last <first.last@example.com>',
    u'First M. Last <first.last@example.com>',
]

for candidate in candidates:
    print('Candidate:', candidate)
    match = address.search(candidate)
    if match:
        print('  Match name :', match.groupdict()['first_name'],
              end=' ')
        print(match.groupdict()['last_name'])
        print('  Match email:', match.groupdict()['email'])
    else:
        print('  No match')
```
地址表达式是使用IGNORECASE标志编译的，因为正确的名称通常是大写的，但电子邮件地址不是。

```
$ python3 re_refer_to_named_group.py

Candidate: First Last <first.last@example.com>
  Match name : First Last
  Match email: first.last@example.com
Candidate: Different Name <first.last@example.com>
  No match
Candidate: First Middle Last <first.last@example.com>
  Match name : First Last
  Match email: first.last@example.com
Candidate: First M. Last <first.last@example.com>
  Match name : First Last
  Match email: first.last@example.com
```
在表达式中使用反向引用的另一种机制，根据先前的组是否匹配来选择不同的模式。可以更正电子邮件模式，以便在存在名称时需要尖括号，如果只有电子邮件地址本身则不需要尖括号。测试组是否匹配的语法是`(?(id)yes-expression|no-expression)`，其中id是组名或编号，yes-expression是组具有值时使用的模式，否则使用no-expression模式。

```python
#re_id.py
import re

address = re.compile(
    '''
    ^

    # A name is made up of letters, and may include "."
    # for title abbreviations and middle initials.
    (?P<name>
       ([\w.]+\s+)*[\w.]+
     )?
    \s*

    # Email addresses are wrapped in angle brackets, but
    # only if a name is found.
    (?(name)
      # remainder wrapped in angle brackets because
      # there is a name
      (?P<brackets>(?=(<.*>$)))
      |
      # remainder does not include angle brackets without name
      (?=([^<].*[^>]$))
     )

    # Look for a bracket only if the look-ahead assertion
    # found both of them.
    (?(brackets)<|\s*)

    # The address itself: username@domain.tld
    (?P<email>
      [\w\d.+-]+       # username
      @
      ([\w\d.]+\.)+    # domain name prefix
      (com|org|edu)    # limit the allowed top-level domains
     )

    # Look for a bracket only if the look-ahead assertion
    # found both of them.
    (?(brackets)>|\s*)

    $
    ''',
    re.VERBOSE)

candidates = [
    u'First Last <first.last@example.com>',
    u'No Brackets first.last@example.com',
    u'Open Bracket <first.last@example.com',
    u'Close Bracket first.last@example.com>',
    u'no.brackets@example.com',
]

for candidate in candidates:
    print('Candidate:', candidate)
    match = address.search(candidate)
    if match:
        print('  Match name :', match.groupdict()['name'])
        print('  Match email:', match.groupdict()['email'])
    else:
        print('  No match')
```
此版本的电子邮件地址解析器使用两个测试。如果name组匹配，则前瞻断言需要两个尖括号并设置括号组。如果name不匹配，则断言要求文本的其余部分不包含尖括号。稍后，如果设置了brackets组，则实际模式匹配代码使用字面模式消耗输入中的括号;否则，它消耗任何空白。

```
$ python3 re_id.py

Candidate: First Last <first.last@example.com>
  Match name : First Last
  Match email: first.last@example.com
Candidate: No Brackets first.last@example.com
  No match
Candidate: Open Bracket <first.last@example.com
  No match
Candidate: Close Bracket first.last@example.com>
  No match
Candidate: no.brackets@example.com
  Match name : None
  Match email: no.brackets@example.com
```

### Modifying Strings with Patterns

除了搜索文本之外，re还支持使用正则表达式作为搜索机制来修改文​​本，并且替换可以引用模式中匹配的组作为替换文本的一部分。使用sub（）将所有出现的模式替换为另一个字符串。

```python
#re_sub.py
import re

bold = re.compile(r'\*{2}(.*?)\*{2}')

text = 'Make this **bold**.  This **too**.'

print('Text:', text)
print('Bold:', bold.sub(r'<b>\1</b>', text))
```
可以使用用于反向引用的`\num`语法插入“对模式匹配的文本”的引用.

```
$ python3 re_sub.py

Text: Make this **bold**.  This **too**.
Bold: Make this <b>bold</b>.  This <b>too</b>.
```
要在替换中使用命名组，请使用语法`\g<name>`。

```python
#re_sub_named_groups.py
import re

bold = re.compile(r'\*{2}(?P<bold_text>.*?)\*{2}')

text = 'Make this **bold**.  This **too**.'

print('Text:', text)
print('Bold:', bold.sub(r'<b>\g<bold_text></b>', text))
```
`\g<name>`语法也适用于数字编号引用，使用它可以消除组号和周围字面数字之间的任何歧义。

```
$ python3 re_sub_named_groups.py

Text: Make this **bold**.  This **too**.
Bold: Make this <b>bold</b>.  This <b>too</b>.
```
传递一个值给count以限制执行的替换次数。

```python
#re_sub_count.py
import re

bold = re.compile(r'\*{2}(.*?)\*{2}')

text = 'Make this **bold**.  This **too**.'

print('Text:', text)
print('Bold:', bold.sub(r'<b>\1</b>', text, count=1))
Only the first substitution is made because count is 1.

$ python3 re_sub_count.py

Text: Make this **bold**.  This **too**.
Bold: Make this <b>bold</b>.  This **too**.
```
subn（）就像sub（）一样工作，除了它返回修改后的字符串和所做的替换计数。

```python
#re_subn.py
import re

bold = re.compile(r'\*{2}(.*?)\*{2}')

text = 'Make this **bold**.  This **too**.'

print('Text:', text)
print('Bold:', bold.subn(r'<b>\1</b>', text))
```
搜索模式在示例中匹配两次。

```
$ python3 re_subn.py

Text: Make this **bold**.  This **too**.
Bold: ('Make this <b>bold</b>.  This <b>too</b>.', 2)
```

### Splitting with Patterns

str.split（）是拆分字符串以解析它们的最常用方法之一。但是，它仅支持使用字面值作为分隔符，如果输入的格式不一致，有时需要使用正则表达式。例如，许多纯文本标记语言将段落分隔符定义为两个或更多换行符（\n）。在这种情况下，由于定义的“或更多”部分，不能使用str.split（）。

使用findall（）识别段落的策略将使用类似`(.+?)\n{2}`的模式。

```python
#re_paragraphs_findall.py
import re

text = '''Paragraph one
on two lines.

Paragraph two.


Paragraph three.'''

for num, para in enumerate(re.findall(r'(.+?)\n{2,}',
                                      text,
                                      flags=re.DOTALL)
                           ):
    print(num, repr(para))
    print()
```
对于输入文本末尾的段落，该模式失败，正如“Paragraph three.”不是输出的一部分的事实所说明的那样。

```
$ python3 re_paragraphs_findall.py

0 'Paragraph one\non two lines.'

1 'Paragraph two.'
```
扩展模式以表示段落以两个或多个换行符结束或输入结束，修复了问题，但使模式更复杂。转换为re.split（）而不是re.findall（）会自动处理边界条件并使模式更简单。

```python
#re_split.py
import re

text = '''Paragraph one
on two lines.

Paragraph two.


Paragraph three.'''

print('With findall:')
for num, para in enumerate(re.findall(r'(.+?)(\n{2,}|$)',
                                      text,
                                      flags=re.DOTALL)):
    print(num, repr(para))
    print()

print()
print('With split:')
for num, para in enumerate(re.split(r'\n{2,}', text)):
    print(num, repr(para))
    print()
```
split（）的模式参数更精确地表示标记规范。两个或多个换行符标记输入字符串中段落之间的分隔符。

```
$ python3 re_split.py

With findall:
0 ('Paragraph one\non two lines.', '\n\n')

1 ('Paragraph two.', '\n\n\n')

2 ('Paragraph three.', '')


With split:
0 'Paragraph one\non two lines.'

1 'Paragraph two.'

2 'Paragraph three.'
```
将表达式括在括号中以定义组会导致split（）更像str.partition（），因此它返回分隔符值以及字符串的其他部分。

```python
#re_split_groups.py
import re

text = '''Paragraph one
on two lines.

Paragraph two.


Paragraph three.'''

print('With split:')
for num, para in enumerate(re.split(r'(\n{2,})', text)):
    print(num, repr(para))
    print()
```
输出现在包括每个段落，以及分隔它们的换行符序列。

```
$ python3 re_split_groups.py

With split:
0 'Paragraph one\non two lines.'

1 '\n\n'

2 'Paragraph two.'

3 '\n\n\n'

4 'Paragraph three.'
```

## difflib — Compare Sequences

目的：比较序列，尤其是文本行。  
difflib模块包含用于计算和处理序列之间差异的工具。它对于比较文本特别有用，并且包括使用几种常见差异格式生成报告的函数。  
本节中的示例将全部使用difflib_data.py模块中的测试数据：

```python
#difflib_data.py
text1 = """Lorem ipsum dolor sit amet, consectetuer adipiscing
elit. Integer eu lacus accumsan arcu fermentum euismod. Donec
pulvinar porttitor tellus. Aliquam venenatis. Donec facilisis
pharetra tortor.  In nec mauris eget magna consequat
convalis. Nam sed sem vitae odio pellentesque interdum. Sed
consequat viverra nisl. Suspendisse arcu metus, blandit quis,
rhoncus ac, pharetra eget, velit. Mauris urna. Morbi nonummy
molestie orci. Praesent nisi elit, fringilla ac, suscipit non,
tristique vel, mauris. Curabitur vel lorem id nisl porta
adipiscing. Suspendisse eu lectus. In nunc. Duis vulputate
tristique enim. Donec quis lectus a justo imperdiet tempus."""

text1_lines = text1.splitlines()

text2 = """Lorem ipsum dolor sit amet, consectetuer adipiscing
elit. Integer eu lacus accumsan arcu fermentum euismod. Donec
pulvinar, porttitor tellus. Aliquam venenatis. Donec facilisis
pharetra tortor. In nec mauris eget magna consequat
convalis. Nam cras vitae mi vitae odio pellentesque interdum. Sed
consequat viverra nisl. Suspendisse arcu metus, blandit quis,
rhoncus ac, pharetra eget, velit. Mauris urna. Morbi nonummy
molestie orci. Praesent nisi elit, fringilla ac, suscipit non,
tristique vel, mauris. Curabitur vel lorem id nisl porta
adipiscing. Duis vulputate tristique enim. Donec quis lectus a
justo imperdiet tempus.  Suspendisse eu lectus. In nunc."""

text2_lines = text2.splitlines()
```

### Comparing Bodies of Text

Differ类适用于文本行序列，并生成人类可读的增量（deltas），或更改指令，包括各行内的差异。Differ生成的默认输出类似于Unix下的diff命令行工具。它包括来自两个列表的原始输入值（包括公共值）和标记数据，以指示进行了哪些更改。

+ 以`-`为前缀的行在第一个序列中，但不在第二个序列中。
+ 以`+`为前缀的行在第二个序列中，但不在第一个序列中。
+ 如果某行的版本之间存在增量差异，则一个前缀为`?`的额外行 用于突出显示新版本中的更改。
+ 如果某行没有改变，则在左列上打印一个额外的空白区域，使其与可能存在差异的其他输出行对齐。

在将文本传递给compare（）之前将文本分解为一系列单独的行会产生比传入大字符串更可读的输出。

```python
#difflib_differ.py
import difflib
from difflib_data import *

d = difflib.Differ()
diff = d.compare(text1_lines, text2_lines)
print('\n'.join(diff))
```
样本数据中两个文本段的开头是相同的，因此第一行打印时没有任何额外的注释。

```
  Lorem ipsum dolor sit amet, consectetuer adipiscing
  elit. Integer eu lacus accumsan arcu fermentum euismod. Donec
```
数据的第三行已更改为在修改后的文本中包含逗号。该行的两个版本都打印出来，第5行的额外信息显示了文本被修改的列，包含添加了`,`字符的事实。

```
- pulvinar porttitor tellus. Aliquam venenatis. Donec facilisis
+ pulvinar, porttitor tellus. Aliquam venenatis. Donec facilisis
?         +
```
输出的以下几行显示删除了额外的空格。

```
- pharetra tortor.  In nec mauris eget magna consequat
?                 -

+ pharetra tortor. In nec mauris eget magna consequat
```
接下来，进行了更复杂的更改，替换了短语中的多个单词。

```
- convalis. Nam sed sem vitae odio pellentesque interdum. Sed
?                 - --

+ convalis. Nam cras vitae mi vitae odio pellentesque interdum. Sed
?               +++ +++++   +
```
段落中的最后一句被显著改变，因此通过删除旧版本并添加新版本来表示差异。

```
  consequat viverra nisl. Suspendisse arcu metus, blandit quis,
  rhoncus ac, pharetra eget, velit. Mauris urna. Morbi nonummy
  molestie orci. Praesent nisi elit, fringilla ac, suscipit non,
  tristique vel, mauris. Curabitur vel lorem id nisl porta
- adipiscing. Suspendisse eu lectus. In nunc. Duis vulputate
- tristique enim. Donec quis lectus a justo imperdiet tempus.
+ adipiscing. Duis vulputate tristique enim. Donec quis lectus a
+ justo imperdiet tempus.  Suspendisse eu lectus. In nunc.
```
ndiff（）函数产生基本相同的输出。该处理专门用于处理文本数据并消除输入中的“噪声”。

#### Other Output Formats

虽然Differ类显示所有输入行，但统一的diff(unified diff)仅包含修改的行和一些上下文。unified_diff（）函数产生这种输出。

```python
#difflib_unified.py
import difflib
from difflib_data import *

diff = difflib.unified_diff(
    text1_lines,
    text2_lines,
    lineterm='',
)
print('\n'.join(diff))
```
lineterm参数用于告诉unified_diff（）跳过额外的换行符到它返回的控制行，因为输入行不包含它们。打印时，换行符将添加到所有行。对于许多流行的版本控制工具的用户来说，输出看起来应该很熟悉。

```
$ python3 difflib_unified.py

---
+++
@@ -1,11 +1,11 @@
 Lorem ipsum dolor sit amet, consectetuer adipiscing
 elit. Integer eu lacus accumsan arcu fermentum euismod. Donec
-pulvinar porttitor tellus. Aliquam venenatis. Donec facilisis
-pharetra tortor.  In nec mauris eget magna consequat
-convalis. Nam sed sem vitae odio pellentesque interdum. Sed
+pulvinar, porttitor tellus. Aliquam venenatis. Donec facilisis
+pharetra tortor. In nec mauris eget magna consequat
+convalis. Nam cras vitae mi vitae odio pellentesque interdum. Sed
 consequat viverra nisl. Suspendisse arcu metus, blandit quis,
 rhoncus ac, pharetra eget, velit. Mauris urna. Morbi nonummy
 molestie orci. Praesent nisi elit, fringilla ac, suscipit non,
 tristique vel, mauris. Curabitur vel lorem id nisl porta
-adipiscing. Suspendisse eu lectus. In nunc. Duis vulputate
-tristique enim. Donec quis lectus a justo imperdiet tempus.
+adipiscing. Duis vulputate tristique enim. Donec quis lectus a
+justo imperdiet tempus.  Suspendisse eu lectus. In nunc.
```
使用context_diff（）产生类似的可读输出。

### Junk Data

生成差异序列的所有函数都接受参数，以指示应忽略哪些行以及应忽略行中的哪些字符。例如，这些参数可用于跳过文件的两个版本中的标记或空格更改。

```python
#difflib_junk.py

#This example is adapted from the source for difflib.py.

from difflib import SequenceMatcher

def show_results(match):
    print('  a    = {}'.format(match.a))
    print('  b    = {}'.format(match.b))
    print('  size = {}'.format(match.size))
    i, j, k = match
    print('  A[a:a+size] = {!r}'.format(A[i:i + k]))
    print('  B[b:b+size] = {!r}'.format(B[j:j + k]))

A = " abcd"
B = "abcd abcd"

print('A = {!r}'.format(A))
print('B = {!r}'.format(B))

print('\nWithout junk detection:')
s1 = SequenceMatcher(None, A, B)
match1 = s1.find_longest_match(0, len(A), 0, len(B))
show_results(match1)

print('\nTreat spaces as junk:')
s2 = SequenceMatcher(lambda x: x == " ", A, B)
match2 = s2.find_longest_match(0, len(A), 0, len(B))
show_results(match2)
```
Differ的默认不显式忽略任何行或字符，而是依赖于SequenceMatcher检测噪声的能力。ndiff()的默认忽略空格和制表符。

```
$ python3 difflib_junk.py

A = ' abcd'
B = 'abcd abcd'

Without junk detection:
  a    = 0
  b    = 4
  size = 5
  A[a:a+size] = ' abcd'
  B[b:b+size] = ' abcd'

Treat spaces as junk:
  a    = 1
  b    = 0
  size = 4
  A[a:a+size] = 'abcd'
  B[b:b+size] = 'abcd'
```

### Comparing Arbitrary Types

SequenceMatcher类比较任何类型的两个序列，只要值是可散列（hashable）的。它使用一种算法来识别序列中最长的连续匹配块，消除了对真实数据没有贡献的“垃圾”值。  
函数get_opcodes（）返回一个指令列表，用于修改第一个序列以使其与第二个序列匹配。指令被编码为五元素元组，包括字符串指令（“操作码”，参见下表）和两对开始和停止索引到序列中（表示为i1，i2，j1和j2）。

difflib.get_opcodes() Instructions

Opcode	  | Definition
:---      | :---
'replace'	| Replace a[i1:i2] with b[j1:j2]
'delete'	| Remove a[i1:i2] entirely
'insert'	| Insert b[j1:j2] at a[i1:i1]
'equal'	  | The subsequences are already equal

```python
#difflib_seq.py
import difflib

s1 = [1, 2, 3, 5, 6, 4]
s2 = [2, 3, 5, 4, 6, 1]

print('Initial data:')
print('s1 =', s1)
print('s2 =', s2)
print('s1 == s2:', s1 == s2)
print()

matcher = difflib.SequenceMatcher(None, s1, s2)
for tag, i1, i2, j1, j2 in reversed(matcher.get_opcodes()):

    if tag == 'delete':
        print('Remove {} from positions [{}:{}]'.format(
            s1[i1:i2], i1, i2))
        print('  before =', s1)
        del s1[i1:i2]

    elif tag == 'equal':
        print('s1[{}:{}] and s2[{}:{}] are the same'.format(
            i1, i2, j1, j2))

    elif tag == 'insert':
        print('Insert {} from s2[{}:{}] into s1 at {}'.format(
            s2[j1:j2], j1, j2, i1))
        print('  before =', s1)
        s1[i1:i2] = s2[j1:j2]

    elif tag == 'replace':
        print(('Replace {} from s1[{}:{}] '
               'with {} from s2[{}:{}]').format(
                   s1[i1:i2], i1, i2, s2[j1:j2], j1, j2))
        print('  before =', s1)
        s1[i1:i2] = s2[j1:j2]

    print('   after =', s1, '\n')

print('s1 == s2:', s1 == s2)
```
此示例比较两个整数列表，并使用get_opcodes（）来派生将原始列表转换为较新版本的指令。以相反的顺序应用修改，以便在添加和删除项目后列表索引保持准确。

```
$ python3 difflib_seq.py

Initial data:
s1 = [1, 2, 3, 5, 6, 4]
s2 = [2, 3, 5, 4, 6, 1]
s1 == s2: False

Replace [4] from s1[5:6] with [1] from s2[5:6]
  before = [1, 2, 3, 5, 6, 4]
   after = [1, 2, 3, 5, 6, 1]

s1[4:5] and s2[4:5] are the same
   after = [1, 2, 3, 5, 6, 1]

Insert [4] from s2[3:4] into s1 at 4
  before = [1, 2, 3, 5, 6, 1]
   after = [1, 2, 3, 5, 4, 6, 1]

s1[1:4] and s2[0:3] are the same
   after = [1, 2, 3, 5, 4, 6, 1]

Remove [1] from positions [0:1]
  before = [1, 2, 3, 5, 4, 6, 1]
   after = [2, 3, 5, 4, 6, 1]

s1 == s2: True
```
SequenceMatcher适用于自定义类以及内置类型，只要它们是可散列的。
