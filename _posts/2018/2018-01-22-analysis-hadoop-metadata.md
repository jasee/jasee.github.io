---
layout: post
title: 分析Hadoop元数据
category: 运维
description: 通过工具解析hadoop元数据并映射成hive表进行统计分析
tags: ["Hadoop"]
---

在管理HDFS时，经常需要统计分析元数据信息，比如用于分析权限、增量、小文件，在分析Hive元数据时也能排上用场。Hadoop自带了工具，可以高效、快速地将镜像文件解析出来，将其映射到Hive表后用起来很方便。

1. 通过Hadoop自带的`oiv`(Offline Image Viewer)命令解析镜像文件，注意找个空闲点的内存大的机器来做。

	```sh
	mkdir tsv
    # 镜像文件较大的话，需要配置大内存来完全装载，和NN的内存配置一样就足够了。
	HADOOP_HEAPSIZE=30000 hdfs oiv -i fsimage_0000000015998638409 -p Delimited -o tsv/fsim
    # 产生的结果较大的话，还可以拆分成多个文件，后续在Hive里查起来会快一点。
	cd tsv && split -b 1000m fsim
	```
2. 建表

    ```text
    CREATE EXTERNAL TABLE `hdfs_meta`(
      `path` string,
      `repl` int,
      `modification_time` string,
      `accesstime` string,
      `preferredblocksize` int,
      `blockcount` double,
      `filesize` double,
      `nsquota` int,
      `dsquota` int,
      `permission` string,
      `username` string,
      `groupname` string)
    ROW FORMAT DELIMITED
      FIELDS TERMINATED BY '\t'
    STORED AS INPUTFORMAT
      'org.apache.hadoop.mapred.TextInputFormat'
    OUTPUTFORMAT
      'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
    LOCATION
      'hdfs://ns/path/to/hdfs_meta'
    ```
3. 上传文件

    ```sh
    hdfs dfs -put x* /path/to/hdfs_meta/
    ```

4. 现在就可以跑SQL分析啦，比如统计小文件(四层目录)

    ```sql
    SELECT n.newpath,
           n.total/10000 total,
           n.tsize,
           n.avg
    FROM
      (SELECT h.newpath,
              count(1) total,
              sum(h.filesize)/1024/1024/1024/1024 tsize,
              sum(h.filesize)/count(1)/1024/1024 AVG
       FROM
         (SELECT regexp_extract(path, '(/[^/]+/[^/]+/[^/]+/[^/]+)/.*', 1) newpath,
                 filesize
          FROM hdfs_meta
          WHERE repl > 0 ) h
       GROUP BY h.newpath) n
    WHERE n.total > 10000
    ```
