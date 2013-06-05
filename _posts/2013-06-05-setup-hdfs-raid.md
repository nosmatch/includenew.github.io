---
layout: post
title: "HDFS Raid 使用手册"
description: ""
category: HDFS
tags: [HDFS]
---
{% include JB/setup %}

之前介绍过[Facebook的HDFS Raid实现](http://jiangbo.me/blog/2012/12/21/hdfs-raid/)，有部分读者咨询具体如何配置，下面使用Facebook的代码进行简单配置。
### 一、安装
阿里的云梯Raid实现与Facebook的略有不同，由于云梯的代码目前尚未开源，因此，安装配置过程使用的时GitHub上Facebook的开源代码。

1. clone facebook的代码
	
		git clone https://github.com/facebook/hadoop-20.git

2. 编译打包 

		ant tar
	编译过程可能比较慢，也会遇到findbugs，forrest配置等问题，如果你只是想体验下facebook的版本，而不是在上面做了二次开发，可以把这些target从build.xml中取消掉。
3. 部署hadoop包，至少1NN+3DN，下文中NN主机名用hnn表示，dn分别为hdn1,hdn2,hdn3(搭建基础hadoop环境的步骤就不赘述了，google一大把)
4. 启动集群，测试搭建的环境是否正常
5. 除了1NN+3DN外，还要额外部署一个RaidNode（hrn），RaidNode的作用详见上篇[设计说明](http://jiangbo.me/blog/2012/12/21/hdfs-raid/)，raid的包已经位于hadoop-0.20.tar.gz中的contrib目录下，直接解压部署即可


### 二、配置
#### 2.1 NameNode配置
为支持Raid的使用，NameNode上需要新增一些配置

| 配置名 | 配置值 | 说明 |
|----------------------|
| dfs.block.replicator.classname | org.apache.hadoop.hdfs.server.namenode.BlockPlacementPolicyRaid | 针对Raid化的数据进行block的placement策略 |
| raid.codecs.json | 见下例 | 以JSON格式描述的纠错码信息: id, parity_dir, tmp_parity_dir, tmp_har_dir, stripe_length, parity_length, priority, erasure_code, description见下。其中，tmp_parity_dir默认为"/tmp$parity_dir", tmp_har_dir默认为"/tmp$parity_dir"。注意，路径必须是全路径，末尾不能有分隔符。另外，json-2.1.jar需要放在所有node的lib目录下。需要保证所设路径是用户可写的权限。 |
	
	<property>
	  <name>dfs.block.replicator.classname</name>
	  <value>org.apache.hadoop.hdfs.server.namenode.BlockPlacementPolicyRaid</value>
	</property>
	
	<property>
	  <name>raid.codecs.json</name>
	  <value>
	    [   
	      {   
	        "id"            : "xor",
	        "parity_dir"    : "/raid",
	        "stripe_length" : 10, 
	        "parity_length" : 1,
	        "priority"      : 100,
	        "erasure_code"  : "org.apache.hadoop.raid.XorCode",
	        "description"   : "XOR code"
	      },
	      {   
	        "id"            : "rs",
	        "parity_dir"    : "/raidrs",
	        "stripe_length" : 10,
	        "parity_length" : 4,
	        "priority"      : 300,
	        "erasure_code"  : "org.apache.hadoop.raid.ReedSolomonCode",
	        "description"   : "ReedSolomonCode code",
	      },
	    ]   
	  </value>
	  <description>JSon string that contains all Raid codecs</description>
	</property>
	
#### 2.2 RaidNode配置
RaidNode的配置中除了要配置集群信息（namenode，jobtracker的地址等），还要额外配置如下参数

| 配置名	| 配置值	 | 说明 |
|---------------------|
| raid.server.address | hrn:60000 | 指定raidnode的地址和端口，比如：hrn:60000 |
| mapred.raid.http.address | hrn:60001 | raidnode webui的地址，比如：hrn:60001 |
| raid.configmanager.class | org.apache.hadoop.raid.FileConfigManager |ConfigManager类型，用于加载raidPolicy的配置 |
| raid.config.file | raid-policy.xml |采用FileConfigManager时，需要指定的raid.xml文件路径。注意，raid.xml的基本格式与hadoop-site.xml保持一致，configuration内部留空即可 |
| raid.parity.har.threshold.days | 3 | 生成的parity文件多久后开始做har归档， 默认3天 |
| raid.statscollector.update.period | 600000 | raidnoe webui的统计信息刷新周期，默认是10分钟 |
| raid.codecs.json | raid编码的json配置 | 同NN上的配置 |

如下是一份最简的raidnode配置

	<?xml version="1.0"?>
	<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
	
	<!-- Put site-specific property overrides in this file. -->
	
	<configuration>
	
	  <property>
	    <name>dfs.data.dir</name>
	    <value>/home/bo.jiangb/cluster-data/dfs/data</value>
	  </property>
	  <property>
	    <name>hadoop.tmp.dir</name>
	    <value>/home/bo.jiangb/cluster-data</value>
	  </property>
	  <property>
	    <name>dfs.name.dir</name>
	    <value>/home/bo.jiangb/cluster-data/dfs/name</value>
	  </property>
	  <property>
	    <name>dfs.replication</name>
	    <value>3</value>
	  </property>
	  <property>
	    <name>dfs.datanode.address</name>
	    <value>0.0.0.0:55510</value>
	  </property>
	  <property>
	    <name>dfs.datanode.ipc.address</name>
	    <value>0.0.0.0:55520</value>
	  </property>
	  <property>
	    <name>dfs.datanode.http.address</name>
	    <value>0.0.0.0:55575</value>
	  </property>
	  <property>
	    <name>dfs.block.replicator.classname</name>
	    <value>org.apache.hadoop.hdfs.server.namenode.BlockPlacementPolicyRaid</value>
	  </property>
	
	  <property>
	    <name>raid.codecs.json</name>
	    <value>
	      [
	      {
	      "id"            : "xor",
	      "parity_dir"    : "/raid",
	      "stripe_length" : 10,
	      "parity_length" : 1,
	      "priority"      : 100,
	      "erasure_code"  : "org.apache.hadoop.raid.XorCode",
	      "description"   : "XOR code"
	      },
	      {
	      "id"            : "rs",
	      "parity_dir"    : "/raidrs",
	      "stripe_length" : 10,
	      "parity_length" : 4,
	      "priority"      : 300,
	      "erasure_code"  : "org.apache.hadoop.raid.ReedSolomonCode",
	      "description"   : "ReedSolomonCode code",
	      },
	      ]
	    </value>
	 </property>
	 <property>
	    <name>mapred.raid.http.address</name>
	    <value>hrn:61000</value>
	  </property>
	  <property>
	    <name>raid.server.address</name>
	    <value>hrn:60011</value>
	  </property>
	  <property>
	    <name>raid.configmanager.class</name>
	    <value>org.apache.hadoop.raid.FileConfigManager</value>
	  </property>
	  <property>
	    <name>raid.config.file</name>
	    <value>/home/bo.jiangb/hadoop-0.20/conf/raid-policy.xml</value>
	  </property>
	</configuration>
	
#### 2.3 Client端配置
Client端提交Raid请求时，除了常规的client配置外，还需要配置RaidNode相应的信息

| 配置名 | 配置值 | 说明 |
| ---------------------|
|  raid.server.address |  hrn:60000 | 指定raidnode的地址和端口  |
| raidshell.raidfile.targetrepl | 2 | Raid后原文件的副本数，缺省设置为2，若设置为1可减少更多空间 |
| raidshell.raidfile.metarepl | 1 | Raid后parity文件的副本数，缺省设置为1 |
| fs.hdfs.impl | org.apache.hadoop.hdfs.DistributedRaidFileSystem | 指定Raid文件系统: org.apache.hadoop.hdfs.DistributedRaidFileSystem，其他对上层是透明的。。如果客户端需要手动进行Raid文件修复等操作，则不配置该选项 |
| hdfs.raid.local.recovery.location | /tmp/raidrecovery | 在Raid文件系统中，如果读到坏块时，临时修复的块存放路径前缀。默认是/tmp/raidrecovery。需要保证所设路径是用户可写的权限。 如果不需要实时修复，可以不配置该选项|
| raid.codecs.json | raid编码的json配置 | 同上述json配置 |

### 三、启动集群
1. raid的jar包位于contrib/raid目录下，默认没有加入到classpath中，需要手添加，修改bin/hadoopo脚本
	
		for f in $HADOOP_HOME/contrib/raid/*.jar; do
			CLASSPATH=${CLASSPATH}:$f
		fi

2. 启动HDFS，再NN上执行

		./bin/start-dfs.sh
		
3. 启动Mapreduce，再Jobtracker上执行

		./bin/start-mapred.sh
		
4. 在raidnode上配置一个raid-policy.xml(位于raid.config.file配置的路径中)，raidPolicy用来控制每个Raid的执行策略，格式如下

	   <configuration>
	    <srcPath prefix="hdfs://dw50:9000/user/bo.jiangb/">
	      <policy name = "bo.jiangb">
	        <property>
	          <name>srcReplication</name>
	          <value>3</value>
	          <description> pick files for RAID only if their replication factor is
	                        greater than or equal to this value.
	          </description>
	        </property>
	        <property>
	          <name>targetReplication</name>
	          <value>2</value>
	          <description> after RAIDing, decrease the replication factor of a file to
	                        this value.
	          </description>
	        </property>
	        <property>
	          <name>metaReplication</name>
	          <value>2</value>
	          <description> the replication factor of the RAID meta file
	          </description>
	        </property>
	        <property>
	          <name>modTimePeriod</name>
	          <value>3600000</value>
	          <description> time (milliseconds) after a file is modified to make it a
	                        candidate for RAIDing
	          </description>
	        </property>
	      </policy>
	  </srcPath>
	</configuration>

	
5. 启动Raid，再RaidNode上执行

		./bin/start-raidnode.sh

### 四、使用Raid
#### 4.1 Raid Shell
RaidShell时提供给用户的一个终端接口，用法如下：

	[bo.jiangb@dw61.kgb.sqa.cm4 hadoop-0.20]$ ./bin/hadoop raidshell
	Usage: java RaidShell
	           [-showConfig ]
	           [-help [cmd]]
	           [-recover srcPath1 corruptOffset]
	           [-recoverBlocks path1 path2...]
	           [-raidFile <path-to-file> <path-to-raidDir> <XOR|RS>
	           [-fsck [path [-threads numthreads] [-count]]]
	           [-usefulHar <XOR|RS> [path-to-raid-har]]
	           [-checkFile path]
	           -purgeParity path <XOR|RS>
	           [-checkParity path]
	
	Generic options supported are
	-conf <configuration file>     specify an application configuration file
	-D <property=value>            use value for given property
	-fs <local|namenode:port>      specify a namenode
	-jt <local|jobtracker:port>    specify a job tracker
	-files <comma separated list of files>    specify comma separated files to be copied to the map reduce cluster
	-libjars <comma separated list of jars>    specify comma separated jar files to include in the classpath.
	-archives <comma separated list of archives>    specify comma separated archives to be unarchived on the compute machines.
	
	The general command line syntax is
	bin/hadoop command [genericOptions] [commandOptions]

##### raid文件

	./bin/hadoop raidshell -raidFile <path-to-file> <path-to-raidDir> <XOR|RS>

用例1：	

	./bin/hadoop raidshell -raidFile /user/bo.jiangb/bigfile2 /back/raidrs XOR

将/user/bo.jiangb/bigfile2文件采用XOR编码进行raid，parity文件生成再/back/raidrs目录中

用例2：

	./bin/hadoop raidshell -raidFile /user/bo.jiangb /back/raidrs RS
	
将/user/bo.jiangb目录采用RS编码进行raid，parity文件生成再/back/raidrs目录中

##### 检查文件状态
	./bin/hadoop raidshell -fsck [path [-threads numthreads] [-count]]
用例：

	[bo.jiangb@drn hadoop-0.20]$ ./bin/hadoop raidshell -fsck /
	Running RAID FSCK with 16 threads on /
	Querying NameNode for list of corrupt files under /
	Processing 0 possibly corrupt files using 16 threads

##### 恢复损坏的文件
	./bin/hadoop raidshell -recover srcPath1 corruptOffset
用例：
	
	[bo.jiangb@drn hadoop-0.20]$ ./bin/hadoop raidshell -recover /user/bo.jiangb/bigfile2 0
	13/06/05 17:38:12 INFO hadoop.RaidShell: RaidShell connecting to dw61/10.232.98.61:60011
	13/06/05 17:38:13 INFO hadoop.RaidShell: RaidShell recoverFile for /user/bo.jiangb/bigfile2 corruptOffset 0
	13/06/05 17:38:39 INFO hadoop.RaidShell: Raidshell created recovery file /tmp/recovered.1370425093021
	
从偏移量0开始恢复文件/user/bo.jiangb/bigfile2，生成的恢复文件位于/tmp/recovered.1370425093021， 在检查无误后可以将该文件恢复到源目录。


### 附：一些可选的配置

|配置项|默认值|描述| 生效位置|
|------------------------|
|raid.config.reload|true|是否对raid-policy配置进行动态加载|RaidNode|
|raid.config.reload.interval|10x1000|raid-policy动态加载的周期，单位是毫秒|RaidNode|
|raid.policy.rescan.interval|3600x1000|扫描policy中配置的path路径中文件是否完成raid的时间间隔，单位是毫秒|RaidNode|
|raid.har.partfile.size|4x1024x1024x1024|har parity文件的大小，单位是字节|RaidNode|
|raid.distraid.max.jobs|10|同时进行的raid job数的上限|RaidNode|
|raid.distraid.max.files|10000|同时进行raid的文件数上限|RaidNode|
|hdfs.raid.local.recovery.location|/tmp/raidrecovery|DRFS恢复文件时存放恢复文件的目录|Client，RaidNode|
|raid.parity.har.threshold.days|3|parity文件过期的时间标准，过期的parity文件会被har，单位是天|RaidNode|
|raidshell.raidfile.targetrepl|2|raid后源文件的副本数| Client, RaidNode|
|raidshell.raidfile.metarepl|1|raid后parity文件的副本数|Client, RaidNode|
|raidshell.raidfile.codec|rs|raid使用的编码方式，可选值为rs或xor|Client， RaidNode|
|raidshell.raidfile.delay.day|0|raid之行的delay时间|Client，RaidNode|
|fs.raid.recoveryfs.uselocal|false|是否使用LocalFileSystem作为recoveryFS|Client|
|raid.blockfix.filespertask|10L|blockfixer修复文件是，每个task最多能处理的文件数上限|RaidNode|
|raid.blockfix.maxpendingjobs|100|blockfixer修复文件时，同时运行job数的上限|RaidNode|
|raid.encoder.bufsize|1024x1024|encoder编码时从stripe获取byte时的buff大小|RaidNode，Client|
|raid.decoder.bufsize|1024x1024|decoder解码时从stripe和parityfile获取byte时的buff大小|RaidNode，Client|