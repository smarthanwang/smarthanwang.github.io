---
title: YARN 在 security 集群中使用统一用户运行 Container
tags:
  - Yarn
  - Security
  - LinuxContainerExecutor
categories:
  - Hadoop-YARN
---

在我们的工作过程中，随着业务越来越多，Hadoop 集群规模也越来越大，同时对集群的安全性也越来越高，最开始的 simple 模式的验证方式已经无法满足需求了，因此，我们决定使用 kerberos 开启集群的安全验证: Run Hadoop in Secure Mode。

在实践中过程中，我们发现在 Secure Mode 时，NM 必须使用 LinuxContainerExecutor，如果使用 DefaultContainerExecutor 的话，MR 任务会在 reduce shuffle 阶段失败退出，参见[YARN-1432](https://issues.apache.org/jira/browse/YARN-1432)。同时，LCE 会使用任务提交用户 JobUser 来运行 Container，如果 NM 上不存在 JobUser，任务就会失败退出。 Hadoop 这样做的逻辑是为了实现用户权限隔离，保证用户任务和数据的安全，但是在实践中过程中，由于我们集群中用户过多，在 NM 上逐一添加用户过于繁琐，而且后续维护起来比较麻烦，因此，我们决定修改这块的代码逻辑，在 Secure Mode 下支持 NM 使用统一的 yarn 用户来执行任务。


## 