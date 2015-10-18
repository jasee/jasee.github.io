---
layout: post
title: 第一个Alfred Workflow
category: 工具
description: 编写自己的第一个Alfred Workflow，用于查询考勤记录，顺带还发现了ZKnet考勤系统的一个漏洞
tags: ["Alfred"]
---

有几个月没写博客了，最近没什么好记录的，看了一些数据库的书，蜻蜓点水没啥自己的领悟；试了前端的DataTables和Highcharts，也是勉强用着，还没有分享的价值。系统性的知识写起来不容易，需要有自己深刻的领悟和见解，对我而言博客里目前主要还是零碎的东西，零碎的东西脑袋记不住，需要写下来参考一下。

今年十一回家把婚礼办了，一切还挺顺利，守业更比创业难，希望以后能顺利安稳的一直过下去^\_^。

进入正题。最近一段时间经常需要看看自己的考勤打卡时间(难道开始等着点下班了么...)，每次需要登录到考勤记录的网站上查看，感觉有点麻烦。之前用Alfred Workflow用的很爽，比如有道词典什么的，我想是时候开始自己写Workflow了。我的目标是：打开Alfred，输入`kq`就可以显示我最近的几条打卡时间。

首先建立一个新Workflow，建议给自己画一个favicon，嘿嘿，大概是这样的:
![Workflow][1]

然后在空白的Workflow中加入一个`Script Filter`，写入以下内容：
![Workflow][2]

这里面我用了一个方便编写Alfred Workflow的Python库，安装方法如下：

```sh
# 不建议安装到系统环境，产生污染并且不利于分享
# https://github.com/deanishe/alfred-workflow
$ pip install --target=/path/to/my/workflow Alfred-Workflow
```

最后实现了预期的效果：
![Workflow][3]

爬考勤数据的脚本我就不贴了，反正返回的是一个考勤时间的字符串列表。不过在爬考勤网站的时候发现了两个有意思的事情(我们公司用的是`ZKNet 9.0`考勤系统)。
第一个，模拟登陆的时候，每次网站都返回一个'登陆失败'的消息回来，我在这个地方浪费了两个小时，反复查找是哪里出了纰漏，最后我突然想，感觉这网站这么简陋怎么会有很强的防爬能力呢，是不是烟雾弹?果然，测试其他页面发现已经登陆成功了，我只能说开发网站的人太坏了。
第二个，我把这个Workflow分享给同事的时候，他说查询到的是我的记录，我回顾了一下请求考勤数据的ajax链接，发现它通过userid这个参数来请求某个用户的数据(见图2，我这里把userid写成参数，并没有登陆后从页面上抓)，但并未和已登入用户的身份作对比，因此登陆后可以获取任何人的考勤记录，其他类似个人信息的数据也是如此，真是一个大漏洞。我顺手把公司所有人的工号、部门、姓名、userid的对应关系爬下来了，以后考虑把我们组的考勤数据都抓出来放到正在开发的运维网站上去，大家就再也不用去考勤网站看数据了。

万事开头难，有了这个第一次，后续重复性工作就可以写更多的Workflow来完成了。

[1]: /public/upload/2015/alfred1.png
[2]: /public/upload/2015/alfred2.png
[3]: /public/upload/2015/alfred3.png

### *参考文档*
*[Alfred-Workflow](http://www.deanishe.net/alfred-workflow/index.html)*
*[Alfred workflow 开发指南](http://myg0u.com/python/2015/05/23/tutorial-alfred-workflow.html)*
