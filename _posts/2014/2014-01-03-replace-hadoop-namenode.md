---
layout: post
title: 替换Hadoop Namenode
category: 服务
description: 对于没有进行Namenode HA的Hadoop集群来说，可能会有替换Namenode的需求，本文记录了进行机器替换的步骤。
tags: ["Hadoop","Namenode"]
---

对于没有进行Namenode HA的Hadoop集群来说，可能会有替换Namenode的需求，如硬件老化故障、性能问题等。[通过使用同一套主机公密钥](/2013/11/05/ssh-publickey/)可以避免Namenode对其他节点的密钥认证失效问题，尽量透明的进行机器替换。
这个记录也可以作为Namenode故障时SecondaryNamenode切换的参考。

### 1. 准备
新机器上架，主机名与namenode相同并进行环境同步。注意以下

```sh
scp /etc/ssh/ssh_host_rsa_key{,.pub} newNamenode:/etc/ssh #同步主机公密钥，默认rsa认证
vim ~/.ssh/known_hosts #删除刚刚增加的newNamenode的记录
scp -r ~/.ssh newNamenode:~/
```

此时可以验证新机器keytab是否有效、ssh信任关系等。

------

### 2. 重启前

1. 关闭所有监控。
2. 停止hadoop job。

### 3. 重启
1. 关闭hadoop集群。
2. 复制namenode的`dfs.name.dir`到新机器。
3. 修改namenode网络配置，调整IP地址并重启网卡。
4. 老namenode可以连通后，修改新机器IP至namenode，重启网卡。
5. 在新namenode上启动hadoop。

此时可以开始校验Hadoop并启动job。
