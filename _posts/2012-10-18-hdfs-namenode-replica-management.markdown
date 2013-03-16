---
layout: post
title: "HDFS源码学习（5）——副本管理（Replica Management)"
date: 2012-10-18 21:43
comments: true
categories: HDFS
---

HDFS中的副本管理通过FSNameSystem.java中的ReplicationMonitor线程来完成。该线程代码结构较为简单。

	  class ReplicationMonitor implements Runnable {
	    static final int INVALIDATE_WORK_PCT_PER_ITERATION = 32;
	    static final float REPLICATION_WORK_MULTIPLIER_PER_ITERATION = 2;
	    public void run() {
	      while (fsRunning) {
	        try {
	          computeDatanodeWork();
	          processPendingReplications();
	          Thread.sleep(replicationRecheckInterval);
	        } catch (InterruptedException ie) {
	          LOG.warn("ReplicationMonitor thread received InterruptedException.", ie);
	          break;
	        } catch (IOException ie) {
	          LOG.warn("ReplicationMonitor thread received exception. " + ie +  " " +
	              StringUtils.stringifyException(ie));
	        } catch (Throwable t) {
	          LOG.fatal("ReplicationMonitor thread received Runtime exception. " + t + " " +
	              StringUtils.stringifyException(t));
	          Runtime.getRuntime().exit(-1);
	        }
	      }
	    }
	  }
	  
该线程只是周期性调用computeDatanodeWork()和processPendingReplications()。

##一、computeDatanodeWork()
	
	  public int computeDatanodeWork() throws IOException {
	    int workFound = 0;
	    int blocksToProcess = 0;
	    int nodesToProcess = 0;
	    // blocks should not be replicated or removed if safe mode is on
	    if (isInSafeMode())
	      return workFound;
	    //计算需要备份的block数和节点数
	    synchronized(heartbeats) {
	      blocksToProcess = (int)(heartbeats.size() 
	          * ReplicationMonitor.REPLICATION_WORK_MULTIPLIER_PER_ITERATION);
	      nodesToProcess = (int)Math.ceil((double)heartbeats.size() 
	          * ReplicationMonitor.INVALIDATE_WORK_PCT_PER_ITERATION / 100);
	    }
		//执行备份
	    workFound = computeReplicationWork(blocksToProcess); 
	    
	    // Update FSNamesystemMetrics counters
	    pendingReplicationBlocksCount = pendingReplications.size();
	    underReplicatedBlocksCount = neededReplications.size();
	    scheduledReplicationBlocksCount = workFound;
	    corruptReplicaBlocksCount = corruptReplicas.size();
	    
	    //HADOOP-5549 : Fix bug of schedule both replication and deletion work in one iteration
	    workFound += computeInvalidateWork(nodesToProcess);
	    return workFound;
	  }

###1.1. computeReplicationWork()
	
	  private int computeReplicationWork(
	                                  int blocksToProcess) throws IOException {
	    // stall only useful for unit tests (see TestFileAppend4.java)
	    if (stallReplicationWork)  {
	      return 0;
	    }
	    
	    // 选取需要备份的block
	    List<List<Block>> blocksToReplicate =
	      chooseUnderReplicatedBlocks(blocksToProcess);
	
	    // 执行备份
	    return computeReplicationWorkForBlocks(blocksToReplicate);
	  }
	  
###1.1.1 选取需要备份的block —— chooseUnderReplicatedBlocks()

	  List<List<Block>> chooseUnderReplicatedBlocks(int blocksToProcess) {
	    // 初始化返回值数据结构，返回值是一个二维优先级列表
	    List<List<Block>> blocksToReplicate =
	      new ArrayList<List<Block>>(UnderReplicatedBlocks.LEVEL);
	    for (int i = 0; i < UnderReplicatedBlocks.LEVEL; i++) {
	      blocksToReplicate.add(new ArrayList<Block>());
	    }
	
	    writeLock();
	    try {
	      synchronized (neededReplications) {
	        if (neededReplications.size() == 0) {
	          return blocksToReplicate;
	        }
	
	        for (int priority = 0; priority<UnderReplicatedBlocks.LEVEL; priority++) {
	        //遍历所有需要备份的block列表（UnderReplicatedBlocks结构）
	        BlockIterator neededReplicationsIterator = neededReplications.iterator(priority);
	        int numBlocks = neededReplications.size(priority);
	        //检查该优先级列表中是否已经开始备份（relIndex数组中保存的是当前每个优先级列表中已备份的block索引）
	        if (replIndex[priority] > numBlocks) {
	          replIndex[priority] = 0;
	        }
	        // skip to the first unprocessed block, which is at replIndex
	        for (int i = 0; i < replIndex[priority] && neededReplicationsIterator.hasNext(); i++) {
	          neededReplicationsIterator.next();
	        }
	        // 计算该优先级下需要备份的block数，低优先级的block备份数不超过总配额的20%
	        int blocksToProcessIter = getQuotaForThisPriority(blocksToProcess,
	            numBlocks, neededReplications.getSize(priority+1));
	        blocksToProcess -= blocksToProcessIter;
	
			//便利改优先级列表将该优先级下的block添加到返回值中
	        for (int blkCnt = 0; blkCnt < blocksToProcessIter; blkCnt++, replIndex[priority]++) {
	          if (!neededReplicationsIterator.hasNext()) {
	            // start from the beginning
	            replIndex[priority] = 0;
	            neededReplicationsIterator = neededReplications.iterator(priority);
	            assert neededReplicationsIterator.hasNext() :
	              "neededReplications should not be empty.";
	          }
			
	          Block block = neededReplicationsIterator.next();
	          blocksToReplicate.get(priority).add(block);
	        } // end for
	        }
	      } // end try
	      return blocksToReplicate;
	    } finally {
	      writeUnlock();
	    }
	  }
	  
###1.1.2. 备份blocks —— computeReplicationWorkForBlocks（）
	  int computeReplicationWorkForBlocks(List<List<Block>> blocksToReplicate) {
	    int requiredReplication, numEffectiveReplicas, priority;
	    List<DatanodeDescriptor> containingNodes;
	    DatanodeDescriptor srcNode;
	    INodeFile fileINode = null;
	
	    int scheduledWork = 0;
	    List<ReplicationWork> work = new LinkedList<ReplicationWork>();
	
	    writeLock();
	    try {
	      synchronized (neededReplications) {
	        for (priority = 0; priority < blocksToReplicate.size(); priority++) {
	          for (Block block : blocksToReplicate.get(priority)) {
	            // block should belong to a file
	            //获取该block所属的INode
	            fileINode = blocksMap.getINode(block);
	            // abandoned block not belong to a file
	            if (fileINode == null ) {
	              neededReplications.remove(block, priority); // remove from neededReplications
	              replIndex[priority]--;
	              continue;
	            }
	            //获取该文件需要的副本数
	            requiredReplication = fileINode.getReplication();
	
	            // 获取一个源datanode节点
	            containingNodes = new ArrayList<DatanodeDescriptor>();
	            NumberReplicas numReplicas = new NumberReplicas();
	            srcNode = chooseSourceDatanode(block, containingNodes, numReplicas);
	            if (srcNode == null) // block can not be replicated from any node
	            {
	              continue;
	            }
	
	          // 检查正在备份中的副本数是否满足备份需要，满足则不需要再备份
	            numEffectiveReplicas = numReplicas.liveReplicas() +
	              pendingReplications.getNumReplicas(block);
	            if (numEffectiveReplicas >= requiredReplication) {
	              neededReplications.remove(block, priority); // remove from neededReplications
	              replIndex[priority]--;
	              continue;
	            }
	            //添加到待备份列表中
	            work.add(new ReplicationWork(block, fileINode, requiredReplication
	                - numEffectiveReplicas, srcNode, containingNodes, priority));
	          }
	        }
	      }
	    } finally {
	      writeUnlock();
	    }
	
	    // 选取一个备份目标datanode
	    for(ReplicationWork rw : work){
	      DatanodeDescriptor targets[] = chooseTarget(rw);
	      rw.targets = targets;
	    }
	
	    writeLock();
	    try {
	      for(ReplicationWork rw : work){
	        DatanodeDescriptor[] targets = rw.targets;
	        if(targets == null || targets.length == 0){
	          rw.targets = null;
	          continue;
	        }
	        synchronized (neededReplications) {
	          Block block = rw.block;
	          priority = rw.priority;
	          // 重新检查INode和备份数，因为全局锁已经释放
	          // block should belong to a file
	          fileINode = blocksMap.getINode(block);
	          // abandoned block not belong to a file
	          if (fileINode == null ) {
	            neededReplications.remove(block, priority); // remove from neededReplications
	            rw.targets = null;
	            replIndex[priority]--;
	            continue;
	          }
	          requiredReplication = fileINode.getReplication();
	

	          NumberReplicas numReplicas = countNodes(block);
	          numEffectiveReplicas = numReplicas.liveReplicas() +
	            pendingReplications.getNumReplicas(block);
	          if (numEffectiveReplicas >= requiredReplication) {
	            neededReplications.remove(block, priority); // remove from neededReplications
	            replIndex[priority]--;
	            rw.targets = null;
	            continue;
	          }
	
	          // 将block添加到datanode的需要备份的block列表中
	          rw.srcNode.addBlockToBeReplicated(block, targets);
	          
	          scheduledWork++;
	
			  //设置namenode的block调度计数器
	          for (DatanodeDescriptor dn : targets) {
	            dn.incBlocksScheduled();
	          }
	
	          // Move the block-replication into a "pending" state.
	          // The reason we use 'pending' is so we can retry
	          // replications that fail after an appropriate amount of time.
	          //将该block移至pendingReplications（PendingReplicationBlocks）中，表示该block的状态为'pending'(正在备份中)。'pending'表示如果失败了还可以重试
	          pendingReplications.add(block, targets.length);
	          NameNode.stateChangeLog.debug(
	            "BLOCK* block " + block
	              + " is moved from neededReplications to pendingReplications");
	
	          // remove from neededReplications
	          //从 neededReplication列表中移除该block
	          if (numEffectiveReplicas + targets.length >= requiredReplication) {
	            neededReplications.remove(block, priority); // remove from neededReplications
	            replIndex[priority]--;
	          }
	        }
	      }
	    } finally {
	      writeUnlock();
	    }
	
	    // 更新 metrics
	    updateReplicationMetrics(work);
	    
	    // 打印debug信息
	    if(NameNode.stateChangeLog.isInfoEnabled()){
	      // log which blocks have been scheduled for replication
	      for(ReplicationWork rw : work){
	        // report scheduled blocks
	        DatanodeDescriptor[] targets = rw.targets;
	        if (targets != null && targets.length != 0) {
	          StringBuffer targetList = new StringBuffer("datanode(s)");
	          for (int k = 0; k < targets.length; k++) {
	            targetList.append(' ');
	            targetList.append(targets[k].getName());
	          }
	          NameNode.stateChangeLog.info(
	            "BLOCK* ask "
	              + rw.srcNode.getName() + " to replicate "
	              + rw.block + " to " + targetList);
	        }
	      }
	    }
	
	    // 记录一次备份操作
	    if (NameNode.stateChangeLog.isDebugEnabled()) {
	      NameNode.stateChangeLog.debug("BLOCK* neededReplications = "
	          + neededReplications.size() + " pendingReplications = "
	          + pendingReplications.size());
	    }
	    return scheduledWork;
	  }
	  
####1.1.2.1. 选取源datanode —— chooseSourceDatanode（）
获取一个源datanode节点

	  private DatanodeDescriptor chooseSourceDatanode(
	                                    Block block,
	                                    List<DatanodeDescriptor> containingNodes,
	                                    NumberReplicas numReplicas) {
	    containingNodes.clear();
	    DatanodeDescriptor srcNode = null;
	    int live = 0;
	    int decommissioned = 0;
	    int corrupt = 0;
	    int excess = 0;
	    Iterator<DatanodeDescriptor> it = blocksMap.nodeIterator(block);
	    Collection<DatanodeDescriptor> nodesCorrupt = corruptReplicas.getNodes(block);
	    while(it.hasNext()) {
	      DatanodeDescriptor node = it.next();
	      Collection<Block> excessBlocks = 
	        excessReplicateMap.get(node.getStorageID());
	      if ((nodesCorrupt != null) && (nodesCorrupt.contains(node)))
	        corrupt++;
	      else if (node.isDecommissionInProgress() || node.isDecommissioned())
	        decommissioned++;
	      else if (excessBlocks != null && excessBlocks.contains(block)) {
	        excess++;
	      } else {
	        live++;
	      }
	      containingNodes.add(node);
	      // Check if this replica is corrupt
	      // If so, do not select the node as src node
	      if ((nodesCorrupt != null) && nodesCorrupt.contains(node))
	        continue;
	      if(node.getNumberOfBlocksToBeReplicated() >= maxReplicationStreams)
	        continue; // already reached replication limit
	      // the block must not be scheduled for removal on srcNode
	      if(excessBlocks != null && excessBlocks.contains(block))
	        continue;
	      // never use already decommissioned nodes
	      if(node.isDecommissioned())
	        continue;
	      // we prefer nodes that are in DECOMMISSION_INPROGRESS state
	      if(node.isDecommissionInProgress() || srcNode == null) {
	        srcNode = node;
	        continue;
	      }
	      if(srcNode.isDecommissionInProgress())
	        continue;
	      // switch to a different node randomly
	      // this to prevent from deterministically selecting the same node even
	      // if the node failed to replicate the block on previous iterations
	      if(r.nextBoolean())
	        srcNode = node;
	    }
	    if(numReplicas != null)
	      numReplicas.initialize(live, decommissioned, corrupt, excess);
	    return srcNode;
	  }
####1.1.2.2. 选取目标datanode —— chooseTarget(rw);

	  private DatanodeDescriptor[] chooseTarget(ReplicationWork work) {
	    if (!neededReplications.contains(work.block)) {
	      return null;
	    }
	    if (work.blockSize == BlockCommand.NO_ACK) {
	      LOG.warn("Block " + work.block.getBlockId() + 
	          " of the file " + work.fileINode.getFullPathName() + 
	          " is invalidated and cannot be replicated.");
	      return null;
	    }
	    if (work.blockSize == DFSUtil.DELETED) {
	      LOG.warn("Block " + work.block.getBlockId() + 
	          " of the file " + work.fileINode.getFullPathName() + 
	          " is a deleted block and cannot be replicated.");
	      return null;
	    }
	    //实际调用replicator(BlockPlacementPolicy)的chooseTarget方法选取target
	    return replicator.chooseTarget(work.fileINode,
	        work.numOfReplicas, work.srcNode,
	        work.containingNodes, null, work.blockSize);
	  }
	
##副本存放策略

![BlockAllocation](img/BlockAllocation.png)

如图所示，HDFS默认的副本存放策略为：

1. 第一个副本存放在当前datanode的本地
2. 第二个副本存放在与第一个副本所在datanode不在同一机架上的一个datanode上
3. 第三个副本存放在与第二个副本同一机架但不同datanode上

