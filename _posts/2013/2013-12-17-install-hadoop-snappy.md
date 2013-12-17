---
layout: post
title: 安装Hadoop Snappy
category: 数据平台
tagline: 为每个datanode安装hadoop Snappy。
description: 记录安装hadoop snappy的过程，可供主动设定参数调用，暂未更新配置及重启hadoop。
tags: ["Hadoop","Snappy"]
---
{% include JB/setup %}

在某台datanode上完成编译后分发到所有节点。
### 1. 安装snappy

``` sh
$ wget https://snappy.googlecode.com/files/snappy-1.1.1.tar.gz
$ ./configure && make && make install
```

### 2. 安装maven

``` sh
$ wget http://mirror.bit.edu.cn/apache/maven/maven-3/3.1.1/binaries/apache-maven-3.1.1-bin.tar.gz
$ tar -zxf apache-maven-3.1.1-bin.tar.gz
$ cp -r apache-maven-3.1.1 /work/maven
$ diff /tmp/.bash_profile ~/.bash_profile 
10c10,11
< PATH=$PATH:$HOME/bin
---
> MAVEN_HOME=/work/maven
> PATH=$PATH:$HOME/bin:$MAVEN_HOME/bin
$ source ~/.bash_profile
```

### 3. 编译`hadoop-snappy`
*可以考虑跳过之前的步骤1，直接使用`$HADOOP_HOME/lib/native/Linux-amd64-64/`目录下的libsnappy库文件，后边只用分发jar包就行*

``` sh
$ svn checkout http://hadoop-snappy.googlecode.com/svn/trunk/ hadoop-snappy-read-only
$ cd hadoop-snappy-read-only
$ ln -s /work/java/jre/lib/amd64/server/libjvm.so /usr/local/lib
$ mvn package
```

## 4. 安装到Hadoop集群
`hadoop-snappy`编译生成的`target/hadoop-snappy-0.0.1-SNAPSHOT.tar.gz`包已经含有`libsnappy`本地库文件，所以各个节点不需要安装`snappy`，分发即可。

``` sh
$ cd target
$ tar -zxf hadoop-snappy-0.0.1-SNAPSHOT.tar.gz
$ for x in $(cat $HADOOP_HOME/conf/slaves);do scp -r hadoop-snappy-0.0.1-SNAPSHOT/lib/* $x:$HADOOP_HOME/lib/;done
```

此时已经可以通过使用类似`SET hive.exec.compress.output=true; SET mapred.output.compression.codec=org.apache.hadoop.io.compress.SnappyCodec; SET mapred.output.compression.type=BLOCK;`这样的命令来调用`hadoop-snappy`进行压缩了。如果需要让Hadoop或Hive默认使用这些配置，则修改对应的配置文件，涉及Hadoop配置需要重启集群。

### 参考文档
*[Hadoop HBase 配置 安装 Snappy 终极教程](http://shitouer.cn/2013/01/hadoop-hbase-snappy-setup-final-tutorial/)*
*[hadoop-snappy](https://code.google.com/p/hadoop-snappy/)*
*[设定Hive参数](http://www.alidata.org/archives/716)*
