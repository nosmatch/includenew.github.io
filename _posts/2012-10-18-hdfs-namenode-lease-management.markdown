---
layout: post
title: "HDFS源码学习（6）——租约管理（lease management)"
date: 2012-10-18 21:44
comments: true
categories: HDFS
---

LeaseManagement是HDFS中的一个同步机制，用于保证同一时刻只有一个client对一个文件进行写或创建操作。如当新建一个文件f时，client向NameNode发起一个create请求，那么leaseManager会想该client分配一个f文件的lease。client凭借该lease完成文件的创建操作。此时其他client无法获得f的当client长时间（默认为超过1min）不进行操作时，发放的lease将被收回。

LeaseManager主要完成两部分工作：

1. 文件create，write，complete操作时，创建lease、更新时间戳、回收lease
2. 一个后台线程定期检查是否有过期的lease

LeaseManager的代码结构如下

![LeaseManager](/images/hdfs/LeaseManager.png)

其中Lease表示一个租约，包括一个client(holder)所拥有的所有文件锁(paths)。

Monitor是检查是否有过期租约的线程。

LeaseManager中有几个主要数据结构：

1. leases（TreeMap<String, Lease>）：维护holder -> leased的映射集合
2. sortedLeases (TreeSet<Lease>): lease集合
3. sortedLeaseByPath(TreeMap<String, Lease>): 维护paths->lease的映射集合

##一、创建lease
当client向NameNode发起create操作时，NameNode.create()调用FSNameSystem.startFile()->FSNameSystem.startFileInternal()，该方法最终会调用leaseManager.addLease(cons.clientName, src)来创建lease。

LeaseManager.addLease()方法如下：
	
	  synchronized Lease addLease(String holder, String src
	      ) throws IOException {
	    Lease lease = getLease(holder);
	    if (lease == null) {
	      lease = new Lease(holder);
	      leases.put(holder, lease);
	      sortedLeases.add(lease);
	    } else {
	      renewLease(lease);
	    }
	    sortedLeasesByPath.put(src, lease);
	    lease.paths.add(src);
	    return lease;
	  }
代码结构简单：判断该client是否有lease，没有则新建一个lease，并将起加到leases集合中。否则更新lease。更新sortedLeasesByPath，将filepath加入到该lease的paths集合中

##二、更新时间戳
针对已经存在的lease，通过LeasemManager.renewLease()来更新该lease的时间戳。代码如下：

	  synchronized void renewLease(Lease lease) {
	    if (lease != null) {
	      sortedLeases.remove(lease);
	      lease.renew();
	      sortedLeases.add(lease);
	    }
	  }
	  
lease.renew()代码如下：

    /** Only LeaseManager object can renew a lease */
    private void renew() {
      this.lastUpdate = FSNamesystem.now();
    }
    
##三、compelete时回收lease
当client调用NameNode.complete()方法时，最终会调用FSNameSystem.completeFileInternal()方法。其中执行finalizeINodeFileUnderConstruction()是调用leaseManager.removeLease()释放lease。

代码结构如下：

	  synchronized void removeLease(String holder, String src) {
	    Lease lease = getLease(holder);
	    if (lease != null) {
	      removeLease(lease, src);
	    }
	  }
 removeLease(lease, src);代码如下：
	 
	  /**
	   * Remove the specified lease and src.
	   */
	  synchronized void removeLease(Lease lease, String src) {
	    sortedLeasesByPath.remove(src);
	    if (!lease.removePath(src)) {
	      LOG.error(src + " not found in lease.paths (=" + lease.paths + ")");
	    }
	
	    if (!lease.hasPath()) {
	      leases.remove(lease.holder);
	      if (!sortedLeases.remove(lease)) {
	        LOG.error(lease + " not found in sortedLeases");
	      }
	    }
	  }
	  
##四、后台线程回收过期lease
Monitor回收lease线程代码结构如下：

	 class Monitor implements Runnable {
	    final String name = getClass().getSimpleName();
	
	    /** Check leases periodically. */
	    public void run() {
	      for(; fsnamesystem.isRunning(); ) {
	        fsnamesystem.writeLock();
	        try {
	          checkLeases();
	        } finally {
	          fsnamesystem.writeUnlock();
	        }
	
	        try {
	          Thread.sleep(2000);
	        } catch(InterruptedException ie) {
	          if (LOG.isDebugEnabled()) {
	            LOG.debug(name + " is interrupted", ie);
	          }
	        }
	      }
	    }
	  }
代码结构简单，每个2s周期性执行checkLeases()。
###4.1 checkLeases()

	  /** Check the leases beginning from the oldest. */
	  synchronized void checkLeases() {
	    for(; sortedLeases.size() > 0; ) {
	      final Lease oldest = sortedLeases.first();
	      if (!oldest.expiredHardLimit()) {
	        return;
	      }
	
	      LOG.info("Lease " + oldest + " has expired hard limit");
	
	      final List<String> removing = new ArrayList<String>();
	      // need to create a copy of the oldest lease paths, becuase 
	      // internalReleaseLease() removes paths corresponding to empty files,
	      // i.e. it needs to modify the collection being iterated over
	      // causing ConcurrentModificationException
	      String[] leasePaths = new String[oldest.getPaths().size()];
	      oldest.getPaths().toArray(leasePaths);
	      for(String p : leasePaths) {
	        try {
	          fsnamesystem.internalReleaseLeaseOne(oldest, p);
	        } catch (IOException e) {
	          LOG.error("Cannot release the path "+p+" in the lease "+oldest, e);
	          removing.add(p);
	        }
	      }
	
	      for(String p : removing) {
	        removeLease(oldest, p);
	      }
	    }
	  }
## Lease Recovery ——租约回收
### lease recovery时机
lease发放之后，在不用时会被回收，回收的产经除上述Monitor线程检测lease过期是回收外，还有：

1. NameNode收到DataNode的Sync block command时
2. DFSClient主动关闭一个流时
3. 创建文件时，如果该DFSClient的lease超过soft limit时

### lease recovery 算法

1) NameNode查找lease信息

2) 对于lease中的每个文件f，令b为f的最后一个block，作如下操作：

2.1) 获取b所在的datanode列表

2.2) 令其中一个datanode作为primary datanode p

2.3) p 从NameNode获取最新的时间戳

2.4) p 从每个DataNode获取block信息

2.5) p 计算最小的block长度

2.6) p 用最小的block长度和最新的时间戳来更新具有有效时间戳的datanode

2.7) p 通知NameNode更新结果

2.8) NameNode更新BlockInfo

2.9) NameNode从lease中删除f，如果此时该lease中所有文件都已被删除，将删除该lease

2.10) Name提交修改的EditLog



##Client续约 —— DFSClient.LeaseChecker

在NameNode上的LeaseManager.Monitor线程负责检查过期的lease，那么client为了防止尚在使用的lease过期，需要定期想NameNode发起续约请求。该任务有DFSClient中的LeaseChecker完成。

LeaseChecker结构如下：

![LeaseChecker](/images/hdfs/LeaseChecker.png)

其中pendingCreates是一个TreeMap<String, OutputStream>用来维护src->当前正在写入的文件的DFSOutputStream的映射。

其核心是周期性（每个1s）调用run()方法来对租约过半的lease进行续约

    public void run() {
      long lastRenewed = 0;
      while (clientRunning && !Thread.interrupted()) {
        //当租约周期过半时需要进行续约
        if (System.currentTimeMillis() - lastRenewed > (LEASE_SOFTLIMIT_PERIOD / 2)) {
          try {
            renew();
            lastRenewed = System.currentTimeMillis();
          } catch (IOException ie) {
            LOG.warn("Problem renewing lease for " + clientName, ie);
          }
        }

        try {
          Thread.sleep(1000);
        } catch (InterruptedException ie) {
          if (LOG.isDebugEnabled()) {
            LOG.debug(this + " is interrupted.", ie);
          }
          return;
        }
      }
    }
其中renew()方法如下：

		private void renew() throws IOException {
	      synchronized(this) {
	        //如果当前创建中的文件列表为空，则不需要续约
	        if (pendingCreates.isEmpty()) {
	          return;
	        }
	      }
	      //向NameNode发起续约请求
	      namenode.renewLease(clientName);
	    }
NameNode接收到renewLease请求后，调用FSNameSystem.renewLease()并最终调用LeaseManager.renewLease()完成续约。


