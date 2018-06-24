---
title: CentOS7 开机加载模块
description:
categories: Linux运维
tags:
  - CentOS
  - Linux
  - modules
---

以 `i40e` 网卡驱动模块为例：  
在 `/etc/sysconfig/modules/` 目录下新建一个文件 `i40e.modules` ，添加以下内容到该文件：

```shell
#! /bin/sh
/sbin/modinfo -F filename i40e > /dev/null 2>&1
if [ $? -eq 0 ];then
	/sbin/modprobe i40e      #强制加 --force
fi
```

执行以下操作

```shell
#增加执行权限，至关重要
$chmod 755 i40e.modules 
#重启系统  
$reboot
#验证模块是否加载
$lsmod | grep i40e
```