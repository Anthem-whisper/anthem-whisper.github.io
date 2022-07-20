---
title: Debian下PHP Xdebug调试环境搭建
date: 2021-02-19
image: 
description: PhpStorm配合Xdebug的调试环境搭建
categories: 
- note
tags:
- 代码审计
---
# Debian下PHP Xdebug调试环境搭建

> 本文永久链接：http://wh1sper.com/debian%e4%b8%8bphp-xdebug%e8%b0%83%e8%af%95%e7%8e%af%e5%a2%83%e6%90%ad%e5%bb%ba/

### 前言

Xdebug是PHP的扩展，用于协助调试和开发。

- 它包含一个用于IDE的调试器
- 它升级了PHP的`var_dump()`函数
- 它为通知，警告，错误和异常添加了堆栈跟踪
- 它具有记录每个函数调用和磁盘变量赋值的功能
- 它包含一个分析器
- 它提供了与PHPUnit一起使用的代码覆盖功能。

本地跟代码必备的工具。
但不推荐在生产环境中使用xdebug，因为他太重了。

### 预置环境

* OS: Kali GNU/Linux Rolling x86_64

* Kernel: 5.7.0-kali1-amd64

* PHP version: 7.4.14

* Server API: FPM/FastCGI
* nginx version: nginx/1.18.0
* PhpStorm version: 2020.3



### 下载安装Xdebug

访问[https://xdebug.org/wizard](https://xdebug.org/wizard)

在web服务器中放入phpinfo，（注意最好是使用web服务，如果是本地php脚本的话`php.ini`可能会不一样）

访问之后将整个页面源码`Ctrl+A  Ctrl+C`复制到框里面：
![image-20210219110753403](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210219110804.png)

分析之后会自动给你最合适的版本，并且告诉你后续安装步骤。

下载给出的`.tar.tgz`文件

1.安装PHP拓展

```
apt-get install php-dev autoconf automake
```

2.解压

```
tar -xvzf xdebug-3.0.2.tgz
cd xdebug-3.0.2
```

3.执行以下命令，如果报错`Command not found`，那就先执行`1.`

```
phpize
```

完成之后会输出类似于以下内容，因环境而异，具体还请参考[https://xdebug.org/wizard](https://xdebug.org/wizard)

```
Configuring for:
...
Zend Module Api No:      20190902
Zend Extension Api No:   320190902
```

随后分别需要执行`./configure`和`make`以及`cp`等命令，因环境而异，具体参考[https://xdebug.org/wizard](https://xdebug.org/wizard)，这里不再累述。



### 配置Xdebug

打开`php.ini`

在末尾写入（最好去掉注释）：

```
[xdebug]
#pay attention,xdebug3 is different from 2
zend_extension = /usr/lib/php/20190902/xdebug.so#这里位置因环境而异
xdebug.mode=debug
xdebug.remote_handler = dbgp
xdebug.client_host = 127.0.0.1 #安装有PhpStorm的机器
xdebug.client_port = 9001 #端口可修改，防止冲突
xdebug.remote_mode=req #可以设为req或jit，req表示脚本一开始运行就连接远程客户端，jit表示脚本出错时才连接远程客户端。
xdebug.idekey = PHPSTROM
```

**注意：**如果你想 Xdebug 和 OPCache 一起使用，必须在 OPCache 之后加载 Xdebug `zend_extension` 代码行。

因为我下载的版本是Xdebug3，相对于Xdebug2，`remote_enable`被`mode`取代，`remote_host`被`client_host`取代，`remote_port`改为`client_port`，默认为`9003`端口，`client_host`默认为`localhost`。

如果你下载的是Xdebug2：

```
[xdebug]
zend_extension= /usr/lib/php/20190902/xdebug.so
xdebug.remote_enable = on
xdebug.remote_port = 9001
xdebug.remote_host = 127.0.0.1
```

写入完毕后，重启`php-fpm`或者重启`apache`。

之后再次访问`phpinfo`，按`Ctrl+F`搜索`Xdebug`，成功的话你会看到和下面差不多的情况

![image-20210219114506481](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210219114515.png)



### 配置PhpStorm

打开PhpStorm；

在`File->Languages & Frameworks->PHP`下，选择`CLI interpreter`为你刚才配置的PHP版本

![image-20210219114827373](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210219114829.png)

成功的话会出现Xdebug的字样

![image-20210219114935735](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210219114938.png)

没有的话重新打开`php.ini`配置

选择Debug分支，保证`port`和`php.ini`中的一致

![image-20210219115215244](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210219115308.png)

配置DBGp Proxy，一样。

![image-20210219115420206](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210219115423.png)

配置Server

![image-20210219115540815](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210219115553.png)

点击右上角，编辑配置

![image-20210219115806714](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210219115808.png)

在Server下拉框中，选择我们在第4步设置的Server服务名称（之前配置的Server），Browser选择你要使用的浏览器

ALL Right;

### 配置FireFox

这一步可以不需要，不过配置之后方便一点；

FireFox访问https://addons.mozilla.org/en-US/firefox/addon/xdebug-helper-for-firefox/，安装插件。

安装完成之后，开启插件，给程序下个断点，访问，如果成功PhpStorm会自动跳出来：

![image-20210219120207661](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210219120209.png)

好了，到这里我们就配置完成了。

随后携带`Cookie: XDEBUG_SESSION="ECLIPSE"`头访问（或者开FireFox插件，一个道理）就可以进入debug了

另外在`nginx.conf`里面设置`fastcgi_read_timeout 600`可以调整fastcgi进程向 nginx 进程发送输出过程的超时时间，默认值60秒，让调试得到时间