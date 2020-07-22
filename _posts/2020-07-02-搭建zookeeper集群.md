---
layout: post
title: 搭建zookeeper集群
categories: [zookeeper]
description: 搭建zookeeper集群
keywords: zookeeper,SpringCloudAlibaba
---

Zookeeper是一个分布式数据一致性的解决方案，分布式应用可以基于它实现诸如数据发布/订阅，负载均衡，命名服务，分布式协调/通知，集群管理，Master选举， 分布式锁和分布式队列等功能。Zookeeper致力于提供一个高性能、高可用、且具有严格的顺序访问控制能力的分布式协调系统

##### zookeeper集群安装
###### 通过docker-compose的方式安装
```
version: '3.1'

services:
  zoo1:
    image: zookeeper
    restart: always
    privileged: true
    hostname: zoo1
    ports:
      - 2181:2181
    volumes: # 挂载数据
      - /Users/wutao/Documents/data/www/java/zookeeper/node01/data:/data
      - /Users/wutao/Documents/data/www/java/zookeeper/node01/datalog:/datalog
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=0.0.0.0:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181

  zoo2:
    image: zookeeper
    restart: always
    privileged: true
    hostname: zoo2
    ports:
      - 2182:2181
    volumes: # 挂载数据
      - /Users/wutao/Documents/data/www/java/zookeeper/node02/data:/data
      - /Users/wutao/Documents/data/www/java/zookeeper/node02/datalog:/datalog
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=0.0.0.0:2888:3888;2181 server.3=zoo3:2888:3888;2181

  zoo3:
    image: zookeeper
    restart: always
    privileged: true
    hostname: zoo3
    ports:
      - 2183:2181
    volumes: # 挂载数据
      #- /usr/local/zookeeper-cluster/node6/data:/data
      #- /usr/local/zookeeper-cluster/node6/datalog:/datalog
      - /Users/wutao/Documents/data/www/java/zookeeper/node03/data:/data
      - /Users/wutao/Documents/data/www/java/zookeeper/node03/datalog:/datalog
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=0.0.0.0:2888:3888;2181

networks: # 自定义网络
  default:
    external:
      name: zoonet
```

###### 集群启动
```
docker-compose up -d
```

##### 常规配置文件说明
```
# zookeeper时间配置中的基本单位 (毫秒)
tickTime=2000
# 允许follower初始化连接到leader最大时长，它表示tickTime时间倍数 即:initLimit*tickTime
initLimit=10
# 允许follower与leader数据同步最大时长,它表示tickTime时间倍数 
syncLimit=5
#zookeper 数据存储目录
dataDir=/tmp/zookeeper
#对客户端提供的端口号
clientPort=2181
#单个客户端与zookeeper最大并发连接数
maxClientCnxns=60
# 保存的数据快照数量，之外的将会被清除
autopurge.snapRetainCount=3
#自动触发清除任务时间间隔，小时为单位。默认为0，表示不自动清除。
autopurge.purgeInterval=1
```

##### 节点
zookeeper 中节点叫znode存储结构上跟文件系统类似，以树级结构进行存储。不同之外在于znode没有目录的概念，不能执行类似cd之类的命令。znode结构包含如下：
- path:唯一路径
- childNode：子节点
- stat:状态属性
- type:节点类型

###### 节点类型
类型| 描述
---|---
PERSISTENT	| 持久节点
PERSISTENT_SEQUENTIAL |	持久序号节点
EPHEMERAL |	临时节点(不可在拥有子节点)
EPHEMERAL_SEQUENTIAL|临时序号节点(不可在拥有子节点
