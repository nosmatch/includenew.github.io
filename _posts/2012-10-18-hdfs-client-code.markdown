---
layout: post
title: "HDFS源码学习（14）——Client代码结构"
date: 2012-10-18 21:59
comments: true
categories: HDFS
---

Client核心代码有DistributedFileSystem和DFSClient。

![Client](/images/hdfs/Client.png)

DistributedFileSystem扩展子FileSystem，在为客户端提供一个文件系统接口实现。其内部使用DFSClient完成各类文件操作。

DFSClient使用ClientProtocol与NameNode通信，完成文件元信息操作。并通过Socket连接完成与DataNode间的block读写操作。

DFSClient代码结构如下：

![DFSClient](/images/hdfs/DFSClient.png)

1. LeaseChecker主要用于lease检查和续约。
2. DFSOutputStream用于提供带buffer的字节流写入功能。client在写入数据时先将数据缓存在本地。并将数据切分成多个packet（默认每个packet为64K）。每个packet又被拆分成多个chunk（默认512Byte），每个chunk都有一个checksum。client写满一个packet后会将该packet加入到一个dataqueue中。由DataStreamer线程负责将每个packet发送给datanode pipeline。发送完一个pakcet，streamer会将其从dataqueue移至ackqueue中。ResponseProcessor负责接收datanode发回的ack信息，每成功接收一个packet的ack信息，ResponseProcessor会将ackqueue中该packet删除。
3. DFSInputStream用于提供字节流的读取，其内部封装了与NN和DN的交互
4. DataStreamer: 负责向datanode pipeline发送packet。其本身是一个Daemon线程，从namenode获取blockId和block存放位置，将packet发送给pipeline中的datanode，每个packet都有一个seqId，每个packet发送完时都会收到datanode的ack信息。当收到所有packet的ack信息后（表示该block已发送完），streamer关闭该block。
5. ResponseProcessor:用于接收datanode返回ack信息，并将响应ackqueue中的packet删除

##创建文件

1. client向NameNode发起创建文件请求
2. NameNode.create（）处理创建文件请求，检查是否有重名，当前是否处于Safe-mode，是否有权限创建文件， 校验通过后创建一个INode记录。
3. NameNode将创建文件的事件记录到EditLog中
4. INode被创建后，NameNode发放给Client一个lease，Client可以使用这个lease通过ClientProtocol访问，进行只读操作。（写操作需要等文件close）


##写入流程
client写入流程如下图所示：

![ClientWrite](/images/hdfs/ClientWrite.png)

1. Client向NameNode发起创建文件的RPC请求
2. NameNode检查文件是否已经存在，是否有权创建等，成功则创建一个文件记录，并发放给Client一个lease
3. Client获得lease之后开始进行数据写入，写入的数据首先被缓存本地，并被拆分为多个packet，放置到dataqueue队列中
4. DataStreamer线程负责检查dataqueue队列，发现有数据时且没有可用block时，向NameNode发送addBlock()请求，申请一个分配一个block空间。NameNode返回给DataStreamer一个blockId和用于存放block的datanode list
5. DataStreamer将每个packet数据发送给datanode pipeline，并将该packet移至ackqueue
6. datanode pipeline中第一个datanode收到packet之后存储到本地block中并穿行备份至后续datanode中
7. pipeline中datanode存储好packet之后会逆序返回ack信息，并最终返回给client.
8. Client端ResponseProcessor捕获到每个packet的ack信息时会将响应ackqueue中的packet删除
9. 当所有数据都写入完成后，client会向NameNode发起一个complete RPC请求，告知文件最新的时间戳和已经发送给datanode的block长度。NameNode检查所有block的副本信息，只有所有block的副本数均满足最低要求时，complete会返回成功。
10. 最后，NameNode将收回client持有的lease。

NameNode处理addBlock()请求的流程大致如下：

1. 校验client是否有该文件的lease
2. 清理上一次写入记录，包括：a.提交上一次写入，b.更新lease有效期；c.将完成的写入记录到EditLog中
3. 清理完毕之后，使用BlockManager分配指定副本数个block及其对应的datanode信息，返回给client

DataNode处理block写入的流程大致如下：

1. 将block复制到本地磁盘
2. 发送block received消息给NameNode告知写入了一个新的block
3. 将block数据发送给datanode pipeline中下一个datanode，进行备份
4. 返回一个ack消息给前一个调用者

后续的datanode收到上一个datanode的备份block请求是做类似的操作。





##读取流程
读取流程相对简单写，如下所示 ：

![ClientWrite](/images/hdfs/ClientRead.png)

1. client向NameNode发起RPC请求，获取文件的blockLocation信息
2. NameNode返回一定长度（10*defaultBlockSize）block的datanode位置信息
3. Client根据返回的blockLocation信息选取距自己最近（同一节点<通一机架<同一机房）的datanode读取数据，读完一个block会对该block进行checksum校验。如果校验正确则关闭与该datanode连接，去读下一个block；如果校验失败，则通知NameNode该block在当前datanode上的副本损坏了，并继续从datanode列表中获取一个datanode，重新读取该block。
4. 当本次获取的blockLocation中的block全部读完，且该文件还有block时，重复1，2，3过程，直至所有blcok全部读完。

##关闭文件（complete）

1. 当Client完成文件写入之后，会调用complete()通知NameNode文件写入完成了，该请求会提交文件写入的最后一个block信息并且告知NameNode写入的block总数以及最新时间戳。
2. NameNode收到请求后会检查是否所有的block的事物都已经提交了，并且每个block的副本数都达到了最小值。如果是则返回true，否则返回false。
3. Client收到返回值后如果失败则重试几次。