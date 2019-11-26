---
title: HDFS 问题总结
tags:
  - HDFS
  - trouble shooting
  - 运维
categories:
  - Hadoop-HDFS
---


在日常维护 HDFS 集群以及为用户提供技术支持时，我们经常会遇到各种 HDFS 相关异常问题，对于一些常见的问题，我们可能知道如何去解决，所以处理起来会比较快。但是当遇到一些不常见的问题时，我们可能没办法根据异常信息直接找到解决方法，可能需要查找一些资料，或者深入源码层面查找问题所在，处理起来比较费时费力，很可能对集群的业务造成影响。

因此，将实际生产环境中 HDFS 集群出现的问题进行总结是有必要的。在本文中，我会把我在集群运维过程遇到的一些有代表性的问题进行总结，包括异常信息，产生原因以及解决办法，以便为后续的运维工作提供一个快速参照，也希望能给大家带来一定的参考。


## 文件未正常关闭

### 异常信息

主要异常信息有以下几种：
- Cannot obtain block length
出现在读取未正常关闭文件时

- Mismatch in length of source
出现在 distcp 文件后，对比源文件与复制文件长度时

### 产生原因

正常 HDFS 客户端向 HDFS 写入数据完成时，会向 NameNode 发起 `completeFile` 的操作，NN 端会 `commit & complete` 文件的最后一个 block，然后更新文件 INode 信息，主要是加入 last block 长度， 再 `finalize INodeFile` 关闭文件，释放文件租约。此时文件对于客户端来说，就是正常关闭的完整文件。

但是如果写入数据时发生了异常，比如客户端崩溃或者 namenode 异常等，completeFile 未成功发起或者完成，就会导致文件最后一个 block 可能未正常关闭，INodeFile 记录的文件长度与实际写入的长度不匹配，租约也无法正常释放。此时文件对于客户端来讲，是未关闭的异常状态，也就是 `openforwrite` 的状态，此时去读文件时，就会出现上述的异常状态。


### 解决方法

这种情况的解决方案是先找出未正常关闭的文件，然后通过手动释放租约，关闭文件。


```bash
1. 使用 fsck 找出未正常关闭文件
hdfs fsck path -openforwrite 

对于 federation 集群需要指定 defaultFS
hdfs fsck -Dfs.defaultFS=hdfs://nameservice:8020/ path -openforwrite 


2. 手动释放租约
hdfs debug recoverLease -path openfile -retries 3
```

## 安全集群 (kerberos) 无法访问非安全集群

### 异常信息

HDFS 支持使用 kerberos 认证作为用户权限校验工具，开启的集群的 security 模式。 安全模式的配置大致为

```xml
  <property>
    <name>hadoop.security.authentication</name>
    <value>kerberos</value>
  </property>
  <property>
    <name>hadoop.security.authorization</name>
    <value>true</value>
  </property>

```

当 security 集群 客户端通过指定 namenode 地址访问另一个 non security 的的集群时会有以下报错：

```java
hadoop fs -ls hdfs:master001.mars.sjs.ted:8020/user 

19/05/31 17:14:34 INFO configfile.ConfigFileHasClientPlugin: Get the user info successfully, login user : hdfs
ls: Failed on local exception: java.io.IOException: Server asks us to fall back to SIMPLE auth, but this client is configured to only allow secure connections.; Host Details : local host is: "cloud_dev.gd.ted/10.135.43.214"; destination host is: "master001.mars.sjs.ted":8020; 

```

### 产生原因

当客户端开启了安全认证模式，而请求的 service 服务使用 SIMPLE 模式，未开启安全认证的时候 IPC Client 在与 server 端建立连接时，会因为验证方式不一致而抛出此异常。

### 解决方式

解决方法是在客户端添加配置，允许 security 的客户端连接 non security & simple 模式的服务端建立连接。

```xml
  <property>   
    <name>ipc.client.fallback-to-simple-auth-allowed</name>
    <value>true</value>
  </property>  
```

## datanode 域名无法解析

### 异常信息

在客户端读写 HDFS 数据的时候，有时会出现以下的异常
```java
java.nio.channels.UnresolvedAddressException
	at sun.nio.ch.Net.checkAddress(Net.java:101)
	at sun.nio.ch.SocketChannelImpl.connect(SocketChannelImpl.java:622)
	at org.apache.hadoop.net.SocketIOWithTimeout.connect(SocketIOWithTimeout.java:192)
	at org.apache.hadoop.net.NetUtils.connect(NetUtils.java:530)

```

### 产生原因

当客户端向 HDFS 发起读写请求时，NameNode 会根据目标文件的 block 存储位置返回一组相应的 datanode 节点给客户端，默认状况下，客户端根据返回的 datanode 的域名和端口信息，发起 socket 连接，进行数据读写。

在集群维护过程中，我们发现有些小型集群，为了省事，未在 dns 服务器上配置可解析的域名，而是所有 datanode 节点本地配置域名，同时在所有 datanode 节点上的 /etc/hosts 里面添加映射来完成域名解析。这种情况下，客户端在读写集群时，根据 datanode 域名去解析 ip 时，就会有这种无法解析域名 UnresolvedAddressException 的报错。

### 解决方法
这种状况的解决方式有两种：  

首先第一种就是跟集群上的 datanode 节点一样，将所有的节点域名映射添加到 /etc/hosts 里，这种方式比较笨，而且难以维护，必须要跟这集群节点信息的更新而更新，所以不建议这样解决。

第二种方式是将客户端禁止默认使用域名进行连接，这样客户端就会默认使用节点 ip 进行连接，也就跳过了域名解析的问题。客户端只需要在配置 `dfs.client.use.datanode.hostname=false` 即可，且不需要频繁更改，所以推荐使用这种解决方式。


## datanode 注册失败，机架信息异常

### 异常信息

当集群新增 datanode 节点，启动时无法向 NN 成功注册，异常信息如下

NN 端 异常：
```java
org.apache.hadoop.net.NetworkTopology: Error: can't add leaf node /J/HD-1100/10.140.49.36:50010 at depth 3 to topology:
org.apache.hadoop.net.NetworkTopology$InvalidTopologyException: Failed to add /J/HD-1164/10.140.83.70:50010: You cannot have a rack and a non-rack node at the same level of the network topology.
	at org.apache.hadoop.net.NetworkTopology.add(NetworkTopology.java:415)
	at org.apache.hadoop.hdfs.server.blockmanagement.DatanodeManager.registerDatanode(DatanodeManager.java:980)
    ...
```

DN 端 异常：

```java
org.apache.hadoop.ipc.RemoteException(org.apache.hadoop.net.NetworkTopology$InvalidTopologyException): Failed to add /J/HD-1164/10.140.83.70:50010: You cannot have a rack and a non-rack node at the same level of the network topology.
        at org.apache.hadoop.net.NetworkTopology.add(NetworkTopology.java:415)
        at org.apache.hadoop.hdfs.server.blockmanagement.DatanodeManager.registerDatanode(DatanodeManager.java:980)
        at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.registerDatanode(FSNamesystem.java:5218)
        ...
```

### 产生原因

datanode 节点的机架信息代表着 datanode 在集群的网络拓扑结构中所在的位置， NN 通过机架信息来计算节点之间的距离，进而通过选用距离最近的节点，提升集群读写任务的本地性，达到读写加速的效果。

机架信息通常使用 /d/r/n 这种3层机构来标识， d 为 data center ，表示数据中心， r 为rack， 表示机架信息， n 为 node，标识节点信息。

通过追踪 datanode 注册模块的代码，我们发现该异常产生原因是 Namenode 通过配置 `net.topology.table.file.name` 指定的节点机架信息配置文件来维护所有节点的机架信息。如果配置文件中不存在某个节点的机架信息，该节点的机架信息就为默认的 `/default-rack` ， NN 使用机架信息中的 '/' 数量来标识机架信息的深度。**最为重要的，一个集群只允许有一种深度的机架信息，所以新增的 datanode 的机架深度必须跟已有的机架信息深度完全一致。** 

所以，当新增节点的机架信息与原来节点机架信息深度不一致时，就会出现此异常。具体可能是新增节点未配置机架信息，使用的 `/default-rack` 与原来节点不一致，也可能是配置时，配置的机架深度与原来节点不一致。

### 解决方式

检查 `net.topology.table.file.name` 指定的机架信息配置文件中，新节点的机架信息是否配置了，如果未配置，按照之前的规则添加正确的机架信息，如果配置了，检查是否正确，如果不正确，修正即可。如果是正确的，可能要考虑之前节点的机架信息是否配置上了，很有可能之前的机架信息没有配置正确，用的都是 `/default-rack`

