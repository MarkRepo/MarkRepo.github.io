---
title: 两种IP冲突检测方法
description: 介绍两种检测IP冲突的工具
categories: 运维
tags:
  - commands
  - 运维
---

## arping

假设需要检测A主机（192.168.1.66）的ip地址是否冲突，在同一网段的B主机（192.168.1.55）上执行以下命令：

```shell
$arping 192.168.1.66

ARPING 192.168.1.66 from 192.168.1.55 eth0
Unicast reply from 192.168.1.66 '[10:ab:ec:75:97:C1]' 2.186ms
Unicast reply from 192.168.1.66 '[40:98:6f:45:19:69]' 1.854ms
Unicast reply from 192.168.1.66 '[40:98:6f:45:19:69]' 1.108ms
```
如果只检查出一个MAC地址，则表示A的IP是唯一的。  
如果如上所示检测出两个MAC地址，则表示同网段内有另一台主机与A主机IP冲突。  

### 检验原理

arping命令是以广播地址发送arp packets，以太网内所有的主机都会收到这个arp packets，但是本机收到之后不会Reply任何信息。
当我们在linux主机端上执行下面的命令时：  
`arping 192.168.1.66`  
会默认使用eth0，向局域网内所有的主机发送一个：  
`who has 192.168.1.66的arp request，tell 192.168.1.55 your mac address`  
当这台主机收到这个arp packets后，则会应答：  
`"I am 192.168.1.66 , mac是40:98:6f:45:19:69"`  
这样我们会收到mac地址为40:98:6f:45:19:69的主机的Reply信息。

### arping 命令简介

发送一个`ARP REQUEST` 到相邻主机

#### 语法

`arping [-AbDfhqUV] [-c count] [-w deadline] [-s source] [-I interface] destination`

#### 描述

使用源地址'source'通过ARP数据包在设备接口interface上Ping目的地址destination。

#### 选项

选项 			| 描述
:--- 			| :---
-A 				| 与`-U`相同，但使用ARP REPLY数据包而不是ARP REQUEST。
-b 				| 仅发送MAC级广播。通常arping从发送广播开始，并在回复后切换到单播接收。
-c count 		| 发送count个ARP REQUEST数据包后停止。如果使用deadline选项，等待count个ARP REPLY数据包，或直到截止时间到期。
-D 				| 重复地址检测模式（DAD）。见RFC2131,4.4.1。如果DAD成功，则返回0，即不会受到回复
-f 				| 在确认目标有效的第一个回复后完成。
-I interface 	| 指定发送ARP REQUEST报文的网络接口。
-h 				| 打印帮助信息并退出。
-q 				| 安静的输出。什么都没显示。
-s source 		| 用于ARP数据包的IP源地址。如果此选项不存在，源地址为：(1)在DAD模式下（选项`-D`）设置为`0.0.0.0`。(2)在未经请求的ARP模式下（选项`-U`或`-A`）设置为destination。(3)否则，它是从路由表计算的。
-U 				| Unsolicited ARP模式用来更新邻居的ARP缓存。预计不会有回复。
-V 				| 打印版本信息并退出。
-w deadline 	| 无论已发送或接收了多少数据包，都指定arping退出之前的超时（以秒为单位）。在这种情况下arping在发送了conunt个数据包后不会停止，它等待截止时间到期或直到count个探测得到应答。

## arp-scan

arp-scan这个工具会在本地网络发送ARP（Address Resolution Protocol）(地址解析协议)包来收集地址。如果有多个MAC地址声称拥有相同的IP地址，那么这里就存在冲突。  
首先安装arp-scan，然后

1. `arp-scan -l` 命令表示查看与本机在同一局域网内的所有机器的ip使用情况
2. `arp-scan –I eth0 -l` 命令表示查看与本机在同一局域网内的所有主机的eth0网卡的ip使用情况