---
title: 获取所有物理网卡接口名称
categories: maintenance
tags:
---

[stackExchange](https://unix.stackexchange.com/questions/432245/how-to-get-only-the-name-of-the-physical-ethernet-interface)

You can tell which interfaces are virtual via

```shell
ls -l /sys/class/net/
```
which gives you this output:

```
[root@centos7 ~]# ls -l /sys/class/net/
total 0
lrwxrwxrwx. 1 root root 0 Mar 20 08:58 ens33 -> ../../devices/pci0000:00/0000:00:11.0/0000:02:01.0/net/ens33
lrwxrwxrwx. 1 root root 0 Mar 20 08:58 lo -> ../../devices/virtual/net/lo
lrwxrwxrwx. 1 root root 0 Mar 20 08:58 virbr0 -> ../../devices/virtual/net/virbr0
lrwxrwxrwx. 1 root root 0 Mar 20 08:58 virbr0-nic -> ../../devices/virtual/net/virbr0-nic
```
From there, you could grep to filter only non-virtual interfaces:

```shell
ls -l /sys/class/net/ | grep -v virtual
```

Another option is to use this small script, adapted from this answer, which prints the name of all interfaces which do not have a MAC address of 00:00:00:00:00:00 i.e. physical:

```shell
#!/bin/bash

for i in $(ip -o link show | awk -F': ' '{print $2}')
do
    mac=$(ethtool -P $i)
    [[ $mac != *"00:00:00:00:00:00"* ]] && echo "$i"
done
```
