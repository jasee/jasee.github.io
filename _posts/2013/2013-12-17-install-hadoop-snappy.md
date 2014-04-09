---
layout: post
title: 安装Hadoop Snappy
category: 服务
description: 记录安装hadoop snappy的过程，可供主动设定参数调用，暂未更新配置及重启hadoop。
tags: ["Hadoop","Snappy"]
---

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

### 4. 安装到Hadoop集群
`hadoop-snappy`编译生成的`target/hadoop-snappy-0.0.1-SNAPSHOT.tar.gz`包已经含有`libsnappy`本地库文件，所以各个节点不需要安装`snappy`，分发即可。

``` sh
$ cd target
$ tar -zxf hadoop-snappy-0.0.1-SNAPSHOT.tar.gz
$ for x in $(cat $HADOOP_HOME/conf/slaves);do scp -r hadoop-snappy-0.0.1-SNAPSHOT/lib/* $x:$HADOOP_HOME/lib/;done
```

修改`core-site.xml`加入以下配置并重启Hadoop。

```xml
  <property>
    <name>io.compression.codecs</name>
    <value>
      org.apache.hadoop.io.compress.SnappyCodec
    </value>
  </property>
```

### 5. 其他
Hadoop从1.0.2及0.23.0版本开始，已经[默认包含了hadoop-snappy的支持](https://issues.apache.org/jira/browse/HADOOP-7206)，这些版本如果需要使用snappy，只需要将第1步生成的libsnappy库文件(`/usr/local/lib/libsnappy*`)放置到所有节点的Hadoop本地库目录(`$HADOOP_HOME/lib/native/Linux-amd64-64/`)中即可。Hadoop默认配置文件中已经包含第4步中的配置，不需要进行重启即可直接使用。

### 参考文档
*[Hadoop HBase 配置 安装 Snappy 终极教程](http://shitouer.cn/2013/01/hadoop-hbase-snappy-setup-final-tutorial/)*
*[hadoop-snappy](https://code.google.com/p/hadoop-snappy/)*
*[设定Hive参数](http://www.alidata.org/archives/716)*
