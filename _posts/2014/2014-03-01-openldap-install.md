---
layout: post
title: OpenLDAP--安装
category: 服务
tagline: Kerberized OpenLDAP的无主备安装
description: 本文记录了Kerberos及OpenLDAP的安装过程，OpenLDAP使用Kerberos及TLS。
tags: ["Kerberos","LDAP","OpenLDAP","TLS"]
---
{% include JB/setup %}

### 说明

如果一件事情做的很牵强，要么就不应该做，要么就是做的方式有问题。之前的博文[安装kerberos化的openldap](/2013/12/10/install-kerberized-ldap/)显然属于后者，本次力争使用一种顺其自然的方式重新安装。由于上篇博文一些探索性的记录尚有价值，也就没有直接更换，另起本篇进行记录。
安装环境及规划:
tao0[1-4].opjasee.com，系统为Centos6.3。
tao01.opjasee.com: 本次安装Kerberos
tao02.opjasee.com: 本次安装OpenLDAP
tao0[34].opjasee.com: 本次只作为客户端

------

## 安装Kerberos
### 1. 下载Kerberos，解压并安装，安装时最新版本是`krb5-1.11.1`

```sh
$ wget http://web.mit.edu/kerberos/dist/krb5/1.11/krb5-1.11.1-signed.tar
$ tar -xf  krb5-1.11.1-signed.tar
$ tar -zxf krb5-1.11.1.tar.gz
$ cd krb5-1.11.1
$ mkdir centos6.3
$ cd centos6.3
$ ../src/configure --prefix=/usr/local/kerberos
$ make
$ make install
```

如果遇到`verto.h: No such file or directory`问题请参考[这里][1]

### 2. 配置Kerberos，三个配置文件如下
`/etc/krb5.conf `

```
[logging]
    default = FILE:/var/log/krb5libs.log
    kdc = FILE:/var/log/krb5kdc.log
    admin_server = FILE:/var/log/kadmind.log

[libdefaults]
    default_realm = OPJASEE.COM
    allow_weak_crypto=true
    default_tgs_enctypes = aes256-cts aes128-cts des-cbc-crc des-cbc-md5 des3-hmac-sha1
    default_tkt_enctypes = aes256-cts aes128-cts des-cbc-crc des-cbc-md5 des3-hmac-sha1
    permitted_enctypes = aes256-cts aes128-cts des-cbc-crc des-cbc-md5 des3-hmac-sha1
    dns_lookup_realm = false
    dns_lookup_kdc = false
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true

[realms]
    OPJASEE.COM = {
        kdc = tao01.opjasee.com:88
        admin_server = tao01.opjasee.com:749
        default_domain = opjasee.com
    }
[domain_realm]
    .opjasee.com = OPJASEE.COM
    opjasee.com = OPJASEE.COM

[kdc]
    profile = /usr/local/kerberos/var/krb5kdc/kdc.conf 

[appdefaults]
    pam = {
        debug = false
        ticket_lifetime = 36000
        renew_lifetime = 36000
        forwardable = true
        krb4_convert = false
    }
```

`/usr/local/kerberos/var/krb5kdc/kdc.conf`

```
[kdcdefaults]
    v4_mode = nopreauth
[realms]
    OPJASEE.COM = {
        acl_file = /usr/local/kerberos/var/krb5kdc/kadm5.acl
        dict_file = /usr/share/dict/words
        admin_keytab = /usr/local/kerberos/var/krb5kdc/kadm5.keytab
    }
```

`/usr/local/kerberos/var/krb5kdc/kadm5.acl`

```
*/admin@OPJASEE.COM  *
```

### 3. 初始化KDC数据库

```sh
$ /usr/local/kerberos/sbin/kdb5_util create -r OPJASEE.COM -s
```

*出现`Loading random data`的时候另开个终端执行点消耗CPU的命令如`for x in $(seq 10000000);do s=$((s+x));done;echo $s`可以加快随机数采集。*
输入的密码一定不要忘记，这个步骤执行完`/usr/local/kerberos/var/krb5kdc`目录下会产生四个文件和一个点文件。
### 4. 添加KDC数据库管理员及生成`admin keytab`

```sh
$ /usr/local/kerberos/sbin/kadmin.local
kadmin.local:  addprinc admin/admin@OPJASEE.COM
kadmin.local:  listprincs
kadmin.local:  ktadd -k /usr/local/kerberos/var/krb5kdc/kadm5.keytab kadmin/admin@OPJASEE.COM kadmin/changepw@OPJASEE.COM
```

### 5. 启动服务查看日志

```sh
$ /usr/local/kerberos/sbin/krb5kdc
$ /usr/local/kerberos/sbin/kadmind
```

`kadmind.log`中出现`Seeding random number generator`时，这时候kadmin是无法连上的，会报`GSS-API (or Kerberos) error while initializing kadmin interface`，得等几分钟或执行一些耗CPU的命令加快随机数采集，日志刷新后方能正常连接。

### 6. Kerberos客户端配置
`Centos6.3`已经具有kerberos客户端，不需额外安装，只需要分发配置文件

```sh
$ for x in  tao0{2,3,4}; do scp /etc/krb5.conf $x:/etc/; done
```

------

## 安装OpenLDAP
### 1. 安装OpenLDAP

```sh
$ yum install openldap openldap-servers openldap-clients openldap-devel compat-openldap
```

OpenLDAP默认使用Mozilla NSS，安装后已经生成了一份证书，可使用`certutil -d /etc/openldap/certs/ -L -n 'OpenLDAP Server'`命令查看。使用如下命令生成RFC格式CA证书并分发到客户机待用。

```sh
$ certutil -d /etc/openldap/certs/ -L -a -n 'OpenLDAP Server' -f /etc/openldap/certs/password > /etc/openldap/ldapCA.rfc
$ for x in tao0{1,3,4}.opjasee.com; do scp /etc/openldap/ldapCA.rfc $x:/tmp/; done

```

*附生成自签名证书的命令供参考*

```sh
$ certutil -d /etc/openldap/certs -S -n 'test cert' -x -t 'u,u,u' -s 'C=XX, ST=Default Province, L=Default City, O=Default Company Ltd, OU=Default Unit, CN=tao02.opjasee.com' -k rsa -v 120 -f /etc/openldap/certs/password
```

### 2. 周边配置
生成ldap运行所用principal，获取`ldapadmin`的TGT待用。

```sh
$ kinit admin/admin
$ kadmin
kadmin:  addprinc ldapadmin@OPJASEE.COM
kadmin:  addprinc -randkey ldap/tao02.opjasee.com@OPJASEE.COM
kadmin:  ktadd -k /etc/openldap/ldap.keytab ldap/tao02.opjasee.com@OPJASEE.COM
$ chown ldap:ldap /etc/openldap/ldap.keytab
$ kinit ldapadmin
```

设置库配置

```sh
$ cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
```

修改`/etc/rsyslog.conf`，将slapd运行日志单独指定

```sh
$ echo 'local4.* /var/log/slapd.log' >> /etc/rsyslog.conf
$ service rsyslog restart

```
### 3. 启动OpenLDAP
修改`/etc/init.d/slapd`在对应位置加入一行

```sh
# OPTIONS, SLAPD_OPTIONS and KTB5_KTNAME are not defined
KRB5_KTNAME=/etc/openldap/ldap.keytab
```

修改`/etc/sysconfig/ldap`，开启ldaps

```sh
SLAPD_LDAPS=yes
```

使用以下命令启动slapd服务并观察`/var/log/slapd.log `日志

```sh
$ service slapd start 
```

### 4. 配置OpenLDAP
OpenLDAP从2.3版本以后默认采用动态配置引擎`slapd-config`，其数据存储位置即目录`/etc/openldap/slapd.d`。尽管该系统的数据文件是透明格式的，还是建议使用`ldapmodify`等命令来修改而不是直接编辑。
建立`setup.ldif`文件，内容如下

```
dn: olcDatabase={2}bdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=opjasee,dc=com
 
dn: olcDatabase={2}bdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: uid=ldapadmin,ou=people,dc=opjasee,dc=com

dn: cn=config
changetype: modify
add: olcAuthzRegexp
olcAuthzRegexp: uid=([^,]*),cn=GSSAPI,cn=auth uid=$1,ou=people,dc=opjasee,dc=com

dn: olcDatabase={2}bdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to dn.base="" by * read
olcAccess: {1}to * by dn="uid=ldapadmin,ou=people,dc=opjasee,dc=com" write by * read
```

使用`ldapmodify -Y EXTERNAL -H ldapi:/// -f setup.ldif`命令导入更新配置。此后只有`ldapadmin@OPJASEE.COM`具有管理员权限。
此时在`tao02.opjasee.com`上执行`ldapsearch -H ldaps://tao02.opjasee.com`应该能正常返回结果。
### 5. 客户端安装及测试
可以使用`yum install openldap-clients`安装openldap客户端，修改`/etc/openldap/lapd.conf'

```
BASE    dc=opjasee,dc=com
URI     ldaps://tao02.opjasee.com
```

导入之前生成的CA证书

```sh
$ certutil -d /etc/openldap/certs/ -A -a -n 'OpenLDAP Server' -t 'CT' -f /etc/openldap/certs/password -i /tmp/ldapCA.rfc
```

此时执行`ldapsearch`应该能正常返回，说明LDAP及TLS正常。`tao02`本身也应该安装客户端，导入证书步骤就不必了。

在`tao01.opjasee.com`上建立文件`test.ldif`并导入和查看。

```sh
$ cat test.ldif
dn: dc=opjasee,dc=com
objectclass: organization
objectclass: dcObject
objectclass: top
o: myCompany
dc: opjasee
description: root entry

dn: ou=people,dc=opjasee,dc=com
objectclass: organizationalUnit
ou: people
description: Users

dn: uid=test,ou=people,dc=opjasee,dc=com
objectclass: inetOrgPerson
objectclass: posixAccount
objectclass: shadowAccount
cn: test account
sn: test
uid: test
uidNumber: 1002
gidNumber: 1002
homeDirectory: /home/test
loginShell: /bin/bash
$ kinit ldapadmin
$ ldapadd -f test.ldif 
$ ldapsearch
```

### *参考文档*
*[OpenLDAP Software 2.4 Administrator's Guide](http://www.openldap.org/doc/admin24/OpenLDAP-Admin-Guide.pdf)*
*[使用网络安全服务 (NSS) 工具](http://docs.oracle.com/cd/E19900-01/820-0847/ablrf/index.html)*
*[How do I use TLS/SSL with Mozilla NSS](http://www.openldap.org/faq/data/cache/1514.html)*
*[open ldap 的配置](http://zhoujian1982318.iteye.com/blog/1732161)*

[1]: https://forum.openwrt.org/viewtopic.php?id=45963
