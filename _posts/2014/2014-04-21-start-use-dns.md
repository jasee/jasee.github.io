---
layout: post
title: 使用Bind搭建内网DNS
category: 运维
description: 随着内网服务器数量增大，维护一份hosts配置越来越困难，需要搭建一个纯内网DNS。
tags: ["Bind","DNS","FQDN","Cache"]
---

随着内网服务器数量增大，维护一份hosts配置越来越困难，需要搭建一个纯内网DNS，具体的需求是：

1. 目前所有服务器使用短名，各个程序配置及记录也是，需要能够继续使用短名。
2. 除了自行定义的解析以外，其他域名解析仍使用公网DNS，也不需要将一些同域名公网配置手工同步到DNS上。
3. 内外网域名的查询性能不能与之前存在明显差异。

测试环境及部署目标：

1. tao01，Centos6.3，部署Bind主服务。
2. tao02，Centos6.3，部署Bind从服务。

### 安装及配置
使用yum进行安装：`yum install bind`
#### 1. 主配置
Bind的主配置文件为`/etc/named.conf`，修改该配置内以下几部分(缩进区域都在options定义内)

```
    listen-on port 53 { any; }; // 监听所有地址
    allow-query     { any; };   // 允许所有来源的查询
    dnssec-enable no;           // 这三条关闭dnssec，内网DNS对安全性要求不高，
    dnssec-validation no;       // 关闭后能显著提高外部域名查询效率
    dnssec-lookaside no;        // 从秒级降低到百毫秒级
    forwarders {202.106.0.20;}; // 使用转发器
include "/etc/named/opjasee.com.zones"; //自定义区域配置写在另外的文件中
```

在上述配置中，将联通DNS作为转发器使用。默认情况下，当DNS收到一个既不是本域也不在本地缓存中的域名查询时，通过根域进行下一步迭代查询，在配置了转发器的情况下，将优先向转发器进行迭代查询，如果转发器在一定时间内没有返回，则继续从根域开始迭代查询。使用ISP的DNS做转发器能够显著提高常用域名的查询效率，在北京BGP机房测试`www.baidu.com`的查询时间，约从500ms降低到15ms。

#### 2. 区域配置
`/etc/named/opjasee.com.zones`配置内容如下：

```
zone "opjasee.com" IN {
        type master;
        file "opjasee.com.zone";
        notify yes;
};

zone "245.168.192.in-addr.arpa" IN {
        type master;
        file "245.168.192.zone";
        notify yes;
};
```

除了后续提到的定期查询更新以外，`notify`参数允许主节点在区域数据 **可能**
发生变更时在15分钟以内(Bind会错开每个区域的更新间隔时间进行通知，在我的小测试环境下基本是即时)通知从节点检查更新，其中从节点的列表是主节点通过查询区域NS记录并排除SOA记录的MNAME字段和自身主机域名之后获得的。另外也可以通过`also-notify`参数通知其他节点，接收节点除了接收主节点的通知以外还可以通过`allow-notify`参数允许接收来自其他节点的通知，多级拓扑适当配置允许通知不断传递，具体见参考资料2。

#### 3. 区域数据
这两个文件中存放的是DNS正反解的具体数据，SOA的参数含义在后面从机配置部分说明。其中`tao01 IN A 192.168.245.131`中的`tao01`是`tao01.opjasee.com.`的别名(简写)。在区域配置中，`zone`语句的第二个字段指定了域名`opjasee.com`，该域名会被附加到区域数据所有不以`.`结尾的名字之后。如果不使用简写则要注意点号，`tao01.opjasee.com IN A 192.168.245.131`这种写法是错误的。反解配置类似。
`/var/named/opjasee.com.zone`

```
$TTL 10m
@ IN SOA tao01.opjasee.com. root (
    2014042101       ; serial
    30m      ; refresh
    10m      ; retry
    1W      ; expire
    3H )    ; minimum

@ IN NS tao01.opjasee.com.
@ IN NS tao02.opjasee.com.
tao01 IN A 192.168.245.131
tao02 IN A 192.168.245.129
tao03 IN A 192.168.245.130
tao04 IN A 192.168.245.132
```

`/var/named/245.168.192.zone`

```
$TTL 10m
@ IN SOA tao01.opjasee.com. root (
    2014042101       ; serial
    30m      ; refresh
    10m      ; retry
    1W      ; expire
    3H )    ; minimum

@ IN NS tao01.opjasee.com.
@ IN NS tao02.opjasee.com.
131 IN PTR tao01.opjasee.com.
129 IN PTR tao02.opjasee.com.
130 IN PTR tao03.opjasee.com.
132 IN PTR tao04.opjasee.com.
```

#### 4. 启动和校验
完成上述三部分配置之后就可以使用`service named start`来启动Bind了，观察`/var/log/message`日志查看是否正常。
可以配置`/etc/resolv.conf`之后使用nslookup进行校验，不过这个配置还有其他作用，放到后面一并说明，先使用dig测试一下，下面这两条命令应该都有正常的返回。

```
dig tao02.opjasee.com @192.168.245.131
dig tao02.opjasee.com @tao01.opjasee.com
```

### 从机DNS配置
DNS这种关键服务，肯定要有备机的啦。备机的`/etc/named.conf`与主机相同，修改`/etc/named/opjasee.com.zones`即可:

```
zone "opjasee.com" IN {
        type slave;
        file "slaves/opjasee.com.zone";
        masters {192.168.245.131;};
};

zone "245.168.192.in-addr.arpa" IN {
        type slave;
        file "slaves/245.168.192.zone";
        masters {192.168.245.131;};
};
```

启动Bind服务即可，此时可以在`/var/named/slaves`目录下看到两个数据文件，内容与主上的类似(格式可能不同)，使用上述类似的dig命令应该能够得到类似的结果。至此已经完成了一个简单的主从DNS的搭建。
如果对主从同步的延迟没有要求，可以不配置`notify`。另外，从机的`file`配置不是必须的，当存在该配置时，从机启动时先从本地数据文件加载数据，然后检查主机是否有更新，当没有该配置时，直接从主机获取，为了提高安全性，配置上比较好。
下面解释之前SOA部分的配置含义:

```
2014042101  ; serial 序号，从机每次检查更新时，先检查该区域序号，如果主机序号大于从机，则说明需要同步区域数据。建议使用YYYMMDDNN格式，每次更新数据时更新
30m         ; refresh 从机每隔多久检查一次主机数据
10m         ; retry 从机如果无法连接主机，重试的间隔时间
1W          ; expire 从机持续无法连接主机时，区域数据的有效时间，超过该时间无法连接主机时，从机也不再提供解析服务。
3H          ; minimum  否定缓存TTL，该区域的否定响应的缓存时间
```

### 使用
#### 1. FQDN与shortname
服务器名称可以是全名(FQDN)，也可以是短名。在已经使用短名的情况下，如果想要能够得到全名(`hostname -f`能正确返回)，主要有[两种方法][1]，要使用DNS，当然是配置`/etc/resolv.conf`

```sh
search opjasee.com
nameserver 192.168.245.131
nameserver 192.168.245.129
```

关于[search和domain的说明][2]有不少，大多人云亦云，我是没看出来特别的区别来，就用`search`吧。如果网络使用了DHCP的话，要处理好`/sbin/dhclient-script`脚本会根据网卡和服务器名重新生成`/etc/resolv.conf`的问题。
部分服务(如`authconfig`的`--ldapserver`参数)因为证书等关系是需要使用全名的，否则无法匹配，如果出现此类问题可以从这方面排查一下。

#### 2. 关于搜索域
配置搜索域以后，当我查找一个带点的名称时，如何判断是在请求一个全名还是搜索域中的短名的呢？使用nslookup的debug模式进行测试，得到以下结论：

1. 如果名称中不包含`.`，那么显然是一个短名，直接在搜索域中进行搜索。
2. 如果名称中包含`.`，则首先认为这是一个全名，在互联网上进行搜索，如果域名存在，则返回结果，如果不存在，则在搜索域中再次搜索。

按照这个测试结论，在 **只使用短名** 进行DNS请求的情况下，服务器使用短名`google.c`或者`c.google.com`(测试的时候该域名不存在)是可以的，但是使用`google.com`则不行。如果想使用带点短名，建议使用不存在的顶级域名，如`tao01.bj`。也可以看出，当短名中包含`.`时，实际上是有一个试错的过程，如果对性能有要求，最好不使用这样的名称，必须使用的话在访问时使用全名。

#### 3. 使用缓存
在一些分布式服务中，各个节点之间每秒可能发生数万次通信，之前使用hosts文件解析，性能上不存在问题，使用DNS后，需要使用nscd服务进行缓存来解决性能问题和降低DNS服务器压力。
使用`yum install nscd`安装后，编辑`/etc/nscd.conf`，保证hosts缓存是打开的即可，正向缓存时间修改成600秒，其他参数不变。其他缓存项目，如passwd或group，不清楚或不使用的话建议关闭。启动nscd后，整个域名服务实际上存在多级缓存了:

1. nscd未缓存的，到内网DNS中查找。
2. 内网DNS中未缓存的，到转发器DNS查找。
3. 转发器DNS也未缓存的，从根域开始查找。(不考虑转发器超时无返回，另假定上面设定的联通DNS没有进一步进行转发器或缓存配置)。

这三级缓存每次向上一级查找后都会将结果进行缓存，这样来看，性能应该不是问题了。在百兆内网进行本地域名解析测试，开启nscd缓存后的百万次查询时间从380秒降低到13.5秒，与直接使用hosts文件解析时间一致。附测试脚本:

```python
#!/usr/bin/python
import socket
import time

t=time.time()
for x in range(1,1000000):
    socket.getaddrinfo('tao02',None)

print time.time() - t
```

另外附清空缓存的两个命令，更多用法参考文档。

```sh
$ rndc flush    # 清空DNS服务器的缓存
$ nscd -i hosts # 清空nscd的hosts缓存
```

------

刚刚建立的DNS域是`opjasee.com`，不满足第2需求的后半部分，实际使用的时候改成`idc.opjasee.com`之类的吧，否则正常的域名如`www.opjasee.com`就没法解析了。
另外，如果不考虑后期维护风险的话，可以尝试使用[dnspod-sr][3],[有篇文章][4]介绍了这个轻量级内网DNS，可以尝试一下。

#### *参考资料*
*[Setting the hostname: FQDN or short name?](http://serverfault.com/questions/331936/setting-the-hostname-fqdn-or-short-name)*
*[DNS BIND Zone Transfers and Updates](http://www.zytrax.com/books/dns/ch7/xfer.html)*

[1]: http://my.oschina.net/jing31/blog/6613
[2]: http://blog.csdn.net/demo_deng/article/details/9629177
[3]: https://github.com/DNSPod/dnspod-sr
[4]: http://www.ttlsa.com/linux/dnspod-sr-little-dns/
