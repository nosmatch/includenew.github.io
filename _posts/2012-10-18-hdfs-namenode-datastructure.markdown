---
layout: post
title: "HDFS源码学习（1）——NameNode主要数据结构"
date: 2012-10-18 11:04
comments: true
categories: HDFS
---

##FSNameSystem
FSNameSystem是HDFS文件系统实际执行的核心，提供各种增删改查文件操作接口。其内部维护多个数据结构之间的关系：

1. fsname->block列表的映射
2. 所有有效blocks集合
3. block与其所属的datanodes之间的映射（该映射是通过block reports动态构建的，维护在namenode的内存中。每个datanode在启动时向namenode报告其自身node上的block）
4. 每个datanode与其上的blocklist的映射
5. 采用心跳检测根据LRU算法更新的机器（datanode）列表

### FSDirectory
FSDirectory用于维护当前系统中的文件树。

其内部主要组成结构包括一个INodeDirectoryWithQuota作为根目录(rootDir)和一个FSImage来持久化文件树的修改操作。

#### INode
HDFS中文件树用类似VFS中INode的方式构建，整个HDFS中文件被表示为INodeFile，目录被表示为INodeDirectory。INodeDiretoryWithQuota是INodeDirectory的扩展类，即带配额的文件目录

![INode](/images/hdfs/INode.png)


INodeFile表示INode书中的一个文件，扩展自INode，除了名字(name)，父节点(parent)等之外，一个主要元素是blocks，一个BlockInfo数组，表示该文件对应的block信息。



### BlocksMap
BlocksMap用于维护Block -> { INode, datanodes, self ref } 的映射
![BlocksMap](/images/hdfs/BlocksMap.png)
BlocksMap结构比较简单，实际上就是一个Block到BlockInfo的映射。
#### Block
Block是HDFS中的基本读写单元，主要包括：

1. blockId: 一个long类型的块id
2. numBytes: 块大小
3. generationStamp: 块更新的时间戳

#### BlockInfo
BlockInfo扩展自Block，除基本信息外还包括一个inode引用，表示该block所属的文件；以及一个神奇的三元组数组Object[] triplets，用来表示保存该block的datanode信息，假设系统中的备份数量为3。那么这个数组结构如下：

![triplets](/images/hdfs/triplets.png)

1. DN1，DN2，DN3分别表示存有改block的三个datanode的引用(DataNodeDescriptor）
2. DN1-prev-blk表示在DN1上block列表中当前block的前置block引用
3. DN1-next-blk表示在DN1上block列表中当前block的后置block引用

DN2,DN3的prev-blk和next-blk类似。
HDFS采用这种结构存放block->datanode list的信息主要是为了节省内存空间，block->datanodelist之间的映射关系需要占用大量内存，如果同样还要将datanode->blockslist的信息保存在内存中，同样要占用大量内存。采用三元组这种方式能够从其中一个block获得到改block所属的datanode上的所有block列表。

#### FSImage
FSImage用于持久化文件树的变更以及系统启动时加载持久化数据。
HDFS启动时通过FSImage来加载磁盘中原有的文件树，系统Standby之后，通过FSEditlog来保存在文件树上的修改，FSEditLog定期将保存的修改信息刷到FSImage中进行持久化存储。
FSImage中文件元信息的存储结构如下（参见FImage.saveFSImage()方法）

![FSImage](/images/hdfs/FSImage.png)
#####FSImage头部信息
1. layoutVersion(int):image layout版本号，0.19版本的hdfs中为-18
2. namespaceId(int): 命名空间ID，系统初始化时生成，在一个namenode生命周期内保持不变，datanode想namenode注册是返回改id作为registerId，以后每次datanode与namenode通信时都携带该id，不认识的id的请求将被拒绝。
3. numberItemOfTree(long): 系统中的文件总数
4. generationTimeStamp: 生成image的时间戳

#####INode信息
FSImage头之后是numberItemOfTree个INode信息，INode信息分为文件(INodeFile)和文件目录(INodeDirectory)两类，两者大体一致，分为INode头，Blocks区（目录没有blocks）和文件权限。

**INode头**

1. nameLen(short): 文件名长度
2. filename(String): 文件名
3. replication(short): 备份数量
4. modificationTime(long): 最近修改时间
5. accessTime(long): 最近访问时间
6. preferedBlockSize(long): 块大小（目录为0）
7. block num(int): 块数量（目录为-1）

**Blocks区**

1. blockId(long)
2. numBytes(long,block大小)
3. generationTimeStamp(long, 更新时间戳）

**文件权限**

1. username(String): 文件用户名
2. group(String): 所属组
3. fileperm(short): 文件权限

#####underconstructionFile区
layoutverion<-18版本的fsimage还包括正在构建的文件区。与普通Inode信息类似，均有inode头和blocks区以及文件权限，除此之外，underConstructionFile还包括：

**client信息**

1. clientName：client明
2. clientMachine： client机器名

**已分配的datanode信息**

1. ipcport： 服务端口
2. capacity: 容量
3. dfsuse： 已使用的空间
4. remaining： 剩余空间
5. lastupdate： 最新更新时间
6. xceiverCount
7. location： datanode位置
8. hostName：主机名
9. state： admin管理状态


## 其他结构
### CorruptReplicasMap
CorruptReplicasMap通过一个TreeMap维护corrupt状态block的blocks-->datanodedescriptor(s)映射。一个block备份在多个datanode中，当其中的一个或多个datanode上的block损坏时，会将该datanode加到treeMap中该block对应的datanodeDescriptor集合中。FSNameSystem通过该Map来维护所有损坏的block与其对应datanode的关系。

### Map<String, LightWeightHashSet<Block>> recentInvalidateSets
维护最近失效的block集合，map中为storageId->ArrayList<Block>，当某个block的一个datanode上副本失效时会将改block和对应的datanode的storeageId添加到recentInvalidateSet中，当datanode想namenode进行heartbeat时，namenode会检查该datanode中是否有损坏的block，如有，则通知datanode删除改block。

### NavigableMap<String, DatanodeDescriptor>  datanodeMap
datanodeMap用于维护datanode->block的映射


### ArrayList<DatanodeDescriptor> heartbeats
维护多有当前活着的节点

### UnderReplicatedBlocks neededReplications
通过一个优先级队列来维护当前需要备份的block集合，副本数越少的block优先级越高，0为最高级，表示当前只有一个副本。

### PendingReplicationBlocks pendingReplications;
维护当前正在备份的block集合，并且进行备份请求的时间统计，并通过一个后台线程（PendingReplicationMonitor）来周期性（默认为5分钟）的统计超时的备份请求，当发生超时时，会将这个block重新添加到neededReplications列表中。
![PendingReplicationBlocks](/images/hdfs/PendingReplicationBlocks.png)

### LightWeightLinkedSet<Block> overReplicatedBlocks
当前需要检查是否备份过多的block集合

### Map<String, Collection<Block>> excessReplicateMap
维护系统中datanode与其上的超额备份block的集合，这些超额的备份将被删除。