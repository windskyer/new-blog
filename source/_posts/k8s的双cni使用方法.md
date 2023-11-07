---
title: k8s的双cni使用方法
categories: Kubernetes
sage: false
date: 2023-04-07 18:07:50
tags: k8s
---

## Multus CNI 简介

> **Multus CNI enables attaching multiple network interfaces to pods in Kubernetes.**

以上是 Multus CNI 项目官方对其存在意义的精简描述，它的存在就是帮助 K8s 的 Pod（可简单理解为一组容器的集合，是 K8s 可管理的最小“容器”单位）建立多网络接口。

Multus CNI 本身不提供网络配置功能，它是通过用其他满足 CNI 规范的插件进行容器的网络配置。
<!-- more -->

## 创建CNI网络

##### calico网络的NetworkAttachmentDefinition

    apiVersion: "k8s.cni.cncf.io/v1"
    kind: NetworkAttachmentDefinition
    metadata:
      name: calico-net #网络名称
      namespace: kube-system
    spec:
      config: '{
      "name": "calico-net",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "calico",
          "log_level": "info",
          "log_file_path": "/var/log/calico/cni/cni.log",
          "datastore_type": "kubernetes",
          "mtu": 0,
          "ipam": {
              "type": "calico-ipam"
          },
          "policy": {
              "type": "k8s"
          },
          "kubernetes": {
              "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
          }
        },
        {
          "type": "portmap",
          "snat": true,
          "capabilities": {"portMappings": true}
        },
        {
          "type": "bandwidth",
          "capabilities": {"bandwidth": true}
        }
      ]
    }'
    

##### ovn网络的NetworkAttachmentDefinition

    apiVersion: "k8s.cni.cncf.io/v1"
    kind: NetworkAttachmentDefinition
    metadata:
      name: ovn-net #网络名称
      namespace: kube-system
      annotations:
        k8s.v1.cni.cncf.io/resourceName: yusur.tech/sriov_dpu
    spec:
      config: '{
        "cniVersion": "0.4.0",
        "name": "ovn-kubernetes",
        "type": "ovn-k8s-cni-overlay",
        "ipam": {},
        "dns": {},
        "logFile": "/var/log/ovn-kubernetes/ovn-k8s-cni-overlay.log",
        "logLevel": "5",
        "logfile-maxsize": 100,
        "logfile-maxbackups": 5,
        "logfile-maxage": 5
        }'
    

## 介绍CNI网络

##### 关键字段说明

:::
v1.multus-cni.io/default-network: calico-net 

#该字段信息表示pod使用calico 网络， calico-net 必须是上面创建的NetworkAttachmentDefinition 对象名称

k8s.v1.cni.cncf.io/networks: '\[{

      "name": "ovn-net",

      "namespace": "kube-system"

      "interface": "eth1",

      "default-route": \["10.124.0.1"\]

    }\]' #该字段表示使用多个辅助cni，ovn-net 和kube-system 必须是上面创建NetworkAttachmentDefinition对象名称
:::

## 使用CNI网络

#### 单CNI

##### Calico网络

    apiVersion: v1
    kind: Service
    metadata:
      name: nginx2-calico
      namespace: default
    spec:
      selector:
        app: nginx2-calico
      ports:
      - protocol: TCP
        port: 80
        targetPort: server
    
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx2-calico
      namespace: default
      annotations:
        #单独使用calico网络
        v1.multus-cni.io/default-network: calico-net
      labels:
        app: nginx2-calico
    spec:
      containers:
      - image: ubuntu:22.04-yusur
        command: ['/bin/sh', '-c', 'nginx -g "daemon off;"']
        imagePullPolicy: IfNotPresent
        name: nginx2-calico
        securityContext:
          runAsUser: 0
          privileged: true
        ports:
        - containerPort: 80
          name: server
      restartPolicy: Always
      nodeSelector:
        kubernetes.io/hostname: host-205

##### OVN网络

    apiVersion: v1
    kind: Service
    metadata:
      name: nginx2-ovn
      namespace: default
    spec:
      selector:
        app: nginx2-ovn
      ports:
      - protocol: TCP
        port: 80
        targetPort: server
    
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx2-ovn
      namespace: default
      annotations:
        #单独使用ovn网络
        v1.multus-cni.io/default-network: ovn-net
      labels:
        app: nginx2-ovn
    spec:
      containers:
      - image: ubuntu:22.04-yusur
        command: ['/bin/sh', '-c', 'nginx -g "daemon off;"']
        imagePullPolicy: IfNotPresent
        name: nginx2-ovn
        securityContext:
          runAsUser: 0
          privileged: true
        ports:
        - containerPort: 80
          name: server
        resources:
          requests:
            yusur.tech/sriov_dpu: '1'
          limits:
            yusur.tech/sriov_dpu: '1'
      restartPolicy: Always
      nodeSelector:
        kubernetes.io/hostname: host-205
        k8s.ovn.org/dpu-host: ""

#### 双CNI 

##### Calico为主，OVN为辅

###### 默认路由为Calico

++暂时不支持，需研发（该期需求）++

###### 默认路由为OVN

    apiVersion: v1
    kind: Service
    metadata:
      name: nginx2-calico-ovn
      namespace: default
    spec:
      selector:
        app: nginx2-calico-ovn
      ports:
      - protocol: TCP
        port: 80
        targetPort: server
    
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx2-calico-ovn
      namespace: default
      annotations:
        #使用calico网络为主，ovn为辅
        v1.multus-cni.io/default-network: calico-net
        k8s.v1.cni.cncf.io/networks: '[{
          "name": "ovn-net",
          "namespace": "kube-system",
          "interface": "eth1",
          "default-route": ["10.124.2.1"]
        }]'
      labels:
        app: nginx2-calico-ovn
    spec:
      containers:
      - image: ubuntu:22.04-yusur
        command: ['/bin/sh', '-c', 'nginx -g "daemon off;"']
        imagePullPolicy: IfNotPresent
        name: nginx2-calico-ovn
        securityContext:
          runAsUser: 0
          privileged: true
        ports:
        - containerPort: 80
          name: server
        resources:
          requests:
            yusur.tech/sriov_dpu: '1'
          limits:
            yusur.tech/sriov_dpu: '1'
      restartPolicy: Always
      nodeSelector:
        kubernetes.io/hostname: host-205
        k8s.ovn.org/dpu-host: ""

##### OVN为主，Calico为辅

###### 默认路由为Calico

    apiVersion: v1
    kind: Service
    metadata:
      name: nginx2-ovn-calico
      namespace: default
    spec:
      selector:
        app: nginx2-ovn-calico
      ports:
      - protocol: TCP
        port: 80
        targetPort: server
    
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx2-ovn-calico
      namespace: default
      annotations:
        #使用ovn网络为主，calico为辅
        v1.multus-cni.io/default-network: ovn-net
        k8s.v1.cni.cncf.io/networks: '[{
          "name": "calico-net",
          "namespace": "kube-system",
          "interface": "eth1",
          "default-route": ["169.254.1.1"]
        }]'
      labels:
        app: nginx2-ovn-calico
    spec:
      containers:
      - image: ubuntu:22.04-yusur
        command: ['/bin/sh', '-c', 'nginx -g "daemon off;"']
        imagePullPolicy: IfNotPresent
        name: nginx2-ovn-calico
        securityContext:
          runAsUser: 0
          privileged: true
        ports:
        - containerPort: 80
          name: server
        resources:
          requests:
            yusur.tech/sriov_dpu: '1'
          limits:
            yusur.tech/sriov_dpu: '1'
      restartPolicy: Always
      nodeSelector:
        kubernetes.io/hostname: host-205
        k8s.ovn.org/dpu-host: ""

###### 默认路由为OVN

++暂时不支持++