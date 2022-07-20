---
title: python面向对象基础学习
date: 2020-07-09
image: https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210228095632.jpeg
description: 重学面向对象
categories: 
- note
tags:
- 开发
---


说了暑假学开发，绝对不咕咕咕（0%）

------

### 面向过程

![](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120171712.png)

假如你拥有了一个做饭洗碗的任务，你准备用面向过程的思想来解决它

可以看到过程非常的繁琐，你将面对所有的步骤

------

### 面向对象

![https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120171728.png](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120171728.png)

假如你现在有个对象，她拥有两个技能：

1.做饭
2.洗碗

那么你只需要面向你的对象，然后你的对象去处理其他所有过程

![](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120171745.png)

那如果你的对象再拥有两个对象：

1.炒菜机
2.洗碗机

那么她只需要每次都做三个动作，不必一次面向所有步骤

------

很妙！对不对？

你只需要对你的对象说需要做什么，然后你的对象就会帮你做好所有的事情，**你既看不到具体的过程，你也不需要知道具体的过程**

你的对象也不需要知道具体的炒菜或者洗碗的步骤，她只需对洗碗机或者炒菜机说需要做什么，洗碗机或者炒菜机就会帮她做完所有的步骤

这个就叫做**封装**。

把搅拌、翻炒、监测火候封装成自动炒菜机
把倒洗洁精、刷碗、擦干水封装成自动洗碗机
再把剩下的封装成你的女朋友

------

### 两者的对比

- 面向对象本身是对面向过程的**封装**
- 面向过程什么最重要？
  - 按步骤划分
  - 把每个任务，分解成具体的每个步骤
- 面向对象什么最重要？
  - 按功能就行划分
  - 找到对象，确定对象的属性、行为
- 如何从面向过程思想转化为面向对象思想？
  1. 和面向过程一样，先找到所有步骤
  2. 试图分离功能代码块
  3. 将这些功能代码块，划分到某对象中
  4. 根据对象行为，抽象出 “类”
- 对象和类的关系
  - 对象->抽象->类->实例化->对象



------

### 对象和类

对象和类里面有两个东西：

1. 属性
2. 方法

属性是静态的、用来描述这个类或者对象本身的变量

比如，张三这个对象，他的属性有年龄、性别、身高、体重
人这个类属性也有年龄、性别、身高、体重

张三能执行吃饭这个动作，这就是他的一个**方法**

现在我们来新建一个类，把他实例化为一个对象并且给他一些属性：

```
# python3
class Person:
    pass
 
zhangsan = Person()#实例化为对象
 
zhangsan.age = 19#给zhangsan对象添加一个属性age并且赋值为19
zhangsan.height = 190#给zhangsan对象添加一个属性height并且赋值为190
 
print(zhangsan.age)#测试能否访问这个属性
print(zhangsan.__dict__)#输出zhangsan所有属性
print(zhangsan.__class__)#输出zhangsan的类
print(zhangsan.__class__.__name__)#输出zhangsan的类的名称
 
运行输出：
19
{'age': 19, 'height': 190}
<class '__main__.Person'>
Person
```

下面是这段程序在内存中的状态：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120171758.png)

其实在python中，万物皆为对象，包括int，float，str这些数据类型也都来自对象。

比如：在python的命令行里面执行：

```
Python 3.8.1 (tags/v3.8.1:1b293b6, Dec 18 2019, 23:11:46) [MSC v.1916 64 bit (AMD64)] on win32
Type "help", "copyright", "credits" or "license" for more information.
>>> ''.__class__
<class 'str'>
>>> ''.__class__.__class__
<class 'type'>
>>>
```

我们可以看到，str类型其实是来自一个叫做str的类，然后这个str的类来自type对象。实际上这个`class 'str'`也是一个对象，也就是说，我们定义一个类的时候，这个类本身也是一个对象。

为了方便区别，我们也把我们自己定义的那个类叫做“类”，把他实例化得到的对象称作“实例”

------

### 属性和方法

#### 属性

1.属性分为**类属性**和**实例属性**。

下面我们来操作一下：

```
#python3
class Person:
    age = 18
 
zhangsan = Person()
zhangsan.sex = 'man'
 
print(zhangsan.sex)
print(Person.age)
 
运行得到：
man
18
```

可见，我们是可以通过`类名.属性名`来访问一个类属性，前面我们说过，这个Person类其实也是一个对象，Person这个变量指向了这个类（对象）存放的内存地址，当然也就可以Person.age访问一个类属性

2.**类属性**为所有实例共享

显而易见，但你修改或者定义了Person类的一个属性，比如age，那么这个类的所有实例都可以拥有这个属性；

当你访问一个实例的属性时，具体步骤为：

1. 在当前实例里面（__dict__）查找这个属性
2. 如果没有，则在上级查找这个属性（.__class__.__dict__）
3. 如此反复，直到基类，没有则报错

3.类属性和实例属性的增删改查

增添：

- 实例名.属性名=xxx
- 实例名.__class__.属性名=xxx
- 类名.属性名=xxx
- 可以通过在类里面定义一个 `类名.__slots__={"aaa"}` 来限制此类的实例只能添加aaa属性

删除：

- del 实例名.属性名
- del 类名.属性名
- 只能删当前类，不能向上一级删除

#### 方法

同理，分为**实例方法**、**类方法**和**静态方法**

1.方法声明和储存都在**类对象**里面

不管是类方法还是实例方法，都必须在类里面进行声明，然后方法名称储存在类里面，指向这个方法本身的内存地址

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120171808.jpeg)

2.实例方法接受的第一个参数必须是一个实例，但是通过实例调用的时候不必填写第一个参数

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120171820.gif)

我们在创建一个实例方法的时候编辑器自动给我们补全了第一个参数self

这里个self只是一个形式参数，可以写成其他名字，不过self更好

3.类方法接受的第一个参数必须是一个类对象，但是通过实例调用时候不必填写第一个参数

如上同理，形式参数为cls，不过在声明的时候需要在方法前面加上@classmethod装饰器

<img src="https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120171830.gif" alt="img" style="zoom:80%;" />

4.静态方法没有必须接受的参数

定义静态方法的时候需要在上一行加入装饰器`@staticmethod`

5.实例方法的几种调用方式

- 通过实例调用方法

调用的时候不必我们传入第一个参数，python自动把p对象传给了第一个参数

<img src="https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120171842.gif" alt="img" style="zoom:80%;" />

- 通过类调用

我们需要手动传递第一个参数，不然就会报错，本质上是直接找到内存中这个方法的地址进行调用

<img src="https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120171854.gif" alt="img" style="zoom:80%;" />

6.类方法或者实例方法，都可以通过实例或者类调用而不必手动传入第一个参数，而直接调用的时候需要手动传入第一个参数

比如我们下面一段代码：

```
#python3
aaa = 'a,aa,aaa'
a = aaa.split(',')#split是class str的里面的一个实例方法，相当于我们传入的第一个参数是aaa这个实例
print(a)
```

------

### 属性的访问权限

python不像其他语言可以通过private、protect这些关键词来修饰属性，python只能通过在属性名称前面添加一个或者两个下划线来声明一个伪私有的属性。

不说了，开局一张图：

单下划线开头变量：

<img src="https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120171904.png" alt="img" style="zoom:80%;" />

双下划线开头变量：

<img src="https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120171916.png" alt="img" style="zoom:80%;" />

感叹号表示会发出警告，×表示不能访问

但是我们还是可以通过 `_类名__属性名` 来访问这个属性：

<img src="https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120171926.gif" alt="img" style="zoom:80%;" />