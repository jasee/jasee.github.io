---
layout: post
title: OpenLDAP--按用户组限制登录
category: 运维
description: 把一直欠的债还上，OpenLDAP中通过Group Filter只允许指定用户组的用户登录。
tags: ["OpenLDAP","Kerberos","SSH"]
---

把之前欠的记录还上。
在14年的时候，写了几篇博客记录了OpenLDAP安装使用的过程，后来也有人问我，按照这样的安装方法，那所有人都可以登录所有服务器，这怎么行呢。当然是不行的，今天就简单的说明下如何补上这个窟窿。

我们采用的方法是，通过配置系统对OpenLDAP的查询进行过滤，就可以限定只允许某些用户或用户组登陆，其他用户在这个系统中相当于不存在。CentOS5和CentOS6的过滤方法略有不同，CentOS7与CentOS6基本一样。直接贴脚本吧，仅供参考。

```sh
function filter_on_centos5() {
    #samples:
    #nss_base_passwd    dc=opjasee,dc=com?sub?gidNumber=1000
    #nss_base_passwd    dc=opjasee,dc=com?sub?|(gidNumber=1000)(gidNumber=1003)
    cp /etc/ldap.conf backup/filter/
    local groups="$1"
    green "***** Getting filter *****"
    n=$(echo $groups | awk -F',' '{print NF}')
    if [ $n -eq 1 ];then
        gid=$(get_gid $groups)
        [ -z $gid ] && { red "Can't find group $1"; exit 3; }
        filter="gidNumber=$gid"
    else
        filter="|"
        for group in $(echo $groups | sed 's/,/ /g'); do
            gid=$(get_gid $group)
            [ -z $gid ] && { red "Can't find group $group"; exit 3; }
            filter="$filter""(gidNumber=$gid)"
        done
    fi
    filter="nss_base_passwd dc=opjasee,dc=com?sub?""$filter"
    echo $filter
    green "***** Config ldap.conf *****"
    echo $filter >> /etc/ldap.conf
}

function filter_on_centos6() {
    #samples:
    #filter passwd (gidNumber=1000)
    #filter passwd (|(gidNumber=1000)(gidNumber=1003))
    cp /etc/nslcd.conf backup/filter
    local groups="$1"
    green "***** Getting filter *****"
    n=$(echo $groups | awk -F',' '{print NF}')
    if [ $n -eq 1 ];then
        gid=$(get_gid $groups)
        [ -z $gid ] && { red "Can't find group $1"; exit 3; }
        filter="(gidNumber=$gid)"
    else
        filter="(|"
        for group in $(echo $groups | sed 's/,/ /g'); do
            gid=$(get_gid $group)
            [ -z $gid ] && { red "Can't find group $group"; exit 3; }
            filter="$filter""(gidNumber=$gid)"
        done
        filter="$filter"")"
    fi
    filter="filter passwd $filter"
    echo $filter
    green "***** Restart nslcd *****"
    echo $filter >> /etc/nslcd.conf
    service nslcd restart
}
```
