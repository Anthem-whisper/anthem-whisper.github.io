---
title: HackTheBox_challenges
date: 2020-07-25
image: https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120181059.jpeg
description: 
categories: 
- ctf_writeup
tags:
- HackTheBox
- 正则
---
### Emdee five for life

考点：python正则，request模块

打开页面，给你一个字符串，md5加密后提交，太慢或者错了就会Too slow：

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120181105.png)

```
<html>
<head>
<title>emdee five for life</title>
</head>
<body style="background-color:powderblue;">
<h1 align='center'>MD5 encrypt this string</h1><h3 align='center'>LWod6afkxEjJV8KZeLRO</h3><center><form action="" method="post">
<input type="text" name="hash" placeholder="MD5" align='center'></input>
</br>
<input type="submit" value="Submit"></input>
</form></center>
</body>
</html>
```

 

当然是脚本：

```
#!/usr/bin/env python3
import requests
import re
import hashlib
 
while 1:
    host = "http://docker.hackthebox.eu:32046/"
    req = requests.session()
    r = req.get(url=host)
    #print(r.text)
    pattern = re.compile(r"<h3 align='center'>(\S+)</h3>")
    h3 = pattern.findall(r.text)
    md5 = hashlib.md5(h3[0].encode('utf-8')).hexdigest()
    #print(h3[0],md5)
    data={'hash':md5}
    s = req.post(url=host, data=data)
    
    if 'Too slow!' in s.text:
        print('Too slow!')
 
    else:
        print(s.text)
        break
```

第11行那里用了 `(\S+)` 就会返回LWod6afkxEjJV8KZeLRO

第二次POST过去的时候需要用session不然就会too slow

------

### FreeLancer

考点：sqli

打开页面，有一个表单

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120181111.png)

 

本来以为是XSS，但是无论怎么样提交都是500，后来看源码，发现了 `<!-- <a href="portfolio.php?id=3">Portfolio 3</a> -->`

访问发现sql注入，Union一把梭可以发现：

```
表名：portfolio,safeadmin 
safeadmin列名：id,username,password,created_at
```

注出来用户 `safeamd` 密码 `$2y$10$s2ZCi/tHICnA97uf4MfbZuhmOZQXdCnrM9VM9LBMHPp68vAXNRf4K`

不过没什么用，因为我们目的是要getshell，其实也不用getshell，只要你用dirbuster爆破出这个文件 `/var/www/html/administrat/panel.php` 这个文件，直接读取他就好了