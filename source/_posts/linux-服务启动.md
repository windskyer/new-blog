---
title: linux-服务启动
date: 2013-09-10 22:13:12
tags: bash
categories: Linux
---

<amp-auto-ads type="adsense" data-ad-client="ca-pub-5216394795966395"></amp-auto-ads>


# 判断linux  系统服务 是否启动的脚本

```bash

#！/bin/bash

if service sshd status &> /dev/nll
then
     echo "service is running"
else
     echo "service is stopd"

fi
```

<!-- more -->

修改系统的默认语言环境

vim /etc/sysconfig/i18n

LANG="en_US.UTF-8"