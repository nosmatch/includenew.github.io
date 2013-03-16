---
layout: post
title: "HDFS源码学习（8）——Backup Mode"
date: 2012-10-18 21:51
comments: true
categories: HDFS
---

##元数据的持久化
HDFS通过主要通过FSImage和FSEditLog来完成文件元数据的持久化。对文件系统的任何修改，NameNode都会通过Editlog记录下来，持久化到本地。同时整个系统的命名空间，所有的文件元信息均保存在FSImage中，包括block->File的映射，文件的属性等等。NameNode启动时会从本地磁盘加载FSImage和FSEditLog，并将EditLog中的日志信息合并到FSImage中进行之持久化（该合并过程称为一个检查点：checkpoint），并构建文件系统的元信息。

但是持久化的数据中不包括block<->datanode的映射信息，该信息由每个datanode向NameNode发起blockReport()请求时报告其所拥有的block信息。

###Editlog记录修改日志
EditLog的持久化文件是一个二进制文件，大体结构如下：

![EditLogFile](/images/hdfs/EditLogFile.png)

EditLog文件开始是一个日志版本号，0.19版本的hdfs中该version为-18。
随后是每一条操作的事务日志，每条日志的起始为一个操作类型位，随后是该操作的详细信息，不同的操作类型所带的详细信息也不同。加载EditLog是根据layoutVersion和edit_op位采取不同的方式解析后面的详细信息。
EditLog能够记录如下17中操作：

![EditLogOp](/images/hdfs/EditLogOp.png)

EditLog为每种操作都提供了相应的log方法，当系统中发生文件修改时，会调用相应的log方法记录日志


##元数据加载与恢复
NameNode启动过程中最终会通过FSImage.loadFSImage()来从fsimage目录中加载最新的fsimage镜像和editslog，并合并构建命名空间。
代码结构如下：
	
	  boolean loadFSImage() throws IOException {
	    // 根据checkpointtime查找最新的fsimage
	    long latestNameCheckpointTime = Long.MIN_VALUE;
	    long latestEditsCheckpointTime = Long.MIN_VALUE;
	    StorageDirectory latestNameSD = null;
	    StorageDirectory latestEditsSD = null;
	    boolean needToSave = false;
	    isUpgradeFinalized = true;
	    Collection<String> imageDirs = new ArrayList<String>();
	    Collection<String> editsDirs = new ArrayList<String>();
	    for (Iterator<StorageDirectory> it = dirIterator(); it.hasNext();) {
	      StorageDirectory sd = it.next();
	      if (!sd.getVersionFile().exists()) {
	        needToSave |= true;
	        continue; // some of them might have just been formatted
	      }
	      boolean imageExists = false, editsExists = false;
	      if (sd.getStorageDirType().isOfType(NameNodeDirType.IMAGE)) {
	        imageExists = getImageFile(sd, NameNodeFile.IMAGE).exists();
	        imageDirs.add(sd.getRoot().getCanonicalPath());
	      }
	      if (sd.getStorageDirType().isOfType(NameNodeDirType.EDITS)) {
	        editsExists = getImageFile(sd, NameNodeFile.EDITS).exists();
	        editsDirs.add(sd.getRoot().getCanonicalPath());
	      }
	      
	      checkpointTime = readCheckpointTime(sd);
	      if ((checkpointTime != Long.MIN_VALUE) && 
	          ((checkpointTime != latestNameCheckpointTime) || 
	           (checkpointTime != latestEditsCheckpointTime))) {
	        // Force saving of new image if checkpoint time
	        // is not same in all of the storage directories.
	        needToSave |= true;
	      }
	      if (sd.getStorageDirType().isOfType(NameNodeDirType.IMAGE) && 
	         (latestNameCheckpointTime < checkpointTime) && imageExists) {
	        latestNameCheckpointTime = checkpointTime;
	        latestNameSD = sd;
	      }
	      if (sd.getStorageDirType().isOfType(NameNodeDirType.EDITS) && 
	           (latestEditsCheckpointTime < checkpointTime) && editsExists) {
	        latestEditsCheckpointTime = checkpointTime;
	        latestEditsSD = sd;
	      }
	      if (checkpointTime <= 0L)
	        needToSave |= true;
	      // set finalized flag
	      isUpgradeFinalized = isUpgradeFinalized && !sd.getPreviousDir().exists();
	    }
	
	    // 确保至少有一个fsimage和一个edits目录
	    if (latestNameSD == null)
	      throw new IOException("Image file is not found in " + imageDirs);
	    if (latestEditsSD == null)
	      throw new IOException("Edits file is not found in " + editsDirs);
	
	    // 确保获得的fsimage和edits是同一个检查点
	    if (latestNameCheckpointTime > latestEditsCheckpointTime
	        && latestNameSD != latestEditsSD
	        && latestNameSD.getStorageDirType() == NameNodeDirType.IMAGE
	        && latestEditsSD.getStorageDirType() == NameNodeDirType.EDITS) {
	      // This is a rare failure when NN has image-only and edits-only
	      // storage directories, and fails right after saving images,
	      // in some of the storage directories, but before purging edits.
	      // See -NOTE- in saveNamespace().
	      LOG.error("This is a rare failure scenario!!!");
	      LOG.error("Image checkpoint time " + latestNameCheckpointTime +
	          " > edits checkpoint time " + latestEditsCheckpointTime);
	      LOG.error("Name-node will treat the image as the latest state of " +
	          "the namespace. Old edits will be discarded.");
	    } else if (latestNameCheckpointTime != latestEditsCheckpointTime)
	      throw new IOException("Inconsitent storage detected, " +
	          "image and edits checkpoint times do not match. " +
	          "image checkpoint time = " + latestNameCheckpointTime +
	          "edits checkpoint time = " + latestEditsCheckpointTime);
	    
	    // 如果上次检查点中断了，则恢复该检查点
	    needToSave |= recoverInterruptedCheckpoint(latestNameSD, latestEditsSD);
	
	    long startTime = FSNamesystem.now();
	    long imageSize = getImageFile(latestNameSD, NameNodeFile.IMAGE).length();
	
	    //
	    // 加载fsimage文件
	    //
	    latestNameSD.read();
	    needToSave |= loadFSImage(getImageFile(latestNameSD, NameNodeFile.IMAGE));
	    LOG.info("Image file of size " + imageSize + " loaded in " 
	        + (FSNamesystem.now() - startTime)/1000 + " seconds.");
	    
	    // 加载最新的edits并作用于fsimage上
	    if (latestNameCheckpointTime > latestEditsCheckpointTime)
	      // the image is already current, discard edits
	      needToSave |= true;
	    else // latestNameCheckpointTime == latestEditsCheckpointTime
	      needToSave |= (loadFSEdits(latestEditsSD) > 0);
	    
	    return needToSave;
	  }
	  
其中FSImage.loadFSImage(File curFile) 代码如下：
	
	  boolean loadFSImage(File curFile) throws IOException {
	    assert this.getLayoutVersion() < 0 : "Negative layout version is expected.";
	    assert curFile != null : "curFile is null";
	
	    FSNamesystem fsNamesys = FSNamesystem.getFSNamesystem();
	    FSDirectory fsDir = fsNamesys.dir;
	
	    //
	    // Load in bits
	    //
	    boolean needToSave = true;
	    DataInputStream in = new DataInputStream(new BufferedInputStream(
	                              new FileInputStream(curFile)));
	    try {
	      /*
	       * Note: Remove any checks for version earlier than 
	       * Storage.LAST_UPGRADABLE_LAYOUT_VERSION since we should never get 
	       * to here with older images.
	       */
	      
	      /*
	       * TODO we need to change format of the image file
	       * it should not contain version and namespace fields
	       */
	      // 读取imageversion
	      int imgVersion = in.readInt();
	      // 读取namespaceid
	      this.namespaceID = in.readInt();
	
	      // 读取镜像中的文件数
	      long numFiles;
	      if (imgVersion <= -16) {
	        numFiles = in.readLong();
	      } else {
	        numFiles = in.readInt();
	      }
	
	      this.layoutVersion = imgVersion;
	      // 读取镜像时间戳
	      if (imgVersion <= -12) {
	        long genstamp = in.readLong();
	        fsNamesys.setGenerationStamp(genstamp); 
	      }
	
	      needToSave = (imgVersion != FSConstants.LAYOUT_VERSION);
	
	      // 读取每个文件的信息
	      short replication = FSNamesystem.getFSNamesystem().getDefaultReplication();
	
	      LOG.info("Number of files = " + numFiles);
	
	      byte[][] pathComponents;
	      byte[][] parentPath = {{}};
	      INodeDirectory parentINode = fsDir.rootDir;
	      for (long i = 0; i < numFiles; i++) {
	        long modificationTime = 0;
	        long atime = 0;
	        long blockSize = 0;
	        //读取文件名(path)
	        pathComponents = readPathComponents(in);
	        //读取副本数
	        replication = in.readShort();
	        //调整副本数，使其不超过系统的最大和最小副本数限制
	        replication = FSEditLog.adjustReplication(replication);
	        //读取文件修改时间
	        modificationTime = in.readLong();
	        if (imgVersion <= -17) {
	        //读取最近访问时间
	          atime = in.readLong();
	        }
	        if (imgVersion <= -8) {
	        //读取block块大小
	          blockSize = in.readLong();
	        }
	        //读取block数
	        int numBlocks = in.readInt();
	        //构建blocks
	        Block blocks[] = null;
	
	        // 老版本hdfs中，numBlocks=0表示目录，新版本中numBlocks=-1表示目录
	        if ((-9 <= imgVersion && numBlocks > 0) ||
	            (imgVersion < -9 && numBlocks >= 0)) {
	           //构建文件block信息
	          blocks = new Block[numBlocks];
	          for (int j = 0; j < numBlocks; j++) {
	            blocks[j] = new Block();
	            if (-14 < imgVersion) {
	              blocks[j].set(in.readLong(), in.readLong(), 
	                            Block.GRANDFATHER_GENERATION_STAMP);
	            } else {
	              // 读取block信息
	              blocks[j].readFields(in);
	            }
	          }
	        }
	        // 老版本inode中不维护blocksize，如果存在多个block，blocksize选取第一个block的大小，如果只有一个block，则选该block大小和默认大小中较大的，如果没有block，则选用默认大小
	        if (-8 <= imgVersion && blockSize == 0) {
	          if (numBlocks > 1) {
	            blockSize = blocks[0].getNumBytes();
	          } else {
	            long first = ((numBlocks == 1) ? blocks[0].getNumBytes(): 0);
	            blockSize = Math.max(fsNamesys.getDefaultBlockSize(), first);
	          }
	        }
	        
	        // 如果是目录（blocks=null），读取该目录的配额
	        long nsQuota = -1L;
	        if (imgVersion <= -16 && blocks == null) {
	          nsQuota = in.readLong();
	        }
	        long dsQuota = -1L;
	        if (imgVersion <= -18 && blocks == null) {
	          dsQuota = in.readLong();
	        }
	        
	        //获取权限信息
	        PermissionStatus permissions = fsNamesys.getUpgradePermission();
	        if (imgVersion <= -11) {
	          permissions = PermissionStatus.read(in);
	        }
	        
	        //如果该path为root，且设置了配额信息，则更新根目录的配额
	        if (isRoot(pathComponents)) { // it is the root
	          // update the root's attributes
	          if (nsQuota != -1 || dsQuota != -1) {
	            fsDir.rootDir.setQuota(nsQuota, dsQuota);
	          }
	          fsDir.rootDir.setModificationTime(modificationTime);
	          fsDir.rootDir.setPermissionStatus(permissions);
	          continue;
	        }
	        //如果该inode的parent与当前路径不一至，获取新的parentPaht
	        if(!isParent(pathComponents, parentPath)) {
	          parentINode = null;
	          parentPath = getParent(pathComponents);
	        }
	        // 将该inode添加到inode树中
	        parentINode = fsDir.addToParent(pathComponents, parentINode, permissions,
	                                        blocks, replication, modificationTime, 
	                                        atime, nsQuota, dsQuota, blockSize);
	      }
	      
	      // 加载datanode信息，imgVersion<-12的版本事实上啥都不做，datanode信息已经不保存在fsimage中
	      this.loadDatanodes(imgVersion, in);
	
	      // 加载正在构建中的文件
	      this.loadFilesUnderConstruction(imgVersion, in, fsNamesys);
	      
	    } finally {
	      in.close();
	    }
	    
	    return needToSave;
	  }
	  
其核心就是加载并解析fsimage文件，构建命名空间，fsimage文件的结构参见《NameNode中主要数据结构》