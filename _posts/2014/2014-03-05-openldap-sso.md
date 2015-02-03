---
layout: post
title: OpenLDAP--单点登录SSO
category: 运维
description: 记录允许使用Kerberos及OpenLDAP进行认证及登录的配置过程，包括建立NFS存放用户家目录，并配置SSH进行单点登录。
tags: ["Kerberos","OpenLDAP","SSH","NFS","单点登录","SSO"]
---

### 说明

由于[重新安装了OpenLDAP](/2014/03/01/openldap-install.html)，本篇博客也重新更新，不过相比于[上一篇](/2013/12/15/sso-with-ldap-and-kerberos.html)内容上变化不大，就当精简格式了。
上篇博文中，我们已经安装了一个Kerberized OpenLDAP，本篇主要说明如何使用OpenLDAP进行登陆认证并进行用户目录及票据转发的简单配置。
安装环境及规划:
tao0[1-4].opjasee.com，系统为Centos6.3。
tao01.opjasee.com: Kerberos服务器
tao02.opjasee.com: OpenLDAP服务器，本次开启NFS服务。
tao03.opjasee.com: 本次为登陆服务端
tao04.opjasee.com: 本次为登陆客户端

------

### 1. kerberos上添加`test`用户认证信息
上一部分我们已经建立了一个`dn: uid=test,ou=people,dc=opjasee,dc=com`的ldap条目，我们需要在kerberos添加认证信息

```sh
$ kinit admin/admin
$ kadmin 
kadmin:  addprinc test@OPJASEE.COM
```

可以使用`kinit test`及`ldapwhoami`测试一下。后续我们使用这个用户进行登录测试。

### 2. 服务端认证方式变更(tao03)
因为认证相关的很多配置文件里赤裸裸的写着`User changes will be destroyed the next time authconfig is run.`，我们也就不手工修改了，使用系统提供的接口命令`authconfig`(authconfig-tui不大适用未来使用)进行配置吧。

```sh
$ yum install pam_ldap nss-pam-ldapd
$ authconfig --savebackup=originConf
$ authconfig --enableldap --enableldapauth --enablekrb5 --enableldaptls --ldapserver="tao02.opjasee.com" --ldapbasedn="dc=opjasee,dc=com" --update
Starting nslcd:                                            [  OK  ]
```

这个命令会重写`/etc/openldap/ldap.conf`这个配置文件，经过查看该配置以及`/etc/nslcd.conf`和`/etc/pam_ldap.conf`，可以发现，虽然在Centos6上安装OpenLDAP客户端时生成的证书目录是`/etc/openldap/certs`，但是周边程序都使用`/etc/openldap/cacerts`，并且刚刚这些命令还生成了这个空目录。为了减少其他程序的配置变更，使用`certs`目录替换`cacerts`。注意，目前的`certs`目录已经导入了CA证书。

```sh
$ rmdir /etc/openldap/cacerts
$ mv /etc/openldap/certs /etc/openldap/cacerts
$ service nslcd restart # 重启nslcd读取证书 
```

此时执行`id test`应该能够看到输出为`uid=1002(test) gid=1002 groups=1002`。
切换到`tao04`上测试登陆，应该可以登陆了。可以看出还有一些问题，我们接下来进行解决。

```sh
$ ssh test@tao03.opjasee.com 
test@tao03.opjasee.com's password: 
Last login: Tue Mar  4 12:39:44 2014 from tao04.opjasee.com
Could not chdir to home directory /home/test: No such file or directory
id: cannot find name for group ID 1002
-bash-4.1$ pwd
/
```

### 3. 使用NFS给用户提供家目录
在`tao02`上配置NFS服务，正式使用需注意权限问题，`exportfs -v`命令可以查看现在目录配置。

```sh
$ yum install nfs4-acl-tools nfs-utils nfs-utils-lib
$ cat /etc/exports 
/home/nfsdir/user *(rw,no_root_squash,sync)
$ mkdir -p /home/nfsdir/user
$ service nfs start  
Starting NFS services:                                     [  OK  ]
Starting NFS quotas:                                       [  OK  ]
Starting NFS mountd:                                       [  OK  ]
Starting NFS daemon:                                       [  OK  ]
$ exportfs -v
/home/nfsdir/user
                <world>(rw,wdelay,no_root_squash,no_subtree_check)
```

在`tao03`上挂载NFS，供远程登陆的用户使用。因为[nfs-ver4使用了新的映射方法][1]，而且[autofs应该需要更多的配置][2]，为了简单起见，使用下面的命令进行挂载，暂时不配置`/etc/fstab`

```sh
$ mkdir -p /home/nfsdir/user
$ mount -t nfs -o vers=3 tao02.opjasee.com:/home/nfsdir/user /home/nfsdir/user
```

可以创建目录或文件进行测试。

### 4. 修改`test`用户属性
修改之前创建`test`用户的`test.ldif`中`homeDirectory`配置

```sh
$ cat mtest.ldif 
dn: uid=test,ou=people,dc=opjasee,dc=com
changetype: modify
replace: homeDirectory
homeDirectory: /home/nfsdir/user/test
$ kint ldapadmin
$ ldapmodify -f mtest.ldif     
```

此时查看可以发现`test`用户的属性已经更新，下面在ldap中添加`gidNumber=1002`的组。

```sh
$ cat addgroup.ldif 
dn: ou=group,dc=opjasee,dc=com
objectClass: organizationalUnit
ou: group
description: Groups

dn: cn=testgroup,ou=group,dc=opjasee,dc=com
objectClass: posixGroup
cn: testgroup
gidNumber: 1002
description: for test
$ ldapadd -f addgroup.ldif 
```

### 5. 新用户登录自动创建家目录
在`tao03`上执行

```sh
$ authconfig --enablemkhomedir --update
```

尝试从`tao04`登录，上述的几个问题都消失了，但是还是需要输入密码，接下来进行解决。

### 6. 配置SSH实现票据登录

Kerberos这个名字起源于希腊神话中地狱看门三头犬，实现kerberos验证也就需要三方参与，目前我们已经有了KDC和test用户，缺少的就是SSH的服务端，刚刚登陆时也可以从`tao03`的`/var/log/secure`日志中看到pam模块因为缺少`/etc/krb5.keytab`而无法直接使用kerberos进行认证，用户只能输入密码。通过下面的配置在`tao03`添加SSH所需的`host principal`。

```sh
$ kinit admin/admin
$ kadmin
kadmin:  addprinc -randkey host/tao03.opjasee.com@OPJASEE.COM
kadmin:  ktadd host/tao03.opjasee.com@OPJASEE.COM
```

此时`test`用户就可以使用自己的TGT向KDC请求SSH登录的票据了，TGT有效期间不需要再次输入密码。

```sh
$ kinit test
$ klist 
$ ssh test@tao03.opjasee.com
[test@tao03 ~]$ klist 
klist: No credentials cache found (ticket cache FILE:/tmp/krb5cc_1002)
```

### 7. 配置票据转发，实现单点登录
从上面的登录结果可以看出，获取TGT之后，虽然从`tao04`上登录`tao03`不再需要输入密码，但是TGT不能继承，用户在`tao03`上依然无法直接继续使用kerberized服务。通过两项配置可以实现`TGT forwarding`进行单点登录。

1. 生成`forwardable TGT`。我们已经在`/etc/krb5.conf`中配置了`forwardable = true`，所以不再需要使用`kinit`命令的`-f`选项，直接生成即可。可用`klist -f`查看`flag`。
2. 配置SSH进行转发。需要配置`/etc/ssh/ssh_config`，设定`GSSAPIDelegateCredentials yes`，在不存在`~/.ssh/config`文件的情况下，SSH客户端默认使用该配置文件。

此时登录`tao03`会生成了新的`TGT`，这样就完成了单点登录。新TGT由于不是使用`kinit`命令直接生成的，而是伴随着此次登录会话传递而来，因此本地缓存不是`/tmp/krb5cc_1002`(1002是uid)，名称中包含了会话ID，该凭据将在会话结束时消失。

### *参考文档*
*[关于 nscd，nslcd 和 sssd 套件的综述](http://webcache.googleusercontent.com/search?q=cache:yXvZKKIwyEkJ:chengkinhung.blogspot.com/2012/08/nscdnslcd-sssd.html+&cd=3&hl=zh-CN&ct=clnk&gl=cn)*
*[RHEL6配置简单LDAP服务器基于TLS加密和NFS的用户家目录自动挂载](http://blog.sina.com.cn/s/blog_64aac6750101gwst.html)*
*[NFS服务器及autofs搭建](http://blog.sina.com.cn/s/blog_5fc3a8b60100w637.html)*
*[使用OpenLDAP集中管理用户帐号](http://www.ibm.com/developerworks/cn/linux/l-openldap/)*
*[Passing Kerberos TGT (ticket-granting ticket) to remote hosts with ssh](http://blog.asteriosk.gr/2009/11/18/passing-kerberos-tgt-ticket-granting-ticket-to-remote-hosts-withopenssh/)*
*[Kerberos验证过程](http://www.cnblogs.com/xwdreamer/archive/2012/08/21/2649601.html)*


[1]:(http://www.361way.com/nfs-mount-nobody/2616.html)
[2]:(https://access.redhat.com/site/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Deployment_Guide/sssd-ldap-autofs.html)
