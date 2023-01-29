---
title: "Docker下PHP调试环境搭建"
date: 2023-01-29T15:52:26+08:00
draft: false
image: 
description: 
comments: true
license: true
math: false
categories:
- note
tags:
- xdebug
- docker
- 代码审计
---



承接上一篇[Debian下PHP 调试环境搭建](https://blog.wh1sper.com/posts/debian%E4%B8%8Bphp%E8%B0%83%E8%AF%95%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/)

但是在虚拟机里安装PhpStorm或者在本机上调试都是不太优雅的方案，使用Docker运行调试不用担心php版本的问题，也不用去担心如何倒腾破环了宿主机的环境。

先补充下Xdebug的原理：

## Xdebug的原理

1. 浏览器向 Httpd 服务器发送一个带有 XDEBUG_SESSION_START 参数的请求，服务器收到这个请求之后交给后端的 PHP（已开启 xdebug 模块）进行处理

2. 如果是默认配置`xdebug.remote_connect_back = 0`，那么xdebug在收到调试通知时会读取配置 `xdebug.remote_host` 和 `xdebug.remote_port`（默认是 `localhost:9000`），然后向这个端口发送通知。

   ![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/main/img/202301291609691.gif)

   如果开启`xdebug.remote_connect_back = 1`，xdebug会将请求来源的 IP 绑定，并通知
   ![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/main/img/202301291620726.gif)

我们可以看到，Xdebug其实是需要从服务端（运行环境）发送消息到客户端（调试IDE）的，但是往往客户端是在内网，没有办法收到服务端的消息。

所以一般会用代理隧道技术解决这个问题，比如ssh。

注意，使用[DBGp 代理](https://xdebug.org/docs/dbgpProxy)只能让你debug的时候可以多人运动，但是**并不能解决内网不可达问题**

使用ssh隧道：

1. 修改 php.ini。让`xdebug.client_host`值为`127.0.0.1`，这样当服务端收到一个带有参数的请求的时候，xdebug会向127.0.0.1:9003发送调试信息

2. 客户端进行 ssh隧道端口转发

```
ssh -N -R 远程IP:远程端口:127.0.0.1:9003 root@远程IP
```

这样就能使用调试远程程序了。



## 使用[DBGp 代理](https://xdebug.org/docs/dbgpProxy)进行多用户调试﻿

可以参考：[通过 Xdebug 代理进行多用户调试](https://phpstorm.org/multiuser-debugging-via-xdebug-proxies.html)﻿

服务端使用DBGp监听`127.0.0.1:9003`，将xdebug发送的请求代理到客户端的9000端口

DBGp 代理可执行文件接受两个主要参数：

- `-i`：用于侦听 IDE（客户端）连接的主机和端口
- `-s`: 侦听调试器引擎（服务器）连接的主机和端口

```bash
./dbgpProxy -s 127.0.0.1:9003 -i 0.0.0.0:9001
```

用于监听客户端连接的地址最好填 `0.0.0.0` ，避免网卡选错的麻烦。

## 宿主机无法收到Xdebug请求

问题：

在Docker中监听是能够收到Xdebug的请求的，但是在宿主机上用netcat或者PhpStorm监听收不到请求

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/main/img/202301291551797.png)

还是网络不可达的问题，可以通过隧道技术或者设置xdebug配置参数client_host/remote_host为宿主机IP地址。

解决方法：

使用`host.docker.internal`，可以直接解析到宿主机IP，参考[官方文档](https://docs.docker.com/desktop/networking/#use-cases-and-workarounds-for-all-platforms)。

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/main/img/202301291634184.png)

但是要注意xdebug2和3的配置参数是不一样的，启动的时候环境变量需要注意一下，可见[官方文档](https://xdebug.org/docs/upgrade_guide#Step-Debugging)

## 使用Dockerfile和Docker compose

按照PhpStorm官方的[repo](https://github.com/JetBrains/phpstorm-docker-images/blob/master/php-82-apache-xdebug-32/Dockerfile)魔改了一个祖国版Dockerfile，我自己电脑上build大概五分钟：

```dockerfile
FROM php:8.2-apache
# COPY --from=composer /usr/bin/composer /usr/bin/composer
RUN    php -r "readfile('https://getcomposer.org/installer');" | php \
    && mv /var/www/html/composer.phar /usr/bin/composer
RUN    sed -i "s@http://deb.debian.org@http://mirrors.aliyun.com@g" /etc/apt/sources.list \
    && apt clean \
    && apt-get update \
    && apt-get -y install libzip-dev libicu-dev \
    && docker-php-ext-install mysqli zip intl \
    && pecl install xdebug-3.2.0 \
    && docker-php-ext-enable xdebug \
    && echo "xdebug.mode=debug" >> /usr/local/etc/php/php.ini

# dbgpProxy, for multiuser debugging
# ssh, for tunnel
# RUN   curl https://xdebug.org/files/binaries/dbgpProxy -o /usr/bin/dbgpProxy \
#     && chmod +x /usr/bin/dbgpProxy \
#     && apt-get install ssh

```

docker-compose.yml:

```yaml
version: '3.2'
services:
  webserver:
    build: .
    image: php-82-apache-xdebug-32
    container_name: php-82-apache-xdebug-32
    ports:
      - "50080:80"
      # - "9001:9001" # for dbgpProxy
    volumes:
      - ./html:/var/www/html
    environment:
      # Xdebug 2
      # XDEBUG_CONFIG: "remote_handler=dbgp remote_host=host.docker.internal idekey=PHPSTORM" 
      # Xdebug 3
      XDEBUG_CONFIG: "remote_handler=dbgp client_host=host.docker.internal" 
      XDEBUG_SESSION: "PHPSTORM"
```

docker和IDE都在一台电脑上的情况下，让xdebug直接把debug信息打到宿主机IP也就是`host.docker.internal`就可以了

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/main/img/202301292235122.png)

## Reference

[PHPSTORM调试Docker项目](https://www.freebuf.com/articles/web/266512.html)

[PhpStorm 利用官方docker镜像启动虚拟环境及调试](https://www.chawu.top/blog/2022-04/phpstorm-docker-xdebug.html)

[成为高级 PHP 程序员的第一步——调试（xdebug 原理篇）](https://learnku.com/articles/4090/the-first-step-to-becoming-a-senior-php-programmer-debugging-xdebug-principle)
