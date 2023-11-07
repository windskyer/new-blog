---
title: Linux bridge
categories: Linux
sage: false
date: 2019-04-02 15:00:16
tags: bridge
---

# Linux bridge 网桥祥解

Linux网桥是网桥的软件实现，这是Linux内核的内核部分。与硬件网桥相类似，Linux网桥维护了一个2层转发表（也称为MAC学习表，转发数据库，或者仅仅称为FDB），它跟踪记录了MAC地址与端口的对应关系。当一个网桥在端口N收到一个包时（源MAC地址为X），它在FDB中记录为MAC地址X可以从端口N到达。这样的话，以后当网桥需要转发一个包到地址X时，它就可以从FDB查询知道转发到哪里。构建一个FDB常常称之为“MAC学习”或仅仅称为“学习”过程。

<!-- more -->

你可以使用以下命令来检查Linux网桥当前转发表或MAC学习表。

```bash
sudo brctl showmacs <bridge-name>
sudo bridge fdb show  #查看所有的mac地址对应的信息
```