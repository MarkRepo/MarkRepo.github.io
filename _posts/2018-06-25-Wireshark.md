---
title: Wireshark过滤规则
description: wireshark捕捉与显示过滤规则总结
categories: tools
tag:
  - wireshark
  - tcpdump
  - net
---

[原文链接](http://openmaniak.com/cn/wireshark_use.php)  
**捕捉过滤器**：用于决定将什么样的信息记录在捕捉结果中。需要在开始捕捉前设置。  
**显示过滤器**：在捕捉结果中进行详细查找。他们可以在得到捕捉结果后随意修改。  
两种过滤器使用的语法是完全不同的。

## 捕捉过滤器

捕捉过滤器的语法与其它使用`Lipcap（Linux`）或者`Winpcap（Windows`）库开发的软件一样，比如著名的`TCPdump`。  
捕捉过滤器必须在开始捕捉前设置完毕，这一点跟显示过滤器是不同的。  
语法与示例：

**Protocol** | **Direction**  |  **Host(s)**  |  **Value**  |  **Logical Operations**  |  **Other expression**  
tcp          |  dst           |      port     |     80      |            and           |    tcp dst port 90 

### Protocol（协议）

可能的值:　ether, fddi, ip, arp, rarp, decnet, lat, sca, moprc, mopdl, tcp and udp.  
如果没有特别指明是什么协议，则默认使用所有支持的协议。 

### Direction（方向）

可能的值: src, dst, src and dst, src or dst  
如果没有特别指明来源或目的地，则默认使用 `src or dst` 作为关键字。  
例如，`host 10.2.2.2`与`src or dst host 10.2.2.2`是一样的。

### Host(s)

可能的值： net, port, host, portrange  
如果没有指定此值，则默认使用"host"关键字。  
例如，`src 10.1.1.1`与`src host 10.1.1.1`相同。

### Logical Operations（逻辑运算）

可能的值：not, and, or  
否(`not`)具有最高的优先级。或(`or`)和与(`and`)具有相同的优先级，运算时从左至右进行。  
例如:  
`not tcp port 3128 and tcp port 23`与`(not tcp port 3128) and tcp port 23`相同。  
`not tcp port 3128 and tcp port 23`与`not (tcp port 3128 and tcp port 23)`不同。  

### 例子

+ `ip src host 10.1.1.1` 显示来源IP地址为10.1.1.1的封包。
+ `host 10.1.2.3`  显示目的或来源IP地址为10.1.2.3的封包。
+ `src portrange 2000-2500`  显示来源为UDP或TCP，并且端口号在2000至2500范围内的封包。  
+ `not imcp`  显示除了icmp以外的所有封包。（icmp通常被ping工具使用）  
+ `src host 10.7.2.12 and not dst net 10.200.0.0/16`  
显示来源IP地址为10.7.2.12，但目的地不是10.200.0.0/16的封包。
+ `(src host 10.4.1.12 or src net 10.6.0.0/16) and tcp dst portrange 200-10000 and dst net 10.0.0.0/8`  
显示来源IP为10.4.1.12或者来源网络为10.6.0.0/16，目的地TCP端口号在200至10000之间，并且目的位于网络10.0.0.0/8内的所有封包。 

### 注意事项

+ 当使用关键字作为值时，需使用反斜杠“\”。  
`ether proto \ip` (与关键字"ip"相同).  
这样写将会以IP协议作为目标。  
`ip proto \icmp` (与关键字"icmp"相同).  
这样写将会以ping工具常用的icmp作为目标。  
+ 可以在"ip"或"ether"后面使用"multicast"及"broadcast"关键字。  
当您想排除广播请求时，"no broadcast"就会非常有用。

查看`TCPdump`文档以获得更详细的捕捉过滤器语法说明。

## 显示过滤器

通常经过捕捉过滤器过滤后的数据还是很复杂。此时您可以使用显示过滤器进行更加细致的查找。  
它的功能比捕捉过滤器更为强大，而且在您想修改过滤器条件时，并不需要重新捕捉一次。  
语法与示例:

Protocol  .  String1  .  String2  |  Comparison Operator  |  Value  |  Logical Operations  |  Other expression
ftp       .    passive  .     ip  |         ==            | 1.1.1.1 |             xor      |   icmp.type

### Protocol（协议）

可以使用大量位于OSI模型第2至7层的协议。点击"Expression..."按钮后，您可以看到它们。比如：IP，TCP，DNS，SSH  
![wireshark-expression-button-samll.gif](/assets/images/tools/wireshark_expression_button_small.gif)
![wireshark-display-filter-1-small.png](/assets/images/tools/wireshark_display_filter_1_small.png)

您同样可以在如下所示位置找到所支持的协议：
![wireshark_supported_protocols.gif](/assets/images/tools/wireshark_supported_protocols.gif)
![wireshark_supported_protocols_window_small.gif](/assets/images/tools/wireshark_supported_protocols_window_small.gif)

Wireshark的网站提供了对各种 [协议以及它们子类的说明](https://www.wireshark.org/docs/dfref/)。 

### String1, String2 (可选项)

协议的子类。
点击相关父类旁的"+"号，然后选择其子类。 
![wireshark_display_filter_2_small.png](/assets/images/tools/wireshark_display_filter_2_small.png)

### Comparison operators（比较运算符）

可以使用6种比较运算符：

英文写法 | C语言写法|含义
:--- | :--: | :---
eq | == | 等于
ne | != | 不等于
gt | >  | 大于
lt | <  | 小于
ge | >= | 大于等于
le | <= | 小于等于

### Logical expressions（逻辑运算符）

英文写法 | C语言写法 | 含义
:--- | :--: | :---
and  |  &&  | 逻辑与
or   | \|\| | 逻辑或
xor  |  ^^  | 逻辑异或
not  |   !  | 逻辑非

### 例子

+ `snmp || dns || icmp` 显示SNMP或DNS或ICMP封包。
+ `ip.addr == 10.1.1.1` 显示来源或目的IP地址为10.1.1.1的封包。
+ `ip.src != 10.1.2.3 or ip.dst != 10.4.5.6`显示来源不为10.1.2.3或者目的不为10.4.5.6的封包。  
+ `ip.src != 10.1.2.3 and ip.dst != 10.4.5.6`显示来源不为10.1.2.3并且目的不为10.4.5.6的封包。   
+ `tcp.port == 25` 显示来源或目的TCP端口号为25的封包。
+ `tcp.dstport == 25` 显示目的TCP端口号为25的封包。
+ `tcp.flags` 显示包含TCP标志的封包。
+ `tcp.flags.syn == 0x02`显示包含TCP SYN标志的封包。

如果过滤器的语法是正确的，表达式的背景呈绿色。如果呈红色，说明表达式有误
![wireshark_display_filter_green.gif](/assets/images/tools/wireshark_display_filter_green.gif)
![wireshark_display_filter_red.gif](/assets/images/tools/wireshark_display_filter_red.gif)