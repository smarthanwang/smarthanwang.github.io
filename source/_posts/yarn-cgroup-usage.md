---
title: Yarn 使用 Cgroup 实现任务资源限制
tags:
  - Yarn
  - Cgroup
  - 资源限制
categories:
  - Hadoop-YARN
date: 2019-10-20
---


## 概述

Linux CGroup 全称是 Linux Control Group，是 Linux 内核提供的一个用来限制进程资源使用的功能，支持如 CPU, 内存，磁盘 IO 等资源的使用限制。用户可以使用 CGroup 对单个进程或者一组进程进行精细化的资源限制，具体使用方式可以查看参考文档。

目前， Yarn NodeManager 能够使用 CGroup 来限制所有 containers 的资源使用，主要是 CPU 资源。如果不用 CGroup， 在 NM 端很难实现对 container 的 CPU 使用进行限制。默认状态下， container 的 CPU 使用是没有限制的，container 申请了 1 vcore ，实际上能够使用所有的 CPU 资源。所以如果 NM 上分配了一个 vcore 申请较少实际上 CPU 使用极高的任务，常常会导致节点上运行的所有的任务都延时。

NM 运行时，可以通过 ContainersMonitor 线程监控 container 内存和 CPU 使用。对于 container 内存使用， 一旦发现其超出申请内存大小，就会立即发起 kill container 命令，回收 container 的资源。ContainersMonitor 虽然也支持 CPU 使用监控，但是 CPU 资源不像内存资源，其使用量的峰值是基本上可以确定的，在所有机器或者系统上都基本一致。 CPU 受限于 CPU 硬件性能， 同一个任务在不同的机器上的 CPU 使用率可能差异巨大，所以不能发现 container CPU 使用超过申请大小就 kill container 。同时，由于 container 都是由子进程的方式启动的， NM 也是很难通过直接控制 container 运行和暂停来调整其 CPU 使用率。 因此，在没有 CGroup 功能的情况下， NM 是很难直接限制 container 的 CPU 使用的。

所以接下来我们主要介绍 Yarn 如何启用 CGroup 来限制 containers CPU 资源占用。

## 启用

NM 启用 CGroup 功能主要需要在 `yarn-site.xml` 里设置以下配置：

### 1. 启用 LCE ：

在 Nodemanager 中， CGroup 功能集成在 LinuxContainerExecutor 中，所以要使用 CGroup 功能，**必须**设置 container-executor 为 LinuxContainerExecutor. 同时需要配置 NM 的 Unix Group，这个是可执行的二进制文件 container-executor 用来做安全验证的，需要与 container-executor.cfg 里面配置的一致。 详细配置可参见 [Yarn ContainerExecutor 配置与使用](yarn-container-executour)。

```xml
    <property>
      <name>yarn.nodemanager.container-executor.class</name>
      <value>org.apache.hadoop.yarn.server.nodemanager.LinuxContainerExecutor<value>
    </property>
    <property>
      <name>yarn.nodemanager.linux-container-executor.group</name>
      <value>hadoop<value>
    </property>
```

### 2. 启用 CGroup ：

LinuxContainerExecutor 并不会强制开启 CGroup 功能， 如果想要开启CGroup 功能，**必须**设置 resource-handler-class 为 CGroupsLCEResourceHandler.

```xml
    <property>
      <name>yarn.nodemanager.linux-container-executor.resources-handler.class</name>
      <value>org.apache.hadoop.yarn.server.nodemanager.util.CgroupsLCEResourcesHandler<value>
    </property>
```

### 3. 配置 Yarn CGroup 目录：

NM 通过 `yarn.nodemanager.linux-container-executor.cgroups.hierarchy` 配置所有 Yarn Containers 进程放置的 CGroup 目录。

如果系统的 CGroup 未挂载和配置，可以在系统上手动挂载和配置和启用 CGroup 功能，也可以通过设置
`yarn.nodemanager.linux-container-executor.cgroups.mount` 为 true，同时设置 CGroup 挂载路径 `yarn.nodemanager.linux-container-executor.cgroups.mount-path` 来实现 NM 自动挂载 CGroup (不建议这样用，问题挺多)。

如果系统的 CGroup 已经挂载且配置完成，而且 Yarn 用户有 CGroup cpu 子目录的写入权限，NM 会在 cpu 目录下创建 `hadoop-yarn` 目录 ，如果该目录已经存在，保证 yarn 用户有写入权限即可。


```xml
    <property>
      <name>yarn.nodemanager.linux-container-executor.cgroups.hierarchy</name>
      <value>/hadoop-yarn<value>
    </property>
    <property>
      <name>yarn.nodemanager.linux-container-executor.cgroups.mount</name>
      <value>false<value>
    </property>
    <property>
      <name>yarn.nodemanager.linux-container-executor.cgroups.mount-path</name>
      <value>/sys/fs/cgroup<value>
    </property>
```


### 4. CPU 资源限制：

NM 主要使用两个参数来限制 containers CPU 资源使用。

首先，使用 `yarn.nodemanager.resource.percentage-physical-cpu-limit` 来设置所有 containers 的总的 CPU 使用率占用总的 CPU 资源的百分比。比如设置为 60，则所有的 containers 的 CPU 使用总和在任何情况下都不会超过机器总体 CPU 资源的 60 %。

然后，使用 `yarn.nodemanager.linux-container-executor.cgroups.strict-resource-usage` 设置是否对 container 的 CPU 使用进行严格限制。如果设置为 true ，即便 NM 的 CPU 资源比较空闲， containers CPU 使用率也不能超过限制，这种配置下，可以严格限制 CPU 使用，保证每个 container 只能使用自己分配到的 CPU 资源。但是如果设置为 false ，container 可以在 NM 有空闲 CPU 资源时，超额使用 CPU，这种模式下，可以保证 NM 总体 CPU 使用率比较高，提升集群的计算性能和吞吐量，所以建议使用非严格的限制方式（实际通过 CGroup 的 cpu share 功能实现）。不论这个值怎么设置，所有 containers 总的 CPU 使用率都不会超过 cpu-limit 设置的值。

NM 会按照机器总的 `CPU num* limit-percent` 来计算 NM 总体可用的实际 CPU 资源，然后根据 NM 配置的 Vcore 数量来计算每个 Vcore 对应的实际 CPU 资源，再乘以 container 申请的 Vcore 数量计算 container 的实际可用的 CPU 资源。这里需要注意的是，在计算总体可用的 CPU 核数时，NM 默认使用的实际的物理核数，而一个物理核通常会对应多个逻辑核（单核多线程），而且我们默认的 CPU 核数通常都是逻辑核，所以我们需要设置 `yarn.nodemanager.resource.count-logical-processors-as-cores` 为 true 来指定使用逻辑核来计算 CPU 资源。

```xml
    <property>
      <name>yarn.nodemanager.resource.percentage-physical-cpu-limit</name>
      <value>80<value>
    </property>
    <property>
      <name>yarn.nodemanager.linux-container-executor.cgroups.strict-resource-usage</name>
      <value>false<value>
    </property>    
    <property>
      <name>yarn.nodemanager.resource.count-logical-processors-as-cores</name>
      <value>true<value>
    </property> 
```
## 注意事项
Linux 内核版本 `3.10.0-327.el7.x86_64` 上 Yarn 启用 CGroup 功能后，会触发内核 BUG，导致内核卡死，重启，NM 挂掉，所有运行的任务失败。所以如果需要启用 CGroup 功能，绝对不能使用`3.10.0-327.el7.x86_64` 版本内核。亲测升级内核版本可解决该问题。

参见 [一次重现内核 Bug 的经历](https://runitao.github.io/a-process-to-reproduce-kernel-crash.html)
## 参考文档
1. [Linux资源管理之cgroups简介](https://tech.meituan.com/2015/03/31/cgroups.html)    
2. [DOCKER基础技术：LINUX CGROUP](https://coolshell.cn/articles/17049.html)   
3. [Using CGroups with YARN](https://hadoop.apache.org/docs/r3.2.0/hadoop-yarn/hadoop-yarn-site/NodeManagerCgroups.html)
4. [一次重现内核 Bug 的经历](https://runitao.github.io/a-process-to-reproduce-kernel-crash.html)
