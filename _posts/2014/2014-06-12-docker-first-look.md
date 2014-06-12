---
layout: post
title: Docker--初识
category: 运维
description: 随着Docker1.0的及相关镜像构建、容器维护工具及共享协作平台的发布，其生态环境越来越成熟，已经可以正式部署到生产环境中了。云计算这波浪潮来势汹汹，不知道未来的运维之路走向何方啊。了解一下Docker，防止不知不觉就被革了命。
tags: ["Docker", "虚拟化", "Container", "Dockerfile", "DockerHub"]
---

上周开始了解Docker，最近几天有事没看，今天发现[Docker1.0都发布了](http://blog.docker.com/2014/06/its-here-docker-1-0/)，Goggle也出了容器管理工具[Kubernetes](https://github.com/GoogleCloudPlatform/Kubernetes)和容器资源统计工具[Cadvisor](https://github.com/google/cadvisor)，真是日新月异。趁着今天后半夜加班，屁颠屁颠的升级到Docker1.0，争取早点把头挤进Docker的门缝。

------

虽然对PaaS了解不多，但是觉得这东西搞不好要革运维的命，如果运维的方向是把公司的资源搞成阿里云那样的，那么学学Docker就很有必要了。我也是刚接触，具体的说不清楚，边学边用边理解吧，网上有很多关于Docker的介绍：

* [Docker入门实战][1]
* [PaaS乱局：Container的新机遇][2]
* [Docker 快速入门][3]
* [Docker能够运行任何应用的“PaaS”云][4]

官方的[交互式教程][5]和[Dockerfile教程][6]以及[相关的文档][7]也非常不错。

### 安装
安装比较简单，我直接按照[官方文档][8]和[阿里云支持docker吗][9]安装的，记录一下做个备忘。

```sh
# 修改DNS配置，默认的DNS连index.docker.io都解析不了，用8.8.8.8吧
$ vim /etc/resolv.conf
# 删除aliyun默认172.16网段的路由配置。docker需要用到该网段，不删除会有冲突
$ route del -net 172.16.0.0 netmask 255.240.0.0
# 注释对应route配置
$ vim /etc/network/interfaces
# 更新内核
$ apt-get update
$ apt-get install linux-image-generic-lts-raring linux-headers-generic-lts-raring
$ reboot
# 安装docker
$ apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 36A1D7869245C8950F966E92D8576A8BA88D21E9
$ sh -c "echo deb https://get.docker.io/ubuntu docker main > /etc/apt/sources.list.d/docker.list"
$ apt-get update
$ apt-get install lxc-docker
# 测试，第一次运行需要pull ubuntu，较慢
$ docker run -i -t ubuntu /bin/bash
```

### 在Docker中搭建Web服务器
通过尝试在Docker中搭建一个Web服务来学习其基本用法。

1. 使用最爱的Centos的镜像，然后部署Httpd服务。

    ```sh
    $ docker pull centos
    # 使用images命令查看现有镜像
    $ docker images
    # 使用Centos6的镜像启动一个容器并进入
    $ docker run -it centos:centos6 /bin/bash
    [Container]$ yum install -y httpd vim
    [Container]$ echo "I come from docker" > /var/www/html/index.html
    # 由于容器在执行完一条命令后就会退出，需要写个脚本HOLD住
    [Container]$ vim /home/http-start.sh
    #!/usr/bin/env bash
    service httpd start
    while true; do
        sleep 1000
    done
    ```
2. 提交快照。就像Git中提交代码一样，需要将容器提交为镜像才能进一步使用。

    ```sh
    # 找到刚刚创建的容器的ID
    $ docker ps -a
    # 将该容器保存为镜像
    $ docker commit 2bda2b092def jasee/httpd
    # 查看本地现有镜像
    $ docker images
    ```
3. 启动服务

    ```sh
    $ docker run -d -p 80 jasee/httpd bash /home/http-start.sh
    $ docker ps #应该有个容器在运行了，还可以看到PORTS映射关系
    # 使用以下命令也可以看到主机本地端口
    $ docker port 02797434fa24 80
    ```

在我的环境里，此时通过主机的的49153端口就可以看到"I come from docker"了。

### Dockerfile
之前的步骤都可以写在Dockerfile里，方便维护修改及分享和模板化，Dockerfile如下：

```sh
# VERSION 0.0.1
FROM centos:centos6
MAINTAINER jasee "jaseetao@gmail.com"
RUN yum install -y httpd
RUN echo "I come from docker" > /var/www/html/index.html
ADD http-start.sh /home/http-start.sh
EXPOSE 80
ENTRYPOINT bash /home/http-start.sh
```

其中`ADD`命令可以将本地或URL提供的文件拷贝到容器中。在此例中已经在本地写好`http-start.sh`并放到Dockerfile同一目录下。
有了Dockerfile之后，就可以使用如下命令建立镜像：

```sh
$ docker build -t jasee/httpd:0.0.1 .
```

尝试运行：

```sh
$ docker run -d -p 80 jasee/httpd:0.0.1
```

又一个Web服务建立起来了。

### DockerHub
Docker1.0发布了，[DockerHub][10]也建起来了。感觉Docker的整个生命周期与生态模式和Git及GitHub非常类似。注册完DockerHub之后，可以建立仓库，将自己的镜像推送到上面。推送命令如下：

```sh
$ docker login
$ docker push jasee/httpd
```

看样子只会推送变更部分，不过就算这样速度也还是比较慢的。DockerHub好像支持从GitHub里读取Dockfile自动构建镜像，这个比较有意思，也许能缓解镜像上传的网络问题。
近期你可能能够看到我这次[测试生成及上传的镜像][11]。

### 后记
随着Docker1.0的及相关镜像构建、容器维护工具及共享协作平台的发布，其生态环境越来越成熟，已经可以正式部署到生产环境中了。云计算这波浪潮来势汹汹，不知道未来的运维之路走向何方啊。
努力向前，共勉。





[1]: http://www.cngump.com/blog/2013/12/29/docker/
[2]: http://www.chinacloud.cn/show.aspx?id=13339&cid=17
[3]: http://cn.soulmachine.me/blog/20131026/
[4]: http://www.yankay.com/docker-paas-for-any-application/
[5]: https://www.docker.io/gettingstarted/#
[6]: https://www.docker.io/learn/dockerfile/level1/
[7]: http://docs.docker.io/
[8]: http://docs.docker.io/installation/ubuntulinux/#ubuntu-precise-1204-lts-64-bit
[9]: http://bbs.aliyun.com/read/152090.html
[10]: https://hub.docker.com/
[11]: https://registry.hub.docker.com/u/jasee/httpd/
