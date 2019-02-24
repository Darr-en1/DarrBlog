---
title: 详解redis install
copyright: true
permalink: 1
top: 0
date: 2018-12-29 16:14:47
tags:    
    - redis
    - linux
categories: redis
password:
---
redis作为使用内存存储的NoSQL,数据存储相较于memcached有更加丰富的数据结构支持。还附有发布订阅，主从复制等功能。让我们来尝试怎么安装它吧。<!--more-->


## linux install redis

### 安装步骤
本人使用的是ubuntu_16_server,首先需要在[redis官网](https://redis.io/download)下载压缩包,当然网络环境允许的情况下可以通过linux命令直接下载，以下为整个俺喜欢过程：
```shell
# 下载压缩包
$ wget http://download.redis.io/releases/redis-5.0.3.tar.gz     
# 解压缩
$ tar xzf redis-5.0.3.tar.gz 
# 进入redis目录       
$ cd redis-5.0.3                    
# 编译Redis
$ make                             
```
### 问题
大多数时候make都会出错，需要下载gcc工具
```shell
$ sudo apt-get clean
$ sudo apt-get update 
$ sudo apt-get build-dep gcc
```
### 远程访问
很多时候程序和redis不在一台服务器上，就需要访问远程Redis服务，Redis默认也只允许本机访，因此需要需改Redis的配置文件  `/etc/redis/redis.conf` 

**解决办法**：
- 注释掉bind 127.0.0.1可以使所有的ip访问redis
- 修改 protected-mode no

指定配置文件然后重启Redis服务：
`$ sudo redis-server /etc/redis/redis.conf`

远程连接：
`$ redis-cli -h {redis_host} -p {redis_port}`

## windows install redis

### windows安装redis弊端
windows安装本人是不建议的，首先官方不支持，redis在windows上的版本会比较低，其次，redis在将数据库持久化到硬盘的时候，需要用到fork系统调用，而Windows并不支持这个调用。在缺少fork调用情况，redis在持久化操作期间就只能够阻塞所有客户端，直到持久化操作完毕为止。虽然也有提供相应解决方案，但依旧不如linux上的支持完整。

### 安装步骤
安装redis可以在[windows_redis](https://github.com/MicrosoftArchive/redis/releases)下载,最好下载`.msi`后缀的文件，在安装过程中会自动写入到windows的服务中，安装完成后，进入redis目录运行`redis-server.exe`,可能会运行不起来，进行如下操作即可：
```shell
redis-cli.exe
shutdown
exit
redis-server.exe
```

