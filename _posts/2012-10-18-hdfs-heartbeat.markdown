---
layout: post
title: "HDFS源码学习(4)——DataNode心跳检测（HeartBeat）"
date: 2012-10-18 21:38
comments: true
categories: HDFS
---

HDFS中DataNode的心跳检测通过FSNameSystem中的HeartbeatMonitor完成。代码结构如下：
	
	  class HeartbeatMonitor implements Runnable {
	    /**
	     */
	    public void run() {
	      while (fsRunning) {
	        try {
	          heartbeatCheck();
	        } catch (Exception e) {
	          FSNamesystem.LOG.error(StringUtils.stringifyException(e));
	        }
	        try {
	          Thread.sleep(heartbeatRecheckInterval);
	        } catch (InterruptedException ie) {
	        }
	      }
	    }
	  }
	  
代码很简单，心跳检测线程周期性调用heartbeatCheck()。
##一、心跳检查——heartbeatCheck()
该方法主要用于检测是否有过期的心跳检测，如有，检测其上的block是否已经进行过重新备份。该线程每次只处理一个datanode。

	  void heartbeatCheck() {
	    if (isInSafeMode()) {
	      // 安全模式下不做心跳检测
	      return;
	    }
	    boolean allAlive = false;
	    while (!allAlive) {
	      boolean foundDead = false;
	      DatanodeID nodeID = null;
	
	      // 获取第一个dead datanode
	      synchronized(heartbeats) {
	        for (Iterator<DatanodeDescriptor> it = heartbeats.iterator();
	             it.hasNext();) {
	          DatanodeDescriptor nodeInfo = it.next();
	          if (isDatanodeDead(nodeInfo)) {
	            foundDead = true;
	            nodeID = nodeInfo;
	            break;
	          }
	        }
	      }
	
	      // 申请fsnamesystem锁，删除dead datanode
	      if (foundDead) {
	        writeLock();
	        try {
	          synchronized(heartbeats) {
	            synchronized (datanodeMap) {
	              DatanodeDescriptor nodeInfo = null;
	              try {
	                nodeInfo = getDatanode(nodeID);
	              } catch (IOException e) {
	                nodeInfo = null;
	              }
	              if (nodeInfo != null && isDatanodeDead(nodeInfo)) {
	                NameNode.stateChangeLog.info("BLOCK* NameSystem.heartbeatCheck: "
	                                             + "lost heartbeat from " + nodeInfo.getName());
	                removeDatanode(nodeInfo);
	              }
	            }
	          }
	        } finally {
	          writeUnlock();
	        }
	      }
	      allAlive = !foundDead;
	    }
	  }
	  
###1.1 判断是否已死 —— isDatanodeDead（）
判断一个datanode是否已经dead的标准很简单，当前距该节点最后的更新时间差是否已经超过心跳检测的过期时间限制

	private boolean isDatanodeDead(DatanodeDescriptor node) {
	    return (node.getLastUpdate() <
	            (now() - heartbeatExpireInterval));
	  }

###1.2 删除datanode —— removeDatanode（）
	
	  private void removeDatanode(DatanodeDescriptor nodeInfo) {
	    synchronized (heartbeats) {
	      if (nodeInfo.isAlive) {
	        updateStats(nodeInfo, false);
	        //从heartbeats中移除
	        heartbeats.remove(nodeInfo);
	        //更新datanode状态
	        nodeInfo.isAlive = false;
	      }
	    }
	
	    nodeInfo.hasInitialBlockReport = false;
	    for (Iterator<Block> it = nodeInfo.getBlockIterator(); it.hasNext();) {
	      //移除该节点上的block
	      removeStoredBlock(it.next(), nodeInfo);
	    }
	    unprotectedRemoveDatanode(nodeInfo);
	    clusterMap.remove(nodeInfo);
	  }
#### 1.2.1 removeStoredBlock（）
该方法更新block->datanode的映射(blocksMap)，如果block还有效，有可能导致block备份发生

	  void removeStoredBlock(Block block, DatanodeDescriptor node) {
	    if (NameNode.stateChangeLog.isDebugEnabled()) {
	      NameNode.stateChangeLog.debug("BLOCK* NameSystem.removeStoredBlock: "
	                                    +block + " from "+node.getName());
	    }
	    assert (hasWriteLock());
	    if (!blocksMap.removeNode(block, node)) {
	      if (NameNode.stateChangeLog.isDebugEnabled()) {
	        NameNode.stateChangeLog.debug("BLOCK* NameSystem.removeStoredBlock: "
	                                      +block+" has already been removed from node "+node);
	      }
	      return;
	    }
	        
	    //
	   	//检查是否需要备份删除的block
	    INode fileINode = blocksMap.getINode(block);
	    if (fileINode != null) {
	      //减小当前系统中安全block（备份数满足最小值的block）数量
	      decrementSafeBlockCount(block);
	      //更新需要备份的block数量
	      updateNeededReplications(block, -1, 0);
	    }
	
	    //
	    // 从excessblocks中删除改block，并从excessReplicateMap删除改datanode
	    Collection<Block> excessBlocks = excessReplicateMap.get(node.getStorageID());
	    if (excessBlocks != null) {
	      if (excessBlocks.remove(block)) {
	        excessBlocksCount--;
	        if (NameNode.stateChangeLog.isDebugEnabled()) {
	          NameNode.stateChangeLog.debug("BLOCK* NameSystem.removeStoredBlock: "
	              +block+" is removed from excessBlocks");
	        }
	        if (excessBlocks.size() == 0) {
	          excessReplicateMap.remove(node.getStorageID());
	        }
	      }
	    }
	    
	    // 从corruptReplicas中移除该block
	    corruptReplicas.removeFromCorruptReplicasMap(block, node);
	  }
##### 1.2.1.1  BlocksMap.removeNode（）  
其中BlocksMap.removeNode()方法如下：

	  boolean removeNode(Block b, DatanodeDescriptor node) {
	    BlockInfo info = blocks.get(b);
	    if (info == null)
	      return false;
	
	    // 从datanode 的blocklist中移除block，并从block的datalist中移除datanode
	    boolean removed = node.removeBlock(info);
	
	    if (info.getDatanode(0) == null     // no datanodes left
	              && info.inode == null) {  // does not belong to a file
	      //从blocksmap中移除该block
	      blocks.remove(b);  // remove block from the map
	    }
	    return removed;
	  }
	  
##### 1.2.1.2 减小当前安全的block数 —— decrementSafeBlockCount()
减小当前副本数安全的的block数，此举有可能触发系统进入安全模式（safemode）

	  void decrementSafeBlockCount(Block b) {
	    if (safeMode == null) // mostly true
	      return;
	    
	    safeMode.decrementSafeBlockCount((short)countNodes(b).liveReplicas());
	  }
	  
其中safeMode.decrementSafeBlockCount()代码如下：

    synchronized void decrementSafeBlockCount(short replication) {

      if (replication == safeReplication-1) {
        //安全的block数减一
        this.blockSafe--;
        //检查是否需要进入到safemode
        checkMode();
      }
    }
    
SafeModeInfo.checkMode()代码如下：

	    private void checkMode() {
	      //当安全的block数比例降至安全值以下，进入安全模式
	      if (needEnter()) {
	        enter();
	        // check if we are ready to initialize replication queues
	        if (canInitializeReplQueues() && !isPopulatingReplQueues()) {
	          //初始化副本队列
	          initializeReplQueues();
	        }
	        reportStatus("STATE* Safe mode ON.", false);
	        return;
	      }
	      // 如果安全模式已经关闭或者门槛小于0，则跳出安全模式
	      if (!isOn() ||                           // safe mode is off
	          extension <= 0 || threshold <= 0) {  // don't need to wait
	        this.leave(true); // leave safe mode
	        return;
	      }
	      
	      //之前已经进入安全模式，直接返回
	      if (reached > 0) {  // threshold has already been reached before
	        reportStatus("STATE* Safe mode ON.", false);
	        return;
	      }
	      // 启动SafeModeMonitor线程
	      reached = now();
	      smmthread = new Daemon(new SafeModeMonitor());
	      smmthread.start();
	      reportStatus("STATE* Safe mode extension entered.", true);
	    }

##### 1.2.1.3 更新需要备份的列表 —— updateNeededReplications（）

	  /* updates a block in under replication queue */
	  void updateNeededReplications(Block block,
	                        int curReplicasDelta, int expectedReplicasDelta) {
	    writeLock();
	    try {
	    //计算当前副本数
	    NumberReplicas repl = countNodes(block);
	    //期望的副本数
	    int curExpectedReplicas = getReplication(block);
	    //将该block更新到需要备份的列表中（neededReplications）
	    neededReplications.update(block, 
	                              repl.liveReplicas(), 
	                              repl.decommissionedReplicas(),
	                              curExpectedReplicas,
	                              curReplicasDelta, expectedReplicasDelta);
	    } finally {
	      writeUnlock();
	    }
	  }
		    
#### 1.2.2 移除datanode —— unprotectedRemoveDatanode

	  void unprotectedRemoveDatanode(DatanodeDescriptor nodeDescr) {
	    //重置清空datanode中block信息
	    nodeDescr.resetBlocks();
	    //从invlidateSet中移除datanode
	    removeFromInvalidates(nodeDescr.getStorageID());
	    if (NameNode.stateChangeLog.isDebugEnabled()) {
	      NameNode.stateChangeLog.debug(
	                                    "BLOCK* NameSystem.unprotectedRemoveDatanode: "
	                                    + nodeDescr.getName() + " is out of service now.");
	    }
	  }

##### 1.2.2.1 removeFromInvalidates()
	
	  private void removeFromInvalidates(String storageID) {
	    //从recentInvalidateSet中移除该datanode
	    Collection<Block> blocks = recentInvalidateSets.remove(storageID);
	    if (blocks != null) {
	      //从正在删除的block总数中减去当前节点上的block总数
	      pendingDeletionBlocksCount -= blocks.size();
	    }
	  }
##二、 处理心跳检测请求 —— handleHeartbeat()
NameNode只负责创建一个HeartbeatMonitor来通过每个datanode的最新更新时间周期性检查是否有过期的datanode，而每个datanode是否的最新更新时间是由datanode主动向namenode报告的，namenode通过handleHeartbeat()处理心跳请求。
	
	  DatanodeCommand[] handleHeartbeat(DatanodeRegistration nodeReg,
	      long capacity, long dfsUsed, long remaining,
	      int xceiverCount, int xmitsInProgress) throws IOException {
	    DatanodeCommand cmd = null;
	    synchronized (heartbeats) {
	      synchronized (datanodeMap) {
	        DatanodeDescriptor nodeinfo = null;
	        try {
	          nodeinfo = getDatanode(nodeReg);
	        } catch(UnregisteredDatanodeException e) {
	          return new DatanodeCommand[]{DatanodeCommand.REGISTER};
	        }
	          
	        // 检查该datanode是否需要被关闭，可以通过设置datanode的adminState为DECOMMISSIONED来关闭一个datanode
	        if (nodeinfo != null && shouldNodeShutdown(nodeinfo)) {
	          setDatanodeDead(nodeinfo);
	          throw new DisallowedDatanodeException(nodeinfo);
	        }
	
	        if (nodeinfo == null || !nodeinfo.isAlive) {
	          return new DatanodeCommand[]{DatanodeCommand.REGISTER};
	        }
	
	        updateStats(nodeinfo, false);
	        nodeinfo.updateHeartbeat(capacity, dfsUsed, remaining, xceiverCount);
	        updateStats(nodeinfo, true);
	        
	        //检查租约恢复状态
	        cmd = nodeinfo.getLeaseRecoveryCommand(Integer.MAX_VALUE);
	        if (cmd != null) {
	          return new DatanodeCommand[] {cmd};
	        }
	      
	        ArrayList<DatanodeCommand> cmds = new ArrayList<DatanodeCommand>(2);
	        //检查正在备份中的副本
	        cmd = nodeinfo.getReplicationCommand(
	              maxReplicationStreams - xmitsInProgress);
	        if (cmd != null) {
	          cmds.add(cmd);
	        }
	        //检查无效的block
	        cmd = nodeinfo.getInvalidateBlocks(blockInvalidateLimit);
	        if (cmd != null) {
	          cmds.add(cmd);
	        }
	        if (!cmds.isEmpty()) {
	          return cmds.toArray(new DatanodeCommand[cmds.size()]);
	        }
	      }
	    }
	
	    //检查是否需要升级系统
	    cmd = getDistributedUpgradeCommand();
	    if (cmd != null) {
	      return new DatanodeCommand[] {cmd};
	    }
	    return null;
	  }