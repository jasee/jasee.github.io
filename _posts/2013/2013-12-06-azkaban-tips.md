---
layout: post
title: Azkaban使用说明
category: 服务
tagline: 使用Azkaban调度Hadoop任务的简单说明
description: Azkaban相比于crontab调度具有很多优势，也有很多不同。本文主要对原有任务迁移到Azkaban上需要关注的内容做一些说明。
tags: ["Azkaban","Hadoop"]
---
{% include JB/setup %}

Azkaban相比于crontab调度具有很多优势，也有很多不同。本文主要对原有任务迁移到Azkaban上需要关注的内容做一些说明。错误及遗漏部分逐步更正更新。

### azkaban基本流程
1. azkaban的项目创建时分配一个project id，第一个版本号为1，每次对进行项目文件上传版本号加一。项目文件zip包以blob方式存放在数据库中。
2. 项目的`.job`文件在项目zip包上传时将单独提取以blob方式存放到数据库的项目的`project_properties`表中，在web端修改job配置会更新这个表，不更新项目zip包。
3. 项目需要执行时，首先将项目文件从数据库下载到本地的executor的`projects`目录下，项目目录为`projectid.version`。如果该目录存在，则不进行下载，如果通过其他手段删除或修改项目文件将导致异常。
4. 项目的实际执行位置为executor的`executions/ExecutionId`，执行时最终使用的job配置以`project_properties`表中的为准。在web上所做修改将在更新zip包version变更后失效。
5. 同一个项目只存在一个项目目录`projectid.version`，旧版本项目目录将被删除。
6. 执行目录保留两天。正在测试单一job运行超过两天的情况。

### 目录及文件
1. azkaban的`.job`文件中的`command`命令在执行目录中执行。
2. 在azkaban的执行目录内，将创建软链链接到项目目录，软链只包括文件不包含目录。
3. job执行过程中，对旧有目录进行操作只在执行目录生效，对旧有文件进行操作则通过软链影响项目文件(删除操作只删除软链不删除项目文件)。
4. 执行目录下的空目录中生成`*`软链，应该是azkaban软链处理的缺陷。 

### 环境
1. `/etc/profile`及`~/.bash_profile`都会被自动加载。
2. 待观察。

### azkaban权限
1. azkaban权限管理较弱，定义在azkaban-users.xml中的用户权限为全局权限，无分组。
2. 更新azkaban-users.xml需重启azkaban web。（待寻找其他方法）
3. job部署、调度及执行账号与观察账号分离。可临时将某些项目权限赋予观察账号进行修补数据等操作。可每人分配一个观察账号。
4. 建立一些单独的测试项目分配给观察账号，用以对azkaban本身的测试及试用。

### job部署需求
1. 关于权限
    1. azkaban使用work账户启动，所有部署其上的job都具有work账户权限，不可使用超出权限的命令。
    2. 涉及hadoop部分，依旧使用kerberos上的root账号，权限暂无变更。
    3. 本地权限与hadoop权限交互部分，如`hadoop.tmp.dir`目录权限，需额外关注。（全局，具体job不用关心）
2. 关于数据目录
    1. azkaban项目目录与执行目录分离，版本变更时项目目录同时变更。
    2. 对于可执行文件位置、配置文件位置等，可以在脚本中设置自动获取当前路径进行路径拼接，也可以使用相对路径，但不能写死绝对路径。
    3. 对于数据文件，两次调度或者两个版本之间具有数据依赖时（如本次调度需要使用上次调度产生的文件，新版job要继续使用原有job生成的文件等），需要指定绝对路径进行数据存储，如`/work/jobdata/jobabc/data/`。
    4. 为了防止任务循环或者补数据冲突，每次调度在HDFS上的结果存储位置应该唯一。azkaban每次调度在本地创建临时环境，不用考虑本地问题。
3. 关于日志
    1. 标准输出将记录到azkaban的数据库中，以blob方式存储在`execution_logs`表中，可以在web端查看。优点：查看方便。缺点：大量日志时性能及清理问题。
    2. 写到相对路径的日志如`./log/logdate.log`将随着执行环境保留两天后被自动清理。需登录服务器查看。优点：自动清理，日志量可以很大。缺点：查看不便，保留时间短。
    3. 写到绝对路径的日志如`/work/jobdata/jobabc/log/logdate.log`将一直保存，但job最好有清理机制。需登录服务器查看。优点：保留时间可控，日志量可以很大。缺点：查看不便，需要自带清理逻辑。
    4. zip包中包含日志空文件，如`./log/naillog`，通过执行环境的软链直接写到项目文件中。优点：自动根据版本划分并清理。缺点：设定方式反人类易出错。
4. 关于补数据
    1. azkaban支持传参及参数重新，zip包中`system.properties`定义的内容可以在执行时引用，如定义`param=${azkaban.flow.start.year}${azkaban.flow.start.month}${azkaban.flow.start.day}${azkaban.flow.start.hour}`，则可以在`.job`文件中使用类似`command.1=./bin/Union_log.sh ${param}`进行引用。
    2. 当job失败时，在上游数据准备好之后，可以开始重新执行失败的job进行数据修补。通过azkaban的`History`列表找到失败job，点击进去选择`Prepare Execution`可以进入执行配置界面。可以选择禁用流程中的某些节点（执行到这些节点时不进行任何操作直接返回成功并跳过）、设置报警等，最后在`Flow Parameters`中重写参数值，如定义`param=2013010101`，然后开始执行。
    3. 如需按照上述两点进行数据修补，需要在程序中设置使用命令行参数，类似`if [ “$1” ]; then param=“$1”; else param=$(date +%Y%m%d%H);done`这种逻辑。
