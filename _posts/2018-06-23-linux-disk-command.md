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

转换并且复制一个文件

### 语法

`dd [OPERAND]...`  
`dd OPTION`

### 描述

用指定大小的块拷贝一个文件，并在拷贝的同时进行指定的转换和格式化处理。
建议在有需要的时候使用`dd`对物理磁盘操作，如果是文件系统的话还是使用`tar` `backup` `cpio`等其他命令更加方便。另外，使用dd对磁盘操作时，最好使用块设备文件。

>注意：指定数字的地方若以下列字符结尾，则乘以相应的数字：b=512；c=1；k=1024；w=2；M=1024k；G=1024M

### 选项

选项 					| 描述
:--- 					| :---
if=文件名 				| 输入文件名，缺省为标准输入。即指定源文件。<`if=input file`>
of=文件名				| 输出文件名，缺省为标准输出。即指定目的文件。<`of=output file`>
ibs=bytes 				| 一次读入bytes个字节，即指定一个块大小为bytes个字节。
obs=bytes 				| 一次输出bytes个字节，即指定一个块大小为bytes个字节。
bs=bytes 				| 同时设置读入/输出的块大小为bytes个字节。
cbs=bytes 				| 一次转换bytes个字节，即指定转换缓冲区大小。
skip=blocks 			| 从输入文件开头跳过blocks个块后再开始复制。
seek=blocks 			| 从输出文件开头跳过blocks个块后再开始复制。(当输出是磁盘和磁带时有效)
count=blocks 			| 仅拷贝blocks个块，块大小等于ibs指定的字节数。
conv=conversion 		| 用指定的参数转换文件。
iflag=FLAGS				| 按逗号分隔的符号列表读取
oflag=FLAGS 			| 按逗号分隔的符号列表写入
status=LEVEL 			| 要打印到stderr的信息的级别; 'none'除了错误消息之外都禁止所有内容，'noxfer'禁止最终传输统计信息，'progress'显示定期传输统计信息

conv参数包括:

参数 		| 描述
:--- 		| :---
ascii 		| 转换ebcdic为ascii
ebcdic 		| 转换ascii为ebcdic
ibm 		| 转换ascii为alternate ebcdic
block 		| 把每一行转换为长度为cbs，不足部分用空格填充
unblock 	| 使每一行的长度都为cbs，不足部分用空格填充
lcase 		| 把大写字符转换为小写字符
ucase 		| 把小写字符转换为大写字符
swab 		| 交换输入的每对字节
noerror 	| 出错时不停止
notrunc 	| 不截短输出文件
sync 		| 将每个输入块填充到ibs个字节，不足部分用空（NUL）字符补齐。
excl  		| 如果输出文件已经存在则失败
nocreate 	| 不创建输出文件
fdatasync 	| 在完成之前物理写入输出文件数据
fync    	| 与fdatasync相同，但同时写入元数据
sparse 		| 尝试寻找而不是写NUL输入块的输出

FLAG 标志：

标志    		| 描述
:--- 		| :---
append 		| 附加模式（只对输出有意义，建议conv=notrunc）
direct 		| 对数据使用direct I/O
directory 	| 如果不是目录则失败
dsync 		| 对数据使用 synchronized I/O
sync 		| 与dsync一样，同时作用于元数据
fullblock 	| 累积完整的输入块（只用于输入标志）
nonblock 	| 使用 non-blocking I/O
noatime 	| 不更新访问时间
nocache 	| 丢弃缓存数据
noctty 		| 不要从文件中分配控制终端
nofollow 	| 不跟随符号链接
count_bytes | 将'count = N'视为字节数
skip_bytes 	| 将'skip = N'视为字节数
seek_bytes 	| 将'seek = N'视为字节数

### 应用实例

1.将本地的/dev/hdb整盘备份到/dev/hdd

	dd if=/dev/hdb of=/dev/hdd

2.将/dev/hdb全盘数据备份到指定路径的image文件

	dd if=/dev/hdb of=/root/image

3.将备份文件恢复到指定盘

	dd if=/root/image of=/dev/hdb

4.备份/dev/hdb全盘数据，并利用gzip工具进行压缩，保存到指定路径

	dd if=/dev/hdb | gzip > /root/image.gz

5.将压缩的备份文件恢复到指定盘

	gzip -dc /root/image.gz | dd of=/dev/hdb

6.备份与恢复MBR  
备份磁盘开始的512个字节大小的MBR信息到指定文件：

	dd if=/dev/hda of=/root/image count=1 bs=512

count=1指仅拷贝一个块；bs=512指块大小为512个字节。  
恢复：

	dd if=/root/image of=/dev/hda

将备份的MBR信息写到磁盘开始部分

7.备份软盘

	dd if=/dev/fd0 of=disk.img count=1 bs=1440k (即块大小为1.44M)

8.拷贝内存内容到硬盘

	dd if=/dev/mem of=/root/mem.bin bs=1024 (指定块大小为1k)

9.拷贝光盘内容到指定文件夹，并保存为cd.iso文件

	dd if=/dev/cdrom(hdc) of=/root/cd.iso

10.增加swap分区文件大小  
第一步：创建一个大小为256M的文件：

	dd if=/dev/zero of=/swapfile bs=1024 count=262144

第二步：把这个文件变成swap文件：

	mkswap /swapfile

第三步：启用这个swap文件：

	swapon /swapfile

第四步：编辑/etc/fstab文件，使在每次开机时自动加载swap文件：

	/swapfile swap swap default 0 0

11.销毁磁盘数据

	dd if=/dev/urandom of=/dev/hda1

注意：利用随机的数据填充硬盘，在某些必要的场合可以用来销毁数据。

12.测试硬盘的读写速度

	dd if=/dev/zero bs=1024 count=1000000 of=/root/1Gb.file
	dd if=/root/1Gb.file bs=64k | dd of=/dev/null

通过以上两个命令输出的命令执行时间，可以计算出硬盘的读、写速度。

13.确定硬盘的最佳块大小：

	dd if=/dev/zero bs=1024 count=1000000 of=/root/1Gb.file
	dd if=/dev/zero bs=2048 count=500000 of=/root/1Gb.file
	dd if=/dev/zero bs=4096 count=250000 of=/root/1Gb.file
	dd if=/dev/zero bs=8192 count=125000 of=/root/1Gb.file

通过比较以上命令输出中所显示的命令执行时间，即可确定系统最佳的块大小。

14.修复硬盘：

	dd if=/dev/sda of=/dev/sda 或dd if=/dev/hda of=/dev/hda

当硬盘较长时间(一年以上)放置不使用后，磁盘上会产生magnetic flux point，当磁头读到这些区域时会遇到困难，并可能导致I/O错误。当这种情况影响到硬盘的第一个扇区时，可能导致硬盘报废。上边的命令有可能使这些数据起死回生。并且这个过程是安全、高效的。

15.利用netcat远程备份，在源主机上执行以下命令备份/dev/hda

	dd if=/dev/hda bs=16065b | netcat < targethost-ip > 1234

在目的主机上执行以下命令来接收数据并写入/dev/hdc

	netcat -l -p 1234 | dd of=/dev/hdc bs=16065b

以下两条指令是目的主机指令的变化分别采用bzip2、gzip对数据进行压缩，并将备份文件保存在当前目录。  

	netcat -l -p 1234 | bzip2 > partition.img
	netcat -l -p 1234 | gzip > partition.img

将一个很大的视频文件中的第i个字节的值改成0x41（也就是大写字母A的ASCII值）

	echo A | dd of=bigfile seek=$i bs=1 count=1 conv=notrunc

16.将USR1信号发送到正在运行的'dd'进程使其将I/O统计信息打印到标准错误，然后恢复复制。

	$ dd if=/dev/zero of=/dev/null& pid=$!
	$ kill -USR1 $pid; sleep 1; kill $pid

	18335302+0 records in 18335302+0 records out 9387674624 bytes (9.4 GB) copied, 34.6279 seconds, 271 MB/s

### /dev/null和/dev/zero的区别

+ /dev/null，空设备，也称为位桶（bit bucket）。任何写入它的输出都会被抛弃。
+ /dev/zero，输入设备，无穷尽地提供0。可以用于向设备或文件写入字符串0。

```shell
if=/dev/zero of=./test.txt bs=1k count=1
ls –l
total 4
-rw-r--r-- 1 oracle dba 1024 Jul 15 16:56 test.txt
find / -name access_log 2>/dev/null
```

#### 使用/dev/null

把/dev/null看作”黑洞”， 它等价于一个只写文件，所有写入它的内容都会永远丢失.，而尝试从它那儿读取内容则什么也读不到。然而， /dev/null对命令行和脚本都非常的有用

禁止标准输出  
`cat $filename >/dev/null` #文件内容丢失，而不会输出到标准输出.  
禁止标准错误  
`rm $badname 2>/dev/null` #这样错误信息[标准错误]就被丢到太平洋去了  
禁止标准输出和标准错误的输出  
`cat $filename 2>/dev/null >/dev/null`  
如果”$filename”不存在，将不会有任何错误信息提示；如果”$filename”存在， 文件的内容不会打印到标准输出。因此，上面的代码根本不会输出任何信息。当只想测试命令的退出码而不想有任何输出时非常有用。

#### 使用/dev/zero

像/dev/null一样， /dev/zero也是一个伪文件， 但它实际上产生连续不断的null的流（二进制的零流，而不是ASCII型的）。 写入它的输出会丢失不见， 而从/dev/zero读出一连串的null也比较困难， 虽然这也能通过od或一个十六进制编辑器来做到。 /dev/zero主要的用处是用来创建一个指定长度用于初始化的空文件，就像临时交换文件。

用/dev/zero创建一个交换临时文件

```shell
#!/bin/bash
# 创建一个交换文件.
# Root 用户的 $UID 是 0.
ROOT_UID=0 
# 不是 root?
E_WRONG_USER=65 
FILE=/swap
BLOCKSIZE=1024
MINBLOCKS=40
SUCCESS=0
# 这个脚本必须用root来运行.
if [ "$UID" -ne "$ROOT_UID" ]
then
echo; echo "You must be root to run this script."; echo
exit $E_WRONG_USER
fi
blocks=${1:-$MINBLOCKS} 
# 如果命令行没有指定，
#+ 则设置为默认的40块.
# 上面这句等同如：
# --------------------------------------------------
# if [ -n "$1" ]
# then
# blocks=$1
# else
# blocks=$MINBLOCKS
# fi
# --------------------------------------------------
if [ "$blocks" -lt $MINBLOCKS ]
then
# 最少要有 40 个块长.
blocks=$MINBLOCKS 
fi
echo "Creating swap file of size $blocks blocks (KB)."
dd if=/dev/zero of=$FILE bs=$BLOCKSIZE count=$blocks # 把零写入文件.
mkswap $FILE $blocks # 将此文件建为交换文件（或称交换分区）.
swapon $FILE # 激活交换文件.
echo "Swap file created and activated."
exit $SUCCESS
```
关于 /dev/zero 的另一个应用是为特定的目的而用零去填充一个指定大小的文件， 如挂载一个文件系统到环回设备 （loopback device）或"安全地" 删除一个文件  
例子创建ramdisk

```shell
#!/bin/bash
# ramdisk.sh
# "ramdisk"是系统RAM内存的一段，
#+ 它可以被当成是一个文件系统来操作.
# 它的优点是存取速度非常快 (包括读和写).
# 缺点: 易失性, 当计算机重启或关机时会丢失数据.
#+ 会减少系统可用的RAM.
# 10 # 那么ramdisk有什么作用呢?
# 保存一个较大的数据集在ramdisk, 比如一张表或字典,
#+ 这样可以加速数据查询, 因为在内存里查找比在磁盘里查找快得多.

#必须用root来运行.
E_NON_ROOT_USER=70
ROOTUSER_NAME=root
MOUNTPT=/mnt/ramdisk
# 2K 个块 (可以合适的做修改)
SIZE=2000
# 每块有1K (1024 byte) 的大小
BLOCKSIZE=1024
# 第一个 ram 设备
DEVICE=/dev/ram0
username=`id -nu`
if [ "$username" != "$ROOTUSER_NAME" ]
then
echo "Must be root to run "`basename $0`"."
exit $E_NON_ROOT_USER
fi
if [ ! -d "$MOUNTPT" ] # 测试挂载点是否已经存在了,
#+ 如果这个脚本已经运行了好几次了就不会再建这个目录了
then 
mkdir $MOUNTPT #+ 因为前面已经建立了.
fi
dd if=/dev/zero of=$DEVICE count=$SIZE bs=$BLOCKSIZE

# 把RAM设备的内容用零填充.
# 为何需要这么做?
mke2fs $DEVICE # 在RAM设备上创建一个ext2文件系统.
mount $DEVICE $MOUNTPT # 挂载设备.
chmod 777 $MOUNTPT # 使普通用户也可以存取这个ramdisk.
# 但是, 只能由root来缷载它.
echo ""$MOUNTPT" now available for use."
# 现在 ramdisk 即使普通用户也可以用来存取文件了.
# 注意, ramdisk是易失的, 所以当计算机系统重启或关机时ramdisk里的内容会消失.
# 拷贝所有你想保存文件到一个常规的磁盘目录下.
# 重启之后, 运行这个脚本再次建立起一个 ramdisk.
# 仅重新加载 /mnt/ramdisk 而没有其他的步骤将不会正确工作.
# 如果加以改进, 这个脚本可以放在 /etc/rc.d/rc.local,
#+ 以使系统启动时能自动设立一个ramdisk.
# 这样很合适速度要求高的数据库服务器.
exit 0
```
