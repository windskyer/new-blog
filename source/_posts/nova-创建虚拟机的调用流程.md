---
title: nova 创建虚拟机的调用流程
categories: Openstack
sage: false
date: 2016-04-20 08:33:47
tags: node
---

<amp-auto-ads type="adsense" data-ad-client="ca-pub-5216394795966395"></amp-auto-ads>

# nova 创建虚拟机的调用流程

1, 根据 route 找到 controller 所对应的action 
2, 调用 compute api  中的create 方法

<!-- more -->

3，调用 compute_task_api  这个api  是conductor.ComputeTaskAPI() conductor.ComputeTaskAPI() 根据 配置文件中的选项use_local 是否启用conductor 服务
4, 调用 conductor 的rpcapi 
5, rpcapi 通过发送cast的消息给 conductor manager 
6, 在conductor manager 中首先是调用 scheduler 的 rpcapi 
6, scheduler rpcapi 通过 call 方法 发送消息给 scheduler manager
7, 在scheduler 的 manager 中 进行 filter， weight 过滤选择，
8，把选中的 host 返回给 conductor manager
9，conductor manager 在 调用computer rpcapi
10,computer rpcapi 获取指定host的 cctxt 通过cast 的方式把消息发给 compute manager 
11，computer manager 去调用相应的 driver 去 创建虚拟机