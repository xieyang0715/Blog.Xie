---
title: 实现haproxy+keepalived高可用集群转发
date: 2020-10-26 01:32:07
tags: plan
toc: true
---



# 1. 规划集群架构

![image-20201026132509495](http://myapp.img.mykernel.cn/image-20201026132509495.png)

VIP迁移过程中，会有轻微丢包

<!--more-->



| ip            | 主机名                                            | vrrp_iptables | vrrp_mcast_group4 | vrrp_strict    | VIP           | state  | priority | virtual_router_id | unicast_src_ip | unicast_peer  | vrrp_instance        | 代理模式 | 安装位置         |
| ------------- | ------------------------------------------------- | ------------- | ----------------- | -------------- | ------------- | ------ | -------- | ----------------- | -------------- | ------------- | -------------------- | -------- | ---------------- |
| 192.168.0.162 | chengdu-chenghua-linux39-keepalived-haproxy-0-162 | 非编译才添加  | 注释：因为单播    | 注释：因为单播 | 192.168.0.248 | MASTER | 100      | 239               | 192.168.0.162  | 192.168.0.163 | magedu-master-backup | http     | /apps/keepalived |
| 192.168.0.163 | chengdu-chenghua-linux39-keepalived-haproxy-0-163 | 非编译才添加  | 注释：因为单播    | 注释：因为单播 | 192.168.0.248 | BACKUP | 80       | 239               | 192.168.0.163  | 192.168.0.162 | magedu-master-backup | http     | /apps/keepalived |
| ip            | 主机名                                            | 网站路径      | 安装位置          |                |               |        |          |                   |                |               |                      |          |                  |
| 192.168.0.151 | chengdu-chenghua-linux39-nginx-0-151              | /apps/html    | /apps/nginx       |                |               |        |          |                   |                |               |                      |          |                  |
| 192.168.0.152 | chengdu-chenghua-linux39-nginx-0-152              | /apps/html    | /apps/nginx       |                |               |        |          |                   |                |               |                      |          |                  |



# 2.安装后端

## 2.1 安装nginx

http://myapp.img.mykernel.cn/install-nginx-1.16.1-noarch.tar

## 2.2 配置

```bash
root@chengdu-chenghua-linux48-nginx-0-151:~# echo $(hostname) > /apps/nginx/html/index.html
root@chengdu-chenghua-linux48-nginx-0-152:~# echo $(hostname) > /apps/nginx/html/index.html
```

## 2.3 启动并测试

```bash
root@chengdu-chenghua-linux48-nginx-0-151:~# /apps/nginx/sbin/nginx 
root@chengdu-chenghua-linux48-nginx-0-152:~# /apps/nginx/sbin/nginx 
root@chengdu-chenghua-linux48-nginx-0-152:~# curl 172.20.0.151
chengdu-chenghua-linux48-nginx-0-151
root@chengdu-chenghua-linux48-nginx-0-152:~# curl 172.20.0.152
chengdu-chenghua-linux48-nginx-0-152
```



# 3. 安装用户入口

## 3.1 安装keepalived

http://myapp.img.mykernel.cn/keepalived-2.1.5-noarch.tar.xz

### 3.1.1 162配置

```bash
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/etc/keepalived# cat keepalived.conf
! Configuration File for keepalived

global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
   vrrp_garp_interval 0
   vrrp_gna_interval 0
   vrrp_iptables
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 239
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
		172.20.0.248/16 dev eth0 label eth0:0
    }
}

root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/etc/keepalived# systemctl start keepalived
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/etc/keepalived# systemctl enable keepalived
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/etc/keepalived# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.20.0.162  netmask 255.255.0.0  broadcast 172.20.255.255
        inet6 fe80::20c:29ff:fe10:8247  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:10:82:47  txqueuelen 1000  (Ethernet)
        RX packets 29803  bytes 33986068 (33.9 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 13505  bytes 5606588 (5.6 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0:0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.20.0.248  netmask 255.255.0.0  broadcast 0.0.0.0
        ether 00:0c:29:10:82:47  txqueuelen 1000  (Ethernet)

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 171  bytes 15840 (15.8 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 171  bytes 15840 (15.8 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/etc/keepalived# 

```

### 3.1.2 163配置

```bash
# 162复制
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/etc/keepalived# scp keepalived.conf 172.20.0.163:/etc/keepalived/

# 163配置
root@chengdu-chenghua-linux48-keepalived-haproxy-0-163:/etc/keepalived# cat keepalived.conf
! Configuration File for keepalived

global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
   vrrp_garp_interval 0
   vrrp_gna_interval 0
   vrrp_iptables
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 239
    priority 80
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
		172.20.0.248/16 dev eth0 label eth0:0
    }
}

root@chengdu-chenghua-linux48-keepalived-haproxy-0-163:/etc/keepalived# systemctl enable --now keepalived
# 日志
# 心跳
```

### 3.1.3 测试VIP漂移

```bash
# 162停止
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/etc/keepalived# systemctl stop keepalived
# 163存在VIP
root@chengdu-chenghua-linux48-keepalived-haproxy-0-163:/etc/keepalived# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.20.0.163  netmask 255.255.0.0  broadcast 172.20.255.255
        inet6 fe80::20c:29ff:feaf:b47a  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:af:b4:7a  txqueuelen 1000  (Ethernet)
        RX packets 27118  bytes 28412625 (28.4 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 17141  bytes 9978787 (9.9 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0:0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.20.0.248  netmask 255.255.0.0  broadcast 0.0.0.0
        ether 00:0c:29:af:b4:7a  txqueuelen 1000  (Ethernet)

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 509  bytes 3036214 (3.0 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 509  bytes 3036214 (3.0 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0


# 162启动，恢复VIP
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/etc/keepalived# systemctl start keepalived
```



## 3.2 安装haproxy

http://myapp.img.mykernel.cn/install-haproxy-2.0.18.tar



vip在一个主机上，而2个haproxy都需要监听在vip。所以一个Haproxy启不来。 需要内核参数 ip_nonlocal_bind = 1

如果不配置这个参数, 等MASTER出问题, VIP漂到BACKUP, 在启服务，已经晚了, 服务已经断了。

而且，现在监听在一个非本机的IP, 一旦VIP地址过去, 就会由BACKUP上的haproxy接替响应。



###  3.2.1 162配置

```bash
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/usr/local/haproxy# cat /etc/haproxy/haproxy.cfg
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
  bind-process 1
  bind :9999
  stats enable
  stats hide-version
  stats uri /haproxy
  stats realm HAPorxy\Stats\Page
  stats auth admin:123456
  stats admin if TRUE
listen http
  bind 172.20.0.248:80 
  mode http
  server web1 172.20.0.151:80  inter 3s fall 3 rise 5
  server web2 172.20.0.152:80  inter 3s fall 3 rise 5
```

162启动并测试

```bash
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/usr/local/haproxy# curl 172.20.0.151
chengdu-chenghua-linux48-nginx-0-151
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/usr/local/haproxy# curl 172.20.0.152
chengdu-chenghua-linux48-nginx-0-152

# 启动
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/usr/local/haproxy# systemctl restart haproxy
# 测试, 可以看出rr
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/usr/local/haproxy# curl 172.20.0.248
chengdu-chenghua-linux48-nginx-0-151
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/usr/local/haproxy# curl 172.20.0.248
chengdu-chenghua-linux48-nginx-0-152
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/usr/local/haproxy# curl 172.20.0.248
chengdu-chenghua-linux48-nginx-0-151
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/usr/local/haproxy# curl 172.20.0.248
chengdu-chenghua-linux48-nginx-0-152
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/usr/local/haproxy# curl 172.20.0.248
chengdu-chenghua-linux48-nginx-0-152
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/usr/local/haproxy# curl 172.20.0.248
chengdu-chenghua-linux48-nginx-0-151
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/usr/local/haproxy# curl 172.20.0.248
chengdu-chenghua-linux48-nginx-0-152
```

### 3.2.2 163配置

```bash
# 复制
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:~# scp /etc/haproxy/haproxy.cfg 172.20.163:/etc/haproxy

# 启动，停止master keepalived, 测试
root@chengdu-chenghua-linux48-keepalived-haproxy-0-163:/etc/keepalived# systemctl enable --now haproxy
root@chengdu-chenghua-linux48-keepalived-haproxy-0-163:/etc/keepalived# ss -tnl
       172.20.0.248:80  
       
```

### 3.2.3 测试漂移VIP时，用户无感知

```bash
# 先在一个主机while请求vip
root@chengdu-chenghua-linux48-keepalived-haproxy-0-163:~# while true; do sleep 1 ; curl 172.20.0.248; done

# MASTER停止keepalived
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:~# systemctl stop keepalived

# vip漂移至BACKUP
root@chengdu-chenghua-linux48-keepalived-haproxy-0-163:/etc/keepalived# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.20.0.163  netmask 255.255.0.0  broadcast 172.20.255.255
        inet6 fe80::20c:29ff:feaf:b47a  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:af:b4:7a  txqueuelen 1000  (Ethernet)
        RX packets 30238  bytes 28667015 (28.6 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 20848  bytes 10410152 (10.4 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0:0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.20.0.248  netmask 255.255.0.0  broadcast 0.0.0.0
        ether 00:0c:29:af:b4:7a  txqueuelen 1000  (Ethernet)

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 608  bytes 3043983 (3.0 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 608  bytes 3043983 (3.0 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        
# 用户无感知
root@chengdu-chenghua-linux48-keepalived-haproxy-0-163:~# while true; do sleep 1 ; curl 172.20.0.248; done
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-152
^C
root@chengdu-chenghua-linux48-keepalived-haproxy-0-163:~# 
```



# 4. 配置keepalived 检测 haproxy脚本(vrrp script)

**注意：此脚本缺少报警，角色切换时，需要配置报警。**

仅需要配置在MASTER角色。或仅在BACKUP配置。

降低优先，配置负数weight, 返回非0状态。

升高优先，配置正数weight, 返回0状态。



>  VIP转移策略(假如脚本配置在MASTER, weight为负数，返回非0状态)
>
> 1. 仲裁盘, nfs共享挂载至/etc/keepalived/magedu/, master配置，master创建文件。
> 2. ping网关通了, 不动优先级。ping不通。MASTER降低优先。 ping -w 3 -c 3 172.20.0.1。通了返回0，不通返回1。
> 3. 服务进程不存在. killall -0 haproxy(psmic) 有进程返回0.无进程返回1



```bash
# 示例配置仲裁盘, 以下已经有脚本配置
apt install nfs-kernel-server || yum install nfs-utils
#/etc/exports   
/data/keepalived *(rw,no_root_squash)
install -dv /data/keepalived
systemctl enable --now nfs-server

# other
showmount -e
mkdir /etc/keepalived/magedu
mount -t nfs NFS_SERVER:/data/keepalived  /etc/keepalived/magedu

# MASTER检测服务(haproxy/nginx/mysql/dns)有问题，就生成down文件，BACKUP就抢VIP。
MASTER># touch /etc/keepalived/down
```



## 4.1 配置MASTER 162

```bash
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:~# man -M /apps/keepalived/share/man/ keepalived.conf

root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:~# cat /etc/keepalived/keepalived.conf 
! Configuration File for keepalived

# global_defs之外
vrrp_script magedu {

   #降低优先，配置负数weight, 返回非0状态。
   #升高优先，配置正数weight, 返回0状态。
   script "/etc/keepalived/scripts/vrrp_script.sh"

   # 间隔几s执行一次脚本, 检测时间越快, 对业务影响越小
   interval 1

   # 有时脚本会请求后端, 一般给长一点
   timeout 10

   #降低优先，配置负数weight, 返回非0状态。

   #升高优先，配置正数weight, 返回0状态。
   weight -80

   # 5稳妥
   rise 5

   # 注意：时间越短，一旦服务停止, 可以更快的调整优先级，让VIP漂动，对业务影响更低。 一般 fall 2.  配置1不严谨。 如果你们觉得一次失败就迁移也可以。
   fall 2

   # 一般root, 或有高权限普通用户。默认root
   user root
   
   # keepalived默认是失败状态, 即master priority 配置100, 默认失败就是80. 然后在执行脚本返回0就+20=100. 返回负数就-20=80                  
   init_fail

}



global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
   vrrp_garp_interval 0
   vrrp_gna_interval 0
   vrrp_iptables
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 239
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
		172.20.0.248/16 dev eth0 label eth0:0
    }
	# 调用脚本
    track_script {
        #<SCRIPT_NAME>
        #<SCRIPT_NAME> weight <-253..253> [reverse|no_reverse]
		magedu	
	}
}
```

```bash
# 脚本准备, centos需要  yum -y install psmisc
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:~# killall -0 haproxy
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:~# echo $?
0
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:~# systemctl stop haproxy
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:~# killall -0 haproxy
haproxy: no process found
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:~# echo $?
1
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:~# which killall
/usr/bin/killall
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:~# /bin/sh -c '/usr/bin/killall -0 haproxy'
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:~# echo $?
0
# 以上可知，haproxy不存在返回非0状态




root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:~# install -dv /etc/keepalived/scripts/

root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:~# cat /etc/keepalived/scripts/vrrp_script.sh
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2020-10-26
#FileName：             /etc/keepalived/scripts/vrrp_script.sh
#URL:                   http://www.magedu.com
#Description：          A test script
#Copyright (C):        2020 All rights reserved
#********************************************************************
# 此处可以添加先尝试重启。实验就不配置了，好确定VIP漂移，另一个接口可以正常接入用户流量
# ps -ef | grep -v grep | grep -v vrrp_script.sh | grep haproxy
# [ $? -eq 0 ] || systemctl start haproxy
###############
sleep 1
/usr/bin/killall -0  haproxy &> /dev/null

root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:~# chmod +x /etc/keepalived/scripts/vrrp_script.sh 
```



## 4.2 重启keepalived，生效配置

```bash
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:~# systemctl restart keepalived.service 

root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:~# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.20.0.162  netmask 255.255.0.0  broadcast 172.20.255.255
        inet6 fe80::20c:29ff:fe10:8247  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:10:82:47  txqueuelen 1000  (Ethernet)
        RX packets 36244  bytes 34550588 (34.5 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 19508  bytes 6722590 (6.7 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0:0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.20.0.248  netmask 255.255.0.0  broadcast 0.0.0.0
        ether 00:0c:29:10:82:47  txqueuelen 1000  (Ethernet)

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 316  bytes 27639 (27.6 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 316  bytes 27639 (27.6 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0


```



## 4.3 测试停止haproxy, 用户几乎感知不到

```bash
root@chengdu-chenghua-linux48-keepalived-haproxy-0-163:~# while true; do sleep 1 ; curl 172.20.0.248; done



root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:~# systemctl stop haproxy
root@chengdu-chenghua-linux48-keepalived-haproxy-0-163:~# while true; do sleep 1 ; curl 172.20.0.248; done
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-151
curl: (7) Failed to connect to 172.20.0.248 port 80: Connection refused
curl: (7) Failed to connect to 172.20.0.248 port 80: Connection refused
curl: (7) Failed to connect to 172.20.0.248 port 80: Connection refused
curl: (7) Failed to connect to 172.20.0.248 port 80: Connection refused
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-151
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-151

```

用户轻微感知, 连接不到，又恢复。



# 监控

## haproxy

https://github.com/prometheus/haproxy_exporter

监控方式

```bash
HTTP stats URL
Specify custom URLs for the HAProxy stats port using the --haproxy.scrape-uri flag. For example, if you have set stats uri /baz,

haproxy_exporter --haproxy.scrape-uri="http://localhost:5000/baz?stats;csv"
Or to scrape a remote host:

haproxy_exporter --haproxy.scrape-uri="http://haproxy.example.com/haproxy?stats;csv"
Note that the ;csv is mandatory (and needs to be quoted).

If your stats port is protected by basic auth, add the credentials to the scrape URL:

haproxy_exporter  --haproxy.scrape-uri="http://user:pass@haproxy.example.com/haproxy?stats;csv"
You can also scrape HTTPS URLs. Certificate validation is enabled by default, but you can disable it using the --no-haproxy.ssl-verify flag:

haproxy_exporter --no-haproxy.ssl-verify --haproxy.scrape-uri="https://haproxy.example.com/haproxy?stats;csv"
Unix Sockets
As alternative to localhost HTTP a stats socket can be used. Enable the stats socket in HAProxy with for example:

stats socket /run/haproxy/admin.sock mode 660 level admin
The scrape URL uses the 'unix:' scheme:

haproxy_exporter --haproxy.scrape-uri=unix:/run/haproxy/admin.sock


Docker
Docker Repository on Quay Docker Pulls

To run the haproxy exporter as a Docker container, run:

docker run -p 9101:9101 quay.io/prometheus/haproxy-exporter:v0.12.0 --haproxy.scrapeuri="http://user:pass@haproxy.example.com/haproxy?stats;csv"
```

### 安装haproxy-exporter

```bash
/usr/local/haproxy_exporter-0.12.0.linux-amd64/haproxy_exporter  --haproxy.scrape-uri="http://admin:123456@192.168.0.249:9999/haproxy?stats;csv"
```

验证数据，访问http://192.168.0.81:9101/metrics

![image-20210324085158015](http://myapp.img.mykernel.cn/image-20210324085158015.png)

### prometheus监控haproxy

#### 配置target

```bash
  prometheus.yml: |
    global:
      scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
      evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
    # Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
    rule_files:
    - "/etc/rules/*.yml"    # 由于deploy挂载configmap在此路径 
    - "/etc/node-exporter-rules/*.yml" # 加载node相关的通用报警  https://www.cnblogs.com/zhaojiedi1992/p/zhaojiedi_liunx_65_prometheus_alertmanager_rule.html
      # - "second_rules.yml"
    scrape_configs:
    - job_name: 'haproxy-exporter'
      static_configs:
      - targets:
        - "192.168.0.81:9101"
        - "192.168.0.82:9101"
```

#### 配置rules

```bash
  haproxy-exporter-configmap.yml: |
    groups:
    #https://awesome-prometheus-alerts.grep.to/rules#mysql
    - name: haproxyAlert
      rules:
      - alert: HaproxyHighHttp4xxErrorRateBackend
        expr: ((sum by (server) (rate(haproxy_server_http_responses_total{code="4xx"}[1m])) / sum by (proxy) (rate(haproxy_server_http_responses_total[1m]))) * 100) > 5
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: HAProxy high HTTP 4xx error rate backend (instance {{ $labels.instance }})
          description: "Too many HTTP requests with status 4xx (> 5%) on backend {{ $labels.fqdn }}/{{ $labels.backend }}\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: HaproxyHighHttp5xxErrorRateBackend
        expr: ((sum by (server) (rate(haproxy_server_http_responses_total{code="5xx"}[1m])) / sum by (proxy) (rate(haproxy_server_http_responses_total[1m]))) * 100) > 5
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: HAProxy high HTTP 5xx error rate backend (instance {{ $labels.instance }})
          description: "Too many HTTP requests with status 5xx (> 5%) on backend {{ $labels.fqdn }}/{{ $labels.backend }}\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: HaproxyHighHttp4xxErrorRateServer
        expr: ((sum by (server) (rate(haproxy_server_http_responses_total{code="4xx"}[1m])) / sum by (proxy) (rate(haproxy_server_http_responses_total[1m]))) * 100) > 5
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: HAProxy high HTTP 4xx error rate server (instance {{ $labels.instance }})
          description: "Too many HTTP requests with status 4xx (> 5%) on server {{ $labels.server }}\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: HaproxyHighHttp5xxErrorRateServer
        expr: ((sum by (server) (rate(haproxy_server_http_responses_total{code="5xx"}[1m])) / sum by (proxy) (rate(haproxy_server_http_responses_total[1m]))) * 100) > 5
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: HAProxy high HTTP 5xx error rate server (instance {{ $labels.instance }})
          description: "Too many HTTP requests with status 5xx (> 5%) on server {{ $labels.server }}\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: HaproxyServerResponseErrors
        expr: (sum by (server) (rate(haproxy_server_response_errors_total[1m])) / sum by (server) (rate(haproxy_server_http_responses_total[1m]))) * 100 > 5
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: HAProxy server response errors (instance {{ $labels.instance }})
          description: "Too many response errors to {{ $labels.server }} server (> 5%).\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: HaproxyBackendConnectionErrors
        expr: (sum by (proxy) (rate(haproxy_backend_connection_errors_total[1m]))) > 100
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: HAProxy backend connection errors (instance {{ $labels.instance }})
          description: "Too many connection errors to {{ $labels.fqdn }}/{{ $labels.backend }} backend (> 100 req/s). Request throughput may be to high.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: HaproxyServerConnectionErrors
        expr: (sum by (proxy) (rate(haproxy_backend_connection_errors_total[1m]))) > 100
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: HAProxy server connection errors (instance {{ $labels.instance }})
          description: "Too many connection errors to {{ $labels.server }} server (> 100 req/s). Request throughput may be to high.\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"


      - alert: HaproxyPendingRequests
        expr: sum by (proxy) (rate(haproxy_backend_current_queue[2m])) > 0
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: HAProxy pending requests (instance {{ $labels.instance }})
          description: "Some HAProxy requests are pending on {{ $labels.fqdn }}/{{ $labels.backend }} backend\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: HaproxyHttpSlowingDown
        expr: avg by (proxy) (haproxy_backend_max_total_time_seconds) > 1
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: HAProxy HTTP slowing down (instance {{ $labels.instance }})
          description: "Average request time is increasing\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: HaproxyRetryHigh
        expr: sum by (proxy) (rate(haproxy_backend_retry_warnings_total[1m])) > 10
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: HAProxy retry high (instance {{ $labels.instance }})
          description: "High rate of retry on {{ $labels.fqdn }}/{{ $labels.backend }} backend\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: HaproxyProxyDown
        expr: haproxy_backend_up == 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: HAProxy proxy down (instance {{ $labels.instance }})
          description: "HAProxy proxy is down\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: HaproxyServerDown
        expr: haproxy_backend_active_servers == 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: HAProxy server down (instance {{ $labels.instance }})
          description: "HAProxy backend is down\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: HaproxyFrontendSecurityBlockedRequests
        expr: sum by (proxy) (rate(haproxy_frontend_denied_connections_total[2m])) > 10
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: HAProxy frontend security blocked requests (instance {{ $labels.instance }})
          description: "HAProxy is blocking requests for security reason\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: HaproxyServerHealthcheckFailure
        expr: increase(haproxy_server_check_failures_total[1m]) > 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: HAProxy server healthcheck failure (instance {{ $labels.instance }})
          description: "Some server healthcheck are failing on {{ $labels.server }}\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: HaproxyDown
        expr: haproxy_up == 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: HAProxy down (instance {{ $labels.instance }})
          description: "HAProxy down\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"





      - alert: HaproxyBackendMaxActiveSession
        expr: ((sum by (backend) (avg_over_time(haproxy_backend_max_sessions[2m])) / sum by (backend) (avg_over_time(haproxy_backend_limit_sessions[2m]))) * 100) > 80
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: HAProxy backend max active session (instance {{ $labels.instance }})
          description: "HAproxy backend {{ $labels.fqdn }}/{{ $labels.backend }} is reaching session limit (> 80%).\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"


      - alert: HaproxyHttpSlowingDown
        expr: avg by (backend) (haproxy_backend_http_total_time_average_seconds) > 1
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: HAProxy HTTP slowing down (instance {{ $labels.instance }})
          description: "Average request time is increasing\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: HaproxyBackendDown
        expr: haproxy_backend_up == 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: HAProxy backend down (instance {{ $labels.instance }})
          description: "HAProxy backend is down\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

      - alert: HaproxyServerDown
        expr: haproxy_server_up == 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: HAProxy server down (instance {{ $labels.instance }})
          description: "HAProxy server is down\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"

```

#### pod中已经同步到内容

![image-20210324085701410](http://myapp.img.mykernel.cn/image-20210324085701410.png)

![image-20210324085724679](http://myapp.img.mykernel.cn/image-20210324085724679.png)

#### 热重载

```bash
curl --location --request POST 'http://prometheus.youwoyouqu.io/-/reload'
```

#### grafana 2428

![image-20210324100630153](http://myapp.img.mykernel.cn/image-20210324100630153.png)

### 优势 

只要进程通过haproxy统一暴露端点，进程访问不了了，haproxy对应的alert就会报警，这样也变相的直接监控了所有进程。

## keepalived

https://github.com/gen2brain/keepalived_exporter

### 安装keepalived_exporter

```bash
root@haproxy01:~# tar xvf keepalived_exporter-0.5.0-amd64.tar.gz -C /usr/local
root@haproxy01:~# touch /tmp/keepalived.data
root@haproxy01:~# nohup /usr/local/keepalived_exporter-0.5.0-amd64/keepalived_exporter  &
http://192.168.0.81:9650/metrics
http://192.168.0.82:9650/metrics
```

### prometheus 监控 keepalived

#### target

```bash
    scrape_configs:
    - job_name: 'keepalived-exporter'
      static_configs:
      - targets:
        - "192.168.0.81:9650"
        - "192.168.0.82:9650"
```

#### rules

```bash
  keepalived-exporter-configmap.yml: |
    groups:
    - name: keepalivedAlert 
      rules:
      - alert: KeepAliveDown 
        expr:  keepalived_up == 0 
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: keepalived server down (instance {{ $labels.instance }})
          description: "keepalived server is down\n  VALUE = {{ $value }}\n  LABELS: {{ $labels }}"
```

#### 重载配置

容器中已经加载后，重载

```bash
curl --location --request POST 'http://prometheus.youwoyouqu.io/-/reload'
```

#### alert

正常

![image-20210324102105930](http://myapp.img.mykernel.cn/image-20210324102105930.png)

