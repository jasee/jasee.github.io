---
layout: post
title: Ganglia--Gweb权限
category: 运维
description: Gweb使用简单的认证和授权进行访问和编辑控制。
tags: ["Ganglia","Monitor","Gweb"]
---

基本环境的安装见[上文][1]，将ghost02单独建立群组`Cluster02`。
一般来说，Ganglia都搭建在内网环境中，Gweb的访问也在公司范围内，但是还是会有一些权限需求，比如部分或全部数据需要限定访问范围,保存的视图不想被改动，监控墙上的项目不被破坏等。Gweb权限管理做得一般，不过这些基本要求也能达到啦。计划实现以下需求：

1. 添加一个管理员，具有所有权限。
2. 将`Cluster01`定义为私有群组。
3. 添加一个访问账号，具有访问私有群组`Cluster01`的权限。
4. 未登录时用户能够访问`Cluster02`，但无法访问`Cluster01`。

------

### 开启Gweb认证
Gweb将默认配置写在`conf_default.php`文件中，如果存在`conf.php`则读取该文件配置并覆盖默认配置。建立`conf.php`文件并写入以下配置

```php
<?php
$conf['auth_system'] = 'enabled';
?>
```

此时访问Gweb页面将会看到一个错误信息，内含`SetEnv ganglia_secret 8da646a96a5d9ab8d72285bebf4f574823ef4f52`类似的语句。这个`ganglia_secret`是一个随机的字符串，待会配置在Apache中，Gweb读取后用来加密认证的用户名。

### 开启Apache认证
Gweb其实本身不做认证，只做授权，其认证信息来源于Apache。只需要在Apache中配置访问`login.php`需认证，用户通过Apache认证后访问该文件，文件读取`$_SERVER['REMOTE_USER']`信息，作为已登录用户名，并根据接下来再说的ACL规则赋予权限。新建`/etc/httpd/conf.d/ganglia.conf`文件并重新加载Apache配置，文件内容如下

```
SetEnv ganglia_secret 8da646a96a5d9ab8d72285bebf4f574823ef4f52
<Location "/ganglia/login.php">
    AuthType Basic
    AuthName "Ganglia Access"
    AuthUserFile /var/lib/ganglia-web/htpasswd
    Require valid-user
</Location>
```

### 用户授权
建立两个用户，命令如下

```sh
$ htpasswd -bc /var/lib/ganglia-web/htpasswd jasee password
$ htpasswd -b /var/lib/ganglia-web/htpasswd viewer viewer
```

在`conf.php`中添加以下语句进行授权

```php
$acl = GangliaAcl::getInstance();
$acl->addRole('jasee', GangliaAcl::ADMIN);
$acl->addPrivateCluster('Cluster01');
$acl->addRole('viewer', GangliaAcl::GUEST);
$acl->allow('viewer', 'Cluster01',GangliaAcl::VIEW);
```

Gweb的权限管理是挺简陋的，在面包屑导航部分，不管是否具有权限，所有的群组名字都会列出，访问到不具有权限的群组时给出一个很挫的界面。不过功能上勉强满足啦。
更多权限信息参考底部参考文档。

#### 参考文档
*[ganglia-web-2/AuthSystem][2]*


[1]: /2014/02/13/use-ganglia-install/
[2]: http://sourceforge.net/apps/trac/ganglia/wiki/ganglia-web-2/AuthSystem
