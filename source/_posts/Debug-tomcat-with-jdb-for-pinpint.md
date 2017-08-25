title: 在tomcat中使用jdb调试
date: 2017-05-11 17:49:18
tags: [jdb,tomcat]
---

pinpoint-web跑起来时报错，报错信息如下：

<!--more-->
# pinpoint 报错

##  报错信息


```
Caused by: org.apache.hadoop.hbase.client.RetriesExhaustedException: Can't get the locations
	at org.apache.hadoop.hbase.client.RpcRetryingCallerWithReadReplicas.getRegionLocations(RpcRetryingCallerWithReadReplicas.java:319)
	at org.apache.hadoop.hbase.client.ScannerCallableWithReplicas.call(ScannerCallableWithReplicas.java:156)
	at org.apache.hadoop.hbase.client.ScannerCallableWithReplicas.call(ScannerCallableWithReplicas.java:60)
	at org.apache.hadoop.hbase.client.RpcRetryingCaller.callWithoutRetries(RpcRetryingCaller.java:200)
	at org.apache.hadoop.hbase.client.ClientScanner.call(ClientScanner.java:320)
	at org.apache.hadoop.hbase.client.ClientScanner.nextScanner(ClientScanner.java:295)
	at org.apache.hadoop.hbase.client.ClientScanner.initializeScannerInConstruction(ClientScanner.java:160)
	at org.apache.hadoop.hbase.client.ClientScanner.<init>(ClientScanner.java:155)
	at org.apache.hadoop.hbase.client.HTable.getScanner(HTable.java:867)
	at org.apache.hadoop.hbase.MetaTableAccessor.fullScan(MetaTableAccessor.java:602)
	at org.apache.hadoop.hbase.MetaTableAccessor.tableExists(MetaTableAccessor.java:366)
	at org.apache.hadoop.hbase.client.HBaseAdmin.tableExists(HBaseAdmin.java:410)
	at com.navercorp.pinpoint.common.hbase.HBaseAdminTemplate.tableExists(HBaseAdminTemplate.java:59)
	... 39 more
```


##  整理解决办法

翻看启动日志也没发现重要的信息. 翻看代码也没有看到为啥（应该说是没有仔细看），只知道是一个变量为null就抛异常了。无奈生产环境不能现场调试，只能想到tomcat 的远程调试jdwp。
尝试使用后发现有个局限性：tomcat只在项目初始化完成后才能使用jdwp，但这个是在项目启动的时候报错。
翻看tomcat 的catalina.sh文件时发现有个jdb，便打算试试这个。
使用jdb后发现有个地方在尝试从Zookeeper获取meta region location(org.apache.hadoop.hbase.client.ZooKeeperRegistry#getMetaRegionLocation)。并且看到有级别是trace的日志。
随即把日志级别改成trace之后重启，就发现了问题所在之处：zk上的有数据丢失。

```
org.apache.hadoop.hbase.client.RpcRetryingCallerWithReadReplicas

296	static RegionLocations getRegionLocations(boolean useCache, int replicaId,
297	               ClusterConnection cConnection, TableName tableName, byte[] row)
298	    throws RetriesExhaustedException, DoNotRetryIOException, InterruptedIOException {
299	
300	  RegionLocations rl;
301	  try {
302	    if (!useCache) {
303	      rl = cConnection.relocateRegion(tableName, row, replicaId);
304	    } else {
305	      rl = cConnection.locateRegion(tableName, row, useCache, true, replicaId);
306	    }
307	  } catch (DoNotRetryIOException e) {
308	    throw e;
309	  } catch (NeedUnmanagedConnectionException e) {
310	    throw new DoNotRetryIOException(e);
311	  } catch (RetriesExhaustedException e) {
312	    throw e;
313	  } catch (InterruptedIOException e) {
314	    throw e;
315	  } catch (IOException e) {
316	    throw new RetriesExhaustedException("Can't get the location", e);
317	  }
318	  if (rl == null) {
319	    throw new RetriesExhaustedException("Can't get the locations");
320	  }
321	
322	  return rl;
323	}
```

## 发现问题原因
日志级别改成trace后发现如下信息：

2017-05-11 17:04:15 [DEBUG](o.a.h.h.z.ZKUtil) hconnection-0x1d9358ee-0x35b5cfb488b0669, quorum=172.16.200.49:2181,172.16.200.50:2181,172.16.200.51:2181, baseZNode=/hbase/hbs-d3ntquwj Unable to get data of znode /hbase/hbs-d3ntquwj/meta-region-server because node does not exist (not an error)

##  确认问题原因

然后去zk上看，果然没有。。。。

```
[zk: 172.16.200.50(CONNECTED) 0] ls /hbase/hbs-d3ntquwj     
[acl, backup-masters, region-in-transition, draining, table, running, table-lock, master, balancer, namespace, hbaseid, online-snapshot, replication, recovering-regions, splitWAL, rs, flush-table-proc]
```


## 解决方法
  反正pinpoint上的数据不重要，请运维删掉重做。
  

#  使用JDB
  
## 常见的命令
1. stop  使程序执行到指定的地方
1. run   继续执行代码（直到碰到断点 ）
1. where  查看当前线程堆栈
1. step   向下执行代码（包含跳入方法）
1. step up   回到调用者（跳入一个方法后，不想看其执行细节时则跳出）
1. locals   查看当前的变量
1. threads  查看线程信息
1. thread  设置默认线程

## 使用命令的过程

### 启动tomcat jdb

sh catalina.sh debug

### 指定断点

stop at org.apache.hadoop.hbase.client.RpcRetryingCallerWithReadReplicas:302

### 继续运行代码 
run 

###  查看变量  
locals 

```
localhost-startStop-1[1] locals
Method arguments:
useCache = true
replicaId = 0
cConnection = instance of org.apache.hadoop.hbase.client.ConnectionManager$HConnectionImplementation(id=5127)
tableName = instance of org.apache.hadoop.hbase.TableName(id=5128)
row = instance of byte[8] (id=5129)
Local variables:
```

### 查看线程信息

threads
```
localhost-startStop-1[1] threads
Group system:
  (java.lang.ref.Reference$ReferenceHandler)0x16c                   Reference Handler                                                         cond. waiting
  (java.lang.ref.Finalizer$FinalizerThread)0x16b                    Finalizer                                                                 cond. waiting
  (java.lang.Thread)0x16a                                           Signal Dispatcher                                                         running
  (sun.misc.GC$Daemon)0x55e                                         GC Daemon                                                                 cond. waiting
  (java.lang.Thread)0xf25                                           process reaper                                                            cond. waiting
Group main:
  (java.lang.Thread)0x1                                             main                                                                      cond. waiting
  (org.apache.juli.AsyncFileHandler$LoggerThread)0x1f6              AsyncFileHandlerWriter-1023714065                                         cond. waiting
  (org.apache.tomcat.util.net.NioBlockingSelector$BlockPoller)0x6d2 NioBlockingSelector.BlockPoller-1                                         running
  (java.lang.Thread)0x706                                           Catalina-startStop-1                                                      cond. waiting
  (java.lang.Thread)0x70d                                           localhost-startStop-1                                                     running (at breakpoint)
  (org.apache.zookeeper.ClientCnxn$SendThread)0x1029                localhost-startStop-1-SendThread(172.16.200.50:2181)                      running
  (org.apache.zookeeper.ClientCnxn$EventThread)0x102b               localhost-startStop-1-EventThread                                         cond. waiting
  (java.lang.Thread)0x10c6                                          org.apache.hadoop.fs.FileSystem$Statistics$StatisticsDataReferenceCleaner cond. waiting
  (org.apache.zookeeper.ClientCnxn$SendThread)0x117b                localhost-startStop-1-SendThread(172.16.200.50:2181)                      running
  (org.apache.zookeeper.ClientCnxn$EventThread)0x117c               localhost-startStop-1-EventThread                                         cond. waiting
```

### 查看当前线程堆栈

where

```
localhost-startStop-1[1] where
  [1] org.apache.hadoop.hbase.client.RpcRetryingCallerWithReadReplicas.getRegionLocations (RpcRetryingCallerWithReadReplicas.java:302)
  [2] org.apache.hadoop.hbase.client.ScannerCallableWithReplicas.call (ScannerCallableWithReplicas.java:156)
  [3] org.apache.hadoop.hbase.client.ScannerCallableWithReplicas.call (ScannerCallableWithReplicas.java:60)
  [4] org.apache.hadoop.hbase.client.RpcRetryingCaller.callWithoutRetries (RpcRetryingCaller.java:200)
  [5] org.apache.hadoop.hbase.client.ClientScanner.call (ClientScanner.java:320)
  [6] org.apache.hadoop.hbase.client.ClientScanner.nextScanner (ClientScanner.java:295)
  [7] org.apache.hadoop.hbase.client.ClientScanner.initializeScannerInConstruction (ClientScanner.java:160)
  [8] org.apache.hadoop.hbase.client.ClientScanner.<init> (ClientScanner.java:155)
  [9] org.apache.hadoop.hbase.client.HTable.getScanner (HTable.java:867)
  [10] org.apache.hadoop.hbase.MetaTableAccessor.fullScan (MetaTableAccessor.java:602)
  [11] org.apache.hadoop.hbase.MetaTableAccessor.tableExists (MetaTableAccessor.java:366)
  [12] org.apache.hadoop.hbase.client.HBaseAdmin.tableExists (HBaseAdmin.java:410)
  [13] com.navercorp.pinpoint.common.hbase.HBaseAdminTemplate.tableExists (HBaseAdminTemplate.java:59)
  [14] com.navercorp.pinpoint.web.dao.hbase.HbaseTraceDaoFactory.getObject (HbaseTraceDaoFactory.java:65)
  [15] com.navercorp.pinpoint.web.dao.hbase.HbaseTraceDaoFactory.getObject (HbaseTraceDaoFactory.java:21)
  [16] org.springframework.beans.factory.support.FactoryBeanRegistrySupport.doGetObjectFromFactoryBean (FactoryBeanRegistrySupport.java:168)
  [17] org.springframework.beans.factory.support.FactoryBeanRegistrySupport.getObjectFromFactoryBean (FactoryBeanRegistrySupport.java:103)
  [18] org.springframework.beans.factory.support.AbstractBeanFactory.getObjectForBeanInstance (AbstractBeanFactory.java:1,525)
  [19] org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean (AbstractBeanFactory.java:251)
  [20] org.springframework.beans.factory.support.AbstractBeanFactory.getBean (AbstractBeanFactory.java:194)
  [21] org.springframework.beans.factory.support.DefaultListableBeanFactory.findAutowireCandidates (DefaultListableBeanFactory.java:1,120)
  [22] org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency (DefaultListableBeanFactory.java:1,044)
  [23] org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveDependency (DefaultListableBeanFactory.java:942)
  [24] org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor$AutowiredFieldElement.inject (AutowiredAnnotationBeanPostProcessor.java:533)
  [25] org.springframework.beans.factory.annotation.InjectionMetadata.inject (InjectionMetadata.java:88)
  [26] org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor.postProcessPropertyValues (AutowiredAnnotationBeanPostProcessor.java:331)
  [27] org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.populateBean (AbstractAutowireCapableBeanFactory.java:1,208)
  [28] org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean (AbstractAutowireCapableBeanFactory.java:537)
  [29] org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean (AbstractAutowireCapableBeanFactory.java:476)
  [30] org.springframework.beans.factory.support.AbstractBeanFactory$1.getObject (AbstractBeanFactory.java:303)
  [31] org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton (DefaultSingletonBeanRegistry.java:230)
  [32] org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean (AbstractBeanFactory.java:299)
  [33] org.springframework.beans.factory.support.AbstractBeanFactory.getBean (AbstractBeanFactory.java:194)
  [34] org.springframework.beans.factory.support.DefaultListableBeanFactory.preInstantiateSingletons (DefaultListableBeanFactory.java:755)
  [35] org.springframework.context.support.AbstractApplicationContext.finishBeanFactoryInitialization (AbstractApplicationContext.java:762)
  [36] org.springframework.context.support.AbstractApplicationContext.refresh (AbstractApplicationContext.java:480)
  [37] org.springframework.web.context.ContextLoader.configureAndRefreshWebApplicationContext (ContextLoader.java:434)
  [38] org.springframework.web.context.ContextLoader.initWebApplicationContext (ContextLoader.java:306)
  [39] org.springframework.web.context.ContextLoaderListener.contextInitialized (ContextLoaderListener.java:106)
  [40] org.apache.catalina.core.StandardContext.listenerStart (StandardContext.java:4,727)
  [41] org.apache.catalina.core.StandardContext.startInternal (StandardContext.java:5,189)
  [42] org.apache.catalina.util.LifecycleBase.start (LifecycleBase.java:150)
  [43] org.apache.catalina.core.ContainerBase.addChildInternal (ContainerBase.java:752)
  [44] org.apache.catalina.core.ContainerBase.addChild (ContainerBase.java:728)
  [45] org.apache.catalina.core.StandardHost.addChild (StandardHost.java:734)
  [46] org.apache.catalina.startup.HostConfig.deployDirectory (HostConfig.java:1,107)
  [47] org.apache.catalina.startup.HostConfig$DeployDirectory.run (HostConfig.java:1,841)
  [48] java.util.concurrent.Executors$RunnableAdapter.call (Executors.java:511)
  [49] java.util.concurrent.FutureTask.run (FutureTask.java:266)
  [50] java.util.concurrent.ThreadPoolExecutor.runWorker (ThreadPoolExecutor.java:1,142)
  [51] java.util.concurrent.ThreadPoolExecutor$Worker.run (ThreadPoolExecutor.java:617)
  [52] java.lang.Thread.run (Thread.java:745)
```
### 跳出

step  up

```
Step completed: "thread=localhost-startStop-1", org.apache.hadoop.hbase.client.ConnectionManager$HConnectionImplementation.locateRegion(), line=1,186 bci=0
```


