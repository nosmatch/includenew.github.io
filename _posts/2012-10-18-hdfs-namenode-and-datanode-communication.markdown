---
layout: post
title: "HDFS源码学习（10）——NameNode与DataNode间的通信"
date: 2012-10-18 21:54
comments: true
categories: HDFS
---

NameNode和DataNode间的通信分为四种场景：

1. 初始时DataNode注册：
2. 周期性心跳检测：
3. 周期性blockreport：
4. 完成一个副本的写入：

##一、初始时DataNode注册
DataNode在启动时会向NameNode注册，注册时需要提交的信息有DatanodeRegistration表示。结构如下：

![DatanodeRegistration](/images/hdfs/DatanodeRegistration.png)

主要包括：

1. name：机器名（主机名+服务端口号）
2. infoPort: 状态信息服务端口好
3. ipcPort： 提供ipc服务的端口号

此外，该类中的storageID是该datanode在集群中的唯一id，在注册时有NameNode分配


注册的主要流程如下：
![DataNodeRegister](/images/hdfs/register.png)


##二、心跳检测（heartbeat）
DataNode通过周期性调用namenode.sendHeartbeat()来完成心跳检测.主要流程如下：

![sendHeartbeat](/images/hdfs/sendHeartbeat.png)

##三、blockReport
DataNode周期性向NameNode发送blockReport，告知自己最新的block信息：
![blockReport](/images/hdfs/blockReport.png)
##四、完成副本写入