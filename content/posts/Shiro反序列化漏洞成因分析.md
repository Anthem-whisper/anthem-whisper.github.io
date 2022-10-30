---
title: "Shiro反序列化漏洞成因分析"
date: 2021-10-04T23:19:01+08:00
draft: false
image: https://shiro.apache.org/assets/images/apache-shiro-logo.png
description: 
comments: true
license: false
math: false
categories:
- note
tags:
- Shiro
- Java
- 反序列化
---

## 概述

> Apache Shiro是一个强大且易用的Java安全框架,执行身份验证、授权、密码和会话管理。使用Shiro的易于理解的API,您可以快速、轻松地获得任何应用程序,从最小的移动应用程序到最大的网络和企业应用程序。

它的原理比较简单：**为了让浏览器或服务器重启后用户不丢失登录状态，Shiro支持将持久化信息序列化并加密后保存在Cookie的rememberMe字段中，下次读取时进行解密再反序列化。但是在Shiro 1.2.4版本之前内置了一个默认且固定的加密Key,导致攻击者可以伪造任意的rememberMe Cookie,进而触发反序列化漏洞。**

![243e118990160b756a782b94162e1cfb.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042151507.png)

Shiro反序列化漏洞目前为止有两个，Shiro-550(Apache Shiro < 1.2.5)和Shiro-721( Apache Shiro < 1.4.2 )。这两个漏洞主要区别在于Shiro550使用已知密钥撞，后者Shiro721是使用登录后rememberMe={value}去爆破正确的key值进而反序列化，对比Shiro550条件只要有足够密钥库（条件比较低）、Shiro721需要登录（要求比较高鸡肋）。

Apache Shiro < 1.4.2默认使用AES/CBC/PKCS5Padding模式
Apache Shiro >= 1.4.2默认使用AES/GCM/PKCS5Padding模式



## Shiro-550：Hard Code->Deserialize->RCE

Shiro 550 反序列化漏洞存在版本：`shiro <1.2.4`，产生原因是因为shiro接受了Cookie里面rememberMe的值，然后去进行Base64解密后，再使用aes密钥解密后的数据，进行反序列化。

这个aes密钥是硬编码（简称写死），也就是他密钥是写死在jar包里面的，众所周知AES 是对称加密，即加密密钥也同样是解密密钥，那如果我们能知道了这个密钥就可以伪造恶意cookie

接下来我们从Cookie的加密和解密过程来了解shiro-550

### Cookie加密过程

直接来看shiro的CookieRememberMeManager

在`org.apache.shiro.web.mgt.CookieRememberMeManager#rememberSerializedIdentity`里面，存在一个将serialized数据Base64加密然后作为Cookie返回的行为

![68c05ff91ce4b5d0bda4e3ef5a5294c7.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042152812.png)

我们看下哪些地方调用了这个方法，狂摁Ctrl+B:

```
org.apache.shiro.web.mgt.CookieRememberMeManager#rememberSerializedIdentity<-
org.apache.shiro.mgt.AbstractRememberMeManager#rememberIdentity<-
org.apache.shiro.mgt.AbstractRememberMeManager#rememberIdentity(重载)<-
org.apache.shiro.mgt.AbstractRememberMeManager#onSuccessfulLogin
```

![d1355b042c9bb75aa20cc794ef024c0f.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042152203.png)

看到这个函数名都知道是登陆成功调用的，如果继续跟下去的话，会有：

```
org.apache.shiro.mgt.DefaultSecurityManager#rememberMeSuccessfulLogin <-
org.apache.shiro.mgt.DefaultSecurityManager#onSuccessfulLogin<-
org.apache.shiro.mgt.DefaultSecurityManager#login<-
……
```

会追溯到Filter之类的，大概就是：

登陆->登陆成功->设置Base64编码后的AES加密的Cookie

在onSuccessfulLogin方法这里下个断点

![631811c25930cc1b6d1ed4c9530d524d.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042152804.png)

在调用rememberIdentity之前先调用isRememberMe判断了用户是否选择了RememberMe选项，如果选了进入rememberIdentity方法

![9a1b38eb40908ae63306d4eaed1ec1be.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042152864.png)

这个方法先创建一个PrincipalCollection对象，包含了登录信息。

随后进入rememberIdentity方法

![26d9f73c5c423aaa187501040d10c476.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042152701.png)

这个方法调用convertPrincipalsToBytes把序列化后的PrincipalCollection对象加密，然后返回

![993fa60071dabd0240454635d6b9fada.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042152777.png)

而这个seriallize方法，调用org.apache.shiro.mgt.AbstractRememberMeManager#getEncryptionCipherKey去获取加密的key

![5f3a1d1b9ed86f045752a2cdd9268da2.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042152321.png)

跟进，发现直接返回了一个属性

![c39d6952fa91d3d667017cb00a89d783.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042152086.png)

转到定义，这个属性貌似是预先定义好的，~~虽然没看出究竟是哪里定义的~~（其实是可以看到的，详见《构造shiro poc》)

不过我们可以看到一个叫做DEFAULT\_CIPHER\_KEY_BYTES的东西，这个就是传说中的硬编码的shirokey

![2743873090f3be29943976ce4647b9bf.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042152168.png)

之后就是调用rememberSerializedIdentity返回base64加密的cookie了。

接下来康康解密过程：

### Cookie解密过程

我们其实可以猜测，加密解密的功能实际上都是由这个`org.apache.shiro.web.mgt.CookieRememberMeManager`类来实现的，在这个类里面四处找一找，可以找到getRememberedSerializedIdentity方法里面有一行：

![ad7dbe2df716477c2c27e20ea47c3a7a.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042152497.png)

这个很像获取Cookie然后去读取值的操作，在这里下个断点，带着Cookie访问服务，果然就断下来了

单步跟进，发现他获取到了我们的Cookie：

![0ab70e6f2d839e52f31de004ef9f28b7.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042153771.png)

随后判断了一下我们Cookie的值是不是等于DELETED\_COOKIE\_VALUE (deleteMe)，如果不是则进行decode并且返回：

![c161dcb1ad4ded67b72cbc65a73b835a.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042153075.png)

返回到了这里：

![9cb23a32e0b8dcbd3d9030b77e63a9b7.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042153249.png)

并且调用convertBytesToPrincipals（这个函数名字是不是很熟悉？），将Cookie的结果转化为凭据（PrincipalCollection对象）

因为之前加密过程调用convertPrincipalsToBytes，是一个序列化过程，那这里显然就是一个反序列化过程，跟进：

![f91f710ef425284287b1f10d913c6f1a.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042153938.png)

解密，而后反序列化；

跟进，触发点在`org.apache.shiro.io.DefaultSerializer#deserialize`：

![7a68bd2f3f5669d0c682bce50d995e31.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042153943.png)

### 1.2.5 版本修复

修改了org.apache.shiro.mgt.AbstractRememberMeManager的硬编码方式，并且去掉了默认key，采用随机生成的shiro AES key

![fb226eec7a9aaf44b840044e8d433cd5.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042153982.png)

但是这个key是可以自定义的：

```java
private static final String ENCRYPTION_KEY = "3AvVhmFLUs0KTA3Kprsdag==";
public CookieRememberMeManager rememberMeManager() {
        CookieRememberMeManager cookieRememberMeManager = new CookieRememberMeManager();
        cookieRememberMeManager.setCookie(rememMeCookie());
        // remeberMe cookie 加密的密钥 各个项目不一样 默认AES算法 密钥长度（128 256 512）
        cookieRememberMeManager.setCipherKey(Base64.decode(ENCRYPTION_KEY));
        return cookieRememberMeManager;
}
```

或者如果你使用了诸如Spring的框架：
spring-shiro.xml
在安全管理器SecurityManager中加入rememberMeManager；

![278339ec403dae1ff5618e3d7e8154d1.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042153200.png)

添加rememberMeManager，调用getCipherKey()随机生成密钥。

![b779730b16239139bd286cab88dcc9d6.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042154259.png)

**理论上只要AES加密钥泄露，都会导致反序列化漏洞**，也就是说，只要你硬编码，就有可能有爆破的风险

## Shiro-721：Padding Oracle Attack->Shiro AES key->shiro550

这个就不是重点了，shiro721本来利用需要先登陆获得有效的rememberMe={value}去爆破正确的key值进而反序列化，利用十分鸡肋。

关于Padding Oracle Attack看这篇：

[padding oracle和cbc翻转攻击](https://skysec.top/2017/12/13/padding-oracle%E5%92%8Ccbc%E7%BF%BB%E8%BD%AC%E6%94%BB%E5%87%BB/)

CBC加密模式：

![063308495a36da7002ebb344a31eafe3.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042154180.png)

CBC解密模式：

![f36fa75edd5315a3a26c3383fe8b8cbf.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042154970.png)

大概过程是这样：

> 比如我们的明文为admin
> 则需要被填充为 admin\\x0b\\x0b\\x0b\\x0b\\x0b\\x0b\\x0b\\x0b\\x0b\\x0b\\x0b
> 一共11个\\x0b
> 如果我们输入一个错误的iv，依旧是可以解密的，但是middle和我们输入的iv经过异或后得到的填充值可能出现错误
> 比如本来应该是admin\\x0b\\x0b\\x0b\\x0b\\x0b\\x0b\\x0b\\x0b\\x0b\\x0b\\x0b
> 而我们错误的得到admin\\x0b\\x0b\\x0b\\x0b\\x0b\\x0b\\x0b\\x0b\\x0b\\x3b\\x2c
> 这样解密程序往往会抛出异常(Padding Error)
> 应用在web里的时候，往往是302或是500报错
> 而正常解密的时候是200
> 所以这时，我们可以根据服务器的反应来判断我们输入的iv

如果发送的rememberMe可以正确解析

![9b6bb4f1012c1b4db623535cf3c94986.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042154063.png)

否则会抛出异常，返回deleteMe

![bfc213fa9f8693a6e6f5c9e922ceec02.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042154714.png)

通过这一点的不同，我们可以向服务发出一个oracle：“我这个iv解密出的padding对不对？”

如果是对的，正确解析，如果是错的返回deleteMe，基于此反复发出Oracle来爆破iv，再控制iv来控制解密后的明文（也就是不需要key了）

这里还有一点，为什么需要一个合法用户的rememberMe，因为Shiro会获取用户信息，如果不是合法用户也会返回异常从而抛出deleteMe，这样Oracle就没办法实现了。

Referer：

> [Java安全之Shiro 550反序列化漏洞分析](https://www.anquanke.com/post/id/225442)
> [浅谈Shiro反序列化获取Key的方式](https://sec-in.com/article/468)
> [Shiro 721 Padding Oracle攻击漏洞分析](https://www.anquanke.com/post/id/193165#h2-11)
> [Shiro-721 RCE Via Padding Oracle Attack](https://github.com/inspiringz/Shiro-721)