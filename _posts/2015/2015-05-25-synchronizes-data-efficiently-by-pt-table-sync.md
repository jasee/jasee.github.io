---
layout: post
title: 使用pt-table-sync工具消除主从数据差异
category: 数据库
description: 使用pt-table-sync工具消除主从数据差异
tags: ["PT", "MySQL" ]
---

今天一对数据库出现主从同步异常，一条删除语句无法在从库执行，在从库跳过该语句后同步正常，但是导致主从异常的那张表仍然存在数据差异，从库比主库多了9921行数据。

之前的修复手段都是在主库上导出该表的数据，清空该表后重新导入数据，但是这种方式影响数据，操作期间的数据插入会丢失，并且自增ID也会有错乱。
这次使用percona提供的工具进行检验和修复。

### 使用`pt-table-checksum`检查数据一致性

```sh
$ pt-table-checksum -u root --ask-pass -h 172.16.x.x --databases=database --tables=table --max-lag=10s --check-interval=1m --no-check-binlog-format
```

这条命令会比较主从库指定表的数据块，并将结果写入`percona.checksums`表中，执行后查询该表确实发现有9921行的数据差异。

### 使用`pt-table-sync`修复主从数据差异

```sh
# DSN会在操作记录里暴露从库的密码，可以写到文件中执行
# 可以用--dry-run或--print预执行查看效果
$ pt-table-sync --sync-to-master h=172.16.x.x,u=root,p=XXXpassword --databases=bc_click_report --tables=fact_realtime_conversion --execute
```

该语句会进行差异检查并消除主从库的数据差异。执行后重新使用上述检查命令检查，数据已经一致。

具体原理和更多用法参考对应命令的文档。
