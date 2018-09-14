---
title: PyMOTW-3 --- Networking
categories: language
tags: PyMOTW-3
---

网络通信用于检索本地运行的算法所需的数据，共享用于分布式处理的信息以及管理云服务。 Python的标准库包含用于创建网络服务以及远程访问现有服务的模块。  
ipaddress模块​​包括用于验证，比较和以其他方式操作IPv4和IPv6网络地址的类。  
低级socket库提供对本机C套接字库的直接访问，可用于与任何网络服务进行通信。selectors提供了一个用于同时监测多个套接字的高级接口，对于允许网络服务器同时与多个客户端通信非常有用。 select提供低级API给selectors使用。  
socketserver中的框架抽象出了创建新网络服务端所需的大量重复工作。可以组合这些类来创建服务，这些服务使用fork或线程并支持TCP或UDP。只需要应用程序提供实际的消息处理。

## ipaddress — Internet Addresses

目的：处理IP协议地址的类。  
ipaddress模块包括用于处理IPv4和IPv6网络地址的类。 这些类支持验证，查找网络上的地址和主机以及其他常见操作。

### Addresses

最基本的对象代表网络地址本身。 将字符串，整数或字节序列传递给ip_address（）以构造地址。 返回值将是IPv4Address或IPv6Address实例，具体取决于所使用的地址类型。

```python
#ipaddress_addresses.py
import binascii
import ipaddress

ADDRESSES = [
    '10.9.0.6',
    'fdfd:87b5:b475:5e3e:b1bc:e121:a8eb:14aa',
]

for ip in ADDRESSES:
    addr = ipaddress.ip_address(ip)
    print('{!r}'.format(addr))
    print('   IP version:', addr.version)
    print('   is private:', addr.is_private)
    print('  packed form:', binascii.hexlify(addr.packed))
    print('      integer:', int(addr))
    print()
```
这两个类可以为不同目的提供地址的各种表示，以及回答基本断言，例如地址是为多播通信保留还是在专用网络上。

```
$ python3 ipaddress_addresses.py

IPv4Address('10.9.0.6')
   IP version: 4
   is private: True
  packed form: b'0a090006'
      integer: 168361990

IPv6Address('fdfd:87b5:b475:5e3e:b1bc:e121:a8eb:14aa')
   IP version: 6
   is private: True
  packed form: b'fdfd87b5b4755e3eb1bce121a8eb14aa'
      integer: 337611086560236126439725644408160982186
```

### Networks

网络由一系列地址定义。 它通常用基地址和掩码表示，该地址和掩码指示地址的哪些部分代表该网络，以及剩余的哪些部分代表该网络上的地址。 掩码可以显式表示，或使用前缀长度值，如下例所示。

```python
#ipaddress_networks.py
import ipaddress

NETWORKS = [
    '10.9.0.0/24',
    'fdfd:87b5:b475:5e3e::/64',
]

for n in NETWORKS:
    net = ipaddress.ip_network(n)
    print('{!r}'.format(net))
    print('     is private:', net.is_private)
    print('      broadcast:', net.broadcast_address)
    print('     compressed:', net.compressed)
    print('   with netmask:', net.with_netmask)
    print('  with hostmask:', net.with_hostmask)
    print('  num addresses:', net.num_addresses)
    print()
```
与地址一样，有两种网络类分别用于IPv4和IPv6网络。 每个类提供用于访问与网络相关联的值的属性或方法，例如广播地址和可供主机使用的网络上的地址。

```
$ python3 ipaddress_networks.py

IPv4Network('10.9.0.0/24')
     is private: True
      broadcast: 10.9.0.255
     compressed: 10.9.0.0/24
   with netmask: 10.9.0.0/255.255.255.0
  with hostmask: 10.9.0.0/0.0.0.255
  num addresses: 256

IPv6Network('fdfd:87b5:b475:5e3e::/64')
     is private: True
      broadcast: fdfd:87b5:b475:5e3e:ffff:ffff:ffff:ffff
     compressed: fdfd:87b5:b475:5e3e::/64
   with netmask: fdfd:87b5:b475:5e3e::/ffff:ffff:ffff:ffff::
  with hostmask: fdfd:87b5:b475:5e3e::/::ffff:ffff:ffff:ffff
  num addresses: 18446744073709551616
```
网络实例是可迭代的，并产生网络上的地址。

```python
#ipaddress_network_iterate.py
import ipaddress

NETWORKS = [
    '10.9.0.0/24',
    'fdfd:87b5:b475:5e3e::/64',
]

for n in NETWORKS:
    net = ipaddress.ip_network(n)
    print('{!r}'.format(net))
    for i, ip in zip(range(3), net):
        print(ip)
    print()
```
此示例仅打印一些地址，因为IPv6网络可以包含的地址远多于输出中的地址。

```
$ python3 ipaddress_network_iterate.py

IPv4Network('10.9.0.0/24')
10.9.0.0
10.9.0.1
10.9.0.2

IPv6Network('fdfd:87b5:b475:5e3e::/64')
fdfd:87b5:b475:5e3e::
fdfd:87b5:b475:5e3e::1
fdfd:87b5:b475:5e3e::2
```
迭代网络会产生地址，但并非所有地址都对主机有效，例如网络的基地址和广播地址。 要查找网络上常规主机可以使用的地址，请使用hosts()方法，它会生成一个生成器。

```python
#ipaddress_network_iterate_hosts.py
import ipaddress

NETWORKS = [
    '10.9.0.0/24',
    'fdfd:87b5:b475:5e3e::/64',
]

for n in NETWORKS:
    net = ipaddress.ip_network(n)
    print('{!r}'.format(net))
    for i, ip in zip(range(3), net.hosts()):
        print(ip)
    print()
```
将此示例的输出与前一示例进行比较表明，主机地址不包括在整个网络上进行迭代时生成的第一个值。

```
$ python3 ipaddress_network_iterate_hosts.py

IPv4Network('10.9.0.0/24')
10.9.0.1
10.9.0.2
10.9.0.3

IPv6Network('fdfd:87b5:b475:5e3e::/64')
fdfd:87b5:b475:5e3e::1
fdfd:87b5:b475:5e3e::2
fdfd:87b5:b475:5e3e::3
```
除了迭代器协议之外，网络还支持in运算符来确定地址是否是网络的一部分。

```python
#ipaddress_network_membership.py
import ipaddress

NETWORKS = [
    ipaddress.ip_network('10.9.0.0/24'),
    ipaddress.ip_network('fdfd:87b5:b475:5e3e::/64'),
]

ADDRESSES = [
    ipaddress.ip_address('10.9.0.6'),
    ipaddress.ip_address('10.7.0.31'),
    ipaddress.ip_address('fdfd:87b5:b475:5e3e:b1bc:e121:a8eb:14aa'),
    ipaddress.ip_address('fe80::3840:c439:b25e:63b0'),
]

for ip in ADDRESSES:
    for net in NETWORKS:
        if ip in net:
            print('{}\nis on {}'.format(ip, net))
            break
    else:
        print('{}\nis not on a known network'.format(ip))
    print()
```
in的实现使用网络掩码来测试地址，因此它比扩展网络上的完整地址列表更有效。

```
$ python3 ipaddress_network_membership.py

10.9.0.6
is on 10.9.0.0/24

10.7.0.31
is not on a known network

fdfd:87b5:b475:5e3e:b1bc:e121:a8eb:14aa
is on fdfd:87b5:b475:5e3e::/64

fe80::3840:c439:b25e:63b0
is not on a known network
```

### Interfaces

网络接口表示网络上的特定地址，并且可以由主机地址和网络前缀或网络掩码表示。

```python
#ipaddress_interfaces.py
import ipaddress

ADDRESSES = [
    '10.9.0.6/24',
    'fdfd:87b5:b475:5e3e:b1bc:e121:a8eb:14aa/64',
]

for ip in ADDRESSES:
    iface = ipaddress.ip_interface(ip)
    print('{!r}'.format(iface))
    print('network:\n  ', iface.network)
    print('ip:\n  ', iface.ip)
    print('IP with prefixlen:\n  ', iface.with_prefixlen)
    print('netmask:\n  ', iface.with_netmask)
    print('hostmask:\n  ', iface.with_hostmask)
    print()
```
接口对象具有分别访问完整网络和地址的属性，以及表达接口和网络掩码的几种不同方式。

```
$ python3 ipaddress_interfaces.py

IPv4Interface('10.9.0.6/24')
network:
   10.9.0.0/24
ip:
   10.9.0.6
IP with prefixlen:
   10.9.0.6/24
netmask:
   10.9.0.6/255.255.255.0
hostmask:
   10.9.0.6/0.0.0.255

IPv6Interface('fdfd:87b5:b475:5e3e:b1bc:e121:a8eb:14aa/64')
network:
   fdfd:87b5:b475:5e3e::/64
ip:
   fdfd:87b5:b475:5e3e:b1bc:e121:a8eb:14aa
IP with prefixlen:
   fdfd:87b5:b475:5e3e:b1bc:e121:a8eb:14aa/64
netmask:
   fdfd:87b5:b475:5e3e:b1bc:e121:a8eb:14aa/ffff:ffff:ffff:ffff::
hostmask:
   fdfd:87b5:b475:5e3e:b1bc:e121:a8eb:14aa/::ffff:ffff:ffff:ffff
```

## socket — Network Communication

目的：提供对网络通信的访问  
socket模块给出低级C API，以便使用BSD套接字接口通过网络进行通信。 它包括用于处理实际数据通道的socket类，还包括用于网络相关任务的函数，例如将服务器名称转换为地址以及格式化要通过网络发送的数据。

### Addressing, Protocol Families and Socket Types

套接字是程序用于在本地或通过Internet来回传递数据的通信信道的一个端点。套接字有两个主要属性来控制它们发送数据的方式：地址族控制所使用的OSI网络层协议，套接字类型控制传输层协议。  
Python支持三个地址族。最常见的AF_INET用于IPv4 Internet寻址。 IPv4地址长度为四个字节，通常表示为四个数字的序列，每个数字一个字节，用点分隔（例如，10.1.1.5和127.0.0.1）。这些值通常被称为“IP地址”。目前几乎所有的Internet网络都是使用IPv4完成的。  
AF_INET6用于IPv6 Internet寻址。 IPv6是Internet协议的“下一代”版本，支持128位地址，IPv4下不可用的流量控制和路由功能。 IPv6的使用率持续增长，特别是随着云计算和由于物联网项目而添加到网络中的额外设备的激增。  
`AF_UNIX`是用于Unix域套接字（UDS）的地址族，它是POSIX兼容系统上可用的进程间通信协议。 UDS的实现通常允许操作系统将数据直接从进程传递到进程，而无需通过网络堆栈。 这比使用AF_INET更有效，但由于文件系统用作寻址的命名空间，因此UDS仅限于同一系统上的进程。 使用UDS而不是其他IPC机制（如命名管道或共享内存）的吸引力在于编程接口与IP网络相同，因此当用程序在单个主机上运行时可以利用其高效的通信，当通过网络发送数据时使用相同的代码。
>Note，AF_UNIX常量仅在支持UDS的系统上定义。

套接字类型通常是用于面向消息的数据报传输的`SOCK_DGRAM`或用于面向流的传输的`SOCK_STREAM`。数据报套接字通常与UDP（用户数据报协议）相关联。它们提供不可靠的单个消息传递。面向流的套接字与TCP传输控制协议相关联。它们在客户端和服务器之间提供字节流，通过超时管理，重传和其他功能确保消息传递或故障通知。  
大多数传递大量数据的应用程序协议（如HTTP）都是基于TCP构建的，因为它可以自动处理消息排序和传递，以使创建复杂的应用程序更容易。 UDP通常用于顺序不太重要的协议（因为消息是自包含的并且通常很小，例如通过DNS查找名称），或者用于多播（将相同数据发送到多个主机）。 UDP和TCP都可以与IPv4或IPv6寻址一起使用。

>Note  
Python socket模块支持其他套接字类型，但它们不太常用，因此这里不做介绍。 有关更多详细信息，请参阅标准库文档。

#### Looking up Hosts on the Network

socket 包括与网络上的域名服务接口通信的函数，因此程序可以将服务器的主机名转换为其数字网络地址。 应用程序在使用它们连接到服务器之前不需要显式转换地址，但在报告错误时包含数字地址以及名称会很有用。  
要查找当前主机的正式名称，请使用gethostname()。

```python
#socket_gethostname.py
import socket

print(socket.gethostname())
```
返回的名称取决于当前系统的网络设置，如果位于不同的网络（例如连接到无线LAN的笔记本电脑），则可能会更改。

```
$ python3 socket_gethostname.py

apu.hellfly.net
```
使用gethostbyname（）查阅操作系统主机名解析API，并将服务器名称转换为其数字地址。

```python
#socket_gethostbyname.py
import socket

HOSTS = [
    'apu',
    'pymotw.com',
    'www.python.org',
    'nosuchname',
]

for host in HOSTS:
    try:
        print('{} : {}'.format(host, socket.gethostbyname(host)))
    except socket.error as msg:
        print('{} : {}'.format(host, msg))
```
如果当前系统的DNS配置在搜索中包含一个或多个域，则name参数不需要是完全限定名称（即，它不需要包括域名以及基本主机名）。 如果找不到该名称，则会引发类型为socket.error的异常。

```
$ python3 socket_gethostbyname.py

apu : 10.9.0.10
pymotw.com : 66.33.211.242
www.python.org : 151.101.32.223
nosuchname : [Errno 8] nodename nor servname provided, or not
known
```
要访问有关服务器的更多命名信息，请使用gethostbyname_ex（）。 它返回服务器的规范主机名，任何别名以及可用于访问它的所有可用IP地址。

```python
#socket_gethostbyname_ex.py
import socket

HOSTS = [
    'apu',
    'pymotw.com',
    'www.python.org',
    'nosuchname',
]

for host in HOSTS:
    print(host)
    try:
        name, aliases, addresses = socket.gethostbyname_ex(host)
        print('  Hostname:', name)
        print('  Aliases :', aliases)
        print(' Addresses:', addresses)
    except socket.error as msg:
        print('ERROR:', msg)
    print()
```
拥有服务器的所有已知IP地址后，客户端可以实现自己的负载平衡或故障转移算法。

```
$ python3 socket_gethostbyname_ex.py

apu
  Hostname: apu.hellfly.net
  Aliases : ['apu']
 Addresses: ['10.9.0.10']

pymotw.com
  Hostname: pymotw.com
  Aliases : []
 Addresses: ['66.33.211.242']

www.python.org
  Hostname: prod.python.map.fastlylb.net
  Aliases : ['www.python.org', 'python.map.fastly.net']
 Addresses: ['151.101.32.223']

nosuchname
ERROR: [Errno 8] nodename nor servname provided, or not known
```
使用getfqdn（）将部分名称转换为完全限定的域名。

```python
#socket_getfqdn.py
import socket

for host in ['apu', 'pymotw.com']:
    print('{:>10} : {}'.format(host, socket.getfqdn(host)))
```
如果输入是别名，则返回的名称不一定与输入参数匹配，例如这里的www。

```
$ python3 socket_getfqdn.py

       apu : apu.hellfly.net
pymotw.com : apache2-echo.catalina.dreamhost.com
```
当服务器的地址可用时，使用gethostbyaddr（）对名称执行“反向”查找。

```python
#socket_gethostbyaddr.py
import socket

hostname, aliases, addresses = socket.gethostbyaddr('10.9.0.10')

print('Hostname :', hostname)
print('Aliases  :', aliases)
print('Addresses:', addresses)
```
返回值是一个元组，包含完整主机名，任何别名以及与该名称关联的所有IP地址。

```
$ python3 socket_gethostbyaddr.py

Hostname : apu.hellfly.net
Aliases  : ['apu']
Addresses: ['10.9.0.10']
```

#### Finding Service Information

除IP地址外，每个套接字地址还包括一个整数端口号。 许多应用程序可以在同一主机上运行，侦听单个IP地址，但在该地址上，一个端口只能有一个套接字。 IP地址，协议和端口号的组合唯一地标识通信信道，并确保通过套接字发送的消息到达正确的目的地。

某些端口号是为特定协议预先分配的。 例如，使用SMTP的电子邮件服务器之间的通信使用TCP端口号25，而Web客户端和服务器使用http端口80。 可以使用getservbyname（）查找具有标准化名称的网络服务的端口号。

```python
#socket_getservbyname.py
import socket
from urllib.parse import urlparse

URLS = [
    'http://www.python.org',
    'https://www.mybank.com',
    'ftp://prep.ai.mit.edu',
    'gopher://gopher.micro.umn.edu',
    'smtp://mail.example.com',
    'imap://mail.example.com',
    'imaps://mail.example.com',
    'pop3://pop.example.com',
    'pop3s://pop.example.com',
]

for url in URLS:
    parsed_url = urlparse(url)
    port = socket.getservbyname(parsed_url.scheme)
    print('{:>6} : {}'.format(parsed_url.scheme, port))
```
虽然标准化服务不太可能改变端口，但是在未来添加新服务时，通过系统调用而不是硬编码来查找值会更灵活。

```
$ python3 socket_getservbyname.py

  http : 80
 https : 443
   ftp : 21
gopher : 70
  smtp : 25
  imap : 143
 imaps : 993
  pop3 : 110
 pop3s : 995
```
要反转服务端口查找，请使用getservbyport（）。

```python
#socket_getservbyport.py
import socket
from urllib.parse import urlunparse

for port in [80, 443, 21, 70, 25, 143, 993, 110, 995]:
    url = '{}://example.com/'.format(socket.getservbyport(port))
    print(url)
```
反向查找对于从任意地址构造服务的URL非常有用。

```
$ python3 socket_getservbyport.py

http://example.com/
https://example.com/
ftp://example.com/
gopher://example.com/
smtp://example.com/
imap://example.com/
imaps://example.com/
pop3://example.com/
pop3s://example.com/
```
可以使用getprotobyname（）检索分配给传输协议的编号。

```python
#socket_getprotobyname.py
import socket

def get_constants(prefix):
    """Create a dictionary mapping socket module
    constants to their names.
    """
    return {
        getattr(socket, n): n
        for n in dir(socket)
        if n.startswith(prefix)
    }

protocols = get_constants('IPPROTO_')

for name in ['icmp', 'udp', 'tcp']:
    proto_num = socket.getprotobyname(name)
    const_name = protocols[proto_num]
    print('{:>4} -> {:2d} (socket.{:<12} = {:2d})'.format(
        name, proto_num, const_name,
        getattr(socket, const_name)))
```
协议号的值是标准化的，并在套接字中定义为带有前缀`IPPROTO_`的常量。

```
$ python3 socket_getprotobyname.py

icmp ->  1 (socket.IPPROTO_ICMP =  1)
 udp -> 17 (socket.IPPROTO_UDP  = 17)
 tcp ->  6 (socket.IPPROTO_TCP  =  6)
```

#### Looking Up Server Addresses

getaddrinfo()将服务的基本地址转换为元组列表，其中包含建立连接所需的所有信息。 每个元组的内容会有所不同，包含不同的网络族或协议。

```python
#socket_getaddrinfo.py
import socket

def get_constants(prefix):
    """Create a dictionary mapping socket module
    constants to their names.
    """
    return {
        getattr(socket, n): n
        for n in dir(socket)
        if n.startswith(prefix)
    }

families = get_constants('AF_')
types = get_constants('SOCK_')
protocols = get_constants('IPPROTO_')

for response in socket.getaddrinfo('www.python.org', 'http'):

    # Unpack the response tuple
    family, socktype, proto, canonname, sockaddr = response

    print('Family        :', families[family])
    print('Type          :', types[socktype])
    print('Protocol      :', protocols[proto])
    print('Canonical name:', canonname)
    print('Socket address:', sockaddr)
    print()
```
该程序演示了如何查找www.python.org的连接信息。

```
$ python3 socket_getaddrinfo.py

Family        : AF_INET
Type          : SOCK_DGRAM
Protocol      : IPPROTO_UDP
Canonical name:
Socket address: ('151.101.32.223', 80)

Family        : AF_INET
Type          : SOCK_STREAM
Protocol      : IPPROTO_TCP
Canonical name:
Socket address: ('151.101.32.223', 80)

Family        : AF_INET6
Type          : SOCK_DGRAM
Protocol      : IPPROTO_UDP
Canonical name:
Socket address: ('2a04:4e42:8::223', 80, 0, 0)

Family        : AF_INET6
Type          : SOCK_STREAM
Protocol      : IPPROTO_TCP
Canonical name:
Socket address: ('2a04:4e42:8::223', 80, 0, 0)
```
getaddrinfo（）接受几个参数来过滤结果列表。 示例中给出的主机和端口值是必需参数。 可选参数是family，socktype，proto和flags。 可选值应为0或socket定义的常量之一。

```python
#socket_getaddrinfo_extra_args.py
import socket

def get_constants(prefix):
    """Create a dictionary mapping socket module
    constants to their names.
    """
    return {
        getattr(socket, n): n
        for n in dir(socket)
        if n.startswith(prefix)
    }

families = get_constants('AF_')
types = get_constants('SOCK_')
protocols = get_constants('IPPROTO_')

responses = socket.getaddrinfo(
    host='www.python.org',
    port='http',
    family=socket.AF_INET,
    type=socket.SOCK_STREAM,
    proto=socket.IPPROTO_TCP,
    flags=socket.AI_CANONNAME,
)

for response in responses:
    # Unpack the response tuple
    family, socktype, proto, canonname, sockaddr = response

    print('Family        :', families[family])
    print('Type          :', types[socktype])
    print('Protocol      :', protocols[proto])
    print('Canonical name:', canonname)
    print('Socket address:', sockaddr)
    print()
```
由于flags包含`AI_CANONNAME`，因此服务器的规范名称（如果主机具有任何别名，可能与用于查找的值不同）包含在此次结果中。 如果没有该标志，则规范名称值将保留为空。

```
$ python3 socket_getaddrinfo_extra_args.py

Family        : AF_INET
Type          : SOCK_STREAM
Protocol      : IPPROTO_TCP
Canonical name: prod.python.map.fastlylb.net
Socket address: ('151.101.32.223', 80)
```

#### IP Address Representations

用C编写的网络程序使用数据类型struct sockaddr将IP地址表示为二进制值（而不是通常在Python程序中找到的字符串地址）。 要在Python表示和C表示之间转换IPv4地址，请使用`inet_aton（）`和`inet_ntoa（）`。

```python
#socket_address_packing.py
import binascii
import socket
import struct
import sys

for string_address in ['192.168.1.1', '127.0.0.1']:
    packed = socket.inet_aton(string_address)
    print('Original:', string_address)
    print('Packed  :', binascii.hexlify(packed))
    print('Unpacked:', socket.inet_ntoa(packed))
    print()
```
打包格式中的四个字节可以传递给C库，通过网络安全传输，或者紧凑地保存到数据库中。

```
$ python3 socket_address_packing.py

Original: 192.168.1.1
Packed  : b'c0a80101'
Unpacked: 192.168.1.1

Original: 127.0.0.1
Packed  : b'7f000001'
Unpacked: 127.0.0.1
```
相关函数`inet_pton（）`和`inet_ntop（）`使用IPv4和IPv6地址，根据传入的地址族参数生成适当的格式。

```python
#socket_ipv6_address_packing.py
import binascii
import socket
import struct
import sys

string_address = '2002:ac10:10a:1234:21e:52ff:fe74:40e'
packed = socket.inet_pton(socket.AF_INET6, string_address)

print('Original:', string_address)
print('Packed  :', binascii.hexlify(packed))
print('Unpacked:', socket.inet_ntop(socket.AF_INET6, packed))
```
IPv6地址已经是十六进制值，因此将打包版本转换为一系列十六进制数字会生成类似于原始值的字符串。

```
$ python3 socket_ipv6_address_packing.py

Original: 2002:ac10:10a:1234:21e:52ff:fe74:40e
Packed  : b'2002ac10010a1234021e52fffe74040e'
Unpacked: 2002:ac10:10a:1234:21e:52ff:fe74:40e
```

### TCP/IP Client and Server

套接字可以配置为服务器并侦听传入消息，或作为客户端连接到其他应用程序。 TCP/IP套接字的两端连接后，通信是双向的。

#### Echo Server

此示例程序基于标准库文档中的示例程序，接收传入的消息并将它们回送给发送方。 它首先创建一个TCP/IP套接字，然后使用bind（）将套接字与服务器地址相关联。 在这个例子中，地址是localhost，指向当前服务器，端口号是10000。

```python
#socket_echo_server.py
import socket
import sys

# Create a TCP/IP socket
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Bind the socket to the port
server_address = ('localhost', 10000)
print('starting up on {} port {}'.format(*server_address))
sock.bind(server_address)

# Listen for incoming connections
sock.listen(1)

while True:
    # Wait for a connection
    print('waiting for a connection')
    connection, client_address = sock.accept()
    try:
        print('connection from', client_address)

        # Receive the data in small chunks and retransmit it
        while True:
            data = connection.recv(16)
            print('received {!r}'.format(data))
            if data:
                print('sending data back to the client')
                connection.sendall(data)
            else:
                print('no data from', client_address)
                break
    finally:
        # Clean up the connection
        connection.close()
```
调用listen（）会将套接字置于服务器模式，而accept（）会等待传入连接。 整数参数是系统在拒绝新客户端之前应在后台排队的连接数。 此示例仅期望一次使用一个连接。  
accept（）返回服务器和客户端之间打开的连接以及客户端的地址。 该连接实际上是另一个端口上的不同套接字（由内核分配）。 使用recv（）从连接中读取数据，并使用sendall（）进行传输。  
完成与客户端的通信后，需要使用close（）清除连接。 此示例使用try：finally块来确保始终调用close（），即使出现错误也是如此。

#### Echo Client

客户端程序设置其套接字的方式与服务器的方式不同。 它使用connect（）将套接字直接连接到远程地址，而不是绑定到端口并进行侦听。

```python
#socket_echo_client.py
import socket
import sys

# Create a TCP/IP socket
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Connect the socket to the port where the server is listening
server_address = ('localhost', 10000)
print('connecting to {} port {}'.format(*server_address))
sock.connect(server_address)

try:

    # Send data
    message = b'This is the message.  It will be repeated.'
    print('sending {!r}'.format(message))
    sock.sendall(message)

    # Look for the response
    amount_received = 0
    amount_expected = len(message)

    while amount_received < amount_expected:
        data = sock.recv(16)
        amount_received += len(data)
        print('received {!r}'.format(data))

finally:
    print('closing socket')
    sock.close()
```
建立连接后，可以使用sendall（）通过套接字发送数据，并使用recv（）接收数据，就像在服务器中一样。 发送整个消息并收到副本后，将关闭套接字以释放端口。

#### Client and Server Together

客户端和服务器应该在单独的终端窗口中运行，以便它们可以相互通信。 服务器输出显示传入的连接和数据，以及发送回客户端的响应。

```
$ python3 socket_echo_server.py
starting up on localhost port 10000
waiting for a connection
connection from ('127.0.0.1', 65141)
received b'This is the mess'
sending data back to the client
received b'age.  It will be'
sending data back to the client
received b' repeated.'
sending data back to the client
received b''
no data from ('127.0.0.1', 65141)
waiting for a connection
```
客户端输出显示传出消息和来自服务器的响应。

```
$ python3 socket_echo_client.py
connecting to localhost port 10000
sending b'This is the message.  It will be repeated.'
received b'This is the mess'
received b'age.  It will be'
received b' repeated.'
closing socket
```

#### Easy Client Connections

通过使用便捷函数create_connection（）连接到服务器，TCP/IP客户端可以节省一些步骤。 该函数接受一个参数，一个包含服务器地址的双值元组，并派生出用于连接的最佳地址。

```python
#socket_echo_client_easy.py
import socket
import sys

def get_constants(prefix):
    """Create a dictionary mapping socket module
    constants to their names.
    """
    return {
        getattr(socket, n): n
        for n in dir(socket)
        if n.startswith(prefix)
    }

families = get_constants('AF_')
types = get_constants('SOCK_')
protocols = get_constants('IPPROTO_')

# Create a TCP/IP socket
sock = socket.create_connection(('localhost', 10000))

print('Family  :', families[sock.family])
print('Type    :', types[sock.type])
print('Protocol:', protocols[sock.proto])
print()

try:

    # Send data
    message = b'This is the message.  It will be repeated.'
    print('sending {!r}'.format(message))
    sock.sendall(message)

    amount_received = 0
    amount_expected = len(message)

    while amount_received < amount_expected:
        data = sock.recv(16)
        amount_received += len(data)
        print('received {!r}'.format(data))

finally:
    print('closing socket')
    sock.close()
```
create_connection（）使用getaddrinfo（）查找候选连接参数，并返回第一个连接创建成功使用的配置打开的套接字。 可以检查family，type和proto属性以确定要返回的套接字的类型。

```
$ python3 socket_echo_client_easy.py
Family  : AF_INET
Type    : SOCK_STREAM
Protocol: IPPROTO_TCP

sending b'This is the message.  It will be repeated.'
received b'This is the mess'
received b'age.  It will be'
received b' repeated.'
closing socket
```

#### Choosing an Address for Listening

将服务器绑定到正确的地址非常重要，以便客户端可以与之通信。 前面的示例都使用“localhost”作为IP地址，这只允许同一服务器上运行的客户端连接。 使用服务器的公共地址（例如gethostname（）返回的值）以允许其他主机进行连接。 此示例修改echo服务器以侦听通过命令行参数指定的地址。

```python
#socket_echo_server_explicit.py
import socket
import sys

# Create a TCP/IP socket
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Bind the socket to the address given on the command line
server_name = sys.argv[1]
server_address = (server_name, 10000)
print('starting up on {} port {}'.format(*server_address))
sock.bind(server_address)
sock.listen(1)

while True:
    print('waiting for a connection')
    connection, client_address = sock.accept()
    try:
        print('client connected:', client_address)
        while True:
            data = connection.recv(16)
            print('received {!r}'.format(data))
            if data:
                connection.sendall(data)
            else:
                break
    finally:
        connection.close()
```
在测试服务器之前，需要对客户端程序进行类似的修改。

```python
#socket_echo_client_explicit.py
import socket
import sys

# Create a TCP/IP socket
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Connect the socket to the port on the server
# given by the caller
server_address = (sys.argv[1], 10000)
print('connecting to {} port {}'.format(*server_address))
sock.connect(server_address)

try:

    message = b'This is the message.  It will be repeated.'
    print('sending {!r}'.format(message))
    sock.sendall(message)

    amount_received = 0
    amount_expected = len(message)
    while amount_received < amount_expected:
        data = sock.recv(16)
        amount_received += len(data)
        print('received {!r}'.format(data))

finally:
    sock.close()
```
使用参数hubert启动服务器后，netstat命令会显示它正在侦听指定主机的地址。

```
$ host hubert.hellfly.net

hubert.hellfly.net has address 10.9.0.6

$ netstat -an | grep 10000

Active Internet connections (including servers)
Proto Recv-Q Send-Q  Local Address          Foreign Address        (state)
...
tcp4       0      0  10.9.0.6.10000         *.*                    LISTEN
...
```
在另一台主机上运行客户端，将hubert.hellfly.net作为运行服务器的主机传递，生成：

```
$ hostname

apu

$ python3 ./socket_echo_client_explicit.py hubert.hellfly.net
connecting to hubert.hellfly.net port 10000
sending b'This is the message.  It will be repeated.'
received b'This is the mess'
received b'age.  It will be'
received b' repeated.'
```
服务器输出是：

```
$ python3 socket_echo_server_explicit.py hubert.hellfly.net
starting up on hubert.hellfly.net port 10000
waiting for a connection
client connected: ('10.9.0.10', 33139)
received b''
waiting for a connection
client connected: ('10.9.0.10', 33140)
received b'This is the mess'
received b'age.  It will be'
received b' repeated.'
received b''
waiting for a connection
```
许多服务器具有多个网络接口，因此具有多个IP地址。 不是运行绑定到每个IP地址的服务的单独副本，而是使用特殊地址`INADDR_ANY`同时侦听所有地址。 尽管socket为`INADDR_ANY`定义了一个常量，但它是一个整数值，必须先将其转换为点符号字符串地址，然后才能传递给bind（）。 作为快捷方式，使用“0.0.0.0”或空字符串（''）而不是进行转换。

```python
#socket_echo_server_any.py
import socket
import sys

# Create a TCP/IP socket
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Bind the socket to the address given on the command line
server_address = ('', 10000)
sock.bind(server_address)
print('starting up on {} port {}'.format(*sock.getsockname()))
sock.listen(1)

while True:
    print('waiting for a connection')
    connection, client_address = sock.accept()
    try:
        print('client connected:', client_address)
        while True:
            data = connection.recv(16)
            print('received {!r}'.format(data))
            if data:
                connection.sendall(data)
            else:
                break
    finally:
        connection.close()
```
要查看套接字使用的实际地址，请调用其getsockname（）方法。 启动服务后，再次运行netstat会显示它正在侦听任何地址上的传入连接。

```
$ netstat -an

Active Internet connections (including servers)
Proto Recv-Q Send-Q  Local Address    Foreign Address  (state)
...
tcp4       0      0  *.10000          *.*              LISTEN
...
```

### User Datagram Client and Server

用户数据报协议（UDP）与TCP/IP的工作方式不同。 TCP是面向流的协议，确保所有数据以正确的顺序传输，UDP是面向消息的协议。 UDP不需要长期存在的连接，因此设置UDP套接字更简单一些。 另一方面，UDP消息必须适合单个数据报（对于IPv4，这意味着它们只能容纳65,507个字节，因为65,535字节的数据包也包含头信息）并且数据传送没有与TCP一样的保障。

#### Echo Server

由于没有连接本身，服务器不需要监听和接受连接。 它只需要使用bind（）将其套接字与端口关联，然后等待单个的消息。

```python
#socket_echo_server_dgram.py
import socket
import sys

# Create a UDP socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

# Bind the socket to the port
server_address = ('localhost', 10000)
print('starting up on {} port {}'.format(*server_address))
sock.bind(server_address)

while True:
    print('\nwaiting to receive message')
    data, address = sock.recvfrom(4096)

    print('received {} bytes from {}'.format(
        len(data), address))
    print(data)

    if data:
        sent = sock.sendto(data, address)
        print('sent {} bytes back to {}'.format(
            sent, address))
```
使用recvfrom（）从套接字读取消息，它返回数据以及客户端的地址。

#### Echo Client

UDP echo客户端与服务器类似，但不使用bind（）将其套接字附加到地址。 它使用sendto（）将其消息直接传递给服务器，并使用recvfrom（）来接收响应。

```python
#socket_echo_client_dgram.py
import socket
import sys

# Create a UDP socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

server_address = ('localhost', 10000)
message = b'This is the message.  It will be repeated.'

try:

    # Send data
    print('sending {!r}'.format(message))
    sent = sock.sendto(message, server_address)

    # Receive response
    print('waiting to receive')
    data, server = sock.recvfrom(4096)
    print('received {!r}'.format(data))

finally:
    print('closing socket')
    sock.close()
```

#### Client and Server Together

运行服务器会产生：

```
$ python3 socket_echo_server_dgram.py
starting up on localhost port 10000

waiting to receive message
received 42 bytes from ('127.0.0.1', 57870)
b'This is the message.  It will be repeated.'
sent 42 bytes back to ('127.0.0.1', 57870)

waiting to receive message
```
客户端的输出是：

```
$ python3 socket_echo_client_dgram.py
sending b'This is the message.  It will be repeated.'
waiting to receive
received b'This is the message.  It will be repeated.'
closing socket
```

### Unix域套接字

从程序员的角度来看，使用Unix域套接字和TCP/IP套接字有两个本质区别。 首先，套接字的地址是文件系统上的路径，而不是包含服务器名称和端口的元组。 其次，在套接字关闭后，在文件系统中创建的表示套接字的节点仍然存在，并且每次服务器启动时都需要删除。 通过在设置部分进行一些更改，可以更新之前的echo服务器示例以使用UDS。  
需要使用地址族AF_UNIX创建套接字。 绑定套接字和管理传入连接的工作方式与TCP/IP套接字相同。

```python
#socket_echo_server_uds.py
import socket
import sys
import os

server_address = './uds_socket'

# Make sure the socket does not already exist
try:
    os.unlink(server_address)
except OSError:
    if os.path.exists(server_address):
        raise

# Create a UDS socket
sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)

# Bind the socket to the address
print('starting up on {}'.format(server_address))
sock.bind(server_address)

# Listen for incoming connections
sock.listen(1)

while True:
    # Wait for a connection
    print('waiting for a connection')
    connection, client_address = sock.accept()
    try:
        print('connection from', client_address)

        # Receive the data in small chunks and retransmit it
        while True:
            data = connection.recv(16)
            print('received {!r}'.format(data))
            if data:
                print('sending data back to the client')
                connection.sendall(data)
            else:
                print('no data from', client_address)
                break

    finally:
        # Clean up the connection
        connection.close()
```
还需要修改客户端设置以使用UDS。 它应该假定套接字的文件系统节点存在，因为服务器通过绑定到该地址来创建它。 发送和接收数据在UDS客户端中的工作方式与之前的TCP/IP客户端相同。

```python
#socket_echo_client_uds.py
import socket
import sys

# Create a UDS socket
sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)

# Connect the socket to the port where the server is listening
server_address = './uds_socket'
print('connecting to {}'.format(server_address))
try:
    sock.connect(server_address)
except socket.error as msg:
    print(msg)
    sys.exit(1)

try:

    # Send data
    message = b'This is the message.  It will be repeated.'
    print('sending {!r}'.format(message))
    sock.sendall(message)

    amount_received = 0
    amount_expected = len(message)

    while amount_received < amount_expected:
        data = sock.recv(16)
        amount_received += len(data)
        print('received {!r}'.format(data))

finally:
    print('closing socket')
    sock.close()

```
程序输出大致相同，并对地址信息进行适当更新。 服务器显示收到的消息并将其发送回客户端。

```
$ python3 socket_echo_server_uds.py
starting up on ./uds_socket
waiting for a connection
connection from
received b'This is the mess'
sending data back to the client
received b'age.  It will be'
sending data back to the client
received b' repeated.'
sending data back to the client
received b''
no data from
waiting for a connection
```
客户端一次性发送消息，并以递增方式接收部分消息。

```
$ python3 socket_echo_client_uds.py
connecting to ./uds_socket
sending b'This is the message.  It will be repeated.'
received b'This is the mess'
received b'age.  It will be'
received b' repeated.'
closing socket
```

#### 权限

由于UDS套接字由文件系统上的节点表示，因此可以使用标准文件系统权限来控制对服务器的访问。

```
$ ls -l ./uds_socket

srwxr-xr-x  1 dhellmann  dhellmann  0 Aug 21 11:19 uds_socket

$ sudo chown root ./uds_socket

$ ls -l ./uds_socket

srwxr-xr-x  1 root  dhellmann  0 Aug 21 11:19 uds_socket
```
现在以root之外的用户身份运行客户端会导致错误，因为该进程没有打开套接字的权限。

```
$ python3 socket_echo_client_uds.py

connecting to ./uds_socket
[Errno 13] Permission denied
```

#### 父子进程间通信

socketpair（）函数对于在Unix下为进程间通信设置UDS套接字很有用。 它创建了一对连接的套接字，可用于在子进程fork后在父进程和子进程之间进行通信。

```python
#socket_socketpair.py
import socket
import os

parent, child = socket.socketpair()

pid = os.fork()

if pid:
    print('in parent, sending message')
    child.close()
    parent.sendall(b'ping')
    response = parent.recv(1024)
    print('response from child:', response)
    parent.close()

else:
    print('in child, waiting for message')
    parent.close()
    message = child.recv(1024)
    print('message from parent:', message)
    child.sendall(b'pong')
    child.close()
```
默认情况下，会创建一个UDS套接字，但调用者也可以传递地址族，套接字类型甚至协议选项来控制套接字的创建方式。

```
$ python3 -u socket_socketpair.py

in parent, sending message
in child, waiting for message
message from parent: b'ping'
response from child: b'pong'
```

### 多播(组播)

点对点连接处理大量通信需求，但随着直接连接数量的增加，在多个对等体之间传递相同的信息变得具有挑战性。分别向每个接收者发送消息会消耗额外的处理时间和带宽，这对于诸如流视频或音频之类的应用来说可能是个问题。使用多播一次向多个端点传递消息可以实现更高的效率，因为网络基础结构可确保将数据包传递给所有接收者。  
多播消息总是使用UDP发送，因为TCP假定一对通信系统。多播地址（称为多播组）是为多播通信保留的常规IPv4地址范围（224.0.0.0到230.255.255.255）的子集。这些地址由网络路由器和交换机专门处理，因此发送到该组的消息可以通过Internet分发给已加入该组的所有接收者。
>注意:某些托管交换机和路由器默认禁用多播流量。如果您在使用示例程序时遇到问题，请检查网络硬件设置。

#### Sending Multicast Messages

这个修改后的echo客户端将向组播组发送消息，然后报告它收到的所有响应。 由于无法知道预期会有多少响应，因此它会使用套接字上的超时值来避免在等待答案时无限期地阻塞。  
套接字还需要配置消息的生存时间值（TTL）。 TTL控制接收数据包的网络数量。 使用`IP_MULTICAST_TTL`选项和setsockopt（）设置TTL。 缺省值1表示路由器不会将数据包转发到当前网段之外。 该值最大可达255，应打包为单个字节。

```python
#socket_multicast_sender.py
import socket
import struct
import sys

message = b'very important data'
multicast_group = ('224.3.29.71', 10000)

# Create the datagram socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

# Set a timeout so the socket does not block
# indefinitely when trying to receive data.
sock.settimeout(0.2)

# Set the time-to-live for messages to 1 so they do not
# go past the local network segment.
ttl = struct.pack('b', 1)
sock.setsockopt(socket.IPPROTO_IP, socket.IP_MULTICAST_TTL, ttl)

try:

    # Send data to the multicast group
    print('sending {!r}'.format(message))
    sent = sock.sendto(message, multicast_group)

    # Look for responses from all recipients
    while True:
        print('waiting to receive')
        try:
            data, server = sock.recvfrom(16)
        except socket.timeout:
            print('timed out, no more responses')
            break
        else:
            print('received {!r} from {}'.format(
                data, server))

finally:
    print('closing socket')
    sock.close()
```
发送者的其余部分看起来像UDP echo客户端，除了它需要多个响应，因此使用循环调用recvfrom（）直到它超时。

#### Receiving Multicast Messages

建立多播接收器的第一步是创建UDP套接字。 创建常规套接字并绑定到端口后，可以使用setsockopt（）的`IP_ADD_MEMBERSHIP`选项将其添加到多播组。 选项值是多播组地址后跟服务器应侦听流量的网络接口(由其IP地址标识)的8字节打包表示。 在这个例子中，接收器使用`INADDR_ANY`侦听所有接口。

```python
#socket_multicast_receiver.py
import socket
import struct
import sys

multicast_group = '224.3.29.71'
server_address = ('', 10000)

# Create the socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

# Bind to the server address
sock.bind(server_address)

# Tell the operating system to add the socket to
# the multicast group on all interfaces.
group = socket.inet_aton(multicast_group)
mreq = struct.pack('4sL', group, socket.INADDR_ANY)
sock.setsockopt(
    socket.IPPROTO_IP,
    socket.IP_ADD_MEMBERSHIP,
    mreq)

# Receive/respond loop
while True:
    print('\nwaiting to receive message')
    data, address = sock.recvfrom(1024)

    print('received {} bytes from {}'.format(
        len(data), address))
    print(data)

    print('sending acknowledgement to', address)
    sock.sendto(b'ack', address)
```
接收器的主循环就像常规的UDP echo服务器一样。

#### Example Output

此示例显示在两个不同主机上运行的多播接收器。 A的地址为192.168.1.13，B的地址为192.168.1.14。

```
[A]$ python3 socket_multicast_receiver.py

waiting to receive message
received 19 bytes from ('192.168.1.14', 62650)
b'very important data'
sending acknowledgement to ('192.168.1.14', 62650)

waiting to receive message

[B]$ python3 source/socket/socket_multicast_receiver.py

waiting to receive message
received 19 bytes from ('192.168.1.14', 64288)
b'very important data'
sending acknowledgement to ('192.168.1.14', 64288)

waiting to receive message
```
The sender is running on host B.

```
[B]$ python3 socket_multicast_sender.py
sending b'very important data'
waiting to receive
received b'ack' from ('192.168.1.14', 10000)
waiting to receive
received b'ack' from ('192.168.1.13', 10000)
waiting to receive
timed out, no more responses
closing socket
```
消息被发送一次，并且从主机A和B中的每一个接收到两个外发消息的确认。

### 发送二进制数据

套接字传输字节流。 这些字节可以包含编码为字节的文本消息，如前面的示例所示，或者它们可以由二进制数据组成，这些二进制数据已经打包到struct的缓冲区中以准备传输。  
此客户端程序将整数，两个字符的字符串和浮点值编码为可传递到套接字以进行传输的字节序列。

```python
#socket_binary_client.py
import binascii
import socket
import struct
import sys

# Create a TCP/IP socket
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_address = ('localhost', 10000)
sock.connect(server_address)

values = (1, b'ab', 2.7)
packer = struct.Struct('I 2s f')
packed_data = packer.pack(*values)

print('values =', values)

try:
    # Send data
    print('sending {!r}'.format(binascii.hexlify(packed_data)))
    sock.sendall(packed_data)
finally:
    print('closing socket')
    sock.close()
```
在两个系统之间发送多字节二进制数据时，重要的是要确保连接的两端都知道字节的顺序以及如何将它们组装回本地体系结构的正确顺序。 服务器程序使用相同的Struct说明符来解压缩它接收的字节，以便按正确的顺序解释它们。

```python
#socket_binary_server.py
import binascii
import socket
import struct
import sys

# Create a TCP/IP socket
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_address = ('localhost', 10000)
sock.bind(server_address)
sock.listen(1)

unpacker = struct.Struct('I 2s f')

while True:
    print('\nwaiting for a connection')
    connection, client_address = sock.accept()
    try:
        data = connection.recv(unpacker.size)
        print('received {!r}'.format(binascii.hexlify(data)))

        unpacked_data = unpacker.unpack(data)
        print('unpacked:', unpacked_data)

    finally:
        connection.close()
```
运行客户端产生输出：

```
$ python3 source/socket/socket_binary_client.py
values = (1, b'ab', 2.7)
sending b'0100000061620000cdcc2c40'
closing socket
```
服务端显示他收到的值：

```
$ python3 socket_binary_server.py

waiting for a connection
received b'0100000061620000cdcc2c40'
unpacked: (1, b'ab', 2.700000047683716)

waiting for a connection
```
浮点值在打包和解包时会丢失一些精度，但是别的数据会按预期传输。 要记住的一件事是，根据整数的值，将它转换为文本然后传输可能更有效，而不是使用struct。 整数1在表示为字符串时使用一个字节，但在打包到结构中时使用四个字节。

### 非阻塞通信和超时

默认情况下，配置套接字以便发送或接收数据块，在套接字准备好之前停止程序执行。调用send（）等待缓冲区空间可用于传出数据，调用recv（）等待另一个程序发送可读取的数据。这种形式的I/O操作很容易理解，但如果两个程序最终都在等待另一个发送或接收数据，则可能导致低效操作甚至死锁。  
有几种方法可以解决这种情况。一种是使用单独的线程与每个套接字进行通信。但是，这可能会引入其他复杂性，即线程之间的通信。另一个选择是将套接字更改为不阻塞，如果尚未准备好处理操作，则立即返回。使用setblocking（）方法更改套接字的阻塞标志。默认值为1，表示阻塞。传递值0会关闭阻塞。如果套接字已关闭阻塞并且尚未准备好进行操作，则会引发socket.error。  
妥协的解决方案是为套接字操作设置超时值。使用settimeout（）将套接字的超时更改为浮点值，该值表示在确定套接字尚未准备好进行操作之前阻塞的秒数。超时到期时，会引发timeout异常。

## selectors — I/O 多路复用抽象

目的：基于select模块为 I/O 多路复用提供与平台无关的抽象。  
selectors模块在select模块中的特定平台 I/O 监视功能之上提供独立于平台的抽象层。

### 操作模型

selectors中的API是基于事件的，类似于select中的poll（）。 有几种实现方式，模块自动设置别名DefaultSelector以引用当前系统配置中最高效的一个。  
选择器对象提供了用于指定在套接字上查找哪些事件的方法，然后让调用者以与平台无关的方式等待事件。 注册兴趣事件会创建一个SelectorKey，它包含套接字、有关感兴趣事件的信息以及可选的应用程序数据。 选择器的所有者调用其select（）方法来检测事件。 返回值是一系列key对象和一个指示已发生事件的位掩码。 使用选择器的程序应重复调用select（），然后适当地处理事件。

### Echo Server

下面的echo服务器示例使用SelectorKey中的应用程序数据来注册要在新事件上调用的回调函数。 主循环从key获取回调并将套接字和事件掩码传递给它。 当服务器启动时，它会在主服务套接字上注册要为read事件调用的accept（）函数。 接受连接会产生一个新的套接字，然后注册其read事件的回调为read（）函数。

```python
#selectors_echo_server.py
import selectors
import socket

mysel = selectors.DefaultSelector()
keep_running = True

def read(connection, mask):
    "Callback for read events"
    global keep_running

    client_address = connection.getpeername()
    print('read({})'.format(client_address))
    data = connection.recv(1024)
    if data:
        # A readable client socket has data
        print('  received {!r}'.format(data))
        connection.sendall(data)
    else:
        # Interpret empty result as closed connection
        print('  closing')
        mysel.unregister(connection)
        connection.close()
        # Tell the main loop to stop
        keep_running = False


def accept(sock, mask):
    "Callback for new connections"
    new_connection, addr = sock.accept()
    print('accept({})'.format(addr))
    new_connection.setblocking(False)
    mysel.register(new_connection, selectors.EVENT_READ, read)


server_address = ('localhost', 10000)
print('starting up on {} port {}'.format(*server_address))
server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.setblocking(False)
server.bind(server_address)
server.listen(5)

mysel.register(server, selectors.EVENT_READ, accept)

while keep_running:
    print('waiting for I/O')
    for key, mask in mysel.select(timeout=1):
        callback = key.data
        callback(key.fileobj, mask)  #套接字和事件掩码

print('shutting down')
mysel.close()
```
当read（）没有从套接字接收数据时，它会将读取事件解释为连接的另一端被关闭而不是发送数据。 它从选择器中删除套接字并将其关闭。 为了避免无限循环，此服务器在完成与单个客户端的通信后也会自行关闭。

### Echo Client

下面的echo客户端示例处理主循环中的所有 I/O 事件，而不是使用回调。 它设置选择器以报告套接字上的读取事件，并报告套接字何时准备好发送数据。 因为它正在查看两种类型的事件，所以客户端必须通过检查掩码值来检查发生的事件。 发送完所有传出数据后，它会将选择器配置更改为仅在有数据要读取时进行报告。

```python
#selectors_echo_client.py
import selectors
import socket

mysel = selectors.DefaultSelector()
keep_running = True
outgoing = [
    b'It will be repeated.',
    b'This is the message.  ',
]
bytes_sent = 0
bytes_received = 0

# Connecting is a blocking operation, so call setblocking()
# after it returns.
server_address = ('localhost', 10000)
print('connecting to {} port {}'.format(*server_address))
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(server_address)
sock.setblocking(False)

# Set up the selector to watch for when the socket is ready
# to send data as well as when there is data to read.
mysel.register(
    sock,
    selectors.EVENT_READ | selectors.EVENT_WRITE,
)

while keep_running:
    print('waiting for I/O')
    for key, mask in mysel.select(timeout=1):
        connection = key.fileobj
        client_address = connection.getpeername()
        print('client({})'.format(client_address))

        if mask & selectors.EVENT_READ:
            print('  ready to read')
            data = connection.recv(1024)
            if data:
                # A readable client socket has data
                print('  received {!r}'.format(data))
                bytes_received += len(data)

            # Interpret empty result as closed connection,
            # and also close when we have received a copy
            # of all of the data sent.
            keep_running = not (
                data or
                (bytes_received and
                 (bytes_received == bytes_sent))
            )

        if mask & selectors.EVENT_WRITE:
            print('  ready to write')
            if not outgoing:
                # We are out of messages, so we no longer need to
                # write anything. Change our registration to let
                # us keep reading responses from the server.
                print('  switching to read-only')
                mysel.modify(sock, selectors.EVENT_READ)
            else:
                # Send the next message.
                next_msg = outgoing.pop()
                print('  sending {!r}'.format(next_msg))
                sock.sendall(next_msg)
                bytes_sent += len(next_msg)

print('shutting down')
mysel.unregister(connection)
connection.close()
mysel.close()
```
客户端跟踪它发送的数据量以及收到的数量。 当这些值匹配且非零时，客户端退出处理循环并通过从选择器中移除套接字并关闭套接字和选择器来干净地关闭。

### Server and Client Together

客户端和服务器应该在单独的终端窗口中运行，以便它们可以相互通信。 服务器输出显示传入的连接和数据，以及发送回客户端的响应。

```
$ python3 source/selectors/selectors_echo_server.py
starting up on localhost port 10000
waiting for I/O
waiting for I/O
accept(('127.0.0.1', 59850))
waiting for I/O
read(('127.0.0.1', 59850))
  received b'This is the message.  It will be repeated.'
waiting for I/O
read(('127.0.0.1', 59850))
  closing
shutting down
```
客户端输出显示传出消息和来自服务器的响应。

```
$ python3 source/selectors/selectors_echo_client.py
connecting to localhost port 10000
waiting for I/O
client(('127.0.0.1', 10000))
  ready to write
  sending b'This is the message.  '
waiting for I/O
client(('127.0.0.1', 10000))
  ready to write
  sending b'It will be repeated.'
waiting for I/O
client(('127.0.0.1', 10000))
  ready to write
  switching to read-only
waiting for I/O
client(('127.0.0.1', 10000))
  ready to read
  received b'This is the message.  It will be repeated.'
shutting down
```

## select — Wait for I/O Efficiently

目的：等待输入或输出通道准备就绪的通知。  
select 模块提供对特定于平台的 I/O 监视功能的访问。 最便携的接口是POSIX函数select（），可在Unix和Windows上使用。 该模块还包括poll（），一个仅限Unix的API，以及几个仅适用于Unix特定变体的选项。
>注意:  新的selectors模块提供了基于select API之上的更高级别的接口。 使用selectors构建可移植代码更容易，因此除非以某种方式需要select提供的低级API，否则请使用selectors。

### Using select()

Python的select（）函数是底层操作系统实现的直接接口。 它监视套接字，打开的文件和管道（任何有一个fileno（）方法用于返回有效文件描述符的对象），直到它们变得可读或可写或发生通信错误。 select（）使得更容易同时监视多个连接，并且比使用套接字超时在Python中编写轮询循环更有效，因为监视发生在操作系统网络层而不是解释器。
>注意：将select（）用于Python文件对象适用于Unix，但在Windows下不受支持。

可以扩展socket 小节中的echo服务器示例，以使用select（）一次监视多个连接。 新版本首先创建一个非阻塞的TCP/IP套接字并将其配置为侦听地址。

```python
#select_echo_server.py
import select
import socket
import sys
import queue

# Create a TCP/IP socket
server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.setblocking(0)

# Bind the socket to the port
server_address = ('localhost', 10000)
print('starting up on {} port {}'.format(*server_address),
      file=sys.stderr)
server.bind(server_address)

# Listen for incoming connections
server.listen(5)
```
select（）的参数是三个包含要监视的通信通道的列表。 第一个是需要检查是否有输入数据可以读取的对象列表，第二个是需要检查是否输出数据可以写入的对象列表，第三个是需要检测错误的对象列表（通常是输入和输出通道对象的组合）。 服务器的下一步是设置包含要传递给select（）的输入源和输出目标的列表。

```python
# Sockets from which we expect to read
inputs = [server]

# Sockets to which we expect to write
outputs = []
```
服务器主循环将连接添加到这些列表中或从这些列表中删除。 由于此版本的服务器将在发送任何数据之前等待套接字变为可写（而不是立即发送回复），因此每个输出连接都需要一个队列作为缓冲区，通过它发送数据。

```python
# Outgoing message queues (socket:Queue)
message_queues = {}
```
服务器程序循环的主要部分，调用select（）来阻塞并等待网络活动。

```python
while inputs:

    # Wait for at least one of the sockets to be
    # ready for processing
    print('waiting for the next event', file=sys.stderr)
    readable, writable, exceptional = select.select(inputs,
                                                    outputs,
                                                    inputs)
```
select()返回三个新列表，包含传入列表内容的子集。readable列表中的所有套接字都有缓冲的传入数据，可供读取。 writeable列表中的所有套接字都在其缓冲区中具有可用空间并且可以写入。 exceptional列表中返回的套接字有一个错误（“异常条件”的实际定义取决于平台）。  
“可读”套接字代表三种可能的情况。 如果套接字是用于监听连接的主“服务器”套接字，那么“可读”条件意味着它已准备好接受另一个传入连接。 除了将新连接添加到要监视的输入列表之外，这里还将客户端套接字设置为不阻塞。

```python
    # Handle inputs
    for s in readable:

        if s is server:
            # A "readable" socket is ready to accept a connection
            connection, client_address = s.accept()
            print('  connection from', client_address,
                  file=sys.stderr)
            connection.setblocking(0)
            inputs.append(connection)

            # Give the connection a queue for data
            # we want to send
            message_queues[connection] = queue.Queue()
```
下一种情况是已建立连接的客户端发送了数据。 使用recv（）读取数据，然后将其放在队列中，以便可以通过套接字发送回客户端。

```python
        else:
            data = s.recv(1024)
            if data:
                # A readable client socket has data
                print('  received {!r} from {}'.format(
                    data, s.getpeername()), file=sys.stderr,
                )
                message_queues[s].put(data)
                # Add output channel for response
                if s not in outputs:
                    outputs.append(s)
```
没有可用数据的可读套接字，因为客户端已断开连接，并且流已准备好关闭。

```python
            else:
                # Interpret empty result as closed connection
                print('  closing', client_address,
                      file=sys.stderr)
                # Stop listening for input on the connection
                if s in outputs:
                    outputs.remove(s)
                inputs.remove(s)
                s.close()

                # Remove message queue
                del message_queues[s]
```
可写连接的情况较少。 如果连接的队列中有数据，则发送下一条消息。 否则，将从输出连接列表中删除连接，以便下次通过循环select（）时不指示套接字已准备好发送数据。

```python
    # Handle outputs
    for s in writable:
        try:
            next_msg = message_queues[s].get_nowait()
        except queue.Empty:
            # No messages waiting so stop checking
            # for writability.
            print('  ', s.getpeername(), 'queue empty',
                  file=sys.stderr)
            outputs.remove(s)
        else:
            print('  sending {!r} to {}'.format(next_msg,
                                                s.getpeername()),
                  file=sys.stderr)
            s.send(next_msg)
```
最后，如果套接字出错，则关闭。

```python
    # Handle "exceptional conditions"
    for s in exceptional:
        print('exception condition on', s.getpeername(),
              file=sys.stderr)
        # Stop listening for input on the connection
        inputs.remove(s)
        if s in outputs:
            outputs.remove(s)
        s.close()

        # Remove message queue
        del message_queues[s]
```
示例客户端程序使用两个套接字来演示带有select（）的服务器如何同时管理多个连接。 客户端首先将每个TCP/IP套接字连接到服务器。

```python
#select_echo_multiclient.py
import socket
import sys

messages = [
    'This is the message. ',
    'It will be sent ',
    'in parts.',
]
server_address = ('localhost', 10000)

# Create a TCP/IP socket
socks = [
    socket.socket(socket.AF_INET, socket.SOCK_STREAM),
    socket.socket(socket.AF_INET, socket.SOCK_STREAM),
]

# Connect the socket to the port where the server is listening
print('connecting to {} port {}'.format(*server_address),
      file=sys.stderr)
for s in socks:
    s.connect(server_address)
```
然后它通过每个套接字一次发送一条消息，并在写入新数据后读取所有可用的响应。

```python
for message in messages:
    outgoing_data = message.encode()

    # Send messages on both sockets
    for s in socks:
        print('{}: sending {!r}'.format(s.getsockname(),
                                        outgoing_data),
              file=sys.stderr)
        s.send(outgoing_data)

    # Read responses on both sockets
    for s in socks:
        data = s.recv(1024)
        print('{}: received {!r}'.format(s.getsockname(),
                                         data),
              file=sys.stderr)
        if not data:
            print('closing socket', s.getsockname(),
                  file=sys.stderr)
            s.close()
```
在一个窗口中运行服务器，在另一个窗口中运行客户端 输出将如下所示，具有不同的端口号。

```
$ python3 select_echo_server.py
starting up on localhost port 10000
waiting for the next event
  connection from ('127.0.0.1', 61003)
waiting for the next event
  connection from ('127.0.0.1', 61004)
waiting for the next event
  received b'This is the message. ' from ('127.0.0.1', 61003)
  received b'This is the message. ' from ('127.0.0.1', 61004)
waiting for the next event
  sending b'This is the message. ' to ('127.0.0.1', 61003)
  sending b'This is the message. ' to ('127.0.0.1', 61004)
waiting for the next event
   ('127.0.0.1', 61003) queue empty
   ('127.0.0.1', 61004) queue empty
waiting for the next event
  received b'It will be sent ' from ('127.0.0.1', 61003)
  received b'It will be sent ' from ('127.0.0.1', 61004)
waiting for the next event
  sending b'It will be sent ' to ('127.0.0.1', 61003)
  sending b'It will be sent ' to ('127.0.0.1', 61004)
waiting for the next event
   ('127.0.0.1', 61003) queue empty
   ('127.0.0.1', 61004) queue empty
waiting for the next event
  received b'in parts.' from ('127.0.0.1', 61003)
waiting for the next event
  received b'in parts.' from ('127.0.0.1', 61004)
  sending b'in parts.' to ('127.0.0.1', 61003)
waiting for the next event
   ('127.0.0.1', 61003) queue empty
  sending b'in parts.' to ('127.0.0.1', 61004)
waiting for the next event
   ('127.0.0.1', 61004) queue empty
waiting for the next event
  closing ('127.0.0.1', 61004)
  closing ('127.0.0.1', 61004)
waiting for the next event
```
客户端输出显示使用两个套接字发送和接收的数据。

```
$ python3 select_echo_multiclient.py
connecting to localhost port 10000
('127.0.0.1', 61003): sending b'This is the message. '
('127.0.0.1', 61004): sending b'This is the message. '
('127.0.0.1', 61003): received b'This is the message. '
('127.0.0.1', 61004): received b'This is the message. '
('127.0.0.1', 61003): sending b'It will be sent '
('127.0.0.1', 61004): sending b'It will be sent '
('127.0.0.1', 61003): received b'It will be sent '
('127.0.0.1', 61004): received b'It will be sent '
('127.0.0.1', 61003): sending b'in parts.'
('127.0.0.1', 61004): sending b'in parts.'
('127.0.0.1', 61003): received b'in parts.'
('127.0.0.1', 61004): received b'in parts.'
```

### 带有超时的非阻塞I/O

select（）还接受一个可选的第四个参数，该参数是在没有通道变为活动状态时中断监视之前等待的秒数。 使用超时值可让主程序调用select（）作为更大处理循环的一部分，在检查网络输入之间执行其他操作。  
超时到期时，select（）返回三个空列表。 更新服务器示例以使用超时需要将额外参数添加到select（）调用并在select（）返回后处理空列表。

```python
#select_echo_server_timeout.py
    readable, writable, exceptional = select.select(inputs,
                                                    outputs,
                                                    inputs,
                                                    timeout)

    if not (readable or writable or exceptional):
        print('  timed out, do some other work here',
              file=sys.stderr)
        continue
```
客户端程序的这个“慢”版本在发送每个消息后暂停，以模拟传输中的延迟或其他延迟。

```python
#select_echo_slow_client.py
import socket
import sys
import time

# Create a TCP/IP socket
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Connect the socket to the port where the server is listening
server_address = ('localhost', 10000)
print('connecting to {} port {}'.format(*server_address),
      file=sys.stderr)
sock.connect(server_address)

time.sleep(1)

messages = [
    'Part one of the message.',
    'Part two of the message.',
]
amount_expected = len(''.join(messages))

try:

    # Send data
    for message in messages:
        data = message.encode()
        print('sending {!r}'.format(data), file=sys.stderr)
        sock.sendall(data)
        time.sleep(1.5)

    # Look for the response
    amount_received = 0

    while amount_received < amount_expected:
        data = sock.recv(16)
        amount_received += len(data)
        print('received {!r}'.format(data), file=sys.stderr)

finally:
    print('closing socket', file=sys.stderr)
    sock.close()
```
使用慢速客户端运行新服务器会产生：

```
$ python3 select_echo_server_timeout.py
starting up on localhost port 10000
waiting for the next event
  timed out, do some other work here
waiting for the next event
  connection from ('127.0.0.1', 61144)
waiting for the next event
  timed out, do some other work here
waiting for the next event
  received b'Part one of the message.' from ('127.0.0.1', 61144)
waiting for the next event
  sending b'Part one of the message.' to ('127.0.0.1', 61144)
waiting for the next event
('127.0.0.1', 61144) queue empty
waiting for the next event
  timed out, do some other work here
waiting for the next event
  received b'Part two of the message.' from ('127.0.0.1', 61144)
waiting for the next event
  sending b'Part two of the message.' to ('127.0.0.1', 61144)
waiting for the next event
('127.0.0.1', 61144) queue empty
waiting for the next event
  timed out, do some other work here
waiting for the next event
closing ('127.0.0.1', 61144)
waiting for the next event
  timed out, do some other work here
```
客户端输出：

```
$ python3 select_echo_slow_client.py
connecting to localhost port 10000
sending b'Part one of the message.'
sending b'Part two of the message.'
received b'Part one of the '
received b'message.Part two'
received b' of the message.'
closing socket
```

### Using poll()

poll（）函数提供与select（）类似的功能，但底层实现更有效。 代价是在Windows下不支持poll（），因此使用poll（）的程序不太可移植。
基于poll（）构建的echo服务器以其他示例中使用的相同套接字配置代码开始。

```python
#select_poll_echo_server.py
import select
import socket
import sys
import queue

# Create a TCP/IP socket
server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.setblocking(0)

# Bind the socket to the port
server_address = ('localhost', 10000)
print('starting up on {} port {}'.format(*server_address),
      file=sys.stderr)
server.bind(server_address)

# Listen for incoming connections
server.listen(5)

# Keep up with the queues of outgoing messages
message_queues = {}
```
传递给poll（）的超时值以毫秒而不是秒表示，因此为了暂停一整秒，超时必须设置为1000。

```python
# Do not block forever (milliseconds)
TIMEOUT = 1000
```
Python使用一个类来实现poll（），该类管理被监视的已注册数据通道。 通过调用register（）添加通道，其中标志指示通道对哪个事件感兴趣。 下表列出了完整的标志集。  
Event Flags for poll()

Event	| Description
:--- 	| :---
POLLIN	| Input ready
POLLPRI	| Priority input ready
POLLOUT	| Able to receive output
POLLERR	| Error
POLLHUP	| Channel closed
POLLNVAL|	Channel not open

echo服务器将设置一些仅用于读取的套接字，以及其他用于读取或写入的套接字。 适当的标志组合保存到局部变量`READ_ONLY`和`READ_WRITE`。

```python
# Commonly used flag sets
READ_ONLY = (
    select.POLLIN |
    select.POLLPRI |
    select.POLLHUP |
    select.POLLERR
)
READ_WRITE = READ_ONLY | select.POLLOUT
```
注册服务器套接字，以便任何传入连接或数据触发一个事件。

```python
# Set up the poller
poller = select.poll()
poller.register(server, READ_ONLY)
```
由于poll（）返回包含套接字文件描述符和事件标志的元组列表，因此需要一个从文件描述符编号到socket对象的映射来检索套接字，以从中读取或写入。

```python
# Map file descriptors to socket objects
fd_to_socket = {
    server.fileno(): server,
}
```
服务器的循环调用poll（），然后通过查找套接字并根据事件中的标志采取操作，以处理返回的“事件”。

```python
while True:

    # Wait for at least one of the sockets to be
    # ready for processing
    print('waiting for the next event', file=sys.stderr)
    events = poller.poll(TIMEOUT)

    for fd, flag in events:

        # Retrieve the actual socket from its file descriptor
        s = fd_to_socket[fd]
```
与select（）一样，当主服务器套接字“可读”时，这实际上意味着客户端存在挂起的连接。 使用READ_ONLY标志注册新连接，以监视通过它的新数据。

```python
        # Handle inputs
        if flag & (select.POLLIN | select.POLLPRI):

            if s is server:
                # A readable socket is ready
                # to accept a connection
                connection, client_address = s.accept()
                print('  connection', client_address,
                      file=sys.stderr)
                connection.setblocking(0)
                fd_to_socket[connection.fileno()] = connection
                poller.register(connection, READ_ONLY)

                # Give the connection a queue for data to send
                message_queues[connection] = queue.Queue()
```
server以外的套接字是已存在的客户端链接，recv（）用于访问等待读取的数据。

```python
            else:
                data = s.recv(1024)
```
如果recv（）返回任何数据，它将被放入套接字的输出队列，并使用modify（）更改该套接字的标志，以使poll（）检测套接字是否准备好输出数据。

```python
                if data:
                    # A readable client socket has data
                    print('  received {!r} from {}'.format(
                        data, s.getpeername()), file=sys.stderr,
                    )
                    message_queues[s].put(data)
                    # Add output channel for response
                    poller.modify(s, READ_WRITE)
```
recv（）返回的空字符串表示客户端已断开连接，因此unregister（）用于告知poll对象忽略套接字。

```python
                else:
                    # Interpret empty result as closed connection
                    print('  closing', client_address,
                          file=sys.stderr)
                    # Stop listening for input on the connection
                    poller.unregister(s)
                    s.close()

                    # Remove message queue
                    del message_queues[s]
```
POLLHUP标志表示客户端“挂断”连接而没有干净地关闭它。 服务器停止轮询消失的客户端。

```python
        elif flag & select.POLLHUP:
            # Client hung up
            print('  closing', client_address, '(HUP)',
                  file=sys.stderr)
            # Stop listening for input on the connection
            poller.unregister(s)
            s.close()
```
可写套接字的处理类似于select（）示例中使用的版本，但modify（）用于更改轮询器中套接字的标志，而不是从输出列表中删除它。

```python
        elif flag & select.POLLOUT:
            # Socket is ready to send data,
            # if there is any to send.
            try:
                next_msg = message_queues[s].get_nowait()
            except queue.Empty:
                # No messages waiting so stop checking
                print(s.getpeername(), 'queue empty',
                      file=sys.stderr)
                poller.modify(s, READ_ONLY)
            else:
                print('  sending {!r} to {}'.format(
                    next_msg, s.getpeername()), file=sys.stderr,
                )
                s.send(next_msg)
```
最后，使用POLLERR的任何事件都会导致服务器关闭套接字。

```python
        elif flag & select.POLLERR:
            print('  exception on', s.getpeername(),
                  file=sys.stderr)
            # Stop listening for input on the connection
            poller.unregister(s)
            s.close()

            # Remove message queue
            del message_queues[s]
```
当基于poll的服务器与`select_echo_multiclient.py`（使用多个套接字的客户端程序）一起运行时，输出如下：

```
$ python3 select_poll_echo_server.py
starting up on localhost port 10000
waiting for the next event
waiting for the next event
waiting for the next event
waiting for the next event
  connection ('127.0.0.1', 61253)
waiting for the next event
  connection ('127.0.0.1', 61254)
waiting for the next event
  received b'This is the message. ' from ('127.0.0.1', 61253)
  received b'This is the message. ' from ('127.0.0.1', 61254)
waiting for the next event
  sending b'This is the message. ' to ('127.0.0.1', 61253)
  sending b'This is the message. ' to ('127.0.0.1', 61254)
waiting for the next event
('127.0.0.1', 61253) queue empty
('127.0.0.1', 61254) queue empty
waiting for the next event
  received b'It will be sent ' from ('127.0.0.1', 61253)
  received b'It will be sent ' from ('127.0.0.1', 61254)
waiting for the next event
  sending b'It will be sent ' to ('127.0.0.1', 61253)
  sending b'It will be sent ' to ('127.0.0.1', 61254)
waiting for the next event
('127.0.0.1', 61253) queue empty
('127.0.0.1', 61254) queue empty
waiting for the next event
  received b'in parts.' from ('127.0.0.1', 61253)
  received b'in parts.' from ('127.0.0.1', 61254)
waiting for the next event
  sending b'in parts.' to ('127.0.0.1', 61253)
  sending b'in parts.' to ('127.0.0.1', 61254)
waiting for the next event
('127.0.0.1', 61253) queue empty
('127.0.0.1', 61254) queue empty
waiting for the next event
  closing ('127.0.0.1', 61254)
waiting for the next event
  closing ('127.0.0.1', 61254)
waiting for the next event
```

### 特定平台选项

select提供的不可移植选项是epoll，Linux支持的 edge polling API; kqueue，它使用BSD的内核队列和kevent，BSD的内核事件接口。 有关它们如何工作的更多详细信息，请参阅操作系统库文档。

## socketserver — 创建网络服务器

目的： 创建网络服务器  
socketserver模块是用于创建网络服务器的框架。 它定义了通过TCP，UDP，Unix流和Unix数据报处理同步网络请求（服务器请求处理程序阻塞，直到请求完成）的类。 它还提供混合（mix-in）类，以便轻松转换服务器为每个请求使用单独的线程或进程。  
处理请求的责任在服务器类和请求处理程序类之间分配。 服务器处理通信问题，例如侦听套接字和接受连接，请求处理程序处理“协议”问题，如解释传入数据，处理数据以及将数据发送回客户端。 这种责任划分意味着许多应用程序可以使用现有服务器类之一而无需任何修改，并为其提供请求处理程序类以使用自定义协议。

### Server Types

socketserver中定义了五种不同的服务器类。 BaseServer定义API，不用于直接实例化和使用。 TCPServer使用TCP/IP套接字进行通信。 UDPServer使用数据报套接字。 UnixStreamServer和UnixDatagramServer使用Unix域套接字，仅在Unix平台上可用。

### Server Objects

要构造服务器，请向其传递一个侦听请求的地址和一个请求处理程序类（而不是实例）。 地址格式取决于服务器类型和使用的套接字族。 有关详细信息，请参阅socket模块文档。  
实例化服务器对象后，使用`handle_request（）`或`serve_forever（）`来处理请求。 `serve_forever（）`方法在无限循环中调用`handle_request（）`，但如果应用程序需要将服务器与另一个事件循环集成，或者使用select（）监视不同服务器的多个套接字，则可以直接调用`handle_request（）`。

### Implementing a Server

在创建服务器时，通常可以重用其中一个现有类并提供自定义请求处理程序类。 对于其他情况，BaseServer包含几个可以在子类中覆盖的方法。

+ `verify_request（request，client_address）`：返回True以处理请求，或返回False以忽略它。 例如，服务器可以拒绝来自某个IP范围的请求或者当他过载时。
+ `process_request（request，client_address）`：调用`finish_request（）`实际执行处理请求的工作。 它也可以像混合类一样创建一个单独的线程或进程。
+ `finish_request（request，client_address）`：使用传递给服务器构造函数的类创建请求处理程序实例。 调用请求处理程序的handle（）来处理请求。

### Request Handlers

请求处理程序完成接收传入请求和决定采取什么操作的大部分工作。 处理程序负责在套接字层之上实现协议（即HTTP，XML-RPC或AMQP）。 请求处理程序从传入数据通道读取请求，对其进行处理，然后将响应写回。 有三个方法可以覆盖。

+ setup（） ：为请求准备请求处理程序。 在StreamRequestHandler中，setup（）方法创建类文件对象，用于读取和写入套接字。
+ handle（）：真正处理请求。 解析传入的请求，处理数据并发送响应。
+ finish（）：清除setup（）期间创建的任何内容。

许多处理程序只使用handle（）方法实现。

### Echo Example

此示例实现了一个简单的服务器/请求处理程序对，它接受TCP连接并回送客户端发送的任何数据。 它从请求处理程序开始。

```python
#socketserver_echo.py
import logging
import sys
import socketserver

logging.basicConfig(level=logging.DEBUG,
                    format='%(name)s: %(message)s',
                    )

class EchoRequestHandler(socketserver.BaseRequestHandler):

    def __init__(self, request, client_address, server):
        self.logger = logging.getLogger('EchoRequestHandler')
        self.logger.debug('__init__')
        socketserver.BaseRequestHandler.__init__(self, request,
                                                 client_address,
                                                 server)
        return

    def setup(self):
        self.logger.debug('setup')
        return socketserver.BaseRequestHandler.setup(self)

    def handle(self):
        self.logger.debug('handle')

        # Echo the back to the client
        data = self.request.recv(1024)
        self.logger.debug('recv()->"%s"', data)
        self.request.send(data)
        return

    def finish(self):
        self.logger.debug('finish')
        return socketserver.BaseRequestHandler.finish(self)
```
实际需要实现的唯一方法是EchoRequestHandler.handle（），但是前面描述的所有方法都包含在内，以说明调用的顺序。 EchoServer类与TCPServer没有什么不同，除了调用每个方法时的日志。

```python
class EchoServer(socketserver.TCPServer):

    def __init__(self, server_address,
                 handler_class=EchoRequestHandler,
                 ):
        self.logger = logging.getLogger('EchoServer')
        self.logger.debug('__init__')
        socketserver.TCPServer.__init__(self, server_address,
                                        handler_class)
        return

    def server_activate(self):
        self.logger.debug('server_activate')
        socketserver.TCPServer.server_activate(self)
        return

    def serve_forever(self, poll_interval=0.5):
        self.logger.debug('waiting for request')
        self.logger.info(
            'Handling requests, press <Ctrl-C> to quit'
        )
        socketserver.TCPServer.serve_forever(self, poll_interval)
        return

    def handle_request(self):
        self.logger.debug('handle_request')
        return socketserver.TCPServer.handle_request(self)

    def verify_request(self, request, client_address):
        self.logger.debug('verify_request(%s, %s)',
                          request, client_address)
        return socketserver.TCPServer.verify_request(
            self, request, client_address,
        )

    def process_request(self, request, client_address):
        self.logger.debug('process_request(%s, %s)',
                          request, client_address)
        return socketserver.TCPServer.process_request(
            self, request, client_address,
        )

    def server_close(self):
        self.logger.debug('server_close')
        return socketserver.TCPServer.server_close(self)

    def finish_request(self, request, client_address):
        self.logger.debug('finish_request(%s, %s)',
                          request, client_address)
        return socketserver.TCPServer.finish_request(
            self, request, client_address,
        )

    def close_request(self, request_address):
        self.logger.debug('close_request(%s)', request_address)
        return socketserver.TCPServer.close_request(
            self, request_address,
        )

    def shutdown(self):
        self.logger.debug('shutdown()')
        return socketserver.TCPServer.shutdown(self)
```
最后一步是添加一个主程序，将服务器设置为在线程中运行，并向其发送数据，以说明在回显数据时调用哪些方法。

```python
if __name__ == '__main__':
    import socket
    import threading

    address = ('localhost', 0)  # let the kernel assign a port
    server = EchoServer(address, EchoRequestHandler)
    ip, port = server.server_address  # what port was assigned?

    # Start the server in a thread
    t = threading.Thread(target=server.serve_forever)
    t.setDaemon(True)  # don't hang on exit
    t.start()

    logger = logging.getLogger('client')
    logger.info('Server on %s:%s', ip, port)

    # Connect to the server
    logger.debug('creating socket')
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    logger.debug('connecting to server')
    s.connect((ip, port))

    # Send the data
    message = 'Hello, world'.encode()
    logger.debug('sending data: %r', message)
    len_sent = s.send(message)

    # Receive a response
    logger.debug('waiting for response')
    response = s.recv(len_sent)
    logger.debug('response from server: %r', response)

    # Clean up
    server.shutdown()
    logger.debug('closing socket')
    s.close()
    logger.debug('done')
    server.socket.close()
```
运行该程序会产生以下输出。

```
$ python3 socketserver_echo.py

EchoServer: __init__
EchoServer: server_activate
EchoServer: waiting for request
EchoServer: Handling requests, press <Ctrl-C> to quit
client: Server on 127.0.0.1:55484
client: creating socket
client: connecting to server
client: sending data: b'Hello, world'
EchoServer: verify_request(<socket.socket fd=7, family=AddressFamily
.AF_INET, type=SocketKind.SOCK_STREAM, proto=0, laddr=('127.0.0.1',
55484), raddr=('127.0.0.1', 55485)>, ('127.0.0.1', 55485))
EchoServer: process_request(<socket.socket fd=7, family=AddressFamil
y.AF_INET, type=SocketKind.SOCK_STREAM, proto=0, laddr=('127.0.0.1',
 55484), raddr=('127.0.0.1', 55485)>, ('127.0.0.1', 55485))
EchoServer: finish_request(<socket.socket fd=7, family=AddressFamily
.AF_INET, type=SocketKind.SOCK_STREAM, proto=0, laddr=('127.0.0.1',
55484), raddr=('127.0.0.1', 55485)>, ('127.0.0.1', 55485))
EchoRequestHandler: __init__
EchoRequestHandler: setup
EchoRequestHandler: handle
client: waiting for response
EchoRequestHandler: recv()->"b'Hello, world'"
EchoRequestHandler: finish
client: response from server: b'Hello, world'
EchoServer: shutdown()
EchoServer: close_request(<socket.socket fd=7, family=AddressFamily.
AF_INET, type=SocketKind.SOCK_STREAM, proto=0, laddr=('127.0.0.1', 5
5484), raddr=('127.0.0.1', 55485)>)
client: closing socket
client: done
```
>Note  
每次程序运行时使用的端口号都会更改，因为内核会自动分配可用端口。 要使服务器每次都侦听特定端口，请在地址元组中提供该数字而不是0。

这是同一服务器的精简版本，没有日志记录调用。 只需要提供请求处理程序类中的handle（）方法。

```python
#socketserver_echo_simple.py
import socketserver

class EchoRequestHandler(socketserver.BaseRequestHandler):

    def handle(self):
        # Echo the back to the client
        data = self.request.recv(1024)
        self.request.send(data)
        return


if __name__ == '__main__':
    import socket
    import threading

    address = ('localhost', 0)  # let the kernel assign a port
    server = socketserver.TCPServer(address, EchoRequestHandler)
    ip, port = server.server_address  # what port was assigned?

    t = threading.Thread(target=server.serve_forever)
    t.setDaemon(True)  # don't hang on exit
    t.start()

    # Connect to the server
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((ip, port))

    # Send the data
    message = 'Hello, world'.encode()
    print('Sending : {!r}'.format(message))
    len_sent = s.send(message)

    # Receive a response
    response = s.recv(len_sent)
    print('Received: {!r}'.format(response))

    # Clean up
    server.shutdown()
    s.close()
    server.socket.close()
```
在这种情况下，由于TCPServer处理所有服务器要求，因此不需要特殊的服务器类。

```
$ python3 socketserver_echo_simple.py

Sending : b'Hello, world'
Received: b'Hello, world'
```

### Threading and Forking

要向服务器添加线程或进程支持，请在服务器的类层次结构中包含适当的mix-in。 混合类重写process_request（）以在准备好处理请求时启动新线程或进程，并且工作在新子进程或线程中完成。

对于线程，使用ThreadingMixIn。

```python
#socketserver_threaded.py
import threading
import socketserver

class ThreadedEchoRequestHandler(
        socketserver.BaseRequestHandler,
):

    def handle(self):
        # Echo the back to the client
        data = self.request.recv(1024)
        cur_thread = threading.currentThread()
        response = b'%s: %s' % (cur_thread.getName().encode(),
                                data)
        self.request.send(response)
        return

class ThreadedEchoServer(socketserver.ThreadingMixIn,
                         socketserver.TCPServer,
                         ):
    pass


if __name__ == '__main__':
    import socket

    address = ('localhost', 0)  # let the kernel assign a port
    server = ThreadedEchoServer(address,
                                ThreadedEchoRequestHandler)
    ip, port = server.server_address  # what port was assigned?

    t = threading.Thread(target=server.serve_forever)
    t.setDaemon(True)  # don't hang on exit
    t.start()
    print('Server loop running in thread:', t.getName())

    # Connect to the server
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((ip, port))

    # Send the data
    message = b'Hello, world'
    print('Sending : {!r}'.format(message))
    len_sent = s.send(message)

    # Receive a response
    response = s.recv(1024)
    print('Received: {!r}'.format(response))

    # Clean up
    server.shutdown()
    s.close()
    server.socket.close()
```
此线程服务器的响应包括处理请求的线程的标识符。

```
$ python3 socketserver_threaded.py

Server loop running in thread: Thread-1
Sending : b'Hello, world'
Received: b'Thread-2: Hello, world'
```
对于单独的进程，请使用ForkingMixIn。

```python
#socketserver_forking.py
import os
import socketserver

class ForkingEchoRequestHandler(socketserver.BaseRequestHandler):

    def handle(self):
        # Echo the back to the client
        data = self.request.recv(1024)
        cur_pid = os.getpid()
        response = b'%d: %s' % (cur_pid, data)
        self.request.send(response)
        return

class ForkingEchoServer(socketserver.ForkingMixIn,
                        socketserver.TCPServer,
                        ):
    pass


if __name__ == '__main__':
    import socket
    import threading

    address = ('localhost', 0)  # let the kernel assign a port
    server = ForkingEchoServer(address,
                               ForkingEchoRequestHandler)
    ip, port = server.server_address  # what port was assigned?

    t = threading.Thread(target=server.serve_forever)
    t.setDaemon(True)  # don't hang on exit
    t.start()
    print('Server loop running in process:', os.getpid())

    # Connect to the server
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((ip, port))

    # Send the data
    message = 'Hello, world'.encode()
    print('Sending : {!r}'.format(message))
    len_sent = s.send(message)

    # Receive a response
    response = s.recv(1024)
    print('Received: {!r}'.format(response))

    # Clean up
    server.shutdown()
    s.close()
    server.socket.close()
```

在这种情况下，子进程ID包含在服务器的响应中：

```
$ python3 socketserver_forking.py

Server loop running in process: 22599
Sending : b'Hello, world'
Received: b'22600: Hello, world'
```

