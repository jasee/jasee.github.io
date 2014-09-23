---
layout: post
title: Storm服务及任务监控
category: 运维
description: ZooKeeper、Storm及运行的任务监控及自动重启脚本。
tags: ["Storm", "ZooKeeper", "Monitor"]
---

机房搬迁终于算尘埃落定，一地鸡毛，太折腾。
上周匆匆上了Storm服务，机器不够，ZK都没有冗余，以后补齐了再整理安装步骤。

由于Storm和ZK是`Fast-Fail`系统，对服务进程进行监控还是比较重要的，提供两个脚本，分别用于监控Storm的服务进程(Nimbus, Supervisor及ZK)和活动的任务。

### Storm服务监控

```py
class CheckStorm(object):

    def __init__(self):
        self.cmds={}
        self.cmds["check_zk"]="/bin/ps -efw | \
                grep '/work/zookeeper/bin/' | grep -v grep"
        self.cmds["check_nimbus"]="/bin/ps -efw | \
                grep 'backtype.storm.daemon.nimbus' | grep -v grep"
        self.cmds["check_worker"]="/bin/ps -efw | \
                grep 'backtype.storm.daemon.supervisor' | grep -v grep"
        self.cmds["start_zk"]="/work/zookeeper/bin/zkServer.sh start"
        self.cmds["start_nimbus"]="/work/storm/bin/storm nimbus &"
        self.cmds["start_worker"]="/work/storm/bin/storm supervisor &"

    def check_storm(self, service):
        self.status = 0
        self.restart = 0
        self.check_cmd = self.cmds['check_' + service]
        self.start_cmd = self.cmds['start_' + service]
        self.p = subprocess.Popen(
                self.check_cmd,shell=True, stdout=subprocess.PIPE)
        if self.p.wait() != 0:
            self.status = 1
            self.p = subprocess.Popen(
                    self.start_cmd,shell=True, stdout=subprocess.PIPE)
            time.sleep(10)
            self.p = subprocess.Popen(
                    self.check_cmd,shell=True, stdout=subprocess.PIPE)
            if self.p.wait() != 0:
                self.restart = 1
        return (self.status, self.restart)
```

### Storm任务监控

```py
class StormJobs(object):

    def __init__(self):
        self.start = {}
        self.start['realtime-compute'] = "/work/storm/bin/storm jar \
                /path/to/xxx.jar \
                xxx.topology.TopologyMain realtime-compute"

    def get_active(self):
        self.cmd="/work/storm/bin/storm list"
        self.p = subprocess.Popen(self.cmd,shell=True, stdout=subprocess.PIPE)
        if self.p.wait() == 0:
            self.lines = self.p.stdout.readlines()[3:]
        else:
            raise MyError("Can't get Storm Job List")
        self.actives = []
        for self.line in self.lines:
            if self.line.split()[1] == 'ACTIVE':
                self.actives.append(self.line.split()[0])

    def check_job(self, job):
        if job in self.actives:
            return (0, 0)
        else:
            self.p = subprocess.Popen(
                    self.start[job],shell=True, stdout=subprocess.PIPE)
            time.sleep(15)
            self.get_active()
            if job in self.actives:
                return (1, 0)
            else:
                return (1, 1)
```

然后根据返回结果记日志及发送邮件或短信报警。
