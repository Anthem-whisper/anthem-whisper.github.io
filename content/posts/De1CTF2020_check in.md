---
title: De1CTF2020_check in
date: 2020-05-04
image: https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120170416.png
description: 
categories: 
- ctf_writeup
tags:
- 文件上传
---
本次De1CTF是XCTF的第一场分站赛，不但吸引了国内许许多多的战队，还有非常多的国外的战队，据说前26名一半都是国外的。当然比赛题目也是非常的难，感觉自己在这种大比赛上面还是不能发挥什么作用。

------

### check in

打开就是一个经典的文件上传页面，上传一个普通的图片马，页面回显：

```
perl|pyth|ph|auto|curl|base|>|rm|ruby|openssl|war|lua|msf|xter|telnet in contents!
```

文件内容被ban了这些，

随后尝试了一些文件名称，只要文件名带`ph、html、.user.ini`之类的都会直接返回filename error

不过这么多东西都被ban了，唯独 `.htaccess` 没有被ban。

不过这个我开局就尝试了，但是没有发现什么姿势。

#### 关于 .htaccess 几种姿势：

1. 将特定文件作为php解析，用作后门。
2. PHP环境下使用 auto_prepend_file 或 auto_append_file 创建后门（对于CGI/FastCGI模式 PHP 5.3.0 以上版本，还可以使用 在目录下创建.user.ini文件 。来引入该参数）
3. CGI启动方式的RCE利用姿势
4. FastCGI启动方式的RCE利用姿势
5. [重定向](https://skysec.top/2017/09/06/有趣的-htaccess/#Tokyo-Westerns-MMA-CTF-2nd-2016应用)

 

这道题ban掉了ph关键字之后，（1）中的方法看似不可以用，实则另有玄机。

在XNUCA2019中，一道web题目ezphp的非预期解中，利用到了 `\` 拼接 `.htaccess` 文件中换行的内容，当时一种是为了利用第一行的 `#` 符号拼接第二行的垃圾数据来达到注释垃圾数据的目的，另一种也是 `\` 拼接绕过关键字过滤。具体可以参看[XNUCA2019 ez系列web题解](https://xz.aliyun.com/t/6111)和[X-NUCA 2019 Web WP](https://evi0s.com/2019/08/30/x-nuca-2019-web-wp/)

文件名 `.htaccess`，MIME改成 `image/jpg` ，文件内容：

```
AddType application/x-httpd-p\
hp .jpg
```

![img](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120170428.png)

那么我们通过\来拼接绕过了ph关键字之后，我们可以发现我们上传的jpg文件不再是无法查看的情况：

![img](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120170439.png)

第二个点，PHPecho短标签。

过滤了 `>` 和 `ph` 的情况下， `<?php ?>` 和 `<script>` 这种被通杀了，参考网上的[一篇文章](https://wp.hellocode.name/?p=470)：

#### PHP 的四种标签写法

```
<?php echo 1; ?> //正常写法
<? echo 1; ?> //短标签写法，5.4 起 <?= 'hello'; === <? echo 'hello';
<% echo 1; %> //asp 风格写法
<script language="php"> echo 1; </script> //长标签写法
```

不同版本的区别

第 1 种是正常写法，没什么可说的。

第 2 种，需要 php.ini 配置文件中的指令 `short_open_tag` 打开后才可用，或者在 PHP 编译时加入了 `--enable-short-tags` 选项。自 PHP5.4 起，短格式的 echo 标记 `<?=` 总会被识别并且合法，而不管 `short_open_tag` 的设置是什么。

第 3 种，不推荐写法，为了 asp 程序员学习 php 所添加的语法糖写法。需要通过 `php.ini` 配置文件中的指令 `asp_tags` 打开后才可用。

第 4 种，在 php7.0 后已经不解析了。

那就直接ojbk：

![img](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed@main/img/20210120170456.png)

De1ctf{cG1_cG1_cg1_857_857_cgll111ll11lll}