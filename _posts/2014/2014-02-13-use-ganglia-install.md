---
layout: post
title: Ganglia--安装
category: 运维
description: Ganglia是一个方便易用的监控系统，本文记录了基本安装过程。
tags: ["Ganglia","Monitor"]
---

Ganglia是一个挺好的监控系统，便于编写的插件脚本、灵活的拓扑机制以及web界面上销魂的正则匹配和图形聚合，谁用谁知道呀。本文记录的Ganglia的基本安装过程。
测试机环境:

* gnode01:Centos6.3，安装gmetad、gmond、gweb。
* gnode02:Centos5.4，安装gmond。
* gnode03:Centos6.3，安装gmond。

计划拓扑:

1. 只使用一个gmetad。部署位置为gnode01。三个节点同属于一个群组。
2. 三个节点的gmond都通过UDP单播发送数据到网络，gnode02、03都接收网络数据进行冗余。
3. 三个节点的gmond都通过127.0.0.1向自身发送数据，可以通过TCP探针获取节点的xml数据文件。

随着环境或时间变化，具体命令可能不同，酌情参考使用。

------

### Ganglia的安装
1. 添加EPEL资源库
    由于默认资源库中不包含Ganglia安装包，需要添加[EPEL(Extra Packages for Enterprise Linux)][1]资源库。

    ```sh
    $ rpm -Uvh http://mirror.chpc.utah.edu/pub/epel/6/i386/epel-release-6-8.noarch.rpm #Centos6.3
    $ rpm -Uvh http://mirror.chpc.utah.edu/pub/epel/5/i386/epel-release-5-4.noarch.rpm #Centos5.4
    ```

2. 安装gmond
    gmond需要安装在每个节点上进行数据采集。

    ```sh
    $ yum install ganglia-gmond
    ```

3. 安装gmetad
    gmetad负责从其他gmetad或gmond源收集数据。集群规模不大的话，使用一个gmetad节点就够了。gmetad使用RRDtool存储和显示数据。

    ```sh
    $ yum install ganglia-gmetad
    ```

4. 安装gweb
    gweb用以将gmetad采集的数据可视化。使用Nginx之类的也可以，支持PHP及PHP的JSON扩展即可。最新gweb参见[这里][2]。

    ```sh
    $ yum install httpd php
    $ wget wget http://downloads.sourceforge.net/project/ganglia/ganglia-web/3.5.12/ganglia-web-3.5.12.tar.gz?r=http%3A%2F%2Fsourceforge.net%2Fprojects%2Fganglia%2Ffiles%2Fganglia-web%2F3.5.12%2F&ts=1392258829&use_mirror=citylan
    $ tar -zxf ganglia-web-3.5.12.tar.gz && cd ganglia-web-3.5.12
    $ diff Makefile originMakefile #修改安装配置
    5c5
    < GDESTDIR = /var/www/html/ganglia
    ---
    > GDESTDIR = /usr/share/ganglia-webfrontend
    16c16
    < APACHE_USER = apache
    ---
    > APACHE_USER = www-data
    $ make install
    ```

### Ganglia配置和启动 
1. 配置gmond。
    gmond默认配置文件是`/etc/ganglia/gmond.conf`(Centos5.4是`/etc/gmond.conf`)，如果没有，可以使用`gmond -t`命令生成。gmond主要分为两个部分，一部分负责数据采集和发送，另一部分负责数据的接收。这两部分直接并无直接的数据交互，发送端直接将数据发送到网络，接收端只从网络侦听数据。为了达到前文中的预期效果，三个节点的gmond配置如下。群组中的每个节点向三处发送信息:ghost2/ghost3/自身,每个节点都配置接收信息。

    ```sh
    $ diff originGmond.conf gmond.conf
    12c12
    <   host_dmax = 0 /*secs */
    ---
    >   host_dmax = 300 /*secs */
    15c15
    <   send_metadata_interval = 0 /*secs */
    ---
    >   send_metadata_interval = 300 /*secs */
    23c23
    <   name = "unspecified"
    ---
    >   name = "Cluster01"
    37c37
    <   #bind_hostname = yes # Highly recommended, soon to be default.
    ---
    >   bind_hostname = yes # Highly recommended, soon to be default.
    43c43
    <   mcast_join = 239.2.11.71
    ---
    >   host = gnode02
    45c45,59
    <   ttl = 1
    ---
    >   ttl = 5
    > }
    > 
    > udp_send_channel {
    >   bind_hostname = yes
    >   host = gnode03
    >   port = 8649
    >   ttl = 5
    > }
    > 
    > udp_send_channel {
    >   bind_hostname = yes
    >   host = 127.0.0.1
    >   port = 8649
    >   ttl = 1
    50d63
    <   mcast_join = 239.2.11.71
    52d64
    <   bind = 239.2.11.71
    ```
    
    各个节点完成配置后使用`service gmond start`启动gmond(可配置开机启动)。Centos5.4存在一定问题，接收到的数据除了包含gnode02之外还具有localhost主机(127.0.0.1)，Centos6.3无此问题。两种系统环境的gmond数据兼容存在问题。
    此例中只配置了一个群组，实际使用中经常将不同业务的服务器分配在不同群组中，每个群组中启用两三个聚合节点即可。由于使用了单播，不同群组可以复用同一个端口(8649)，如果使用多播则可以使用同一个多播地址的不同端口。
2. 配置gmetad
    gmetad可以从聚合节点上轮询数据，这次配置了两个聚合节点进行冗余。默认使用`/etc/ganglia/gmetad.conf`配置文件，此例中配置变更如下

    ```sh
    $ diff origingmetad.conf gmetad.conf
    39c39
    < data_source "my cluster" localhost
    ---
    > data_source "Cluster01" gnode02:8649 gnode03:8649
    77c77
    < # trusted_hosts 127.0.0.1 169.229.50.165 my.gmetad.org
    ---
    > trusted_hosts 127.0.0.1 gnode01 gnode02 gnode03
    ```

    可以使用`service gmetad start`命令启动gmetad(可配置开机启动)。此时使用`telnet gnode01 8651`可以获取xml格式的数据。
3. 配置gweb
    gweb本身不需要配置，需要配置的是Apache。暂时直接使用`service httpd start`启动服务即可，可以访问`http://gnode01/ganglia`查看ganglia数据。

### 使用rrdcached降低磁盘压力
在Ganglia采集近2万个数据时，gmetad所在的老服务器磁盘IO基本一直跑满，此时使用rrdcached进行缓存能显著降低磁盘压力。

rrdcached的示例配置如下:

```
RUN_RRDCACHED=1
RRDCACHED_USER="root"
OPTS="-s apache -b /work/data/ganglia/rrds/ -w 300 -z 300"
PIDFILE="/var/run/rrdcached/rrdcached.pid"
SOCKFILE="/var/run/rrdcached/rrdcached.sock"
SOCKPERMS=0660
```

同时需要在gmetad的启动脚本(`/etc/init.d/gmetad`)中加入:

```sh
export RRDCACHED_ADDRESS=unix:/var/run/rrdcached/rrdcached.sock
```

在gweb的配置(`conf.php`)中加入:

```php
$conf['rrdcached_socket'] = "unix:/var/run/rrdcached/rrdcached.sock";
```

重启gmetad。这时，不会每更新一个数据就打开、写入、关闭rrd文件一次，而是先写入缓存，每300s才写一次文件(由于配置了`-z 300`，这些rrd文件的写入将会随机分散，避免同时写入数万个文件)。访问gweb的时候，也会从缓存中抽取还未写入rrd文件的数据进行展示。

[1]: http://fedoraproject.org/wiki/EPEL
[2]: http://sourceforge.net/projects/ganglia/files/ganglia-web/
