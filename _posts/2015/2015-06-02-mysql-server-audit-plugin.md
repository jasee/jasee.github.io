---
layout: post
title: 使用server_audit插件进行MySQL操作审计
category: 数据库
description: 使用server_audit插件进行MySQL操作审计
tags: ["Audit", "MySQL"]
---

默认MySQL没法查看数据库的用户登陆记录，也很难从binlog中分析操作记录。MariaDB提供了一个审计插件，也可以用在MySQL上，这次记录一下启用该插件的过程。

1. 首先到[下载页面](https://mariadb.com/my_portal/download/audit_plugin)下载该插件，目前该插件最新版本是1.2.0。访问该下载页面需要注册登录。
2. 登录数据库，查看该数据库的插件目录:

    ```
    show variables like 'plugin_dir';
    ```
3. 将插件拷贝到对应目录里，并在`my.cnf`中增加如下配置:

    ```
    plugin_dir = /usr/lib64/mysql/plugin
    plugin-load = server_audit=server_audit.so
    server_audit_events = connect,query_dml,query_ddl
    server_audit_output_type = file
    server_audit_file_path = /work/mysql/log/audit.log
    server_audit_file_rotate_size = 1G
    server_audit_logging = ON
    server_audit_file_rotations = 30
    ```
4. 安装插件，此时数据库会自动从配置文件中读取插件参数配置，至此数据库就可以记录连接和操作信息了。

    ```
    INSTALL PLUGIN server_audit SONAME 'server_audit.so';
    show variables like 'server_audit%';
    ```

需要注意的是:

1. SELECT是DML。
2. GRANT等是DCL，按照上述的配置的话，这些语句不会记录。
