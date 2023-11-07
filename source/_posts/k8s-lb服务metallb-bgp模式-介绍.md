---
title: "k8s\_lb服务metallb(bgp模式)介绍"
categories: Kubernetes
tags: k8s
sage: false
hidden: false
date: 2023-06-08 14:05:00
---

## metallb 介绍

MetalLB是一个为基础 Kubernetes集群提供负载均衡实现的工具，使用标准路由协议。

Kubernetes在基础集群中不提供网络负载均衡器（类型为LoadBalancer的服务）的实现。Kubernetes提供的网络负载平衡器实现都是调用各种IaaS平台（如GCP、AWS、Azure等）的接口代码。如果您没有运行在受支持的IaaS平台上（如GCP、AWS、Azure等），则创建时LoadBalancers将无限期处于“挂起”状态。

在基础集群中，操作员只有两个接口来将用户流量引入他们的集群，“NodePort”和“externalIPs”服务。

这两个选项在生产使用中都有显著的缺点，这使得基础集群成为 Kubernetes 生态系统中的二等公民。

MetalLB旨在通过提供与标准网络设备集成的网络负载均衡器实现来解决这种不平衡，以便基础群集上的external services尽可能“正常工作”。
<!-- more -->
## 前置要求

* 需要运行Kubernetes 1.13.0或更高版本的Kubernetes集群，该集群尚未具备网络负载平衡功能。

* 可以与MetalLB共存的集群网络（如 calico）。

* 一些IPv4地址供MetalLB分配。

* 当使用BGP操作模式时，您将需要一个或多个能够使用BGP协议进行通信的路由器。

* 当使用L2操作模式时，节点之间必须允许端口7946上（TCP和UDP，其他端口可配置）的流量通过，这是memberlist所需的。

## 安装 metallb 服务

如果您正在使用IPVS模式下的kube-proxy，则自Kubernetes v1.14.2起，您必须启用严格的ARP模式。

请注意，如果您使用kube-router作为服务代理，则不需要此操作，因为它默认启用了 strict ARP。

您可以通过编辑当前集群中的kube-proxy配置来实现此操作：

    kubectl edit configmap -n kube-system kube-proxy
    # set
    apiVersion: kubeproxy.config.k8s.io/v1alpha1
    kind: KubeProxyConfiguration
    mode: "ipvs"
    ipvs:
      strictARP: true
    
    
    
    # see what changes would be made, returns nonzero returncode if different
    kubectl get configmap kube-proxy -n kube-system -o yaml | \
    sed -e "s/strictARP: false/strictARP: true/" | \
    kubectl diff -f - -n kube-system
    
    # actually apply the changes, returns nonzero returncode on errors only
    kubectl get configmap kube-proxy -n kube-system -o yaml | \
    sed -e "s/strictARP: false/strictARP: true/" | \
    kubectl apply -f - -n kube-system
    
    # delete pods
     kubectl  delete pods -l k8s-app=kube-proxy -n kube-system

#### 创建 metallb

    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/main/config/manifests/metallb-native.yaml
    
    
    # 查看对应的metalb  controller控制器pod 是否启动
    root@master001:~/lb# kubectl  -n metallb-system  get pods -l app=metallb
    NAME                                                   READY   STATUS    RESTARTS   AGE
    controller-848965b68-8slv7                             1/1     Running   0          8m8s
    speaker-5hx2t                                          1/1     Running   0          8m8s
    speaker-8vbmh                                          1/1     Running   0          8m8s
    speaker-r46dt                                          1/1     Running   0          8m8s
    
    
    # pod 介绍
    这将在 metallb-system 命名空间下部署 MetalLB。清单中包含以下组件：
    
    - metallb-system/controller 部署。它是处理 IP 地址分配的集群控制器。
    
    - metallb-system/speaker 守护进程集。它是使用您选择的协议来使服务可达的组件。
    
    - controller和 speaker 的服务帐户，以及组件需要运行所需的 RBAC 权限。

#### 创建 IP pool 池

IPAddressPool 是定义要分配给负载均衡器服务的IP地址

使用所需的IP创建IPAddressPool。将autoAssign设置为false是有意义的，这样它就不会被其他服务错误地占用 - 我们的服务将显式请求该池。

    cat <<EOF >ip.yaml
    apiVersion: metallb.io/v1beta1
    kind: IPAddressPool
    metadata:
      name: example-pool
      namespace: metallb-system
    spec:
      addresses:
      - 172.19.0.0/24
      autoAssign: false
    EOF

IPAddressPools可以同时存在多个实例，地址可以通过CIDR，范围定义，并且可以分配IPv4和IPv6地址。

#### 创建 bgp adv

    cat <<EOF >adv.yaml
    apiVersion: metallb.io/v1beta1
    kind: BGPAdvertisement
    metadata:
      name: example-bgp-adv
      namespace: metallb-system
    spec:
      ipAddressPools:
      - example-pool
        #  nodeSelectors:
        #  - matchLabels:
        #      egress-service.k8s.ovn.org/some-namespace-example-service: ""
    EOF

#### 创建 bgp peer

    cat <<EOF > peer.yaml
    apiVersion: metallb.io/v1beta2
    kind: BGPPeer
    metadata:
      name: example
      namespace: metallb-system
    spec:
      myASN: 64512
      peerASN: 65001
      peerAddress: 192.168.122.1
      peerPort: 179
    EOF

配置路由器 bgp 功能

[《LB服务和BGP路由器测试》]()

#### 创建 LB SVC

    cat <<EOF > svc.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: example-service
      annotations:
        # 指定lb的地址池
        metallb.universe.tf/address-pool: example-pool
    spec:
      selector:
        app: nginx1-ovn
      ports:
        - name: http
          protocol: TCP
          port: 80
          targetPort: 80
      type: LoadBalancer
    
    EOF
    
    kubectl apply -f svc.yaml

#### 查看 LB SVC

    kubectl  describe svc example-service
    Name:                     example-service
    Namespace:                default
    Labels:                   <none>
    Annotations:              metallb.universe.tf/address-pool: example-pool
                              metallb.universe.tf/ip-allocated-from-pool: example-pool
    Selector:                 app=nginx1-ovn
    Type:                     LoadBalancer
    IP Family Policy:         SingleStack
    IP Families:              IPv4
    IP:                       10.86.73.55
    IPs:                      10.86.73.55
    LoadBalancer Ingress:     172.19.0.100 # 分配的lb地址
    Port:                     http  80/TCP
    TargetPort:               80/TCP
    NodePort:                 http  32541/TCP
    Endpoints:                10.124.0.3:80
    Session Affinity:         None
    External Traffic Policy:  Cluster

#### 高级用法

参考：[https://metallb.universe.tf/configuration/\_advanced\_bgp\_configuration/](https://metallb.universe.tf/configuration/_advanced_bgp_configuration/)
