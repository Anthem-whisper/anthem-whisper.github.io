---
title: "Jolokia读写文件新姿势"
date: 2021-12-01T13:11:00+08:00
draft: false
image: https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202112011312493.png
description: 
comments: true
license: false
math: false
categories:
- note
tags:
- Java
- Jolokia
---

之前看到一个Jolokia新的利用姿势：

[SSRF to RCE with Jolokia and MBeans](https://thinkloveshare.com/hacking/ssrf_to_rce_with_jolokia_and_mbeans/)

这篇文章介绍了在SSRF只能GET请求的场景下，Jolokia一些读写文件，加载.so文件的利用方法

我参考这篇文章，结合了之前APISandbox的一些打法出在了 [NCTF 2021](https://ctf.njupt.edu.cn/727.html) 上面

------

Java管理扩展（JavaManagementExtensions，JMX）是一种Java技术，它提供用于管理和监视应用程序、系统对象、设备（如打印机）和面向服务的网络的工具，这些资源由被称为MBean（Managed Bean）的对象表示。在JMX API中，可以动态加载和实例化类。可以使用Java动态管理工具包设计和开发管理和监视应用程序。



Jolokia是一个JMX-HTTP桥接器，它可以利用JSON通过HTTP实现JMX远程管理，具有快速、简单等特点。除了支持基本的JMX操作之外，它还提供一些独特的特性来增强JMX远程管理如：批量请求，细粒度安全策略等。



我们通过阅读[官方文档](https://jolokia.org/reference/html/protocol.html)可以知道，Jolokia URL模式大概是：`/jolokia/action/package:MBeanSelector/method/param1/param2`这个样子，使用`/`的时候需要用`!`转义



文章中提到了几种（GET请求触发）手法，分别是：

- 使用`vmSystemProperties`获取JVM信息
- 使用`JavaFlightRecorder`任意文件写
- 使用`compilerDirectivesAdd`任意文件读
- 使用`jvmtiAgentLoad`任意加载 .so
- 使用`vmLog`写入日志文件



作者通过`vmlog`往Tomcat ROOT路径写jsp webshell来实现从SSRF到RCE的转变。



![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202112011307334.png)



当时正在构思NCTF校赛出什么题目，看到这几种利用手法，就想着结合之前APISandbox的一些东西，出一道actuator的综合利用，题目设计大概是这样的：



一个正常的Springboot应用，`/actuator`配置里面暴露了一些本来不应该暴露的端点，其中就有jolokia，env等



![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202112011307472.png)



直接访问`/actuator/jolokia/`会403，是因为我使用SpringSecurity限制了本地IP访问



用APIKit扫描，或者访问`/actuator/mappings`可以看到有一个隐藏的API`/user/list`：



![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202112011308743.png)



这个API返回XML数据，可以自然想到XXE。

为了不让XXE直接读文件，我加了waf过滤了除http协议之外的协议，预期是利用XXE来SSRF打`/actuator/jolokia/`。



这儿有俩坑点，docker端口是开在58082的，SSRF的时候请求端口需要访问`/actuator/env`来获得，是8080



构造SSRF之后，访问`/actuator/jolokia/list`会发现报错：



![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202112011308716.png)



这是因为`/jolokia/list`返回的数据太长了，而且里面有一些特殊符号会报`XML document structures must start and end within the same entity.`。



于是后面给了附件pom.xml，可以本地起起来看一下有什么Mbean。



![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202112011308295.png)



然后可以判断远程环境是否存在这个Mbean：



![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202112011308676.png)



如果不存在返回的是上图，如果存在返回的是下图两种情况



![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202112011308232.png)



那么便可以直接用`com.sun.management:type=DiagnosticCommand/compilerDirectivesAdd/!/flag`来读取flag了。