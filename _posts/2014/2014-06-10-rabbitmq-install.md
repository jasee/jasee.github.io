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

### 理想拓扑
目前理想的拓扑如下。

```
                                                         VIP
                                                          |
                                              +-----------+--------------+
                                              |                          |
                                  ............v...... HeartBeat .........v.........
                                  .     KeepAlived  .<--------->.     KeepAlived  .
                                  ...................           ...................
                                  .       HAProxy---------+-------------HAProxy   .
                                  .                 .     |     .                 .
                                  .         +-----------Rolling------------+---------------------------+
                                  .         |       .           .          |      .                    |
      +-----------------+         ..........v........           ...........v.......         ...........v.......
      |     rabbitmq01  |         |     rabbitmq02  |           |     rabbitmq03  |         |     rabbitmq04  |
      |  RabbitMQ       |         |  RabbitMQ       |<--------->|  RabbitMQ       |<------->|  RabbitMQ       |
      |    Disk         |         |    Ram          |   Mirror  |    Ram          |  Mirror |    Ram          | ...
      |    Stats        |<------->|                 |<--------->|                 |<------->|                 |
      +-----------------+ Cluster + ----------------+  Cluster  + ----------------+ Cluster + ----------------+
```

是否可行尚未验证，对HAProxy不熟悉。如果不是因为资源有限，可以考虑LVS，由于LVS负载均衡是基于IP的，因此LVS主从和服务节点不能在一起，否则VIP的流量会在主从之间弹几次。
