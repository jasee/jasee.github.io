---
layout: post
title: Linux本地安装Jekyll
category: 工具
description: Jekyll是一个静态站点生成器，Github Pages就是用它将用户的文档转换成静态网站的。将博客托管到Github Pages上之后，更新内容时最好在本地预览，没有格式等问题后再提交，需要本地安装Jekyll。以前装过两次，但是都没有进行记录，今天在Centos6.3上又装了一遍，顺带记录一下，方便以后参考使用。
tags: ["Blog","Jekyll"]
---

Jekyll是一个静态站点生成器，Github Pages就是用它将用户的文档转换成静态网站的。将博客托管到Github Pages上之后，更新内容时最好在本地预览，没有格式等问题后再提交，需要本地安装Jekyll。以前装过两次，但是都没有进行记录，今天在Centos6.3上又装了一遍，顺带记录一下，方便以后参考使用。
### 安装Ruby
直接从[官网][1]下载最新版本的进行安装，防止低版本出现[奇怪的问题][2]。需要安装ruby自身提供的zlib库，否则后续添加淘宝预源[会有问题][3]。

```sh
$ wget http://cache.ruby-lang.org/pub/ruby/2.1/ruby-2.1.1.tar.gz
$ tar -zxf ruby-2.1.1.tar.gz && ruby-2.1.1
$ ./configure && make && make install
$ cd ext/zlib
$ ruby extconf.rb 
$ make && make install
```

### 安装RubyGems
同样，从[官网][4]下载最新版本进行安装，由于国内的网络原因(你懂的)，改用淘宝提供的RubyGems镜像。

```sh
$ wget http://production.cf.rubygems.org/rubygems/rubygems-2.2.2.tgz
$ tar -zxf rubygems-2.2.2.tgz && cd rubygems-2.2.2
$ ruby setup.rb
$ gem sources --remove https://rubygems.org/ && gem sources -a http://ruby.taobao.org/
```

### 安装Jekyll
直接使用`gem install jekyll`安装完事，相关的依赖和模块会自动安装的。
可以切换到博客目录下使用`jekyll server`命令看看效果。

#### *参考文档*
*[Jekyll Installation](http://jekyllrb.com/docs/installation/)*

[1]: https://www.ruby-lang.org/en/downloads/
[2]: http://stackoverflow.com/questions/10725767/error-installing-jekyll-native-extension-build
[3]: http://www.cnblogs.com/netbuddy/p/3501147.html
[4]: http://rubygems.org/pages/download
