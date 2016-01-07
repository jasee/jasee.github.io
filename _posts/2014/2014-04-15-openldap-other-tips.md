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
# 重启rsyslog
$ service rsyslog restart
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
# 重启syslog
$ service syslog restart
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

### Centos5/6客户端部署脚本
使用脚本确实能大大加快速度，这个脚本安全性一般，我测试了多台服务器没啥问题，使用时需注意自己环境。另外，由于我的环境中还有另外一个Kerberos，因此SSH Service Principal及Keytab生成部分仍手工执行，防止覆盖现有环境。

```sh
#!/usr/bin/env bash

NORMAL=$(tput sgr0)
GREEN=$(tput setaf 2)
YELLOW=$(tput setaf 3)
RED=$(tput setaf 1; tput bold)

function red() {
        echo -e "$RED$*$NORMAL"
}

function green() {
        echo -e "$GREEN$*$NORMAL"
}

function yellow() {
        echo -e "$YELLOW$*$NORMAL"
}

function get_system_release_info() {
    local thisSystem
    if grep 'CentOS release 5' /etc/issue > /dev/null; then
        thisSystem='CentOS5'
    elif grep 'CentOS release 6' /etc/issue > /dev/null; then
        thisSystem='CentOS6'
    else
        thisSystem='other'
    fi
    echo $thisSystem
}

function install_on_centos6() {
    local backupPath="$1"
    green "***** Enable LDAP on CentOS 6 *****"
    green "===== Get basic files"
    wget http://tao02.opjasee.com/ldap/ldapCA.rfc || { red 'Can not get ldapCA.rfc'; exit 1; }
    wget http://tao02.opjasee.com/ldap/krb5.conf || { red 'Can not get krb5.conf'; exit 1; }
    wget http://tao02.opjasee.com/ldap/client.sh || { red 'Can not get client.sh'; exit 1; }
    [ -f /etc/krb5.conf ] && cp /etc/krb5.conf $backupPath
    cp krb5.conf /etc/

    green "===== Install packages"
    yum install -y openldap-clients pam_ldap nss-pam-ldapd || { red 'Install error'; exit 1; }

    green "===== Enable LDAP auth"
    authconfig --savebackup=originConf
    authconfig --enableldap --enableldapauth --enablekrb5 --enableldaptls --ldapserver="tao02.opjasee.com" --ldapbasedn="dc=opjasee,dc=com" --update || { red 'authconfig error'; exit 1; }
    rmdir /etc/openldap/cacerts && mv /etc/openldap/certs /etc/openldap/cacerts || { red 'CA dir error'; exit 1; }
    certutil -d /etc/openldap/cacerts/ -A -a -n 'OpenLDAP Server' -t 'CT' -f /etc/openldap/cacerts/password -i ldapCA.rfc || { red 'Import CA error'; exit 1; }
    service nslcd restart
    authconfig --enablemkhomedir --update

    green "===== Config sudo"
    cp /etc/sudo-ldap.conf $backupPath
    cp /etc/nsswitch.conf $backupPath
    cat << EOF >> /etc/sudo-ldap.conf
uri ldap://tao02.opjasee.com
sudoers_base ou=SUDOers,dc=opjasee,dc=com
base dc=opjasee,dc=com
ssl start_tls
tls_cacertdir /etc/openldap/cacerts
EOF
    echo "sudoers:    files ldap" >> /etc/nsswitch.conf

    green "===== Set log audit"
    cat << EOF >> /etc/rsyslog.d/client.conf
### begin forwarding rule ###
$WorkDirectory /var/lib/rsyslog # where to place spool files
$ActionQueueFileName fwdRule1   # unique name prefix for spool files
$ActionQueueMaxDiskSpace 1g     # 1gb space limit (use as much as possible)
$ActionQueueSaveOnShutdown on   # save messages to disk on shutdown
$ActionQueueType LinkedList     # run asynchronously
$ActionResumeRetryCount 3       # infinite retries if host is down
local1.info @@tao02.opjasee.com:514
### end of the forwarding rule ###
EOF
    service rsyslog restart
    cat client.sh >> /etc/profile.d/client.sh 

    green "***** Almost OK *****"
    yellow "Consider of kerberos conflicts, following kerberos cmds shoule be exc manually now"
    yellow "BE CAREFUL WITH KINIT. DON'T CONVER KERBEROS TGT!"
    yellow "addprinc -randkey host/${HOSTNAME}.opjasee.com@OPJASEE.COM"
    yellow "ktadd host/${HOSTNAME}.opjasee.com@OPJASEE.COM"
}

function install_on_centos5() {
    local backupPath="$1"
    green "***** Enable LDAP on CentOS 5 *****"
    green "===== Get basic files"
    wget http://tao02.opjasee.com/ldap/ldapCA.rfc || { red 'Can not get ldapCA.rfc'; exit 1; }
    wget http://tao02.opjasee.com/ldap/krb5.conf || { red 'Can not get krb5.conf'; exit 1; }
    wget http://tao02.opjasee.com/ldap/client.sh || { red 'Can not get client.sh'; exit 1; }
    [ -f /etc/krb5.conf ] && cp /etc/krb5.conf $backupPath
    cp krb5.conf /etc/

    green "===== Install packages"
    yum install -y openldap-clients krb5-workstation nss_ldap sudo || { red 'Install error'; exit 1; }

    green "===== Enable LDAP auth"
    authconfig --savebackup=originConf
    authconfig --enableldap --enableldapauth --enablekrb5 --enableldaptls --ldapserver="tao02.opjasee.com" --ldapbasedn="dc=opjasee,dc=com" --update || { red 'authconfig error'; exit 1; }
    cp ldapCA.rfc /etc/openldap/cacerts && cacertdir_rehash /etc/openldap/cacerts || { red 'Import CA error'; exit 1; }
    authconfig --enablemkhomedir --update

    green "===== Config sudo"
    cp /etc/ldap.conf $backupPath
    cp /etc/nsswitch.conf $backupPath
    cat << EOF >> /etc/ldap.conf
uri ldap://tao02.opjasee.com
sudoers_base ou=SUDOers,dc=opjasee,dc=com
base dc=opjasee,dc=com
ssl start_tls
tls_cacertdir /etc/openldap/cacerts
EOF
    echo "sudoers:    files ldap" >> /etc/nsswitch.conf

    green "===== Set log audit"
    cp /etc/sysconfig/syslog $backupPath
    cp /etc/syslog.conf $backupPath
    sed -i 's/SYSLOGD_OPTIONS="-m 0"/SYSLOGD_OPTIONS=" -r -x -m 0"/' /etc/sysconfig/syslog
    echo 'local1.info @tao02.opjasee.com' >> /etc/syslog.conf
    service syslog restart
    cat client.sh >> /etc/profile.d/client.sh 

    green "***** Almost OK *****"
    yellow 'You need modify /etc/bashrc(backuped),see https://XXX' # 见本文Centos5客户端安装说明部分
    cp /etc/bashrc $backupPath
    yellow "Consider of kerberos conflicts, following kerberos cmds shoule be exc manually now"
    yellow "BE CAREFUL WITH KINIT. DON'T CONVER KERBEROS TGT!"
    yellow "addprinc -randkey host/${HOSTNAME}.opjasee.com@OPJASEE.COM"
    yellow "ktadd host/${HOSTNAME}.opjasee.com@OPJASEE.COM"
}

[ $EUID = 0 ] || { red 'You shoule be root'; exit 1; }
workPath=/tmp/ldap.$(date +%Y%m%d)
backupPath=/tmp/ldap.$(date +%Y%m%d).bak
[ -d $workPath ] || mkdir -p $workPath
[ -d $backupPath ] || mkdir -p $backupPath
cd $workPath
thisSystem=$(get_system_release_info)
if [ $thisSystem = 'CentOS5' ]; then
    install_on_centos5 "$backupPath"
elif [ $thisSystem = 'CentOS6' ]; then
    install_on_centos6 "$backupPath"
else
    red "Not CentOS5 or CentOS6"
fi
```

### Slave Kerberos

1. 按照之前主的Kerberos安装方法安装Kerberos。
2. 从主Kerberos上复制以下文件到从机上:
    * /etc/krb5.conf
    * /usr/local/kerberos/var/krb5kdc/kdc.conf
    * /usr/local/kerberos/var/krb5kdc/kadm5.acl
    * /usr/local/kerberos/var/krb5kdc/.k5.OPJASEE.COM
3. 主从上增加host principal，参考命令:

    ```
    addprinc -randkey host/tao01.opjasee.com
    addprinc -randkey host/tao02.opjasee.com
    ktadd host/tao01.opjasee.com # tao01上
    ktadd host/tao02.opjasee.com # tao02上
    ```

4. 主从上增加`/usr/local/kerberos/var/krb5kdc/kpropd.acl`文件，内含以下内容:

    ```
    host/tao01.opjasee.com@OPJASEE.COM
    host/tao02.opjasee.com@OPJASEE.COM
    ```

    然后在主从上都启动kpropd: `/usr/local/kerberos/sbin/kpropd`
5. 同步测试，在主上执行以下命令:

    ```sh
    $ /usr/local/kerberos/sbin/kdb5_util dump /usr/local/kerberos/var/krb5kdc/slave_datatrans
    $ /usr/local/kerberos/sbin/kprop -f /usr/local/kerberos/var/krb5kdc/slave_datatrans tao02.opjasee.com
    ```

    返回`SUCCEEDED`就说明OK了。

6. 定时同步，将上面的命令写在脚本里定时调度即可:

    ```sh
    #!/usr/bin/env bash
    slaves="tao02.opjasee.com"
    /usr/local/kerberos/sbin/kdb5_util dump /usr/local/kerberos/var/krb5kdc/slave_datatrans

    for slave in $slaves
    do
        /usr/local/kerberos/sbin/kprop -f /usr/local/kerberos/var/krb5kdc/slave_datatrans $slave
    done
    ```

    一切没问题后，从机上开启kdc，可以修改krb5.conf进行测试。

### Slave OpenLDAP

1. 与主机一样，使用yum安装OpenLDAP。
2. 将主机以下文件或目录复制到从机上:
    * /etc/openldap
    * /etc/rsyslog.conf
    * /etc/logrotate.conf
    * /etc/init.d/slapd
    * /etc/sysconfig/ldap
    * /etc/rsyslog.d/server.conf

    需要注意保持权限一致。重启`rsyslog`。
3. 数据同步。OpenLDAP提供了多种高大上的主从方案来应对复杂场景的需求，但是我们只需要一个冷备即可，因此定期同步`/var/lib/ldap`目录就行了。
4. 切换步骤:
    1. 修改DNS，将tao01.opjasee.com指向从机地址。
    2. 在从机上执行`hostname tao01.opjasee.com`。由于使用了证书等东西，这一步不可缺少。
    3. 从机上启动slapd。
    4. 客户端(r)syslog可能需要重启，以便进行操作记录传输。在CentOS 5.4上，syslog需重启才能使用新IP，CentOS 6.3的rsyslog未测试。

以上Kerberos的热备和LDAP的冷备已经足够我们应付主机暂时异常的需求了。不过主机要是再也恢复不了的话，将备机变成主机可能还得进一步调整，忧桑。

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
    [ -e ~/.myhosts ] && rm -f ~/.myhosts
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

### 欢迎登陆
既然所有人都通过一台机器登陆，那么提供一个欢迎界面也开始有价值了。前两天看到一个图片，我也跟着画了一个，如下(这里的添加注释符号仅为消除语法颜色)：

```
#                             _oo0oo_
#                            o8888888o
#                            88" . "88
#                            (| -_- |)
#                            0\  =  /0
#                         ____/`---`\___
#                       .`  \\|     |//  `.
#                      /  \\|||  :  |||// \
#                     /  _||||| -:- |||||- \
#                     |   | \\\  -  /// |   |
#                     | \_|  ''\---/''  |   |
#                     \  .-\__  '-'  ___/-. /
#                   ___`. .`  /--.--\  `. . ___
#                ."" '<  `.___\_<|>_/___.'  >'""
#               | | :  `- \`.;`\ _ /`;.`/ - ` : | |
#               \  \ `-.   \_ __\ /__ _/   .-` /  /
#          ======`-.____`-.___\_____/___.=`____.-`======
#                             `=---=`
#          
#          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
#                     佛祖保佑       永不死机
#                     心无外法       法外无心
```

把这个放到`/etc/motd`中，每个用户登陆时就能看见啦。
说点正经的，有个命令叫`figlet`，可以很方便的生成字符图形，使用`yum install figlet`安装，使用效果如下(这里的添加注释符号仅为消除语法颜色)：

```sh
$ figlet Welcome !
#__        __   _                            _ 
#\ \      / /__| | ___ ___  _ __ ___   ___  | |
# \ \ /\ / / _ \ |/ __/ _ \| '_ ` _ \ / _ \ | |
#  \ V  V /  __/ | (_| (_) | | | | | |  __/ |_|
#   \_/\_/ \___|_|\___\___/|_| |_| |_|\___| (_)
```

不过使用`/etc/motd`有一个缺点，就是输出只有一个颜色，比较单调。为了拥有更多的色彩，可以将类似如下的脚本添加到`/etc/profile.d/`目录下。:

```sh
#!/bin/bash
NORMAL=$(tput sgr0)
GREEN=$(tput setaf 2)
YELLOW=$(tput setaf 3)
RED=$(tput setaf 1; tput bold)

function red() {
        echo -e "$RED$*$NORMAL"
}

function green() {
        echo -e "$GREEN$*$NORMAL"
}

function yellow() {
        echo -e "$YELLOW$*$NORMAL"
}

green "Welcome to relay server!"
red "注意事项!"
yellow "以上是测试内容。"
```

注：`/etc/profile.d/`的脚本输出在`/etc/motd`之后。

### 审计日志切割
可以通过在`/etc/logrotate.conf`中增加以下配置来完成。以下配置对`/var/log/client`进行每日切割并保存三年。

```
/var/log/client {
    daily
    create 0600 root root
    rotate 1095
    postrotate
    /bin/kill -HUP `cat /var/run/syslogd.pid 2> /dev/null` 2> /dev/null || true
    endscript
}
```

另外，`/var/log/slapd.log`也最好加到`/etc/logrotate.d/syslog`中，避免时间久了后变得太大。

### 对现有系统的影响。
当ldap服务无法使用时，客户端本地用户登陆会出现一定的迟缓，Centos6大概需要10s，而Centos5则需要数分钟。两者重试机制不同，需关注。
