---
layout: post
title: Gitlab升级记录
category: 运维
description: 记录了从Gitlab 6.1.0升级到8.2.1的过程
tags: ["Gitlab"]
---

## 背景
目前我们使用的Gitlab版本为6.1.0，最新的版本为8.2.1。Gitlab发展日新月异，为了以下目标我们准备对Gitlab做一次升级:

    1. 更好的维护性
    2. 更高的性能
    3. 更高的安全性和更少的Bug
    4. 更多的功能，特别是Gitlab-CI
    5. 更易用的界面和交互逻辑

经过调研Gitlab提供的升级步骤，我们将本次升级分为三个阶段

    1. 从6.1.0升级到7.14.3
    2. 从7.14.3源码安装升级到7.14.3 Omnibus安装，同时迁移到安装了CentOS7的新服务器。
    3. 从7.14.3升级到最新版8.2.1


## 0、前置步骤

1. 安装CentOS7

    1. 为新服务器安装CentOS7(步骤略)。
    2. 为了避免更换服务器导致大家的`known_hosts`失效，需要将原gitlab服务器的`/etc/ssh/ssh_host_rsa_key*`两个文件复制到新服务器上。
    3. 为了避免git pull或https的认证失败，临时在/etc/hosts中增加本机ip到gitlab.opjasee.com的映射。
2. 安装Omnibus gitlab 7.14.3

    ```sh
    # 后面需要用到git命令下载mysql-postgresql转换工具
    $ sudo yum install -y curl openssh-server git
    $ sudo systemctl enable sshd
    $ sudo systemctl restart sshd
    $ sudo systemctl stop firewalld
    $ sudo systemctl disable firewalld
    # 被墙已挂代理下载
    $ wget https://packages.gitlab.com/gitlab/gitlab-ce/packages/el/7/gitlab-ce-7.14.3-ce.0.el7.x86_64.rpm/download
    $ sudo rpm -ivh gitlab-ce-7.14.3-ce.0.el7.x86_64.rpm
    # 修改/etc/gitlab/gitlab.rb
    $ sudo gitlab-ctl reconfigure
    ```

## 1、[老服务器]从6.1.0升级到7.14.3

1. 关闭gitlab

    ```sh
    $ sudo service gitlab stop
    ```
2. 执行备份(本步骤约由执行系统快照替代)

    ```sh
    $ cd /home/git/gitlab
    $ sudo -u git -H bundle exec rake gitlab:backup:create RAILS_ENV=production
    ```
3. 获取最新代码

    ```sh
    $ cd /home/git/gitlab
    $ sudo -u git -H git fetch --all
    $ sudo -u git -H git checkout -- db/schema.rb
    $ sudo -u git -H git checkout 7-14-stable
    ```
4. 安装相关依赖
    其实这个版本只是一个过渡版本，并不一定需要所有，不过也不需要仔细调研，先装上。下面更新redis也类似。

    ```sh
    # 注意epel的Fedora的https问题
    $ yum install krb5-devel krb5-libs cmake nodejs
    ```
5. 更新redis，通过socket访问

    ```sh
    $ sudo cp /etc/redis.conf /etc/redis.conf.orig
    $ sed 's/^port .*/port 0/' /etc/redis.conf.orig | sudo tee /etc/redis.conf
    $ echo 'unixsocket /var/run/redis/redis.sock' | sudo tee -a /etc/redis.conf
    $ sudo sed -i '/# unixsocketperm/ s/^# unixsocketperm.*/unixsocketperm 0775/' /etc/redis.conf
    # 重启redis
    $ sudo -u redis -H kill $(cat /var/run/redis/redis.pid) && sudo -u redis -H /usr/sbin/redis-server /etc/redis.conf
    $ sudo usermod -aG redis git
    $ sudo -u git -H cp config/resque.yml.example config/resque.yml
    $ sudo -u git -H sed -i 's|^  # socket.*|  socket: /var/run/redis/redis.sock|' /home/git/gitlab-shell/config.yml
    ```
6. 更新gitlab-shell

    ```sh
    $ cd /home/git/gitlab-shell
    $ sudo -u git -H git fetch
    $ sudo -u git -H git checkout v2.6.5
    ```
7. 更新库文件，执行迁移

    ```sh
    $ cd /home/git/gitlab
    $ vim Gemfile # 更换源为https://ruby.taobao.org
    $ sudo -u git -H bundle install --without development test postgres --deployment
    $ sudo -u git -H bundle exec rake db:migrate RAILS_ENV=production
    $ sudo -u git -H bundle exec rake migrate_iids RAILS_ENV=production
    $ sudo -u git -H bundle exec rake assets:clean assets:precompile cache:clear RAILS_ENV=production
    $ sudo chmod u+rwx,g+rx,o-rwx /home/git/gitlab-satellites
    $ sudo cp lib/support/init.d/gitlab /etc/init.d/gitlab
    ```
8. 更新配置文件开启服务

    ```sh
    # 更新以下三个配置文件(参考example改成自己的配置)
    # /home/git/gitlab/config/gitlab.yml (https://gitlab.com/gitlab-org/gitlab-ce/raw/7-14-stable/config/gitlab.yml.example)
    # /home/git/gitlab/config/unicorn.rb (https://gitlab.com/gitlab-org/gitlab-ce/raw/7-14-stable/config/unicorn.rb.example)
    # /home/git/gitlab-shell/config.yml (https://gitlab.com/gitlab-org/gitlab-shell/raw/v2.6.5/config.yml.example)
    $ sudo -u git -H cp config/initializers/rack_attack.rb.example config/initializers/rack_attack.rb
    $ sudo cp lib/support/logrotate/gitlab /etc/logrotate.d/gitlab
    $ sudo service gitlab start
    ```
9. 检查服务状态

    ```sh
    $ cd /home/git/gitlab
    $ sudo -u git -H bundle exec rake gitlab:env:info RAILS_ENV=production
    $ sudo -u git -H bundle exec rake gitlab:check RAILS_ENV=production
    ```

## 2、[老/新服务器]备份、迁移和恢复

1. 关闭gitlab

    ```sh
    $ sudo service gitlab stop
    ```
2. 执行备份

    ```sh
    $ cd /home/git/gitlab
    $ sudo -u git -H bundle exec rake gitlab:backup:create RAILS_ENV=production
    $ cd tmp/backups
    $ sudo -u git -H mysqldump --compatible=postgresql --default-character-set=utf8 -r gitlabhq_production.mysql -u root gitlabhq_production
    ```
3. 将备份文件传输到新服务器

    ```sh
    $ scp /home/git/gitlab/tmp/backups/1448859844_gitlab_backup.tar root@new-gitlab:/tmp
    $ scp /home/git/gitlab/tmp/backups/gitlabhq_production.mysql root@new-gitlab:/tmp
    ```
4. 在新服务器上将备份中的MySQL转换为Postgresql(约15分钟)

    ```sh
    $ cd /var/opt/gitlab/backups
    $ sudo -u git -H mkdir postgresql
    $ sudo chown git /tmp/1448859844_gitlab_backup.tar && sudo -u git -H mv /tmp/1448859844_gitlab_backup.tar postgresql
    $ sudo chown git /tmp/gitlabhq_production.mysql && sudo -u git -H mv /tmp/gitlabhq_production.mysql postgresql
    $ cd postgresql
    $ sudo -u git -H git clone https://github.com/gitlabhq/mysql-postgresql-converter.git -b gitlab
    $ sudo -u git -H mkdir db
    $ sudo -u git -H python mysql-postgresql-converter/db_converter.py gitlabhq_production.mysql db/database.sql
    $ sudo -u git -H ed -s db/database.sql < mysql-postgresql-converter/move_drop_indexes.ed
    $ sudo -u git -H gzip db/database.sql
    $ sudo -u git -H tar rf 1448859844_gitlab_backup.tar db/database.sql.gz
    $ sudo -u git -H mv 1448859844_gitlab_backup.tar ../
    # 升级成功后记得删除postgresql目录
    ```
5. 恢复备份

    ```sh
    $ sudo gitlab-ctl stop unicorn
    $ sudo gitlab-ctl stop sidekiq
    # 加上这个变量，否则执行到gitlab:shell:setup重新生成authorized_keys时会报错
    $ LC_ALL="en_US.UTF-8" sudo gitlab-rake gitlab:backup:restore BACKUP=1448859844
    $ sudo chmod -R ug+rwX,o-rwx /var/opt/gitlab/git-data/repositories
    $ sudo chmod -R ug-s /var/opt/gitlab/git-data/repositories
    $ find /var/opt/gitlab/git-data/repositories -type d -print0 | sudo xargs -0 chmod g+s
    $ sudo gitlab-rake gitlab:satellites:create RAILS_ENV=production
    $ sudo gitlab-ctl start
    $ sudo gitlab-rake gitlab:check
    ```

## 3、[新服务器]从7.14.3升级到8.2.1

迁移到Omnibus7.14.3之后再升级就很方便了，从7.10开始Omnibus引入了`gitlab-ctl upgrade`命令，只需要执行下列命令:

```sh
# 被墙已挂代理下载
$ wget https://packages.gitlab.com/gitlab/gitlab-ce/packages/el/7/gitlab-ce-8.2.1-ce.0.el7.x86_64.rpm/download
$ rpm -Uvh gitlab-ce-8.2.1-ce.0.el7.x86_64.rpm
$ sudo gitlab-rake gitlab:check
```

当执行rpm进行升级时Gitlab会自动执行以下命令:

    1. 关闭gitlab服务。
    2. 使用当前的旧版本Gitlab创建备份(轻量级备份，仅备份数据库)
    3. 运行`gitlab-ctl reconfigure`，进行必要的数据库更新迁移。
    4. 重新启动Gitlab服务。


## 4、新老服务器切换

1. 关闭老的Gitlab服务器。
2. 将老服务器的IP配置到新服务器上。
3. 删除新服务器临时添加的hosts映射。
4. 重启新服务器并校验。

## 5、升级总结

升级按照预期完成。
后来发现一个问题，升级后新建项目报错，后来清理了缓存问题消失，感觉自动升级的步骤里应该加上这一步。

```sh
$ sudo gitlab-rake cache:clear
```
