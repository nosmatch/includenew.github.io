---
layout: post
title: "HDFS-2246:使用BlockReaderLocal优化本地block读取"
date: 2012-12-10 16:09
comments: true
categories: HDFS
---

## 一、背景
HDFS中对Block的读取使用DataXceiver通过Socket发送Packet数据进行，client端通过一个BlockReader来接收Socket中的Block数据。详见[HDFS数据读取流程]()。大致流程如下：

1. client端调用FileSystem.open()获取一个DFSInputStream
2. client端调用DFSInputStream.read(byte[] buffer, int off, int len)读取数据
3. client端通过DFSInputStream.blockSeekTo()向NameNode发起请求，定位到需要读取的block所在的Datanode，构建一个BlockReader用于读去对应DataNode上的block数据，返回该DataNodeInfo
4. BlockReader主要负责同DataNode间建立一个Socket链接用于读取数据，BlockReader首先向DataNode发送请求头（包括操作类型，blockId，block时间戳，起始偏移量，读取数据的长度，client名）
6. DataNode接收到DataXceiverServer接收到client端请求后，构建一个DataXceiver处理该请求。
7. DataXceiver首先解析请求头，获取请求操作类型，当发现是READ_BLOCK操作后，调用相应的readBlock()方法处理
8. readBlock方法解析请求中需要的blockId，构建一个BlockSender用于读取磁盘上的block文件数据并发送给client
9. BlockSender读取磁盘上的block文件，将数据按照chunk通过socket发送给client的blockReader，同时，BlockSender在发送chunk后需要从meta文件中读取该chunk的checksum数据，同样发送给client，用于该chunk的checksum校验。
10. client端通过BlockReader.readChunks()接收BlockSender发送的chunk数据，并进行checksum校验，校验成功后向DataNode发送checksumOk。
11. 循环6-10，直至当前block的数据全部被读取完成。
12. 循环执行3-11, 直至需要读取的文件数据都被读取完。

这个过程是在集群环境想，client读取datanode上数据的一个正常流程，但事实上当client和datanode位于同一个物理节点上时（如Hadoop集群中，task运行在datanode上），这个过程显的有些多余，client可以直接通过本地文件系统api读取文件，而不需要走繁杂的socket流程。

## 二、设计实现

HDFS-2246中提供了一个BlockReaderLocal的实现，当client发现从NameNode返回的Block所属的datanode和client位于同一节点上时，构建一个BlockReaderLocal用于读取本地文件。
上述3-10的流程将简化为：

3. client端向NameNode发起请求获取block所属的datanode信息后，判断该datanode是否和client位于同一节点，是且开启了本地读取功能，则构建一个BlockReaderLocal读取本地文件，否则构建一个BlockReader按照原流程进行。
4. BlockReaderLocal通过DataNode.getBlockLocaPathInfo()从DataNode获取block的本地文件路径信息。
5. BlockReaderLocal构建InputStream读取block文件和meta文件信息
6. 对于需要checksum的场景（默认），通过blockReaderLocal.readChunks()按chunk读取本地文件，同时读取meta文件中该chunk的checksum数据，进行校验
7. 对于跳过checksum的场景，直接通过InputStream.read()读取block数据。

### 扩展DataNode协议接口

client端需要能够从DataNode获取block文件的本地文件路径信息。因此扩展ClientDataNodeProtocol，增加一个
	BlockLocalPathInfo getBlockLocalPathInfo(Block block) throws IOException;
	
接口用于获取block的本地路径信息

### 本地文件读取

BlockReaderLocal共过BufferedInputStream直接读取本地文件，注意此处HDFS-2246的patch中使用的是FileInputStream，实际测试过程中发现，FileInputStream对本地文件的读取性能较差， 替换为使用BufferedInputStream

### checksum较验

为了完成checksum校验，BlockReaderLocal同时需要读取block的meta文件，每当block文件读取一个chunk时需要从meta文件读取一个checksum数据，进行checksum校验，通过校验后进行下一个chunk的读取和校验。由于BlockReaderLocal读取的是本地文件，避免的网络传输对数据的影响，因此可以配置跳过checksum检查，以提高读取性能。默认是需要做checksum的。

### 本机判断

当前patch中的实现主要用IP来判断是否block所在的datanode与client是否位于同一节点上。

## 测试

通过TestDFSIO工具测试一个单节点的集群，2个文件，每个文件1000M
./bin/hadoop jar hadoop-0.19.1-dc-test.jar TestDFSIO -read -nrFiles 2 -fileSize 1000
测试结果对比:
socket读取：

	---
	12/12/07 13:52:57 INFO mapred.FileInputFormat: ----- TestDFSIO ----- : read
	12/12/07 13:52:57 INFO mapred.FileInputFormat:            Date & time: Fri Dec 07 13:52:57 CST 2012
	12/12/07 13:52:57 INFO mapred.FileInputFormat:        Number of files: 2
	12/12/07 13:52:57 INFO mapred.FileInputFormat: Total MBytes processed: 2000
	12/12/07 13:52:57 INFO mapred.FileInputFormat:      Throughput mb/sec: 283.5270768358378
	12/12/07 13:52:57 INFO mapred.FileInputFormat: Average IO rate mb/sec: 283.5281982421875
	12/12/07 13:52:57 INFO mapred.FileInputFormat:  IO rate std deviation: 0.5685961122141402

本地读取

	---
	12/12/07 13:48:59 INFO mapred.FileInputFormat: ----- TestDFSIO ----- : read
	12/12/07 13:48:59 INFO mapred.FileInputFormat:            Date & time: Fri Dec 07 13:48:59 CST 2012
	12/12/07 13:48:59 INFO mapred.FileInputFormat:        Number of files: 2
	12/12/07 13:48:59 INFO mapred.FileInputFormat: Total MBytes processed: 2000
	12/12/07 13:48:59 INFO mapred.FileInputFormat:      Throughput mb/sec: 369.61744594344856
	12/12/07 13:48:59 INFO mapred.FileInputFormat: Average IO rate mb/sec: 369.6180725097656
	12/12/07 13:48:59 INFO mapred.FileInputFormat:  IO rate std deviation: 0.4800772

另外通过Patch中提供的TestShortCircuitLocalRead工具，测试结果如下：

本地读取并进行checksum校验

	true no 1 32000000
	---
	Iteration 20 took 115453
	Iteration 20 took 115803
	Iteration 20 took 115748

socket读取并进行checksum校验（默认）

	no no 1 32000000
	---
	Iteration 20 took 128820
	Iteration 20 took 135305
	Iteration 20 took 129145
