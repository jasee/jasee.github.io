---
layout: post
title: 为Galera Cluster搭建从库
category: 数据库
description: 为Galera Cluster搭建从库
tags: ["Galera Cluster", "MySQL", "Replication"]
---

为Galera Cluster添加从库有以下好处：

1. 可以用于进行数据库备份而不影响集群服务。
2. 一些特别大的但对时间要求不高的操作可以在从库上运行。

另外这次搭建从库不准备继续使用传统的跟踪binlog位置的方法，改用GTID，有以下好处：

1. 方便将从库切换到另一个主库下。不过这一点在MariaDB Galera Cluster上可能没这么简单，尝试了一下没成功，具体方法后续调研。
2. 从库的记录是crash-safe的。这主要受益于`mysql.gtid_slave_pos`系统表是事务引擎。

Galera的安装及配置以及Slave的安装和配置都集成在Saltstack里了，这里不再记录，需要关注的是Galera Cluster需要打开[log_slave_updates](https://mariadb.com/kb/en/mariadb/replication-and-binary-log-server-system-variables/#log_slave_updates)配置(打开这个配置后节点的`gtid_current_pos`变量都置空了，可能还有其他变化)。下面记录了从库搭建数据部分的步骤：

1. 在Galera Cluster的一个节点上执行备份。

    ```sh
    $ innobackupex --defaults-file=/etc/my.cnf --host=127.0.0.1 --port=3306 --user=backup --password='XXXpassword' /work/tmp
    ```
2. 在该节点上查询备份时的GTID。

    ```text
    SELECT BINLOG_GTID_POS("mysql-bin.000016", 25896);
    # binglog的位置信息可以在备份中找到。
    ```
3. 在从机上恢复数据。

    ```sh
    $ innobackupex --apply-log 2015-05-12_14-34-41
    $ innobackupex --defaults-file=/etc/my.cnf --copy-back 2015-05-12_14-34-41
    $ chown mysql:mysql -R /work/mysql
    $ service mysql start
    ```
4. 在从机上启动复制进程。

    ```text
    SET GLOBAL gtid_slave_pos = "0-111-6315";
    CHANGE MASTER TO master_host="172.16.x.x", master_port=3306, master_user="replicate", master_password='XXXpassword', master_use_gtid=slave_pos;
    ```

后续再调研GTID逻辑、增量备份以及备份时候含有Galera信息等问题。
