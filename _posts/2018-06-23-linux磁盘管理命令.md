---
title: Linux 常用磁盘管理命令总结
description: 基于CenOS7的man-page总结Linux磁盘管理命令
categories: Linux运维
tags:
  - Linux
  - commands
  - 运维
---

## df

报告文件系统磁盘空间使用率

### 语法

`df [OPTION]... [FILE]...`

### 描述

df显示包含每个文件名参数的文件系统上可用的磁盘空间量。 如果未给出文件名，则会显示所有当前安装的文件系统上可用的空间。 磁盘空间默认显示为1K块，如果设置了环境变量`POSIXLY_CORRECT`，将使用512字节的块。   
如果参数是包含**已装入文件系统**的磁盘设备节点的绝对文件名，则df将显示该文件系统上的可用空间，而不是包含该设备节点的文件系统上的可用空间。 此版本的df无法显示未安装的文件系统上可用的空间，因为在大多数系统上这样做需要很多不可移植的与文件系统结构密切相关的知识。

### 选项

选项 						| 描述
:--- 						| :---
`-a`, `--all`             	| 包含所有文件系统
`-B`, `--block-size=SIZE` 	| 以指定区块大小显示区块数目，例如`-BM`显示1M单元的数目
`--direct`              	| 以文件代替挂载点显示统计数据
`--total` 			  		| 生成总计信息
`-h`, `--human-readable`  	| 以人类易读的格式显示大小
`-H`, `--si` 			  	| 与上相似，但是用1000的幂而不是1024
`-i`, `--inodes` 		  	| 显示inode代替区块的使用信息
`-k` 					  	| `--block-size=1K`
`-l`, `--local` 		  	| 只显示本地文件系统
`--no-sync` 			  	| 在获取使用信息前不调用`sync`(默认行为)
`--output[=FIELD_LIST]` 	| 使用`FIELD_LIST`定义输出格式，若忽略`FIELD_LIST`，显示所有列
`-P`, `--portabilit` 	  	| 使用`POSIX`输出格式
`--sync` 				  	| 在获取使用信息前调用`sync`
`-t`, `--type=TYPE` 	  	| 只显示`TYPE`类型的文件系统
`-T`,`--print-type`	  		| 显示文件系统类型
`-x`, `--exclude-type=TYPE` | `TYPE`类型文件系统不显示
`-v` 						| 忽略
`--help` 					| 显示帮助信息
`--version` 				| 显示版本信息

显示值以`--block-size`和`DF_BLOCK_SIZE`，`BLOCK_SIZE`和`BLOCKSIZE`环境变量中的第一个可用`SIZE`为单位。 否则，单位默认为1024个字节（如果设置`POSIXLY_CORRECT`，则为512）。  
SIZE是一个整数和可选单位（例如：10M是10 * 1024 * 1024）。 单位是`K，M，G，T，P，E，Z，Y`（1024的幂）或`KB，MB，...`（1000的幂）。
FIELD_LIST是要包含的列的逗号分隔列表。  
有效的字段名称是：'source'，'fstype'，'itotal'，'iused'，'iavail'，'ipcent'，'size'，'used'，'avail'，'pcent'，'file'和'target'

## du

估计文件空间使用量

### 语法

`du [OPTION]... [FILE]...`  
`du [OPTION]... --files0-from=F`

### 描述

总结每个FILE的磁盘使用情况，递归查找目录。

### 选项

选项 						| 描述
:--- 						| :---
`-0`, `--null` 				| 用`0`字节而不是换行结束每个输出行
`-a`, `--all` 				| 为所有文件写入计数，而不仅是目录
`--apparent-size` 	| 显示表面大小，而非磁盘使用量;虽然表面大小通常较小，但由于（'稀疏'）文件中的空洞，内部碎片，间接块等原因它可能更大。
`-B`, `--block-size=SIZE` 	| 以指定区块大小显示区块数目，例如`-BM`显示1M单元的数目
`-b`, `--bytes` 			| 等价于 `--apparent-size --block-size=1`
`-c`, `--total` 			| 生成总计信息
`-D`, `--dereference-args` 	| 显示指定符号链接的原文件大小
`-d`, `--max-depth=N` 		| 指定统计目录的深度， `--max-depth=0` 相当于`--summarize`
`--files0-from=F`	 		| 统计`F`中指定的以`NUL`结尾的文件名的磁盘使用率。如果`F`是`-`，从标准输入读取名字。
`-H` 						| 与 `--dereference-args (-D)`相同
`-h`, `--human-readable`	| 以人类易读的格式（`K,M,G`）显示大小
`--inodes` 					| 显示`inode`代替区块的使用信息
`-k` 						| `--block-size=1K`
`-L`,`--dereference` 		| 解引用所有的符号链接
`-l`, `--count-links` 		| 重复计算硬链接文件
`-m` 						| `--block-size=1M`
`-P`, `--no-dereference` 	| 不跟随符号链接（默认行为）
`-S` ,`--separate-dirs` 	| 不包含子目录大小
`--si` 						| 与`-h`相同，但使用`1000`的幂
`-s`, `--summarize`			| 只为一个参数显示一个总计信息
`-t`, `--threshold=SIZE` 	| 排除小于`SIZE`的条目（如果为正数），或者大于`SIZE`的条目（如果为负数）
`--time` 					| 显示目录中任何文件或其任何子目录的最后修改时间
`--time=WORD`				| 显示`WORD`时间代替最后修改时间，`WORD`可以是：`atime, access,use,ctime or status`
`--time-style=STYLE` 		| 使用`STYLE`显示时间，可以是`full-iso, long-iso, iso, 或者 +FORMAT`；FORMAT被解释成'date'
`-X`, `--exclude-from=FILE` | 排除与FILE中的任何模式匹配的文件
`--exclude=PATTERN` 		| 排除匹配`PATTERN`模式的文件
`-x`, `--one-file-system`	| 以一开始处理时的文件系统为准，若遇上其它不同的文件系统目录则略过。
`--help` 					| 显示帮助信息
`--version`					| 显示版本信息

显示值以`--block-size`和`DF_BLOCK_SIZE`，`BLOCK_SIZE`和`BLOCKSIZE`环境变量中的第一个可用`SIZE`为单位。 否则，单位默认为1024个字节（如果设置`POSIXLY_CORRECT`，则为512）。  
SIZE是一个整数和可选单位（例如：10M是10 * 1024 * 1024）。 单位是`K，M，G，T，P，E，Z，Y`（1024的幂）或`KB，MB，...`（1000的幂）。

## fdisk

## lsblk

列出块设备

### 语法

`lsblk [options] [device...]`

### 描述

`lsblk`列出有关所有可用或指定块设备的信息。 `lsblk`命令读取`sysfs`文件系统以收集信息。默认情况下，命令以树状格式打印所有块设备（RAM磁盘除外）。 使用`lsblk --help`获取所有可用列的列表。默认输出以及来自选项（如`--fs`和`--topology`）的默认输出可能会更改。 所以只要有可能，您应该避免在脚本中使用默认输出。 在需要稳定输出的环境中始终使用`--output columns-list`来显式定义预期的列。

### 选项

选项 					| 描述
:--- 					| :---
`-a`, `--all` 			| 列出空设备（默认跳过）
`-b`, `--bytes` 		| 以字节显示SIZE列
`-D`, `--discard` 		| 打印每个设备的丢弃能力（TRIM，UNMAP）信息
`-d`, `--nodeps` 		| 不要打印持有者设备或从属设备
`-e`, `--exclude list` 	| 排除由逗号分隔的主设备号列表中的设备，默认排除RAM磁盘，过滤器只适用于顶层设备。
`-f`, `--fs` 			| 显示文件系统信息
`-h`, `--help` 			| 显示帮助信息
`-I`, `--include list` 	| 包含由逗号分隔的主设备号列表中的设备， 过滤器只适用于顶层设备。
`-i`, `--ascii` 		| 使用ASCII码字符树格式
`-l`, `--list` 			| 使用列表格式显示
`-m`, `--perms` 		| 显示权限信息
`-n`, `--noheadings` 	| 不显示标题
`-o`, `--ouput list` 	| 指定显示列，使用 +list 扩展默认显示列
`-P`, `--pairs` 		| 使用 key="value"格式显示
`-p`, `--paths` 		| 显示完整的设备路径
`-r`, `--raw` 			| 使用原始格式显示
`-S`, `--scsi` 			| 只显示SCSI设备，所有的分区，持有者和从属设备被忽略。
`-s`, `--inverse`		| 以相反的依赖关系显示
`-t`, `--topology` 		| 显示拓扑结构信息
`-V`, `--version`		| 显示版本信息

## mount

## umount

## dd