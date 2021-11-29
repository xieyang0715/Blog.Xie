---
title: 安装配置memcached高可用
date: 2020-11-05 07:42:12
tags: plan
toc: true
---



# 1. 规划

| IP            | 主机名                                             | 角色 | 配置 | VIP          |
| ------------- | -------------------------------------------------- | ---- | ---- | ------------ |
| 172.20.0.162  | chengdu-chenghua-linux48-keepalive-haproxy-0-162   | 主   | 1C1G | 172.20.0.248 |
| 172.20.0.163  | chengdu-chenghua-linux48-keepalive-haproxy-0-163   | 备   | 1C1G |              |
| 172.20.100.51 | cd-ch-linux48-centos-memcached-100-51.magedu.local | 主   | 1C1G |              |
| 172.20.100.52 | cd-ch-linux48-centos-memcached-100-52.magedu.local | 备   | 1C1G |              |

![image-20201109091450953](http://myapp.img.mykernel.cn/image-20201109091450953.png)

haproxy一主一备

memcached是双写，但是一般只有一个memcached在线，另一个只有在其中一个挂掉才会上线。

<!--more-->



# 2. 配置memcached master1 [172.20.100.51]

http://myapp.img.mykernel.cn/install-memcached-128-repcached-221.tar.xz

```bash
#/apps/repcached/bin/memcached -d -m 2048 -p 11211 -u root -c 2048 -x 172.20.100.52 -X 16000
```



# 3. 配置memcached master2 [172.20.100.52]

http://myapp.img.mykernel.cn/install-memcached-128-repcached-221.tar.xz

```bash
#/apps/repcached/bin/memcached -d -m 2048 -p 11211 -u root -c 2048 -x 172.20.100.51 -X 16000
```



# 4. 验证写入

```bash
[root@cd-ch-linux48-centos-memcached-100-51 ~]#  telnet 172.20.100.52 11211
Trying 172.20.100.52...
Connected to 172.20.100.52.
Escape character is '^]'.
set name 3 0 4 # 设置key, 不超时，长度4字节
jack # 写一个4字节
STORED

get name # 获取key
VALUE name 3 4
jack
END

```



# 5. 对接keepalived + haproxy

http://liangcheng.mykernel.cn/2020/10/26/%E5%AE%9E%E7%8E%B0haproxy-keepalived%E9%AB%98%E5%8F%AF%E7%94%A8%E9%9B%86%E7%BE%A4%E8%BD%AC%E5%8F%91/



# 6. 配置haproxy

## 6.1 haproxy 162配置

```bash
global
maxconn 100000
chroot /usr/local/haproxy
stats socket /var/lib/haproxy/haproxy.sock1 mode 600 level admin process 1 # 非交互完成服务器热上线和下线
user haproxy
group haproxy
daemon
#---
nbproc 1   #默认单进程启动
#nbthread 4  #可设置为单进程多线程或者多进程单线程，以及针对进程进程cpu绑定
cpu-map 1 1 # 因为服务器前2个核系统调用
#--
pidfile /var/lib/haproxy/haproxy.pid
log 127.0.0.1 local3 info
defaults
option http-keep-alive
option forwardfor
option redispatch
maxconn 100000
mode http
timeout connect 10s
timeout client 1m
timeout server 30m
listen stats    #启动web监控
  bind-process all
  bind :9999
  stats enable
  stats hide-version
  stats uri /haproxy
  stats realm HAPorxy\Stats\Page
  stats auth admin:123456
  #stats refresh 3s
  stats admin if TRUE
listen http
  bind 172.20.0.248:11211
  mode tcp
  server 172.20.100.51 172.20.100.51:11211  check inter 3s fall 2 rise 5
  server 172.20.100.52 172.20.100.52:11211  check inter 3s fall 2 rise 5 backup
```



## 6.2 haproxy 163配置

```bash
global
maxconn 100000
chroot /usr/local/haproxy
stats socket /var/lib/haproxy/haproxy.sock1 mode 600 level admin process 1 # 非交互完成服务器热上线和下线
user haproxy
group haproxy
daemon
#---
nbproc 1   #默认单进程启动
#nbthread 4  #可设置为单进程多线程或者多进程单线程，以及针对进程进程cpu绑定
cpu-map 1 1 # 因为服务器前2个核系统调用
#--
pidfile /var/lib/haproxy/haproxy.pid
log 127.0.0.1 local3 info
defaults
option http-keep-alive
option forwardfor
option redispatch
maxconn 100000
mode http
timeout connect 10s
timeout client 1m
timeout server 30m
listen stats    #启动web监控
  bind-process all
  bind :9999
  stats enable
  stats hide-version
  stats uri /haproxy
  stats realm HAPorxy\Stats\Page
  stats auth admin:123456
  #stats refresh 3s
  stats admin if TRUE
listen http
  bind 172.20.0.248:11211
  mode tcp
  server 172.20.100.51 172.20.100.51:11211  check inter 3s fall 2 rise 5
  server 172.20.100.52 172.20.100.52:11211  check inter 3s fall 2 rise 5 backup
```



# 7. 验证连接VIP

```bash
[root@cd-ch-linux48-centos-memcached-100-51 ~]#  telnet 172.20.0.248 11211
Trying 172.20.0.248...
Connected to 172.20.0.248.
Escape character is '^]'.
get name
VALUE name 3 4
jack
END


# 连接VIP过程中，如果停止master
[root@cd-ch-linux48-centos-memcached-100-51 ~]# kill -9 16202
[root@cd-ch-linux48-centos-memcached-100-51 ~]#  telnet 172.20.0.248 11211
Trying 172.20.0.248...
Connected to 172.20.0.248.
Escape character is '^]'.
get name
VALUE name 3 4
jack
END
Connection closed by foreign host. # 连接中断


# 重新连接, 可以发现另一个memcached有新连接
[root@cd-ch-linux48-centos-memcached-100-51 ~]#  telnet 172.20.0.248 11211
Trying 172.20.0.248...
Connected to 172.20.0.248.
Escape character is '^]'.

[root@cd-ch-linux48-centos-memcached-100-52 ~]# lsof -ni:11211
COMMAND     PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
memcached 16132 root    7u  IPv4  45763      0t0  TCP 172.20.100.52:memcache->172.20.0.162:64441 (ESTABLISHED)
memcached 16132 root    9u  IPv4  44578      0t0  TCP *:memcache (LISTEN)
memcached 16132 root   10u  IPv6  44579      0t0  TCP *:memcache (LISTEN)
memcached 16132 root   11u  IPv4  44582      0t0  UDP *:memcache 
memcached 16132 root   12u  IPv6  44583      0t0  UDP *:memcache 
[root@cd-ch-linux48-centos-memcached-100-51 ~]#  telnet 172.20.0.248 11211
Trying 172.20.0.248...
Connected to 172.20.0.248.
Escape character is '^]'.
set name 0 0 4 # 写入正常
jack
STORED
```



# 8. 监控

1. 进程和端口, keepalived, haproxy, memcached
2. keepalived角色切换

