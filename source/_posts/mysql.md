---
title: mysql
date: 2013-09-19 16:03:41
tags: mysql
categories: db
---

# mysql 主从复制方法

两台主机
hostname : linux-web ip : 10.5.5.23
hostname : linux-mysql ip : 10.5.5.27

host1: linux-web

<!-- more -->

```bash
vim /etc/my.cnf

server-id=1
log-bin=master-bin

master-host=10.5.5.27 #hostname linux-mysql

master-user=slave #mysql 中的账户 slave(在linux-mysql) 用于相互链接
master-password=slave

replicate-ignore-db=mysql #不同步mysql database
replicate-do-db=sync #指定同步database
```

host2:linux-mysql 同上
需要改 server-id=2

```bash
cd /var/lib/mysql
#删除所有的以前的日志

service mysqld restart #(两台都重启)
```

# 如何修改MySQL监听IP地址

1.编辑/etc/my.cnf

在[mysqld]节中增加下面一行：

bind-address=0.0.0.0  #全部地址或者指定的ip地址

2.重启服务

service mysqld restart

3.验证

netstat -tln