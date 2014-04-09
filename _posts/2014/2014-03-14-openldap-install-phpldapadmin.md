---
layout: post
title: OpenLDAP--安装phpLDAPadmin
category: 运维
description: 由于需要对Kerberized OpenLDAP进行管理，安装支持sasl认证的phpLDAPadmin就顺利成章，本文记录Apache支持Kerberos认证及phpLDAPadmin使用sasl方式登陆的过程。
tags: ["Apache","OpenLDAP","Kerberos","Mod_auth_kerb","phpLDAPadmin"]
---

[phpLDAPadmin][1]是一个LDAP web客户端，它提供一个简单的、支持多语言多环境的LDAP管理功能。

### 安装httpd及phpLDAPadmin

```sh
$ yum install httpd php php-bcmath php-gd php-mbstring php-xml php-ldap mod_auth_kerb
$ wget http://sourceforge.net/projects/phpldapadmin/files/phpldapadmin-php5/1.2.3/phpldapadmin-1.2.3.zip/download
$ unzip phpldapadmin-1.2.3.zip
$ cp -r phpldapadmin-1.2.3 /var/www/html/ldap
$ chown apache:apache -R /var/www/html/ldap
```

### 配置httpd

生成httpd服务`service key`

```sh
$ kinit admin/admin
$ kadmin
kadmin:  addprinc -randkey HTTP/tao02.opjasee.com@OPJASEE.COM
kadmin:  ktadd -k /etc/httpd/http.keytab HTTP/tao02.opjasee.com@OPJASEE.COM
$ chown apache:apache /etc/httpd/http.keytab
```

新建配置或在`/etc/httpd/conf.d/auth_kerb.conf`追加如下内容

```
<Location /ldap/htdocs>
    AuthType Kerberos
    AuthName "LDAP Admin"
    KrbAuthRealms OPJASEE.COM
    KrbServiceName HTTP/tao02.opjasee.com@OPJASEE.COM
    Krb5KeyTab /etc/httpd/http.keytab
    KrbMethodNegotiate Off
    KrbMethodK5Passwd On
    KrbSaveCredentials On
    require valid-user
</Location>
```

开启`KrbMethodNegotiate`需要浏览器端进行支持，如果浏览器端不支持的话，在进行密码认证前会有一次无用的登陆框，因此设置为关闭。另外`KrbSaveCredentials`一定要打开，这样httpd才能把Ticket传递给phpLDAPadmin进行认证。如果要只允许个别用户能够登陆的话，`require`参数设置为对应principal，多个principal使用逗号分隔。

配置完成后启动httpd服务。

### 配置phpLDAPadmin

参考`/var/www/html/ldap/config/config.php.example`建立配置文件`/var/www/html/ldap/config/config.php`，最好将其属主数组改成apache。

```php
<?php
$servers = new Datastore();
$servers->newServer('ldap_pla');
$servers->setValue('server','name','LDAP Server');
$servers->setValue('server','host','tao02.opjasee.com');
$servers->setValue('server','port',389);
$servers->setValue('server','base',array('dc=opjasee,dc=com'));
$servers->setValue('login','bind_id','');
$servers->setValue('login','bind_pass','');
$servers->setValue('server','tls',true);

# SASL auth
$servers->setValue('login','auth_type','sasl');
$servers->setValue('sasl','mech','GSSAPI');
$servers->setValue('sasl','realm','OPJASEE.COM');
$servers->setValue('sasl','authz_id',null);
$servers->setValue('sasl','authz_id_regex','/^uid=([^,]+)(.+)/i');
$servers->setValue('sasl','authz_id_replacement','$1');
$servers->setValue('sasl','props',null);

$servers->setValue('appearance','password_hash','md5');
$servers->setValue('login','attr','dn');
$servers->setValue('login','fallback_dn',false);
$servers->setValue('login','class',array());
$servers->setValue('server','read_only',false);
$servers->setValue('appearance','show_create',true);

$servers->setValue('auto_number','enable',true);
$servers->setValue('auto_number','mechanism','search');
$servers->setValue('auto_number','search_base',null);
$servers->setValue('auto_number','min',array('uidNumber'=>1000,'gidNumber'=>500));
$servers->setValue('auto_number','dn',null);
$servers->setValue('auto_number','pass',null);

$servers->setValue('login','anon_bind',true);
$servers->setValue('custom','pages_prefix','custom_');
$servers->setValue('unique','attrs',array('mail','uid','uidNumber'));
$servers->setValue('unique','dn',null);
$servers->setValue('unique','pass',null);

$servers->setValue('server','visible',true);
$servers->setValue('login','timeout',30);
$servers->setValue('server','branch_rename',false);
$servers->setValue('server','custom_sys_attrs',array('passwordExpirationTime','passwordAllowChangeTime'));
$servers->setValue('server','custom_attrs',array('nsRoleDN','nsRole','nsAccountLock'));
$servers->setValue('server','force_may',array('uidNumber','gidNumber','sambaSID'));
?>
```

此时访问`http://tao02.opjasee.com/ldap`，Apache会要求输入认证信息，认证成功后直接进入phpLDAPadmin管理界面并处于已登录状态。
目前还存在一个问题，就是无法退出登录切换用户，只能关闭浏览器让httpd再次认证。

#### *参考文档*

*[Kerberos Module for Apache](http://modauthkerb.sourceforge.net/configure.html)*

[1]: http://phpldapadmin.sourceforge.net/wiki/index.php/Main_Page
