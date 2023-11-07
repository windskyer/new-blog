---
title: k8s集群dns优化
categories: Kubernetes
sage: false
date: 2021-06-02 16:20:53
tags: dns
---

# 目的

使用Service Mesh 拥抱云原⽣，拥抱开源社区已经成为⼈尽皆知的共识。
Service Mesh 体系中，使⽤ kubernetes Service 是必不可少的⼀环，在我们逐步迁移服务到Mesh体系的过
程中发现，存在部分应⽤的某些调⽤接经常超时。
经过我们的排查，这些出问题的调⽤往往是通过短链接的⽅式来访问的。
在使⽤短链接时，每次调⽤都会建⽴新的连接，这也会伴随着⼀次次 DNS 查询。
DNS 查询速度成为了提⾼调⽤延迟的罪魁祸⾸。

<!-- more -->

# 缓存

在原来体系中，使⽤宿主机的 unbound 进⾏ DNS 缓存，来缓解⼤量 DNS 查询带来的压⼒。
在 Kubernetes 上，官⽅推荐使⽤⼀个名为 [NodeLocal DNS Cache](https://kubernetes.io/zh/docs/tasks/administer-cluster/nodelocaldns/
) 的组件，来作为 DNS 的主机缓存。
通过部署 node-local-dns 的 DaemonSet，并重新配置 kubelet 参数 --cluster-dns=169.254.20.10.254.0.10。
之后部署到节点的容器就可以享受来⾃ node-local-dns 的缓存了。

# 测试

在其中两台机器上，进⾏了node-local-dns 并修改了主机上的 kubelet 参数。
我们把使⽤短链接的应⽤调度到这两台机器上。
按照预期来说，使⽤了本地的 DNS 缓存，短链接绝⼤多数的请求都被 Cache住 了才是。
然⽽⼀波故障开始出现：这个应⽤接⼝开始⼤量超时！
我们发现主机的 node-cache cpu 使⽤率竟然达到了惊⼈的 4 核！
这不应该啊！此时 DNS 的查询量理论上应该只有15k，难道使⽤ CoreDNS 的 node-cache 只能承载这么点RPS？
⼀定存在某种误解！
这个时候我们进⼊容器直接抓包看， 果然有⻤！
我们发现DNS查询总是先加上 .xx.svc.cluster.local 后缀进⾏查询，然后加上 svc.cluster.local 、 cluster.local
, 最后才查询我们原来要查的域名。

⼀看 /etc/resolve.conf :
search xx.svc.cluster.local svc.cluster.local cluster.local options ndots:5
最后⼀⾏的 ndots:5 映⼊眼帘，这就是说，如果查询的域名少于5个点，先加上集群内的后缀依次查询，如果
都查不到再按照原始的域名进⾏查询。
好家伙查询量直接翻三四倍以上！
我们修改容器 [DNSConfig](https://kubernetes.io/docs/concepts/services-networking/dns-podservice/#pod-dns-config) ，再部署来看看。
错误果然减少了，但是还是多！
这个时候，不禁让我们在想： CoreDNS 你是不是不中⽤啊？

# 再测试

我们决定先对⽐⼀下 CoreDNS 实现的 node-cache 和我们现在⽣产⼤量使⽤的 unbound 。
安装⼀个⽤于压测 DNS 的开源⼯具， [dnsperf](https://github.com/DNS-OARC/dnsperf)
它的使⽤⽅式也很简单:
dnsperf -d ./input.txt -n10000 -Q 8000 -s 169.254.20.10 -t 0.2
-d 意思是输⼊的测试⽂件路径
-Q 每秒限制多少次查询
-s 查询的服务器地址
-t 超时时间（秒）
-n 重复多少次
我们把测试的域名写进测试⽂件，格式如下：
www.baidu.com A
由于我们服务器本地是开了unbound的，因此我们分别使⽤⼀下两个命令对 unbound 和 node-cache 进⾏测
试：

```sh
$ dnsperf -d ./input.txt -n 100000 -Q 20000 -s  127.0.0.1 -t 0.2
...
Statistics:
Queries sent: 100000
Queries completed: 100000 (100.00%)
Queries lost: 0 (0.00%)
Response codes: NOERROR 100000 (100.00%)
Average packet size: request 31, response 126
Run time (s): 5.000073
Queries per second: 19999.708004
Average Latency (s): 0.000021 (min 0.000012, max 0.002078)
Latency StdDev (s): 0.000024
```

```sh
$ dnsperf -d ./input.txt -n 100000 -Q 20000 -s 169.254.20.10 -t 0.2
Statistics:
 Queries sent: 100000
 Queries completed: 99852 (99.85%)
 Queries lost: 148 (0.15%)
 Response codes: NOERROR 99852 (100.00%)
 Average packet size: request 31, response 202
 Run time (s): 5.000052
 Queries per second: 19970.192310
 Average Latency (s): 0.000492 (min 0.000026, max 0.068636)
 Latency StdDev (s): 0.003775
```

在此测试过程中，unbound 使⽤的 CPU ⼤概是 0.3 核左右，⽽ node-cache 达到了 1.6 核。
这是典型的吃的⽐别⼈多，⼲活却不如别⼈利索啊！
这个时候，我们拿着测试结果跟⾦⼭云的同学沟通了⼀下，我们都认为这玩意⼉不给⼒，⼲脆是不是可以⽤
Unbound 实现 node-cache 呢？
根据官⽅的关于[nodelocal-cache-dns](https://github.com/kubernetes/enhancements/blob/master/keps/sig-network/1024-nodelocal-cachedns/README.md)的提案描述：
It is possible to run any program as caching agent by modifying the daemonset and configmap spec.
Publishing an image with Unbound DNS can be added as a follow up.
这句话说，有可能运⾏任何进程作为缓存的agent，只需要修改daemonset和configmap！⽽且很快就会发布
基于 unbound 实现的镜像！
然⽽当我们看了⼀下⽬前的[代码](https://github.com/kubernetes/dns)
两年前描述的这个场景，在今天完全没有任何关于此的迹象！
痛哉！

# 研究
我对着 local-cache-dns 的 DaemonSet 配置仔细看了⼜看 ，终于，被我发现了两处华点：

>1. 容器的 CPU request 配置为 25m 。
>2. 没有配置 GOMAXPROCS

CPU request 设置的⼩，也就意味着调度的级别⾮常⾮常低！任务很容易饿出问题！参
看： https://access.redhat.com/documentation/enus/red_hat_enterprise_linux/6/html/resource_management_guide/sec-cpu
没有配置 GOMAXPROCS，也就意味着 Go ⽤了 CPU 核⼼数作为 GOMAXPROCS，⽽我们机器是开了超线
程的 72 核，也就是说：
Go 调度器很容易出现延迟问题！

# 验证

⾸先配置 GOMAXPROCS 环境变量为 "2" ！
同样的测试，结果变成了：
```sh
DNS Performance Testing Tool
Version 2.8.0
[Status] Command line: dnsperf -d ./input.txt -n 100000 -Q 20000 -s
169.254.20.10 -t 0.2
[Status] Sending queries (to 169.254.20.10:53)
[Status] Started at: Tue Nov 16 13:33:35 2021
[Status] Stopping after 100000 runs through file
[Status] Testing complete (end of file)
Statistics:
 Queries sent: 100000
 Queries completed: 100000 (100.00%)
 Queries lost: 0 (0.00%)
 Response codes: NOERROR 100000 (100.00%)
 Average packet size: request 31, response 202
 Run time (s): 5.000138
 Queries per second: 19999.448015
dns.md 2021/11/16
5 / 6
 Average Latency (s): 0.000060 (min 0.000028, max 0.001399)
 Latency StdDev (s): 0.000051
```

百分百没有超时！最⼤ latency 1ms ！
看起来没有改 cpu request 也得到了不错的结果？

# 测试

让我们把压测的 RPS 调⼩⼀点看看会发⽣什么：

```sh
$ dnsperf -d ./input.txt -n 100000 -Q 2000 -s 169.254.20.10 -t 0.2
..
Statistics:
 Queries sent: 100000
 Queries completed: 99772 (99.77%)
 Queries lost: 228 (0.23%)
 Response codes: NOERROR 99772 (100.00%)
 Average packet size: request 31, response 202
 Run time (s): 50.000076
 Queries per second: 1995.436967
 Average Latency (s): 0.000066 (min 0.000030, max 0.006804)
 Latency StdDev (s): 0.000133
```

228个请求超时，最⼤响应时间上涨到了 6.8ms！
果然，推测中的内核调度延迟它出现了！
让我们把 cpu request 从 25m 调整到 500m，重新测试⼀下：

```sh
$ dnsperf -d ./input.txt -n 100000 -Q 2000 -s 169.254.20.10 -t 0.2
..
Statistics:
 Queries sent: 100000
 Queries completed: 100000 (100.00%)
 Queries lost: 0 (0.00%)
 Response codes: NOERROR 100000 (100.00%)
 Average packet size: request 31, response 202
 Run time (s): 50.000074
 Queries per second: 1999.997040
 Average Latency (s): 0.000053 (min 0.000035, max 0.003924)
 Latency StdDev (s): 0.000032
```

这个结果可以说⼗分接近我们需要的稳定性了！

# 总结
虽然，最后经过优化的 node-cache 在延迟上达到了可以接受的范畴，但是，相⽐于 unbound ，它的
latency 仍然是不稳定的，CPU 消耗更是⽐ unbound ⾼出⼀⼤截。
虽然在单节点上表现不明显，但是在整个集群消耗的 CPU 就⾮常可观了！
因此，我们暂时决定使⽤node-cache，未来，我们还是期望能切到更稳定、可靠的 unbound 实现的
NodeLocal DNSCache