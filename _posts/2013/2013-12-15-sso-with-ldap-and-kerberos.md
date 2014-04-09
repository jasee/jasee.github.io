---
layout: post
title: 使用LDAP及Kerberos进行单点登录
category: 运维
description: 记录允许使用Kerberos及LDAP进行认证及登录的配置过程，包括建立NFS存放用户家目录，并配置SSH进行单点登录。
tags: ["Kerberos","LDAP","SSH","NFS","单点登录","SSO"]
---

### 说明
[统一认证平台(一):安装kerberos化的openldap](/2013/12/10/install-kerberized-ldap/)
[统一认证平台(二):使用LDAP及Kerberos进行单点登录](/2013/12/15/sso-with-ldap-and-kerberos/)

本文记录`tao03`允许使用Kerberos及LDAP进行认证及登录的配置过程，包括建立NFS存放用户家目录，并配置SSH进行单点登录。
tao01-04，系统为Centos6.3，已完成kerberized ldap搭建。
tao01: 已经安装kerberos
tao02: 已经安装openldap，本次安装NFS
tao03: 本次作为服务端
tao04: 本次作为客户端

------

### 1. kerberos上添加`test`用户认证信息
上一部分我们已经建立了一个`dn: uid=test,ou=people,dc=da,dc=adk`的ldap条目，我们需要在kerberos添加认证信息

``` sh
$ kinit admin/admin
$ kadmin 
kadmin:  addprinc test@DA.ADK
```

可以使用`kinit test`及`ldapwhoami`测试一下。后续我们使用这个用户进行登录测试。

### 2. 服务端认证方式变更(tao03)
因为认证相关的很多配置文件里赤裸裸的写着`User changes will be destroyed the next time authconfig is run.`，我们也就不手工修改了，使用系统提供的接口命令`authconfig`(authconfig-tui不大适用未来使用)进行配置吧。

``` sh
$ yum install pam_ldap nss-pam-ldapd
$ authconfig --savebackup=originConf
$ authconfig --enableldap --enableldapauth --enablekrb5 --enableldaptls --ldapserver="tao02" --ldapbasedn="dc=da,dc=adk" --update
Starting nslcd:                                            [  OK  ]
Starting oddjobd:                                          [  OK  ]
```

如果只开启LDAP的话启动的是`sssd`服务。

此时执行`id test`应该能够看到输出为`uid=1002(test) gid=1002 groups=1002`。
修改配置文件`/etc/pam_ldap.conf`以下两条：

```
host tao02
base dc=da,dc=adk
```

切换到`tao04`上测试登陆，应该可以登陆了。可以看出还有一些问题，我们接下来进行解决。

``` sh
$ ssh test@tao03
test@tao03's password: 
Could not chdir to home directory /home/test: No such file or directory
id: cannot find name for group ID 1002
-bash-4.1$ pwd
/
```
### 3. 使用NFS给用户提供家目录
在`tao02`上配置NFS服务，正式使用需注意权限问题，`exportfs -v`命令可以查看现在目录配置。

``` sh
$ yum install nfs4-acl-tools nfs-utils nfs-utils-lib
$ cat /etc/exports 
/home/nfsdir/user *(rw,no_root_squash,sync)
$ mkdir /home/nfsdir/user
$ service nfs start  
Starting NFS services:                                     [  OK  ]
Starting NFS quotas:                                       [  OK  ]
Starting NFS mountd:                                       [  OK  ]
Starting NFS daemon:                                       [  OK  ]
Starting RPC idmapd:                                       [  OK  ]
$ exportfs -v
/home/nfsdir/user
                <world>(rw,wdelay,no_root_squash,no_subtree_check)
```

在`tao03`上挂载NFS，供远程登陆的用户使用。因为[nfs-ver4使用了新的映射方法][1]，而且[autofs应该需要更多的配置][2]，为了简单起见，使用下面的命令进行挂载，暂时不配置`/etc/fstab`

``` sh
$ mkdir /home/nfsdir/user
$ mount -t nfs -o vers=3 tao02:/home/nfsdir/user /home/nfsdir/user
```

可以创建目录或文件进行测试。

### 4. 修改`test`用户属性
修改之前创建`test`用户的`test.ldif`中`homeDirectory`配置

``` sh
$ cat mtest.ldif 
dn: uid=test,ou=people,dc=da,dc=adk
changetype: modify
replace: homeDirectory
homeDirectory: /home/nfsdir/user/test
$ kint ldapadmin
$ ldapmodify -f mtest.ldif     
SASL/GSSAPI authentication started
SASL username: ldapadmin@DA.ADK
SASL SSF: 56
SASL data security layer installed.
modifying entry "uid=test,ou=people,dc=da,dc=adk"
```

此时查看可以发现`test`用户的属性已经更新，下面在ldap中添加`gidNumber=1002`的组。

``` sh
$ cat addgroup.ldif 
dn: ou=group,dc=da,dc=adk
objectClass: organizationalUnit
ou: group
description: Groups

dn: cn=testgroup,ou=group,dc=da,dc=adk
objectClass: posixGroup
cn: testgroup
gidNumber: 1002
description: for test
$ ldapadd -f addgroup.ldif 
SASL/GSSAPI authentication started
SASL username: ldapadmin@DA.ADK
SASL SSF: 56
SASL data security layer installed.
adding new entry "ou=group,dc=da,dc=adk"
adding new entry "cn=testgroup,ou=group,dc=da,dc=adk"
```

### 5. 新用户登录自动创建家目录
在`tao03`上执行

``` sh
$ authconfig --enablemkhomedir  --update
```

尝试从`tao04`登录，可以看到上述的几个问题都消失了，但是还是需要输入密码，下次再进行单点登录测试。

``` sh
$ ssh test@tao03
test@tao03's password: 
Creating home directory for test.
Last login: Sun Dec 15 15:47:33 2013 from tao04
[test@tao03 ~]$ pwd
/home/nfsdir/user/test
[test@tao03 ~]$ ll ..
total 4
drwxr-xr-x 2 test testgroup 4096 Dec 15 15:49 test
[test@tao03 ~]$ ll -a
total 28
drwxr-xr-x 2 test testgroup 4096 Dec 15 15:52 .
drwxr-xr-x 3 root root      4096 Dec 15 15:49 ..
-rw-r--r-- 1 test testgroup   18 Dec 15 15:49 .bash_logout
-rw-r--r-- 1 test testgroup  176 Dec 15 15:49 .bash_profile
-rw-r--r-- 1 test testgroup  124 Dec 15 15:49 .bashrc
-rw------- 1 test testgroup  584 Dec 15 15:52 .viminfo
```

通过上面的操作，已经能够让系统使用kerberos进行认证并从ldap中获取用户属性信息，完成登录，但是还存在两个问题。

1. 需要输入密码进行认证，每次登录输入的密码将在网络上传输至`tao03`，`tao03`以客户端的方式向kerberos验证密码是否正确，这样显然不是最安全的。
2. 每次登录都需要输入密码，无法一次登录处处通行。

下面我们将分别解决这两个问题。

### 6. 配置SSH实现票据登录

Kerberos这个名字起源于希腊神话中地狱看门三头犬，实现kerberos验证也就需要三方参与，目前我们已经有了KDC和test用户，缺少的就是SSH的服务端。通过下面的配置在`tao03`添加SSH所需的`host principal`。

``` sh
$ kinit admin/admin
$ kadmin
kadmin:  addprinc -randkey host/tao03@DA.ADK
kadmin:  ktadd host/tao03@DA.ADK
```

此时`test`用户就可以使用自己第一次登录时获得的TGT向KDC请求SSH登录的票据了，TGT有效期间不需要再次输入密码。

``` sh
$ kinit test
$ klist 
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: test@DA.ADK

Valid starting     Expires            Service principal
12/15/13 23:20:24  12/16/13 23:20:24  krbtgt/DA.ADK@DA.ADK
        renew until 12/15/13 23:20:24
$ ssh test@tao03
Last login: Sun Dec 15 23:05:12 2013 from tao03
[test@tao03 ~]$ klist 
klist: No credentials cache found (ticket cache FILE:/tmp/krb5cc_1002)
```

### 7. 配置票据转发，实现单点登录
从上面的登录结果可以看出，获取TGT之后，虽然从`tao04`上登录`tao03`不再需要输入密码，但是TGT不能继承，用户在`tao03`上依然无法直接继续使用kerberized服务。通过两项配置可以实现`TGT forwarding`进行单点登录。

1. 生成`forwardable TGT`。我们已经在`/etc/krb5.conf`中配置了`forwardable = true`，所以不再需要使用`kinit`命令的`-f`选项，直接生成即可。可用`klist -f`查看`flag`。
2. 配置SSH进行转发。需要配置`/etc/ssh/ssh_config`，设定`GSSAPIDelegateCredentials yes`，在不存在`~/.ssh/config·文件的情况下，SSH客户端默认使用该配置文件。

进行登录测试，可以看出，在登录`tao03`的时候已经生成了新的`TGT`，这样就完成了单点登录。新TGT由于不是使用`kinit`命令直接生成的，而是伴随着此次登录会话传递而来，因此本地缓存不是`/tmp/krb5cc_1002`(1002是uid)，名称中包含了会话ID，该凭据将在会话结束时消失。

``` sh
$ ssh test@tao03
Last login: Sun Dec 15 23:21:57 2013 from tao04
[test@tao03 ~]$ klist 
Ticket cache: FILE:/tmp/krb5cc_1002_zbJKf29577
Default principal: test@DA.ADK

Valid starting     Expires            Service principal
12/15/13 23:38:09  12/16/13 23:20:24  krbtgt/DA.ADK@DA.ADK
        renew until 12/15/13 23:20:24
```


### *参考文档*
*[关于 nscd，nslcd 和 sssd 套件的综述](http://webcache.googleusercontent.com/search?q=cache:yXvZKKIwyEkJ:chengkinhung.blogspot.com/2012/08/nscdnslcd-sssd.html+&cd=3&hl=zh-CN&ct=clnk&gl=cn)*
*[RHEL6配置简单LDAP服务器基于TLS加密和NFS的用户家目录自动挂载](http://blog.sina.com.cn/s/blog_64aac6750101gwst.html)*
*[NFS服务器及autofs搭建](http://blog.sina.com.cn/s/blog_5fc3a8b60100w637.html)*
*[使用OpenLDAP集中管理用户帐号](http://www.ibm.com/developerworks/cn/linux/l-openldap/)*
*[Passing Kerberos TGT (ticket-granting ticket) to remote hosts with ssh](http://blog.asteriosk.gr/2009/11/18/passing-kerberos-tgt-ticket-granting-ticket-to-remote-hosts-withopenssh/)*
*[Kerberos验证过程](http://www.cnblogs.com/xwdreamer/archive/2012/08/21/2649601.html)*


[1]:(http://www.361way.com/nfs-mount-nobody/2616.html)
[2]:(https://access.redhat.com/site/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Deployment_Guide/sssd-ldap-autofs.html)
