---
title: Padding Oracle Attack&CBC字节翻转攻击
date: 2020-04-21
image: 
description: 由NPUCTF 2020一道题学习Padding Oracle Attack和CBC字节翻转攻击
categories: 
- note
tags:
- WebCrypto
---
### **前言**

在西工大举办的NPUCTF2020上面做到一道webcrypto的题，以前从来没接触过密码安全方向，这次正好学一下入个门

### **正文**

参考

Padding Oracle Attack：

《白帽子讲web安全》P239

[>>一叶飘零师傅](https://skysec.top/2017/12/13/padding-oracle和cbc翻转攻击/)

CBC翻转攻击：

[>>某师傅](https://www.jianshu.com/p/7f171477a603)

[>>某某师傅](https://blog.csdn.net/include_heqile/article/details/79942993)

先来讲一讲CBC模式加密原理：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120170001.png)

 

1. 首先将明文（Plaintext）分组(常见的以16或8字节为一组)，位数不足的使用特殊字符填充。
2. 生成一个随机的初始化向量(IV)和一个密钥。
3. 将IV和第一组明文异或（xor运算）。
4. 用密钥对3中xor后产生的密文加密。
5. 用4中产生的密文对第二组明文进行xor操作。
6. 用密钥对5中产生的密文加密。
7. 重复4-7，到最后一组明文。
8. 将IV和加密后的密文拼接在一起，得到最终的密文

解密过程：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120170010.png)

1. 先从密文中取出IV，然后对剩下的密文分组（16或8字节为一组）
2. 使用秘钥解密第一组密文，将解密结果与IV做异或运算，得到明文1
3. 然后使用秘钥解第二组密文，将解密的结果与上一组密文进行异或运算，得出明文2
4. 重复2-3，直至所有密文解密完毕

 

以上就是CBC模式的加密解密过程，接下来讲两种手段：

#### **Padding Oracle Attack**

直译为 “填充Oracle攻击” ，这里主要关注一下解密过程：

密文cipher首先进行一系列处理，如图中的Block Cipher Decryption
我们将处理后的值称为 `middle` 中间值
然后 `middle` 与我们输入的iv进行异或操作
得到的即为明文
但这里有一个规则叫做Padding填充：
因为加密是按照16位一组分组进行的
而如果不足16位，就需要进行填充

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120170017.png)

有几个空，就要填充几个“几”

比如明文为admin，那么需要填充的就是 `admin\x0b\x0b\x0b\x0b\x0b\x0b\x0b\x0b\x0b\x0b\x0b` （11个\x0b）

如果我们输入一个错误的iv，依旧是可以解密的，但是 `middle` 和我们输入的iv经过异或后得到的填充值可能出现错误

比如本来应该是 `admin\x0b\x0b\x0b\x0b\x0b\x0b\x0b\x0b\x0b\x0b\x0b`
而我们错误的得到 `admin\x0b\x0b\x0b\x0b\x0b\x0b\x0b\x0b\x0b\x3b\x2c`

这样解密程序往往会抛出异常(Padding Error)

应用在web里的时候，往往是302或是500报错
而正常解密的时候是200
所以这时，我们可以根据服务器的反应来判断我们输入的iv

看例子：

我们假设middle中间值为：

```
0x39 0x73 0x23 0x22 0x07 0x6a 0x26 0x3d
```

正确的解密iv应该为

```
0x6d 0x36 0x70 0x76 0x03 0x6e 0x22 0x39
```

解密后正确的明文为：

```
T E S T 0x04 0x04 0x04 0x04
```

但是关键点在于，我们可以知道iv的值，却不能得到中间值和解密后明文的值
而我们的目标是**只根据我们输入的iv值和服务器的状态去判断出解密后明文的值**
这里的攻击即叫做 `Padding Oracle Attack` 攻击
这时候我们选择进行爆破攻击

如果我们构造一个iv为：

```
0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00
```

那么 `middle` 的值和这个iv异或将会得到原封不动的 `middle` 的值

```
0x39 0x73 0x23 0x22 0x07 0x6a 0x26 0x3d
```

现在这个解密结果是不对的

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120170100.png)

 

正确的 `padding` 值只可能是：

- 1个字节的padding为0x01
- 2个字节的padding为0x02，0x02
- 3个字节的padding为0x03，0x03，0x03
- 4个字节的padding为0x04，0x04，0x04，0x04
- ……

因此我们希望慢慢调整iv的值，并且希望解密后最后一个值为正确的 `padding` 比如一个0x01，我们于是遍历最后一位iv：

 

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120170112.png)

那么最后一位中间密文就是： `0x01^0x3C=0x3D` （这个一定成立，看图）,原来的明文就是 `0x3D^0x0F=0x32`（中间密文^原来的iv）

知道了最后一位的中间密文，就可以遍历倒数第二位iv了，这个时候应该为 `0x02` 而非 `0x01` 了。看图就懂：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120170121.png)

以此类推，我们可以就能推算所有中间密文，再用 `中间密文^原来的iv` 就能算出明文了

#### **CBC字节翻转攻击**

有了上面的CBC加密解密过程的基础，这个手段其实很容易理解；

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120170128.png)

由解密算法可知：

```
A=B^C
```

由 `^` 运算的性质我们可以知道：

```
A=B^C、B=A^C、C=A^B
```

这是最关键的一点，我们可以推导出三者做异或运算的结果是0

```
C=A^B
C^C=A^B^C=0
```

我们修改了B的值，就一定会影响到A

```
B^X^C=A^X
```

也就是说，我们只要给B异或了X，A的值也会改变为他之前的值异或X的结果

#### **一 道CTF的例子**

NPUCTF2020_web🐕

```
我摊牌了，就是懒得写前端
<?php 
error_reporting(0);
include('config.php');   # $key,********$file1*********
define("METHOD", "aes-128-cbc");  //定义加密方式
define("SECRET_KEY", $key);    //定义密钥
define("IV","6666666666666666");    //定义初始向量 16个6
define("BR",'<br>');
if(!isset($_GET['source']))header('location:./index.php?source=1');


#var_dump($GLOBALS);   //听说你想看这个？
function aes_encrypt($iv,$data)
{
    echo "--------encrypt---------".BR;
    echo 'IV:'.$iv.BR;
    return base64_encode(openssl_encrypt($data, METHOD, SECRET_KEY, OPENSSL_RAW_DATA, $iv)).BR;
}
function aes_decrypt($iv,$data)
{
    return openssl_decrypt(base64_decode($data),METHOD,SECRET_KEY,OPENSSL_RAW_DATA,$iv) or die('False');  #不返回密文，解密成功返回1，解密失败返回False
}
if($_GET['method']=='encrypt')
{
    $iv = IV;
    $data = $file1;    
    echo aes_encrypt($iv,$data);
} else if($_GET['method']=="decrypt")
{
    $iv = @$_POST['iv'];
    $data = @$_POST['data'];
    echo aes_decrypt($iv,$data);
}
echo "我摊牌了，就是懒得写前端".BR;

if($_GET['source']==1)highlight_file(__FILE__);
?>
```

padding Oracle Attack

exp：

```
# coding:utf-8
import requests
import base64

# b'\x97.\xda\xb8\xa5P\t\x95\xae\x9b\xf5\xbf\xe2\x8b.<'
CYPHERTEXT = base64.b64decode("ly7auKVQCZWum/W/4osuPA==")
# initialization vector
IV = "6666666666666666"
# PKCS7 16个字节为1组
N = 16
# intermediaryValue ^ IV = plainText
inermediaryValue = ""
plainText = ""
# 爆破时不断需要更改的iv
iv = ""
URL = "http://webdog.popscat.top/index.php?method=decrypt&source=1"


def xor(a, b):
    """
    用于输出两个字符串对位异或的结果
    """
    return "".join([chr(ord(a[i]) ^ ord(b[i])) for i in range(len(a))])


for step in range(1, N + 1):
    padding = chr(step) * (step - 1)
    print(step,end=",")
    for i in range(0, 256):
        print(i)
        """
        iv由三部分组成：
            待爆破位置 chr(0)*(N-step) 
            正在爆破位置 chr(i) 
            使 iv[N-step+1:] ^ inermediaryValue = padding 的 xor(padding,inermediaryValue)
        """
        iv = chr(0)*(N-step)+chr(i)+xor(padding,inermediaryValue)
        data = {
            "data": "ly7auKVQCZWum/W/4osuPA==",
            "iv": iv
        }
        r = requests.post(URL,data = data)
        if r.text !="False":
            inermediaryValue = xor(chr(i),chr(step)) + inermediaryValue
            print(inermediaryValue)
            break
plainText = xor(inermediaryValue,IV)
print(plainText)
```

运行，得到 `FlagIsHere.php` 访问之，得到如下源码：

```
F7LMTk/3nKSVUoSQuOS/dA==
<?php 
#error_reporting(0);
include('config.php');    //**************$file2********last step!!
define("METHOD", "aes-128-cbc");
define("SECRET_KEY", "6666666");
session_start();

function get_iv(){    //生成随机初始向量IV
    $random_iv='';
    for($i=0;$i<16;$i++){
        $random_iv.=chr(rand(1,255));
    }
    return $random_iv;
}

$lalala = 'piapiapiapia';

if(!isset($_SESSION['Identity'])){
    $_SESSION['iv'] = get_iv();

    $_SESSION['Identity'] = base64_encode(openssl_encrypt($lalala, METHOD, SECRET_KEY, OPENSSL_RAW_DATA, $_SESSION['iv']));
}
echo base64_encode($_SESSION['iv'])."<br>";

if(isset($_POST['iv'])){
    $tmp_id = openssl_decrypt(base64_decode($_SESSION['Identity']), METHOD, SECRET_KEY, OPENSSL_RAW_DATA, base64_decode($_POST['iv']));
    echo $tmp_id."<br>";
    if($tmp_id ==='weber')die($file2);
}

highlight_file(__FILE__);
?>
```

整理一下已知信息：

```
iv=
F7LMTk/3nKSVUoSQuOS/dA==
\x17 \xb2 \xcc \x4e \x4f \xf7 \x9c \xa4 \x95 \x52 \x84 \x90 \xb8 \xe4 \xbf \x74 
加密后：
$Identity='MLvuYeH07rhiAa5NJ1p75A=='
$Identity='\x30 \xbb \xee \x61 \xe1 \xf4 \xee \xb8 \x62 \x01 \xae \x4d \x27 \x5a \x7b \xe4 '
```

目的就是传入新的iv对identity进行解密，如果解密结果为'weber'那么就爽歪歪，这里考察的就是CBC字节翻转攻击

和Padding Oracle Attack不一样，这里不需要推测中间密文，根据我上面说的

```
B^X^C=A^X
```

本来是 `piapiapiapia\x04\x04\x04\x04` 现在我们需要改为 `weber\x0B\x0B\x0B\x0B\x0B\x0B\x0B\x0B\x0B\x0B\x0B` ，就拿第一位来说：

我们想要把 `p` 改为 `w` ，那么我就要找出 `X` 的值， `'p'^X='w'` 很容易算出 `X='p'^'w'` 那么我们只需要在将B异或一个 `('p'^'w')` ，就可以达到目的。

不过当时比赛的时候平台被D了，我是在出题人的学生机上面跑，脚本应该没问题，但是跑不出来，换了正常环境iv又不一样懒得重写，所以用了学长的脚本：

```
import base64
def bxor(b1, b2): # use xor for bytes
    parts = []
    for b1, b2 in zip(b1, b2):
        parts.append(bytes([b1 ^ b2]))
    return b''.join(parts)
iv = base64.b64decode("h34HL5RbMPw8oTaQ+P58nw==")
text = b"piapiapiapia\x04\x04\x04\x04"
result = b"weber\x0b\x0b\x0b\x0b\x0b\x0b\x0b\x0b\x0b\x0b\x0b"
middle = bxor(iv,text)
iv = bxor(middle,result)
print(base64.b64encode(iv))
```

跑出来POST一个iv过去得到一个网址： `https://c-t.work/s/034d3b3bf3fb48||verification code:2q2hwm`

好像是有个附件，下载来是一个xxx.class文件考的Java反编译。。。不过百度能搜到，用工具 `jd-gui-1.4.0` 一下就跑出来。

得到数组，就是flag的ASCII码

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120170140.png)

```
q = [102, 108, 97, 103, 123, 119, 101, 54, 95, 52, 111, 103, 
     95, 49, 115, 95, 101, 52, 115, 121, 103, 48, 105, 110, 103, 125]
for i in q:
    print(chr(i), end='')
```

flag{we6_4og_1s_e4syg0ing}