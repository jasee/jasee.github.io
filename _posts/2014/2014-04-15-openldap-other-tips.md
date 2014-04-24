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

# sudo配置
# /etc/sudo-ldap.conf中追加以下内容
uri ldap://tao02.opjasee.com/
sudoers_base ou=SUDOers,dc=opjasee,dc=com
base dc=opjasee,dc=com
ssl start_tls
tls_cacertdir /etc/openldap/cacerts
# nsswitch.conf增加以下内容
sudoers:    files ldap

# 操作审计，新增两个文件：
# 新增/etc/rsyslog.d/client.conf，写入
### begin forwarding rule ###
$WorkDirectory /var/lib/rsyslog # where to place spool files
$ActionQueueFileName fwdRule1   # unique name prefix for spool files
$ActionQueueMaxDiskSpace 1g     # 1gb space limit (use as much as possible)
$ActionQueueSaveOnShutdown on   # save messages to disk on shutdown
$ActionQueueType LinkedList     # run asynchronously
$ActionResumeRetryCount 3       # infinite retries if host is down
local1.info @@tao02.opjasee.com:514
### end of the forwarding rule ###
# 新增/etc/profile.d/client.sh，写入
export PROMPT_COMMAND='{ msg=$(history 1 | { read x y; echo $y; });logger -p local1.info "[euid=$(whoami)][$(who am i)][$(pwd)]:$msg"; }'
```

因为一些原因，没有使用NFS做用户家目录，也没有进行SSH票据转发，以上步骤不包含这两部分，其他部分参考之前的文档说明。因为时间问题，暂未将以上步骤整合成具有较高容错性的脚本。

### Centos5客户端配置
在Centos5上，上述步骤有部分需要改动，改动部分如下：

```sh
# 客户端安装及开启ldap认证需要变更
$ yum install openldap-clients krb5-workstation -y
$ yum install nss_ldap -y
$ authconfig --enableldap --enableldapauth --enablekrb5 --enableldaptls --ldapserver="tao02.opjasee.com" --ldapbasedn="dc=opjasee,dc=com" --update
$ cp /tmp/ldapCA.rfc /etc/openldap/cacerts/
$ cacertdir_rehash /etc/openldap/cacerts
$ authconfig --enablemkhomedir --update

# sudo配置需要变更
$ yum install sudo # 更新sudo版本
# 在/etc/ldap.conf中保存如下内容
uri ldap://tao02.opjasee.com/
sudoers_base ou=SUDOers,dc=opjasee,dc=com
base dc=opjasee,dc=com
ssl start_tls
tls_cacertdir /etc/openldap/cacerts
# nsswitch.conf增加以下内容
sudoers:    files ldap

# 操作审计变更
# Centos5.4上还是用syslog进行日志记录。修改/etc/sysconfig/syslog的一个参数为
SYSLOGD_OPTIONS=" -r -x -m 0"
# 在/etc/syslog.conf中追加一行
local1.info @tao02.opjasee.com
# 新增/etc/profile.d/client.sh，写入
export PROMPT_COMMAND='{ msg=$(history 1 | { read x y; echo $y; });logger -p local1.info "[euid=$(whoami)][$(who am i)][$(pwd)]:$msg"; }'
# 另外还需要修改/etc/bashrc，防止其覆盖PROMPT_COMMAND变量，找到case $TERM in语句，在case外面加一个if，如下
if [ -z "$PROMPT_COMMAND" ]; then
    case $TERM in
        ...
    esac
fi
```

`syslog`使用UDP协议发送远程日志，所以别忘了在日志服务器端打开UDP 514端口进行监听。Centos5在这方面的易用性比Centos6差好多，而且开启日志记录后偶尔有命令卡顿现象。

### sudo配置风险
如果服务器本地存在和ldap上同名的用户或用户组，那么本地用户就具有对应ldap用户同样的sudo权限。并且无需经过Kerberos认证即可使用sudo，比较危险，需要注意。

### sudo多命令或多机器配置
如果一个用户或用户组需要在不止一台服务器上拥有不止一条的sudo命令，可以使用类似下面的设置

```
# %op, SUDOers, opjasee.com
dn: cn=%op,ou=SUDOers,dc=opjasee,dc=com
objectClass: top
objectClass: sudoRole
cn: %op
sudoCommand: /bin/su test -l
sudoCommand: /bin/su work -l
sudoHost: tao03.opjasee.com
sudoHost: tao04.opjasee.com
sudoOption: !authenticate
sudoRunAsUser: ALL
sudoUser: %op
```

### 域名变更
如果所使用的域名发生变更，需要重新生成相关数据。部分说明如下：

1. `/usr/local/kerberos/var/krb5kdc/`目录下存放Kerberos数据，注意还有个点文件。
2. `/var/lib/ldap`下存放slapd的数据。
3. 可以通过删除`/etc/openldap`目录，使用`yum reinstall openldap openldap-server`可重新来获取新的证书及原始配置。
4. 客户端也可以删除`/etc/openldap`目录，使用`yum reinstall openldap`重新生成。
5. 所有principal和keytab都需要重新生成，证书需要重新导入，配置需要重做。
6. 客户端重新执行一遍更新后的`authconfig`命令即可更新ldap、nslcd等服务的相关配置。

### ssh自动补全
为了提高易用性，根据ldap的sudoer配置为每个用户做个ssh机器名补全。每个用户登陆后，使用ssh时能直接补全其具有sudo权限的机器列表。
下面是一个简单的可以工作(仅仅可以工作:-D)的脚本示例：

```sh
#!/bin/bash

init(){
    gidNumber=$(ldapsearch "uid=$USER" gidNumber -b "ou=people,dc=opjasee,dc=com" 2>/dev/null | grep -v '^#' | grep '^gidNumber' | awk '{print $2}')
    userGroup=$(ldapsearch "gidNumber=$gidNumber" cn -b "ou=group,dc=opjasee,dc=com" 2>/dev/null | grep -v '^#' | grep '^cn' | awk '{print $2}')
    sudoHosts=$(ldapsearch "(|(cn=$USER)(cn=%$userGroup))" sudoHost -b "ou=SUDOers,dc=opjasee,dc=com" 2>/dev/null | grep -v '^#' | grep '^sudoHost' | awk '{print $2}')
}

main(){
    init
    [ -e ~/.myhosts ] && rm ~/.myhosts
    touch ~/.myhosts # 粗暴手段避免本地用户报文件不存在
    for h in $sudoHosts; do
        if [ $h == "ALL" ]; then
            # 目前手动生成了全量列表放在all_hosts里，也可以考虑从hosts或DNS中定期或实时导出
            cp /etc/ssh/all_hosts ~/.myhosts
            return 0
        fi
        echo $h >> ~/.myhosts
    done
}

main
```

然后在`/etc/bashrc`中加入:

```sh
source /etc/ssh/ssh_tab.sh
complete -W "$(cat ~/.myhosts)" ssh
```

### 对现有系统的影响。
当ldap服务无法使用时，客户端本地用户登陆会出现一定的迟缓，Centos6大概需要10s，而Centos5则需要数分钟。两者重试机制不同，需关注。
