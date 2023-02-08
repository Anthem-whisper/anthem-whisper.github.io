---
title: "Zentao"
date: 2023-01-31T11:35:17+08:00
draft: true
image: 
description: 
comments:  true
license: true
math: false
categories:
- note
tags:
- 代码审计
---





## 环境搭建

使用docker-compose.yml启动环境：

```yaml
version: '3.2'
services:
  db:
    image: mysql:5.6
    restart: always
    container_name: db
    ports:
      - "3306:3306"
    command:
      - --default_authentication_plugin=mysql_native_password
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
    environment:
      MYSQL_DATABASE: zentao
      MYSQL_USER: zentao
      MYSQL_PASSWORD: zentao
      MYSQL_ROOT_PASSWORD: 'root'
    networks:
      extnetwork2:
        ipv4_address: 172.192.1.100
  webserver:
    depends_on:
      - db
    image: wh1sperdiyu/php-72-apache-xdebug-30
    container_name: php-72-apache-xdebug-30
    ports:
      - "80:80"
    volumes:
      - ./html:/var/www/html
    environment:
      XDEBUG_CONFIG: "remote_handler=dbgp client_host=host.docker.internal" 
      XDEBUG_SESSION: "PHPSTORM"
    extra_hosts:
      - "mysql:172.192.1.100"
    networks:
      extnetwork2:
        ipv4_address: 172.192.1.10
networks:
  extnetwork2:
    ipam:
      config:
        - subnet: 172.192.1.0/24
          gateway:  172.192.1.1
```

安装的时候发现需要pdo_mysql扩展，进容器安装下：

```bash
docker-php-ext-install pdo_mysql
echo "extension=/usr/local/lib/php/extensions/no-debug-non-zts-20170718/pdo_mysql.so" >> /usr/local/etc/php/php.ini
```

为了让PHP重新加载ini文件，需要重启apache进程，但它是容器的守护进程，那就直接重启下容器就行：

```bash
docker restart php-72-apache-xdebug-30
```

