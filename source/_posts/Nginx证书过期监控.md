---
title: Nginx证书过期监控
date: 2021-03-12 06:12:29
tags:
- zabbix
- nginx
---



今天突然接到阿里云电话nginx证书即将过期，所以运维也需要监控nginx证书是否过期。



# 环境准备

- zabbix
- python3

# 配置获取证书是否过期脚本

python脚本, 由于nginx文件在`/usr/local/nginx/conf/nginx.conf` 所以以下为

事先需要在agent主机上安装命令

```bash
curl -L https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -o /usr/local/bin/cfssl
chmod +x /usr/local/bin/cfssl
curl -L https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -o /usr/local/bin/cfssljson
chmod +x /usr/local/bin/cfssljson
curl -L https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -o /usr/local/bin/cfssl-certinfo
chmod +x /usr/local/bin/cfssl-certinfo

```

准备脚本`/etc/zabbix/scripts/certs.py`

```python
#!/usr/bin/python3
# 脚本会提前2个月告诉证书已经过期
# 0 存在过期      1 都没有过期

import datetime
import json
import subprocess
import sys


def cmdexec(cmd):
  p = subprocess.Popen(cmd, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
  stdout,stderr=p.communicate()
  stdout = stdout.decode()
  stderr = stderr.decode()
  return  stdout,stderr


cmds,_ = cmdexec(r"""grep -o '[[:space:]]ssl_certificate[[:space:]]\+[^;]\+' /usr/local/nginx/conf/nginx.conf | awk '{printf "cat %s  | sed -n \"1,/END CERTIFICATE/p\" | cfssl-certinfo -cert -  \n",$2}'""")

result = []
expires = []
for line in cmds.splitlines():
  jsonstr,_ = cmdexec(line)
  j = json.loads(jsonstr)
  #print(j,type(j)) # DICT, <dict>

  # a = datetime.datetime.strptime("2021-04-12T12:00:00Z", "%Y-%m-%dT%H:%M:%SZ")

  expire_datetime = j.get("not_after")
  #print("过期时间 {}".format(expire_datetime))
  a = datetime.datetime.strptime(expire_datetime, "%Y-%m-%dT%H:%M:%SZ")
  b = datetime.datetime.now()
  day_delta = datetime.timedelta(days=60)
  flag = (a - day_delta > b) # False 过期
  result.append(flag)
  if not flag:
    expire_name = j.get('subject').get('common_name')
    expires.append(expire_name)

if sys.argv[1] == "status":
  print(0 if False in result else 1)
else:
  print(expires)
```

# 配置agent

```diff
"/etc/zabbix/zabbix_agentd.conf" 24L, 660C                                                                                                                                                                                                                                                               23,1          All
PidFile=/var/run/zabbix/zabbix_agentd.pid
LogFile=/var/log/zabbix/zabbix_agentd.log
LogFileSize=10
DebugLevel=3
SourceIP=172.16.0.221
EnableRemoteCommands=1
LogRemoteCommands=1
Server=172.16.0.222,172.16.0.225,172.16.0.223

ListenPort=10050
StartAgents=10
ServerActive=172.16.0.222
Hostname=Zabbix Server
RefreshActiveChecks=120
BufferSize=200
Timeout=30
AllowRoot=1
Include=/etc/zabbix/zabbix_agentd.d/*.conf
UnsafeUserParameters=0
UserParameter=haproxy,/etc/zabbix/scripts/haproxy.sh
UserParameter=mongo_slow,/etc/zabbix/scripts/mongo_slow.py
UserParameter=mysql_slow[*],/etc/zabbix/scripts/mysql_slow.py
+UserParameter=certs[*],/etc/zabbix/scripts/certs.py $1
```



测试zabbix脚本正常

```bash
[root@k8s-master1 ~]# zabbix_get -s 172.16.0.221 -k 'certs[x]'
[] # 获取不正常的列表
[root@k8s-master1 ~]# zabbix_get -s 172.16.0.221 -k 'certs[status]'
1 # 1表示都正常，0表示至少有1个不正常
```



# 配置zabbix web

## 配置监控项

![image-20210312141749085](http://myapp.img.mykernel.cn/image-20210312141749085.png)

![image-20210312141800392](http://myapp.img.mykernel.cn/image-20210312141800392.png)



![image-20210312141808019](http://myapp.img.mykernel.cn/image-20210312141808019.png)

> 实验可以写1s, 可以快速获取

## 查看数据

![image-20210312141919067](http://myapp.img.mykernel.cn/image-20210312141919067.png)

0表示过期

1表示都正常



## 配置触发器

![image-20210312141940129](http://myapp.img.mykernel.cn/image-20210312141940129.png)

![image-20210312141949031](http://myapp.img.mykernel.cn/image-20210312141949031.png)

![image-20210312141955345](http://myapp.img.mykernel.cn/image-20210312141955345.png)

## 查看告警

![image-20210312142007472](http://myapp.img.mykernel.cn/image-20210312142007472.png)

## 配置动作

即告警产生后，得发送邮件让运维晓得

![image-20210312142041420](http://myapp.img.mykernel.cn/image-20210312142041420.png)

等5分钟

![image-20210312142051528](http://myapp.img.mykernel.cn/image-20210312142051528.png)

![image-20210312142058328](http://myapp.img.mykernel.cn/image-20210312142058328.png)

# 解决问题

##  获取过期的nginx域名

![image-20210312142114440](http://myapp.img.mykernel.cn/image-20210312142114440.png)

## 申请证书

![image-20210312142134025](http://myapp.img.mykernel.cn/image-20210312142134025.png)

## 下载证书

![image-20210312142143527](http://myapp.img.mykernel.cn/image-20210312142143527.png)

## 上传服务器

![image-20210312142201319](http://myapp.img.mykernel.cn/image-20210312142201319.png)

## nginx配置新证书

![image-20210312142212137](http://myapp.img.mykernel.cn/image-20210312142212137.png)

```bash
[root@izpxkqmt2zlej4z live-ssl]# /usr/local/nginx/sbin/nginx -t
nginx: the configuration file /usr/local/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful
[root@izpxkqmt2zlej4z live-ssl]# /usr/local/nginx/sbin/nginx -s reload
```

## 网页验证

![image-20210312142235187](http://myapp.img.mykernel.cn/image-20210312142235187.png)

## 报警消除

![image-20210312142254544](http://myapp.img.mykernel.cn/image-20210312142254544.png)

![image-20210312142300586](http://myapp.img.mykernel.cn/image-20210312142300586.png)

![image-20210312142523534](http://myapp.img.mykernel.cn/image-20210312142523534.png)