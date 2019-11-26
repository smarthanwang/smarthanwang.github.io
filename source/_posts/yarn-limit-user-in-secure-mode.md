---
title: YARN 在 security 集群中使用统一用户运行 Container
tags:
  - Yarn
  - Security
  - LinuxContainerExecutor
categories:
  - Hadoop-YARN
---

在我们的工作过程中，随着业务越来越多，Hadoop 集群规模也越来越大，同时对集群的安全性也越来越高，最原始的 SIMPLE 验证方式已经无法满足需求了，因此，我们决定使用 kerberos, 开启集群的安全验证: Run Hadoop in Secure Mode。

在实践中过程中，我们发现在 Secure Mode 时，NM 必须使用 LinuxContainerExecutor，如果使用 DefaultContainerExecutor 的话，MR 任务会在 reduce shuffle 阶段失败退出，参见[YARN-1432](https://issues.apache.org/jira/browse/YARN-1432)。同时，LCE 会使用任务提交用户 JobUser 来运行 Container，如果 NM 上不存在 JobUser，任务就会失败退出。 

其实简单想想就能理解 Hadoop 这样设计的逻辑是为了实现用户隔离，保证用户任务和数据的安全，但是在实践过程中，由于我们集群中用户过多，在 NM 上逐一添加用户过于繁琐，而且后续维护起来比较麻烦，因此，我们决定修改这块的代码逻辑，使 NM 在 Secure Mode 下支持使用统一的 yarn 用户来执行任务。


## 原始逻辑解析

为了实现我们的目的，先得了解 NM 这块代码的实现逻辑，所以我对这块的代码逻辑进行了梳理，发现安全用户限制的逻辑存在于 LinuxContainerExecutor 和 SecureIOUtils 两个类中。

### LinuxContainerExecutor 

LCE 在初始化，启动以及清理 container 之前，都会调用 `getRunAsUser(jobUser)` 方法获取该 container 实际运行时该使用的用户，再使用该用户去对 container 执行相应的操作。

如果集群未开启安全验证方式（使用 SIMPLE 验证方式： 如果配置 nonsecure 模式下限制用户时，getRunAsUser 返回统一的用户，默认是 nobody。如果配置不限制用户时，getRunAsUser 返回任务提交用户。

如果集群开启了安全验证方式 (使用 kerberos 验证方式)： getRunAsUser 返回任务提交用户。


```java

  /**
   * Determine if UserGroupInformation is using Kerberos to determine
   * user identities or is relying on simple authentication
   * 
   * @return true if UGI is working in a secure environment
   */
  public static boolean isSecurityEnabled() {
    return !isAuthenticationMethodEnabled(AuthenticationMethod.SIMPLE);
  }

  // nonsecure 模式下是否限制用户，默认 true
  boolean containerLimitUsers = conf.getBoolean(
    "yarn.nodemanager.linux-container-executor.nonsecure-mode.limit-users", true);
  // nonsecure 模式下默认使用的本地用户， 默认 nobody
  String nonsecureLocalUser = conf.get(
    "yarn.nodemanager.linux-container-executor.nonsecure-mode.local-user" ,"nobody");

  String getRunAsUser(String user) {
    if (UserGroupInformation.isSecurityEnabled() ||
        !containerLimitUsers) {
      return user;
    } else {
      return nonsecureLocalUser;
    }
  }
```

LCE 获取到实际用户后，会去调用 container-executor 二进制可执行文件来初始化和启动 container，container-executor 在启动时会执行 set_user 函数来设置 container 实际的执行用户，在这个函数中又会调用 check_user 函数，对该用户进行一系列的校验，主要包括：不是 root， UID 大于配置的 minUid，该用户本地实际存在以及不在禁用用户列表里。当校验不通过时，就会报错退出。所以，当 LCE 传递进来的指定用户在机器上不存在时，container 就会失败退出。


container-executor.c
```c
/**
 * Is the user a real user account?
 * Checks:
 *   1. Not root
 *   2. UID is above the minimum configured.
 *   3. Not in banned user list
 * Returns NULL on failure
 */
struct passwd* check_user(const char *user) {
  if (strcmp(user, "root") == 0) {
    fprintf(LOGFILE, "Running as root is not allowed\n");
    fflush(LOGFILE);
    return NULL;
  }
  char *min_uid_str = get_section_value(MIN_USERID_KEY, &executor_cfg);
  int min_uid = DEFAULT_MIN_USERID;
  if (min_uid_str != NULL) {
    char *end_ptr = NULL;
    min_uid = strtol(min_uid_str, &end_ptr, 10);
    if (min_uid_str == end_ptr || *end_ptr != '\0') {
      fprintf(LOGFILE, "Illegal value of %s for %s in configuration\n",
	      min_uid_str, MIN_USERID_KEY);
      fflush(LOGFILE);
      free(min_uid_str);
      return NULL;
    }
    free(min_uid_str);
  }
  struct passwd *user_info = get_user_info(user);
  if (NULL == user_info) {
    fprintf(LOGFILE, "User %s not found\n", user);
    fflush(LOGFILE);
    return NULL;
  }
  if (user_info->pw_uid < min_uid && !is_whitelisted(user)) {
    fprintf(LOGFILE, "Requested user %s is not whitelisted and has id %d,"
	    "which is below the minimum allowed %d\n", user, user_info->pw_uid, min_uid);
    fflush(LOGFILE);
    free(user_info);
    return NULL;
  }
  char **banned_users = get_section_values(BANNED_USERS_KEY, &executor_cfg);
  banned_users = banned_users == NULL ?
    (char**) DEFAULT_BANNED_USERS : banned_users;
  char **banned_user = banned_users;
  for(; *banned_user; ++banned_user) {
    if (strcmp(*banned_user, user) == 0) {
      free(user_info);
      if (banned_users != (char**)DEFAULT_BANNED_USERS) {
        free_values(banned_users);
      }
      fprintf(LOGFILE, "Requested user %s is banned\n", user);
      return NULL;
    }
  }
  if (banned_users != NULL && banned_users != (char**)DEFAULT_BANNED_USERS) {
    free_values(banned_users);
  }
  return user_info;
}
```

### SecureIOUtils

SecureIOUtils 是一个公用的提供安全的读写本地文件接口的工具类，当集群开启安全验证之后，SecureIOUtils 提供的读写接口会对发起读写用户与本地文件的属主进行对比，如果不一致，则抛出 IOException。

MR 的 ShuffleHandler 在读取 map 输出的 spill 文件时会调用 SecureIOUtils，ShuffleHandler 读时使用的用户任务提交用户，当用户与文件属主不一致时，shuffle 就会一直失败，最终任务的 reduce 因为 shuffle 失败而退出。

```java

  private static void checkStat(File f, String owner, String group, 
      String expectedOwner, 
      String expectedGroup) throws IOException {
    boolean success = true;
    if (expectedOwner != null &&
        !expectedOwner.equals(owner)) {
      if (Path.WINDOWS) {
        UserGroupInformation ugi =
            UserGroupInformation.createRemoteUser(expectedOwner);
        final String adminsGroupString = "Administrators";
        success = owner.equals(adminsGroupString)
            && ugi.getGroups().contains(adminsGroupString);
      } else {
        success = false;
      }
    }
    if (!success) {
      throw new IOException(
          "Owner '" + owner + "' for path " + f + " did not match " +
              "expected owner '" + expectedOwner + "'");
    }
  }

```

## 优化方案

根据上述的代码逻辑分析，我们发现其实想要实现我们的目的其实是很简单的，只要修改两点就可以做到：

- LCE getRunAsUser 支持 security 模式下返回统一用户
- SecureIOUtils 支持 security 模式下用户读取不属于自己的文件

代码修改实现起来也很简单，但是我们还是需要考虑一下怎样实现更加优雅，能更好的兼容原有的功能。说到这，额外多说一句，在日常的项目开发工作中，有很多朋友都喜欢以实现功能优先，不考虑其他的问题，先使用最快最简单的方式实现。比如说上述两个功能，很多朋友可能就直接修改 LCE 的 getRunAsUser 方法使之在任何情况下都返回同一个基于配置用户，更有甚者直接硬编码返回 `yarn` 用户，然后 SecureIOUtils 中直接去除安全模式时的用户比较。这样一来基本没什么修改就实现了功能。

对于这种开发方式，其实我是不太赞同的，因为这种开发方式确实能够快速满足当前需求，但是往往都是以牺牲掉代码的兼容性和扩展性为代价的。这种方式，对于一些比较小的项目，可能还能够适用，对于类似 hadoop 这种大型的持续开发演进的系统，这种方式就是不可取的。比如我刚才举例的实现方式，确实快，可能也就改了 2 行代码，但是它的后果就是，修改上线后的 NM 不再支持用户权限隔离功能了，后续如果业务真有了这个需求，也是没办法支持的，除非把代码再改回来。

因此，在 hadoop 这种大型系统的功能开发工作中，我们在实现功能的同时，更要考虑实现功能的方式与系统本身的兼容性，以及功能后续的扩展性，以防止当前的实现方式在后续带来更多的问题。具体到本文中的功能，我们首先应该考虑在保证在 security 模式下 NM 原有的任务用户隔离功能的基础上，同时支持 NM 使用同一用户来运行所有 container 。我们都知道 hadoop 基于配置的，很多功能属性的启用都是通过配置，所以我们在实现该功能时，也应该通过配置来实现。具体配置如下：


YarnConfiguration.java
```java

  // secure-mode 模式下是否开启用户限制，默认为 true, 即使用同一用户运行 container
  // 设置成为 false 时，则与原始逻辑一致 
  public static final String NM_SECURE_MODE_LIMIT_USERS = NM_PREFIX +
      "linux-container-executor.secure-mode.limit-users";
  public static final boolean DEFAULT_NM_SECURE_MODE_LIMIT_USERS = true;

  // secure-mode 模式下开启用户限制时使用的用户，默认 nobody，建议配置为 yarn
  public static final String NM_SECURE_MODE_LOCAL_USER_KEY = NM_PREFIX +
      "linux-container-executor.secure-mode.local-user";
  public static final String DEFAULT_NM_SECURE_MODE_LOCAL_USER = "nobody";

```
CommonConfigurationKeys.java

```java

  // secure-mode 模式下是否限制用户只能读写属于自己的本地文件，默认是 true，与原始逻辑一致
  // 设置成 false 时，会跳过文件用户检测，及允许用户读取不属于自己的本地文件。
  public static final String HADOOP_SECURITY_IO_LIMIT_USER_KEY =
      "hadoop.security.io.limit-user";
  public static final boolean HADOOP_SECURITY_IO_LIMIT_USER_DEFAULT =true;

```

具体代码就不全部贴出来了，参见基于 3.2.0 分支实现的 [branch-3.2.0.patch](yarn_limit_user_in_securemode-branch-3.2.0.patch) 即可。patch 合并之后，通过设置以下配置我们就可以实现在 security 模式下，NM 统一使用 yarn 用户来运行任务。

```xml

  <property>
    <name>yarn.nodemanager.linux-container-executor.secure-mode.limit-users</name>
    <value>true</value>
  </property>

  <property>
    <name>yarn.nodemanager.linux-container-executor.secure-mode.local-user</name>
    <value>yarn</value>
  </property>

  <property>
    <name>hadoop.security.io.limit-user</name>
    <value>false</value>
  </property>

```

