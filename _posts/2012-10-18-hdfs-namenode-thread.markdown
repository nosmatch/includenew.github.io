---
layout: post
title: "HDFS源码学习（3）——NameNode中的线程"
date: 2012-10-18 21:37
comments: true
categories: HDFS
---

NameNode存在三种运行模式：

1. Normal： NameNode正常服务的状态
2. Safe mode：NameNode重启时进入Safe mode，该模式下整个系统是只读的，以便于NameNode手机DataNode信息
3. Backup mode：备份NameNode处于Backup mode，被动的接收主NameNode的检查点信息


在NameNode中存在如下几种线程：

1. DataNode 健康检查管理线程
2. 副本管理线程
3. 租约管理（lease Management）
4. IPC Handler 线程