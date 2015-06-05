---
layout: post
title: Flask-SQLAlchemy的'MySQL server has gone away'问题
category: 数据库
description: 解决Flask-SQLAlchemy的'MySQL server has gone away'问题
tags: ["MySQL", "Flask", "SQLAlchemy"]
---

前段时间将Flask程序的线上数据库迁移到Galera Cluster，之后就经常收到'MySQL server has gone away'错误。

由于之前并没有这个错误，那么问题就出在这次迁移上。首先查看了新旧数据库的`wait_timeout`参数，原来的数据库为默认的8小时，新的数据库为1小时。但是第一时间并没有太关注这个差异，因为我觉得1小时和8小时只是量的问题，以前晚上没人用这个程序的时候空闲也超过8小时，但是早上并没出现这个问题。

然后考虑到Galera Cluster之上新接了一个MaxScale，发现其默认超时时间是半小时，与后端不一致，于是也调整了一下。

这个调整对错误并没有什么改进，开始重新怀疑1小时和8小时的差异，后来看见了下面这段话:

> SQLALCHEMY_POOL_RECYCLE:
> > Number of seconds after which a connection is automatically recycled. This is required for MySQL, which removes connections after 8 hours idle by default. Note that Flask-SQLAlchemy automatically sets this to 2 hours if MySQL is used.

这样问题就明了了，在默认2小时的超时回收策略下，1小时和8小时量变变质变。调整`SQLALCHEMY_POOL_RECYCLE`参数至50分钟，问题解决。
