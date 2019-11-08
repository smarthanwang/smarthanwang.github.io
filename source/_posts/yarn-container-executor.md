---
title: Yarn ContainerExecutor 配置与使用
tags:
  - Yarn
  - ContainerExecutor
  - LinuxContainerExecutor
categories:
  - Hadoop-YARN
date: 2019-10-12
---

在 Yarn 的架构中，将集群中的计算资源，主要是内存和 CPU ，封装抽象出了 Container 的概念， 类似于 `container_001 <memory:2048, vCores:1>`。 Container 由 ResourceManager 负责调度与分配，由资源所在的 NodeManager 负责启动与管理。

Container 所封装的计算资源是由集群中的 NodeManager 提供的，所以 Container 的启动，运行，监控， 管理也需要对应的 NodeManager 来执行。 ContainerExecutor 正是 NodeManager 用来启动与管理 Container 的工具。

Yarn 3.0 版本中，在 Linux 系统环境下，ContainerExecutor 有两种实现：
- DefaultContainerExecutor:   简称 DCE ,如其名，是默认的 ContainerExecutor 实现。 如果用户未指定 ContainerExecutor 的具体实现，NM 就会使用它。 DCE 直接使用 bash 来启动 container 进程，所有 container 都使用 NM 进程用户 (yarn) 启动。


- LinuxContainerExecutor:   简称 LCE，相比于 DCE ，它能提供更多有用的功能，如用户权限隔离，支持使用提交任务用户来启动 container；支持使用 cgroup 进行资源限制； 支持运行 docker container (合并了 2.x 版本中的 DockerContainerExecutor)。 LCE 使用可执行的二进制文件 container-executor 来启动 container 进程，container 的用户根据配置可以统一使用默认用户，也可以使用提交任务的用户（需要提前在 NM 上添加所有支持的用户）。

在默认的情况下，如果我们的集群是 non-secure ，而且没有什么特殊需求时，使用 DCE 就足够了，因为 DCE 配置和使用都很简单。但是当我们的集群要求安全性，想要支持 container 用户权限隔离，或者严格限制 container 的资源使用，再或者支持 docker container，我们就需要使用 LCE。


## 配置

NM 具体使用哪种 ContainerExecutor 的实现由 yarn-site.xml 里的 `yarn.nodemanager.container-executor.class` 属性来配置。

### DefaultContainerExecutor
DCE 的配置比较简单，只需要在 yarn-site.xml 指定 `yarn.nodemanager.container-executor.class` 属性为 `org.apache.hadoop.yarn.server.nodemanager.DefaultContainerExecutor` 即可，不需要额外的配置。

```xml
    <property>
      <name>yarn.nodemanager.container-executor.class</name>
      <value>org.apache.hadoop.yarn.server.nodemanager.DefaultContainerExecutor</value>
    </property>
```

### LinuxContainerExecutor
LCE 配置相对更加复杂一些， 具体如下：

**1. yarn-site.xml 配置**

```xml

  <!-- NM 的 Unix 用户组，需要跟 container-executor.cfg 里面的配置一致，主要用来验证是否有安全访问 container-executor 二进制的权限 -->
  <property>
    <name>yarn.nodemanager.linux-container-executor.group</name>
    <value>hadoop</value>
  </property>

  <!-- 是否限制 container 的启动用户，true：container 使用统一的用户启动 false: container 使用任务用户启动 -->
  <property>
    <name>yarn.nodemanager.linux-container-executor.nonsecure-mode.limit-users</name>
    <value>true</value>
  </property>

  <!-- 限制 container 启动用户时，统一使用的用户，如果不设置，默认为 nobody -->
  <property>
    <name>yarn.nodemanager.linux-container-executor.nonsecure-mode.local-user</name>
    <value>yarn</value>
  </property>

```

**2. container-executor.cfg 配置**

container-executor.cfg 是 container-executor 二进制程序的配置文件，会在其启动时读取校验。
具体的属性如下：
- yarn.nodemanager.linux-container-executor.group: NM 的 Unix 用户组， 需要与 yarn-site.xml 里一致。
- allowed.system.users: 允许使用的系统用户，多个用户使用 ',' 分隔，可以不设置，即允许所有用户。
- banned.users: 禁止使用的用户，多个用户使用 ',' 分隔。
- min.user.id: 允许使用的用户的 uid 最小值，防止有其他超级用户。

配置样例如下：

```properties
yarn.nodemanager.linux-container-executor.group=hadoop
allowed.system.users=hdfs,yarn,mapred,
banned.users=root,bin
min.user.id=100
```

## 启用

DCE 的启用不需要额外的操作，配置完成后直接重启 NM 即可。LCE 的启用则还需要一系列的准备工作。

**1. 设置 container-executor 权限**    

container-executor 二进制的 owner 必须是 root，属组必须与 NM 属组相同 (hadoop)，同时，它的权限必须设置成为 6050，以赋予它 setuid 的权限，来实现使用不同的用户来启动 container。

```sh
chown root:hadoop /usr/lib/hadoop-yarn/bin/container-executor 
chmod 6050 /usr/lib/hadoop-yarn/bin/container-executor
```

**2. 设置 container-executor.cfg 权限**    

container-executor.cfg 二进制的 owner 必须是 root，属组必须与 NM 属组相同 (hadoop)，同时，它的权限必须设置成为 0400， 以保证它是只读不可写的
```sh
chown root:hadoop /etc/hadoop/conf/container-executor.cfg
chmode 0400 /etc/hadoop/conf/container-executor.cfg
```

**3. 设置 yarn-local 和 yarn-log dirs 权限**    

所有配置的 yarn-local 和 yarn-log 目录属主必须是 yarn:hadoop ，这个一般集群搭建时已经修改好了，不需要再处理。


**4. 任务用户管理**    

如果 `yarn.nodemanager.linux-container-executor.nonsecure-mode.limit-users` 设置的为 false，即使用提交任务用户来运行 container，则需要在集群所有的 NM 节点上将需要的用户通过 useradd 命令，逐个添加到机器上，否则任务运行时会因为找不到指定用户而失败。当集群规模比较大，用户很多时，添加用户还是比较繁琐的，建议统一使用 yarn 用户来启动 container。参考上面的 LCE 配置即可。


当上述的操作都完成后，重启 NM ， LCE 就会被启用。


## 注意事项

container-executor 和 container-executor.cfg 文件的权限一定要按照要求修改，否则 NM 会启动失败。如果改了权限后，NM 还是报权限问题，还需要将文件所在的目录用户组也改成 root:hadoop，权限改成 755。


## 参考文档

- [Configuring YARN container executor](https://www.ibm.com/support/knowledgecenter/en/SSPT3X_4.2.0/com.ibm.swg.im.infosphere.biginsights.install.doc/doc/inst_adv_yarn_config.html)
- [YARN Secure Containers](https://hadoop.apache.org/docs/r3.2.0/hadoop-yarn/hadoop-yarn-site/SecureContainer.html)
- [理解和配置LinuxContainerExecutor
](https://makeling.github.io/bigdata/dcb921f7.html)
