---
title: Autotools 总结
description: GNU构建系统Autotools总结
categories: tools
tags:
  - GNU
  - autotool
---

# GNU 构建系统

## The User Point of View

### 标准安装过程

```shell
~ % tar zxf amhello-1.0.tar.gz
~ % cd amhello-1.0
~/amhello-1.0 % ./configure
~/amhello-1.0 % make
~/amhello-1.0 % make check
~/amhello-1.0 % su
Password:
/home/adl/amhello-1.0 # make install
/home/adl/amhello-1.0 # exit
~/amhello-1.0 % make installcheck
```

### 标准Makefile 目标

目标 | 描述
:--- | :---
make all | 构建程序，库，文档等等（与make同）
make install | 安装需要安装的东西
make install-strip | 与make install 相同，并且去掉调试符号
make uninstall | make install 反过程
make clean | 删除构建的东西（make all 反过程）
make distclean | 另外删除由 ./configure 创建的东西
make check | 运行测试用例，如果有的话
make installcheck | 测试安装的程序或库，如果支持
make dist | 构建包PACKAGE-VERSION.tar.gz
make diskcheck | 带有完整性检查，倾向于使用这个

### 标准文件层次

目录变量 | 默认值
prefix | /usr/local
exec-prefix | prefix
bindir | exec-prefix/bin
libdir | exec-prefix/lib
includedir | prefix/include 
datarootdir | prefix/share
datadir | datarootdir
mandir | datarootdir/man
infodir | datarootdir/info

### 标准配置变量

变量 | 描述
CC | C 编译器
CFLAGS | C 编译器选项
CXX | C++编译器
CXXFLAGS | C++编译器选项
LDFLAGS | 链接选项
CPPFLAGS | C/C++ 预处理选项

`./configure --help`查看完整列表

## The Power User Point of View

### 用 `config.site` 覆盖默认设置

一般的配置选项可以放在 `prefix/share/config.site` 文件中，例如：

```shell
~/amhello-1.0 % cat ~/usr/share/config.site
test -z "$CC" && CC=gcc-3
test -z "$CPPFLAGS" && CPPFLAGS=-I$HOME/usr/include
test -z "$LDFLAGS" && LDFLAGS=-L$HOME/usr/lib

~/amhello-1.0 % ./configure --prefix ~/usr
configure: loading site script /home/adl/usr/share/config.site
```

### 平行构建树

目标文件、程序、库被构建在 `confiure` 命令运行的目录

```shell
~ % tar zxf ~/amhello-1.0.tar.gz
~ % cd amhello-1.0
~/amhello-1.0 % mkdir build && cd build
~/amhello-1.0/build % ../configure
~/amhello-1.0/build % make
```
Source files are in ~/ amhello-1.0/ ,  
built files are all in ~/ amhello-1.0/ build/

### 多架构下的平行构建树

多架构下构建可以共享源码树

```shell
#Have the source on a (possibly read-only) shared directory
~ % cd /nfs/src
/nfs/src % tar zxf ~/amhello-1.0.tar.gz
```
```shell
#Compilation on first host
~ % mkdir /tmp/amh && cd /tmp/amh
/tmp/amh % /nfs/src/amhello-1.0/configure
/tmp/amh % make && sudo make install
```
```shell
#Compilation on second host, assuming shared data
~ % mkdir /tmp/amh && cd /tmp/amh
/tmp/amh % /nfs/src/amhello-1.0/configure
/tmp/amh % make && sudo make install-exec
```

### 两部分安装

make install  
      =  
make install-exec  安装平台相关文件  
      +  
make install-data  安装平台无关文件（可以被多台机器共享）  

### 交叉编译

`./configure --build i686-pc-linux-gnu  --host i586-mingw32msvc`  
当然首先要安装交叉编译器  
交叉编译配置选项：  
`--build=BUILD` 构建包的系统环境  
`--host=HOST` 构建的程序或者哭运行的系统环境  
`--target=TARGET` 只有当构建编译工具时配置： 工具将为这个系统环境创建输出  
简单的交叉编译， 只需要`--host=HOST`

### 安装时重命名程序

`--program-prefix=PREFIX` 在程序名前加前缀 PREFIX  
`--program-suffix=SUFFIX` 在程序名前加后缀 SUFFIX  
`--program-transform-name=PROGRAMz` 在程序名上运行 `sed PROGRAM`  

```shell
~/amhello-1.0 % ./configure --program-prefix test-
~/amhello-1.0 % make
~/amhello-1.0 % sudo make install
Will install hello as /usr/local/bin/test-hello.
```

## The Packager Point of View

### 使用 `DESTDIR` 构建二进制包

`DESTDIR` 用于在安装时重定向包  

```shell
~/amhello-1.0 % ./configure --prefix /usr
~/amhello-1.0 % make
~/amhello-1.0 % make DESTDIR=$HOME/inst install
~/amhello-1.0 % cd ~/inst
~/inst % tar zcvf ~/amhello-1.0-i686.tar.gz .
./
./usr/
./usr/bin/
./usr/bin/hello
```

## The Maintainer Point of View

### 准备发布

使用 `make dist`,  `make distcheck` 命令打包。
`make distcheck` 确保迄今为止呈现的大部分用例都能正常工作:

* 测试VPATH构建（只读源码树）
* 确保`make clean`, `make distclean`, `make uninstall` 不会忽略文件
* 检查`DESTDIR`安装工作
* 运行测试用例（包含`make check`, `make installcheck`）

### 依赖检查

编译时依赖跟踪只在源文件改变时需要，在一次性安装构建中可以安全的禁用，慢方法必须明确启用。  
`--disable-dependency-tracking` 快速的一次性构建  
`--enable-dependency-tracking` 不要拒绝慢依赖提取器

### 嵌套的包

+ 自动配置的包可以任意深度的嵌套
	+ 一个包可以以子目录的方式发布一个它使用的第三方库
	+ 可以用这种方法集合很多包发布一系列工具
+ 对于安装人员
	+ 只要对一个单一的包执行configure, build, install
	+ `configure` 选项会递归的传递给子包
	+ `configure --help=recursive` 显示所有关于所有子包的帮助信息
+ 对于维护者
	+ 简单的集成
	+ 子包是自动的

## 配置过程

+ `*.in` 文件是配置模板， 通过`configure` 生成构建过程需要的配置文件。  
+ `config.log`   包含对配置过程的跟踪
+ `configure -C` 缓存结果到`config.cache`以加速重构建。
![configure_process.png](/assets/images/configure_process.png)

# GNU Autotools

## Hello World

### src/main.c

```c
// src/main.c
#include <config.h>
#include <stdio.h>

int main(void)
{
	puts("Hello World!");
	puts("This is PACKAGE_STRING");
	return 0;
}
```

### 生成模板文件

![gen-template-files.png](/assets/images/generating-template-files.png)

### Autotools Inputs

```shell
#configure.ac
AC_INIT([amhello], [1.0], [bug-report@address])
AM_INIT_AUTOMAKE([foreign -Wall -Werror])
AC_PROG_CC
AC_CONFIG_HEADERS([config.h])
AC_CONFIG_FILES([Makefile src/Makefile])
AC_OUTPUT
```
```shell
#Makefile.am
SUBDIRS = src
```
```shell
#src/Makefile.am
bin_PROGRAMS = hello
hello_SOURCES = main.c
```

### 准备包

```shell
~/amhello % ls -R
.:
Makefile.am configure.ac src/
./src:
Makefile.am main.c
```
```shell
~/amhello % autoreconf --install
configure.ac:2: installing './install-sh'
configure.ac:2: installing './missing'
src/Makefile.am: installing './depcomp'
```
```shell
~/amhello % ls -R
.:
Makefile.am configure.ac
Makefile.in depcomp*
aclocal.m4 install-sh*
autom4te.cache/ missing*
config.h.in src/
configure*
./autom4te.cache:
output.0 requests traces.1
output.1 traces.0
./src:
Makefile.am Makefile.in main.c
```

+ `Makefile.in`,`config.h.in`,`configure*`,`./src/Makefile.in` --- 期望的配置模板
+ `aclocal.m4` --- `configure.ac`中使用的第三方宏定义
+ `depcomp*`,`install-sh*`,`missing*` --- 构建过程使用的辅助工具
+ `autom4te.cache/`,`output.0`,`requests`,`traces.1`,`output.1`,`traces.0` --- Autotools缓存文件

```shell
~/amhello % ./configure
checking for a BSD-compatible install... /usr/bin/install -c
checking whether build environment is sane... yes
checking for gawk... gawk
checking whether make sets $(MAKE)... yes
checking for gcc... gcc
...
checking dependency style of gcc... gcc3
#生成的文件
configure: creating './config.status'
config.status: creating 'Makefile'
config.status: creating 'src/Makefile'
config.status: creating 'config.h'
config.status: executing depfiles commands
```
```shell
~/amhello % make
~/amhello % src/hello
Hello World!
This is amhello 1.0.
```
```shell
~/amhello % make distcheck
...
========================================
amhello archives ready for distribution:
amhello-1.0.tar.gz
========================================
~/amhello %

~/amhello % tar ztf amhello-1.0.tar.gz
amhello-1.0/
amhello-1.0/Makefile.am
amhello-1.0/Makefile.in
amhello-1.0/aclocal.m4
amhello-1.0/config.h.in
amhello-1.0/configure
amhello-1.0/configure.ac
amhello-1.0/depcomp
amhello-1.0/install-sh
amhello-1.0/missing
amhello-1.0/src/
amhello-1.0/src/Makefile.am
amhello-1.0/src/Makefile.in
amhello-1.0/src/main.c
~/amhello %
```

## 介绍核心Autotools

### 两个核心包

#### GNU Autoconf

工具 | 描述
:--- | :---
autoconf | 从`configure.ac` 生成 `configure`
autoheader | 从`configure.ac` 生成 `config.h.in`
autoreconf | 按正确的顺序运行所有工具
autoscan | 扫描源代码检查可移植性问题以及`configure.ac`中相关宏。
autoupdate | 更新`configure.ac`中过时的宏
ifnames | 从所有的`#if` `#ifdef`指令收集标识符 
autom4te | `Autoconf`的心脏。它驱动M4并且实现以上大部分工具的特性。不仅仅用于创建`configure`文件。

#### GUN Automake

工具 | 描述
:--- | :---
automake | 用`Makefile.am`(s) 和`configure.ac` 生成 `Makefile.in`(s)
aclocal | 扫描`configure.ac`以获取第三方宏的使用，以及收集定义到`aclocal.m4`中。

![behind-autoreconf](/assets/images/behind-autoreconf.png)
事实上，不需要记住所有这些工具的使用。一般使用`autoreconf --install` 来完成包的初始化。当你改变了一些输入文件，依赖Makefiles中的重建规则来重新运行相关的autotool。你只需要对每个工具的作用有个大概的印象以理解发生错误。

## Hello World Explained

### configure.ac 解释

```shell
#configure.ac
AC_INIT([amhello], [1.0], [bug-report@address])
AM_INIT_AUTOMAKE([foreign -Wall -Werror])
AC_PROG_CC
AC_CONFIG_HEADERS([config.h])
AC_CONFIG_FILES([Makefile src/Makefile])
AC_OUTPUT
```
* 初始化`Autoconf`。 指定包名， 版本号， 和bug报告地址
* 初始化`Automake`。 开启所有的`Automake`警告并报告为错误。这是一个`foreign`包，`foreign`可以忽略一些GNU 编码标准
* 检查 C 编译器
* 声明 `config.h` 作为输出头文件
* 声明 `Makefile`, `src/Makefile` 作为输出文件
* 实际输出所有声明文件

`foreign` 可以忽略一些GNU 编码标准：

```shell
#configure.ac without the foreign option
...
AM_INIT_AUTOMAKE([ -Wall -Werror])
...
~/amhello % autoreconf --install
configure.ac:2: installing './install-sh'
configure.ac:2: installing './missing'
src/Makefile.am: installing './depcomp'
Makefile.am: installing './INSTALL'
Makefile.am: required file './NEWS' not found
Makefile.am: required file './README' not found
Makefile.am: required file './AUTHORS' not found
Makefile.am: required file './ChangeLog' not found
Makefile.am: installing './COPYING'
autoreconf: automake failed with exit status: 1
```

### Makefile.am 解释

```shell
#Makefile.am
SUBDIRS = src
```
在 `src/` 目录递归构建。顶层的Makefile.am 通常是简短的。

```shell
#src/Makefile.am
bin_PROGRAMS = hello
hello_SOURCES = main.c
```
构建的程序将会被安装在 bindir目录。  
只有一个程序需要构建： hello  
构建hello， 只需要编译main.c

## 使用Autoconf

### 从configure.ac 到 configure 和 config.h.in

+ `autocon`f 是一个宏处理器
+ 它转化`configure.ac`(使用宏指令的shell脚本)到`configure`(使用成熟的shell脚本)
+ `autoconf` 提供一些宏来执行一般的配置检查
+ `configure.ac` 只包含宏，没有`shell`结构并不罕见
+ 处理`configure.ac` 时可以跟踪宏的出现。这也是`autoheader`创建`config.h.in` 的方法。
+ 真正的宏处理器是`GNU M4`.  `autoconf` 在它之上提供一些框架，和宏池。

### M4

```shell
#example.m4
m4_define(NAME1, Harry)
m4_define(NAME2, Sally)
m4_define(MET, $1 met $2)
MET(NAME1, NAME2)
~ % m4 -P example.m4
Harry met Sally
```

+ 宏的参数被处理
+ 然后宏被处理
+ 最后宏的输出也被处理
+ 一个字符串可以使用 引号 保护

下面是一个粗心导致的错误：

```shell
#example.m4
m4_define(NAME1, 'Harry, Jr.')
m4_define(NAME2, Sally)
m4_define(MET, $1 met $2)
MET(NAME1, NAME2)
```
```shell
m4 -P example.m4
MET(Harry, Jr., Sally) # 中间过程
Harry met Jr. #最后结果
```

#### M4引用规则

+ 引用每一个宏参数一次
+ 因此它只在它被输出后处理

```shell
m4_define('NAME1', 'Harry, Jr.')
m4_define('NAME2', 'Sally')
m4_define('MET', '$1 met $2')
MET('NAME1', 'NAME2')
```
```shell
m4 -P example.m4
NAME1 met NAME2 # 中间步骤
Harry, Jr. met Sally # 最后结果
```

#### 空格问题

为了方便说明，用`@`代替空格

+ 括号必须粘着宏名

```shell
#example.m4
m4_define('NAME1', 'Harry, Jr.')
m4_define('NAME2', 'Sally')
m4_define('MET', '$1 met $2')
MET@('NAME1', 'NAME2')

~ % m4 -P example.m4
met@(NAME1, NAME2)
```
+ 引号后面或者里面的空格是参数的一部分

```shell
#example.m4
m4_define('NAME1', 'Harry, Jr.')
m4_define('NAME2', 'Sally')
m4_define('MET', '$1 met $2')
MET('@NAME1@'@, 'NAME2')

~ % m4 -P example.m4
@Harry, Jr.@@ met Sally
```
+ 引号前面的空格被忽略

```shell
#example.m4
m4_define('NAME1',@'Harry, Jr.')
m4_define('NAME2',@'Sally')
m4_define('MET',@'$1 met $2')
MET( 'NAME1',@'NAME2')

~ % m4 -P example.m4
Harry, Jr. met Sally
```

### Autoconf 在 M4 顶层

+ Autoconf 是更加机械的M4， 且有许多的预定义宏
+ 引用使用`[ ]` 代替 `' '`
+ 因为以上原因，在shell中使用test命令代替`[]`

```shell
#if [ "$x" = "$y" ]; then ...
if test "$x" = "$y"; then ...
```
+ 使用AC_DEFUN 定义宏

```shell
AC_DEFUN([NAME1], [Harry, Jr.])
AC_DEFUN([NAME2], [Sally])
AC_DEFUN([MET], [$1 met $2])
MET([NAME1], [NAME2])
```

### configure.ac 结构

```
configure.ac
# Prelude.
AC_INIT([amhello], [1.0], [bug-report@address])
AM_INIT_AUTOMAKE([foreign -Wall -Werror])
# Checks for programs.
AC_PROG_CC
# Checks for libraries.
# Checks for header files.
# Checks for typedefs, structures, and compiler characteristics.
# Checks for library functions.
# Output files.
AC_CONFIG_HEADERS([config.h])
AC_CONFIG_FILES([Makefile src/Makefile])
AC_OUTPUT
```

### prelude 使用的Autoconf宏

宏 | 解释
:--- |:---
`AC_INIT(PACKAGE, VERSION, BUG-REPORT-ADDRESS)` | 强制Autoconf初始化
`AC_PREREQ(VERSION)` | 要求的最低Autoconf版本E.g. `AC_PREREQ([2.65])`
`AC_CONFIG_SRCDIR(FILE)` | 安全检查。FILE是一个将被发布的源文件,并且确保`configure`不会在外部空间运行。E.g. `AC_CONFIG_SRCDIR([src/main.c])`
`AC_CONFIG_AUX_DIR(DIRECTORY)` | 辅助脚本，例如`install-sh`, `depcomp` 应该被放在`DIRECTORY`

### 有用的程序检查

+ `AC_PROG_CC`, `AC_PROG_CXX`, `AC_PROG_F77`,...  
编译器检查  
+ `AC_PROG_SED`, `AC_PROG_YACC`, `AC_PROG_LEX`,...  
寻找好的实现， 并且设置`$SED`, `$YACC`, `$LEX`,...  
+ `AC_CHECK_PROGS(VAR, PROGS, [VAL-IF-NOT-FOUND])`   
使用第一个找到的`PROGS` 定义`VAR`， 否则使用 `VAL-IF-NOT-FOUND`.

```shell
AC_CHECK_PROGS([TAR], [tar gtar], [:])
if test "$TAR" = :; then
AC_MSG_ERROR([This package needs tar.])
fi
```

### 有用的Autoconf行为宏

+ `AC_MSG_ERROR(ERROR-DESCRIPTION, [EXIT-STATUS])`  
打印 ERROR-DESCRIPTION(也会写到config.log)且终止`configure`  
+ `AC_MSG_WARN(ERROR-DESCRIPTION)`  
与上相同， 但不终止  
+ `AC_DEFINE(VARIABLE, VALUE, DESCRIPTION)`  
输出下面的内容到config.h

```shell
/* DESCRIPTION */
#define VARIABLE VALUE
```
+ `AC_SUBST(VARIABLE, [VALUE])`  
在Makefile中定义`$(VARIABLE)`为`VALUE`

### 检查库

+ `AC_CHECK_LIB(LIBRARY, FUNCT, [ACT-IF-FOUND], [ACT-IF-NOT])`  
检查是否存在库`LIBRARY`并且包含`FUNCT`， 如果是执行`ACT-IF-FOUND`, 否则执行 `ACT-IF-NOT`  
例如：  
`AC_CHECK_LIB([efence], [malloc], [EFENCELIB=-lefence])`  
`AC_SUBST([EFENCELIB])`  
将在后面的链接规则中使用$(EFENCELIB)  
如果 `ACT-IF-FOUND `没有设置 并且库存在， `AC_CHECK_LIB` 将执行`LIBS="-lLIBRARY $LIBS"` 和 `#define HAVE_LIB(LIBRARY)`
(Automake 使用$LIBS 链接一切)

### 检查头文件

+ `AC_CHECK_HEADERS(HEADERS...)`  
检查`HEADERS` 并且为每一个找到的头文件执行`#define HAVE_(HEADER)_H`  
例如：  
`AC_CHECK_HEADERS([sys/param.h unistd.h])`  
`AC_CHECK_HEADERS([wchar.h])`  
可能定义 `HAVE_SYS_PARAM_H`, `HAVE_UNISTD_H`, `HAVE_WCHAR_H`.
+ `AC_CHECK_HEADER(HEADER, [ACT-IF-FOUND], [ACT-IF-NOT])`  
只检查一个头文件。

### 输出命令

+ `AC_CONFIG_HEADERS(HEADERS...)`  
为所有的`HEADER.in` 创建 `HEADER`. 只使用一个这样的头文件，除非你知道自己在做什么。（`autoheader` 只为第一个HEADER创建HEADER.in）  
HEADERS 包含AC_DEFINE 的定义  
例如：  
	+ `AC_CONFIG_HEADERS([config.h])`  
	从config.h.in 创建config.h  
	+ `AC_CONFIG_HEADERS([config.h:config.hin])`（DJGPP只支持一个点）  
	从config.hin 创建config.h  
+ `AC_CONFIG_FILES(FILES...)`  
从所有的FILE.in创建FILE  
FILES 包含AC_SUBST的定义  
例如：  
`AC_CONFIG_FILES([Makefile sub/Makefile script.sh:script.in])`  
Automake 为每一个有FILES.am的FILE创建FILE.in。  

```shell
script.in
#!/bin/sh
SED='@SED@'
TAR='@TAR@'
d=$1; shift; mkdir "$d"
for f; do
"$SED" 's/#.*//' "$f" \
>"$d/$f"
done
"$TAR" cf "$d.tar" "$d"
```
```shell
script.sh
#!/bin/sh
SED='/usr/xpg4/bin/sed'
TAR='/usr/bin/tar'
d=$1; shift; mkdir "$d"
for f; do
"$SED" 's/#.*//' "$f" \
>"$d/$f"
done
"$TAR" cf "$d.tar" "$d"
```
`.in` 模板文件中 `@XYZ@`是 `AC_SUBST([XYZ])`定义的占位符，`config.status` 替换他们  
`Makefile.in` 也使用`@XYZ@` 作为替换符，但是Automake执行所有的 `XYZ=@XYZ@` 定义， 你可以简单的使用 $(XYZ)。

## 使用Automake

### Automake 原则
+ Automake帮助创建可移植和兼容GNU标准的Makefile
	+ 你可能使用其他种类的构建系统
	+ 不要使用Automake如果你不喜欢GNU构建系统。如果你不适应它的模具，Automake会阻止你。
+ automake 从简单的Makefile.am文件创建复杂的Makefile.in文件
	+ 把Makefile.in当成内部细节
+ Makefile.am 遵循与Makefile大致相同的语法， 但通常只包含变量的定义
	+ automake 从这些定义中生成 构建规则
	+ 可以在Makefile.am中添加额外的Makefile 规则， automake会在输出中保留他们。

### 在configure.ac中声明Automake

`AM_INIT_AUTOMAKE([OPTIONS...])`  
检查 `automake`生成的`Makfiles` 需要的工具  
有用的选项：

选项 | 描述
:--- | :---
-Wall | 开启警告
-Werror | 警告作为错误报告
foreign | 放松一些GNU 标准的规定
1.11.1 | 要求一个最低automake版本
dist-bzip2 | make dist, make distcheck 时也会创建 tar.bz2 归档
tar-ustar | 使用ustar格式创建tar归档

### 声明目标惯例用法

```shell
#Makefile.am
option_where_PRIMARY = targets ...
```

+ `where`: 目标安装位置：  
	+ `bin_` ： $(bindir)
	+ `lib_` ： $(libdir)
	+ `custom_` ： $(customdir ) （自定义目录）  
	+ `noinst_` ： 不会被安装.  
	+ `check_` ： 由`make check`构建.
+ `PRIMARY`: 目标被构建为：  
	+ _PROGRAMS  
	+ _LIBRARIES  
	+ _LTLIBRARIES (Libtool libraries)  
	+ _HEADERS  
	+ _SCRIPTS  
	+ _DATA  
+ `option` Optionally: 
	+ dist_ 发布的目标 (if not the default)
	+ nodist_ 不发布.

```shell
#Makefile.am
bin_PROGRAMS = foo run-me
foo_SOURCES = foo.c foo.h print.c print.h
run_me_SOURCES = run.c run.h print.c
```

+ 这些程序将被安装在`$(bindir)`
+ 每个程序的源文件放在 `program_SOURCES`
+ 非字母数字字符被映射成 `_`
+ `Automake` 自动从这些文件计算需要构建和链接的目标文件列表
+ 头文件不会被编译。 列出他们只是让他们被发布（`Automake`不会发布它不知道的文件）
+ 在两个程序中使用相同的文件是允许的
+ 根据扩展名推导编译器和链接器。

### （静态）库
+ 添加 `AC_PROG_RANLIB` 到`configure.ac`

```shell
#Makefile.am
lib_LIBRARIES = libfoo.a libbar.a
libfoo_a_SOURCES = foo.c privfoo.h
libbar_a_SOURCES = bar.c privbar.h
include_HEADERS = foo.h bar.h
```
+ 这些库将被安装在 `$(libdir)`
+ 库名必须匹配 `lib*.a`
+ 公有头文件被安装在`$(includedir)`
+ 私有头文件不会被安装，就像普通源文件一样

### 目录布局
+ 你应该在每个目录有一个Makefile文件（也就是一个Makefile.am）
+ 所有Makefile都必须声明在configure.ac

```shell
#configure.ac
AC_CONFIG_FILES([Makefile lib/Makefile src/Makefile src/dira/Makefile src/dirb/Makefile])
```

+ make 运行在顶层目录
+ Makefile.am 使用SUBDIRS变量按顺序递归。

```shell
#Makefilele.am
SUBDIRS = lib src
```
```shell
#src/Makefilele.am
SUBDIRS = . dira dirb
```
+ 当前目录被默认构建， 可以使用`.`覆盖

### (srcdir) 和 VPATH构建

+ VPATH构建： 源文件不必在当前目录
+ 有两个双胞胎树： 构建树 和 源码树
	+ `Makefile` 和 `objects` 在构建树
	+ `Makefile.in`、`Makefile.am` 和源文件在源码树
	+ 如果`configure` 运行在当前目录， 这两个树是同一个
+ 在每一个`Makefile`， `config.status` 会定义 `$(srcdir)`： 指向源代码目录
+ 当在`Automake` 变量中引用源文件或目标时，`make`在会`source`和`build`两个目录查找。
+ 为工具指定标志，或者写自定义命令时，你可能需要 `$(srcdir)`。 
例如:    
告诉编译器从 `dir/` 包含头文件， 你应该写 `-I$(srcdir)/dir`, 而不是 `-Idir`.(`-Idir` 将从构建树拉取头文件)。

### Convenience Libraries

```shell
#lib/Makefile.am
noinst_LIBRARIES = libcompat.a
libcompat_a_SOURCES = xalloc.c xalloc.h
```

这是一个快捷库，只用于包的构建。

```shell
#src/Makefile.am
LDADD = ../lib/libcompat.a
AM_CPPFLAGS = -I$(srcdir)/../lib
bin_PROGRAMS = foo run-me
foo_SOURCES = foo.c foo.h print.c print.h
run_me_SOURCES = run.c run.h print.c
run_me_LDADD = ../lib/libcompat.a
run_me_CPPFLAGS = -I$(srcdir)/../lib
```
+ LDADD 用于链接所有程序
+ AM_CPPFLAGS 包含额外的预处理标志
+ 可以使用每目标变量：应用于单个程序

### 每目标标志

假设foo是一个程序或者库  

标志 | 描述
:--- | :---
foo_CFLAGS | 额外C编译器标志
foo_CPPFLAGS | 额外预处理标志（-Is, -Ds）
foo_LDADD | 额外链接目标文件， -ls， -Ls（如果foo是一个程序）
foo_LIBADD| 额外链接目标文件，-ls，-LS（如果foo是一个库）
foo_LDFLAGS|额外链接标志

`foo_XXXFLAGS` 的默认值是$(`AM_XXXFLAGS`),  
使用普通的名字引用包里的库（使用-ls, -Ls 引用外部库）

```shell
#src/Makefile.am
bin_PROGRAMS = foo run-me
foo_SOURCES = foo.c foo.h print.c print.h
run_me_SOURCES = run.c run.h print.c
run_me_CPPFLAGS = -I$(srcdir)/../lib

#EFENCELIB=-lefence
run_me_LDADD = ../lib/libcompat.a $(EFENCELIB)
```

### 什么文件会发布呢？
`make dist`, `make diskcheck` 创建一个压缩包，包含：

+ 所有使用 ..._SOURCES 声明的源文件
+ 所有使用 ..._HEADERS 声明的头文件
+ 所有使用 `dist_..._SCRIPTS` 声明的脚本
+ 所有使用 `dist_..._DATA` 声明的数据文件
+ 普通文件， 如changelog， NEWS等。使用`automake --help` 列出所有这些文件。
+ 使用EXTRA_DIST 列出的额外文件或目录

```shell
#Makefile.am
SUBDIRS = lib src
EXTRA_DIST = HACKING
```

### 使用"条件"

+ 条件允许条件构建和无条件发布

```shell
#Conditional Programs
bin_PROGRAMS = foo
if WANT_BAR
bin_PROGRAMS += bar
endif
foo_SOURCES = foo.c
bar_SOURCES = bar.c
```
```shell
#Conditional Sources
bin_PROGRAMS = foo
foo_SOURCES = foo.c
if WANT_BAR
foo_SOURCES += bar.c
endif
```
+ 如果WANT_BAR 为true， bar被构建
+ 如果WANT_BAR 为true， bar.o被链接到foo
+ 不管WANT_BAR 的值，foo.c 和bar.c 总是被发布，
+ 这是可移植的，`config.status `会注释掉`Makefile.in`中不再有效的规则
+ WANT_BAR 必须在configure.ac中声明和赋值

### 声明条件

`AM_CONDITIONAL(NAME, CONDITION)`  
`CONDITION`是一个shell指令，如果判断成功，NAME有效。

### 建议

+ 使用 -Wall -Werror
+ 保持配置简单
+ 不要对Automake撒谎

### autoreconf 重新构建

+ 如果make失败，手动运行autoreconf 重新构建配置文件  
`~/amhello % autoreconf --install`
+ 如果这没有帮助， 尝试：  
`~/amhello % autoreconf --install --force`
+ 如果仍然无效， 再尝试：  
`~/amhello % make -k maintainer-clean`  
`~/amhello % autoreconf --install --force`

只在必要时执行。这些命令将使你的包花费长时间重配置和重编译。
