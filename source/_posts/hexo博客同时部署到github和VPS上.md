---
title: hexo博客同时部署到github和VPS上
categories: Kubernetes
sage: false
date: 2020-02-18 11:41:44
tags:
---

# 前言

有一台VPS一直空闲，未免有点浪费，所以想把博客部署到VPS上，并且绑定域名

之前尝试过这么做过，但是一直都没有成功，因为这其中有很多细节都是需要注意的，所以还是写一篇博客来记录这次的部署过程

以后换VPS的时候就不用像这次一样到处查找资料了

<!-- more -->

# 原理

部署到VPS的原理即是在VPS上搭建git服务器，然后每次提交public文件夹中的文件到VPS时，通过git-hooks钩子来同时复制文件到网站的根目录上

其实关键的是git服务器的搭建，因为想要免密码ssh登录的话需要配置一系列东西

部署到github就不说了，因为这个非常好部署，github其实也是一个git服务器，但是其已经搭建好了，我们只需要将公钥交给github即可完成免密码登录

# git服务器的搭建

## 安装git并配置用户与邮箱
首先创建好git用户并且输入密码

```bash
$ useradd git
```

随后就是配置git的用户名和邮箱
```bash
$ git config --global user.name "username"  
$ git config --global user.email "mail@gmail.com" 
```

配置完成后生成SSH密钥
```bash
$ ssh-keygen -t rsa -C "mail@gmail.com" 
```

我们选择保存在 /home/git/.ssh/id_rsa中，以后这个文件夹还是有用的

## 添加公钥
新建一个名为authorized_keys的文件，并将公钥复制进去

```bash
$ vim .ssh/authorized_keys
```

随后打开RSA认证

```bash
$ vim /etc/ssh/sshd_config
```

找到下面这一行，修改为

AuthorizedKeysFile .ssh/authorized_keys

这样公钥就已经配置好了

## 修改权限

修改权限这一步是非常重要的，我之前的原因应该就是卡在这一步，所以无法达到免密码ssh登录

```bash
$ chmod 700 .ssh 
$ chmod 600 .ssh/authorized_keys
$ cd /home
$ chown -R git:git git
```

关闭git的shell登录
为了安全起见，我们拒绝git的shell权限

修改下面文件内容

```bash
$ vim /etc/passwd
将

git:x:1001:1001:,,,:/home/git:/bin/bash
改成

git:x:1001:1001:,,,:/home/git:/usr/bin/git-shell
```
Nginx的配置与Githooks的脚本实现
Nginx配置文件
修改配置文件前先备份

```bash
$ cd /etc/nginx/sites-available    
$ cp default default.bak     
$ vim default
```

然后在文件中输入下面配置内容
```bash
server {
        listen 80 default;
        root /var/www/blog;
        index index.html index.htm index.nginx-debian.html;
        server_name vhyz.me www.vhyz.me;
        location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                try_files $uri $uri/ =404;
        }
        location ~* ^.+\.(css|js|txt|xml|swf|wav)$ {
                root /var/www/blog;
                access_log   off;
                expires      10m;
        }
        location ~* ^.+\.(ico|gif|jpg|jpeg|png)$ {
                root /var/www/blog;
                access_log   off;
                expires      1d;
        }
}
```

修改完保存，重启nginx

```bash
$ sudo service nginx restart
```
新建blog.git
新建blog.git裸仓库与网站根目录
```bash
$ cd /home/git
$ git init --bare blog.git
$ cd /var/www
$ mkdir blog
```
然后修改用户组权限
```bash
$ chown git:git -R /var/www/blog
$ chown git:git -R /home/git/blog.git
```
记住每个仓库都需要这样设置权限

配置Git Hooks
新建post-receive文件
```bash
$ cd /home/git/blog.git/hooks  
$ vim post-receive 
```  
然后输入下面脚本内容
```bash
#!/bin/bash
 GIT_REPO=/home/git/blog.git
 TMP_GIT_CLONE=/tmp/blog
 PUBLIC_WWW=/var/www/blog
 rm -rf ${TMP_GIT_CLONE}
 git clone $GIT_REPO $TMP_GIT_CLONE
 rm -rf ${PUBLIC_WWW}/*
 cp -rf ${TMP_GIT_CLONE}/* ${PUBLIC_WWW}

```
保存完之后赋予可执行权限
```bash
$ chmod +x post-receive
```

## hexo本地配置
在站点配置文件中deploy修改为下面内容,your_ip代表了服务器的地址

```yaml
deploy:
-   type: git
    repo: https://github.com/vhyz/vhyz.github.io
    branch: master                            
    message:                                  
-   type: git
    repo: git@your_ip:blog.git
    branch: master                            
    message:                                  
```

如果服务器端口不是默认22，则需要在本地的.ssh文件夹中创建config配置文件

输入下面内容
```bash
Host 
HostName 
User git
Port 
IdentityFile ~/.ssh/id_rsa
```
Host与HostName均为你的服务器IP地址

然后输入
```bash
hexo g -d
```
即可完成一次对两个服务器的部署

[转载自](https://vhyz.me/2018/06/09/hexo%E5%8D%9A%E5%AE%A2%E5%90%8C%E6%97%B6%E9%83%A8%E7%BD%B2%E5%88%B0github%E5%92%8CVPS%E4%B8%8A/index.html)