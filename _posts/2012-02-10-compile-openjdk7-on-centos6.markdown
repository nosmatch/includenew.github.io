---
layout: post
title: "CentOS6编译OpenJDK7"
date: 2012-02-10 14:39
comments: true
categories: JVM
---

## 一.环境准备
### 1.jdk
在编译JDK7之前，需要有个JDK6版本，这个貌似有个鸡生蛋，还是蛋生鸡的问题，不过，这个确实需要:)
### 2.alsa包
linux版本的jdk编译需要ALSA（Advanced Linux Sound Architecture）包，大部分linux发行版都没有预装，CentOS可以通过如下命令检查：
     rpm -qa |grep alsa
alsa-lib和alsa-lib-devel均需要。CentOS缺少alsa-lib-devel，通过如下命令安装：
     yum install alsa-lib-devel
### 3.cups-devel
	yum install cups-devel
### 4.libXi-devel
	yum install libXi-devel

### 5.freetype2.3
 
	wget http://download.savannah.gnu.org/releases/freetype/freetype-2.3.12.tar.gz
	tar -xvf freetype-2.3.12.tar.gz
	cd freetype-2.3.12
	./configure && make && make install
### 6. ant
	wget http://mirror.bit.edu.cn/apache//ant/binaries/apache-ant-1.8.2-bin.zip
	unzip apache-ant-1.8.2-bin.zip
### 7.g++
	yum install gcc gcc-c++

## 二、设置环境变量
jdk编译过程中有一些环境变量需要设置，详细的请参考README-builds.html，下面写的只是一些必须设置的环境变量：
	export ALT_BOOTDIR=/usr/opt/jdk # 预装的jdk7目录
	export ANT_HOME=ant安装目录
	export ALT_FREETYPE_HEADERS_PATH=/usr/local/include/freetype2 #freetype2头文件安装目录
	export ALT_FREETYPE_LIB_PATH=/usr/local/lib #freetype2 lib目录
## 三、编译
### 1.健全检查
可以通过如下命令检查环境配置是否准备好：

	make sanity ARCH_DATA_MODEL=64
如果最终输出：

	Sanity check passed.
则表示环境检查通过，否则需要根据提示信息排查问题。
### 2.执行编译
通过如下命令开始编译：
	make ARCH_DATA_MODEL=64
### 3.问题排查：
编译过程中出现一些问题：

### 1)缺少jaxp和jaxws
错误信息
	ERROR: Cannot find source for project jaxp
原因是现在jaxp源码分支和jdk源码分支分开了，但是jaxws是jdk中的一部分，所以完全编译需要jaxp源码，针对该问题的描述可以查看README-build.html中TroubleShooting部分。
解决方式有两种：
一种是先下载好源码包，以drops的方式安装，具体参考README-build.html
另外一种是使用在线安装，在编译时加入允许下载源码的配置:

	make ARCH_DATA_MODEL=64 ALLOW_DOWNLOADS=true

### 2)缺少X＊库
编译过程中多次出现如下缺少X*, awt之类的错误，基本上都是因为缺乏图形相关的库

	../../../src/solaris/native/sun/awt/img_util_md.h:32: ??:expected specifier-qualifier-list before 'XID'
	make[5]: *** [/home/jiangbo/Workspace/jdk/openjdk/build/linux-amd64/tmp/sun/sun.awt/awt/obj64/BufImgSurfaceData.o] Error 1
	make[5]: *** Waiting for unfinished jobs....
	make[5]: Leaving directory `/home/jiangbo/Workspace/jdk/openjdk/jdk/make/sun/awt'
	make[4]: *** [library_parallel_compile] Error 2
	make[4]: Leaving directory `/home/jiangbo/Workspace/jdk/openjdk/jdk/make/sun/awt'
	make[3]: *** [all] Error 1
	make[3]: Leaving directory `/home/jiangbo/Workspace/jdk/openjdk/jdk/make/sun'
	make[2]: *** [all] Error 1
	make[2]: Leaving directory `/home/jiangbo/Workspace/jdk/openjdk/jdk/make'
	make[1]: *** [jdk-build] Error 2
	make[1]: Leaving directory `/home/jiangbo/Workspace/jdk/openjdk'
	make: *** [build_product_image] Error 2

解决方式时安装X相关的库

	yum install libX*
这个有些暴力，不过比较有效:)

## 四、测试编译结果
漫长的编译之后直至出现如下类似内容时，表示编译完成了：

	-- Build times ----------
	Target all_product_build
	Start 2012-02-09 10:38:39
	End   2012-02-09 11:14:37
	00:01:41 corba
	00:06:19 hotspot
	00:15:49 jaxp
	00:01:30 jaxws
	00:10:03 jdk
	00:00:36 langtools
	00:35:58 TOTAL

编译完成后，编译结果维语build/linux-amd64目录下，可以写个简单的Java程序测试编译结果

Test.java
	public class Test{
	        public static void main(String[] args){
	                System.out.println("Hello");
	        }
	}

编译
	[root@localhost openjdk]# ./build/linux-amd64/bin/java Test.java
执行
	[root@localhost openjdk]# ./build/linux-amd64/bin/java Test
	Hello

