---
layout: post
title: 建立本地Git仓库中心
category: 工具
tagline: 提升私人本地库安全系数和使用体验
description: 有一些Git本地库不适合推送到github上，但是只有一个本地库既存在被误删的可能，也缺失管理体验，为此可以建立一个本地仓库中心，用来建立远程库。
tags: ["Git"]
---
{% include JB/setup %}

#### 缘起
接触Git不久，虽然我不写多少代码，但是也非常喜欢Git来管理一些脚本和配置文件。并不是所有东西都适合推到github上的，但是也听说过“只用本地仓库的Git算不上真正的Git“这句话。确实，如果只有一个本地库的话，操作起来难免有所担心，如果把整个本地库直接`rm -rf`了哭都来不及。能通过理智手段解决的问题就不要留着考验自己的意志了，为了不湿鞋，在某个一般不会去操作的目录建个`local depot`，这样有了远程库，本地库就更有Git的感觉了，也不用太担心手抖。当然，磁盘还是要定期备份的，否则一锅端呐。

#### 方法
假设我们已经有了一个本地库`repo`，希望在另外一个路径下为其建立远程库，操作如下：

```sh
$ mkdir /path/to/remote/repo.git
$ cd /path/to/remote/repo.git
$ git init --bare
$ ls
HEAD        branches    config      description hooks       info        objects     refs
$ cd /path/to/local/repo
$ git remote add origin /path/to/remote/repo.git
$ git push origin master
```

这样我们就建立了远程库并完成本地库对其第一次推送。上述操作有两处较为特别，远程库目录名以`.git`结尾并且执行`git init`的时候使用了`--bare`选项，从输出中可以看出，这个远程库是不具有工作目录的。虽然本质上来说，Git库都具有同等地位，但是逻辑上来说被大家用以交流分享的库依然具有*中央库*的含义，Git版本库主要分为两种，分别被称为`bare repository`及`development repository`，对应着*中央库*和本地库，前者不应该作为日常开发使用，使用`--bare`选项创建的正是这一种。之所以存在`bare repostitory`，是因为如果*中央库*具有工作目录的话，在其中进行修改时候如果有其他开发者推送更新则产生冲突。今天需要建立的远程库不需要直接对其修改，使用`bare repository`类型。通常使用`.git`结尾的目录来表示该库是一个`bare repository`。

#### 其他
后续如果从位于本地文件系统的远程库克隆，可以使用`git clone --no-hardlinks /path/to/remote/repo.git /path/to/local/repot`命令。如果不使用`--no-hardlinks`选项，那么本地库不会复制`.git/objects`目录，而是默认进行硬链以节省空间，为了彻底分离建议使用该选项。

如果本地库还没有建立，先建立本地库还是先建立远程库都是可以的，依然参照上面的说明。
