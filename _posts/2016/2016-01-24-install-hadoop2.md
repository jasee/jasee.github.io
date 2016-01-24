---
layout: post
title: CDH部署流程记录
category: 服务
description: 记录手工安装CDH5.5.1的大概过程
tags: ["Hadoop", "CDH"]
---

最近测试安装了CDH5.5.1，大概的流程如下，具体配置内容较长，不在此细表。
此次测试采用手工安装方式，未采用Cloudera Manager，配置了HA和Kerberos。

## 测试环境

共5台虚拟机，角色分配如下：

1. Kerberos需要一台，部署在hadoop01。
2. Zookeeper需要三台，部署在hadoop01/02/03。
3. JournalNode需要三台，部署在hadoop01/02/03。
4. NameNode需要两台，部署在hadoop01/02。
5. ResourceManager需要两台，部署在hadoop01/02。
6. DataNode/NodeManager需要三台，部署在hadoop03/04/05。
7. Historyserver需要一台，部署在hadoop03。

## 下载安装jdk1.7.0_55

```sh
$ sudo mkdir /usr/java
$ sudo cp -r jdk1.7.0_55/ /usr/java/
$ sudo ln -s /usr/java/jdk1.7.0_55 /usr/java/default
# 在/etc/profile等位置使用以下语句调用该java
$ export JAVA_HOME=/usr/java/default
# 为了使用aes256,需要安装JCE Policy File
# http://www.cloudera.com/content/www/en-us/documentation/cdh/5-1-x/CDH5-Security-Guide/cdh5sg_jce_policy_file_install.html
$ sudo cp US_export_policy.jar local_policy.jar /usr/java/jdk1.7.0_55/jre/lib/security/
```

## 添加cdh仓库
将以下内容保存到`/etc/yum.repos.d/cloudera-cdh5.repo`:

```text
[cloudera-cdh5]
# Packages for Cloudera's Distribution for Hadoop, Version 5, on RedHator CentOS 6 x86_64
name=Cloudera's Distribution for Hadoop, Version 5
baseurl=https://archive.cloudera.com/cdh5/redhat/6/x86_64/cdh/5/
gpgkey =https://archive.cloudera.com/cdh5/redhat/6/x86_64/cdh/RPM-GPG-KEY-cloudera
gpgcheck = 1
```

然后导入该仓库的key:

```sh
$ sudo rpm --import http://archive.cloudera.com/cdh5/redhat/5/x86_64/cdh/RPM-GPG-KEY-cloudera
```

## 安装Kerberos

本步骤仅作为简单记录，详细过程可参考OpenLDAP安装记录。

```sh
# hadoop01上安装Kerberos Server
$ sudo yum install krb5-server
# 所有服务器安装客户端
$ sudo yum install -y krb5-workstation
# 各配置及初始化及后续命令省略
```

## 安装Zookeeper

线上Storm集群已有ZK，可以考虑复用。

在hadoop01/02/03上安装`zookeeper-server`:

```sh
$ sudo yum install zookeeper-server
$ sudo mkdir /work/zkdata
```

将以下内容写入这三台服务器`/etc/zookeeper/conf/zoo.cfg`:

```text
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/work/zkdata
clientPort=2181
autopurge.snapRetainCount=10
autopurge.purgeInterval=1
server.1=hadoop01:2888:3888
server.2=hadoop02:2888:3888
server.3=hadoop03:2888:3888
```

在三台服务器上依次执行以下命令完成第一次启动:

```sh
$ sudo chown zookeeper:zookeeper -R /work/zkdata/
$ sudo service zookeeper-server init
# 在三台服务器上建立`/work/zkdata/myid`文件，并依次写入1/2/3(和上面配置的server对应)。
# 由于init操作会清空/work/zkdata目录，因此建立myid的操作需要在init之后进行，该文件权限可保留为root
$ sudo service zookeeper-server start
```

## 安装CDH

```sh
# 在hadoop01/02/03安装JournalNode
$ sudo yum clean all; sudo yum install hadoop-hdfs-journalnode
# 在hadoop01/02安装NameNode和ZKFC
$ sudo yum clean all; sudo yum install hadoop-hdfs-namenode hadoop-hdfs-zkfc
# 在hadoop01/02安装ResourceManager
$ sudo yum clean all; sudo yum install hadoop-yarn-resourcemanager
# 在hadoop03/04/05安装nodemanager/datanode/mapreduce
$ sudo yum clean all; sudo yum install hadoop-yarn-nodemanager hadoop-hdfs-datanode hadoop-mapreduce
# 在hadoop03上安装historyserver/proxyserver
$ sudo yum clean all; sudo yum install hadoop-mapreduce-historyserver
# 对于hadoop客户端，需要安装client
$ sudo yum clean all; sudo yum install hadoop-client
# 根据需求确定是否保留开机自动启动
```

## 配置CDH

```sh
# 以下步骤需要在所有节点执行
# 首先复制默认配置到自定义目录，后续只在该目录修改，不要直接改动默认目录
$ sudo cp -r /etc/hadoop/conf.empty /etc/hadoop/conf.opjasee_cluster
# CDH通过alternatives确定应该使用哪个配置目录
$ sudo alternatives --install /etc/hadoop/conf hadoop-conf /etc/hadoop/conf.opjasee_cluster 50
$ sudo alternatives --set hadoop-conf /etc/hadoop/conf.opjasee_cluster
# 下面这个命令可以看当前的alternatives配置
$ sudo alternatives --display hadoop-conf
# 具体配置请直接参考conf目录下配置文件，不在此列出
```

另外`/etc/default/`目录下定义了日志路径等环境变量，具体配置见`default`目录下文件，按需覆盖。

## 准备本地目录

在不同节点上所需的目录不同，以下步骤为全部节点的综合，可按需配置。

```sh
# HDFS
$ sudo mkdir -p /work/hadoop-hdfs/dfs/nn /work/hadoop-hdfs/dfs/jn /work/hadoop-hdfs/dfs/dn
$ sudo chown hdfs:hadoop -R /work/hadoop-hdfs/dfs
$ sudo chmod 700 /work/hadoop-hdfs/dfs/nn /work/hadoop-hdfs/dfs/dn
# YARN
$ sudo mkdir -p /work/hadoop-yarn/local /work/hadoop-yarn/logs
$ sudo chown yarn:hadoop /work/hadoop-yarn/local /work/hadoop-yarn/logs
# Kerberos
$ sudo mkdir /work/hadoop-keytab
$ sudo mkdir -p /work/hadoop-log/hdfs /work/hadoop-log/mapred /work/hadoop-log/yarn
# Logs
$ sudo chown hdfs:hadoop /work/hadoop-log/hdfs
$ sudo chown mapred:hadoop /work/hadoop-log/mapred
$ sudo chown yarn:hadoop /work/hadoop-log/yarn
$ sudo chmod 775 /work/hadoop-log/hdfs /work/hadoop-log/mapred /work/hadoop-log/yarn
```

## 准备Kerberos

在各个节点执行以下命令生成对应的keytab(iTerm的Broadcast Input在这里真的挺方便的)。

```sh
$ kadmin -p admin/admin -q "addprinc -randkey hdfs/$HOSTNAME.opjasee.com@OPJASEE.COM"
$ kadmin -p admin/admin -q "addprinc -randkey mapred/$HOSTNAME.opjasee.com@OPJASEE.COM"
$ kadmin -p admin/admin -q "addprinc -randkey yarn/$HOSTNAME.opjasee.com@OPJASEE.COM"
$ kadmin -p admin/admin -q "addprinc -randkey HTTP/$HOSTNAME.opjasee.com@OPJASEE.COM"
$ kadmin -p admin/admin -q "xst -k /tmp/hdfs-unmerged.keytab hdfs/$HOSTNAME.opjasee.com@OPJASEE.COM"
$ kadmin -p admin/admin -q "xst -k /tmp/mapred-unmerged.keytab mapred/$HOSTNAME.opjasee.com@OPJASEE.COM"
$ kadmin -p admin/admin -q "xst -k /tmp/yarn-unmerged.keytab yarn/$HOSTNAME.opjasee.com@OPJASEE.COM"
$ kadmin -p admin/admin -q "xst -k /tmp/http.keytab HTTP/$HOSTNAME.opjasee.com@OPJASEE.COM"
# keytab合并
$ ktutil
ktutil:  rkt /tmp/hdfs-unmerged.keytab
ktutil:  rkt /tmp/http.keytab
ktutil:  wkt /tmp/hdfs.keytab
ktutil:  clear
ktutil:  rkt /tmp/mapred-unmerged.keytab
ktutil:  rkt /tmp/http.keytab
ktutil:  wkt /tmp/mapred.keytab
ktutil:  clear
ktutil:  rkt /tmp/yarn-unmerged.keytab
ktutil:  rkt /tmp/http.keytab
ktutil:  wkt /tmp/yarn.keytab
$ sudo mv /tmp/hdfs.keytab /tmp/mapred.keytab /tmp/yarn.keytab /work/hadoop-keytab/
$ rm -f /tmp/hdfs-unmerged.keytab /tmp/mapred-unmerged.keytab /tmp/yarn-unmerged.keytab /tmp/http.keytab
$ sudo chown hdfs:hadoop /work/hadoop-keytab/hdfs.keytab
$ sudo chown mapred:hadoop /work/hadoop-keytab/mapred.keytab
$ sudo chown yarn:hadoop /work/hadoop-keytab/yarn.keytab
$ sudo chmod 400 /work/hadoop-keytab/*.keytab
# On NameNode, For NameNode sshfence fencing method
$ sudo cp /home/vagrant/.ssh/id_rsa /work/hadoop-keytab/
$ sudo chown hdfs:hdfs /work/hadoop-keytab/id_rsa
```

## 第一次启动HDFS

```sh
# 先在三个JN上启动journalnode
$ sudo service hadoop-hdfs-journalnode start
# 在hadoop01上格式化NameNode
$ sudo -u hdfs hdfs namenode -format
# 在hadoop02同步元数据
$ sudo -u hdfs hdfs namenode -bootstrapStandby
# 在hadoop01上初始化znode
$ hdfs zkfc -formatZK
# 在hadoop01/02上分别启动ZKFC
$ sudo service hadoop-hdfs-zkfc start
# 在hadoop01启动主NN
$ sudo service hadoop-hdfs-namenode start
# 在hadoop02启动备NN
$ sudo service hadoop-hdfs-namenode start
# 在hadoop03/04/05上启动DN
$ sudo service hadoop-hdfs-datanode start
# 观察主备的50070端口，查看是否按预期工作
# 可用以下命令上传文件进行测试
$ sudo -u hdfs kinit -k -t /work/hadoop-keytab/hdfs.keytab hdfs/hadoop01.opjasee.com@OPJASEE.COM
$ sudo -u hdfs hadoop fs -put hdfs-site.xml /
```

## 第一次启动YARN

```sh
# 建立所需HDFS目录
$ sudo -u hdfs kinit -k -t /work/hadoop-keytab/hdfs.keytab hdfs/hadoop01.opjasee.com@OPJASEE.COM
$ sudo -u hdfs hadoop fs -mkdir /tmp
$ sudo -u hdfs hadoop fs -chmod -R 1777 /tmp
$ sudo -u hdfs hadoop fs -mkdir -p /tmp/hadoop-yarn/staging/history
$ sudo -u hdfs hadoop fs -chown mapred:hadoop /tmp/hadoop-yarn/staging/history
$ sudo -u hdfs hadoop fs -chmod -R 1777 /tmp/hadoop-yarn/staging
$ sudo -u hdfs hadoop fs -chmod -R 1777 /tmp/hadoop-yarn/staging/history
# 创建需要运行mapred的用户目录(经测试此步骤可以省略，仅供参考)
$ sudo -u hdfs hadoop fs -mkdir -p /user/tao
$ sudo -u hdfs hadoop fs -chown tao /user/tao
# 在hadoop03启动historyserver
$ sudo service hadoop-mapreduce-historyserver start
# 在hadoop01/02启动rm
$ sudo service hadoop-yarn-resourcemanager start
# 在hadoop03/04/05启动nm
$ sudo service hadoop-yarn-nodemanager start
# 可通过19888端口观察historserver状态
# 可通过8088端口观察rm状态
# 可通过8042端口观察nm状态
```

## 进行一次测试

```sh
$ kinit tao/hadoop01.opjasee.com@OPJASEE.COM
# 按照Secure Hadoop Yarn部分的配置，每个节点都需要存在对应的账号用以运行容器
# 在OpenLDAP环境中，统一配置即可
# 在本测试环境所有节点中，手工添加，注意uid
$ sudo useradd tao -u 1000
$ hadoop jar /usr/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar pi 10 100
$ hadoop jar /usr/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar wordcount /hdfs-site.xml /user/tao/test
```
