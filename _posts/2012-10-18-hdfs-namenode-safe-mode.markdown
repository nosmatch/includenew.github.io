---
layout: post
title: "HDFS源码学习（9）——安全模式（SafeMode）"
date: 2012-10-18 21:52
comments: true
categories: HDFS
---

## 一、SafeModeInfo
SafeModeInfo维护了系统安全模式下的状态信息，每当系统进入安全模式时都会创建一个SafeModeInfo实例维护状态信息，离开时会销毁这个实例。
该类结构如下

![SafeModeInfo](/images/hdfs/SafeModeInfo.png)

其中threshold和extension为可配置项：

1. threshold表示离开安全模式时打到最低备份数的block的比例
2. extension表示进入安全模式的最低时长



## 二、SafeModeMonitor
FSNameSystem中SafeModeMonitor代码结构如下：
	
	  class SafeModeMonitor implements Runnable {
	    /** interval in msec for checking safe mode: {@value} */
	    private static final long recheckInterval = 1000;
	      
	    /**
	     */
	    public void run() {
	      while (fsRunning && (safeMode != null && !safeMode.canLeave())) {
	        try {
	          Thread.sleep(recheckInterval);
	        } catch (InterruptedException ie) {
	        }
	      }
	      // leave safe mode and stop the monitor
	      try {
	        leaveSafeMode(true);
	      } catch(SafeModeException es) { // should never happen
	        String msg = "SafeModeMonitor may not run during distributed upgrade.";
	        assert false : msg;
	        throw new RuntimeException(msg, es);
	      }
	      smmthread = null;
	    }
	  }
其核心就是每个1秒检测一次是否能够离开模式（safeMode.canLeave()），如果可以，则尝试离开并停止SafeModeMonitor线程（leaveSafeMode(true)）
###1.1. 是否能离开 —— safeMode.canLeave()
能够离开安全模式的标准是：
1. 已进入安全模式的时长大于等于 extension
2. 安全的block数比例打到门槛值

    synchronized boolean canLeave() {
      if (reached == 0)
        return false;
      if (now() - reached < extension) {
        reportStatus("STATE* Safe mode ON.", false);
        return false;
      }
      return !needEnter();
    }
      
    /** 
     * There is no need to enter safe mode 
     * if DFS is empty or {@link #threshold} == 0
     */
    boolean needEnter() {
      return getSafeBlockRatio() < threshold;
    }

###1.2. 离开安全模式 —— leaveSafeMode(true);

	  public void leaveSafeMode(boolean checkForUpgrades) throws SafeModeException {
	    writeLock();
	    try {
	    if (!isInSafeMode()) {
	      NameNode.stateChangeLog.info("STATE* Safe mode is already OFF."); 
	      return;
	    }
	    //获取升级状态，如在升级中，不能离开安全模式
	    if(getDistributedUpgradeState())
	      throw new SafeModeException("Distributed upgrade is in progress",
	                                  safeMode);
		//调用SafeModeInfo.leave()离开安全模式
	    safeMode.leave(checkForUpgrades);
	    } finally {
	      writeUnlock();
	    }
	  }

### 1.2.1 SafeModeInfo.leave()

	    synchronized void leave(boolean checkForUpgrades) {
	      if(checkForUpgrades) {
	        // 验证是否需要升级
	        boolean needUpgrade = false;
	        try {
	          needUpgrade = startDistributedUpgradeIfNeeded();
	        } catch(IOException e) {
	          FSNamesystem.LOG.error(StringUtils.stringifyException(e));
	        }
	        if(needUpgrade) {
	          //如果需要升级，进入手动安全模式
	          safeMode = new SafeModeInfo();
	          return;
	        }
	      }
	      // 如果备份队列未初始化完，继续初始化该队列
	      if (!isPopulatingReplQueues()) {
	        initializeReplQueues();
	      }
	      long timeInSafemode = now() - systemStart;
	      NameNode.stateChangeLog.info("STATE* Leaving safe mode after " 
	                                    + timeInSafemode/1000 + " secs.");
	      NameNode.getNameNodeMetrics().safeModeTime.set((int) timeInSafemode);
	      
	      if (reached >= 0) {
	        NameNode.stateChangeLog.info("STATE* Safe mode is OFF."); 
	      }
	      reached = -1;
	      safeMode = null;
	      NameNode.stateChangeLog.info("STATE* Network topology has "
	                                   +clusterMap.getNumOfRacks()+" racks and "
	                                   +clusterMap.getNumOfLeaves()+ " datanodes");
	      NameNode.stateChangeLog.info("STATE* UnderReplicatedBlocks has "
	                                   +neededReplications.size()+" blocks");
	    }