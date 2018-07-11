---
title: PPTP-L2TP-PPPOE 配置详解
description:
categories: 
  - 运维
tags:
  - pptp
  - L2TP
  - PPPoE
  - VPN
---

## PPPOE

### server

参考 <http://laibulai.iteye.com/blog/1171898>

+ 安装rp-pppoe：`yum install rp-pppoe`  
+ 配置 /etc/ppp/pppoe-server-options,内容：

```shell
# PPP options for the PPPoE server
# LIC: GPL
require-pap
require-chap
login
lcp-echo-interval 10
lcp-echo-failure 2
logfile /var/log/pppoe.log

ms-dns 8.8.8.8
ms-dns 8.8.4.4
defaultroute
```
+ 添加用户名密码，`/etc/ppp/chap-secrets`:  
`pppoe  *  "123456"  *  `
+ 添加防火墙规则，做nat转换

```shell
iptables -A POSTROUTING -t nat -s 10.10.10.0/24 -j MASQUERADE
iptables -A FORWARD -p tcp --syn -s 10.10.10.0/24 -j TCPMSS --set-mss 1256
echo 1 > /proc/sys/net/ipv4/ip_forward
sysctl -w net.ipv4.ip_forward=1
```
+ 启动pppoe  

```shell
pppoe-server -I eth0 -L 10.10.10.1 -R 10.10.10.100-200
```
 
### client

参考 <http://www.njust.edu.cn/web/Linux-PPPoE.pdf>  
>注：client端也要配置pap，chap验证，否则拨号无法通过验证。(测试过windows client，linux client，没有测过client server在同一台机器的情况)

1. 安装rp-pppoe客户端：`yum install rp-pppoe`
2. `adsl-setup`，根据相关提示进行填写
3. adsl-start,adsl-stop,pppoe-status
 
## L2TP

>server client都要验证ipsec服务，另外查看服务器iptables的reject有没有阻止，有的话删掉

### server
参考<http://lizhug.com/tech/centos6-5%E6%90%AD%E5%BB%BAl2tp-ipsec-vpn-vpn%E7%B3%BB%E5%88%97%EF%BC%88%E4%B8%80%EF%BC%89/>

+ 添加包源，安装

```shell
rpm -Uvh http://dl.fedoraproject.org/pub/epel/6/i386/epel-release-6-8.noarch.rpm
yum install openswan ppp xl2tpd
```
+ 配置

<1>修改`/etc/xl2tpd/xl2tpd.conf`

```shell
[global]
listen-addr = 45.62.96.30   #改成自己本机的IP
ipsec saref = yes
[lns default]
ip range = 192.168.1.128-192.168.1.254    #分配的客户端IP
local ip = 192.168.1.1                 #本地IP  不用改
refuse chap = yes                     #改成refuse
refuse pap = yes
require authentication = yes
name = l2tp
ppp debug = yes
pppoptfile = /etc/ppp/options.xl2tpd
length bit = yes
```
<2>修改`/etc/ppp/options.xl2tpd`

```shell
ipcp-accept-local
ipcp-accept-remote
ms-dns 8.8.8.8
ms-dns 8.8.4.4
noccp
auth
crtscts
debug
hide-password
modem
lock
proxyarp
```
<3>修改`/etc/ipsec.conf` 在末尾添加(注：conn前面没有空格，其他行以tab空开)

```shell
conn L2TP-PSK-NAT
　　rightsubnet=vhost:%priv
　　also=L2TP-PSK-noNAT
conn L2TP-PSK-noNAT
　　authby=secret
　　pfs=no
　　auto=add
　　keyingtries=3
　　rekey=no
　　ikelifetime=8h
　　keylife=1h
　　type=transport
　　left=45.62.96.30  #这边替换成你的本机IP
　　leftprotoport=17/1701
　　right=%any
　　rightprotoport=17/%any
```
<4>添加`/etc/ipsec.secrets` 预定义密钥(注：PSK前面有空格)

```shell
vi /etc/ipsec.secrets
45.62.96.30     %any: PSK       "test1234"    #ip地址替换成你的本机ip
```
<5>设置网络策略（这个我也不懂）,直接在终端输入

```shell
for each in /proc/sys/net/ipv4/conf/*
do
    echo 0 > $each/accept_redirects
    echo 0 > $each/send_redirects
done
```

<6> 创建账户文件

```shell
vi /etc/ppp/chap-secrets
里面格式为（如我的）
#用户名 * 密码 *
lizhug * 1234567890 *
```
<7>修改系统配置文件/etc/sysctl.conf  在结尾添加

```shell
net.ipv4.ip_forward = 1
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.log_martians = 0
net.ipv4.conf.default.log_martians = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.icmp_ignore_bogus_error_responses = 1
```
执行 `sysctl -p` 使上面的设置生效

<8>修改防火墙设置,直接在终端执行

```shell
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
service iptables save
service iptables restart
```
<9>测试执行(注：这里遇到了faild initiallize nss database 的错误，通过更新nss-tools包成功：yum install nss-tools)

```shelll
ipsec setup start
ipsec verify
```
出现如下提示就OK，如果出现了FAILED，直接粘贴google找修改方法

```shell
[root@default ~]# ipsec verify 
Checking your system to see if IPsec got installed and started correctly:
Version check and ipsec on-path [OK]
Linux Openswan U2.6.32/K(no kernel code presently loaded)
Checking for IPsec support in kernel [OK]
 SAref kernel support [N/A]
Checking that pluto is running [OK]
 Pluto listening for IKE on udp 500 [OK]
 Pluto listening for NAT-T on udp 4500 [OK]
Checking for 'ip' command [OK]
Checking /bin/sh is not /bin/dash [OK]
Checking for 'iptables' command [OK]
Opportunistic Encryption Support [DISABLED]
```
重启xl2tp  
`service xl2tpd restart`  
<10>开放端口以及转发参考：<http://blog.csdn.net/musiccow/article/details/22904997>
原样执行下面所有命令，

```shell
#Allow ipsec traffic
iptables -A INPUT -m policy --dir in --pol ipsec -j ACCEPT
iptables -A FORWARD -m policy --dir in --pol ipsec -j ACCEPT

#Do not NAT VPN traffic
iptables -t nat -A POSTROUTING -m policy --dir out --pol none -j MASQUERADE

#Forwarding rules for VPN
iptables -A FORWARD -i ppp+ -p all -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT

#Ports for Openswan / xl2tpd
iptables -A INPUT -m policy --dir in --pol ipsec -p udp --dport 1701 -j ACCEPT
iptables -A INPUT -p udp --dport 500 -j ACCEPT
iptables -A INPUT -p udp --dport 4500 -j ACCEPT
 
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE
```
再执行下面保存iptables

```shell
service iptables save
service iptables restart
```

<11>添加自启动

```shell
chkconfig xl2tpd on
chkconfig iptables on
chkconfig ipsec on
```

### client

安装：

```shell
yum install xl2tpd
yum install ppp
```
配置`/etc/xl2tpd/xl2tpd.conf`

```shell
[lac testvpn（VPN名称）]
name = wufuqiang                                 ; l2tp帐号
lns = 192.168.20.10                                           ; l2tp server的IP
pppoptfile = /etc/ppp/peers/testvpn.l2tpd         ; pppd拨号时使用的配置文件
ppp debug = yes
```
vi `/etc/ppp/peers/testvpn.l2tpd`

```shell
remotename testvpn
user wufuqiang
password 1234567890
unit 0
lock
nodeflate
nobsdcomp
noauth
persist
nopcomp
noaccomp
maxfail 5
debug
```
配置文件都建好后，可以启动xl2tpd了，注意启动不代表拨号

运行方式1： 运行`/etc/init.d/xl2tpd start`即可，这种启动方式会自动去找`/etc/xl2tpd/xl2tpd.conf`这个配置文件，（每次修改xl2tpd.conf配置文件都要重启（restart）xl2tpd服务，重新加载配置文件方能成功！）  
运行方式2：# `xl2tpd -c "/your/config_file/path"`，如果使用此方法，要确保存在`/var/run/xl2tpd/`这个目录，其实看看`/etc/init.d/xl2tpd`这个文件也可以看出来，如果不存在，脚本会创建这个目录

开始拨号：

```shell
echo 'c testvpn' > /var/run/xl2tpd/l2tp-control
```
拨号成功的话，通过ifconfig可以看见有个ppp0的接口

断开连接:

```shell
echo 'd testvpn' > /var/run/xl2tpd/l2tp-control
```
启动xl2tpd到拨号，整个过程可查看日志

```shell
tail -f /var/log/message
```
 
## PPTP

参考<http://www.dabu.info/centos6-4-structures-pptp-vpn.html>

### server

检测

```shell
modprobe ppp-compress-18 && echo ok
```
安装ppp和iptables

```shell
yum install -y perl ppp iptables //centos默认安装了iptables和ppp  
```
版本要对。`yum list installed ppp`显示版本  
`ppp 2.4.4——————>pptpd 1.3.4`  
`ppp 2.4.5——————>pptpd 1.4.0`  

安装pptpd

```shell
rpm -Uvh http://poptop.sourceforge.net/yum/stable/rhel6/pptp-release-current.noarch.rpm
yum install pptpd
```
修改配置`/etc/ppp/options.pptpd`，加入以下内容

```
ms-dns 8.8.8.8
ms-dns 8.8.4.4
```
配置文件/etc/ppp/chap-secrets

```
# Secrets for authentication using CHAP
# client server secret IP addresses
myusername pptpd mypassword *
```
myusername是你的vpn帐号，mypassword是你的vpn的密码，`*`表示对任何ip，记得不要丢了这个星号。我这里根据这个格式，假设我的vpn的帐号是ksharpdabu，密码是 sky。那么，应该如下：  
`ksharpdabu pptpd sky *`  
配置文件/etc/pptpd.conf, 添加下面两行：

```shell
localip 192.168.9.1
remoteip 192.168.9.11-30 //表示vpn客户端获得ip的范围
```
关键点：pptpd.conf这个配置文件必须保证最后是以空行结尾才行，否则会导致启动pptpd服务时，出现“Starting pptpd:”，一直卡着不动的问题，无法启动服务，切记呀！
配置文件/etc/sysctl.conf 将`net.ipv4.ip_forward = 0` 改成 `net.ipv4.ip_forward = 1`  
保存修改后的文件

```shell
#/sbin/sysctl -p
```
启动pptp vpn服务和iptables

```shell
/sbin/service pptpd start 或者 #service pptpd start
```
 
### client

参考<https://linuxconfig.org/how-to-establish-pptp-vpn-client-connection-on-centos-rhel-7-linux>

安装pptp `yum install pptp`
pptp支持模块

```
#modprobe nf_conntrack_pptp ;
```
或者添加mppe模块:

```
modprobe ppp_mppe
```
配置 /etc/ppp/chap-secrets

```
admin  pptp password *
```
在/etc/ppp/peers/目录下创建linuxconfig文件，以下是内容

```
pty "pptp 192.168.20.10 --nolaunchpppd"
name wfqgtxvpn
remotename pptpd
require-mppe-128
file /etc/ppp/options.pptp
ipparam linuxconfig
```
连接

```
#pppd call linuxconfig
```
查看/var/log/messages 日志，分析错误

断开

```
pkill pppd
```
