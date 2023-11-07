---
title: kubeadm 安装v1.24.x版本k8s
categories: Kubernetes
sage: false
date: 2023-02-03 14:53:00
tags: k8s
---

### ubuntu安装containerd
以下以Ubuntu为例
说明：安装containerd与安装docker流程基本一致，差别在于不需要安装docker-ce
>1. containerd: apt-get install -y containerd.io
>2. docker: apt-get install docker-ce docker-ce-cli containerd.io
<!-- more -->

***卸载旧版本***
```sh
sudo apt-get remove docker docker-engine docker.io containerd runc
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
```

***准备包环境***
更新apt，允许使用https
```sh
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```
添加docker官方GPG key
```sh
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```
设置软件仓库源
```sh
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

***安装containerd***
```sh
# 安装containerd
sudo apt-get update
sudo apt-get install -y containerd.io

# 如果是安装docker则执行：
#sudo apt-get install docker-ce docker-ce-cli containerd.io

# 查看运行状态
systemctl enable containerd
systemctl status containerd
```
或者安装指定版本
```sh
# 查看版本
apt-cache madison containerd

# 指定版本
sudo apt-get install containerd=<VERSION>
```
***修改配置***
在 Linux 上，containerd 的默认 CRI 套接字是 /run/containerd/containerd.sock

生成默认配置
```sh
containerd config default > /etc/containerd/config.toml
```
修改CgroupDriver为systemd
k8s官方推荐使用systemd类型的CgroupDriver
```sh
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```

重启containerd
```sh
systemctl restart containerd
```

### 离线二进制安装containerd

把containerd、runc、cni-plugins、nerdctl二进制下载到本地，再上传到对应服务器，解压文件到对应目录，修改containerd配置文件，启动containerd。
```sh
#!/bin/bash
set -e

ContainerdVersion=$1
ContainerdVersion=${ContainerdVersion:-1.6.6}

RuncVersion=$2
RuncVersion=${RuncVersion:-1.1.3}

CniVersion=$3
CniVersion=${CniVersion:-1.1.1}

NerdctlVersion=$4
NerdctlVersion=${NerdctlVersion:-0.21.0}

CrictlVersion=$5
CrictlVersion=${CrictlVersion:-1.24.2}

echo "--------------install containerd--------------"
wget https://github.com/containerd/containerd/releases/download/v${ContainerdVersion}/containerd-${ContainerdVersion}-linux-amd64.tar.gz
tar Cxzvf /usr/local containerd-${ContainerdVersion}-linux-amd64.tar.gz

echo "--------------install containerd service--------------"
wget https://raw.githubusercontent.com/containerd/containerd/681aaf68b7dcbe08a51c3372cbb8f813fb4466e0/containerd.service
mv containerd.service /lib/systemd/system/

mkdir -p /etc/containerd/
containerd config default > /etc/containerd/config.toml

echo "--------------install runc--------------"
wget https://github.com/opencontainers/runc/releases/download/v${RuncVersion}/runc.amd64
chmod +x runc.amd64
mv runc.amd64 /usr/local/bin/runc

echo "--------------install cni plugins--------------"
wget https://github.com/containernetworking/plugins/releases/download/v${CniVersion}/cni-plugins-linux-amd64-v${CniVersion}.tgz
rm -fr /opt/cni/bin
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v${CniVersion}.tgz

echo "--------------install nerdctl--------------"
wget https://github.com/containerd/nerdctl/releases/download/v${NerdctlVersion}/nerdctl-${NerdctlVersion}-linux-amd64.tar.gz
tar Cxzvf /usr/local/bin nerdctl-${NerdctlVersion}-linux-amd64.tar.gz

echo "--------------install crictl--------------"
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v${CrictlVersion}/crictl-v${CrictlVersion}-linux-amd64.tar.gz
tar Cxzvf /usr/local/bin crictl-v${CrictlVersion}-linux-amd64.tar.gz

# 启动containerd服务
systemctl daemon-reload
systemctl restart contaienrd
```

### 安装crictl
Containerd中默认带有ctr命令工具，它是一个简单的 CLI 接口，用作 containerd 本身的一些调试用途，投入生产使用时还是应该配合docker 或者 cri-containerd。

crictl是一个命令行接口，用于与CRI兼容的容器运行时。你可以使用它来检查和调试Kubernetes节点上的容器运行时和应用程序。crictl及其源代码托管在[cri-tools](https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md)仓库中。

```sh
VERSION="v1.26.0"
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-$VERSION-linux-amd64.tar.gz
```

***配置 crictl***
要查看或编辑当前配置，请查看或编辑/etc/crictl.yaml的内容
```sh
cat /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: true

# /run/containerd/containerd.sock 为containerd中的“grpc.address” 配置，两个需要对应上。
```

***crictl  测试***
```sh
root@work001:/tmp# crictl version 
Version:  0.1.0
RuntimeName:  containerd
RuntimeVersion:  1.6.16
RuntimeApiVersion:  v1
```

### 安装k8s

***安装kubeadm，kubelet, kubectl 指定版本***
```sh
sudo apt-get install -y kubelet=1.24.10-00 kubeadm=1.24.10-00 kubectl=1.24.10-00 --allow-downgrades

# 锁定版本不升级
sudo apt-mark hold kubelet kubeadm kubectl
```

***安装master节点***
```sh
# 创建master结点
sudo kubeadm init --kubernetes-version=v1.24.10 --apiserver-advertise-address=192.168.122.10 --image-repository registry.aliyuncs.com/google_containers --pod-network-cidr=10.123.0.0/16 --service-cidr=10.86.0.0/16

# 不安装kube-proxy 参数 --kubernetes-version=v1.24.10
# 输出node加入集群命令：kubeadm token create --print-join-command
```
***加入node节点***
```sh
# 登入各个节点
kubeadm join 192.168.122.10:6443 --token d2171t.6f3ybg6j313440qz --discovery-token-ca-cert-hash sha256:6674692af2959148fa84205b628646e03e81da4d75e988ba742909e821b6511a 
```