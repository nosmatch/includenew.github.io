---
layout: post
title: "HDFS源码学习(2)——NameNode初始化"
date: 2012-10-18 21:35
comments: true
categories: HDFS
---

##main()
	
	  public static void main(String argv[]) throws Exception {
	    try {
	      StringUtils.startupShutdownMessage(NameNode.class, argv, LOG);
	      //创建nameNode
	      NameNode namenode = createNameNode(argv, null);
	      if (namenode != null)
	        namenode.join();
	    } catch (Throwable e) {
	      LOG.error(StringUtils.stringifyException(e));
	      System.exit(-1);
	    }
	  }

###createNameNode()
	
	  public static NameNode createNameNode(String argv[], 
	                                 Configuration conf) throws IOException {
	    if (conf == null)
	      conf = new Configuration();
	      //从命令行参数中提取启动配置项数据
	    StartupOption startOpt = parseArguments(argv);
	    if (startOpt == null) {
	      printUsage();
	      return null;
	    }
	    //设置启动参数
	    setStartupOption(conf, startOpt);
	
	    switch (startOpt) {
	      case FORMAT:
	        boolean aborted = format(conf, true);
	        System.exit(aborted ? 1 : 0);
	      case FINALIZE:
	        aborted = finalize(conf, true);
	        System.exit(aborted ? 1 : 0);
	      default:
	    }
	
		//新建NameNode
	    NameNode namenode = new NameNode(conf);
	    return namenode;
	  }

### NameNode()

	  public NameNode(Configuration conf) throws IOException {
	    super(conf);
	    try {
	    //初始化
	      initialize(getConf());
	    } catch (IOException e) {
	      this.stop();
	      throw e;
	    }
	  }
	  
### initialize()
	
	  private void initialize(Configuration conf) throws IOException {
	    InetSocketAddress socAddr = NameNode.getAddress(conf);
	    int handlerCount = conf.getInt("dfs.namenode.handler.count", 10);
	    // 关键->创建一个RPC Server
	    this.server = RPC.getServer(this, socAddr.getHostName(), socAddr.getPort(),
	                                handlerCount, false, conf);
	
	    // The rpc-server port can be ephemeral... ensure we have the correct info
	    this.serverAddress = this.server.getListenerAddress(); 
	    FileSystem.setDefaultUri(conf, getUri(serverAddress));
	    LOG.info("Namenode up at: " + this.serverAddress);
	
	    myMetrics = new NameNodeMetrics(conf, this);
	
		//关键->创建一个FSNameSystem
	    this.namesystem = new FSNamesystem(this, conf);
	    //启动HTTP Server
	    startHttpServer(conf);
	    //启动RPC Server
	    this.server.start();  //start RPC server   
	
	    startTrashEmptier(conf);
	  }
	  
## FSNameSystem()

	  FSNamesystem(NameNode nn, Configuration conf) throws IOException {
	    try {
	      //初始化FSNameSystem
	      initialize(nn, conf);
	      userPasswordInformation = new UserPasswordInformation(conf);
	      extendAccessControlList = new ExtendAccessControlList(conf);
	    } catch(IOException e) {
	      LOG.error(getClass().getSimpleName() + " initialization failed.", e);
	      close();
	      throw e;
	    }
	  }
### FSNameSystem.initialize()

	  private void initialize(NameNode nn, Configuration conf) throws IOException {
	    this.systemStart = now();
	    this.fsLock = new ReentrantReadWriteLock(); // non-fair locking
	    setConfigurationParameters(conf);
	
	    this.nameNodeAddress = nn.getNameNodeAddress();
	    this.registerMBean(conf); // register the MBean for the FSNamesystemStutus

	    //创建FSDirectory
	    this.dir = new FSDirectory(this, conf);
	    StartupOption startOpt = NameNode.getStartupOption(conf);

	    //加载FSImage
	    this.dir.loadFSImage(getNamespaceDirs(conf),
	                         getNamespaceEditsDirs(conf), startOpt);
	    long timeTakenToLoadFSImage = now() - systemStart;
	    LOG.info("Finished loading FSImage in " + timeTakenToLoadFSImage + " msecs");
	    NameNode.getNameNodeMetrics().fsImageLoadTime.set(
	                              (int) timeTakenToLoadFSImage);
	    this.safeMode = new SafeModeInfo(conf);
	    setBlockTotal();
	    //创建PendingReplicationBlocks
	    pendingReplications = new PendingReplicationBlocks(
	                            conf.getInt("dfs.replication.pending.timeout.sec", 
	                                        -1) * 1000L);
		//创建心跳检查线程	                                        
	    this.hbthread = new Daemon(new HeartbeatMonitor());
	    //创建租约管理线程
	    this.lmthread = new Daemon(leaseManager.new Monitor());
	    //创建副本管理线程
	    this.replthread = new Daemon(new ReplicationMonitor());
	    hbthread.start();
	    lmthread.start();
	    replthread.start();
	
	    // 副本超额block管理线程
	    this.overreplthread = new Daemon(new OverReplicationMonitor());
	    overreplthread.start();
	    
	    this.hostsReader = new HostsFileReader(conf.get("dfs.hosts",""),
	                                           conf.get("dfs.hosts.exclude",""));
		//创建退役节点管理线程
	    this.dnthread = new Daemon(new DecommissionManager(this).new Monitor(
	        conf.getInt("dfs.namenode.decommission.interval", 30),
	        conf.getInt("dfs.namenode.decommission.nodes.per.interval", 5)));
	    dnthread.start();
	
	    this.dnsToSwitchMapping = ReflectionUtils.newInstance(
	        conf.getClass("topology.node.switch.mapping.impl", ScriptBasedMapping.class,
	            DNSToSwitchMapping.class), conf);
	    
	    /* If the dns to swith mapping supports cache, resolve network 
	     * locations of those hosts in the include list, 
	     * and store the mapping in the cache; so future calls to resolve
	     * will be fast.
	     */
	    if (dnsToSwitchMapping instanceof CachedDNSToSwitchMapping) {
	      dnsToSwitchMapping.resolve(new ArrayList<String>(hostsReader.getHosts()));
	    }
	    //创建副本定位器用于定位副本存放位置
	    this.replicator = BlockPlacementPolicy.getInstance(
	        conf,
	        this,
	        this.clusterMap,
	        this.hostsReader,
	        this.dnsToSwitchMapping,
	        this);
	  }
	  
## FSDirectory(this, conf)
新建FSDirecotry

	 FSDirectory(FSNamesystem ns, Configuration conf) {
	 	//创建一个FSImage，并实例化构建FSDirectory
	    this(new FSImage(), ns, conf);
	    fsImage.setCheckpointDirectories(FSImage.getCheckpointDirs(conf, null),
	                                FSImage.getCheckpointEditsDirs(conf, null));
	  }
### this(new FSImage(), ns, conf);
	 
	FSDirectory(FSImage fsImage, FSNamesystem ns, Configuration conf) {
	    this.bLock = new ReentrantReadWriteLock(); // non-fair
	    this.cond = bLock.writeLock().newCondition();
	    //创建根目录
	    rootDir = new INodeDirectoryWithQuota(INodeDirectory.ROOT_NAME,
	        ns.createFsOwnerPermissions(new FsPermission((short)0755)),
	        Integer.MAX_VALUE, -1);
	    this.fsImage = fsImage;
	    namesystem = ns;
	    initialize(conf);
	  }
### FSDirectory.initialize()

	  private void initialize(Configuration conf) {
	    MetricsContext metricsContext = MetricsUtil.getContext("dfs");
	    directoryMetrics = MetricsUtil.createRecord(metricsContext, "FSDirectory");
	    directoryMetrics.setTag("sessionId", conf.get("session.id"));
	  }
	  
### FSDirectory.loadFSImage()

	  void loadFSImage(Collection<File> dataDirs,
	                   Collection<File> editsDirs,
	                   StartupOption startOpt) throws IOException {
	    // format before starting up if requested
	    if (startOpt == StartupOption.FORMAT) {
	      fsImage.setStorageDirectories(dataDirs, editsDirs);
	      fsImage.format();
	      startOpt = StartupOption.REGULAR;
	    }
	    try {
	      //从datadir和editdirs加载FSImage
	      if (fsImage.recoverTransitionRead(dataDirs, editsDirs, startOpt)) {
	        fsImage.saveNamespace(true);
	      }
	      //初始化Editlog
	      FSEditLog editLog = fsImage.getEditLog();
	      assert editLog != null : "editLog must be initialized";
	      if (!editLog.isOpen())
	        editLog.open();
	      fsImage.setCheckpointDirectories(null, null);
	    } catch(IOException e) {
	      fsImage.close();
	      throw e;
	    }
	    writeLock();
	    try {
	      this.ready = true;
	      cond.signalAll();
	    } finally {
	      writeUnlock();
	    }
	  }
### FSImage.recoverTransitionRead（）
	  boolean recoverTransitionRead(Collection<File> dataDirs,
	                             Collection<File> editsDirs,
	                                StartupOption startOpt
	                                ) throws IOException {
	    assert startOpt != StartupOption.FORMAT : 
	      "NameNode formatting should be performed before reading the image";
	    
	    // none of the data dirs exist
	    if (dataDirs.size() == 0 || editsDirs.size() == 0)  
	      throw new IOException(
	        "All specified directories are not accessible or do not exist.");
	    
	    if(startOpt == StartupOption.IMPORT 
	        && (checkpointDirs == null || checkpointDirs.isEmpty()))
	      throw new IOException("Cannot import image from a checkpoint. "
	                          + "\"fs.checkpoint.dir\" is not set." );
	
	    if(startOpt == StartupOption.IMPORT 
	        && (checkpointEditsDirs == null || checkpointEditsDirs.isEmpty()))
	      throw new IOException("Cannot import image from a checkpoint. "
	                          + "\"fs.checkpoint.edits.dir\" is not set." );
	    
	    setStorageDirectories(dataDirs, editsDirs);
	    // 1.检查所有目录的状态和一致性
	    Map<StorageDirectory, StorageState> dataDirStates = 
	             new HashMap<StorageDirectory, StorageState>();
	    boolean isFormatted = false;
	    for (Iterator<StorageDirectory> it = 
	                      dirIterator(); it.hasNext();) {
	      StorageDirectory sd = it.next();
	      StorageState curState;
	      try {
	        curState = sd.analyzeStorage(startOpt);
	        // sd is locked but not opened
	        switch(curState) {
	        case NON_EXISTENT:
	          // name-node fails if any of the configured storage dirs are missing
	          throw new InconsistentFSStateException(sd.getRoot(),
	                                                 "storage directory does not exist or is not accessible.");
	        case NOT_FORMATTED:
	          break;
	        case NORMAL:
	          break;
	        default:  // recovery is possible
	          sd.doRecover(curState);      
	        }
	        if (curState != StorageState.NOT_FORMATTED 
	            && startOpt != StartupOption.ROLLBACK) {
	          sd.read(); // read and verify consistency with other directories
	          isFormatted = true;
	        }
	        if (startOpt == StartupOption.IMPORT && isFormatted)
	          // import of a checkpoint is allowed only into empty image directories
	          throw new IOException("Cannot import image from a checkpoint. " 
	              + " NameNode already contains an image in " + sd.getRoot());
	      } catch (IOException ioe) {
	        sd.unlock();
	        throw ioe;
	      }
	      dataDirStates.put(sd,curState);
	    }
	    
	    if (!isFormatted && startOpt != StartupOption.ROLLBACK 
	                     && startOpt != StartupOption.IMPORT)
	      throw new IOException("NameNode is not formatted.");
	    if (layoutVersion < LAST_PRE_UPGRADE_LAYOUT_VERSION) {
	      checkVersionUpgradable(layoutVersion);
	    }
	    if (startOpt != StartupOption.UPGRADE
	          && layoutVersion < LAST_PRE_UPGRADE_LAYOUT_VERSION
	          && layoutVersion != FSConstants.LAYOUT_VERSION)
	        throw new IOException(
	                          "\nFile system image contains an old layout version " + layoutVersion
	                          + ".\nAn upgrade to version " + FSConstants.LAYOUT_VERSION
	                          + " is required.\nPlease restart NameNode with -upgrade option.");
	    // check whether distributed upgrade is reguired and/or should be continued
	    verifyDistributedUpgradeProgress(startOpt);
	
	    // 2. Format unformatted dirs.
	    this.checkpointTime = 0L;
	    for (Iterator<StorageDirectory> it = 
	                     dirIterator(); it.hasNext();) {
	      StorageDirectory sd = it.next();
	      StorageState curState = dataDirStates.get(sd);
	      switch(curState) {
	      case NON_EXISTENT:
	        assert false : StorageState.NON_EXISTENT + " state cannot be here";
	      case NOT_FORMATTED:
	        LOG.info("Storage directory " + sd.getRoot() + " is not formatted.");
	        LOG.info("Formatting ...");
	        sd.clearDirectory(); // create empty currrent dir
	        break;
	      default:
	        break;
	      }
	    }