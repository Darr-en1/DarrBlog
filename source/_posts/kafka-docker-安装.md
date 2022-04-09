---
title: kafka docker 安装
copyright: true
permalink: 1
top: 0
date: 2022-04-07 11:01:48
tags:
    - python
    - test
    - 源码
categories: python
password:
---

想在本地搭建kafka集群，需要保证zookeeper的高可用，整个服务搭建过程比较耗时，且卸载也较为麻烦，如何快速搭建整个kafka集群呢，试一试docker吧！<!--more-->



##  Docker-compose

编写docker-compose.yaml

```yaml
version: '3.1'
services:
  zoo1:
    image: zookeeper:3.5.7
    restart: always
    hostname: zoo1
    container_name: zoo1
    ports:
      - "2181:2181"
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888

  zoo2:
    image: zookeeper:3.5.7
    restart: always
    hostname: zoo2
    container_name: zoo2
    ports:
      - "2182:2181"
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888

  zoo3:
    image: zookeeper:3.5.7
    restart: always
    hostname: zoo3
    container_name: zoo3
    ports:
      - "2183:2181"
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888

  kafka1:
    image: wurstmeister/kafka
    restart: always
    hostname: kafka1
    container_name: kafka1
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: kafka1
      KAFKA_LISTENERS: PLAINTEXT://kafka1:9092
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka1:9092
      KAFKA_ADVERTISED_PORT: 9092
      KAFKA_ZOOKEEPER_CONNECT: zoo1:2181,zoo2:2181,zoo3:2181
    depends_on:
      - zoo1
      - zoo2
      - zoo3

  kafka2:
    image: wurstmeister/kafka
    restart: always
    hostname: kafka2
    container_name: kafka2
    ports:
      - "9093:9093"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: kafka2
      KAFKA_LISTENERS: PLAINTEXT://kafka2:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka2:9093
      KAFKA_ADVERTISED_PORT: 9093
      KAFKA_ZOOKEEPER_CONNECT: zoo1:2181,zoo2:2181,zoo3:2181
    depends_on:
      - zoo1
      - zoo2
      - zoo3

  kafka3:
    image: wurstmeister/kafka
    restart: always
    hostname: kafka3
    container_name: kafka3
    ports:
      - "9094:9094"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: kafka3
      KAFKA_LISTENERS: PLAINTEXT://kafka3:9094
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka3:9094
      KAFKA_ADVERTISED_PORT: 9094
      KAFKA_ZOOKEEPER_CONNECT: zoo1:2181,zoo2:2181,zoo3:2181
    depends_on:
      - zoo1
      - zoo2
      - zoo3
```

执行docker-compose命令

![image-20220407112018737](/images/kafka-docker-安装/image-20220407112018737.png)

####  测试

kafka1创建 topic

![image-20220407114624830](/images/kafka-docker-安装/image-20220407114624830.png)

向topic 发送一条消息

![image-20220407114925597](/images/kafka-docker-安装/image-20220407114925597.png)

打开一个新的控制台，进入kafka2， 从起始偏移量开始监听

![image-20220407115122974](/images/kafka-docker-安装/image-20220407115122974.png)

我们发现，消息从kafka1 同步至kafka2了，一个kafka集群已经搭建完成，是不是很简单。
