---
title: kubernetes 调度器扩展
categories: Kubernetes
sage: false
date: 2020-06-20 10:46:44
tags: k8s, scheduler
---

## Kubernetes集群中添加新的调度规则

本文介绍第三种方法。这种方法主要用于需要对非标准Kubernetes调度程序直接管理的资源进行调度决策的场景。扩展程序帮助对这类资源进行调度决策。（注意这三种方式不是互斥的）

有三种办法可以再Kubernetes集群中添加新的调度规则（预选和优选规则）：

>1. 添加规则到kube-scheduler源码并重新编译部署， 参考此文档
>2. 实现自定义的调度程序，它可以代替或与标准的kubernetes调度程序同时运行在集群中
>3. 实现一个调度扩展程序，kubernetes 标准调度程序会在进行最后的调度决策前调用它

<!-- more -->

调度pod时，扩展程序允许外部进程对节点进行过滤并确定优先级。调度程序会向扩展程序发出两个单独的http/https请求，对应 “filter” 和 “prioritize” 两个操作。另外，如果扩展程序实现了 “bind” 操作，则可以直接将pod绑定到一个节点。

要使用扩展程序，则必须创建一个调度策略配置文件。该配置文件声明了如何连接到扩展程序、使用http还是https以及超时时间。

```go
// Holds the parameters used to communicate with the extender. If a verb is unspecified/empty,
// it is assumed that the extender chose not to provide that extension.
type ExtenderConfig struct {
 // URLPrefix at which the extender is available
 URLPrefix string `json:"urlPrefix"`
 // Verb for the filter call, empty if not supported. This verb is appended to the URLPrefix when issuing the filter call to extender.
 FilterVerb string `json:"filterVerb,omitempty"`
 // Verb for the prioritize call, empty if not supported. This verb is appended to the URLPrefix when issuing the prioritize call to extender.
 PrioritizeVerb string `json:"prioritizeVerb,omitempty"`
 // Verb for the bind call, empty if not supported. This verb is appended to the URLPrefix when issuing the bind call to extender.
 // If this method is implemented by the extender, it is the extender's responsibility to bind the pod to apiserver.
 BindVerb string `json:"bindVerb,omitempty"`
 // The numeric multiplier for the node scores that the prioritize call generates.
 // The weight should be a positive integer
 Weight int `json:"weight,omitempty"`
 // EnableHttps specifies whether https should be used to communicate with the extender
 EnableHttps bool `json:"enableHttps,omitempty"`
 // TLSConfig specifies the transport layer security config
 TLSConfig *client.TLSClientConfig `json:"tlsConfig,omitempty"`
 // HTTPTimeout specifies the timeout duration for a call to the extender. Filter timeout fails the scheduling of the pod. Prioritize
 // timeout is ignored, k8s/other extenders priorities are used to select the node.
 HTTPTimeout time.Duration `json:"httpTimeout,omitempty"`
}
```

下面是一个带有扩展程序配置项的调度策略文件：

```json
{
  "predicates": [
   {
      "name": "HostName"
   },
   {
      "name": "MatchNodeSelector"
   },
   {
      "name": "PodFitsResources"
   }
 ],
  "priorities": [
   {
      "name": "LeastRequestedPriority",
      "weight": 1
   }
 ],
  "extenders": [
   {
      "urlPrefix": "http://127.0.0.1:12345/api/scheduler",
      "filterVerb": "filter",
      "enableHttps": false
   }
 ]
}
```

传递给扩展程序的FilterVerb端点的参数是根据k8s筛选规则过滤之后的节点集合以及对应的pod。传递给扩展器上PrioritizeVerb端点的参数是通过k8s筛选规则过滤和扩展程序过滤之后节点集合以及对应的pod。

```go
// ExtenderArgs represents the arguments needed by the extender to filter/prioritize
// nodes for a pod.
type ExtenderArgs struct {
 // Pod being scheduled
 Pod   api.Pod      `json:"pod"`
 // List of candidate nodes where the pod can be scheduled
 Nodes api.NodeList `json:"nodes"`
}
```

"filter" 调用返回节点列表（schedulerapi.ExtenderFilterResult）。“prioritize” 调用返回每个节点的优先级（schedulerapi.HostPriorityList）。

“filter” 调用可以跟进自身的过滤规则删减节点。“prioritize” 调用返回的分数会加到k8s（通过对应的优先级函数计算）的分数来用于最终节点的选择。

“bind” 调用用于将pod绑定到一个节点。这是一个可选功能，如果实现了该功能，则由扩展程序负责调用apiserver进行绑定。该操作传递给扩展程序的的参数包括pod 名称，命名空间以及节点名称。

```go
// ExtenderBindingArgs represents the arguments to an extender for binding a pod to a node.
type ExtenderBindingArgs struct {
 // PodName is the name of the pod being bound
 PodName string
 // PodNamespace is the namespace of the pod being bound
 PodNamespace string
 // PodUID is the UID of the pod being bound
 PodUID types.UID
 // Node selected by the scheduler
 Node string
}
```