---
title: k8s 集群启用 Pod 亲和性名称空间选择器
categories: Kubernetes
sage: false
date: 2021-09-10 16:01:52
tags: scheduler
---

# 问题描述
由于某些原因需要配置 pod 间的反亲和规则，不让两个 pod 调度到同一个节点。但是在配置完反亲和后，两个 pod 还是被调度到了同一个节点上。
<!-- more -->

# 问题解决
问题的关键是没有指定 namespaces 字段。如果不指定 namespaces 字段，或者指定了 namespaces 字段但是为空，那么作用范围将仅限于该 pod 的命名空间。对于同一个资源的不同副本来说，都是同一个命名空间，不存在该问题。但是要想配置不同类型资源 pod 的反亲和，则要指定 namespaces 字段来限定范围。例如下面编排文件：
```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: nginx
  name: nginx
  namespace: default
spec:
  progressDeadlineSeconds: 600
  replicas: 2
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: nginx
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        k8s-app: nginx
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: k8s-app
                operator: In
                values:
                - nginx
            namespaceSelector: {} # 设置此选项
            topologyKey: kubernetes.io/hostname
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
        resources:
          requests:
            cpu: 100m
            memory: 70Mi
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      nodeSelector:
        kubernetes.io/role: node
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      tolerations:
      - effect: NoSchedule
        key: node.kubernetes.io/unschedulable
```

# 启用
用户也可以使用 namespaceSelector 选择匹配的名字空间，namespaceSelector 是对名字空间集合进行标签查询的机制。 亲和性条件会应用到 namespaceSelector 所选择的名字空间和 namespaces 字段中 所列举的名字空间之上。 注意，空的 namespaceSelector（{}）会匹配所有名字空间，而 null 或者空的 namespaces 列表以及 null 值 namespaceSelector 意味着“当前 Pod 的名字空间”。

此功能特性是 Beta 版本的，默认是被启用的。你可以通过针对 kube-apiserver 和 kube-scheduler 设置 特性门控 PodAffinityNamespaceSelector 来禁用此特性。

支持版本情况：

|特性|默认值|状态|开始（Since）|结束（Until）|
|---|---|---|---|---|
|PodAffinityNamespaceSelector|false|Alpha|1.21|1.21|
|PodAffinityNamespaceSelector|true|Beta|1.22||

在kube-apiserver 和 kube-scheduler 启动参数中开启该选项
```yaml
--feature-gates="...,PodAffinityNamespaceSelector=true"
```

## 选项注解
```go
type PodAffinityTerm struct {
	// A label query over a set of resources, in this case pods.
	// +optional
	LabelSelector *metav1.LabelSelector `json:"labelSelector,omitempty" protobuf:"bytes,1,opt,name=labelSelector"`
	// namespaces specifies a static list of namespace names that the term applies to.
	// The term is applied to the union of the namespaces listed in this field
	// and the ones selected by namespaceSelector.
	// null or empty namespaces list and null namespaceSelector means "this pod's namespace"
	// +optional
	Namespaces []string `json:"namespaces,omitempty" protobuf:"bytes,2,rep,name=namespaces"`
	// This pod should be co-located (affinity) or not co-located (anti-affinity) with the pods matching
	// the labelSelector in the specified namespaces, where co-located is defined as running on a node
	// whose value of the label with key topologyKey matches that of any node on which any of the
	// selected pods is running.
	// Empty topologyKey is not allowed.
	TopologyKey string `json:"topologyKey" protobuf:"bytes,3,opt,name=topologyKey"`
	// A label query over the set of namespaces that the term applies to.
	// The term is applied to the union of the namespaces selected by this field
	// and the ones listed in the namespaces field.
	// null selector and null or empty namespaces list means "this pod's namespace".
	// An empty selector ({}) matches all namespaces.
	// This field is beta-level and is only honored when PodAffinityNamespaceSelector feature is enabled.
	// +optional
	NamespaceSelector *metav1.LabelSelector `json:"namespaceSelector,omitempty" protobuf:"bytes,4,opt,name=namespaceSelector"`
}
```
Namespaces: 不标注和值为[] 表示亲和性和反亲和性只是在此namespace 中生效
NamespaceSelector：不标注只是在此namespace 中生效
NamespaceSelector: {} 该值表示所有namespace 同一亲和性和反亲和性生效

