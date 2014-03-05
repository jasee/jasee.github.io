---
layout: post
title: Ganglia--清理网络流量异常数据
category: 运维
tagline: 使用rrdtool清理网络流量异常数据
description: 因为一些网卡驱动异常，ganglia采集的网卡数据中可能出现异常大的数值，可以使用rrdtool修改数据进行清理。
tags: ["Ganglia","Monitor","rrd"]
---
{% include JB/setup %}

使用Ganglia时，可能碰到[网络流量值异常的情况][1]，如果不具备升级网卡的条件，或者不想因为引入Ganglia进行过多的变更，可以考虑修改rrd数据文件来定期清理异常数据。
根据实际观察，异常流量都是PB级的，与正常流量差别数个数量级，可以很容易进行区分。主要思路如下

1. 遍历rrd文件目录，寻找网络相关rrd文件。
2. 使用`rrdtool dump`命令导出xml格式的数据。
3. 将超过一定数值的数据判断为异常流量，使用上一个正常值替换。
4. 使用`rrdtool restore`命令将修复好的xml格式文件转换为rrd文件。

由于gmetad一直在更新rrd数据文件，可能与清理过程冲突导致rrd文件损坏，进而导致gmetad崩溃，因此需要一定的判断及补救逻辑(当然能预防其实是更好的，清理数据这个思路也是一种妥协)。下面提供具体的脚本：

```sh
#!/bin/sh
logger() {
    echo $(date +"%Y-%m-%d %H:%M:%S") "[$1]" "$2" >> $logfile
}

usage() {
    cat << EOF
Usage: gangliafix.sh rrddirectory
EOF
}

fix() {
    local cur_dir  workdir
    workdir="$1"
    cd "$workdir"
    cur_dir=$(pwd)
    logger INFO "start fix $cur_dir"
    for sub in $(ls "$cur_dir"); do
        if [ -d "$sub" -a "$sub" != "backup" ]; then
            cd "$sub"
            fix "$cur_dir/$sub"
            cd ..
        elif [ "$sub" == "bytes_in.rrd" -o "$sub" == "bytes_out.rrd" -o "$sub" == "pkts_out.rrd" -o "$sub" == "pkts_in.rrd" ]; then
            [ -d "backup" ] || mkdir backup || { logger WARN "create $cur_dir/backup failed";continue; }
            cp -p "$sub" ./backup/
            #异常值被上一个正常值替换，如果异常值在第一行，该值本次不处理。三个命令依赖执行，否则dump异常restore时会block
            rrdtool dump "$sub" temp.xml && sed -i '/<row><v>[0-9]\.[0-9]\{10\}e+0[0-9]<\/v>/h;/<row><v>[0-9]\.[0-9]\{10\}e+1[0-9]<\/v>/{G;s/\(.*\)<row><v>\(.*\)<\/v><\/row>\n.*<row><v>\(.*\)<\/v><\/row>/\1<row><v>\3<\/v><\/row>/}' temp.xml && rrdtool restore -f temp.xml "$sub"
            rrdtool dump "$sub" temp.xml
            #处理该脚本和gmetad同时更新文件引起冲突的问题
            if [ "$?" != "0" ];then
                cp ./backup/"$sub" ./
                logger WARN "$cur_dir/$sub corrupt, rollback"
                sleep 5
                ps -efw | grep gmetad | grep -v grep > /dev/null
                if [ "$?" == "0" ];then
                    logger INFO "gmetad keep running"
                else
                    logger ERROR "gmetad crashed, try to start"
                    /etc/init.d/gmetad start
                    sleep 10
                    ps -efw | grep gmetad | grep -v grep > /dev/null && logger INFO "gmetad start ok" || { logger CRITICAL "gmetad start error, skip the remaining fix"; return 2; }
                fi
            fi
            rm temp.xml
        fi
    done
}

rollback() {
    local cur_dir  workdir
    workdir="$1"
    cd "$workdir"
    cur_dir=$(pwd)
    for sub in $(ls "$cur_dir"); do
        if [ -d "$sub" -a "$sub" != "backup" ]; then
            cd "$sub"
            rollback "$cur_dir/$sub"
            cd ..
        elif [ "$sub" == "bytes_in.rrd" -o "$sub" == "bytes_out.rrd" -o "$sub" == "pkts_out.rrd" -o "$sub" == "pkts_in.rrd" ]; then
            [ -d "backup" ] || { logger WARN "no $cur_dir/backup, skip";continue; }
            cp ./backup/"$sub" ./ && logger INFO "roolback $cur_dir/$sub"
        fi
    done
}

gmetadFail() {
    logger CRITICAL "gmetad crashed, start roll back"
    rollback "$1"
    logger INFO "roll back end,try to start gmetad"
    /etc/init.d/gmetad start
    sleep 20
    if ps -efw | grep gmetad | grep -v grep; then
        logger INFO "gmetad restart ok"
        /usr/local/bin/sendsms 151xxxxxxxx "gmetad crashed, restart ok"
    else
        logger CRITICAL "gmetad restart failed"
        /usr/local/bin/sendsms 151xxxxxxxx "gmetad crashed, restart failed"
    fi
}

# init
JOBHOME=$(cd $(dirname $0);pwd)
[ -d "$JOBHOME"/log ] || mkdir "$JOBHOME"/log
logfile="$JOBHOME"/log/$(date +%Y%m%d)
exlogfile="$JOBHOME"/log/$(date +%Y%m%d --date "30 days ago")
[ -e "$exlogfile" ] && rm "$exlogfile"

if [ -d "$1" ]; then
    logger INFO "start fix"
    fix "$1" && logger INFO "fix ok" || logger ERROR "fix error, going to rollback all"
else
    usage
    exit 2
fi

# rrd文件异常时直接跳至此处，否则作为最终检查
sleep 20
ps -efw | grep gmetad | grep -v grep || gmetadFail "$1"
```

[1]: http://hi.baidu.com/zypublic111/item/2ca8aee8de74332f5b7cfbd8
