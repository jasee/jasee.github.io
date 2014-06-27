---
layout: post
title: Docker--存储
category: 运维
description: 了解Docker对于需要分享或保留的数据的处理方式。
tags: ["Docker", "虚拟化", "Container", "Volume"]
---

Docker的使用了一个叫AUFS的文件系统，可以增量的改变及增加文件，并最终合并为一个新的文件系统，就像叠蛋糕一样。镜像是一个有很多层的只能看不能吃的蛋糕，容器是在这个蛋糕的基础之上，[增加了可读写的一层来进行进一步的DIY][1]，一旦在容器中完成了修改并提交，就生成了一个新的只读的镜像了。在这种机制之下，容器产生的数据其实以容器已有的内容作为根基的，如果容器挂了或者被删除了，那么这些数据也就丢失了，如果想要升级镜像，数据也无法传承。如果想要加固这些数据，那么得找一种不依赖这种柔软的、容易被吃掉的底座才行。

比如我有一个运行Redis的容器，如果直接将RDB文件存在容器的内，那么如果我需要更新Redis版本，那么更新后提交为镜像时就将RDB文件也包含在内了，这显然不是我们希望的。或者我从别处获取了一个新的Redis镜像，想要用它替换现有的容器，那么现有容器中的Redis数据就很难保留了。

### 数据卷
数据卷(Data volumes)是Docker提供的一种特别的目录，这种目录不使用上面提到的AUFS，可以用来做数据分享或保留。主要特性有：

1. 针对这个特殊目录(数据卷)所做的修改是直接生效的，不会像AUFS那样进行糅合。
2. 数据卷可以在多个容器之间重用或者分享。
3. 将容器提交为镜像的时候，数据卷的内容及改变不会包含在内。
4. 当没有容器继续使用数据卷时数据卷依然会保留。

这些特性配合下文提到的方法，基本就解决了之前的问题。

### 使用数据卷
在启动容器的时候，使用`-v`参数就可以挂载一个数据卷。`-v`参数可以多次使用以便挂载多个数据卷目录。

```sh
$ docker run -d -v /logs ubuntu:14.04 /bin/sh -c 'while true; do date >> /logs/date.log; sleep 1; done'
```

上面的命令建立了数据卷`/logs`并定时写入日志。在Dockerfile中使用`VOLUME`指令可以达到同样的效果。

### 映射到本地目录
有些人出于方便、直观或者安全(我不知道这样是不是更安全)的原因，会想让Docker直接把数据写到主机本地来，Docker提供了这样的功能，示例如下：

```sh
$ docker run -d -v /work/docker/logs:/logs ubuntu:14.04 /bin/sh -c 'while true; do date >> /logs/date.log; sleep 1; done'
```

这样就将本地的`/work/docker/logs`目录挂载为容器内的`/logs`目录，查看本地的`/work/docker/logs/date.log`可以看到容器在不断的写入内容。如果本地目录只是为了提供信息并且担心被容器不小心破坏的话，还可以采用只读方式挂载，对应的参数形式为`/work/docker/logs:/logs:ro`。需要注意的是本地路径必须以绝对路径形式给出，如果路径不存在Docker会自动创建一个。这个功能在Dockerfile中是没有对应的指令的，因为这种操作会破坏Dockerfile分享时的主机独立性。

### 使用数据卷容器
这是一个类似于网络存储的概念，通过建立一个拥有数据卷的容器可以很方便的进行数据分享。
首先建立一个数据卷容器：

```sh
$ docker run -d -v /logs --name log_home ubuntu:14.04 /bin/sh -c 'while true; do date >> /logs/date.log; sleep 10; done'
```

这样就建立了一个名为log_home的容器。关于`--name`参数在以后说到容器连接的时候再细说，暂时知道使用这个名字就能连接到对应容器即可。
下面我们再建立一个容器并挂载刚刚这个容器的数据卷：

```
$ docker run -d --volumes-from log_home --name logger01 ubuntu:14.04 /bin/sh -c 'while true; do echo "I am logger01" >> /logs/date.log; sleep 10; done'
```
还可以建立多个挂载log_home的容器。而且，挂载是可以是串行的，比如：

```sh
$ docker run -d --volumes-from logger01 --name logger02 ubuntu:14.04 /bin/sh -c 'while true; do echo "I am logger02" >> /logs/date.log; sleep 10; done'
```

这样就把logger01的`/logs`目录挂载到容器logger02上了。实际上无论通过哪个容器挂载，最后都是同一个映射，被挂载的容器后续存在与否都没有任何影响。

通过使用`docker inspect -f {{.Volumes}}`命令分别查看这三个容器的挂载信息，可以发现他们是映射到本地的同一个目录的，在我的主机上是`/var/lib/docker/vfs/dir/3f4...`。可见Docker把数据卷这种特殊的目录单独放在`/var/lib/docker/vfs`下了。

删除log_home、logger01或者logger02中的一到两个对数据卷映射没有影响，因此我们可以很方便的在容器间进行升级或数据迁移：停止当前实例，开启一个新的容器挂载之前容器的所有数据卷，删除原有容器。当我们把数据卷对应的这三个容器都删除时，映射关系就消失了，本地的数据依然存在，但不大好找。这和硬链倒是挺像。

当挂载一个容器的数据卷时，将挂载该容器的所有数据卷。另外`--volumes-from`选项可以多次使用，将多个容器的所有数据卷全部挂载，经测试，如果多个容器拥有重复的数据卷名称，将挂载第一个。

另外发现`docker cp`命令无法拷贝数据卷的内容。

### 数据卷的备份、恢复及迁移
其实知道使用`inspect`命令能够获取数据卷与本地的映射关系后，读取上已经能够直接对数据卷进行操作了，不过还是推荐使用Docker自带的方法，谁知道有没有其他坑呢。

1. 将log_home容器的`/logs`目录备份到当前目录：

    ```sh
    $ docker run --volumes-from log_home -v $(pwd):/backup ubuntu:14.04 tar cvf /backup/backup.tar /logs
    ```
2. 新建容器(或者是恢复建立上个命令的容器)

    ```sh
    $ docker run -v /logs --name log_home_new ubuntu:14.04 /bin/bash
    ```
3. 恢复备份数据至新容器

    ```sh
    $ docker run --volumes-from log_home_new -v $(pwd):/backup ubuntu:14.04 tar xvf /backup/backup.tar
    ```

以上步骤通过将数据中转到主机本地进行保存和使用，比较适合的场景是：容器异常的恢复，跨主机容器的数据传输，部分数据的分享，容器间后续的数据独立。如果是同一个主机的两个容器并且没有安全问题和后续数据冲突的话，还是直接挂载比较方便。也可以看出，`-v`和`--volumes-from`结合起来还是比较灵活的。

[1]: http://docs.docker.com/terms/layer/#ufs-def
