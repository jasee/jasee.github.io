---
layout: post
title: PythonChallenge--关卡0-6
category: 编程
description: 一个非常好玩的借助Python解谜过关的游戏！
tags: ["Python", "Game", "PythonChallenge"]
---

[PythonChallenge][pythonchallenge]是一个解谜游戏，玩家通过解开当前页面的提示获得下一关的页面地址，整个过程需要使用Python辅助，涉及的内容很丰富，娱乐的同时还能练习Python(或者应该说练习Python的时候还能娱乐?)，非常赞。下面记录了我玩这个游戏的记录，没头绪的地方就看一眼别人的记录，有些跳转实在有点大。

每一关的页面地址都可以点击标题跳转。

### 0. [从零开始][0]

刚开始没看图片，直接试着把`0.html`改成`1.html`，页面提示'2**38 is much much larger.'，这才仔细看图片。直接：

```py
print 2**28
```

### 1. [ASCII映射][1]
结合提示可以知道我们需要将下面的字符串都向后偏移两位来获得答案，为了降低复杂度提示里字符串都是小写的。最开始采用比较原始的方法：

```py
s = "g fmnc wms bgblr rpylqjyrc gr zw fylb. rfyrq ufyr amknsrcpq ypc dmp. bmgle gr gl zw fylb gq glcddgagclr ylb rfyr'q ufw rfgq rcvr gq qm jmle. sqgle qrpgle.kyicrpylq() gq pcamkkclbcb. lmu ynnjw ml rfc spj."
r = ""

for i in s:
    if i.isalpha():
        if i == "y" or i == "z":
            r += chr(ord(i) + 2 - 26)
        else:
            r += chr(ord(i) + 2)
    else:
        r += i

print r
```
得到结果：

```
i hope you didnt translate it by hand. thats what computers are for. doing it in by hand is inefficient and that's why this text is so long. using string.maketrans() is recommended. now apply on the url.
```

现在将url中的`map`用同样的方法翻译到`ocr`就得到了下一关的地址。提示中推荐使用`string.maketrans()`方法，这个确实比上面的安全简便，使用这种方法再写一遍：

```py
import string

s = "g fmnc wms bgblr rpylqjyrc gr zw fylb. rfyrq ufyr amknsrcpq ypc dmp. bmgle gr gl zw fylb gq glcddgagclr ylb rfyr'q ufw rfgq rcvr gq qm jmle. sqgle qrpgle.kyicrpylq() gq pcamkkclbcb. lmu ynnjw ml rfc spj."
l = string.lowercase
t = string.maketrans(l, l[2:]+l[:2])
print s.translate(t)
```

### 2. [字符数统计][2]
第一眼看去还以为要搞OCR，提示上说明内容可能包含在页面里，也是，这种图片要识别基本没可能吧，打开Chrome审查发现一堆乱码一样的东西，让我们寻找出现次数最小的字符。我直接从网页上将这一千多行内容拷贝下来赋给s：

```py
t = {}
for i in s:
    if i in t:
        t[i] += 1
    else:
        t[i] = 0
print t
```

结果如下：

```
{'\n': 1220, '!': 6078, '#': 6114, '%': 6103, '$': 6045, '&': 6042, ')': 6185, '(': 6153, '+': 6065, '*': 6033, '@': 6156, '[': 6107, ']': 6151, '_': 6111, '^': 6029, 'a': 0, 'e': 0, 'i': 0, 'l': 0, 'q': 0, 'u': 0, 't': 0, 'y': 0, '{': 6045, '}': 6104}
```

可以发现有几个字母只出现了一次。找一下出现的顺序，得到网址`equality`

```py
k = ""
for i in s:
    if i.islower():
        k += i
print k
```

### 3. [正则匹配][3]
 
又把文本藏在页面里，看来是鼓励抓页面了。结合文本，提示的意思应该是：小写字母，两边不多不少都是三个大写字母。

```py
import urllib2
import re

src = urllib2.urlopen("http://www.pythonchallenge.com/pc/def/equality.html").read()
s = re.compile(r'<!--(.*)-->', re.S).findall(src)[0]
r = re.compile(r'[^A-Z][A-Z]{3}([a-z])[A-Z]{3}[^A-Z]').findall(s)
print ''.join(r)
```

### 4. [小爬虫][4]
页面显示了一个php网址，打开后发现图片可以点击，点击后显示下一个页面的地址，简单试了下，需要不断跳转，跳转的慢了还会用红字提醒你手酸了吧，意思该用Python啦。

```py
page_base = "http://www.pythonchallenge.com/pc/def/linkedlist.php?nothing=12345"
def new_page(id):
    return "http://www.pythonchallenge.com/pc/def/linkedlist.php?nothing=" + id
next = "12345"
while True:
    src = urllib2.urlopen(new_page(next)).read()
    print src
    r = re.compile(r'nothing is ([0-9]+)').findall(src)
    if r:
        next = r[0]
    else:
        break
```

中间还提示数字除以二再继续，改一下`next`值继续跑脚本，最后得到`peak.html`。另外我在输出中看到过一个页面(82682)显示如下内容，不知道是否有什么玄机：

```
There maybe misleading numbers in the text. 
One example is 82683. Look only for the next nothing and the next nothing is 63579
```

### 5. [拼图][5]
这个页面也没啥好看的，打开浏览器审阅可以看到隐藏了一个零像素的链接(banner.p)，打开发现一串奇怪的字符。这个实在没头绪，看攻略了，原来'peak hell sounds familiar?'说的是`pickle`啊，脑袋里从来没有这个词，跪了。pickle是Python特有的Python对象序列化协议，查查手册试着把这些字符反序列化得到一个列表，其内容如下：

```
[(' ', 95)]
[(' ', 14), ('#', 5), (' ', 70), ('#', 5), (' ', 1)]
[(' ', 15), ('#', 4), (' ', 71), ('#', 4), (' ', 1)]
[(' ', 15), ('#', 4), (' ', 71), ('#', 4), (' ', 1)]
```
看起来是坐标，每个列表一行，列表里每个元组表示一种字符及其数量。程序如下：

```py
banner = urllib2.urlopen("http://www.pythonchallenge.com/pc/def/banner.p")
r = pickle.load(banner)
for line in r:
    s = ""
    for i in line:
        s += i[0]*i[1]
    print s
```

输出是一个大大的`channel`。

### 6. [拉链][6]
作者说他觉得到这个点该要点捐赠了，不过这配图是什么情况？貌似页面里有个zip，看来拉链是这个意思啊。。。尝试用zipfile模块解压，报错：

```
AttributeError: addinfourl instance has no attribute 'seek'
```

找到一个[绕过的方法][seekerror]，脚本如下：

```py
import urllib2
import zipfile
import cStringIO

src = cStringIO.StringIO(urllib2.urlopen("http://www.pythonchallenge.com/pc/def/channel.zip").read())
z = zipfile.ZipFile(src)
print z.namelist()
print z.read('readme.txt')
```

压缩包里有一对数字命名的文件和一个'readme.txt'，查看'readme.txt'的内容：

```
welcome to my zipped list.

hint1: start from 90052
hint2: answer is inside the zip
```

看来又和第四关一样了，怪不得关卡页面标题是'now there are pairs'，类似的解法：

```py
next = "90052"
while True:
    content = z.read(next + '.txt')
    print content
    r = re.compile(r'nothing is ([0-9]+)').findall(content)
    if r:
        next = r[0]
    else:
        break
```

找到文件`46145.txt`，内容为'Collect the comments'。zipfile模块的`getinfo()`可以获取每个压缩对象的信息，其中包含了注释内容，上面的语句修改为：

```py
next = "90052"
comments = []
while True:
    content = z.read(next + '.txt')
    r = re.compile(r'nothing is ([0-9]+)').findall(content)
    if r:
        comments.append(z.getinfo(next + '.txt').comment)
        next = r[0]
    else:
        break

print ''.join(comments)
```

结果是个大大的'hockey'，访问这个页面，显示'it's in the air. look at the letters.'。我看了半天也不知道要看什么，看攻略原来是看氧气(oxygen)！有点坑爹啊。

后面的改天再玩，玩不过去就看攻略！



[0]: http://www.pythonchallenge.com/pc/def/0.html
[1]: http://www.pythonchallenge.com/pc/def/map.html
[2]: http://www.pythonchallenge.com/pc/def/ocr.html
[3]: http://www.pythonchallenge.com/pc/def/equality.html
[4]: http://www.pythonchallenge.com/pc/def/linkedlist.html
[5]: http://www.pythonchallenge.com/pc/def/peak.html
[6]: http://www.pythonchallenge.com/pc/def/channel.html

[pythonchallenge]: http://www.pythonchallenge.com/
[seekerror]: https://mail.python.org/pipermail/image-sig/2004-April/002729.html
