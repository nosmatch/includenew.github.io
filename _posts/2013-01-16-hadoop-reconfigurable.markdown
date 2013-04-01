---
layout: post
title: "HADOOP动态加载配置"
date: 2013-01-16 15:01
comments: true
categories: Hadoop HDFS
---
##简介
Hadoop集群运行过程中有时会需要对配置进行修改，而通常需要重启才能生效，如该HDFS Namenode的一个配置，需要重启NN才能生效。而对于规模较大的系统，重启的成本较高。[HADOOP-7001](https://issues.apache.org/jira/browse/HADOOP-7001)引入了一个reconfigurable机制。简述如下：

![Reconfig_class](/images/hdfs/reconfig_class.png)

HADOOP-7001提供了一个Reconfigure接口用于定义可动态配置的的行为。并提供了一个ReconfigurableBase抽象的基类实现。该基类有两个抽象方法，所有想要实现动态配置的节点，都需要实现这两个方法：

	//某个可以动态配置的属性变化时需要进行的处理
	protected abstract void reconfigurePropertyImpl(String property, String newVal) 
    	throws ReconfigurationException;
    
    //定义该节点上所有可以进行动态配置的属性集合
    public abstract Collection<String> getReconfigurableProperties();
    
为了便于完成配置项变更，HADOOP-7001还提供了一个ReconfigurationServlet工具便于从web端变更配置。使用时只需要将该servelt加入到相应节点的httpserver中，并在context中加入conf.servlet.reconfigurable.$P的参数，值为对应的Reconfigurable实现（一般为节点自身实现），其中$P表示的是ReconfigurationServlet在httpServer中对应的path。

##NameNode中的使用

[HDFS-1477](https://issues.apache.org/jira/browse/HDFS-1477)中提供了NameNode Reconfigurable的实现。概要分析如下：
### 1. 扩展ReconfigurableBase

首先需要扩展Reconfigurable来使NameNode支持动态配置

	public class NameNode extends ReconfigurableBase implements ClientProtocol, 
		DatanodeProtocol,NamenodeProtocol, FSConstants {
		  
		  //。。。。
		  //此处省略N多无关代码
		  //		
		  
		  //实现当发生配置变更时Namenode的具体处理行为
		  @Override
		  public void reconfigurePropertyImpl(String property, String newVal) 
		    throws ReconfigurationException {
		    // just pass everything to the namesystem
		    if (namesystem.isPropertyReconfigurable(property)) {
		     namesystem.reconfigureProperty(property, newVal);
		    } else if ("fs.trash.interval".equals(property)) {
		      try {
		        if (newVal == null) {
		          // set to default
		          trash.setDeleteInterval(60L * Trash.MSECS_PER_MINUTE);
		        } else {
		          trash.setDeleteInterval((long)(
		              Float.valueOf(newVal) * Trash.MSECS_PER_MINUTE));
		        }
		        LOG.info("RECONFIGURE* changed trash deletion interval to " +
		            newVal);
		      } catch (NumberFormatException e) {
		        throw new ReconfigurationException(property, newVal,
		            getConf().get(property));
		      }
		    } else {
		      throw new ReconfigurationException(property, newVal,
		                                         getConf().get(property));
		    }
		  }
		  
		  //设置NameNode上允许动态配置的属性值
		  @Override
		  public List<String> getReconfigurableProperties() {
		    // only allow reconfiguration of namesystem's reconfigurable properties
		    List<String> properties = namesystem.getReconfigurableProperties();
		    properties.add("fs.trash.interval");
		    return properties;
		  }

###2. 在httpserver中配置ReconfigurationServlet
为了便于配置，需要在httpserver中添加ReconfigurationServlet，具体代码如下

	private void startHttpServer(Configuration conf) throws IOException {
	    
	    //省略无关代码
	   
	    //设置context属性
	    httpServer.setAttribute(ReconfigurationServlet.
	                            CONF_SERVLET_RECONFIGURABLE_PREFIX +
	                            CONF_SERVLET_PATH, NameNode.this);
	    //添加servelt， path为nnconfchange
	    httpServer.addServlet("nnconfchange", CONF_SERVLET_PATH,
	                          ReconfigurationServlet.class);
	    this.httpServer.start();
	
	    // The web-server port can be ephemeral... ensure we have the correct info
	    infoPort = this.httpServer.getPort();
	    this.httpAddress = new InetSocketAddress(infoHost, infoPort);
	    conf.set("dfs.http.address", infoHost + ":" + infoPort);
	    LOG.info("Web-server up at: " + infoHost + ":" + infoPort);
	  }
 
###3.具体使用
登录到NN，修改配置文件，通过web访问 http://hdfsnn:port/nnconfchange来查看配置项，点击apply可以是新的配置项生效，如果配置项变更出错会返回500.    


