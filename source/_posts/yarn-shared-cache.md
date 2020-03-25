---
title: Yarn Shared Cache 功能简介
tags:
  - Yarn
  - Yarn Shared Cache
  - 依
categories:
  - Hadoop-YARN
date: 2020-01-20
---


Yarn Shared Cache （Yarn 共享缓存服务）是社区 Hadoop 2.9 版本之后上新增的功能，参见[YARN-1492](https://issues.apache.org/jira/browse/YARN-1492)，它的目的是提供了一种安全且可扩展的方式向 HDFS 上传和管理应用程序依赖资源，降低 Yarn application 因为依赖资源的上传以及本地化带来的时间消耗。

通过使用 Yarn Shared Cache， 对于相同的依赖资源，YARN application 可以直接使用其他 application 上传的资源或者该 application 的先前运行时自己上传的资源，而无需每次都重新上传以及本地化相同的资源文件，从而节省网络资源并大大减少YARN应用程序启动时间。

本文主要简单介绍 Yarn Shared Cache 的架构和使用方式，具体官方文档可参见 [SharedCache](https://hadoop.apache.org/docs/r2.9.2/hadoop-yarn/hadoop-yarn-site/SharedCache.html)

## 架构

Yarn Shared Cache 主要由4个部分组成:

- **Shared Cache Client: 共享缓存客户端**  
主要提供与 SCM 交互的接口，用户或者开发者可以使用 shared cache client 计算资源校验值(checksum)，使用校验值向 SCM 获取共享缓存中该资源的存储路径。
 

- **Shared Cache HDFS Directory：共享缓存 HDFS 存储目录**  
共享存储资源全部都存储在指定的 HDFS 目录中，共享缓存目录通过HDFS权限进行保护，并且全局只读，只允许信任用户去写。这个目录只有共享缓存管理过程和在 NM 的 uploader 在上传资源时会修改。资源通过MD5 校验值被分到各个子目录。

    ```
    /sharedcache/a/8/9/a896857d078/foo.jar
    /sharedcache/5/0/f/50f11b09f87/bar.jar
    /sharedcache/a/6/7/a678cb1aa8f/job.jar
    ```

- **Shared Cache Manager (SCM)：共享缓存管理器**   
  SCM是一个单独的服务进程，它可以运行在任何节点上。这样，管理员可以在不影响其他组件的同时，启动，停止SCM。 

  它负责处理从客户端以及 uploader 发来的请求，以及管理所有共享的资源。同时负责维护所有存储在 HDFS 目录中的缓存数据以及这些数据的元信息。它由两个主要的组件组成：后端存储服务和缓存清理服务。  

  后端存储服务负责维护和持久化 shared cache 的元数据。包括在缓存中的资源，资源最后一次被使用时间，当前使用资源的应用等信息。目前是使用内存存储，并且在重启后会重建。  

  清理服务负责确保不再使用的资源会被从缓存中移除。它周期性的扫描资源，清理过期的和没有应用使用的资源。 
     


-  **Shared Cache Uploader and Localization：缓存上传以及本地化**

    Shared Cache uploader 是一个运行在  NM  上的服务，用来上传资源到共享缓存。它负责确认资源校验值，根据校验值动态确定资源对应的HDFS目录，然后上传资源到该目录中，然后通知 SCM 该资源已经被添加到缓存。

    Shared Cache uploader 是异步的，不会阻塞 yarn应用的启动。

    一旦资源上传到共享缓存中，Yarn 使用 NM 本地化机制即可使得资源可获得。基于 NM 本地化资源的机制，由于 Shared Cache 中的缓存文件的 HDFS 地址和时间戳一直保持不变，当 NM 本地化过一次该资源后，后续再使用该资源时直接使用本地缓存的资源即可，也不会每次都重新本地化资源。因此，后续依赖于该资源的任务都不需要再次上传和本地化该资源，降低了任务启动时间。


## 部署

### 初始化设置

管理员需要通过以下几步来初始化设置 Shared Cache
```bash
1.创建HDFS目录，默认是 /sharecache 可配置
    
    yarn.sharedcache.root-dir=/sharecache 
    
2.设置共享缓存目录0755

    hadoop fs -chmod 755 /sharecache

3.确认此目录的属主是 SCM 和 NM 的启动用户(yarn) 

    hadoop fs -chown yarn:yarn /sharecache

4.在yarn-site.xml里面开启 sharedcache 功能

    yarn.sharedcache.enabled=true
    
5.启动共享缓存管理器

    yarn --daemon start sharedcachemanager
```

### 所有配置选项

Yarn Shared Cache 所需要的所有配置大致如下：

Name        |        Description        | 	Default value
---|---|---
yarn.sharedcache.enabled	|Whether the shared cache is enabled	|false
yarn.sharedcache.root-dir	|The root directory for the shared cache	|/sharedcache
yarn.sharedcache.nested-level	|The level of nested directories before getting to the checksum directories. It must be non-negative.	|3
yarn.sharedcache.store.class	|The implementation to be used for the SCM store	|org.apache.hadoop.yarn.server.sharedcachemanager.store.InMemorySCMStore
yarn.sharedcache.app-checker.class	|The implementation to be used for the SCM app-checker	|org.apache.hadoop.yarn.server.sharedcachemanager.RemoteAppChecker
yarn.sharedcache.store.in-memory.staleness-period-mins	|A resource in the in-memory store is considered stale if the time since the last reference exceeds the staleness period. This value is specified in minutes.	|10080
yarn.sharedcache.store.in-memory.initial-delay-mins	|Initial delay before the in-memory store runs its first check to remove dead initial applications. Specified in minutes.	|10
yarn.sharedcache.store.in-memory.check-period-mins	|The frequency at which the in-memory store checks to remove dead initial applications. Specified in minutes.	|720
yarn.sharedcache.admin.thread-count|	The number of threads used to handle SCM admin interface (1 by default)	|1
yarn.sharedcache.cleaner.period-mins|	The frequency at which a cleaner task runs. Specified in minutes.|	1440
yarn.sharedcache.cleaner.initial-delay-mins|	Initial delay before the first cleaner task is scheduled. Specified in minutes.	|10
yarn.sharedcache.cleaner.resource-sleep-ms	|The time to sleep between processing each shared cache resource. Specified in milliseconds.	|0
yarn.sharedcache.uploader.server.thread-count	|The number of threads used to handle shared cache manager requests from the node manager (50 by default)	|50
yarn.sharedcache.client-server.address	|The address of the client interface in the SCM (shared cache manager)	|0.0.0.0:8045
yarn.sharedcache.uploader.server.address	|The address of the node manager interface in the SCM (shared cache manager)	|0.0.0.0:8046
yarn.sharedcache.admin.address	|The address of the admin interface in the SCM (shared cache manager)	|0.0.0.0:8047
yarn.sharedcache.webapp.address|	The address of the web application in the SCM (shared cache manager)	|0.0.0.0:8788
yarn.sharedcache.client-server.thread-count	|The number of threads used to handle shared cache manager requests from clients (50 by default)	|50
yarn.sharedcache.checksum.algo.impl	|The algorithm used to compute checksums of files (SHA-256 by default)	|org.apache.hadoop.yarn.sharedcache.ChecksumSHA256Impl
yarn.sharedcache.nm.uploader.replication.factor	|The replication factor for the node manager uploader for the shared cache (10 by default)	|10
yarn.sharedcache.nm.uploader.thread-count	|The number of threads used to upload files from a node manager instance (20 by default)	|20
yarn.sharedcache.keytab    |	keytab for SCM	    | required for security cluster
yarn.sharedcache.principal |	principal for SCM	| required for security cluster
 

## MR 支持 YARN Shared Cache

社区对 MR 任务进行了 YARN Shared Cache 功能支持，通过客户端配置来开启 MR 任务资源直接使用 Yarn Shared Cache进行托管。MR 客户端会优先使用 YARN Shared Cache 托管依赖资源，如果 SCM 服务不可用时，就会回退到原来的上传依赖资源到任务的提交目录的模式。 

使用主要使用的配置如下。

```xml
  <property>
    <description>Whether the shared cache is enabled</description>
    <name>yarn.sharedcache.enabled</name>
    <value>true</value>
  </property>
 
  <property>
    <description>The address of the client interface in the SCM (shared cache manager)</description>
    <name>yarn.sharedcache.client-server.address</name>
    <value>shardcache-server-host:8045</value>
  </property>
 
  <property>
    <name>yarn.sharedcache.keytab</name>
    <value>/etc/hadoop/conf/yarn.keytab</value>
  </property>
  <property>
    <name>yarn.sharedcache.principal</name>
    <value>yarn/_HOST@HADOOP.COM</value>
  </property>


    <property>
        <name>mapreduce.job.sharedcache.mode</name>
        <value>libjars,files,archives</value>
        <description>
           A comma delimited list of resource categories to submit to the
           shared cache. The valid categories are: jobjar, libjars, files,
           archives. If "disabled" is specified then the job submission code
           will not use the shared cache.
        </description>
    </property>
```




## 参考文档

- [SharedCache](https://hadoop.apache.org/docs/r2.9.2/hadoop-yarn/hadoop-yarn-site/SharedCache.html)

- [MR Support for YARN Shared Cache](https://hadoop.apache.org/docs/r3.1.2/hadoop-mapreduce-client/hadoop-mapreduce-client-core/SharedCacheSupport.html)


