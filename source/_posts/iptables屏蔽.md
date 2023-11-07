---
title: iptables屏蔽
categories: Kubernetes
sage: false
date: 2017-01-29 16:17:08
tags: iptables
---

<amp-auto-ads type="adsense" data-ad-client="ca-pub-5216394795966395"></amp-auto-ads>

# 用 iptables 屏蔽来自某个国家的 IP

方法很容易，先到 IPdeny 下载以国家代码编制好的 IP 地址列表，比如下载 cn.zone：

`wget http://www.ipdeny.com/ipblocks/data/countries/cn.zone`

<!-- more -->

有了国家的所有 IP 地址，要想屏蔽这些 IP 就很容易了，直接写个脚本逐行读取 cn.zone 文件并加入到 iptables 中：

```bash
#!/bin/bash
# Block traffic from a specific country
# written by vpsee.com

COUNTRY="cn"
IPTABLES=/sbin/iptables
EGREP=/bin/egrep

if [ "$(id -u)" != "0" ]; then
   echo "you must be root" 1>&2
   exit 1
fi

resetrules() {
$IPTABLES -F
$IPTABLES -t nat -F
$IPTABLES -t mangle -F
$IPTABLES -X
}

resetrules

for c in $COUNTRY
do
        country_file=$c.zone

        IPS=$($EGREP -v "^#|^$" $country_file)
        for ip in $IPS
        do
           echo "blocking $ip"
           $IPTABLES -A INPUT -s $ip -j DROP
        done
done

exit 0
```