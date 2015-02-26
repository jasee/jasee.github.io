---
layout: post
title: Salt搭建主备master及问题
category: 运维
description: 在Salt中搭建多个master进行冗余的步骤及遇到的问题。
tags: ["Salt"]
---

"备份不做，十恶不赦，冗余不做，日子甭过"。Saltstack提供了一个方便的master冗余方案，使得我们在master服务器宕机后不至于无法工作。按照以下步骤，我们可以搭建具有主备master的Saltstack系统，平时在主master上工作，万一发生宕机异常时，切换到备master上继续干活。

### 同步配置文件
首先，要同步主备的`/etc/salt/master`文件，该文件包括`nodegroups`、`external_auth`等配置。

### 同步master的公私钥
由于Salt的通信都是加密的，minion只保存一份master的公钥(`/etc/salt/pki/minion/minion_master.pub`)，为了保证其和主备master都能通信，需要两个master使用同一套公私钥。Salt master在启动时会自动生成一份公私钥，在主master启动后备master启动前，同步这套公私钥到备机同样位置:

```text
/etc/salt/pki/master/master.pem
/etc/salt/pki/master/master.pub
```

之后启动备master。

### 配置minion
minion的配置也非常简单，只需要罗列master的地址:

```text
master: 
  - master01.opjasee.com
  - master02.opjasee.com
```

### 同步minion公钥
如果master没有开启`auto_accept`并且不想每次都在主备上重复执行接受、拒绝或删除minion公钥的命令，那么可以选择同步minion的公钥保存目录:

```text
/etc/salt/pki/master/minions
/etc/salt/pki/master/minions_pre
/etc/salt/pki/master/minions_rejected
```

### 同步pillar_roots和file_roots
`fileserver_backend`这个地方最开始想使用gitfs的，但是考虑再三还是决定使用`file_roots`，直接克隆一个git项目到`/srv/salt`。放弃gitfs的原因有三个，也可能是我理解有误：

1. 我们的Gitlab是部署在办公区的一台虚拟机上的，平时上上线没问题，但是考虑到机房和办公区直接的VPN质量和物业断电的频率，还是不能将这么明显的单点挂在Salt上，虽然其能保证主备master的文件一致，但得不偿失。
2. `file_roots`支持gitfs，但是没看到什么地方说`pillar_roots`也支持，如果后者还是需要定期进行同步，那么gitfs的便利也带来了一种环境两种同步方式的问题。
3. 直接从Git仓库里取文件不如本地直观，感觉也没有`git pull`前再检查一遍安全。

因此目前暂定不直接使用gitfs，而是通过多个git项目分别管理`pillar_roots`和`file_roots`，这时候备master上可以通过`rsync`或`git pull`来定期同步，一个参考的命令:

```sh
$ rsync -av --progress --delete --timeout=30 root@master01.opjasee.com:/srv/pillar /srv/
```

另外，可以在master的配置中使用正则或通配排除一部分无关或临时文件，避免minion自动同步时将这些目录或文件也同步过去:

```text
file_ignore_regex:
  - '/\.git($|/)'

file_ignore_glob:
  - '*.pyc'
  - '*.bak'
  - '*.swp'
```

这时候在minion看来master上的这些文件或目录都不存在了，包括直接使用`cp.get_file salt://xxx/.git/config /tmp/`这种命令也无法读取`.git`目录下的文件了。

### 一个问题
乐滋滋的写完准备收工，突然发现了一问题！
虽然Salt号称目前minion已经可以[与多个master同时连接][1]，也不需要在minion中配置`master_type`了，我在测试环境里也确实可以在主备master上都可以执行命令，但是过了几分钟后，备master就开始陆续无法获取minion的执行结果了，主master的日志里记录以下错误:

```text
[salt.loaded.int.returner.local_cache       ][ERROR   ] An inconsistency occurred, a job was received with a job id that is not present in the local cache: 20150202190624634189
```

从minion的debug日志来看，其能够接收并执行job，应该是返回发错了地方。
继续观察与测试，发现了一个有趣的现象：

1. 当所有minion重启后，主备master都能够正常执行命令。
2. 如果此时在某台master上执行了`state.highstate`，那么这台master就会出现上述问题，再也获取不到minion的返回了。
3. 此时在另一台master上执行`state.highstate`和其他命令，都不会有问题。

看来第一个在某minion上调度highstate的master将会丢失该minion后续命令的返回，由于我配置了highstate的定时调度，所以过了几分钟一个master就出问题了。
目前已经向开发者[反馈了该问题][2]，暂不确定是不是我个人环境或使用的问题，还是确实是一个bug，待续。


如果这个问题暂时不能解决，那么要么关闭备master，将其做成冷备；要么重启所有minion后直接在备master上执行`highstate`自废武功，以便主master能保持正常，经测试，在这种情况下，如果主master异常，重启所有minion后，备master执行`highstate`是不会引发问题的，可以暂用这个方案。感觉多master存活时，`highstate`将使minion只与一个master正常连接。

*后续：后来使用了ZeroMQ4就没有出现此问题了。*

另外Salt丢节点默认也没个提示，暂时先`alias salt='salt -v'`吧。

[1]: http://docs.saltstack.com/en/latest/topics/tutorials/multimaster.html
[2]: https://github.com/saltstack/salt/issues/20197
