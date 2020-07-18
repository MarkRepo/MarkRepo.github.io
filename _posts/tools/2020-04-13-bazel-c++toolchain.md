---
title: Bazel — C++ ToolChain configuration
description: 官方文档学习
categoreis: tools
tags:
  - C/CPP
  - Bazel
  - Google


---

# C++ ToolChain Configuration

## Overview

为了使用正确的选项调用编译器，Bazel需要一些有关编译器内部的知识，例如include目录和重要标志。 换句话说，Bazel需要简化的编译器模型来了解其工作原理。

Bazel需要了解以下内容：

* 编译器是否支持ThinLTO，模块，动态链接或PIC（与位置无关的代码）。
* 所需工具（例如gcc，ld，ar，objcopy等）的路径。
* 内置系统包含目录。 Bazel需要这些来验证源文件中包含的所有头文件均已在构建文件中正确声明。
* 默认的sysroot。
* 用于编译，链接，归档的标志。
* 支持的编译模式（opt，dbg，fastbuild）使用哪些标志。
* 编译器特别要求的make变量。

如果编译器支持多种架构，则Bazel需要单独配置它们。

CcToolchainConfigInfo是一个提供程序，它提供了必要的粒度级别，用于配置Bazel的C ++规则的行为。默认情况下，Bazel会自动为您的构建配置CcToolchainConfigInfo，但是您可以选择手动配置它。为此，您需要一个能提供CcToolchainConfigInfo的Starlark规则，并且需要将cc_toolchain的toolchain_config属性指向您的规则。您可以通过调用cc_common.create_cc_toolchain_config_info（）创建CcToolchainConfigInfo。您可以在@rules_cc//cc:cc_toolchain_config_lib.bzl中找到该过程中所需的所有结构的Starlark构造函数。

当C++目标进入分析阶段时，Bazel将根据BUILD文件选择适当的cc_toolchain目标，并从cc_toolchain.toolchain_config属性中指定的目标中获取CcToolchainConfigInfo提供程序。 cc_toolchain目标将此信息通过CcToolchainProvider传递给C ++目标。

例如，由诸如cc_binary或cc_library之类的规则实例化的编译或链接操作需要以下信息：

* 要使用的编译器或链接器
* 编译器/链接器的命令行标志
* 通过--copt /--linkopt选项传递的配置标志
* 环境变量
* 沙盒中执行动作需要的工件

cc_toolchain指向的Starlark目标中指定了以上所有信息（沙箱中所需的工件除外）。

待传送到沙箱的工件在cc_toolchain目标中声明。 例如，使用cc_toolchain.linker_files属性，您可以指定链接器二进制文件和工具链库，以附带到沙箱中。



## toolchain selection

工具链选择逻辑的操作如下：

* 用户在BUILD文件中指定cc_toolchain_suite目标，并使用--crosstool_top选项将Bazel指向目标
* cc_toolchain_suite目标引用多个工具链。 --cpu和--compiler标志的值确定选择这些工具链中的哪一个，仅基于--cpu标志值，或者基于--cpu | --compiler。选择过程如下：
  * 如果指定了--compiler选项，则Bazel使用 --cpu | --compiler从cc_toolchain_suite.toolchains属性中选择相应的条目。如果Bazel找不到相应的条目，则将引发错误。
  * 如果未指定--compiler选项，则Bazel仅使用--cpu从cc_toolchain_suite.toolchains属性中选择相应的条目。
  * 如果未指定标志，Bazel将检查主机系统并根据其发现结果选择--cpu值。请参阅检查机制代码。

一旦选择了工具链，Starlark规则中的相应feature和action_config对象将控制构建的配置（即，本文档后面介绍的项目）。这些消息允许在Bazel中实现成熟的C ++功能，而无需修改Bazel二进制文件。 

## features

feature是需要命令行标志，操作，对执行环境的约束或依赖关系更改的实体。 feature可以跟以下操作一样简单，例如允许BUILD文件选择标志的配置（例如treat_warnings_as_errors）或与C ++规则进行交互并包括新的编译操作和对编译的输入，例如header_modules或thin_lto。

理想情况下，CcToolchainConfigInfo包含features列表，其中每个features由一个或多个标志组组成，每个标志组定义适用于特定Bazel动作的标志列表。

feature通过名称指定，它允许Starlark规则配置与Bazel版本完全脱钩。 换句话说，Bazel版本不会影响CcToolchainConfigInfo配置的行为，只要这些配置不需要使用新feature即可。

通过以下方式之一启用feature：

* feature的启用字段设置为true。
* Bazel或规则所有者明确启用了它。
* 用户通过--feature Bazel选项或features规则属性启用它。

功能可以具有相互依赖性，取决于命令行标志，BUILD文件设置和其他变量。

### Feature relationships

依赖关系通常由Bazel直接管理，Bazel可以简单地执行要求并管理BUILD中定义的features的本质所固有的冲突。 工具链规范允许在Starlark规则中直接使用更精细的约束来控制feature的支持和扩展。 这些是：

| **Constraint**                                               | **Description**                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `requires = [    feature_set (features = [        'feature-name-1',        'feature-name-2'    ]), ]` | Feature-level. The feature is supported only if the specified required features are enabled. For example, when a feature is only supported in certain build modes (`opt`, `dbg`, or `fastbuild`). If `requires` contains multiple `feature_set`s the feature is supported if any of the `feature_set`s is satisfied (when all specified features are enabled). |
| `implies = ['feature']`                                      | Feature-level. This feature implies the specified feature(s). Enabling a feature also implicitly enables all features implied by it (that is, it functions recursively).Also provides the ability to factor common subsets of functionality out of a set of features, such as the common parts of sanitizers. Implied features cannot be disabled. |
| `provides = ['feature']`                                     | Feature-level. Indicates that this feature is one of several mutually exclusive alternate features. For example, all of the sanitizers could specify `provides = ["sanitizer"]`.This improves error handling by listing the alternatives if the user asks for two or more mutually exclusive features at once. |
| `with_features = [   with_feature_set(     features = ['feature-1'],     not_features = ['feature-2'],   ), ]` | Flag set-level. A feature can specify multiple flag sets with multiple. When `with_features` is specified, the flag set will only expand to the build command if there is at least one `with_feature_set` for which all of the features in the specified `features` set are enabled, and all the features specified in `not_features` set are disabled. If `with_features` is not specified, the flag set will be applied unconditionally for every action specified. |

## Actions

actions提供了在不假定action会如何执行的情况下修改action执行环境的灵活性。 action_config指定action调用的工具二进制文件，而feature指定当action被调用时该工具行为的配置（标志）。

由于action可以修改Bazel action graph，因此feature会引用action来指示它们会影响哪些Bazel操作。 CcToolchainConfigInfo提供程序包含actions, 这些action具有标志和与其相关联的工具，例如c ++-compile。通过将标记与功能相关联，将标记分配给每个action。

每个action名称代表Bazel执行的一种action类型，例如编译或链接。但是，action与Bazel action type之间存在多对一的关系，其中Bazel action type是指实现action的Java类（例如CppCompileAction）。特别是，下表中的“assembler actions”和“compiler actions”是CppCompileAction，而链接动作是CppLinkAction。

### Assembler actions

| **Action**            | **Description**                                           |
| --------------------- | --------------------------------------------------------- |
| `preprocess-assemble` | Assemble with preprocessing. Typically for `.S` files.    |
| `assemble`            | Assemble without preprocessing. Typically for `.s` files. |

### Compiler actions

| **Action**               | **Description**                                              |      |
| ------------------------ | :----------------------------------------------------------- | ---- |
| `cc-flags-make-variable` | Propagates `CC_FLAGS` to genrules.                           |      |
| `c-compile`              | Compile as C.                                                |      |
| `c++-compile`            | Compile as C++.                                              |      |
| `c++-header-parsing`     | Run the compiler's parser on a header file to ensure that the header is self-contained, as it will otherwise produce compilation errors. Applies only to toolchains that support modules. |      |

### Link actions

| **Action**                        | **Description**                                             |
| --------------------------------- | ----------------------------------------------------------- |
| `c++-link-dynamic-library`        | Link a shared library containing all of its dependencies.   |
| `c++-link-nodeps-dynamic-library` | Link a shared library only containing `cc_library` sources. |
| `c++-link-executable`             | Link a final ready-to-run library.                          |

### AR actions

AR actions assemble object files into archive libraries (`.a` files) via `ar` and encode some semantics into the name.

| **Action**                | **Description**                    |
| ------------------------- | ---------------------------------- |
| `c++-link-static-library` | Create a static library (archive). |

### LTO actions

| **Action**    | **Description**                                        |
| ------------- | ------------------------------------------------------ |
| `lto-backend` | ThinLTO action compiling bitcodes into native objects. |
| `lto-index`   | ThinLTO action generating global index.                |

## Action Config

action_config是一种Starlark结构，它通过指定在操作期间要调用的工具（二进制文件）和由功能定义的标志集（其将约束应用于操作的执行）来描述Bazel动作。 action_config（）构造函数具有以下参数：

| **Attribute** | **Description**                                              |
| ------------- | ------------------------------------------------------------ |
| `action_name` | 此动作对应的Bazel动作。 Bazel使用此属性来发现每个操作的工具和执行要求。 |
| `tools`       | 要调用的可执行文件。 应用于操作的工具将是列表中具有与功能配置匹配的功能集的第一个工具。 必须提供默认值。 |
| `flag_sets`   | 适用于一组操作的标志列表。 与功能相同。                      |
| `env_sets`    | 适用于一组操作的环境约束列表。 与功能相同。                  |

一个action_config可以要求并隐含其他功能和action_config，如先前所述的功能关系所规定。 此行为类似于功能的行为。

最后两个属性相对于要素上的相应属性是多余的，并且由于两个Bazel动作需要某些标志或环境变量而被包括在内，因此我们希望避免不必要的action_config + feature对。 通常，最好在多个action_configs中共享一个功能。

您不能在同一工具链中定义多个具有相同action_name的action_config。 这样可以避免工具路径中的歧义，并增强了action_config的意图-在工具链中的单个位置清楚地描述了动作的属性。