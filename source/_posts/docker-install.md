---
title: CentOS7 安装Docker-CE
date: 2019-03-26 11:56:02
tags: docker
categories: Kubernetes
---

# 1 卸载老版本Docker

```bash
 sudo yum remove docker \
              docker-common \
              docker-selinux \
              docker-engine

```

<!-- more -->

# 2 设置仓库

```bash
sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
```

```bash
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

<amp-auto-ads type="adsense" data-ad-client="ca-pub-5216394795966395">docker</amp-auto-ads>

# 3 安装docker

```bash
 sudo yum install docker-ce
 ```

# 4 启动Docker

```bash
sudo systemctl start docker
```

# 5 卸载Docker CE

```bash
sudo yum remove docker-ce
sudo rm -rf /var/lib/docker
```