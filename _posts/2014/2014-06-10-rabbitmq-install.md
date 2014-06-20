---
layout: post
title: 部署RabbitMQ
category: 服务
description: 学习部署一个兼具性能和可用性的RabbitMQ集群。
tags: ["RabbitMQ", "消息队列", "HAProxy"]
---

这两天开始看RabbitMQ，目前在使用的环境是之前的老兄搭的，只是起了两个节点做了普通的集群，安装方式也比较奇怪。希望能够整理一下，把剩下的节点加进去，把冗余性提上来。

### 安装步骤
在CentOS6上直接使用EPEL安装。

```sh
# 安装Erlang环境
$ rpm -Uvh http://mirror.chpc.utah.edu/pub/epel/6/i386/epel-release-6-8.noarch.rpm
$ yum install erlang
# 安装RabbitMQ
$ wget https://www.rabbitmq.com/releases/rabbitmq-server/v3.3.1/rabbitmq-server-3.3.1-1.noarch.rpm
$ rpm --import http://www.rabbitmq.com/rabbitmq-signing-key-public.asc
$ yum install rabbitmq-server-3.3.1-1.noarch.rpm
```

### 配置

1. Erlang Cookie配置

    将安装时自动生成的`/var/lib/rabbitmq/.erlang.cookie`文件分发到所有节点上，保持一致。
2. 启用管理插件

    ```sh
    $ rabbitmq-plugins enable rabbitmq_management
    ```

3. 配置环境，将数据写到SSD盘(路径按照实际环境)

    ```sh
    $ mkdir -p /work/rabbitmq/mnesia /work/rabbitmq/log
    $ chown rabbitmq:rabbitmq  -R /work/rabbitmq
    # 新建/etc/rabbitmq/rabbitmq-env.conf文件，写入以下内容
    MNESIA_BASE=/work/rabbitmq/mnesia
    LOG_BASE=/work/rabbitmq/log
    ```

4. 启动服务

    ```sh
    service rabbitmq-server start
    ```

5. 除主节点以外，其他节点还需执行以下命令组建集群

    ```sh
    $ rabbitmqctl stop_app
    $ rabbitmqctl join_cluster --ram rabbit@rabbitmq01
    $ rabbitmqctl start_app
    ```

6. 更新用户及权限

    ```sh
    # 增加管理员admin
    $ rabbitmqctl add_user admin password
    $ rabbitmqctl set_user_tags admin administrator
    # 以后增加vhost时如有需要需要继续安装以下命令设置，或者通过Web管理界面设置
    $ rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
    # 删除RabbitMQ第一次启动时默认建立的guest用户
    $ rabbitmqctl delete_user guest
    ```

### 冗余和持久化

按照我目前的理解，RabbitMQ的持久化能力及冗余性从低到高排列如下：

1. 自动删除

    建立队列时，如果将`AUTO DELETE`参数设置为"True"，那么当所有连接到该队列的消费者都断开后，队列自动删除。
2. 默认

    默认交换机及队列建立后不进行持久化，虽然消费端都断开后队列不自动删除，但是当节点重启后，交换机及队列消失。
3. 持久化

    交换机和队列在建立时可以分别设定为持久化保存，当两者都持久化时，对应的绑定规则也会持久化。持久化的交换机、队列及绑定规则在节点重启后继续存在。
4. 消息内容持久化

    在3的基础之上，可以设置消息分发模式为2，此时每条消息都会存储在磁盘上，节点重启时队列内容不会消失。如果对可靠性要求不高，可以设置为1，能够加快速度。
5. 队列镜像

    在普通集群模式，通过其他节点存取某节点建立的队列的消息时，两个节点之间只建立了一个临时通道，消息实际上只存在于建立该队列的节点上。如果队列所在节点宕机，那么该队列内容无法访问，并且由于队列持久化的原因，无法在其他节点重建相同的队列，只能等待故障节点恢复。当开启队列镜像后，队列内容会同步到每个镜像节点之上，任何节点异常队列都可以继续访问。

镜像队列最为安全，但是开销最大，每一条消息的存取都会在所有镜像节点之间同步。实际使用时各个队列根据需求使用不同的持久化及冗余模式。

### 使用及测试

1. 测试环境结构

    ```
                                Service IP
                                    |
          +-------------------------+----+      +--------------------+
          |     rabbitmq01       HAProxy |      |     rabbitmq02     |
          |    (192.168.1.1)   +----+----+      |    (192.168.1.2)   |
          |  RabbitMQ          |    | Polling   |  RabbitMQ          |
          |    Disk            |<---+---------->|    Ram             |
          |    Stats           |<-------------->|                    |
          +--------------------+  Cluster Sync  + -------------------+
    ```
2. 队列镜像

    ```sh
    $ rabbitmqctl add_vhost tao
    $ rabbitmqctl add_user jasee password
    $ rabbitmqctl set_user_tags jasee management
    $ rabbitmqctl set_permissions -p tao jasee ".*" ".*" ".*"
    # tao vhost的所有队列开启镜像，更多参数见官方文档
    $ rabbitmqctl set_policy -p tao ha-all "^" '{"ha-mode":"all","ha-sync-mode":"automatic"}'
    ```
3. 搭建HAProxy

    ```sh
    $ yum install haproxy
    $ cat /etc/haproxy/haproxy.cfg
    global
        log         127.0.0.1 local2
        chroot      /var/lib/haproxy
        pidfile     /var/run/haproxy.pid
        maxconn     4000
        user        haproxy
        group       haproxy
        daemon
        stats socket /var/lib/haproxy/stats

    defaults
        log                 global
        mode                tcp
        option              tcplog
        option              dontlognull
        retries             3
        option              redispatch
        maxconn             3000
        timeout connect     5s
        timeout client      1d
        timeout server      1d

    listen rabbitmq_cluster 0.0.0.0:5670
        mode tcp
        balance roundrobin
        server rabbitmq01 192.168.1.1:5672 check inter 2000 rise 2 fall 3
        server rabbitmq02 192.168.1.2:5672 check inter 2000 rise 2 fall 3
    $ service haproxy start
    ```
4. 测试
    使用如下的生产者和消费者脚本，反复关闭两个节点中的任一个，生产和消费都可以继续进行。
    * producer.py

        ```py
        #!/usr/bin/env python
        # -*- coding: utf-8 -*-

        from amqplib import client_0_8 as amqp
        import time

        conn = amqp.Connection(host="192.168.1.1:5670", userid="jasee", password="password", virtual_host="tao", insist=False)
        chan = conn.channel()

        for x in xrange(100000):
            try:
                msg =amqp.Message("this is %s" % x)
                msg.properties["delivery_mode"] = 2
                chan.basic_publish(msg,exchange="test",routing_key="hello")
                time.sleep(1)
            except KeyboardInterrupt:
                print "Stop Send"
                break
            except:
                try:
                    chan.close()
                    conn.close()
                except:
                    pass
                print "Reconnecting..."
                time.sleep(10)
                conn = amqp.Connection(host="192.168.1.1:5670", userid="jasee", password="password", virtual_host="tao", insist=False)
                chan = conn.channel()
                print "Reconnected"
                chan.basic_publish(msg,exchange="test",routing_key="hello")

        chan.close()
        conn.close()
        ```
    * consumer.py

        ```py
        #!/usr/bin/env python
        # -*- coding: utf-8 -*-

        from amqplib import client_0_8 as amqp
        import time

        conn = amqp.Connection(host="192.168.1.1:5670", userid="jasee", password="password", virtual_host="tao", insist=False)
        chan = conn.channel()

        chan.queue_declare(queue="sayHello", durable=True, exclusive=False, auto_delete=False)
        chan.exchange_declare(exchange="test", type="direct", durable=True, auto_delete=False)
        chan.queue_bind(queue="sayHello", exchange="test", routing_key="hello")

        def recv_callback(msg):
            print 'Received: %s' % msg.body

        chan.basic_consume(queue="sayHello", no_ack=True, callback=recv_callback,consumer_tag="testConsumer")
        while True:
            try:
                chan.wait()
            except KeyboardInterrupt:
                chan.basic_cancel("testConsumer")
                print "Consumer End"
                chan.close()
                conn.close()
                break
            except:
                try:
                    chan.close()
                    conn.close()
                except:
                    pass
                print "Reconnecting..."
                time.sleep(10)
                conn = amqp.Connection(host="192.168.1.1:5670", userid="jasee", password="password", virtual_host="tao", insist=False)
                chan = conn.channel()
                chan.queue_declare(queue="sayHello", durable=True, exclusive=False, auto_delete=False)
                chan.exchange_declare(exchange="test", type="direct", durable=True, auto_delete=False)
                chan.queue_bind(queue="sayHello", exchange="test", routing_key="hello")
                chan.basic_consume(queue="sayHello", no_ack=True, callback=recv_callback,consumer_tag="testConsumer")
                print "Reconnected"
        ```

### 使用Keepalived+LVS

做冗余当然就做完全的冗余，有了HAProxy但是HAProxy本身单点的话出了问题也白搭。之前想要使用Keepalived+HAProxy，但是在发现了[怎么样让 LVS 和 realserver 工作在同一台机器上][1]之后，还是决定继续使用LVS+Keepalived的完美组合。LVS比HAProxy简单、干净、纯粹，最重要的是目前使用的比较多。

1. LVS+Keepalived安装步骤

    ```sh
    $ ln -s /usr/src/kernels/2.6.32-358.18.1.el6.x86_64 /usr/src/linux
    $ tar -zxf ipvsadm-1.24.tar.gz
    $ cd ipvsadm-1.24
    $ make && make install
    $ cd ..
    $ yum install openssl-devel openssl popt-devel
    $ tar -zxf keepalived-1.1.20.tar.gz
    $ cd keepalived-1.1.20
    $ ./configure --prefix=/ && make && make install
    ```
2. 配置

    为了节省资源，我们将LVS和RabbitMQ部署在一起，需要在rabbitmq01和rabbitmq02上执行以下命令：

    ```sh
    # 这部分命令在两台服务器上都执行，其中192.168.1.100是VIP
    $ echo 1 > /proc/sys/net/ipv4/conf/lo/arp_ignore
    $ echo 2 > /proc/sys/net/ipv4/conf/lo/arp_announce
    $ echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
    $ echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
    $ ifconfig lo:1 192.168.1.100 netmask 255.255.255.255
    # 此命令在rabbitmq01上执行
    $ iptables -t mangle -I PREROUTING -d 192.168.1.100 -p tcp -m tcp --dport 5672 -m mac ! --mac-source $mac_of_rabbitmq02 -j MARK --set-mark 0x1
    # 此命令在rabbitmq02上执行
    $ iptables -t mangle -I PREROUTING -d 192.168.1.100 -p tcp -m tcp --dport 5672 -m mac ! --mac-source $mac_of_rabbitmq01 -j MARK --set-mark 0x2
    ```
    以上命令可以写入`/etc/rc.local`。另外rabbitmq01、02上的Keepalived配置分别为：
    * rabbitmq01

        ```
        global_defs {
           router_id rabbitmq01
        }

        vrrp_instance rabbitmq {
            state MASTER
            interface eth0
            lvs_sync_daemon_inteface eth0
            virtual_router_id 1
            priority 150
            advert_int 5
            authentication {
                auth_type PASS
                auth_pass passwd
            }
            virtual_ipaddress {
                192.168.1.100
            }
        }

        virtual_server fwmark 1 {
                delay_loop 10
                lb_algo wrr
                lb_kind DR
                persistence_timeout 6000
                protocol TCP

                real_server  192.168.1.1 5672 {
                        inhibit_on_failure
                        weight 10
                        TCP_CHECK {
                                connect_timeout 3
                                nb_get_retry 3
                                delay_before_retry 3
                                connect_port 5672
                        }
                }

                real_server  192.168.1.2 5672 {
                        inhibit_on_failure
                        weight 10
                        TCP_CHECK {
                                connect_timeout 3
                                nb_get_retry 3
                                delay_before_retry 3
                                connect_port 5672
                        }
                }
        }
        ```
    * rabbitmq02

        ```
        global_defs {
           router_id rabbitmq02
        }

        vrrp_instance rabbitmq {
            state BACKUP
            interface eth0
            lvs_sync_daemon_inteface eth0
            virtual_router_id 1
            priority 100
            advert_int 5
            authentication {
                auth_type PASS
                auth_pass passwd
            }
            virtual_ipaddress {
                192.168.1.100
            }
        }

        virtual_server fwmark 2 {
                delay_loop 10
                lb_algo wrr
                lb_kind DR
                persistence_timeout 6000
                protocol TCP

                real_server  192.168.1.1 5672 {
                        inhibit_on_failure
                        weight 10
                        TCP_CHECK {
                                connect_timeout 3
                                nb_get_retry 3
                                delay_before_retry 3
                                connect_port 5672
                        }
                }

                real_server  192.168.1.2 5672 {
                        inhibit_on_failure
                        weight 10
                        TCP_CHECK {
                                connect_timeout 3
                                nb_get_retry 3
                                delay_before_retry 3
                                connect_port 5672
                        }
                }
        }
        ```
3. 目前的结构

    此时启动两台服务器的Keepalived服务即可，此时测试环境的结构为:

    ```
                                           VIP
                                            |
                                +-----------+--------------+
                                |                          |
                    ............v...... HeartBeat .........v.........
                    .     Keepalived  .<--------->.     Keepalived  .
                    ..........+........           ...........+.......
                    .        LVS------------+---------------LVS     .
                    .                 .     |     .                 .
                    .         +-----------DR/wrr-------------+      .
                    .         |       .           .          |      .
                    ..........v........           ...........v.......
                    |     rabbitmq01  |           |     rabbitmq02  |
                    |  RabbitMQ       |<--------->|  RabbitMQ       |
                    |    Disk         |   Mirror  |    Ram          |
                    |    Stats        |<--------->|                 |
                    + ----------------+  Cluster  + ----------------+
    ```
4. 一些说明
    1. 问题

        能够在RS上同时部署LVS主备的关键点就是前面的两条`iptables`语句。如果LVS不是按照数据包标签来进行负载均衡和流量转发，那么会碰到以下问题：
        1. 为了保证切换速度，Keepalived会在LVS备机上也启用LVS。
        2. 主LVS在接收到流量之后，有一半的概率将该数据包发往备LVS所在RS节点。
        3. 备LVS所在RS节点收到数据包后，首先经由ip_vs模块处理，又有一半的概率将数据包转发到主LVS所在的RS节点。
        4. 反复以上过程，产生很多无用流量和乱七八糟的转发。
    2. 解决方法

        通过将不是对面LVS发送的、到指定IP和端口的数据包打标签，然后在LVS里配置(见上述keepalived.conf)转发带有该标签的数据包，可以解决该问题，请求过程为：
        1. 客户端发送对VIP的RabbitMQ端口的访问，主LVS上的防火墙规则将该数据打上标签1。
        2. LVS的配置中配置对标签1进行均衡转发，一半的流量直接本机处理，一半的流量转发到备LVS所在RS节点。
        3. 备LVS所在RS节点的防火墙因发现来源MAC地址为主LVS，不会打标签2。
        4. 备LVS的针对标签2的均衡转发不会生效。
        5. 备LVS所在RS的RabbitMQ正常处理该请求。

按照以上设定，LVS主备和RS可以共存，访问RabbitMQ的流量也可以正常进行均衡。Keepalived将会负责心跳检测和故障时的VIP漂移，一台服务器宕机影响不大。


### 拓扑和其他
目前计划使用的拓扑如下。

```
                                                    VIP
                                                     |
                                         +-----------+--------------+
                                         |                          |
                             ............v...... HeartBeat .........v.........
                             .     Keepalived  .<--------->.     Keepalived  .
                             ..........+........           ...........+.......
                             .        LVS------------+---------------LVS     .
                             .                 .     |     .                 .
                             .         +-----------DR/wrr-------------+      .
                             .         |       .           .          |      .                  IP
 +-----------------+         ..........v........           ...........v.......         +--------+--------+
 |     rabbitmq01  |         |     rabbitmq02  |           |     rabbitmq03  |         |     rabbitmq04  |
 |  RabbitMQ       |         |  RabbitMQ       |<--------->|  RabbitMQ       |         |  RabbitMQ       |
 |    Disk         |         |    Ram          |   Mirror  |    Ram          |         |    Ram          | ...
 |    Stats        |<------->|                 |<--------->|                 |<------->|                 |
 +-----------------+ Cluster + ----------------+  Cluster  + ----------------+ Cluster + ----------------+
```

其中对安全性要求较高的消息走VIP，对速度要求较高的消息走集群剩余节点的节点IP。队列镜像需要主动设定，以下为参考命令：

```sh
# 对vhost tao中的所有队列启用镜像
$ rabbitmqctl set_policy -p tao ha-tao-all "^" '{"ha-mode":"all","ha-sync-mode":"automatic"}'
# 取消刚才的设定
$ rabbitmqctl clear_policy -p tao ha-tao-all
# 对vhost tao中的test队列启用镜像
$ rabbitmqctl set_policy -p tao ha-tao-test "test '{"ha-mode":"all","ha-sync-mode":"automatic"}'
# 后面的设定在此处意义不大，为了性能考虑我们只镜像两个节点，基本不存在增加slave节点的操作。
```

另外，开启队列镜像模式之后对性能的影响还是挺大的。由于线下测试环境较差，数据波动较大，具体数值参考价值不大，只说一下大概。在使用1000字节的消息进行测试时，相比于未开启镜像：

1. 写入速度下降到30%-40%。
2. 读取速度下降到70%左右。
3. 同时读写，速度下降约一半。

100Mb网络+台式机，每秒读写消息在k级别。如果不是因为两个节点中有一个磁盘节点，性能也许会好一些。如果镜像节点更多，性能还会进一步下降，未做测试。对安全性要求不高的队列，还是不要使用镜像模式比较好。

[1]: http://www.linuxde.net/2012/05/10652.html
