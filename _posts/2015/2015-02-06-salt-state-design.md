---
layout: post
title: 从MySQL范例看Salt的State设计
category: 运维
description: 通过官方的MySQL配置实例，学习如何规范合理的进行Salt State配置设计
tags: ["Salt", "MySQL"]
---

对于Salt的配置管理功能来说，一个清晰合理的sls格式及结构是其能够可持续发展的关键。[Salt最佳实践][1]里面提到了以下几条重要规则：

1. 尽可能的清晰和模块化。
2. pillar和salt之间要有清晰的映射关系。
3. 在需要的地方使用变量，但不要滥用。
4. 敏感数据放到pillar中存储。
5. 不要在pillar的敏感数据部分使用grains匹配对应服务器。

为了更深的体会这些规则，我准备分析一下[官方的Mysql配置范例][2]，学习Salt State的设计、结构以及常用语法。

### 目录结构
经过简单整理，该范例的目录结构如下：

```text
.
├── pillar
│   └── mysql.sls                      # pillar与state相对应
└── salt
    ├── mysql                          # 每类服务单独一个目录
    │   ├── client.sls                 # 服务的每个部分单独一个sls，这里是安装mysql客户端
    │   ├── database.sls               # 这里是mysql的db结构配置
    │   ├── defaults.yaml              # 各类系统的默认服务端、客户端等配置
    │   ├── files                      # 文件、模板单独一个目录
    │   │   └── my.cnf                 # mysql server的配置模板
    │   ├── init.sls                   # 当服务器状态为mysql时的配置，类似于默认配置
    │   ├── python.sls                 # python的mysql模块，注意并没有与client混为一谈
    │   ├── remove_test_database.sls   # 删除test db的配置
    │   ├── server.sls                 # mysql server配置
    │   ├── supported_sections.yaml    # 
    │   └── user.sls                   # mysql用户权限配置
    └── scripts                        # 辅助的脚本目录，感觉放到mysql目录下更好
        └── import_users.py            # 将mysql用户权限以合适的格式导入pillar的脚本
```


### 动态适配不同的系统
在[之前测试][3]时，是在pillar中定义不同服务的配置文件路径的，如果按照这个思路，那么可能会在sls和pillar中使用太多if语句。从[vim][4]或者[git][5]等示例的`map.jinja`文件中，可以看出salt已经给出了一种更清晰的解决方案。在本范例中，对应的文件是`defaults.yaml`，从中抽取一个片段：

```yaml
{% raw %}
{% load_yaml as rawmap %}
Ubuntu:
  server: mysql-server
  client: mysql-client
...
...
CentOS:
  server: mysql-server
  client: mysql
  service: mysqld
  python: MySQL-python
  config:
    file: /etc/my.cnf
    sections:
      mysqld_safe:
        log_error: /var/log/mysqld.log
        pid_file: /var/run/mysqld/mysqld.pid
      mysqld:
        datadir: /var/lib/mysql
        socket: /var/lib/mysql/mysql.sock
        user: mysql
        port: 3306
        bind_address: 127.0.0.1
...
...
{% endload %}
{% endraw %}
```

在后续的sls中都可以很方便的调用，比如`client.sls`：

```yaml
{% raw %}
{% from "mysql/defaults.yaml" import rawmap with context %}
{%- set mysql = salt['grains.filter_by'](rawmap, grain='os', merge=salt['pillar.get']('mysql:server:lookup')) %}

mysql:
  pkg.installed:
    - name: {{ mysql.client }}
{% endraw %}
```

这样再也不用将这些数据臃肿的定义在pillar里或者在每个文件里都重复写一遍if了。
其中`load_yaml`和`from ... import ...`是序列与反序列化yaml数据，使用格式如下：

```yaml
{% raw %}
# doc1.sls
{% load_yaml as var1 %}
    foo: it works
{% endload %}
{% load_yaml as var2 %}
    bar: for real
{% endload %}
{% endraw %}
```

```yaml
{% raw %}
# doc2.sls
{% from "doc1.sls" import var1, var2 as local2 %}
{{ var1.foo }} {{ local2.bar }}
{% endraw %}
```

最重要的是上面的`salt['grains.filter_by']`：

```text
salt.modules.grains.filter_by(lookup_dict, grain='os_family', merge=None, default='default', base=None)
```

在本例中，通过`grains='os'`为不同的系统匹配了不同的配置，并且使用`merge=salt['pillar.get']('mysql:server:lookup'))`为将敏感数据存储在pillar中提供了扩展(pillar的mysql:lookup字段目前应该废弃了，挺迷惑人的，一度以为`mysql:server:lookup`写错了)，这个merge应该也能够用于合并通用配置。

### 模块化并解决依赖问题
Salt提倡尽可能的模块化，本例也确实做到了这一点，服务端、客户端、Python连接模块、库表结构和用户权限等，都分别写在单独的文件里。`include`命令能够非常方便的对这些sls进行引用，下面看看本例的`init.sls`是如何进行引用并解决一些问题的。

```yaml
{% raw %}
{% from 'mysql/database.sls' import db_states with context %}
{% from 'mysql/user.sls' import user_states with context %}

{% macro requisites(type, states) %}
      {%- for state in states %}
        - {{ type }}: {{ state }}
      {%- endfor -%}
{% endmacro %}

include:
  - mysql.server
  - mysql.database
  - mysql.user

{% if (db_states|length() + user_states|length()) > 0 %}
extend:
  mysqld:
    service:
      - require_in:
        {{ requisites('mysql_database', db_states) }}
        {{ requisites('mysql_user', user_states) }}
  {% for state in user_states %}
  {{ state }}:
    mysql_user:
      - require:
        - sls: mysql.database
  {% endfor %}
{% endif %}
{% endraw %}
```

由于这个文件不长且亮点较多，就都粘贴在这了。`init.sls`是一个目录的默认入口，在本例里，三行`include`语句平平无奇，会给对应服务器安装mysql服务端，并导入表结构和用户(如果有的话)，这个文件的亮点在extend部分。
模块化的好处是显而易见的，但是模块化之后需要解决依赖问题。在本例中，database和user都依赖于service，而且user依赖于database，为了解决这些依赖，`init.sls`里进行了以下操作：

1. 导入database和user的state ID，当然database和user都提供了这个数据。
2. 编写了一个宏，用以避免重复和改善输入格式。(需要关注[空白控制][6])
3. 使用[extend][7]扩展引用的sls，添加各模块依赖关系。
4. 使用[require_in][8]，将'a依赖于X，b依赖于X，c依赖于...'变为'依赖于X的有abc..'，大幅精简这种多对一的依赖关系写法。
5. 使用[require sls][9]，将多对多的依赖关系精简为多对一个整体。

如果`require_in`也能支持sls就好了。这个文件主要向我们展示了如何在引用sls的时候传递授依赖的state ID并精简一对多及多对多依赖关系的写法。


### 变与不变
不同的服务器需要安装不同的服务；不同的系统安装服务的方式不同；不同的数据库需要不同的数据库结构和用户。在这个示例中，通过以下方法处理了变与不变的关系。

1. 通过pillar存储业务所需的变量。 

    pillar可以用于存储敏感数据，本例中也确实将用户密码、权限等信息存储在这里。pillar还可以用于将过程与数据分离，state不用关心具体的用户权限，只需要根据pillar数据动态生成，pillar也不需要关注各个系统的服务安装过程，只需要定义数据库表结构和用户权限。
2. 通过`filter_by`处理系统差异。

    在上面已经提到，同一个服务在不同的系统上需要不同的安装方式，通过这种过滤手段，state文件自动适用不同的系统。
3. 通过程序生成配置。

    在本例中，state文件中充满了大量的jinja语句，大部分的state定义其实是通过程序解析数据动态生成的，这一方面简化了sls文件，另一方面也使程序(state)本身与数据分离，提高了扩展性。另外值得注意的就是pillar数据的层级关系，定义好的变量能够很方便的在配置文件模板中使用；以及`user.sls`等文件中的state
    ID收集，对外提供了解决依赖问题所需的数据。

### 总结
这个示例的具体语法上面仍然有很多值得学习的，不在此一一列出。通过本例可以学习如何合理的组织一个复杂服务的方方面面，其核心思路如下：

1. 模块化服务的不同部分。
2. 各模块分别对外提供state ID，通过`extend`处理模块之间的依赖。
3. 通过`filter_by`动态适配不同的系统。
4. 敏感及更高层次的变量定义在pillar中，与state分离，更方便进行差异化配置。

以上是作为初学者对Salt配置的一些初步想法，可能在后续的使用中会有更多更具体的感受。官方提供了[很多范例][10]，有需求的时候要多多参考学习(拷过来改改^_^)，避免闭门造车。

[1]: http://docs.saltstack.com/en/latest/topics/best_practices.html#general-rules
[2]: https://github.com/saltstack-formulas/mysql-formula
[3]: /2015/01/21/salt-practice.html
[4]: https://github.com/saltstack-formulas/vim-formula/blob/master/vim/map.jinja
[5]: https://github.com/saltstack-formulas/git-formula/blob/master/git/map.jinja
[6]: http://jinja.pocoo.org/docs/dev/templates/#whitespace-control
[7]: http://docs.saltstack.com/en/latest/ref/states/extend.html
[8]: http://docs.saltstack.com/en/latest/ref/states/requisites.html#id1
[9]: http://docs.saltstack.com/en/latest/ref/states/requisites.html#require-an-entire-sls-file
[10]: https://github.com/saltstack-formulas
