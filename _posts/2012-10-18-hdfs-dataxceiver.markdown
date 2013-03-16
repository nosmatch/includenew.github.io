---
layout: post
title: "HDFS源码学习（15）——DataXceiverServer"
date: 2012-10-18 22:00
comments: true
categories: HDFS
---

HDFS中有两种类型的通信机制，一种是进行消息传递的Hadoop IPC机制，一种是用于处理数据传输的DataXceiver机制。前者包括client<->namenode之间的通信，以及datanode<->namenode间通信，后者包括client<->datanode, datanode<->datanode间的数据传输。

##DataXceiverServer
DataNode在启动时会通过DataXceiverServer开启一个Socket端口，负责block数据的读写。DataXceiverServer本身作为一个守护线程，监听dfs.datanode.address配置的数据读写服务端口。当有请求来时，新建一个DataXceiver线程处理请求。

##DataXceiver
DataXceiver线程用于处理一个读/写数据流请求，其run方法入下主要是根据请求中不同的请求类型，调用响应的处理方法。


请求操作类型定义在DataTransferProtocol中，主要有：

1. OP_WRITE_BLOCK： 写入Block数据，对应writeBlock()方法
2. OP_READ_BLOCK： 读取Block数据，对应readBlock()方法
3. OP_READ_METADATA： 读取Block元数据，对应readMetadata()方法
4. OP_REPLACE_BLOCK： 替换Block，将block发送到目标datanode上，用于IO负载均衡；对应replaceBlock()方法。
5. OP_COPY_BLOCK：复制Block，将block发送到proxy source上，用于IO负载均衡；对应copyBlock()方法。
6. OP_BLOCK_CHECKSUM：获取Block的checksum；对应getBlockChecksum()方法。

请处理返回的状态也定义在该类中：

1. OP_STATUS_SUCCESS： 成功
2. OP_STATUS_ERROR： 请求出错
3. OP_STATUS_ERROR_CHECKSUM： checksum校验出错
4. OP_STATUS_ERROR_INVALID： 读取无效block
5. OP_STATUS_ERROR_EXISTS：block不存在
6. OP_STATUS_CHECKSUM_OK： checksum校验正常


###1.读取block——readBlock()

OP_READ_BLOCK的请求数据格式如下：

![READ_BLOCK](/images/hdfs/ReadBlock.png)

返回数据格式如下：

![READ_Response](/images/hdfs/ReadResponse.png)

readBlock()主要从disk读取block数据，构建一个DataOutputStream数据流，并新建一个BlockSender将这个数据流发送出去（datanode或者client）。

BlockSender.sendBlock()发送的Block的流程大体如下：

1. 读取block的meta信息，获得checksum并发送
2. 发送数据读取的偏移量
3. 将block数据切分为packet，发送给client
4. 所有packet发送完之后，关闭checksum文件和block文件



###2.写入block——writeBlock()
OP_WRITE_BLOCK的请求数据格式如下：

![WRITE_BLOCK](/images/hdfs/WriteRequest.png)


writeBlock()解析请求信息，构建一个BlockReceiver处理数据接收和写入，在client（或上一datanode节点）-当前datanode节点-下一datanode节点之间建立一个如下连接。

![WRITE_BLOCK](/images/hdfs/WriteBlock.png)

1. BlockReceiver从上按packet一节点读取数据，写入到本地disk
2. 如有下一备份节点，将该packet转发给下一节点
3. 将该packet加入到ackqueue队列中等待ack消息
4. 当下一节点完成该packet写入后会返回该packet对应的ack信息
5. PakcetResponder接收到ack信息后，将ackqueue中该packet删除，并向前置节点发送ack信息


