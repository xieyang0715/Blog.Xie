---
title: "firewall-cmd管理FILTER表"
date: 2021-05-06 09:24:16
tags:
- "firewall"
---



使用方法：https://blog.csdn.net/Qguanri/article/details/51673845



```bash
# 启动
systemctl start firewalld.service
systemctl enable firewalld.service



# 添加指定源地址的80端口
[root@localhost ~]# firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="8.136.182.22" port protocol="tcp" port="80" accept"
success
[root@localhost ~]# firewall-cmd --reload 
success
## 验证
[root@localhost ~]# iptables -vnL IN_public_allow  | grep 182.22
    0     0 ACCEPT     tcp  --  *      *       8.136.182.22         0.0.0.0/0            tcp dpt:80 ctstate NEW
```

