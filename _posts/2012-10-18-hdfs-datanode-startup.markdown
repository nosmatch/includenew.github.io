---
layout: post
title: "HDFS源码学习（13）——DataNode启动过程"
date: 2012-10-18 21:57
comments: true
categories: HDFS
---

##main()

	  public static void main(String args[]) {
	    try {
	      StringUtils.startupShutdownMessage(DataNode.class, args, LOG);
	      DataNode datanode = createDataNode(args, null);
	      if (datanode != null)
	        datanode.join();
	    } catch (Throwable e) {
	      LOG.error(StringUtils.stringifyException(e));
	      System.exit(-1);
	    }
	  }
	  
##createDataNode()

	  public static DataNode createDataNode(String args[],
	                                 Configuration conf) throws IOException {
		// 初始化datanode
	    DataNode dn = instantiateDataNode(args, conf);
	    // 启动datanode后台线程
	    runDatanodeDaemon(dn);
	    return dn;
	  }
	  
### 1.instantiateDataNode（）

	  public static DataNode instantiateDataNode(String args[],
	                                      Configuration conf) throws IOException {
		// 处理配置
	    if (conf == null)
	      conf = new Configuration();
	    if (!parseArguments(args, conf)) {
	      printUsage();
	      return null;
	    }
	    if (conf.get("dfs.network.script") != null) {
	      LOG.error("This configuration for rack identification is not supported" +
	          " anymore. RackID resolution is handled by the NameNode.");
	      System.exit(-1);
	    }
	    // 获取data目录配置
	    String[] dataDirs = conf.getStrings("dfs.data.dir");
	    dnThreadName = "DataNode: [" +
	                        StringUtils.arrayToString(dataDirs) + "]";
		//创建datanode实例
	    return makeInstance(dataDirs, conf);
	  }
#### 1.1. makeInfstance()
该方法主要用于检查给定的data目录中至少有一个可以创建，并实例化DataNode

	  public static DataNode makeInstance(String[] dataDirs, Configuration conf)
	    throws IOException {
	    ArrayList<File> dirs = new ArrayList<File>();
	    for (int i = 0; i < dataDirs.length; i++) {
	      File data = new File(dataDirs[i]);
	      try {
	        DiskChecker.checkDir(data);
	        dirs.add(data);
	      } catch(DiskErrorException e) {
	        LOG.warn("Invalid directory in dfs.data.dir: " + e.getMessage());
	      }
	    }
	    if (dirs.size() > 0) 
	      return new DataNode(conf, dirs);
	    LOG.error("All directories in dfs.data.dir are invalid.");
	    return null;
	  }
#### 1.2 new DataNode()
	
	  DataNode(Configuration conf, 
	           AbstractList<File> dataDirs) throws IOException {
	    // 设置配置信息
	    super(conf);
	    datanodeObject = this;
	    supportAppends = conf.getBoolean("dfs.support.append", false);
	    this.conf = conf;
	    try {
	      // 启动DataNode
	      startDataNode(conf, dataDirs);
	    } catch (IOException ie) {
	      shutdown();
	      throw ie;
	    }
	  }
##### 1.2.1 startDataNode()
代码较长，仅列出主要步骤：

1. 设置配置信息
2. 向NameNode发起RPC请求，获取版本和StorageID信息
3. 获取启动配置
4. 初始化存储信息，构建FSDataSet
5. 获取可用的端口号
6. 调整注册信息中的机器名，加上端口号
7. 初始化DataXceiverServer
8. 设置blockReport和heartbeat各自的时间间隔
9. 初始化blockScanner
10. 初始胡并启动servlet info server，提供内容查询的http服务
11. 初始化ipc server，该ipc server主要用于完成DataNode间的block recover。

  	  
### runDatanodeDaemon()

	  public static void runDatanodeDaemon(DataNode dn) throws IOException {
	    if (dn != null) {
	      //register datanode
	      dn.register();
	      dn.dataNodeThread = new Thread(dn, dnThreadName);
	      dn.dataNodeThread.setDaemon(true); // needed for JUnit testing
	      dn.dataNodeThread.start();
	    }
	  }
	  
#### 2.1 向NameNode注册 —— dn.register();

	  private void register() throws IOException {
	    if (dnRegistration.getStorageID().equals("")) {
	      setNewStorageID(dnRegistration);
	    }
	    while(shouldRun) {
	      try {
	        // reset name to machineName. Mainly for web interface.
	        dnRegistration.name = machineName + ":" + dnRegistration.getPort();
	        // 通过NameProtocal向NameNode注册
	        dnRegistration = namenode.register(dnRegistration);
	        break;
	      } catch(SocketTimeoutException e) {  // namenode is busy
	        LOG.info("Problem connecting to server: " + getNameNodeAddr());
	        try {
	          Thread.sleep(1000);
	        } catch (InterruptedException ie) {}
	      }
	    }
	    assert ("".equals(storage.getStorageID()) 
	            && !"".equals(dnRegistration.getStorageID()))
	            || storage.getStorageID().equals(dnRegistration.getStorageID()) :
	            "New storageID can be assigned only if data-node is not formatted";
	    if (storage.getStorageID().equals("")) {
	      storage.setStorageID(dnRegistration.getStorageID());
	      storage.writeAll();
	      LOG.info("New storage id " + dnRegistration.getStorageID()
	          + " is assigned to data-node " + dnRegistration.getName());
	    }
	    if(! storage.getStorageID().equals(dnRegistration.getStorageID())) {
	      throw new IOException("Inconsistent storage IDs. Name-node returned "
	          + dnRegistration.getStorageID() 
	          + ". Expecting " + storage.getStorageID());
	    }
	    
	    if (supportAppends) {
	      Block[] bbwReport = data.getBlocksBeingWrittenReport();
	      long[] blocksBeingWritten = BlockListAsLongs.convertToArrayLongs(bbwReport);
	      //如果支持append，则报告正在写入的block信息
	      namenode.blocksBeingWrittenReport(dnRegistration, blocksBeingWritten);
	    }
	    // 调整下一次的BR时间，使其在下次heartbeat时进行
	    scheduleBlockReport(initialBlockReportDelay);
	  }


#### 2.2 启动datanode线程 —— dn.dataNodeThread.start();
datanode线程本身非常简单，不停调用offerSevice提供服务：
	
	  public void run() {
	    LOG.info(dnRegistration + "In DataNode.run, data = " + data);
	
	    // start dataXceiveServer
	    dataXceiverServer.start();
	    new Thread(new CrashVolumeChecker()).start();//added by wukong
	        
	    while (shouldRun) {
	      try {
	        startDistributedUpgradeIfNeeded();
	        offerService();
	      } catch (Exception ex) {
	        LOG.error("Exception: " + StringUtils.stringifyException(ex));
	        if (shouldRun) {
	          try {
	            Thread.sleep(5000);
	          } catch (InterruptedException ie) {
	          }
	        }
	      }
	    }
	        
	    LOG.info(dnRegistration + ":Finishing DataNode in: "+data);
	    shutdown();
	  }
	  
##### 2.2.1 offerService()
offerService的核心是周期性进行heartbeat和blockReport，主要流程如下：

![offerService](/images/hdfs/offerService.png)

	  public void offerService() throws Exception {
	     
	    LOG.info("using BLOCKREPORT_INTERVAL of " + blockReportInterval + "msec" + 
	       " Initial delay: " + initialBlockReportDelay + "msec");
	    LOG.info("using DELETEREPORT_INTERVAL of " + deletedReportInterval + "msec");
	    LOG.info("using HEARTBEAT_INTERVAL of " + heartBeatInterval + "msec");
	    LOG.info("using HEARTBEAT_EXPIRE_INTERVAL of " + heartbeatExpireInterval + "msec");
	
	    //
	    // Now loop for a long time....
	    //
	
	    while (shouldRun) {
	      try {
	        long startTime = now();
	
	        //
	        // Every so often, send heartbeat or block-report
	        //
	        
	        if (startTime - lastHeartbeat > heartBeatInterval /* 3 secs*/) {
	          //
	          // All heartbeat messages include following info:
	          // -- Datanode name
	          // -- data transfer port
	          // -- Total capacity
	          // -- Bytes remaining
	          //
	          lastHeartbeat = startTime;
	          DatanodeCommand[] cmds = namenode.sendHeartbeat(dnRegistration,
	                                                       data.getCapacity(),
	                                                       data.getDfsUsed(),
	                                                       data.getRemaining(),
	                                                       xmitsInProgress.get(),
	                                                       getXceiverCount());
	          myMetrics.heartbeats.inc(now() - startTime);
	          //LOG.info("Just sent heartbeat, with name " + localName);
	          if (!processCommand(cmds))
	            continue;
	        }
	        
	        reportReceivedBlocks();
	        
	        DatanodeCommand cmd = blockReport();
	        processCommand(cmd);
	
	        // start block scanner
	        if (blockScanner != null && blockScannerThread == null &&
	            upgradeManager.isUpgradeCompleted()) {
	          LOG.info("Starting Periodic block scanner.");
	          blockScannerThread = new Daemon(blockScanner);
	          blockScannerThread.start();
	        }
	            
	        //
	        // There is no work to do;  sleep until hearbeat timer elapses, 
	        // or work arrives, and then iterate again.
	        //
	        long waitTime = heartBeatInterval - (System.currentTimeMillis() - lastHeartbeat);
	        synchronized(receivedAndDeletedBlockList) {
	          if (waitTime > 0 && receivedAndDeletedBlockList.size() == 0) {
	            try {
	              receivedAndDeletedBlockList.wait(waitTime);
	            } catch (InterruptedException ie) {
	            }
	            delayBeforeBlockReceived();
	          }
	        } // synchronized
	        
	      } catch(RemoteException re) {
	        String reClass = re.getClassName();
	        if (UnregisteredDatanodeException.class.getName().equals(reClass) ||
	            DisallowedDatanodeException.class.getName().equals(reClass) ||
	            IncorrectVersionException.class.getName().equals(reClass)) {
	          LOG.warn("DataNode is shutting down: " + 
	                   StringUtils.stringifyException(re));
	          shutdown();
	          return;
	        }
	        LOG.warn(StringUtils.stringifyException(re));
	      } catch (IOException e) {
	        LOG.warn(StringUtils.stringifyException(e));
	      }
	    } // while (shouldRun)
	  } // offerService
	  