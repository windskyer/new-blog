---
title: ipvsadm命令用法
categories: Kubernetes
sage: false
date: 2021-08-16 14:22:06
tags: ipvs
---

架构lvs时经常会使用ipvsadm命令来查看状态，尽管可以使用 ipvsadm --help ，查看命令的使用方法。但会花时间在调试命令上面，特别针对初学者来说，更是如此。特此把一些较为常用的ipvsadm状态查看命令整理，以供大家参考。

<!-- more -->

1. 查看lvs的连接状态命令： ipvsadm  -l  --stats
```sh
#ipvsadm  -l  --stats

IP Virtual Server version 1.2.1 (size=4096)

Prot LocalAddress:Port               Conns   InPkts  OutPkts  InBytes OutBytes

  -> RemoteAddress:Port

TCP  10.6.2.149:ssh                      4        6        0      308        0

  -> 10.6.2.143:ssh                      0        0        0        0        0

  -> 10.6.2.146:ssh                      4        6        0      308        0

TCP  10.6.2.149:http                    11       11        0      596        0

  -> 10.6.2.143:http                     0        0        0        0        0

  -> 10.6.2.146:http                    11       11        0      596        0

TCP  10.6.2.151:ldap                    11       60        0     3729        0

  -> 10.6.2.147:ldap                     5       32        0     1972        0

  -> 10.6.2.148:ldap                     6       28        0     1757        0

TCP  10.6.2.151:mysql                  327     7001        0   961447        0

  -> 10.6.2.147:mysql                  313     6600        0   883068        0

  -> 10.6.2.148:mysql                   14      401        0    78379        0
```

说明：

1. Conns    (connections scheduled)  已经转发过的连接数
2. InPkts   (incoming packets)       入包个数
3. OutPkts  (outgoing packets)       出包个数
4. InBytes  (incoming bytes)         入流量（字节）  
5. OutBytes (outgoing bytes)         出流量（字节）



2. 查看lvs速率  ：ipvsadm   -l  --rate
```sh
#ipvsadm   -l  --rate


Prot LocalAddress:Port                 CPS    InPPS   OutPPS    InBPS   OutBPS

  -> RemoteAddress:Port

TCP  10.6.2.149:ssh                      0        0        0        0        0

  -> 10.6.2.143:ssh                      0        0        0        0        0

  -> 10.6.2.146:ssh                      0        0        0        0        0

TCP  10.6.2.149:http                     0        0        0        0        0

  -> 10.6.2.143:http                     0        0        0        0        0

  -> 10.6.2.146:http                     0        0        0        0        0

TCP  10.6.2.151:ldap                     0        0        0        0        0

  -> 10.6.2.147:ldap                     0        0        0        0        0

  -> 10.6.2.148:ldap                     0        0        0        0        0

TCP  10.6.2.151:mysql                    0        0        0        0        0

  -> 10.6.2.147:mysql                    0        0        0        0        0

  -> 10.6.2.148:mysql                    0        0        0        0        0
```

说明：

1. CPS      (current connection rate)   每秒连接数
2. InPPS    (current in packet rate)    每秒的入包个数
3. OutPPS   (current out packet rate)   每秒的出包个数
4. InBPS    (current in byte rate)      每秒入流量（字节）
5. OutBPS   (current out byte rate)      每秒入流量（字节）