---
title: kube-proxy支持多CNI网络
categories: Kubernetes
tags: k8s
sage: false
hidden: false
date: 2023-11-07 16:00:56
---

# 简介

Kube-proxy 是 Kubernetes 集群中的一个关键组件，负责实现 Kubernetes 服务发现和负载均衡。它运行在每个节点上，以确保网络流量正确路由到集群中的服务。
以下是 Kube-proxy 的一些主要功能和特点：

服务代理：Kube-proxy会监视 Kubernetes API Server 中的 Service 和 Endpoints 对象，以了解集群中的服务和其后端 Pod。当有新服务创建或更新时，Kube-proxy会自动更新代理规则，以确保服务可用性。

IP 负载均衡：Kube-proxy通过维护 IP 负载均衡规则，将请求分发到服务的后端 Pod。这有助于实现服务的负载均衡，确保请求能够均匀地分发到多个 Pod 上。

<!-- more -->

面向用户的服务发现：Kube-proxy允许用户通过 Kubernetes 的服务抽象来访问应用程序，而不需要了解后端 Pod 的详细信息。用户只需使用服务名称和端口号，而不必担心具体的 Pod IP 地址。

支持多种代理模式：Kube-proxy支持多种代理模式，包括iptables、IPVS、和 Windows 等。这些模式允许根据集群的具体需求选择不同的代理方式。

节点故障处理：Kube-proxy会监视节点上的 Pod 和服务，并在节点故障时自动处理重新路由流量，以确保服务的高可用性。

安全策略支持：Kube-proxy也可以与网络策略 (Network Policies) 集成，允许管理员定义网络访问控制规则，以保护集群中的服务。

总之，Kube-proxy是 Kubernetes 集群中的一个关键组件，它简化了服务发现和负载均衡的管理，使用户能够轻松地访问和管理其应用程序，同时确保高可用性和网络安全。不过需要注意的是，Kubernetes 1.20版本及以后，Kube-proxy引入了一个新的模式，即"Kube-proxy mode"，用于简化其设计和维护。因此，Kube-proxy的具体实现细节可能会随着Kubernetes版本的演进而变化。

# 图解
![1](kube-proxy支持多CNI网络/1.png)

![2](kube-proxy支持多CNI网络/2.png)

# 参数

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    k8s-app: kube-proxy
  name: kube-proxy
  namespace: kube-system
spec:
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kube-proxy
  template:
    metadata:
      creationTimestamp: null
      labels:
        k8s-app: kube-proxy
    spec:
      containers:
      - command:
        - /usr/local/bin/kube-proxy
        - --config=/var/lib/kube-proxy/config.conf
        - --hostname-override=$(NODE_NAME)
        - --exclude-cluster-cidr=10.244.0.0/16, # 设置需要排除的pod cidr
        - --v=5
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: spec.nodeName
        image: flftuu/kube-proxy-amd64:v1.24.10-dirty
        imagePullPolicy: IfNotPresent
```

# 镜像

```shell
docker pull flftuu/kube-proxy-amd64:v1.24.10-dirty
```

# 效果

```shell
[root@master ~]# kubectl get ep -n ovn-poc 
NAME        ENDPOINTS          AGE
service-a   10.244.0.11:80     29h
service-b   10.144.219.69:80   29h
[root@master ~]# 
[root@master ~]# kubectl get svc -n ovn-poc 
NAME        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
service-a   ClusterIP   10.96.164.53   <none>        10001/TCP   29h
service-b   ClusterIP   10.96.67.152   <none>        10001/TCP   29h

# 查看iptables 规则
[root@master ~]# iptables -L -vn -t nat | grep 10.244.0.11
[root@master ~]# iptables -L -vn -t nat | grep 10.144.219.69
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.144.219.69        0.0.0.0/0            /* ovn-poc/service-b:http */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* ovn-poc/service-b:http */ tcp to:10.144.219.69:80
    0     0 KUBE-SEP-JK5DYGPVQ34YI2X6  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* ovn-poc/service-b:http -> 10.144.219.69:80 */
```