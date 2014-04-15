---
layout: post
title: OpenLDAP--其他
category: 运维
description: 目前整个认证系统的基本框架已经有了，本文记录一些其他安装使用上的零散的或补充性的说明。
tags: ["OpenLDAP","Kerberos","SSH","Rsyslog"]
---

目前整个认证系统的基本框架已经有了，本文记录一些其他安装使用上的零散的或补充性的说明。

### 多Kerberos时客户端配置
有时候一台服务器需要连接到多个Kerberos服务器，`/etc/krb5.conf`对应关键配置类似:

```
[libdefaults]
    default_realm = A.COM

[realms]
    A.COM = {
        kdc = ldapA:88
        admin_server = ldapA:749
        default_domain = a.com
    }
    B.COM = {
        kdc = ldapB:88
        admin_server = ldapB:749
        default_domain = b.com
    }

[domain_realm]
    .a.com = A.COM
    a.com = A.COM
    .b.com = B.COM
    b.com = B.COM
```

使用kinit之类的命令时，如果没有指定域，则连接默认域的Kerberos，否则使用指定域的配置。

### 客户端配置步骤总结
为了测试及说明，之前的记录较为混乱，以下将一台服务器加入OpenLDAP认证体系客户端的具体过程：

```sh
# 先分发krb5.conf及ldapCA.rfc，步骤略
# 客户端安装及开启ldap认证
$ yum install openldap-clients -y
$ yum install pam_ldap nss-pam-ldapd -y
$ authconfig --enableldap --enableldapauth --enablekrb5 --enableldaptls --ldapserver="tao02.opjasee.com" --ldapbasedn="dc=opjasee,dc=com" --update
$ rmdir /etc/openldap/cacerts # 变更证书位置，适应周边配置
$ mv /etc/openldap/certs /etc/openldap/cacerts
$ certutil -d /etc/openldap/cacerts/ -A -a -n 'OpenLDAP Server' -t 'CT' -f /etc/openldap/cacerts/password -i /tmp/ldapCA.rfc
$ service nslcd restart
$ authconfig --enablemkhomedir --update
# SSH聚合，参考kadmin语句如下：
addprinc -randkey host/tao03.opjasee.com@OPJASEE.COM
ktadd host/tao03.opjasee.com@OPJASEE.COM
# 操作审计，新增两个文件：
$ cat /etc/rsyslog.d/client.conf
### begin forwarding rule ###
$WorkDirectory /var/lib/rsyslog # where to place spool files
$ActionQueueFileName fwdRule1   # unique name prefix for spool files
$ActionQueueMaxDiskSpace 1g     # 1gb space limit (use as much as possible)
$ActionQueueSaveOnShutdown on   # save messages to disk on shutdown
$ActionQueueType LinkedList     # run asynchronously
$ActionResumeRetryCount 3       # infinite retries if host is down
local1.info @@tao02.opjasee.com:514
### end of the forwarding rule ###
$ cat /etc/profile.d/client.sh
export PROMPT_COMMAND='{ msg=$(history 1 | { read x y; echo $y; });logger -p local1.info "[euid=$(whoami)][$(who am i)][$(pwd)]:$msg"; }'
```

因为一些原因，没有使用NFS做用户家目录，也没有进行SSH票据转发，以上步骤不包含这两部分，其他部分参考之前的文档说明。因为时间问题，暂未将以上步骤整合成具有较高容错性的脚本。
