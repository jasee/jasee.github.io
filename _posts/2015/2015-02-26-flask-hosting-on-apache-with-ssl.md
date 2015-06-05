---
layout: post
title: Flask程序托管至Apache并启用SSL
category: 运维
description: 简单记录如何使用Apache的WSGI模块服务Flask程序，并在此基础上启用SSL。
tags: ["Flask", "Apache"]
---

几个月前用Flask写了一个简单的资产管理页面，节前又完成了SaltStack的部署，最近计划通过Salt完成服务器硬件信息的自动采集更新，因此需要为资产页面增加API。为了提高安全性，准备通过SSL访问Flask页面，经过一番搜索，发现不太方便继续使用Flask自带的`runserver`，是时候托管到Apache下面了。由于我对Apache不太熟悉，这次记录一下相关的步骤和配置以备参考。

### 安装mod_wsgi

```sh
$ sudo yum install mod_wsgi
```

并在Apache配置中增加一行：

```text
LoadModule wsgi_module modules/mod_wsgi.so
```

### 建立`.wsgi`文件

mod_wsgi模块通过执行指定的`.wsgi`文件获取app对象，在我的环境里，该文件内容如下:

```py
# 使用虚拟Python环境
activate_this = '/home/op/.virtualenvs/adop/bin/activate_this.py'
execfile(activate_this, dict(__file__=activate_this))
import sys
sys.path.insert(0, '/var/www/adop.adsage.com')
# 虽然我用了flask Manager模块管理app，但是此处还是要直接导入app而不是manager对象
from manage import app as application
```

### 配置Apache虚拟主机

```text
<VirtualHost *:80>
    ServerAdmin tao@opjasee.com
    DocumentRoot "/var/www/adop.opjasee.com"
    ServerName adop.opjasee.com
    WSGIDaemonProcess adop user=apache group=apache processes=2 threads=5
    WSGIScriptAlias / /var/www/adop.opjasee.com/adop.wsgi 
    ErrorLog "logs/adop.opjasee.com-error_log"
    CustomLog "logs/adop.opjasee.com-access_log" common
    <Directory /var/www/adop.opjasee.com>
         Options FollowSymLinks
         AllowOverride None
         WSGIProcessGroup adop
         WSGIApplicationGroup %{GLOBAL}
         Order allow,deny
         allow from all
    </Directory>
</VirtualHost>
```

### 配置Apache SSL

```text
......
......
ServerAdmin tao@opjasee.com
DocumentRoot "/var/www/adop.opjasee.com"
ServerName adop.opjasee.com
WSGIScriptAlias / /var/www/adop.opjasee.com/adop.wsgi               
ErrorLog "logs/adop.opjasee.com-error_log"
CustomLog "logs/adop.opjasee.com-access_log" common
<Directory /var/www/adop.opjasee.com>
     Options FollowSymLinks
     AllowOverride None
     WSGIProcessGroup adop
     WSGIApplicationGroup %{GLOBAL}
     Order allow,deny
     allow from all
</Directory>
......
......
SSLCertificateFile /etc/httpd/adop.crt
SSLCertificateKeyFile /etc/httpd/adop.key
......
......
```

在配置443端口的wsgi时，可以复用80端口所用的后端app实例，因此配置中去掉了`WSGIDaemonProcess`那一行。

### 设置页面跳转

当访问一个非https地址时，自动跳转到对应的https地址上，该功能可通过Flask-SSLify模块完成。在`app/__init__.py`内增加如下内容并在配置文件中做相应修改：

```py
if not app.debug and not app.testing and not app.config['SSL_DISABLE']:
    from flask.ext.sslify import SSLify
    sslify = SSLify(app)
```


### *参考文档*
*[mod_wsgi (Apache)](http://flask.pocoo.org/docs/0.10/deploying/mod_wsgi/)*
*[Why are you using embedded mode of mod_wsgi?](http://blog.dscpl.com.au/2012/10/why-are-you-using-embedded-mode-of.html)*
*[SSL on Apache2 with WSGI](http://stackoverflow.com/questions/4893432/ssl-on-apache2-with-wsgi)*
