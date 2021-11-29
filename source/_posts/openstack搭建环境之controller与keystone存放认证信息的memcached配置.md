---
title: openstack搭建环境之controller与keystone存放认证信息的memcached配置
date: 2020-12-25 03:40:32
tags:
toc: true
---



用于缓存 openstack 各服务的身份认证令牌信息

本次由于实验环境资源不够，就和mysql部署在同一个主机。

![image-20201225104216105](http://myapp.img.mykernel.cn/image-20201225104216105.png)

<!--more-->



# 部署

```bash
# 安装
yum install memcached -y 

#编辑/etc/sysconfig/memcached归档并完成以下操作：
#配置服务以使用控制器节点的管理IP地址。这是为了使其他节点能够通过管理网络访问：
cat > /etc/sysconfig/memcached <<EOF
PORT="11211"
USER="memcached"
# 最大连接
MAXCONN="1024"
# 内存
CACHESIZE="1024"
# listen 所有地址
OPTIONS="-l 0.0.0.0,::1"
EOF

# 开机启动并启动
systemctl enable memcached.service
systemctl start memcached.service

# 验证端口11211, ss -tnl.
# 验证vip可以通, telnet 172.16.0.248 11211
```



# 查看缓存中的session数据

```bash
# stats items  #列出所有keys
```

```bash
# stats cachedump ID 0  #获得key的值，0表示全部列出

示例：
# stats cachedump 18 0
  ITEM tokens/2097d8da1e0c8b5fd47c7b88bdd90ccf0e048aeaf270259961d5badfea94bb05 [3770 b; 1566702863 s]
  ITEM tokens/02c45c48d752ae26c3c855e721e723328ae3952c927fe0a8ab97e5dc505fee90 [3768 b; 1566702607 s]
  ITEM :1:django.contrib.sessions.cache8br7n9lt6neouxxs3j53zhkx3f9e64gi [3872 b; 1566704690 s]
  END
```

```bash
# get KEY_NAME  #get命令获取指定key的值
# get tokens/cb7458bde28bb5edb7412c6df84a8ce3c65c056da02c3419c04392c7d7dd8459 #获取指定key的值
# get :1:django.contrib.sessions.cache8br7n9lt6neouxxs3j53zhkx3f9e64gi
# get VALUE :1:django.contrib.sessions.cache8br7n9lt6neouxxs3j53zhkx3f9e64gi 1 3872
```



