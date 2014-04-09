---
layout: post
title: OpenLDAP--使用sudo进行权限分配
category: 运维
description: 目前，已经可以使用Kerberos进行认证，使用OpenLDAP存储用户属性，使得服务以此允许用户登陆并设定基本属性，但是所有用户在所有机器都是同一个普通用户权限，为了实际使用，还需要引入权限差异功能。sudo可以满足该需求，通过读取存储在OpenLDAP中的配置，允许不同的用户在不同的机器使用不同的(特权)命令。
tags: ["Sudo","OpenLDAP","Kerberos"]
---

目前，已经可以使用Kerberos进行认证，使用OpenLDAP存储用户属性，使得服务以此允许用户登陆并设定基本属性，但是所有用户在所有机器都是同一个普通用户权限，为了实际使用，还需要引入权限差异功能。sudo可以满足该需求，通过读取存储在OpenLDAP中的配置，允许不同的用户在不同的机器使用不同的(特权)命令。以下是具体的配置过程。

### LDAP导入sudo schema
"Schema是LDAP的一个重要组成部分，类似于数据库的模式定义，LDAP的Schema定义了LDAP目录所应遵循的结构和规则，比如一个 objectclass会有哪些属性，这些属性又是什么结构等等，schema给LDAP服务器提供了LDAP目录中类别，属性等信息的识别方式，让这些可以被LDAP服务器识别。" -- 来自[LDAP Schema的概念和基本要素](1)
OpenLDAP的默认schema中是不包含sudo所需要的数据结构的，需要自行导入。sudo提供了配置文件`/usr/share/doc/sudo-1.8.6p3/schema.OpenLDAP`，不过使用[slapd-config](2)方式配置OpenLDAP时需要转换一下才能导入使用。
`sudoschema.ldif`内容:

```
dn: cn={12}sudo,cn=schema,cn=config
objectClass: olcSchemaConfig
cn: {12}sudo
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.1
    NAME 'sudoUser'
    DESC 'User(s) who may  run sudo'
    EQUALITY caseExactIA5Match
    SUBSTR caseExactIA5SubstringsMatch
    SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.2
    NAME 'sudoHost'
    DESC 'Host(s) who may run sudo'
    EQUALITY caseExactIA5Match
    SUBSTR caseExactIA5SubstringsMatch
    SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.3
    NAME 'sudoCommand'
    DESC 'Command(s) to be executed by sudo'
    EQUALITY caseExactIA5Match
    SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.4
    NAME 'sudoRunAs'
    DESC 'User(s) impersonated by sudo (deprecated)'
    EQUALITY caseExactIA5Match
    SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.5
    NAME 'sudoOption'
    DESC 'Options(s) followed by sudo'
    EQUALITY caseExactIA5Match
    SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.6
    NAME 'sudoRunAsUser'
    DESC 'User(s) impersonated by sudo'
    EQUALITY caseExactIA5Match
    SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.7
    NAME 'sudoRunAsGroup'
    DESC 'Group(s) impersonated by sudo'
    EQUALITY caseExactIA5Match
    SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.8
    NAME 'sudoNotBefore'
    DESC 'Start of time interval for which the entry is valid'
    EQUALITY generalizedTimeMatch
    ORDERING generalizedTimeOrderingMatch
    SYNTAX 1.3.6.1.4.1.1466.115.121.1.24 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.9
    NAME 'sudoNotAfter'
    DESC 'End of time interval for which the entry is valid'
    EQUALITY generalizedTimeMatch
    ORDERING generalizedTimeOrderingMatch
    SYNTAX 1.3.6.1.4.1.1466.115.121.1.24 )
olcAttributeTypes: ( 1.3.6.1.4.1.15953.9.1.10
    NAME 'sudoOrder'
    DESC 'an integer to order the sudoRole entries'
    EQUALITY integerMatch
    ORDERING integerOrderingMatch
    SYNTAX 1.3.6.1.4.1.1466.115.121.1.27 )
olcObjectClasses: ( 1.3.6.1.4.1.15953.9.2.1 NAME 'sudoRole' SUP top STRUCTURAL
    DESC 'Sudoer Entries'
    MUST ( cn )
    MAY ( sudoUser $ sudoHost $ sudoCommand $ sudoRunAs $ sudoRunAsUser $ sudoRunAsGroup $ sudoOption $ sudoOrder $ sudoNotBefore $ sudoNotAfter $ description )
    )
```

导入命令:

```sh
$ ldapadd -Y EXTERNAL -H ldapi:/// -f sudoschema.ldif
```
此时`/etc/openldap/slapd.d/cn=config/cn=schema`目录下应该多出了一个`cn={12}sudo.ldif`文件。

### 在OpenLDAP中建立SUDOers子树
sudoers的配置信息存放在`ou=SUDOers`的子树中，sudo首先在该子树中寻找条目`cn=defaults`，如果找到了，那么所有的`sudoOption`属性都会被解析作为全局默认值，就像是配置在`/etc/sudoers`文件中的`Defaults`语句一样。
当sudo到OpenLDAP中查询一个sudo用户时一般有两到三次查询。第一次查询解析全局配置，第二次查询匹配用户名或者用户所在的组(特殊标签`ALL`也在此次查询中匹配)，如果没有找到匹配项，则发出第三次查询，此次查询返回所有包含用户网络组的条目并检查该用户是否存在在这些组中。接下来创建OpenLDAP的SUDOers子树。
phpLDAPadmin不是很好用，还是使用LDIF文件导入吧:

```
dn: ou=SUDOers,dc=opjasee,dc=com
objectClass: top
objectClass: organizationalUnit
ou: SUDOers

dn: cn=defaults,ou=SUDOers,dc=opjasee,dc=com
objectClass: top
objectClass: sudoRole
cn: defaults
sudoOption: !visiblepw
sudoOption: always_set_home
sudoOption: env_reset
sudoOption: requiretty

dn: cn=%op,ou=SUDOers,dc=opjasee,dc=com
objectClass: top
objectClass: sudoRole
cn: %op
sudoCommand: ALL
sudoHost: ALL
sudoOption: !authenticate
sudoRunAsUser: ALL
sudoUser: %op

dn: cn=%rd,ou=SUDOers,dc=opjasee,dc=com
objectClass: top
objectClass: sudoRole
cn: %rd
sudoCommand: /bin/su test -l
sudoHost: tao03.opjasee.com
sudoOption: !authenticate
sudoRunAsUser: ALL
sudoUser: %rd
```

此时用`ldapsearch`命令应该能看到刚刚建立的条目了。
这个示例设定了op组内的用户能够在任何位置使用sudo执行任何命令，rd组内的用户只能在一台机器上使用sudo执行一条命令。

### 建立用户
建立op和rd组，并分别添加一个用户。

```
dn: cn=op,ou=group,dc=opjasee,dc=com
objectClass: posixGroup
cn: op
gidNumber: 2000

dn: cn=rd,ou=group,dc=opjasee,dc=com
objectClass: posixGroup
cn: rd
gidNumber: 2001

dn: uid=opa,ou=people,dc=opjasee,dc=com
objectclass: account
objectclass: posixAccount
objectclass: shadowAccount
cn: opa
uid: opa
uidNumber: 1100
gidNumber: 2000
homeDirectory: /home/nfsdir/user/opa
loginShell: /bin/bash

dn: uid=rda,ou=people,dc=opjasee,dc=com
objectclass: account
objectclass: posixAccount
objectclass: shadowAccount
cn: rda
uid: rda
uidNumber: 1400
gidNumber: 2001
homeDirectory: /home/nfsdir/user/opa
loginShell: /bin/bash
```

当然，还需要到Kerberos里添加账号，参考命令:`addprinc opa@OPJASEE.COM`
配置完成后，这两个账号已经能登陆服务器了，不过目前不能使用`sudo`命令，需要配置sudo读取LDAP数据。

### 配置sudo
sudo的LDAP配置文件是`/etc/sudo-ldap.conf`(sudo -V查看)，添加如下内容:

```
uri ldap://tao02.opjasee.com/
sudoers_base ou=SUDOers,dc=opjasee,dc=com
base dc=opjasee,dc=com
ssl start_tls
tls_cacertdir /etc/openldap/certs
```

sudo按照`/etc/nsswitch.conf`中的配置内容顺序读取sudoer配置，后匹配到的信息会覆盖之前匹配的，以ldap信息为准的话在`nsswitch.conf`中加入如下内容，如果需要本地配置优先的话顺序反过来就行了。

```
sudoers:    files ldap
```

至此已经完成配置，用户`opa`和`rda`分别具有预期的sudo权限。

#### *参考文档*

*[LINUX下基于LDAP集中系统用户认证系统](http://www.ttlsa.com/linux/openldap-openssh-lpk-sudo-tls-auth/)*
*[Sudoers LDAP Manual](http://www.sudo.ws/sudoers.ldap.man.html)*
*[LDAP Server and CentOS 6.3](http://www.6tech.org/2013/01/ldap-server-and-centos-6-3/)*

[1]: http://blog.csdn.net/zcGate/article/details/1922843
[2]: http://www.openldap.org/doc/admin24/slapdconf2.html
