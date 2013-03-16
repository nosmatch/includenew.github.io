---
layout: post
title: "HDFS源码学习（7）——Block管理"
date: 2012-10-18 21:46
comments: true
categories: HDFS
---

HDFS通过一个BlockManager管理集群中所有的block信息


##主要数据结构
###Block
Block是HDFS读写的基本单元，集群中每个block通过一个long id来唯一标示。

###BlockInfo
维护一个block的元信息，主要通过
###BlockMap
通过一个GSet<Block, BlockInfo>维护一个block与其元数据信息的映射关系，元信息包括其所属的BlockCollection和存储该block的datanode节点，每个BlockMap有个初始容量capacity

###BlockCollection

##Block和副本管理
###Block和副本状态
Block有如下状态：

1. committed：所有的副本已经被创建且更新至最新
2. Under construction: 需要创建一个或多个副本
3. To be deleted: 所有副本需要被删除。发生在文件被删除或者block被重写
4. Over-replicated: 过多的副本存在。此时副本中的一个需要设置为无效并删除。

副本有如下状态：

1. Current: 正常状态，该副本正确反应block内容
2. Conrrupt: 某个副本损坏。副本损坏是由client报告给namenode的。client通过checksum检查副本是否损坏，如果损坏了，通过BlockManager.invalidateBlock()处理
3. On a faild DataNode: DataNode Heartbeat发现有DataNode失效时，即将在改datanode上创建的副本将被删除
4. Out of Date: 当Datanode失效，且副本所属的block发生更新后，Datanode恢复正常。过期的block将通过blockreport报告给namenode，并将其删除
5. Under construction: 副本尚未被写入并在Datanode上被验证。在NameNode看来，只有当收到blockReport并且报告中timestamp正确时，猜人物副本写入正常。

##Block分配

##Block查询