---
layout: post
title: "HDFS源码学习（11）——SecondaryNameNode"
date: 2012-10-18 21:55
comments: true
categories: HDFS
---

##概述
SecondaryNameNode在HDFS中的主要作用是帮助master NameNode周期性执行checkpoint操作。

NameNode将运行过程中对文件的修改记录保存在EditLog中。当NameNode重新启动时会从FSImage中加载命名空间镜像，并将EditLog中的内容合并到FSImage中，将合并后的FSImage写入到磁盘，同时清空EditLog，共后续使用。但如果NameNode长时间不重启，随时间增长，EditLog将会越来越大（每次文件操作都要记录），大量占用NameNode磁盘空间，且会导致下一次重启花费大量时间在合并Editlog上。

为了解决这个问题，SecondaryNameNode会定期从NameNode下载最新的FSImage和EditLog，合并editLog日志到FSImage，将合并后的FSImage上传到NameNode，并清空NameNode上的Editlog，将EditLog日志大小控制在一定限度下。

##代码解析
SecondaryNameNode代码结构如下：

![SecondaryNameNode](/images/hdfs/SecondaryNameNode.png)

SecondaryNameNode本身就是实现了Runnable接口，即一个可执行线程。

checkpointImage表示当前SecondNameNode上的FsImage镜像，该类CheckpointStorage扩展自FSImage。

其中有两个主要的可配置属性：

1. checkpointPeriod: 两次检查点的间隔时间，可通过fs.checkpoint.period配置
2. checkpointSize: EditLog文件的最大值，当EditLog超过这个最大值时会强制之行checkpoint，可通过fs.checkpoint.size配置，默认是64M



###run()
run方法代码如下：

	  public void run() {
	
	    //
	    // Poll the Namenode (once every 5 minutes) to find the size of the
	    // pending edit log.
	    //
	    long period = 5 * 60;              // 5 minutes
	    long lastCheckpointTime = 0;
	    if (checkpointPeriod < period) {
	      period = checkpointPeriod;
	    }
	
	    while (shouldRun) {
	      try {
	        Thread.sleep(1000 * period);
	      } catch (InterruptedException ie) {
	        // do nothing
	      }
	      if (!shouldRun) {
	        break;
	      }
	      try {
	        long now = System.currentTimeMillis();
	
	        long size = namenode.getEditLogSize();
	        if (size >= checkpointSize || 
	            now >= lastCheckpointTime + 1000 * checkpointPeriod) {
	          doCheckpoint();
	          lastCheckpointTime = now;
	        }
	      } catch (IOException e) {
	        LOG.error("Exception in doCheckpoint: ");
	        LOG.error(StringUtils.stringifyException(e));
	        e.printStackTrace();
	      } catch (Throwable e) {
	        LOG.error("Throwable Exception in doCheckpoint: ");
	        LOG.error(StringUtils.stringifyException(e));
	        e.printStackTrace();
	        Runtime.getRuntime().exit(-1);
	      }
	    }
	  }
	  
其核心就是周期性（默认每个5分钟)调用doCheckpoint().
### doCheckpoint()

	  void doCheckpoint() throws IOException {
	
	    // 准备合并所需的空间
	    startCheckpoint();
	
	    // 通知NameNode将修改信息记录到新的editlog中，并获取一个用于上传合并后的fsimage的token
	    CheckpointSignature sig = (CheckpointSignature)namenode.rollEditLog();
	
	    // error simulation code for junit test
	    if (ErrorSimulator.getErrorSimulation(0)) {
	      throw new IOException("Simulating error0 " +
	                            "after creating edits.new");
	    }
		//从NameNode获取fsimage和editslog
	    downloadCheckpointFiles(sig);   // Fetch fsimage and edits
	    //合并editlog到fsimage
	    doMerge(sig);                   // Do the merge
	  
	    //上传合并后的fsimage到NameNode
	    putFSImage(sig);
	
	    // error simulation code for junit test
	    if (ErrorSimulator.getErrorSimulation(1)) {
	      throw new IOException("Simulating error1 " +
	                            "after uploading new image to NameNode");
	    }
		// 通知NameNode使用该fsimage作为最新的镜像
	    namenode.rollFsImage();
	    checkpointImage.endCheckpoint();
	
	    LOG.warn("Checkpoint done. New Image Size: " 
	              + checkpointImage.getFsImageName().length());
	  }
	  
#### 1. startCheckpoint()
该方法主要用于准备合并所需的磁盘空间，代码如下：

	  private void startCheckpoint() throws IOException {
	    checkpointImage.unlockAll();
	    // 关闭当前Editlog
	    checkpointImage.getEditLog().close();
	    // 检查当前checkpoints目录，如果不存在则创建一个新目录，如果目录中存在异常，则尝试恢复该目录
	    checkpointImage.recoverCreate(checkpointDirs, checkpointEditsDirs);
	    // 为新的checkpoint准备目录空间，将当前的目录空间更名为lastcheckpoint.™p，新建一个current目录 
	    checkpointImage.startCheckpoint();
	  }


#### 2. namenode.rollEditLog();
namenode.rollEditLog()实际通过NameNodeProtocol调用NameNode.rollEditLog()方法，并最终调用FSImage.rollEditLog()，该方法主要完成：

1. 调用FSEditLog.rollEditLog()关闭当前editLog，新建一个editLog：edits.new
2. 返回一个CheckpointSignature做为上传合并后镜像的token

##### 2.1. FSEditLog.rollEditLog()
该方法主要完成：

1. 关闭当前editlog， 打开一个新的editlog： edit.new
2. 返回editlog的最新更新时间

代码结构如下

	  synchronized void rollEditLog() throws IOException {
	    //检查edit.new是否已经存在，如果存在，检查是否所有目录都存在，如果是则认为edits.new已经建好了，直接返回
	    //
	    if (existsNew()) {
	      for (Iterator<StorageDirectory> it = 
	               fsimage.dirIterator(NameNodeDirType.EDITS); it.hasNext();) {
	        File editsNew = getEditNewFile(it.next());
	     if (!editsNew.exists()) { 
	          throw new IOException("Inconsistent existance of edits.new " +
	                                editsNew);
	        }
	      }
	      return; // nothing to do, edits.new exists!
	    }
	
		//关闭当前的editLog
	    close();                     // close existing edit log
	
	    //
	    // 新建一个editLog： edits.new
	    //
	    for (Iterator<StorageDirectory> it = 
	           fsimage.dirIterator(NameNodeDirType.EDITS); it.hasNext();) {
	      StorageDirectory sd = it.next();
	      try {
	        EditLogFileOutputStream eStream = 
	             new EditLogFileOutputStream(getEditNewFile(sd));
	        eStream.create();
	        editStreams.add(eStream);
	      } catch (IOException e) {
	        // remove stream and this storage directory from list
	        processIOError(sd);
	       it.remove();
	      }
	    }
	  }

#### 3. downloadCheckpointFiles()
该方法用于从NameNode下载FSImage和FSEditLog，代码结构如下：

	  private void downloadCheckpointFiles(CheckpointSignature sig
	                                      ) throws IOException {
	    
	    checkpointImage.cTime = sig.cTime;
	    checkpointImage.checkpointTime = sig.checkpointTime;
	
	    // 获取fsimage
	    String fileid = "getimage=1";
	    File[] srcNames = checkpointImage.getImageFiles();
	    assert srcNames.length > 0 : "No checkpoint targets.";
	    TransferFsImage.getFileClient(fsName, fileid, srcNames);
	    LOG.info("Downloaded file " + srcNames[0].getName() + " size " +
	             srcNames[0].length() + " bytes.");
	
	    // 获取editlog
	    fileid = "getedit=1";
	    srcNames = checkpointImage.getEditsFiles();
	    assert srcNames.length > 0 : "No checkpoint targets.";
	    TransferFsImage.getFileClient(fsName, fileid, srcNames);
	    LOG.info("Downloaded file " + srcNames[0].getName() + " size " +
	        srcNames[0].length() + " bytes.");
	
		// 标示checkpoint所需文件已经准备完成
	    checkpointImage.checkpointUploadDone();
	  }
  
#### 4. doMerge()
doMerge主要完成editLog与fsimage的合并，实际调用的checkpointImage.doMerge(sig);

    private void doMerge(CheckpointSignature sig) throws IOException {
      getEditLog().open();
      StorageDirectory sdName = null;
      StorageDirectory sdEdits = null;
      Iterator<StorageDirectory> it = null;
      it = dirIterator(NameNodeDirType.IMAGE);
      if (it.hasNext())
        sdName = it.next();
      it = dirIterator(NameNodeDirType.EDITS);
      if (it.hasNext())
        sdEdits = it.next();
      if ((sdName == null) || (sdEdits == null))
        throw new IOException("Could not locate checkpoint directories");
      // 加载fsimage
      loadFSImage(FSImage.getImageFile(sdName, NameNodeFile.IMAGE));
      // 加载editlog，并合并到fsimage

      loadFSEdits(sdEdits);
      // 校验新fsimage的一致性，主要包括版本，更新时间，namespaceId
      sig.validateStorageInfo(this);
      // 将fsImage存储到本地，并创建新edits
      saveFSImage();
    }
    
该方法与NameNode启动时类似:

1. 加载fsiamge
2. 加载editslog，并作用到fsimage中
3. 校验合并后的fsimage
4. 将新fsimage存储到本地，并新建空的edit是目录



#### 5. putFSImage(sig);
该方法比较简单，主要通过TransferFsImage工具类将合并后的fsimage上传到NameNode

	  private void putFSImage(CheckpointSignature sig) throws IOException {
	    String fileid = "putimage=1&port=" + infoPort +
	      "&machine=" +
	      InetAddress.getLocalHost().getHostAddress() +
	      "&token=" + sig.toString();
	    LOG.info("Posted URL " + fsName + fileid);
	    TransferFsImage.getFileClient(fsName, fileid, (File[])null);
	  }

#### 6. namenode.rollFsImage();
上传完新fsiamge之后，SecondaryNameNode通过namenode.rollFsImage()通知NameNode使用新的fsimage.ckpt作为最新镜像，并清空editslog。该请求最终由NameNode上的FsImage.rollFsImage()处理，代码如下：

	 void rollFSImage() throws IOException {
	    if (ckptState != CheckpointStates.UPLOAD_DONE) {
	      throw new IOException("Cannot roll fsImage before rolling edits log.");
	    }
	    //
	    // 校验fsimage.ckpt和edits.new是否存在于所有目录
	    if (!editLog.existsNew()) {
	      throw new IOException("New Edits file does not exist");
	    }
	    for (Iterator<StorageDirectory> it = 
	                       dirIterator(NameNodeDirType.IMAGE); it.hasNext();) {
	      StorageDirectory sd = it.next();
	      File ckpt = getImageFile(sd, NameNodeFile.IMAGE_NEW);
	      if (!ckpt.exists()) {
	        throw new IOException("Checkpoint file " + ckpt +
	                              " does not exist");
	      }
	    }
	    //删除旧的edits，并将edits.new重命名为edits
	    editLog.purgeEditLog(); // renamed edits.new to edits
	
	    //
	    // 重命名fsimage
	    for (Iterator<StorageDirectory> it = 
	                       dirIterator(NameNodeDirType.IMAGE); it.hasNext();) {
	      StorageDirectory sd = it.next();
	      File ckpt = getImageFile(sd, NameNodeFile.IMAGE_NEW);
	      File curFile = getImageFile(sd, NameNodeFile.IMAGE);
	      // renameTo fails on Windows if the destination file 
	      // already exists.
	      if (!ckpt.renameTo(curFile)) {
	        curFile.delete();
	        if (!ckpt.renameTo(curFile)) {
	          // Close edit stream, if this directory is also used for edits
	          if (sd.getStorageDirType().isOfType(NameNodeDirType.EDITS))
	            editLog.processIOError(sd);
	        // add storage to the removed list
	          removedStorageDirs.add(sd);
	          it.remove();
	        }
	      }
	    }
	
	    //
	    // 更新所有目录的fstime
	    //
	    this.layoutVersion = FSConstants.LAYOUT_VERSION;
	    this.checkpointTime = FSNamesystem.now();
	    for (Iterator<StorageDirectory> it = 
	                           dirIterator(); it.hasNext();) {
	      StorageDirectory sd = it.next();
	      // delete old edits if sd is the image only the directory
	      if (!sd.getStorageDirType().isOfType(NameNodeDirType.EDITS)) {
	        File editsFile = getImageFile(sd, NameNodeFile.EDITS);
	        editsFile.delete();
	      }
	      // delete old fsimage if sd is the edits only the directory
	      if (!sd.getStorageDirType().isOfType(NameNodeDirType.IMAGE)) {
	        File imageFile = getImageFile(sd, NameNodeFile.IMAGE);
	        imageFile.delete();
	      }
	      try {
	        sd.write();
	      } catch (IOException e) {
	        LOG.error("Cannot write file " + sd.getRoot(), e);
	        // Close edit stream, if this directory is also used for edits
	        if (sd.getStorageDirType().isOfType(NameNodeDirType.EDITS))
	          editLog.processIOError(sd);
	      //add storage to the removed list
	        removedStorageDirs.add(sd);
	        it.remove();
	      }
	    }
	    ckptState = FSImage.CheckpointStates.START;
	  }