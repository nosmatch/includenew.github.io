---
layout: post
title: "本地编译Hadoop小记"
date: 2012-09-24 15:28
comments: true
categories: Hadoop
---

##Git源码
	git clone git://git.apache.org/hadoop-common.git
视网速不通，略慢
## 编译
	cd hadoop-common
	mvn install -DskipTests
	
抛异常：

	[ERROR] Failed to execute goal org.apache.maven.plugins:maven-antrun-plugin:1.6:run (compile-proto) on project hadoop-common: An Ant BuildException has occured: exec returned: 127 -> [Help 1]
	org.apache.maven.lifecycle.LifecycleExecutionException: Failed to execute goal org.apache.maven.plugins:maven-antrun-plugin:1.6:run (compile-proto) on project hadoop-common: An Ant BuildException has occured: exec returned: 127
		at org.apache.maven.lifecycle.internal.MojoExecutor.execute(MojoExecutor.java:217)
		at... 
		Caused by: /Users/Shared/Workspace/hadoop/hadoop-common/hadoop-common-project/hadoop-common/target/antrun/build-main.xml:23: exec returned: 127
		at org.apache.tools.ant.taskdefs.ExecTask.runExecute(ExecTask.java:650)
		at org.apache.tools.ant.taskdefs.ExecTask.runExec(ExecTask.java:676)
		at org.apache.tools.ant.taskdefs.ExecTask.execute(ExecTask.java:502)
		at org.apache.tools.ant.UnknownElement.execute(UnknownElement.java:291)
		at sun.reflect.GeneratedMethodAccessor16.invoke(Unknown Source)
		... 21 more

原因是缺少protocol buffer， 找不到protoc命令。
###安装protocol buffer
	wget https://protobuf.googlecode.com/files/protobuf-2.4.1.tar.bz2
	tar -xvf protobuf-2.4.1.tar.bz2
	cd protobuf-2.4.1
	./configure && make
	make install
	
###导入Eclipse

	mvn eclipse:eclipse -DdownloadSources=true -DdownloadJavadocs=true