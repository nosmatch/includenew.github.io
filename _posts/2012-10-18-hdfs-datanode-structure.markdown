---
layout: post
title: "HDFS源码学习（12）——DataNode主要数据结构"
date: 2012-10-18 21:56
comments: true
categories: HDFS
---

HDFS中DataNode主要负责维护block->stream bytes的映射关系，即实际block数据的存储。
一个datanode的磁盘上存储目录下实际的文件部署结构如下：

	data/
	├── blocksBeingWritten
	├── current
	│   ├── VERSION
	│   ├── blk_-1148021215131449924
	│   ├── blk_-1148021215131449924_1001.meta
	│   ├── blk_-8598609183581346893
	│   ├── blk_-8598609183581346893_1002.meta
	│   ├── blk_6693595845022390257
	│   ├── blk_6693595845022390257_1003.meta
	│   └── dncp_block_verification.log.curr
	├── detach
	├── storage
	└── tmp
	
data目录的路径是hdfs-site.xml中配置的dfs.data.dir的路径，表示每个datanode上数据存储的目录

1) blocksBeingWritten：当前正在写入的block，写完之后会将block移至current目录

2) current：当前已经写入的block文件目录

2.1) VERSION为存储的VERSION文件，包括namespaceId，存储Id，存储版本，存储类型，创建时间戳等信息

2.2）blk-*:文件为实际的block数据文件

2.3）blk-*_xxx.meta: block的元信息文件

2.4）dncp_*.log.curr: 当前copy文件

3) detach：copy-on-write使用的目录

4) tmp： 临时目录，DataNode启动时会检查 tmp的数据并删除。

## Storage相关
Storage用于描述存储的类型，状态，目录等信息。
其主要结构如下：
![Storage](/images/hdfs/Storage.png)
### StorageInfo
StorageInfo表示一个存储的通用信息，包括：

1. layoutVersion： 存储文件中的版本号
2. namespaceId： 存储所属的命名空间ID
3. ctime： 该存储创建的时间戳


### Storage
存储信息的抽象类，管理一个server（NameNode或DataNode）上的存储目录。

Storage有两个关键属性：

1. storageType: 表示该存储所属的节点类型（NameNode或是DataNode）
2. storageDirs: 该存储上存储目录的列表(ArrayList<StorageDirectory>),StorageDirectory表示一个存储目录。

####StorageDirectory

表示一个存储目录，有三个属性：

1. root：根目录
2. lock：当前目录的文件锁
3. dirType：目录类型

####StorageSate
表示存储的状态：

1. NON_EXISTENT: 目录不存在
2. NOT_FORMATTED: 目录未格式化
3. COMPLETE_UPGRADE: 升级完成
4. RECOVER_UPGRADE: 撤销升级
5. COMPLETE_FINALIZE: 提交完成
6. COMPLETE_ROLLBACK: 回滚完成
7. RECOVER_ROLLBACK: 撤销回滚
8. COMPLETE_CHECKPOINT: checkpoint完成
9. RECOVER_CHECKPOINT: 撤销checkpoint
10. NORMAL: 正常


### DataStorage
DataStorage是DataNode上使用的存储类，指定了datanode上各类存储文件的前缀：

1. subdir：子目录前缀
2. blk_：块文件前缀
3. dncp_：拷贝文件前缀

##DatanodeBlockInfo
DataNode使用DatanodeBlockInfo管理block和其元数据之间的映射关系，结构如下：

![DatanodeBlockInfo](/images/hdfs/DatanodeBlockInfo.png)

1. volmun：block所属的卷
2. file：block文件
3. detached：是否完成copy-on-write

## FSDataSet相关
DataNode通过FSDataSet来完成数据的存储。FSDataset类结构如下：
![FSDataset](/images/hdfs/FSDataset.png)

### FSVolume
FSVolumne用于进行block文件所属的卷管理，统计存储目录额使用情况，其中：

1. currentDir： 当前数据目录, 对应data/current目录
2. dataDir： 数据目录
3. tmpDir： 临时目录, 对应data/tmp目录
4. dtacheDir: 用于实现写时复制的文件，对应data/detach目录
5. usage: 目录使用的空间
6. dfsusage: dfs使用的空间
7. reseved: 空余空间
8. blocksBeingWritten: 正在写入的block，对应data/blocksBeingWritten目录

### FSVolumeSet

FSVolumeSet是FSVolume的集合，提供了所有容量，剩余空间等方法。其中getNextVolume中提供了round-robin策略选取下一个volume，从而实现简单的IO负载均衡，提高IO处理能力。

### FSDataSet
FSDataSet是在FSVolumeSet之上进行封装实现FSDatasetInterface借口，向外提供块查询和操作方法。

其中有几个主要属性:

1. volumes: 卷集合（FSVolumeSet）
2. ongoingCreates: 当前活动的文件
3. maxBlocksPerDir: 每个目录下最多能存放发block数，可通过dfs.datanode.numblocks配置
4. volumeMap：块与块文件的映射信息(HashMap<Block, DatanodeBlockInfo>)，当前集合中所有的块信息均维护在该map中


### FSDir
用于构建block块在datanode磁盘上的层次结构，默认情况下每个目录下最多64个子目录，最多能存储64个块。目录初始化时会递归扫描目录下的所有子目录和文件，构建一个树形结构。

addBlock时，首先尝试在当前目录新加块，如果当前目录没有空闲空间，则尝试在子目录中添加，如果没有子目录，则新建一个子目录。

### BlockAndFile
Block与其文件名的封装
### ActiveFile
表示一个当前活动中的文件

