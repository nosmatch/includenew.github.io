---
layout: post
title: "HDFS FastCopy"
description: "HDFS FastCopy for Federation"
category: HDFS
tags: [HDFS]
---
{% include JB/setup %}
[[PageOutline]]

### 一、背景 
HDFS原始实现两个集群间数据复制使用的是DistCp的方式，主要通过提交Job的并行读取文件数据，写入到目标文件，这种方式需要使所有block数据经由源集群DN-》client节点-》目标集群DN，效率较低，Facebook实现了一个FastCopy来提升集群间copy的效率，目前已在社区提交了一个issue[HDFS-2139](https://issues.apache.org/jira/browse/HDFS-2139 HDFS-2139)，尚未提交实现。本文主要参考Facebook的FastCopy实现。

### 二、设计
FastCopy的主要流程如下：

1. 查询源NS中文件的meta信息，获取源文件所有的block信息
2. 对于每个Block，获取其在原集群中的location信息
3. 对于源文件中的每个block，在目标NS中的文件上添加空的block信息
4. 对于所有的源Block，通过DN的copyBlock接口实现local copy
5. 每个目标DN在完成block的copy之后向目标NS的NN中报告接收的Block
6. 等待所有的block都copy完成后推出
基本结构如下：

![](/images/hdfs/FastCopy.png)

#### 2.1 File Meta复制

1. FastCopy首先获取源文件的meta信息(FileStatus)和blocks locations(LocatedBlocks)信息
2. 检查源文件是否处于构建中，LocatedBlocks.isUnderConstruction()，如果是则跳过该文件
3. 在目标NS中创建目标文件， 副本数，permission， blockSize等信息与源文件一致，为避免目标文件已存在，默认使用覆写模式创建

#### 2.2 Block 复制

对于源文件的LocatedBlocks中所有的block信息，进行Block复制：

1. 通过向目标NS中addBlock获取目标NS中block的DN 列表
2. 对源block的DN列表和目标Block的DN列表进行排序对齐，使相同的DN在各自的列表中的位置相同
3. 通过源Datanode的copyBlock()接口实现想目标DN的block数据复制

DataNode.copyBlock()的具体实现分为三种情况：
##### 2.2.1 同一DN实例 
这种情况存在于Federation中同一DN节点服务于两个NS的情况，事实上只需要为源Block的文件创建一个HardLink指向目标block文件
##### 2.2.2 同一节点上不通DN实例
（云梯中暂时不存在这种单节点多DN实例的情况）
这种情况实际上文件位于同一台物理节点上，也可以通过HardLink完成，但由于两个DN实例维护不不同的VolumeMap，因此，需要源DN实例调用目标DN的copyBlockLocal（）接口实现，copyBlockLocal本质上也是使用HardLink来完成copy
##### 2.2.3 不同DN节点
通常源DN和目标DN都是位于不同节点上的，需要通过网络传输block数据，这个传输过程与client向DN写入Block数据基本一致，因此可以直接使用DataTransfer来完成。

#### 2.3 Lease更新
由于FastCopy在复制过程中需要对目标文件进行写入，但不是使用FileSystem的API（HDFS默认的lease机制是位于DFSClient中），因此FastCopy需要自己完成Lease的更新。对于每个copy的文件，FastCopy都需要启用一个LeaseChecker线程定期更新lease，保证数据写入的一致性。

#### 2.4 复制状态监控
批量的文件copy是异步执行的，FastCopy内部通过一个fileStatusMap维护所有需要复制的文件，文件的状态中包括文件名，文件的block数以及已经完成复制的block， 以及一个blocksStatusMap维护每个需要复制的block的状态，block的状态包括block的副本总数，已经写入成功的副本数，以及写入失败的副本数。

### 三、实现
#### 3.1 类图结构

![](/images/hdfs/RFastCopy-class.png)
#### 3.2 实现逻辑

1. FastCopy首先处理命令行参数，提取源文件和目标文件path，为每一个src：dst对构建FastCopyFileRequest
2. 构建FastCopy实例处理FastCopyFileRequest
3. FastCopy内部的调度处理通过ExecutorService维护一个线程池（线程池大小可以通过命令行-t 参数来控制，默认是5），每个线程是由实现了Future接口的FastFileCopy来对每个文件的copy进行处理。
4. FastFileCopy内部实现上述设计文档描述的一个文件元信息copy的流程，并通过BlockCopyRpc异步调用DN的copyBlock接口实现block的复制
5. FastCopy为每个文件的copy维护一个LeaseChecker，更新lease信息。
6. 同时FastCopy通过内部维护fileStatusMap和blocksStatusMap来对copy过程状态进行管理

### 四、其他
Facebook的实现中还涉及了HDFS HardLink的使用， （注意HDFS HardLink和上文所述的Block文件的HardLink是两码事，Block文件的HardLink指的是原始的操作系统中文件系统的硬链接，HDFS中提供了一个HardLink的工具类实现与linux shell中 ln 相同的效果；而HDFS的HardLink指的是HDFS文件系统层面的HardLink，不同的文件元信息指向相同的block信息，具体参见 [HDFS-245](https://issues.apache.org/jira/browse/HDFS-245) )，对于支持HardLink的系统，可以直接使用HDFS的HardLink来构建目标文件，使目标文件连接到源文件，由于云梯暂时不支持HDFS HardLink，所以本文暂不讨论该过程。

### 五、参考资料

1. HDFS FastCopy: [HDFS-2139](https://issues.apache.org/jira/browse/HDFS-2139 HDFS-2139)
2. HDFS Symbol Link: [HDFS-245](https://issues.apache.org/jira/browse/HDFS-245)
3. Facebook hadoop: <https://github.com/facebook/hadoop-20>
