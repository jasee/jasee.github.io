---
layout: post
title: OpenLDAP--使用rsyslog进行用户操作记录
category: 运维
description: 使用OpenLDAP后，服务器上将增加很多私人账号，为了提高安全性及出现问题时定位排查，需要记录操作历史。本地`history记录存在诸多问题，如单用户多回话记录互相覆盖、记录长度有限容易被修改及本地保存异常时难以获取等，通过简单的配置`rsyslog`，就可以避免以上问题实现基本的操作审计功能。
tags: ["Rsyslog","OpenLDAP"]
---

本篇内容和OpenLDAP关系不大，不过也是该系列内容之一，仍归为LDAP使用一类。
使用OpenLDAP后，服务器上将增加很多私人账号，为了提高安全性及出现问题时定位排查，需要记录操作历史。本地`history记录`存在诸多问题，如单用户多回话记录互相覆盖、记录长度有限容易被修改及本地保存异常时难以获取等，通过简单的配置`rsyslog`，就可以避免以上问题实现基本的操作审计功能。

### 日志收集端配置
以`tao02`作为日志服务器端，接收记录所有客户端操作记录。在`tao02`上增加配置文件`/etc/rsyslog.d/server.conf`:

```
$ModLoad imtcp
$InputTCPServerRun 514
local1.info /var/log/client
```

配置完成后重启rsyslog服务。为了提高可靠性，我们使用TCP进行传输。`local1.info`是自定义的日志来源和日志级别，后续进行日志记录的时候也会进行指定，避免接收到其他不相干日志。

### 客户端配置
以`tao03`为例，设置`PROMPT_COMMAND`发送命令记录到rsyslog并配置rsyslog将指定类型的日志发送到`tao02`。
增加配置`/etc/rsyslog.d/client.conf`并重启rsyslog:

```
### begin forwarding rule ###
$WorkDirectory /var/lib/rsyslog # where to place spool files
$ActionQueueFileName fwdRule1   # unique name prefix for spool files
$ActionQueueMaxDiskSpace 1g     # 1gb space limit (use as much as possible)
$ActionQueueSaveOnShutdown on   # save messages to disk on shutdown
$ActionQueueType LinkedList     # run asynchronously
$ActionResumeRetryCount 3       # infinite retries if host is down
local1.info @@tao02.opjasee.com:514
### end of the forwarding rule ###
```

增加配置文件`/etc/profile.d/client.sh`:

```sh
export PROMPT_COMMAND='{ msg=$(history 1 | { read x y; echo $y; });logger -p local1.info "[euid=$(whoami)][$(who am i)][$(pwd)]:$msg"; }'
```

### 测试和其他
以下记录是使用`rda`登陆`tao03`并使用`sudo su test -l`命令切换到`test`用户后执行`echo "test message"`产生的:

```
Mar 31 14:45:16 tao03 rda: [euid=test][rda      pts/1        2014-03-31 14:28 (tao04.opjasee.com)][/home/nfsdir/user/test]:echo "test message"
```

可以将该记录拆分成以下部分:

```
Mar 31 14:45:16 tao03 rda: #rsyslog自带前缀
[euid=test] # whoami的执行结果，当前有效用户
[rda      pts/1        2014-03-31 14:28 (tao04.opjasee.com)] # who am i执行结果，登陆信息
[/home/nfsdir/user/test]:echo "test message" # 执行命令所在目录及命令内容
```

这个记录还是比较全面的，用户的登陆位置、登陆时间都有记录，也能够观察到用户切换，如果设置了`HISTTIMEFORMAT`，还能够看到命令执行的确切时间(命令记录会长一点)
不过也有一些问题，比如记录中存在用户名重复以及用户直接回车产生重复记录，好在有`pwd`存在，`cd ..`这类的命令还是能分辨出来的。另外，记录方式存在一定延后，命令执行完回到命令提示符才能进行记录，特别是`su`这种命令，需要会话结束才能记录，稍有点乱。
另外，rsyslog还可以将日志输出到mysql中，方便进行进一步的分析，具体见参考资料。

#### *参考资料*
*[Linux用户操作审计记录方案](http://jiechao2012.blog.51cto.com/3251753/1143762)*
*[利用Rsyslog集中收集系统日志和用户操作记录以及相关处理方法](http://cyr520.blog.51cto.com/714067/1214850)*
