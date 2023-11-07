---
title: dockerfile
date: 2019-03-21 17:41:19
tags: docker
categories: Kubernetes
---

# 1. Dockerfile的说明

dockerfile指令忽略大小写，建议大写，#作为注释，每行只支持一条指令，指令可以带多个参数。

dockerfile指令分为构建指令和设置指令。

1. 构建指令：用于构建image，其指定的操作不会在运行image的容器中执行。
2. 设置指令：用于设置image的属性，其指定的操作会在运行image的容器中执行。

<!-- more -->

<amp-auto-ads type="adsense" data-ad-client="ca-pub-5216394795966395"></amp-auto-ads>

# 2. Dockerfile指令说明

## 2.1. FROM（指定基础镜像）[构建指令]

该命令用来指定基础镜像，在基础镜像的基础上修改数据从而构建新的镜像。基础镜像可以是本地仓库也可以是远程仓库。

指令有两种格式：

1. FROM `image`   【默认为latest版本】
2. FROM `image`:`tag`     【指定版本】

## 2.2. MAINTAINER（镜像创建者信息）[构建指令]

将镜像制作者（维护者）的信息写入image中，执行docker inspect时会输出该信息。

格式：MAINTAINER `name`

## 2.3. RUN（安装软件用）[构建指令]

RUN可以运行任何被基础镜像支持的命令（即在基础镜像上执行一个进程），可以使用多条RUN指令，指令较长可以使用\来换行。

指令有两种格式：

1. RUN `command` (the command is run in a shell - `/bin/sh -c`)
2. RUN ["executable", "param1", "param2" ... ] (exec form) 
   - 指定使用其他终端实现，使用exec执行。
   - 例子：RUN["/bin/bash","-c","echo hello"]

## 2.4. CMD（设置container启动时执行的操作）[设置指令]

用于容器启动时的指定操作，可以是自定义脚本或命令，只执行一次，多个默认执行最后一个。

指令有三种格式：

1. CMD ["executable","param1","param2"] (like an exec, this is the preferred form) 
   - 运行一个可执行文件并提供参数。
2. CMD command param1 param2 (as a shell) 
   - 直接执行shell命令，默认以/bin/sh -c执行。
3. CMD ["param1","param2"] (as default parameters to ENTRYPOINT) 
   - 和ENTRYPOINT配合使用，只作为完整命令的参数部分。

## 2.5. ENTRYPOINT（设置container启动时执行的操作）[设置指令]

指定容器启动时执行的命令，若多次设置只执行最后一次。

ENTRYPOINT翻译为“进入点”，它的功能可以让容器表现得像一个可执行程序一样。

例子：ENTRYPOINT ["/bin/echo"] ，那么docker build出来的镜像以后的容器功能就像一个/bin/echo程序，docker run -it imageecho “this is a test”，就会输出对应的字符串。这个imageecho镜像对应的容器表现出来的功能就像一个echo程序一样。

指令有两种格式：

1. ENTRYPOINT ["executable", "param1", "param2"] (like an exec, the preferred form)

   - 和CMD配合使用，CMD则作为完整命令的参数部分，ENTRYPOINT以JSON格式指定执行的命令部分。CMD可以为ENTRYPOINT提供可变参数，不需要变动的参数可以写在ENTRYPOINT里面。

   - 例子：

     ENTRYPOINT ["/usr/bin/ls","-a"]

     CMD ["-l"] 

2. ENTRYPOINT command param1 param2 (as a shell)

   - 独自使用，即和CMD类似，如果CMD也是个完整命令[CMD command param1 param2 (as a shell) ]，那么会相互覆盖，只执行最后一个CMD或ENTRYPOINT。
   - 例子：ENTRYPOINT ls -l

## 2.6. USER（设置container容器启动的登录用户）[设置指令]

设置启动容器的用户，默认为root用户。

格式：USER daemon

## 2.7. EXPOSE（指定容器需要映射到宿主机的端口）[设置指令]

该指令会将容器中的端口映射为宿主机中的端口[确保宿主机的端口号没有被使用]。通过宿主机IP和映射后的端口即可访问容器[避免每次运行容器时IP随机生成不固定的问题]。前提是EXPOSE设置映射端口，运行容器时加上-p参数指定EXPOSE设置的端口。EXPOSE可以设置多个端口号，相应地运行容器配套多次使用-p参数。可以通过docker port +容器需要映射的端口号和容器ID来参考宿主机的映射端口。

格式：EXPOSE `port` [`port`...]

## 2.8. ENV（用于设置环境变量）[构建指令]

在image中设置环境变量[以键值对的形式]，设置之后RUN命令可以使用该环境变量，在容器启动后也可以通过docker inspect查看环境变量或者通过 docker run --env key=value设置或修改环境变量。

格式：ENV `key` `value` 

例子：ENV JAVA_HOME /path/to/java/dirent

## 2.9. ADD（从src复制文件到container的dest路径）[构建指令]

复制指定的src到容器中的dest，其中src是相对被构建的源目录的相对路径，可以是文件或目录的路径，也可以是一个远程的文件url。`dest` 是container中的绝对路径。所有拷贝到container中的文件和文件夹权限为0755，uid和gid为0。

- 如果src是一个目录，那么会将该目录下的所有文件添加到container中，不包括目录；
- 如果src文件是可识别的压缩格式，则docker会帮忙解压缩（注意压缩格式）；
- 如果`src`是文件且`dest`中不使用斜杠结束，则会将`dest`视为文件，`src`的内容会写入`dest`；
- 如果`src`是文件且`dest`中使用斜杠结束，则会`src`文件拷贝到`dest`目录下。

格式：ADD `src` `dest` 

## 2.10. COPY（复制文件）

复制本地主机的src为容器中的dest，目标路径不存在时会自动创建。

格式：COPY `src` `dest`

## 2.11. VOLUME（指定挂载点）[设置指令]

创建一个可以从本地主机或其他容器挂载的挂载点，使容器中的一个目录具有持久化存储数据的功能，该目录可以被容器本身使用也可以被其他容器使用。

格式：VOLUME ["`mountpoint`"] 

其他容器使用共享数据卷：docker run -t -i -rm -volumes-from container1 image2 bash [container1为第一个容器的ID，image2为第二个容器运行image的名字。]

## 2.12. WORKDIR（切换目录）[设置指令]

相当于cd命令，可以多次切换目录，为RUN,CMD,ENTRYPOINT配置工作目录。可以使用多个WORKDIR的命令，后续命令如果是相对路径则是在上一级路径的基础上执行[类似cd的功能]。

格式：WORKDIR /path/to/workdir

## 2.13. ONBUILD（在子镜像中执行）

当所创建的镜像作为其他新创建镜像的基础镜像时执行的操作命令，即在创建本镜像时不运行，当作为别人的基础镜像时再在构建时运行（可认为基础镜像为父镜像，而该命令即在它的子镜像构建时运行，相当于在子镜像构建时多加了一些命令）。

格式：ONBUILD `Dockerfile关键字` 

# 3. docker build

```bash
Usage: docker build [OPTIONS] PATH | URL | -

Build a new image from the source code at PATH

-c, --cpu-shares=0                      CPU shares (relative weight)

--cgroup-parent=                       Optional parent cgroup for the container

--cpu-period=0                           Limit the CPU CFS (Completely Fair Scheduler) period

--cpu-quota=0                            Limit the CPU CFS (Completely Fair Scheduler) quota

--cpuset-cpus=                           CPUs in which to allow execution (0-3, 0,1)

--cpuset-mems=                         MEMs in which to allow execution (0-3, 0,1)

--disable-content-trust=true       Skip image verification

-f, --file=                                     Name of the Dockerfile (Default is 'PATH/Dockerfile')

--force-rm=false                          Always remove intermediate containers

--help=false                                 Print usage

-m, --memory=                          Memory limit

--memory-swap=                       Total memory (memory + swap), '-1' to disable swap

--no-cache=false                        Do not use cache when building the image

--pull=false                                 Always attempt to pull a newer version of the image

-q, --quiet=false                         Suppress the verbose output generated by the containers

--rm=true                                  Remove intermediate containers after a successful build

-t, --tag=                                   Repository name (and optionally a tag) for the image

--ulimit=[] Ulimit options
```
