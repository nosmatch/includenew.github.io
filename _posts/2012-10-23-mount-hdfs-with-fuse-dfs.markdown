---
layout: post
title: "使用FUSE-DFS mount HDFS"
date: 2012-10-23 10:24
comments: true
categories: HDFS
---

##介绍
Hadooop源码中自带了contrib/fuse-dfs模块，用于实现通过libhdfs和fuse将HDFS mount到*inux的本地。

##编译
###环境
1. Linux: 2.6.18-164.el5 x86_64
2. JDK: 1.6.0_23 64bit
3. Hadoop: 0.19.1 下面假设源码目录为$HADOOP_SRC_HOME
4. Ant: 1.8.4
5. GCC: 4.1.2(系统默认)

### 编译libhdfs
#### 修改configure执行权限
	
	$chmod +x $HADOOP_SRC_HOME/src/c++/pipes/configure
	$chmod +x $HADOOP_SRC_HOME/src/c++/utils/configure
#### 修改Makefile，调整编译模式
64位机中，需要修改libhdfs的Makefile，将GCC编译的输出模式由32(-m32)位改为64(-m64)位

	CC = gcc
	LD = gcc
	CFLAGS =  -g -Wall -O2 -fPIC
	LDFLAGS = -L$(JAVA_HOME)/jre/lib/$(OS_ARCH)/server -ljvm -shared -m64(这里) -Wl,-x
	PLATFORM = $(shell echo $$OS_NAME | tr [A-Z] [a-z])
	CPPFLAGS = -m64(还有这里) -I$(JAVA_HOME)/include -I$(JAVA_HOME)/include/$(PLATFORM)
	
####编译
在$HADOOP_HOME目录下执行

	$ ant compile -Dcompile.c++=true -Dlibhdfs=true
编译结果将生成libhdfs库，位于$HADOOP_SRC_HOME/build/libhdfs目录下

### 编译fuse-dfs
####安装fuse库
fuse-dfs依赖fuse库，可通过

	sudo lsmod|grep fuse

检查是否已经安装，如没有，可通过：

	yum -y install fuse fuse-devel fuse-libs

安装相关依赖库。

####设置编译库路径

设置编译库路径，将libhdfs的库加入到编译路径中
	export LD_LIBRARY_PATH=/usr/lib:/usr/local/lib:$HADOOP_SRC_HOME/build/c++/Linux-amd64-64/lib:$JAVA_HOME/jre/lib/amd64/server

####编译
编译contrib/fuse-dfs模块：

	ant compile-contrib -Dlibhdfs=1 -Dfusedfs=1
	
编译完成将会生成$HADOOP_HOME/build/contrib/fuse-dfs/目录，内有：

	fuse-dfs]$ ls
	fuse_dfs  fuse_dfs_wrapper.sh  test

其中fuse\_dfs是可执行程序，fuse\_dfs\_wrapper.sh是包含一些环境变量设置的脚本，不过其中大部分需要修改:(

#### 修改fuse\_dfs\_warpper.sh

	#Hadoop安装目录
	export HADOOP_HOME=/home/bo.jiangb/yunti-trunk/build/hadoop-0.19.1-dc
	#将fuse_dfs加入到PATH
	export PATH=$HADOOP_HOME/contrib/fuse_dfs:$PATH
	#将hadoop的jar加入到CLASSPATH
	for f in ls $HADOOP_HOME/lib/*.jar $HADOOP_HOME/*.jar ; do
	export  CLASSPATH=$CLASSPATH:$f
	done
	#设置机器模式
	export OS_ARCH=amd64
	#设置JAVA_HOME
	export  JAVA_HOME=/home/admin/tools/jdk1.6
	#将libhdfs加入到链接库路径中
	export LD_LIBRARY_PATH=$JAVA_HOME/jre/lib/$OS_ARCH/server:/home/bo.jiangb/yunti-trunk/build/libhdfs:/usr/local/lib
	./fuse_dfs $@

##使用
###mount
1. 新建一个空目录

	$mkdir /tmp/dfs
	
2. 	挂载dfs
	$./fuse_dfs_wrapper.sh dfs://master_node(namenode地址):port /tmp/dfs -d
-d表示debug模式，如果正常，可以将-d参数去掉。

###unmount
卸载可通过：
	fusermount -u /tmp/dfs