---
layout: post
title: 浅谈SSH信任关系
category: Linux 
tagline: 基于公钥认证的SSH信任关系建立及维护
description: 同一集群使用一套公密钥，认证过程、known_hosts维护。 
tags: ["SSH","Linux","信任关系","公密钥"]
---
{% include JB/setup %}

# 简介
使用SSH进行登录认证通常有两种方式：

* 基于密码。
* 基于公钥。

基于密码的认证是最简单最基础的认证，也正因为其简单和基础，这种认证方式在小规模的集群管理上就力有不逮，每次登录需要输入密码不利于集群内服务器之间的自动化管理。基于公钥加密的认证方式虽然在最初的设置上稍微复杂，但是能够在集群内部及集群之间建立信任关系，方便进行管理和维护，很多工具类似于HADOOP系统的管理脚本，都是依托于信任关系才能顺利执行的。而且，由于公钥认证方式不需要在网络上传输密钥（密码），安全性也比密码认证方式要高。在小规模的集群管理上，建立及维护SSH信任关系也就成了一项基本工作。

# 基本认证流程
提到SSH认证方式，就不能不提对称加密和非对称加密。在上文提到的两种认证方式中，密码加密属于对称加密，加密及解密用的是相同的会话密钥；而公钥加密则属于非对称加密，在认证过程中使用公钥加密的密文将使用私钥进行解密。下面简单介绍下两种SSH认证方式的认证流程。
系统在建立之初，就会生成一套 **系统级** 的公密钥，以CentOS为例，默认的公密钥分别是`/etc/ssh/ssh_host_rsa_key.pub`及`/etc/ssh/ssh_host_rsa_key`，这套公密钥在认证过程中由服务端使用。

* 基于密码的认证流程
    1. 版本号协商。客户端连接服务端22端口，双方协商协议版本号。
    2. 密钥及算法协商。双方协商确定加密算法。服务端明文发送自身公钥`/etc/ssh/ssh_host_rsa_key.pub`，客户端根据该公钥验证服务端身份，并使用该公钥加密会话密钥发送给服务端，服务端使用私钥`/etc/ssh/ssh_host_rsa_key`解密密文得到会话密钥，后续会话都使用会话密钥进行加密。
    3. 认证阶段。采用密码认证时，客户端将用户名和密码使用会话密钥加密后发送给服务端，服务端使用会话密钥解密后得到明文用户名和密码，并通过与本地或远处认证该用户名和密码的合法性，返回认证成功或失败的消息。
    4. 会话请求和交互。认证通过后客户端向服务端发送会话请求，服务端接收后进入交互会话阶段，这个阶段双方的数据交互都通过会话密钥进行加解密。

* 基于公钥的认证流程（只有第3步认证阶段与密码认证有差异）
    1. 版本号协商。客户端连接服务端22端口，双方协商协议版本号。
    2. 密钥及算法协商。双方协商确定加密算法。服务端明文发送自身公钥`/etc/ssh/ssh_host_rsa_key.pub`，客户端根据该公钥验证服务端身份，并使用该公钥加密会话密钥发送给服务端，服务端使用私钥`/etc/ssh/ssh_host_rsa_key`解密密文得到会话密钥。后续会话都使用会话密钥进行加密。
    3. 认证阶段。采用公钥认证时，客户端将用户名和公钥使用会话密钥加密后发送给服务端，服务端使用会话密钥解密后得到明文用户名和公钥，并在用户家目录下的.ssh目录中寻找该公钥，如果此公钥存在，则服务端生成质询（随机字符串），并使用收到的公钥加密质询并在使用会话密钥再次加密后发送给客户端，客户端使用会话密钥和自身私钥两次解密该密文得到质询，并使用会话密钥加密该质询发送给服务端，服务端使用会话密钥解密后得到质询并进行判断，如果是自己生成的质询则返回认证成功。
    4. 会话请求和交互。认证通过后客户端向服务端发送会话请求，服务端接收后进入交互会话阶段，这个阶段双方的数据交互都通过会话密钥进行加解密。

# 建立信任关系

* 生成 **用户级** 公密钥

````
$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/home/jasee/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/jasee/.ssh/id_rsa.
Your public key has been saved in /home/jasee/.ssh/id_rsa.pub.
```

* 配置信任公钥

```
$ cd ~/.ssh/
$ cp id_rsa.pub authorized_keys
```

* 将本机加入known_hosts

```
$ ssh jasee@tao
The authenticity of host 'tao (172.16.19.69)' can't be established.
RSA key fingerprint is fa:cf:be:5b:ac:d3:03:b2:29:97:34:e1:8d:f6:19:4f.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'tao,172.16.19.69' (RSA) to the list of known hosts.
Last login: Wed Sep  4 09:34:22 2013
[jasee@tao ~]$ exit
logout
Connection to tao closed. 
```

* 建立于另外一台服务器的信任关系

```
$ scp -r ~/.ssh jasee@da-op-test1:/home/jasee/
The authenticity of host 'da-op-test1 (172.16.18.201)' can't be established.
RSA key fingerprint is 10:cc:79:26:1d:5c:77:a9:1b:46:0f:f1:c1:cd:59:84.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'da-op-test1,172.16.18.201' (RSA) to the list of known hosts.
jasee@da-op-test1's password: 
known_hosts            100%    805    0.8KB/s   00:00
authorized_keys        100%    391    0.4KB/s   00:00
id_rsa                 100%   1675    1.6KB/s   00:00
id_rsa.pub             100%    391    0.4KB/s   00:00
```

这样我们就生成了公密钥并完成对自身的一次公钥认证登陆，并建立起了两台服务器之间jasee账号的信任关系。在同一集群内，权限一致，不需要每个机器单独生成公密钥，使用同一套方便集群内及集群间进行管理和维护。此时两台服务器具有相同的`~/.ssh/`目录，内含四个文件如下：

1. authorized_keys
服务端，包含被认证的公钥，拥有对应私钥的客户端都可以依托信任关系使用公钥认证方式，无需密码即可登录本服务器。
2. id_rsa
客户端，用户私钥。
3. id_rsa.pub
客户端，用户公钥。
4. known_hosts
客户端，存储了已知客户端的公钥。为了防止中间人攻击，客户端在初次连接某服务端时，需要确定服务端返回的公钥是否是正确的（知名主机可将自己公钥放在官网上），被确认的主机公钥将会保存在known_hosts里，再次连接时不需要确认。

# 维护信任关系
当集群内各个服务器使用同一套公密钥的时候，维护信任关系退化成两点：

1. 分发`~/.ssh`目录。
2. 维护known_hosts内容。

把`~/.ssh`当成普通目录分发到对应位置即可，可使用批量化工具，注意权限。

* 关于known_hosts

```
$ cat known_hosts 
tao,172.16.19.69 ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAxA+Hl1jQNHAWdnnBP9mawYxDgYwupHU5yhy5dfD5y8cmoFtmFhx9W8VDSlVMMqgXpTX/H8rsjDLmUHVgpceWT2Orwx9P9ih8iXaWJ/NbvDNzsX7KhLhWY2/VQTP4hjDNfOzwki+FeCW5rbRposWClHnt91/0sv3pOtkgm7JrbEn4N0V62KVYT+R0+TqOzLqZe88YTgVlxrFlvUdZt5EjhjkMDYgJ7rFe++IPKA/FE58zMpI1wrOZsKjyDYHcagfANEO3yhWV+9tXaUGl8i6db4STaCbblCSvj3mbyrtv3YAw8usGiiJyJ49RUa32DnJwI4JUw57+4+ltfF4Mq6WEIQ==
da-op-test1,172.16.18.201 ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEA6qgR3iT6jUaJrgOi5Z+QoGBr077ApYyhJh7NNmaM781KCbAwAUP0z4cJuuTqZQcbgZmh2o5R0pxYWPPfDBhDMMcBsKK3MP/uy6/t3/rIAq1VaFFva+sp1aG/m1C8iphZ2PKk8u6itIRFZle3FrADnP0zoLrjTgP9GfgGSN3DwCi1IPAAa3S7RWgKAXxhvWyhS1rYZF60G5M/UJGRNRg0C9fZUb8j3i+EHG8iPfvQcJc2sX7MWkYStWmuaAbhMY4/u3tApjb3jzCy0Q/Gj6im/dFhE1GraDoJg1QkvlsnbnuXUJ6hd3Zt35A20ibQIixi23uh6QQ4epmuK9MBcCTq+Q==
```

可以看出，known_hosts的每一行记录了一个可信服务端，第一个域记录了服务端主机名，第二个域记录了服务端IP、公密钥加密算法及服务端 **系统级** 公钥。两个主机的系统级公钥如下：

```
$ cat /etc/ssh/ssh_host_rsa_key.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAxA+Hl1jQNHAWdnnBP9mawYxDgYwupHU5yhy5dfD5y8cmoFtmFhx9W8VDSlVMMqgXpTX/H8rsjDLmUHVgpceWT2Orwx9P9ih8iXaWJ/NbvDNzsX7KhLhWY2/VQTP4hjDNfOzwki+FeCW5rbRposWClHnt91/0sv3pOtkgm7JrbEn4N0V62KVYT+R0+TqOzLqZe88YTgVlxrFlvUdZt5EjhjkMDYgJ7rFe++IPKA/FE58zMpI1wrOZsKjyDYHcagfANEO3yhWV+9tXaUGl8i6db4STaCbblCSvj3mbyrtv3YAw8usGiiJyJ49RUa32DnJwI4JUw57+4+ltfF4Mq6WEIQ== 
ssh da-op-test1 'cat /etc/ssh/ssh_host_rsa_key.pub '
ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEA6qgR3iT6jUaJrgOi5Z+QoGBr077ApYyhJh7NNmaM781KCbAwAUP0z4cJuuTqZQcbgZmh2o5R0pxYWPPfDBhDMMcBsKK3MP/uy6/t3/rIAq1VaFFva+sp1aG/m1C8iphZ2PKk8u6itIRFZle3FrADnP0zoLrjTgP9GfgGSN3DwCi1IPAAa3S7RWgKAXxhvWyhS1rYZF60G5M/UJGRNRg0C9fZUb8j3i+EHG8iPfvQcJc2sX7MWkYStWmuaAbhMY4/u3tApjb3jzCy0Q/Gj6im/dFhE1GraDoJg1QkvlsnbnuXUJ6hd3Zt35A20ibQIixi23uh6QQ4epmuK9MBcCTq+Q==
```

为了在整个集群中（或者可信的集群间）畅通无阻，我们需要维护一份完整且正确的known_hosts（这也是上文中建立信任关系时第一次就连接本机的原因）。

* 集群新增服务器
集群新增服务器时，在分发`~/.ssh`之后，需要对所有需要连接到这台服务器的客户端上的known_hosts做一次更新（一般情况下集群内的每一台服务器都即时客户端又是服务端，集群外的每一台服务器都只是客户端）。
更新方式有以下几种
    1. 在单台客户端上使用ssh连接新增服务器，输入`yes`确认新服务器身份后得到最新的known_hosts文件，将此文件分发到同类服务器上。
    2. 直接在某客户端上编辑known_hosts文件并分发。
    3. 批量在所有客户端上执行第1种或第2种方法，不分发。

* 集群变更服务器
如果集群服务器出现替换、重装系统等，known_hosts和新服务器的公钥值会不一致，出现防止欺诈的提醒，有以下几个方法来解决：
    1. 删除掉known_hosts里的旧记录，安装上文方法重新添加新记录并分发。
    2. 可以保留原系统（替换或重装）前的系统级公密钥，放在新系统上。

```
$ ssh da-op-test1
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that the RSA host key has just been changed.
The fingerprint for the RSA key sent by the remote host is
10:cc:79:26:1d:5c:77:a9:1b:46:0f:f1:c1:cd:59:84.
Please contact your system administrator.
Add correct host key in /home/jasee/.ssh/known_hosts to get rid of this message.
Offending key in /home/jasee/.ssh/known_hosts:2
RSA host key for da-op-test1 has changed and you have requested strict checking.
Host key verification failed.
```

# 总结
*一切皆文件*，信任关系的建立和维护都是围绕着认证过程对几个文件进行管理。

