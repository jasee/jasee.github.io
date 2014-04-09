---
layout: post
title: HDFS配额
category: 服务
description: 本文介绍了HDFS配额的基本知识和设置、查看及取消方法。HDFS配额较为原始简单，不过能够满足一般需求，可根据情况使用。
tags: ["Hadoop","HDFS","配额"]
---

HDFS允许管理员对目录进行配额限制，限制类型分为两种:

* Name Quotas。节点配额，能够限制一个目录的子目录及文件数量。当达到配额上限时，创建目录或文件的操作就会失败，也无法移动其他文件或目录到此目录之下。配额目录本身也需要占用一个配额，所以配置为1的目录就只能保持空目录状态了。目录默认不存在配额限制，HDFS允许设定低于现有目录节点数量的配额大小。
* Space Quotas。大小配额，能够限制目录的大小(bytes)。在块分配时，如果剩余空间低于块大小，那么分配失败。如果移动文件到该目录下会导致超出配额，那么移动命令失败。配额限制是计算冗余块的，进行配额限制时要将文件大小乘以冗余量(默认是3)，如果改变文件冗余块数，那么配额也会相应消耗或释放。目录默认也是没有进行空间配额限制的，跟节点配额类似，也可以设置一个超过现有大小的配额。文档上写着"A quota of zero still permits files to be created, but no blocks can be added to the
    files"，但实际上我在Hadoop1.2.1版本上测试时无法将配额设置为0。目录和块的元数据信息是不消耗配额的，但是如果已经超过配额则无法创建目录。

配额限制是保存在fsimage里的，设置或者取消配额会产生一条变更日志。节点配额和大小配额都有上限(Long.Max_Value)，在我的系统上是9223372036854775807(约8.3P)。

配额的设置、取消和查看可以通过以下几条命令来完成:

```sh
$ hadoop dfsadmin -setQuota N dir1 dir2 ... # 设置节点配额
$ hadoop dfsadmin -setSpaceQuota N dir1 dir2 ... # 设置大小配额
$ hadoop dfs -count -q dir1 dir2 ...
# 查看配额情况，前四个分别是节点配额、剩余节点配额、大小配额，剩余大小配额。
# 未设置配额的话对应值分别为none和inf
$ hadoop dfsadmin -clrQuota N dir1 dir2 ... # 取消节点配额限制
$ hadoop dfsadmin -clrSpaceQuota N dir1 dir2 ... # 取消大小配额限制
```

我的测试环境是Hadoop1.2.1，块大小128M，默认复制3份进行冗余。

1. 当大小配额在`3 * 128M`以下时，多小的文件都无法上传。
2. 当大小配额在`3 * 128M`时，多小的文件都只能上传一个。
3. 当大小配额为`3 * 129M`时，先传一个10M的再传一个10K的不行，顺序反过来则都可以上传。

*(文件上传失败时文件名已经建立了，只是没有block)*
确实验证了上述的说法，即文件创建时检查剩余配额大小是否高于所需块大小(而不是文件实际大小)。当然，当配额在TB级的时候这点差异基本可以忽略。

#### *参考文档*
*[Quotas Guide](http://hadoop.apache.org/docs/r1.2.1/hdfs_quota_admin_guide.html)*
