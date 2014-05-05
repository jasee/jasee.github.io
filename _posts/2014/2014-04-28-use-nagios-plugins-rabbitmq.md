---
layout: post
title: 使用nagios-plugins-rabbitmq监控RabbitMQ
category: 运维
description: 开始使用传说中的终极Shell--Zsh
tags: ["Zsh"]
---

最近在寻找监控RabbitMQ的方法，有个[nagios-plugins-rabiitmq][1]看起来很不错，通过RabbitMQ的API进行多种数据的采集，现在已经包含8个监控脚本了。另外看到[一篇博客][2]说的是如何使用这个插件，写的很好，我想直接翻译过来，但是他博客的[协议][3]不允许，而且也没联系上他，只能作罢。另外他的文章中有一些错误，干脆自己动手写篇，不过难免存在重复。

------

## 插件安装
其实就是个下载：

```sh
$ git clone https://github.com/jamesc/nagios-plugins-rabbitmq.git
$ rm -rf nagios-plugins-rabbitmq/.git/
$ mv nagios-plugins-rabbitmq /usr/local/nagios/libexec/
```

然后用一下脚本，测试自己的Nagios是否缺失Perl插件支持。

```sh
$ cd /usr/local/nagios/libexec/nagios-plugins-rabbitmq/scripts/
$ ./check_rabbitmq_server
Can't locate Nagios/Plugin.pm in @INC (@INC contains: /usr/local/lib64/perl5 /usr/local/share/perl5 /usr/lib64/perl5/vendor_perl /usr/share/perl5/vendor_perl /usr/lib64/perl5 /usr/share/perl5 .) at ./check_rabbitmq_server line 12.
BEGIN failed--compilation aborted at ./check_rabbitmq_server line 12.
```

在我的环境就报了上面的错误，下面安装需要的模块。

## 安装Nagios的Perl插件模块
使用`cpan shell`安装依赖的Perl模块：

```
cpan> install Math::Calc::Units
cpan> install Config:Tiny
cpan> install JSON
```

然后安装Nagios插件模块：

```sh
$ wget http://search.cpan.org/CPAN/authors/id/T/TO/TONVOON/Nagios-Plugin-0.36.tar.gz
$ tar xvfz Nagios-Plugin-0.36.tar.gz && cd Nagios-Plugin-0.36
$ perl Makefile.PL && make && make install
```

这时候重新执行上面的命令，效果大概如下：

```sh
$ ./check_rabbitmq_server
Usage: check_rabbitmq_server [options] -H hostname
Missing argument: hostname
```

## 使用说明

目前这个监控插件里共有8个脚本，下面将分别说明使用方法。这些脚本默认使用`guest/guest`这个账号来连接RabbitMQ的API，如果你在RabbitMQ中修改过的话，使用下面的脚本的时候需要添加`-u "username" -p
"passwd"`参数，下面进行说明的时候就省略这组参数了。
另外，RabbitMQ作为Erlang服务，服务节点名字(sname)一般是`rabbit@mq-server`，所以下面脚本里的`-H`参数值不能是IP，否则无法联通。如果没有使用DNS的话，在Nagios服务器的hosts里进行映射，当然，这个只是测试时需要，Nagios本身不依赖DNS解析。

### `check_rabbitmq_server`
这个脚本可以用来检测RabbitMQ节点的基本状态，比如内存使用率等，和登陆Web管理界面后在首页看到的内容类似，效果如下：

```sh
$ ./check_rabbitmq_server -H "mq-server" --port=15672
RABBITMQ_SERVER OK - Memory OK (0.30%) Process OK (0.26%) FD OK (27.83%) Sockets OK (31.36%) | Memory=0.30%;80;90 Process=0.26%;80;90 FD=27.83%;80;90 Sockets=31.36%;80;90
```

各个参数的默认Warn阈值都是80%，Critical阈值都是90%，可以通过参数进行调整，见后续示例。

### `check_rabbitmq_overview`
这个脚本用来检查整个集群以下三个指标并设定阈值：

1. messages_total
2. messages_ready
3. messages_unacknowledged

使用示例如下：

```sh
$ ./check_rabbitmq_overview -H "mq-server" --port=15672 -w 100,10,10 -c 200,20,20
RABBITMQ_OVERVIEW CRITICAL - messages_unacknowledged CRITICAL (37), messages OK (37) messages_ready OK (0) | messages=37;100;200 messages_ready=0;10;20 messages_unacknowledged=37;10;20
```

因为设定的Critical阈值是200、20、20，`messages_unacknowledged`的数量是37，所以检测结果是`CRITICAL`。

### `check_rabbitmq_objects`
这个脚本会检测集群的一系列数据的值，不过没有阈值或数值对比，只要对应API访问正常就返回`OK`。需要注意的是，部分参数需要账号对对应的Vhost具有访问权限才能够获取，一个类型为monitoring的账号在添加所有Vhost读权限前后返回的结果如下：

```sh
$ ./check_rabbitmq_objects -H "mq-server" --port=15672
# 默认权限
RABBITMQ_OBJECTS OK - Gathered Object Counts | vhost=5;; exchange=0;; binding=0;; queue=0;; channel=258;;
# 添加多个Vhost读权限后
RABBITMQ_OBJECTS OK - Gathered Object Counts | vhost=5;; exchange=50;; binding=38;; queue=19;; channel=258;;
```

### `check_rabbitmq_queue`
这个脚本还是比较有用的，检测的内容和`check_rabbitmq_queue`有点重复，不过是队列级别的，示例如下，其中`consumers`是`consumers_total`而非`consumers_active`。

```sh
$ ./check_rabbitmq_queue -H "mq-server" --port=15672 --vhost='vhostname' --queue='queuename'
RABBITMQ_QUEUE OK - messages OK (0) messages_ready OK (0) messages_unacknowledged OK (0) consumers OK (6) | messages=0;; messages_ready=0;; messages_unacknowledged=0;; consumers=6;;
```

也可以类似上面的参数来设定阈值。

### `check_rabbitmq_aliveness`
这个脚本用来检测RabbitMQ是否可用，默认是在`/`这个Vhost中添加一个名为`aliveness-test`的队列然后进行读写测试，也可以指定使用其他Vhost进行测试，需要注意的是账号需要对待测的Vhost具有所有权限(配置及读写)，完成测试后这个队列并不会被自动删除。
脚本直接返回测试结果：

```sh
$ ./check_rabbitmq_aliveness -H "mq-server" --port=15672 --vhost='test'
RABBITMQ_ALIVENESS OK - vhost: test
```

### `check_rabbitmq_watermark`
RabbitMQ中有个水位线设置，内存或磁盘达到此阈值后就停止接受消息，这个脚本就是调用API来返回是否有内存或磁盘达到使用阈值的。

```sh
$ ./check_rabbitmq_watermark -H "mq-server" --port=15672
RABBITMQ_WATERMARK OK - mem_alarm disk_free_alarm
```

### `check_rabbitmq_partition`和`check_rabbitmq_shovels`
我也是刚刚接手RabbitMQ，目前并没有使用[分区冗余][4]和[Shovel插件][5]，所以这两个脚本暂时没法演示了，不过从脚本内容上来看返回都比较简单，前者检测网络分区是否发生，后者检测Shovel插件存活状态。如果后续用到了这两项功能，再回过头来补充。
这些脚本都比较好懂，一般来说看看脚本里的`url`定义和对应的[RabbitMQ API的说明][6]，就知道作用了。

## Nagios的相关配置
通过在命令行使用应该对这些脚本的功能和用法有些认识了，下面以`check_rabbitmq_queue`为例来说明如何在Nagios使用这些脚本。
首先添加Nagios命令，在我的环境里是编辑`/usr/local/nagios/etc/objects/commands.cfg`添加以下内容：

```
define command {
    command_name check_rabbitmq_queue
    command_line $USER1$/nagios-plugins-rabbitmq/scripts/check_rabbitmq_queue -H $ARG1$ --port=$ARG2$ -u $ARG3$ -p $ARG4$ --vhost=$ARG5$ --queue=$ARG6$ -w $ARG7$ -c $ARG8$
}
```

然后在服务配置里添加类似下面的内容：

```
define service {
    host_name               mq-server
    service_description     check-mq-queue
    check_period            24x7
    max_check_attempts      4
    normal_check_interval   1
    retry_check_interval    2
    contact_groups          testgroup
    notification_interval   10
    notification_period     24x7
    notification_options    w,u,c,r
    check_command           check_rabbitmq_queue!mq-server!15672!user!passwd!vhost!queue!1000,900,100,20!2000,1800,200,40
    }
```

重新加载Nagios配置观察即可，其他命令脚本更少，添加方式类似。

## 补充
另外，还可以通过Ganglia插件来采集RabbitMQ数据并使用Nagios进行监控。目前官网Github已经[RabbitMQ插件][7]了，不过连缩进都有问题，不知道怎么提交上去的，不过从[Pull Requests][8]中发现了一个改进版，可以试试使用[这个插件][9]。用这种方式的优点是，不但能够监控异常，还能够可视化地追踪各指标变化情况，当然缺点就是你得先有个Ganglia，而且增大了耦合。
关于如何在Nagios使用Ganglia数据过几天我会另起文章，不过网上也有很多说明了，自己找找。

[1]: https://github.com/jamesc/nagios-plugins-rabbitmq
[2]: http://www.thegeekstuff.com/2013/12/nagios-plugins-rabbitmq/
[3]: http://www.thegeekstuff.com/copyright/
[4]: https://www.rabbitmq.com/partitions.html
[5]: https://www.rabbitmq.com/management.html
[6]: http://hg.rabbitmq.com/rabbitmq-management/raw-file/rabbitmq_v3_3_0/priv/www/api/index.html
[7]: https://github.com/ganglia/gmond_python_modules/tree/master/rabbit
[8]: https://github.com/ganglia/gmond_python_modules/pull/140
[9]: https://github.com/cburroughs/gmond_python_modules/tree/babbitty/rabbit
