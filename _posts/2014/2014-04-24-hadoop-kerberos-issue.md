---
layout: post
title: Hadoop多Kerberos冲突
category: 服务
description: Hadoop客户端在同时连接两个Kerberos及启用DNS后碰到的问题及绕行方案。
tags: ["Hadoop","Kerberos","DNS","OpenLDAP"]
---

Hadoop使用Kerberos果然问题多多，最近准备使用LDAP和DNS，又碰上了，记录一下吧。

### 背景
之前为了解决Hadoop的安全问题，仓促之间上了个Kerberos，使用了一个随便起的域名，记为`HADOOP.REALM.COM`吧，相关的服务器也没有使用DNS，直接shortname加hosts搞定。
近期准备使用LDAP和DNS，[前者][1]使用新域名(记为`LDAP.REALM.COM`），[后者][2]使得机器全名变成了`shortname.ldap.realm.com`，分别引起了一个问题。由于近期没有重启Hadoop的计划，也只能找方法来绕过。

### 问题及应对
如果不将`LDAP.REALM.COM`设置成Kerberos默认域，那么登陆认证就无法完成(也许有地方能配置，最好别改)。`/etc/krb5.conf`关键配置如下(Hadoop的Kerberos有冗余)：

```
[libdefaults]
    default_realm = LDAP.REALM.COM

[realms]
    LDAP.REALM.COM = {
        kdc = ldapKrbA.ldap.realm.com:88
    }
    HADOOP.REALM.COM = {
        kdc = hadoopKrbA:88
        kdc = hadoopKrbB:88
    }
```

问题一，按照这个配置，在Hadoop客户端服务器上使用principal全名分别获取两个域的TGT是没有问题的，但hadoop命令报错:`failure to login`。
由于默认域是`LDAP.REALM.COM`，执行hadoop命令时，拿着`HADOOP.REALM.COM`的TGT去找`LDAP.REALM.COM`，报错也就是理所应当的了，可以通过在`hadoop-env.sh`中加上如下的语句来告诉Hadoop去哪找Kerberos：

```sh
export HADOOP_OPTS="-Djava.security.krb5.realm=HADOOP.REALM.COM -Djava.security.krb5.kdc=hadoopKrbA:hadoopKrbB $HADOOP_OPTS"
```

问题二，上述问题解决后，DNS引起的问题又出现了，hadoop命令报错:`Fail to create credential. (63) - No service creds`。
Kerberos的认证是由三方完成的，客户端和服务端都要向Kerberos提供有效票据，使用DNS后，NameNode的机器名从`shortname`变成了`shortname.ldap.realm.com`，HADOOP的Kerberos中只有`host/shortname@HADOOP.REALM.COM`，没有`host/shortname.ldap.realm.com@HADOOP.REALM.COM`，当然也就无法完成请求服务票流程了。解决方法是在`/etc/hosts`里保留原来的NN的解析:

```
ip  shortname
```
这样在进行反解后仍然去请求`host/shortname@HADOOP.REALM.COM`。短名全名、正解反解目前还没理清，涉及认证这一块的东西有条件使用全名的直接使用全名比较好。

### 其他
上面的两个问题只是Hadoop客户端上的，从这个架势来看，Hadoop集群上想使用LDAP特别是DNS是不可能了，只能等待机会整个迁移到新Kerberos。


[1]: /2014/03/01/openldap-install.html
[2]: /2014/04/21/start-use-dns.html
