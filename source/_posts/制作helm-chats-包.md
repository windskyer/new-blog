---
title: 制作helm chats 包
categories: Kubernetes
sage: false
date: 2023-04-20 17:51:40
tags: helm
---

## helm3安装方法

#### 使用脚本安装

Helm现在有个安装脚本可以自动拉取最新的Helm版本并在 [本地安装](https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3)。

您可以获取这个脚本并在本地执行。它良好的文档会让您在执行之前知道脚本都做了什么。

    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
    chmod 700 get_helm.sh
    ./get_helm.sh

如果想直接执行安装，运行curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash。

<!-- more -->
#### 通过包管理器安装

Helm社区成员贡献了针对Apt的一个 helm包通常是最新的。

    curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
    sudo apt-get install apt-transport-https --yes
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
    sudo apt-get update
    sudo apt-get install helm

参考文档：[https://helm.sh/zh/docs/](https://helm.sh/zh/docs/)

#### helm操作命令

添加公共仓库

    # 也可以换成微软的源，速度快，内容和官方同步的 
    helm repo add stable http://mirror.azure.cn/kubernetes/charts              
    helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts

查看仓库

    helm repo ls
    NAME      	URL                                                   
    my-repo   	https://charts.bitnami.com/bitnami                    
    yusur-repo	http://10.2.20.7:8081                                 
    harbor    	https://harbor.yusur.tech/chartrepo/leid              
    stable    	http://mirror.azure.cn/kubernetes/charts              
    aliyun    	https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts

#### helm安装插件

安装push插件

    helm plugin install https://github.com/chartmuseum/helm-push
    
    Downloading and installing helm-push v0.10.3 ...
    https://github.com/chartmuseum/helm-push/releases/download/v0.10.3/helm-push_0.10.3_darwin_arm64.tar.gz
    Installed plugin: cm-push

## 制作helm包

#### 创建chart包

    # 创建新chart包
    helm create mychart
    Creating mychart

## 搭建私有helm仓库

Chartmuseum 除了给我们提供一个类似于web服务器的功能之外，还提供了其他有用的功能，便于日常我们私有仓库的管理。

*   根据chart文件自动生成index.yaml(无须使用helm repo index手动生成)
    
*   helm push的插件，可以在helm命令之上实现将chart文件推送到chartmuseum上
    
*   相应的tls配置，Basic认证，JWT认证（Bearer token认证）
    
*   提供了Restful的api（可以使用curl命令操作）和可以使用的cli命令行工具
    
*   提供了各种后端存储的支持（Amazon s3, Google Cloud Storage, 阿里、百度、腾讯，开源对象存储等）
    
*   提供了Prometheus的集成，对外提供自己的监控信息。
    
*   没有用户的概念，但是基于目录实现了一定程度上的多租户的需求。
    

#### Chartmuseum 搭建

直接使用最简单的 docker run 方式，使用local 本地存储方式，通过 -v 映射到宿主机 /opt/charts

更多支持安装方式见官网

    mkdir /opt/charts
    docker run -d \
      --name chartmuseum \
      -u 0:0 \
      -p 8081:8080 \
      -e DEBUG=1 \
      -e STORAGE=local \
      -e STORAGE_LOCAL_ROOTDIR=/charts \
      -v /opt/charts:/charts \
      chartmuseum/chartmuseum:latest
    
    # 使用 curl 测试下接口，没有报错就行，当前仓库内容还是空的
    # curl localhost:8081/api/charts
    {}

#### 添加私有仓库Chartmuseum

    helm repo add yusur-repo http://10.2.20.7:8081

#### cm-push chats包到Chartmuseum

    # 添加cm-push 插件
    helm plugin install https://github.com/chartmuseum/helm-push
    
    # 使用cm-push上传chat 包
    helm cm-push mariadb-11.5.7.tgz yusur-repo
    
    # 使用cm-push上传chat 目录
    helm cm-push ./mychart yusur-repo

## 复用harbor仓库

harbor自带helm私有仓库的功能，不需要再部署一个helm私有仓库，在这里给大家介绍一下helm如何上传chart包到harbor

#### cm-push chats包到harbor

添加私有仓库harbor

    # 添加repo
    helm repo add harbor https://harbor.yusur.tech/chartrepo/library --username xxx --password xxx

    # 使用cm-push 命令 上传chat包
    helm cm-push mariadb-11.5.7.tgz harbor
    
    # 上传cm-push chat目录
    helm cm-push ./mychart harbor

#### push chats包到harbor

使用oci协议

    # 登录到注册中心
    helm registry login -u leid harbor.yusur.tech -p xxxxxx

    # 使用push 命令上传chat包
    helm push mychart-0.1.0.tgz oci://harbor.yusur.tech/leid
    
    # 使用pull 下载包
    helm pull oci://harbor.yusur.tech/leid/mychart --version 0.1.0
    
    # 查看chat 信息
    helm show all oci://harbor.yusur.tech/leid/mychart --version 0.1.0
