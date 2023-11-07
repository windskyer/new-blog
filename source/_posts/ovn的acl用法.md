---
title: ovn的acl用法
categories: Kubernetes
sage: false
date: 2023-04-10 17:47:34
tags:
---

# ovn实现ACL

**第一种方法通过k8s标准资源networkpolicy实现ACL（推荐使用）**

**什么是网络策略？**

首先NetworkPolicy是k8s的一种resource，可以通过以下三种维度对整个namespace或单一POD进行隔离：pod（允许某些pod访问）、namespace（允许某些namespace访问）、ip段（CIDR，但是pod所运行node节点始终可以访问该pod）当定义基于pod和namespace的NetworkPolicy时，需要根据标签来选择对应的pod或者namespace。

另外当需要使用NetworkPolicy资源时，k8s集群采用的网络插件必须支持，比如ovn-kubernetes等等
<!-- more -->

**隔离和非隔离pod**

默认情况所有的pod都是非隔离的，当pod被NetworkPolicy选择后，那么pod就是一个隔离的pod，出栈和入栈都要遵循NetworkPolicy的策略，并且如果有多个NetworkPolicy时，他们是叠加的而不是冲突的。

限制一个pod的NetworkPolicy需要从两个方向去考虑：ingress和egress。ingress表示入方向，egress表示出方向。

**基于namespace的网络策略**

以下yaml文件代表在app下创建一个NetworkPolicy资源，该策略作用在所有被打了app=nginx-test标签的pod，策略内容为只接受来自namespace为app的访问。

apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: ns-network-policy

  namespace: app

spec:　

  #定义策略的类型：值是一个列表，支持Ingress和Egress两种，默认Egress不做任何限制

  policyTypes: \["Ingress"\]

  #表示该策略作用的pod，以标签匹配，当该值是一个{}空字典时，代表所有POD

  podSelector:

    matchLabels:

      app: nginx-test

  #以下表示ingress策略

  ingress:

    - from:

       - namespaceSelector:

          matchLabels:

            kubernetes.io/metadata.name: app

**基于POD的网络策略**

以下yaml文件策略表示：允许带有app=nginx-test2的所有pod访问app=nginx-test的pod

apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: ns-network-policy1

  namespace: app

spec:

  policyTypes: \["Ingress"\]

  podSelector: 

    matchLabels:

      app: nginx-test

  ingress:

    - from:

       - podSelector: 

          matchLabels:

            app: nginx-test2

**基于IP段的网络策略**

以下yaml文件表示策略为：允许10.244.0.0/16段但是不包括10.244.1.0/24来访问带有app=nginx-test的pod

apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: ns-network-policy

  namespace: olda

spec:

  policyTypes: \["Ingress"\]

  podSelector: 

    matchLabels:

      app: nginx-test

  ingress:

    - from:

       - ipBlock: 

          cidr: 10.244.0.0/16

          except:

          - 10.244.1.0/24

**关于端口号的限制**

**单个端口号，只允许app名称空间下的pod连接80端口，具体策略如下**

apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: test-network-policy

  namespace: app

spec:

  policyTypes: \["Ingress"\]

  podSelector: 

    matchLabels:

      app: test

  ingress:

    - from:

       - namespaceSelector: 

          matchLabels:

            kubernetes.io/metadata.name: app

      ports:

      - protocol: TCP

        port: 80

**多个端口号，在网络策略里增加3306端口**

apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: test-network-policy

  namespace: app

spec:

  policyTypes: \["Ingress"\]

  podSelector: 

    matchLabels:

      app: test

  ingress:

    - from:

       - namespaceSelector: 

          matchLabels:

            kubernetes.io/metadata.name: app

      ports:

      - protocol: TCP

        port: 80

      - protocol: TCP

        port: 3306

**常用的网络策略具体yaml文件示例**

**拒绝所有入站流量，具体yaml文件示例：**

apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: default-deny-ingress

spec:

  podSelector: {}

  policyTypes:

  - Ingress

这确保即使没有被任何其他 NetworkPolicy 选择的 Pod 仍将被隔离以进行入口。 此策略不影响任何 Pod 的出口隔离。

**允许所有入站流量，具体yaml文件示例：**

apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: allow-all-ingress

spec:

  podSelector: {}

  ingress:

  - {}

  policyTypes:

  - Ingress

有了这个策略，任何额外的策略都不会导致到这些 Pod 的任何入站连接被拒绝。 此策略对任何 Pod 的出口隔离没有影响。

**拒绝所有出站流量，具体yaml文件示例：**

apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: default-deny-egress

spec:

  podSelector: {}

  policyTypes:

  - Egress

此策略可以确保即使没有被其他任何 NetworkPolicy 选择的 Pod 也不会被允许流出流量。 此策略不会更改任何 Pod 的入站流量隔离行为。

**允许所有出战流量，具体yaml文件示例：**

apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: allow-all-egress

spec:

  podSelector: {}

  egress:

  - {}

  policyTypes:

  - Egress

**拒绝所有入站和出站流量，具体yaml文件示例：**

apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: default-deny-all

spec:

  podSelector: {}

  policyTypes:

  - Ingress

  - Egress

**限制特定的pod访问，具体yaml文件示例：**

kind: NetworkPolicy

apiVersion: networking.k8s.io/v1

metadata:

  name: app-test

spec:

  podSelector:

    matchLabels:

      app: bookstore

      role: api

  ingress:

  - from:

      - podSelector:

          matchLabels:

            app: bookstore

**允许所有的namespace访问pod，具体yaml文件示例：**

kind: NetworkPolicy

apiVersion: networking.k8s.io/v1

metadata:

  namespace: default

  name: web-allow-all-namespaces

spec:

  podSelector:

    matchLabels:

      app: web

  ingress:

  - from:

    - namespaceSelector: {}

**允许特定的namespace访问pod，具体yaml文件示例：**

kind: NetworkPolicy

apiVersion: networking.k8s.io/v1

metadata:

  name: web-allow-prod

spec:

  podSelector:

    matchLabels:

      app: web

  ingress:

  - from:

    - namespaceSelector:

        matchLabels:

          purpose: production

**根据namespace和pod相结合的策略允许pod访问，具体yaml文件示例：**

kind: NetworkPolicy

apiVersion: networking.k8s.io/v1

metadata:

  name: web-allow-all-ns-monitoring

  namespace: default

spec:

  podSelector:

    matchLabels:

      app: web

  ingress:

    - from:

      - namespaceSelector:     # chooses all pods in namespaces labelled with team=operations

          matchLabels:

            team: operations  

        podSelector:           # chooses pods with type=monitoring

          matchLabels:

            type: monitoring

**根据特定的pod和port结合策略允许访问pod，具体yaml文件示例：**

kind: NetworkPolicy

apiVersion: networking.k8s.io/v1

metadata:

  name: api-allow-5000

spec:

  podSelector:

    matchLabels:

      app: apiserver

  ingress:

  - ports:

    - port: 5000

    from:

    - podSelector:

        matchLabels:

          role: monitoring

**限制访问外部的流量，但可以访问DNS，具体yaml文件示例：**

apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: foo-deny-egress

spec:

  podSelector:

    matchLabels:

      app: foo

  policyTypes:

  - Egress

  egress:

  # allow DNS resolution

  - to:

    - namespaceSelector:

        matchLabels:

          kubernetes.io/metadata.name: kube-system

      podSelector:

        matchLabels:

          k8s-app: kube-dns

    ports:

      - port: 53

        protocol: UDP

      - port: 53

        protocol: TCP

**设置了网络策略，如何在ovn里查看是否生成相应的ACL**

登录ovn-kubernetes的环境，使用如下命令登录如图的pod：

kubectl exec -it -n ovn-kubernetes ovnkube-master-788b596d78-l7gkt -c ovnkube-master -- bash -il

登录之后，使用如下命令，查看ACL的数据：

Ovn-nbctl find acl

此时没有设置网络策略，就没有多余的ACL，设置一个网络策略之后就会发现多一些ACL的数据：

注意：如果针对同一个规则同时设置了deny和allow的ACL，allow的优先级会比deny优先级高，所以会允许allow。

**Networkpolicy使用限制：**

通过网络策略（至少目前还）无法完成的工作，到 Kubernetes 1.26 为止，NetworkPolicy API 还不支持以下功能， 不过你可能可以使用操作系统组件（如 SELinux、OpenVSwitch、IPTables 等等） 或者第七层技术（Ingress 控制器、服务网格实现）或准入控制器来实现一些替代方案。 如果你对 Kubernetes 中的网络安全性还不太了解，了解使用 NetworkPolicy API 还无法实现下面的用户场景是很值得的。

*   强制集群内部流量经过某公用网关（这种场景最好通过服务网格或其他代理来实现）；
    
*   与 TLS 相关的场景（考虑使用服务网格或者 Ingress 控制器）；
    
*   特定于节点的策略（你可以使用 CIDR 来表达这一需求不过你无法使用节点在 Kubernetes 中的其他标识信息来辩识目标节点）；
    
*   基于名字来选择服务（不过，你可以使用 [标签](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/labels/) 来选择目标 Pod 或名字空间，这也通常是一种可靠的替代方案）；
    
*   创建或管理由第三方来实际完成的“策略请求”；
    

*   实现适用于所有名字空间或 Pods 的默认策略（某些第三方 Kubernetes 发行版本或项目可以做到这点）；
    
*   高级的策略查询或者可达性相关工具；
    
*   生成网络安全事件日志的能力（例如，被阻塞或接收的连接请求）；
    
*   显式地拒绝策略的能力（目前，NetworkPolicy 的模型默认采用拒绝操作， 其唯一的能力是添加允许策略）；
    
*   禁止本地回路或指向宿主的网络流量（Pod 目前无法阻塞 localhost 访问， 它们也无法禁止来自所在节点的访问请求）。
    

**参考资料**

[++https://kubernetes.io/docs/concepts/services-networking/network-policies/++](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

**第二种方法通过ovn-kubernetes自定义的crd并根据资源来控制实现ACL（因限制太多不推荐使用）**

**找到生成的crd的yaml文件k8s.ovn.org\_egressfirewalls.yaml**

在部署ovn的文件夹ovn/ovn-kubernetes/dist/yaml里，有一个k8s.ovn.org\_egressfirewalls.yaml文件，是根据命令生成yaml文件时生产的：

./daemonset.sh --image=yusur/ovn-daemonset-f:fullmode --net-cidr=10.123.0.0/16 --svc-cidr=10.86.0.0/16 --gateway-mode="local" --k8s-apiserver=https://192.168.122.189:6443

**创建EgressFirewall资源**

找到上述的yaml文件k8s.ovn.org\_egressfirewalls.yaml，使用命令：

kubectl apply -f k8s.ovn.org\_egressfirewalls.yaml

**修改ovnkube-master.yaml里面的参数使EgressFirewall资源生效**

如果在已经部署的环境中，请修改部署ovn的文件夹ovn/ovn-kubernetes/dist/yaml里的ovnkube-master.yaml，修改里面的- name: OVN\_EGRESSFIREWALL\_ENABLE为true

需要重新按修改后的yaml文件重新拉起这个pod

如果是即将部署的环境，请使用如下参数添加即可：

./daemonset.sh   --egress-firewall-enable=true

**设置EgressFirewall规则**

EgressFirewall特性允许集群管理员限制项目中的pod可以访问的外部主机。 

EgressFirewall对象规则适用于所有共享的pod带有egressfirewall对象的名称空间。仅一个名称空间，支持有一个EgressFirewallObject。

如下示例：

kind: EgressFirewall

apiVersion: k8s.ovn.org/v1

metadata:

  name: default  #该name必须设置为default，否则会创建失败

  namespace: default

spec:

  egress:

  - type: Allow

    to:

      dnsName: www.openvswitch.org

  - type: Allow

    to:

      cidrSelector: 1.2.3.0/24

  - type: Allow

    to:

      cidrSelector: 4.5.6.0/24

    ports:

      - protocol: UDP

        port: 55

  - type: Deny

    to:

      cidrSelector: 0.0.0.0/0

这个例子允许连接到默认名称空间中的Pods，www.openvswitch.org转换为的主机，任何外部主机在1.2.3.0到1.2.3.255范围内的主机，并且允许添加只有端口上的UDP协议才会将流量设置为4.5.6.0到4.5.6.255第55号，并拒绝所有其他外部主机的流量。端口是可选的，允许用户指定特定的端口To和协议允许或拒绝流量。规则的优先级由其在出口中的位置决定数组中。较早的规则在较晚的规则之前被处理。在前面的例子，如果规则颠倒过来，所有流量都被拒绝，包括到1.2.3.0/24 CIDR块中的主机的任何流量。使用DNS特性假设已定位节点和主节点 在与添加到ovn的DNS条目类似的位置数据库是由主机生成的。 注意:在deny规则中使用DNS名称时，请谨慎使用。DNS拦截器将永远不会完美地工作，并可能允许访问一个被拒绝的主机，如果节点上的DNS解析与主节点上的DNS解析不同。

使用kubectl apply -f xxx.yaml文件之后，同样会在ovn-master的pod里面看见新增加的ACL
