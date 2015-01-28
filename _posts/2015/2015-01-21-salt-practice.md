---
layout: post
title: Salt小实践
category: 运维
description: 一个简单的Salt实践，刚好也能够把Grains/Pillar/State/Template/Schedule这些最基础的内容都包含进去，动手实践一下还是能增加理解的。
tags: ["Salt"]
---

前几天去厦门玩了一圈，回来发现一月都快结束了，Q1看来又要继续忙下去了。
最近看了看SaltStack，今天准备简单的实践一下。其实更早的时候就应该搞一个配置管理工具了，只不过有个自己的小工具能做批量管理，也就一拖再拖，现在看来，如果自动化运维想更进一步，这个东西就不能没有了。
今天想要使用Salt，完成一个简单的负载均衡，目标如下：

### 目标

1. 三台服务器，1台做LB，另外两台做NODE，都用Nginx。当然完成配置后，加减或者调整机器角色应该是很容易的。
2. 要有一些动态的东西，比如这次假装每天流量从0点开始随时间正比上升，nginx worker数量跟着变化。
3. 全部操作应该都通过Salt进行，并且不要用`cmd.run`这种模块。

这个目标还是很简单的，刚好也能够把Grains/Pillar/State/Template/Schedule这些最基础的内容都包含进去，动手实践一下还是能增加理解的。主要参考了《Python自动化运维》的第十章。

先贴一下最后的目录结构吧：

```
/srv/
├── pillar
│   ├── lb_server.sls
│   ├── node_server.sls
│   ├── schedule.sls
│   └── top.sls
└── salt
    ├── data.sls
    ├── _grains
    │   └── nginx_grains.py
    ├── nginx
    │   ├── index.html
    │   ├── lb.conf
    │   └── node.conf
    ├── nginx.sls
    └── top.sls
```

### 服务器分组
配置`/etc/salt/master`，将服务器分为LB和NODE两组，前者使用Nginx进行负载均衡，后者使用Nginx提供WEB服务。修改后最好重启一次salt-master，在我的测试环境里，不重启时Targeting中生效，但Pillar中不可用。

```
nodegroups:
  lb_group: 'test02.opjasee.com'
  node_group: 'test03.opjasee.com,test04.opjasee.com'
```

### 自定义Grains
编写grains_modules(`/srv/salt/_grains/nginx_grains.py`)，为动态更新Nginx配置提供参数。

```py
import time, commands

def NginxGrains():
    grains = {}
    nginx_worker = 5
    max_open_file = 65535
    try:
        nginx_worker = int(time.strftime('%H', time.localtime(time.time())))/2 + 2
    except:
        pass
    try:
        getulimit = commands.getstatusoutput('source /etc/profile; ulimit -n')
        if getulimit[0] == 0:
            max_open_file = int(getulimit[1])
    except:
        pass
    grains['nginx_worker'] = nginx_worker
    grains['max_open_file'] = max_open_file
    return grains
```

同步并加载该模块，可以使用`grains.item`命令测试效果。

```sh
$ salt '*' saltutil.sync_all
$ salt '*' sys.reload_modules
```

### 自定义Pillar
1. `/srv/pillar/top.sls`，定义lb_group和node_group节点的配置文件。在`lb_server.sls`中，只需要定义nginx源配置文件位置，在`node_server.sls`中，还额外定义了nginx数据目录及`index.html`模板的位置。

    ```
    base:
      lb_group:
        - match: nodegroup
        - lb_server
      node_group:
        - match: nodegroup
        - node_server
    ```
2. `/srv/pillar/lb_server.sls`

    ```
    nginx:
      conf: salt://nginx/lb.conf
    ```
3. `/srv/pillar/node_server.sls`

    ```
    nginx:
      root: /work/nginx/node
      conf: salt://nginx/node.conf
      data: salt://nginx/index.html
    ```

配置完成后使用`salt '*' saltutil.refresh_pillar`命令刷新。

### 配置State

1. `/srv/salt/top.sls`，lb和node都需要安装nginx服务，node另外需要同步`index.html`模板等文件。

    ```
    base:
      '*':
        - nginx
      node_group:
        - match: nodegroup
        - data
    ```
2. `/srv/salt/nginx.sls`，lb和node通过pillar中的定义获取不同的配置文件。

    ```
    {% raw %}
    nginx:
      pkg:
        - installed
      file.managed:
        - source: {{ pillar['nginx']['conf'] }}
        - name: /etc/nginx/nginx.conf
        - user: root
        - group: root
        - mode: 644
        - template: jinja
      service.running:
        - enable: True
        - reload: True
        - watch:
          - file: /etc/nginx/nginx.conf
          - pkg: nginx
    {% endraw %}
    ```
3. `/srv/salt/data.sls`，`index.html`模板展示一些节点信息，将`nginx.conf`放过来是为了在修改文件句柄等参数时方便观察。

    ```
    {% raw %}
    index.html:
      file.managed:
        - source: {{ pillar['nginx']['data'] }}
        - name: {{ pillar['nginx']['root'] }}/index.html
        - user: nginx
        - group: nginx
        - mode: 644
        - template: jinja
        - makedirs: True
    nginx.conf:
      file.managed:
        - source: {{ pillar['nginx']['conf'] }}
        - name: {{ pillar['nginx']['root'] }}/nginx.conf.txt
        - user: nginx
        - group: nginx
        - mode: 644
        - template: jinja
        - makedirs: True
    {% endraw %}
    ```

### 模板文件

1. `/srv/salt/nginx/index.html`

    ```html
    {% raw %}
    <html>
    <h2>HOSTNAME: {{ grains['fqdn'] }}</h2>
    <h2>OS: {{ grains['os'] }} {{ grains['osrelease'] }}</h2>
    <h2>WORKER: {{ grains['nginx_worker'] }}</h2>
    <h2>MAX_OPEN_FILE: {{ grains['max_open_file'] }}</h2>
    <h2>ROOT: {{ pillar['nginx']['root']}}</h2>
    <h2>WEIGHT: {{ grains['num_cpus'] }}</h2>
    <h2><a href="nginx.conf.txt">nginx.conf</a></h2>
    </html>
    {% endraw %}
    ```
2. `/srv/salt/nginx/lb.conf`，需要注意使用`pillar['master']['nodegroups']['node_group']`获取的并不是直接可用的列表。另外权重简单的使用CPU个数确定。

    ```
    {% raw %}
    user    nginx;
    worker_processes  {{ grains['nginx_worker'] }};  
      
    error_log  /var/log/nginx/error.log;  
      
    pid        /var/run/nginx.pid;  
      
    events {  
        worker_connections  {{ grains['max_open_file'] }};  
    }  
      
    http {  
        include       /etc/nginx/mime.types;  
        default_type  application/octet-stream;  
        log_format log_main '$remote_addr - $remote_user [$time_local] "$request" '
                                '$status $body_bytes_sent "$http_referer" '
                                '"$http_user_agent" $http_x_forwarded_for $request_time '; 
        access_log  /var/log/nginx//access.log log_main;  
      
        sendfile        on;  
        tcp_nopush      on;  
        tcp_nodelay     on;  
      
        keepalive_timeout  60;  
     
        upstream test {  
        {% for node in pillar['master']['nodegroups']['node_group'].split(',') %}
          server {{ node }}:80 weight={{ grains['num_cpus'] }} max_fails=3 fail_timeout=30s;
        {% endfor %}
         }  
      
        server {  
                listen       80;  
                server_name  test.opjasee.com;     
      
                location / {  
                  proxy_connect_timeout   3;  
                  proxy_send_timeout      30;  
                  proxy_read_timeout      30;  
                  proxy_pass http://test;  
                }  
       }  
    }
    {% endraw %}
    ```
3. `/srv/salt/nginx/node.conf`

    ```
    {% raw %}
    # {{ grains['fqdn'] }}
    user    nginx;
    worker_processes  {{ grains['nginx_worker'] }};  
      
    error_log  /var/log/nginx/error.log;  
      
    pid        /var/run/nginx.pid;  
      
    events {  
        worker_connections  {{ grains['max_open_file'] }};  
    }  
      
    http {  
        include       /etc/nginx/mime.types;  
        default_type  application/octet-stream;  
        log_format log_main '$remote_addr - $remote_user [$time_local] "$request" '
                                '$status $body_bytes_sent "$http_referer" '
                                '"$http_user_agent" $http_x_forwarded_for $request_time '; 
        access_log  /var/log/nginx/access.log log_main;  
      
        sendfile        on;  
        tcp_nopush      on;  
        tcp_nodelay     on;  
      
        keepalive_timeout  60;  
     
        server {  
                listen       80;  
                server_name  _;     
      
                location / {  
                  root {{ pillar['nginx']['root'] }};
                  index index.html;
                }  
       }  
    }
    {% endraw %}
    ```

此时执行`salt '*' state.highstate`即可完成节点上的服务部署。

### 定时更新
将调度配置在Master Pillar上比较灵活，具体方法为:

1. `/srv/pillar/top.sls`增加两行:

    ```
    base:
      '*':
        - schedule
    ```
2. `/srv/pillar/schedule.sls`中写入以下内容:

    ```
    schedule:
      highstate:
        function: state.highstate
        minutes: 5
    ```
3. 刷新pillar配置并观察:

    ```sh
    $ salt '*' saltutil.refresh_pillar
    ```
