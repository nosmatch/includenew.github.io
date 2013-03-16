---
layout: post
title: "如何用ruby获取本机IP&发送給Gtalk"
date: 2012-01-04 17:35
comments: true
categories: 
---
### 问题
有一台server用的是动态ip，每次重启后ip地址就变了，因此写了一个脚本，每次在server重启后，自动将ip发送到我的gtalk上。

### 解决
#### 如何获取本机IP
用ruby获取本机的动态ip，网上很多教程都用

	require 'socket'
	
	IPSocket.getaddress(Socket.gethostname)
	puts TCPSocket.gethostbyname(Socket.gethostname)
	
这个在mac下是正常的，但是在linux下就只能拿到127.0.0.1
在StackOverflow上有另一种解决方案
	
	require 'socket'
	
	def local_ip
	  orig, Socket.do_not_reverse_lookup = Socket.do_not_reverse_lookup, true  # turn off reverse DNS resolution temporarily
	
	  UDPSocket.open do |s|
	    s.connect '64.233.187.99', 1
	    s.addr.last
	  end
	ensure
	  Socket.do_not_reverse_lookup = rig
	end
	
这段主要是通过开启一个UDP链接来获取本地对外ip，因为UDP是无状态，所以不会实际建立网络链接，但是会获取本机对外ip。

#### 如何发送消息
获取ip后需要通过gtalk发送，gtalk使用的是xmpp协议，ruby中协议有多种开源实现，比较简单通用的是xmpp4r，详细教程请看这里
首先需要安装xmpp4r-simple

	gem install xmpp4r-simple
	
注意，貌似这个gem不支持1.9.*，所以使用之前先将ruby切换到1.8.7版本
然后编写代码，主要两步：

* 建立链接
* 发送消息

代码如下：

	 require 'rubygems'
	 require 'xmpp4r-simple'  
	
	 username = gmailusername
	 password = gmailpassword
	 to_username = destination_gmailusername  
	
	 puts "Connecting to jabber server.."
	 jabber = Jabber::Simple.new(username+'@gmail.com',password)
	 puts "Connected."
	 jabber.deliver(to_username+"@gmail.com", "Hello..!")
	 sleep(1)

注意，最后那个sleep不能少，尽管我还不知道为啥:(

如此以来整个的获取ip，发送給gtalk的脚本为：
	
	require 'rubygems'
	require 'socket'
	require 'xmpp4r-simple'
	
	def local_ip
	  orig, Socket.do_not_reverse_lookup = Socket.do_not_reverse_lookup, true  # turn off reverse DNS resolution temporarily
	
	  UDPSocket.open do |s|
	    s.connect '64.233.187.99', 1
	    s.addr.last
	  end
	ensure
	  Socket.do_not_reverse_lookup = orig
	end
	
	jabber = Jabber::Simple.new('gmailuseanme@gmail.com','password')
	jabber.deliver('destusenam@gmail.com', local_ip)
	sleep 1

#### 设置自动运行
linux设置自动运行是老生常谈了，在/etc/rc.local加上一句运行脚本的命令即可：

	ruby /home/jiangbo/ruby/getIP.rb

### 结尾
这样每次机器重启时，就能够通过gtalk获取到ip了，不过还遗留一个问题，就是在机器重启时，必须保证接收消息的gtalk在线，因为这种方式的消息gtalk不会自动重法，目前不知怎么解决。
