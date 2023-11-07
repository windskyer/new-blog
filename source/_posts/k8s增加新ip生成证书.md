---
title: k8s增加新ip生成证书
categories: Kubernetes
sage: false
date: 2023-04-26 11:50:43
tags: k8s
---

## 背景原因

环境为双网卡环境，默认部署使用单一网卡的ip 证书信任， 另外一网卡ip没做证书信任，现在需要重新生成证书来添加ip信任，让apiserver 支持多ip地址访问

## 备份 kubernetes 目录

    cp -r /etc/kubernetes{,-bak}

<!-- more -->

#### 查看证书内的 ip

    for i in $(find /etc/kubernetes/pki -type f -name "*.crt");do echo ${i} && openssl x509 -in ${i} -text | grep 'DNS:';done

可以看到，只有 apiserver 和 etcd 的证书里面是包含了 ip 的

    root@yusur-62:~# for i in $(find /etc/kubernetes/pki -type f -name "*.crt");do echo ${i} && openssl x509 -in ${i} -text | grep 'DNS:';done
    /etc/kubernetes/pki/etcd/server.crt
                    DNS:localhost, DNS:yusur-62, IP Address:192.168.28.62, IP Address:127.0.0.1, IP Address:0:0:0:0:0:0:0:1
    /etc/kubernetes/pki/etcd/peer.crt
                    DNS:localhost, DNS:yusur-62, IP Address:192.168.28.62, IP Address:127.0.0.1, IP Address:0:0:0:0:0:0:0:1
    /etc/kubernetes/pki/etcd/healthcheck-client.crt
    /etc/kubernetes/pki/etcd/ca.crt
                    DNS:etcd-ca
    /etc/kubernetes/pki/apiserver.crt
                    DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster.local, DNS:yusur-62, IP Address:10.86.0.1, IP Address:192.168.28.62
    /etc/kubernetes/pki/front-proxy-client.crt
    /etc/kubernetes/pki/apiserver-kubelet-client.crt
    /etc/kubernetes/pki/front-proxy-ca.crt
                    DNS:front-proxy-ca
    /etc/kubernetes/pki/apiserver-etcd-client.crt
    /etc/kubernetes/pki/ca.crt
                    DNS:kubernetes

#### 生成集群配置

查看集群kubeadm 配置信息

    # 查看集群kubeadm 配置信息
    kubectl -n kube-system get cm kubeadm-config -o jsonpath={.data.ClusterConfiguration} | tee /root/kubeadm.yaml
    apiServer:
      extraArgs:
        authorization-mode: Node,RBAC
      timeoutForControlPlane: 4m0s
    apiVersion: kubeadm.k8s.io/v1beta3
    certificatesDir: /etc/kubernetes/pki
    clusterName: kubernetes
    controllerManager: {}
    dns: {}
    etcd:
      local:
        dataDir: /var/lib/etcd
    imageRepository: registry.aliyuncs.com/google_containers
    kind: ClusterConfiguration
    kubernetesVersion: v1.24.10
    networking:
      dnsDomain: cluster.local
      podSubnet: 10.123.0.0/16
      serviceSubnet: 10.86.0.0/16

增加新ip

    # 编辑配资文件vim /root/kubeadm.yaml
    apiServer:
      # 增加下面的配置，添加master节点多个网卡ip地址
      certSANs:
      - 192.168.28.62
      - 172.16.10.62
      # 增加上面的配置
      extraArgs:
        authorization-mode: Node,RBAC
      timeoutForControlPlane: 4m0s
    apiVersion: kubeadm.k8s.io/v1beta3
    certificatesDir: /etc/kubernetes/pki
    clusterName: kubernetes
    controllerManager: {}
    dns: {}
    etcd:
      local:
        dataDir: /var/lib/etcd
        # 增加下面的配置，添加master节点多个网卡ip地址
        serverCertSANs:
        - 192.168.28.62
        - 172.16.10.62
        # 增加上面的配置
    imageRepository: registry.aliyuncs.com/google_containers
    kind: ClusterConfiguration
    kubernetesVersion: v1.24.10
    networking:
      dnsDomain: cluster.local
      podSubnet: 10.123.0.0/16
      serviceSubnet: 10.86.0.0/16

## 删除原有的证书

需要保留 ca ，sa，front-proxy 这三个证书

    rm -rf /etc/kubernetes/pki/{apiserver*,front-proxy-client*}
    rm -rf /etc/kubernetes/pki/etcd/{healthcheck*,peer*,server*}

## 重新生成证书

#### 更新证书

    kubeadm init phase certs all --config /root/kubeadm.yaml

#### 再次查看证书内的 ip

    for i in $(find /etc/kubernetes/pki -type f -name "*.crt");do echo ${i} && openssl x509 -in ${i} -text | grep 'DNS:';done

这里可以得到验证，不会覆盖之前证书内已经有的 ip，会将新的 ip 追加到后面

    /etc/kubernetes/pki/etcd/server.crt
                    DNS:localhost, DNS:yusur-62, IP Address:192.168.28.62, IP Address:127.0.0.1, IP Address:0:0:0:0:0:0:0:1, IP Address:172.16.10.62
    /etc/kubernetes/pki/etcd/peer.crt
                    DNS:localhost, DNS:yusur-62, IP Address:192.168.28.62, IP Address:127.0.0.1, IP Address:0:0:0:0:0:0:0:1
    /etc/kubernetes/pki/etcd/healthcheck-client.crt
    /etc/kubernetes/pki/etcd/ca.crt
                    DNS:etcd-ca
    /etc/kubernetes/pki/apiserver.crt
                    DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster.local, DNS:yusur-62, IP Address:10.86.0.1, IP Address:192.168.28.62, IP Address:172.16.10.62
    /etc/kubernetes/pki/front-proxy-client.crt
    /etc/kubernetes/pki/apiserver-kubelet-client.crt
    /etc/kubernetes/pki/front-proxy-ca.crt
                    DNS:front-proxy-ca
    /etc/kubernetes/pki/apiserver-etcd-client.crt
    /etc/kubernetes/pki/ca.crt
                    DNS:kubernetes

## 将配置更新到CM中

这样，以后有升级，或者增加其他 ip 时，也会将配置的 CertSANs 的 ip 保留下来，方便以后删减

    kubeadm init phase upload-config kubeadm --config /root/kubeadm.yaml
