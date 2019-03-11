---
layout: w
title: docker基础
date: 2019-03-05 18:49:24
tags: 
    - docker
    - linux
categories: docker
---
docker的出现为环境配置，项目部署提供了空前便捷。相较于虚拟机的臃肿，运行缓慢，docker既轻量有快捷，成本开销都更小<!--more-->

### docker install 
下载适配当前liunx版本的docker

`sudo wget -q0- https://get.docker.com | sh`

docker需要使用root用户运行，可以通过将用户添加到"docker" group,注销并重新登录使其生效

`sudo usermod -aG docker [USER]`

### 命令

#### image 操作命令

查找image(支持通配符)

`docker search [IMAGE]`

获取 image

`docker pull [IMAGE]`

查看 image(支持通配符)

`docker images`

删除 image

`docker rmi [IMAGE]`

查看image 历史（创建过程）

`docker history ubuntu:latest`

查看详细信息

`docker inspect ubuntu:latest`

添加镜像标签（指向同一个镜像文件，只是别名不同）

`docker tag ubuntu:latest myubuntu:latest`

创建 image

`docker build [IMAGE]`

#### container 操作命令

启动容器(通过本地镜像创建容器并启动，本地没有远端拉取)

`docker run [IMAGE]`

终端启动容器

`docker run -it -p 80:80 [IMAGE] \bin\bash`

守护进程启动容器

`docker run --name webserver -d -p 80:80 [IMAGE]`
- -d 守护进程后台启动
- --name 容器名
- -p 端口映射

容器关闭时清理容器的匿名data volumes

docker run --rm -it [IMAGE]

通过交互终端打开容器（一般用于打开守护进程在后端启动的容器）

`docker exec -it webserver bash`

查看容器修改（会有大量无关内容）

`docker diff webserver`

清理所有处于终止状态的容器

`docker container prune`

查看当前运行的 container(options  -a 所有容)
docker ps  

关闭容器(start 启动 restart 重启)

`docker stop [IMAGE]`

删除容器(需要关闭)

`docker rm [IMAGE]`

#### 关于镜像删除

删除行为分为两类，一类是 Untagged，另一类是 Deleted。当一个镜像有多个标签时,删除只是删除该镜像的的某一个标签(Untagged),当只有一个标签就会彻底删除(Deleted)。

当有该镜像创建的容器存在时，镜像无法删除，需要删除容器，`docker rmi -f [IMAGE]` 强制删除镜像 （不推荐）

#### 关于--rm操作

在Docker容器退出时，默认容器内部的文件系统仍然被保留，以方便调试并保留用户数据。

但是，对于foreground容器，由于其只是在开发调试过程中短期运行，其用户数据并无保留的必要，因而可以在容器启动时设置--rm选项，这样在容器退出时就能够自动清理容器内部的文件系统。示例如下：

docker run --rm [IMAGE]

显然，--rm选项不能与-d同时使用，即只能自动清理foreground容器，不能自动清理detached容器

注意，--rm选项也会清理容器的匿名data volumes，容器也会一并删除。


### 创建镜像

1. 从已经创建的容器中更新镜像，commit提交这个镜像
2. 基于本地模板导入
3. 使用 Dockerfile指令来创建一个新的镜像


#### 基于已有容器创建
docker commit [选项] <容器ID或容器名> [<仓库名>[:<标签>]]

`docker commit --author 'darren' --message '修改首页' dad9a2486a8b nginx:darr`

黑盒操作，虽然可以diff命令查看，但维护麻烦，会使得镜像臃肿。

#### 导入创建
镜像导出

`docker save [IMAGE ID] > save.tar`

镜像的导入

` docker load < save.tar`

容器导出(导出容器的快照)

` docker export [IMAGE ID] > export.tar`

容器导入

`cat export.tar | docker import - [IMAGE NAME:TAG]`

镜像导入和容器导入的区别：
1. 容器导入 是将当前容器 变成一个新的镜像,容器快照文件将丢弃所有的历史记录和元数据信息（即仅保存容器当时的快照状态）
2. 镜像导入 是复制的过程,体积更大

save 和 export区别：
1. save 保存镜像所有的信息-包含历史
2. export 只导出当前的信息

#### Dockerfile

在 Dockerfile 文件所在目录执行：`docker build -t [IMAGE TAG] .`

**镜像构建上下文（Context）**

当构建的时候，用户会指定构建镜像上下文的路径，docker build 命令得知这个路径后，会将路径下的所有内容打包，然后上传给 Docker 引擎。这样 Docker引擎收到这个上下文包后，展开就会获得构建镜像所需的一切文件。所需文件需存放在dockerfile同级目录。


指令：
FROM
RUN
COPY
ADD
CMD
ENTRYPOINT
ENV
ARG
VOLUME
EXPOSE
WORKERDIR
USER
HEALTHCHECK 
ONBUILD

ADD COPY 区别： 
- ADD可以将远端的文件复制进容器
- ADD可以在复制压缩包进容器时会自动解压

CMD ENTRYPOINT HEALTHCHECK 只能有一个，输入多个最后一个生效

**CMD ENTRYPOINT 区别**

run container 后面带的参数 会将 CMD 指令覆盖，但是不能覆盖ENTRYPOINT指令，会变成ENTRYPOINT的参数添加。
`docker run -it --entrypoint=[指令] [IMAGE]` 就可以覆盖ENTRYPOINT指令



对于容器而言，其启动程序就是容器应用进程，容器就是为了主进程而存在的，主进程退出，容器就失去了存在的意义，从而退出，其它辅助进程不是它需要关心的东西。守护进程一般在系统启动时开始运行，除非强行终止，否则直到系统关机都保持运行,因此会以守护进程的方式在后端启动服务。


Docker 容器通过虚拟网桥进行连接
