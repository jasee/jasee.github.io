---
layout: post
title: 升级hadoop版本
category: 服务
description: 升级Hadoop版本过程记录，hadoop升级主要在于HDFS数据布局升级
tags: ["Hadoop","HDFS"]
---

2013年初将一个Hadoop集群从版本0.20.2升级至1.0.4，记录整理如下，耗时供参考。
集群规模：25个节点，100T数据(34T * 3)，文件及块共1150万。

#### 0. 准备工作
在所有hadoop服务器上建立升级目录：`lhcmd da-tb-hadoop-* 'mkdir -p /work/hadoopupdate/backup`。
将配置好的新Hadoop包分发到所有Hadoop服务器的/work/hadoopupdate/目录下，并在校验md5值后解压。调整NN及SNN的heapsize参数。
因为之前进行了网络调整，hadoop已经关闭，故需先打开HDFS进行升级前状态保存:`start-dfs.sh`。

#### 1. 查看现有hadoop是否已经完成上一次升级

```sh
$ hadoop dfsadmin -upgradeProgress status
```

返回`There are no upgrades in progress`表示已经完成，可以继续本次升级。

#### 2. 在NN上执行以下几条命令，记录现有HDFS状态，留作升级后对比(选用)
```sh
$ hadoop dfsadmin -safemode enter
$ hadoop dfs -lsr / >/work/hadoopupdate/oldlsr
#耗时14分钟，输出文件大小为1300M。
$ hadoop fsck / >/work/hadoopupdate/oldfsck
#此命令可能耗时60分钟，未使用。
$ hadoop dfsadmin -report>/work/hadoopupdate/oldreport
#几乎不耗时。
```

#### 3. 关闭集群并确保各个服务器上没有Hadoop进程

```sh
$ stop-all.sh
$ lhcmd da-tb-hadoop-* 'ps -efw|grep hadoop'
```

#### 4. 备份镜像文件

```sh
$ cp -r /work/hadoopneed/name /work/hadoopupdate/backup/
```

secondarynamenode此次镜像目录位置更改，否则最好也备份一下。

#### 5. 备份移除所有服务器的现有Hadoop目录

```sh
$ lhcmd da-tb-hadoop-* 'mv /work/hadoop /work/hadoopupdate/backup/'
```

#### 6. 将新版Hadoop放置在`/work`目录下

```sh
$ lhcmd da-tb-hadoop-* 'cp -r /work/hadoopupdate/hadoop /work/'
```

#### 7. 使用升级参数启动hdfs

```sh
$ start-dfs.sh -upgrade
```

此时各台服务器应该都启动HDFS进程，但是不启动MR。注意启动是否正常。

#### 8. 查看升级进行状态

```sh
$ hadoop dfsadmin -upgradeProgress status
#该命令将从几个维度显示升级进度。datanode每分钟向namenode汇报一次升级情况，如果status命令几分钟都没有任何更新，可以使用以下命令查看详情：
$ hadoop dfsadmin -upgradeProgress details
#该命令将显示每一个datanode节点的升级情况。
#如果整个升级过程被个别节点或数据块堵塞，则根据status命令的结果选择是否使用force命令强制完成升级：hadoop dfsadmin -upgradeProgress force
#当升级完成时，status命令应该显示如下：
#Upgrade for version -32 has been completed.
#Upgrade is not finalized.
#NN升级时间小于2分钟，整个HDFS升级耗时12分钟。
```

#### 9. 在初步验证hdfs正常后执行第二步中的命令生成新hadoop集群状态数据，与老数据进行对比

```sh
$ hadoop dfsadmin -safemode enter
$ hadoop dfs -lsr / >/work/hadoopupdate/newlsr
$ hadoop fsck / >/work/hadoopupdate/newfsck #耗时较长，未执行
$ hadoop dfsadmin -report>/work/hadoopupdate/newreport
```

#### 10. 正常后启动MR并观察

```sh
$ start-mapred.sh
```

观察日志目录和日志(此次调整了日志目录等参数)，观察NN/SNN/DN工作情况。

#### 11. 其他及观察：
在`/etc/profile`中追加一行`export HADOOP_HOME_WARN_SUPPRESS=1`，以停止生成`Warning: $HADOOP_HOME is deprecated`。
全面观察hadoop状态。
更新Hadoop客户端。
至此hadoop升级部分暂时完成，需要按照顺序开启Hadoop Job以及进行补数据等操作。

#### 12. 观察一段时间后确定升级完成

```sh
$ hadoop dfsadmin -finalizeUpgrade
```

实际观察时间为两天，这两天内的所有删除操作实际上都是不生效的，磁盘只增不减。同时清理升级过程中临时数据，备份清理清理老版本。
