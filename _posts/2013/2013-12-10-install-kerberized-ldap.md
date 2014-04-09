---
layout: post
title: 安装kerberos化的openldap
category: 运维
description: 本文记录了kerberos及openldap的安装过程，openldap使用kerberos及TLS。
tags: ["Kerberos","OpenLDAP","TLS"]
---

### 说明
[统一认证平台(一):安装kerberos化的openldap](/2013/12/10/install-kerberized-ldap/)
[统一认证平台(二):使用LDAP及Kerberos进行单点登录](/2013/12/15/sso-with-ldap-and-kerberos/)

本文记录了kerberos及openldap的安装过程，openldap使用kerberos及TLS。
tao01-04，系统为Centos6.3，已完成hosts映射及root账号信任关系。
tao01: 本次安装kerberos
tao02: 本次安装openldap
tao03、04: 本次只作为客户端

------

## 安装kerberos(tao01)
### 1. 下载kerberos，解压并安装，安装时最新版本是`krb5-1.11.1`

``` sh
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

### 2. 配置kerberos，三个配置文件如下
`/etc/krb5.conf `

```
[logging]
    default = FILE:/var/log/krb5libs.log
    kdc = FILE:/var/log/krb5kdc.log
    admin_server = FILE:/var/log/kadmind.log

[libdefaults]
    default_realm = DA.ADK
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
    DA.ADK = {
        kdc = tao01:88
        admin_server = tao01:749
        default_domain = da.adk
    }
[domain_realm]
    .da.adk = DA.ADK
    da.adk = DA.ADK

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
    DA.ADK = {
        acl_file = /usr/local/kerberos/var/krb5kdc/kadm5.acl
        dict_file = /usr/share/dict/words
        admin_keytab = /usr/local/kerberos/var/krb5kdc/kadm5.keytab
    }
```

`/usr/local/kerberos/var/krb5kdc/kadm5.acl`

```
*/admin@DA.ADK  *
```

### 3. 初始化KDC数据库

``` sh
$ /usr/local/kerberos/sbin/kdb5_util create -r DA.ADK -s
```

*出现`Loading random data`的时候另开个终端执行点消耗CPU的命令如`for x in $(seq 10000000);do s=$((s+x));done;echo $s`可以加快随机数采集。*
输入的密码一定不要忘记，这个步骤执行完`/usr/local/kerberos/var/krb5kdc`目录下会产生四个文件和一个点文件。
### 4. 添加kdc数据库管理员及生成admin_keytab

``` sh
$ /usr/local/kerberos/sbin/kadmin.local
kadmin.local:  addprinc admin/admin@DA.ADK
kadmin.local:  listprincs
kadmin.local:  ktadd -k /usr/local/kerberos/var/krb5kdc/kadm5.keytab kadmin/admin@DA.ADK kadmin/changepw@DA.ADK
```

### 5. 启动服务查看日志

``` sh
$ /usr/local/kerberos/sbin/krb5kdc
$ /usr/local/kerberos/sbin/kadmind
$ cat /var/log/krb5kdc.log
$ cat /var/log/kadmind.log
```

`kadmind.log`中出现`Seeding random number generator`时，这时候kadmin是无法连上的，会报`GSS-API (or Kerberos) error while initializing kadmin interface`，得等几分钟或执行一些耗CPU的命令加快随机数采集，日志刷新后方能正常连接。

### 6. kerberos客户端配置
`Centos6.3`已经具有kerberos客户端，不需额外安装，只需要分发配置文件

``` sh
$ for x in  tao0{2,3,4}; do scp /etc/krb5.conf $x:/etc/; done
```

------

## 安装openldap(tao02)
### 1. 下载openldap并解压安装，安装时最新版本是`openldap-2.4.38`

``` sh
$ yum install db4 db4-utils db4-devel cyrus-sasl*
$ wget ftp://ftp.openldap.org/pub/OpenLDAP/openldap-release/openldap-2.4.38.tgz
$ tar -zxf openldap-2.4.38.tgz
$ cd openldap-2.4.38
$ ./configure --prefix=/usr/local/openldap --with-cyrus-sasl
$ make depend
$ make
$ make install
$ 
```
### 2. 配置TLS证书
使用`yum install openssl`后，修改`/etc/pki/tls/openssl.cnf`，前后diff结果如下：

```
50c50
< certificate   = $dir/cacert.pem       # The CA certificate
---
> certificate   = $dir/CA.crt           # The CA certificate
55c55
< private_key   = $dir/private/cakey.pem# The private key
---
> private_key   = $dir/private/CA.key# The private key
135c135
< #stateOrProvinceName_default  = Default Province
---
> stateOrProvinceName_default   = Default Province
148c148
< #organizationalUnitName_default       =
---
> organizationalUnitName_default        = Default Unit
```
使用以下一系列命令生成CA根证书及ldap的TLS证书，注意CN部分一定是ldap所在服务器全名。

``` sh
$ cd /etc/pki/CA
$ touch index.txt
$ echo 01 > serial
$ (umask 077; openssl genrsa -out private/CA.key)
Generating RSA private key, 1024 bit long modulus
..++++++
..........++++++
e is 65537 (0x10001)
# openssl req -new -x509 -key private/CA.key > CA.crt
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:
State or Province Name (full name) [Default Province]:
Locality Name (eg, city) [Default City]:
Organization Name (eg, company) [Default Company Ltd]:
Organizational Unit Name (eg, section) [Default Unit]:
Common Name (eg, your name or your server's hostname) []:tao02
Email Address []:
$ openssl genrsa -out ldap.key
Generating RSA private key, 1024 bit long modulus
.....++++++
.........++++++
e is 65537 (0x10001)
$ openssl req -new -key ldap.key -out ldap.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:
State or Province Name (full name) [Default Province]:
Locality Name (eg, city) [Default City]:
Organization Name (eg, company) [Default Company Ltd]:
Organizational Unit Name (eg, section) [Default Unit]:
Common Name (eg, your name or your server's hostname) []:tao02
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
$ mkdir /usr/local/openldap/etc/openldap/certs
$ cp ldap.csr /usr/local/openldap/etc/openldap/certs/; cd /usr/local/openldap/etc/openldap/certs
$ openssl ca -in ldap.csr -out ldap.crt
Using configuration from /etc/pki/tls/openssl.cnf
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number: 1 (0x1)
        Validity
            Not Before: Dec 13 09:31:16 2013 GMT
            Not After : Dec 13 09:31:16 2014 GMT
        Subject:
            countryName               = XX
            stateOrProvinceName       = Default Province
            organizationName          = Default Company Ltd
            organizationalUnitName    = Default Unit
            commonName                = tao02
        X509v3 extensions:
            X509v3 Basic Constraints: 
                CA:FALSE
            Netscape Comment: 
                OpenSSL Generated Certificate
            X509v3 Subject Key Identifier: 
                CC:88:38:0C:93:22:75:C8:BC:A2:7D:BD:13:3D:62:BF:B9:8E:39:A5
            X509v3 Authority Key Identifier: 
                keyid:29:F4:55:62:E6:EC:15:39:99:59:90:0F:1A:CD:D7:40:47:DA:96:54

Certificate is to be certified until Dec 13 09:31:16 2014 GMT (365 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
$ cp /etc/pki/CA/CA.crt .
$ cp /etc/pki/CA/ldap.key .
$ rm ldap.csr 
```

### 3. 配置openldap
生成ldap运行所用principal

``` sh
$ kinit admin/admin
$ kadmin
kadmin:  addprinc ldapadmin@DA.ADK
kadmin:  addprinc -randkey ldap/tao02@DA.ADK
kadmin:  ktadd -k /usr/local/openldap/etc/openldap/ldap.keytab ldap/tao02@DA.ADK
```

将tao01上`krb5-1.11.1/src/plugins/kdb/ldap/libkdb_ldap/kerberos.schema`拷贝到`/usr/local/openldap/etc/openldap/schema/`目录下。
修改`/usr/local/openldap/etc/openldap/slapd.conf`配置，模板都加上，省的有些类型条目添加不了。与默认配置diff结果如下

```
5a6,23
> include         /usr/local/openldap/etc/openldap/schema/kerberos.schema
> include         /usr/local/openldap/etc/openldap/schema/cosine.schema
> include         /usr/local/openldap/etc/openldap/schema/inetorgperson.schema
> include         /usr/local/openldap/etc/openldap/schema/openldap.schema
> include         /usr/local/openldap/etc/openldap/schema/collective.schema
> include         /usr/local/openldap/etc/openldap/schema/corba.schema
> include         /usr/local/openldap/etc/openldap/schema/duaconf.schema
> include         /usr/local/openldap/etc/openldap/schema/dyngroup.schema
> include         /usr/local/openldap/etc/openldap/schema/java.schema
> include         /usr/local/openldap/etc/openldap/schema/misc.schema
> include         /usr/local/openldap/etc/openldap/schema/nis.schema
> include         /usr/local/openldap/etc/openldap/schema/pmi.schema
> include         /usr/local/openldap/etc/openldap/schema/ppolicy.schema
> 
> # this section rewrites principals as needed for Kerberos authentication
> sasl-regexp
>      uid=(.*),cn=GSSAPI,cn=auth
>      uid=$1,ou=people,dc=da,dc=adk
7a26,32
> # Everyone can read everything
> access to dn.base="" by * read
> 
> # The admin dn has full write access
> access to *
>         by dn="uid=ldapadmin,ou=people,dc=da,dc=adk" write
>         by * read
15a41,45
> TLSCACertificateFile  /usr/local/openldap/etc/openldap/certs/CA.crt
> TLSCertificateFile    /usr/local/openldap/etc/openldap/certs/ldap.crt
> TLSCertificateKeyFile /usr/local/openldap/etc/openldap/certs/ldap.key
> TLSVerifyClient               never
> 
54,55c84,85
< suffix                "dc=my-domain,dc=com"
< rootdn                "cn=Manager,dc=my-domain,dc=com"
---
> suffix "dc=da,dc=adk"
> rootdn "cn=admin,dc=da,dc=adk"
65a96,100
> index uid             eq
> index uidNumber       eq
> index uniqueMember    eq
> index gidNumber       eq
> index memberUid       eq
```

设置库配置

``` sh
$ cp /usr/local/openldap/etc/openldap/DB_CONFIG.example /usr/local/openldap/var/openldap-data/DB_CONFIG
```

修改`/etc/rsyslog.conf`，将slapd运行日志单独指定

``` sh
$ echo 'local4.* /var/log/slapd.log' >> /etc/rsyslog.conf
$ service rsyslog restart

```
### 4. 启动openldap
从其他地方找了一个启动脚本，修改保存为`/etc/init.d/slapd`，正式环境需要建立ldap账号而不是使用root。内容如下

``` sh
#!/bin/bash

# Source function library.
. /etc/init.d/functions

# Define default values of options allowed in /etc/sysconfig/ldap
SLAPD_LDAP="yes"
SLAPD_LDAPI="no"
SLAPD_LDAPS="no"
SLAPD_URLS=""
SLAPD_SHUTDOWN_TIMEOUT=3
# OPTIONS, SLAPD_OPTIONS and KTB5_KTNAME are not defined
# OPTIONS, SLAPD_OPTIONS and KRB5_KTNAME are not defined
KRB5_KTNAME=/usr/local/openldap/etc/openldap/ldap.keytab

# Source an auxiliary options file if we have one
if [ -r /etc/sysconfig/ldap ] ; then
        . /etc/sysconfig/ldap
fi

slapd=/usr/local/openldap/libexec/slapd 
slaptest=/usr/local/openldap/sbin/slaptest
lockfile=/var/lock/subsys/slapd
configfile=/usr/local/openldap/etc/openldap/slapd.conf
pidfile=/var/run/slapd.pid
slapd_pidfile=/usr/local/openldap/var/run/slapd.pid

RETVAL=0

#
# Pass commands given in $2 and later to "test" run as user given in $1.
#
function testasuser() {
        local user= cmd=
        user="$1"
        shift
        cmd="$@"
        if test x"$user" != x ; then
                if test x"$cmd" != x ; then
                        /sbin/runuser -f -m -s /bin/sh -c "test $cmd" -- "$user"
                else
                        false
                fi
        else
                false
        fi
}

#
# Check for read-access errors for the user given in $1 for a service named $2.
# If $3 is specified, the command is run if "klist" can't be found.
#
function checkkeytab() {
        local user= service= klist= default=
        user="$1"
        service="$2"
        default="${3:-false}"
        if test -x /usr/kerberos/bin/klist ; then
                klist=/usr/kerberos/bin/klist
        elif test -x /usr/bin/klist ; then
                klist=/usr/bin/klist
        fi
        KRB5_KTNAME="${KRB5_KTNAME:-/etc/krb5.keytab}"
        export KRB5_KTNAME
        if test -s "$KRB5_KTNAME" ; then
                if test x"$klist" != x ; then
                        if LANG=C $klist -k "$KRB5_KTNAME" | tail -n 4 | awk '{print $2}' | grep -q ^"$service"/ ; then
                                if ! testasuser "$user" -r ${KRB5_KTNAME:-/etc/krb5.keytab} ; then
                                        true
                                else
                                        false
                                fi
                        else
                                false
                        fi
                else
                        $default
                fi
        else
                false
        fi
}

function configtest() {
        local user= ldapuid= dbdir= file=
        # Check for simple-but-common errors.
        user=root
        prog=`basename ${slapd}`
        ldapuid=`id -u $user`
        # Unaccessible database files.
        dbdirs=""
        if [ -f $configfile ]; then
                        dbdirs=`LANG=C egrep '^directory[[:space:]]+' $configfile | sed 's,^directory[[:space:]]*,,'`
        else
                exit 6
        fi
        for dbdir in $dbdirs; do
                if [ ! -d $dbdir ]; then
                        exit 6
                fi
                for file in `find ${dbdir}/ -not -uid $ldapuid -and \( -name "*.dbb" -or -name "*.gdbm" -or -name "*.bdb" -or -name "__db.*" -or -name "log.*" -or -name alock \)` ; do
                        echo -n $"$file is not owned by \"$user\"" ; warning ; echo
                done
                if test -f "${dbdir}/DB_CONFIG"; then
                        if ! testasuser $user -r "${dbdir}/DB_CONFIG"; then
                                file=DB_CONFIG
                                echo -n $"$file is not readable by \"$user\"" ; warning ; echo
                        fi
                fi
        done
        # Unaccessible keytab with an "ldap" key.
        if checkkeytab $user ldap ; then
                file=${KRB5_KTNAME:-/etc/krb5.keytab}
                echo -n $"$file is not readable by \"$user\"" ; warning ; echo
        fi
        # Check the configuration file.
        slaptestout=`/sbin/runuser -m -s "$slaptest" -- "$user" "-u" 2>&1`
        slaptestexit=$?
#       slaptestout=`echo $slaptestout 2>/dev/null | grep -v "config file testing succeeded"`
        # print warning if slaptest passed but reports some problems
        if test $slaptestexit == 0 ; then
                if echo "$slaptestout" | grep -v "config file testing succeeded" >/dev/null ; then
                        echo -n $"Checking configuration files for $prog: " ; warning ; echo
                        echo "$slaptestout"
                fi
        fi
        # report error if configuration file is wrong
        if test $slaptestexit != 0 ; then
                echo -n $"Checking configuration files for $prog: " ; failure ; echo
                echo "$slaptestout"
                if /sbin/runuser -m -s "$slaptest" -- "$user" "-u" > /dev/null 2> /dev/null ; then
                        #dirs=`LANG=C egrep '^directory[[:space:]]+[[:print:]]+$' $configfile | awk '{print $2}'`
                        for directory in $dbdirs ; do
                                if test -r $directory/__db.001 ; then
                                        echo -n $"stale lock files may be present in $directory" ; warning ; echo
                                fi
                        done
                fi
                exit 6
        fi
}

function start() {
        [ -x $slapd ] || exit 5
        [ `id -u` -eq 0 ] || exit 4
        configtest
        # Define a couple of local variables which we'll need. Maybe.
        user=root
        prog=`basename ${slapd}`
        harg="$SLAPD_URLS"
        if test x$SLAPD_LDAP = xyes ; then
                harg="$harg ldap:///"
        fi
        if test x$SLAPD_LDAPS = xyes ; then
                harg="$harg ldaps:///"
        fi
        if test x$SLAPD_LDAPI = xyes ; then
                harg="$harg ldapi:///"
        fi
        # System resources limit.
        if [ -n "$SLAPD_ULIMIT_SETTINGS" ]; then
                ulimit="ulimit $SLAPD_ULIMIT_SETTINGS &>/dev/null;"
        else
                ulimit=""
        fi
        # Release reserverd port
        [ -x /sbin/portrelease ] && /sbin/portrelease slapd &>/dev/null || :
        # Start daemons.
        echo -n $"Starting $prog: "
        daemon --pidfile=$pidfile --check=$prog $ulimit ${slapd} -h "\"$harg\"" -u ${user} $OPTIONS $SLAPD_OPTIONS 
        RETVAL=$?
        if [ $RETVAL -eq 0 ]; then
                touch $lockfile
                ln $slapd_pidfile $pidfile
        fi
        echo
        return $RETVAL
}

function stop() {
        # Stop daemons.
        prog=`basename ${slapd}`
        [ `id -u` -eq 0 ] || exit 4
        echo -n $"Stopping $prog: "

        # This will remove pid and args files from /var/run/openldap
        killproc -p $slapd_pidfile -d $SLAPD_SHUTDOWN_TIMEOUT ${slapd}
        RETVAL=$?

        # Now we want to remove lock file and hardlink of pid file
        [ $RETVAL -eq 0 ] && rm -f $pidfile $lockfile
        echo
        return $RETVAL
}

# See how we were called.
case "$1" in
        configtest)
                configtest
                ;;
        start)
                start
                RETVAL=$?
                ;;
        stop)
                stop
                RETVAL=$?
                ;;
        status)
                status -p $pidfile ${slapd}
                RETVAL=$?
                ;;
        restart|force-reload)
                stop
                start
                RETVAL=$?
                ;;
        condrestart|try-restart)
                status -p $pidfile ${slapd} > /dev/null 2>&1 || exit 0
                stop
                start
                ;;
        usage)
                echo $"Usage: $0 {start|stop|restart|force-reload|status|condrestart|try-restart|configtest|usage}"
                RETVAL=0
                ;;
        *)
                echo $"Usage: $0 {start|stop|restart|force-reload|status|condrestart|try-restart|configtest|usage}"
                RETVAL=2
esac

exit $RETVAL
```
使用以下命令启动slapd服务并观察`/var/log/slapd.log `日志

``` sh
$ service slapd start 
```

### 5. 导入kerberos用户作为管理员，删除默认rootdn
建立`setup.ldif`文件，内容如下

```
dn: dc=da,dc=adk
objectclass: organization
objectclass: dcObject
objectclass: top
o: myCompany
dc: da
description: root entry

dn: ou=people,dc=da,dc=adk
objectclass: organizationalUnit
ou: people
description: Users

dn: uid=ldapadmin,ou=people,dc=da,dc=adk
objectclass: inetOrgPerson
objectclass: posixAccount
objectclass: shadowAccount
cn: LDAP admin account
sn: ldapadmin
uid: ldapadmin
uidNumber: 1001
gidNumber: 100
homeDirectory: /home/ldap
loginShell: /sbin/nologin
```

导入数据库，密码是配置文件中的`secret`

``` sh
$ /usr/local/openldap/bin/ldapadd -x -D "cn=admin,dc=da,dc=adk" -W -f setup.ldif
```

删除`slapd.conf`文件中的`rootdn`和`rootpw`两行并重启slpad服务。此时只有`ldapadmin@DA.ADK`具有管理权限。

### 6. 客户端安装及测试
可以使用`yum install openldap-clients`安装openldap客户端，由于[openldap-clients-2.4.23-26开始使用moznss代替TLS][1]，为了继续使用TLS需要哈希化证书文件.

``` sh
$ cd /etc/openldap
$ scp tao02:/usr/local/openldap/etc/openldap/certs/CA.crt cacerts/
$ cacertdir_rehash cacerts
```

修改`/etc/openldap/ldap.conf`以下两个配置

```
BASE    dc=da,dc=adk
URI     ldap://tao02
```

此时执行`ldapsearch`及`ldapsearch -ZZ`应该能正常返回，说明LDAP及TLS正常。其他客户端安装可以分发配置和证书目录。`tao02`本身也应该安装客户端。

在`tao01`上建立文件`test.ldif`

```
dn: uid=test,ou=people,dc=da,dc=adk
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
```

在`tao01`上执行以下命令应该有类似输出

``` sh
$ kdestroy 
$ ldapsearch
SASL/GSSAPI authentication started
ldap_sasl_interactive_bind_s: Local error (-2)
        additional info: SASL(-1): generic failure: GSSAPI Error: Unspecified GSS failure.  Minor code may provide more information (Credentials cache file '/tmp/krb5cc_0' not found)
$ kinit admin/admin
$ ldapsearch
SASL/GSSAPI authentication started
SASL username: admin/admin@DA.ADK
SASL SSF: 56
SASL data security layer installed.
# extended LDIF
#
# LDAPv3
# base <> (default) with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# search result
search: 5
result: 32 No such object

# numResponses: 1
$ ldapwhoami
SASL/GSSAPI authentication started
SASL username: admin/admin@DA.ADK
SASL SSF: 56
SASL data security layer installed.
dn:uid=admin/admin,ou=people,dc=da,dc=adk
$ ldapadd -f test.ldif 
SASL/GSSAPI authentication started
SASL username: admin/admin@DA.ADK
SASL SSF: 56
SASL data security layer installed.
adding new entry "uid=test,ou=people,dc=da,dc=adk"
ldap_add: Insufficient access (50)
        additional info: no write access to parent
$ kinit ldapadmin
$ ldapadd -f test.ldif 
SASL/GSSAPI authentication started
SASL username: ldapadmin@DA.ADK
SASL SSF: 56
SASL data security layer installed.
adding new entry "uid=test,ou=people,dc=da,dc=adk"
$ ldapsearch -b 'dc=da,dc=adk'
SASL/GSSAPI authentication started
SASL username: ldapadmin@DA.ADK
SASL SSF: 56
SASL data security layer installed.
# extended LDIF
#
# LDAPv3
# base <dc=da,dc=adk> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# da.adk
dn: dc=da,dc=adk
objectClass: organization
objectClass: dcObject
objectClass: top
o: myCompany
dc: da
description: root entry

# people, da.adk
dn: ou=people,dc=da,dc=adk
objectClass: organizationalUnit
ou: people
description: Users

# ldapadmin, people, da.adk
dn: uid=ldapadmin,ou=people,dc=da,dc=adk
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: LDAP admin account
sn: ldapadmin
uid: ldapadmin
uidNumber: 1001
gidNumber: 100
homeDirectory: /home/ldap
loginShell: /sbin/nologin

# test, people, da.adk
dn: uid=test,ou=people,dc=da,dc=adk
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: test account
sn: test
uid: test
uidNumber: 1002
gidNumber: 1002
homeDirectory: /home/test
loginShell: /bin/bash

# search result
search: 5
result: 0 Success

# numResponses: 5
# numEntries: 4
```
同时可以查看日志文件`tao01:/var/log/krb5kdc.log`及`tao02:/var/log/slapd.log`进行观察。

#### 备注
可以使用`yum install openldap openldap-servers openldap-clients openldap-devel compat-openldap`安装openldap，据说将`/etc/openldap/slap.d`目录移除后可以建立自己的`slapd.conf`配置文件。通过这种方法可以不用修改`/etc/init.d/slpad`脚本，减少对服务的变动。

### *参考文档*
*[Integrating LDAP and Kerberos](http://www.linux-mag.com/id/4738/,"Part Two不再标注")*
*[配置OpenLDAP使用Kerberos验证](http://blog.sina.com.cn/s/blog_5ce87d560100gjhm.html,"主要参考Kerberos配置文件")*
*[Kerberos and LDAP](https://help.ubuntu.com/10.04/serverguide/kerberos-ldap.html)*
*[OpenLDAP Software 2.4 Administrator's Guide](http://www.openldap.org/doc/admin24/OpenLDAP-Admin-Guide.pdf)*
*[OpenLDAP在LINUX下的安装说明](http://www.cnblogs.com/kungfupanda/archive/2009/12/15/1564555.html)*
*[RHEL6配置简单LDAP服务器基于TLS加密和NFS的用户家目录自动挂载](http://blog.sina.com.cn/s/blog_64aac6750101gwst.html)*
*[Openldap集成tls/ssl](http://mosquito.blog.51cto.com/2973374/1098456)*
*[HOWTO solve OpenLDAP bdb_equality_candidates errors](http://blog.remibergsma.com/2012/03/05/howto-solve-openldap-bdb_equality_candidates-errors/)*

[1]: http://unixadminschool.com/blog/2013/04/rhel-6-3-ldap-series-part-4-troubleshooting/
