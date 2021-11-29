---
title: keepalived编译安装及优化
date: 2020-10-22 09:13:56
tags:
toc: true
---





生成vip, 完成高可用

提升高可用性，冗余机制：

1. 主备，lvs/nginx/haproxy + keepalived
2. 主主：keepalived 双master或多master

避免脑裂

1. ping网关(vrrp script)
2. 心跳单播(vrrp strict)
3. 仲裁盘(vrrp script)
4. 心跳检测线，网线直连

故障转移(抢占和非抢占)

1. 资源切换后，重要的服务不要自动切回，切换后给管理员发通知, 管理员决定是否切回。
2. 如果是负载均衡, 就切回来，压力小。

HA实现 

1. VRRP协议早期解决网关单点风险	
2. 软件层基于vrrp协议实现HA- keepalive

HA集群

- keepalived + haproxy/lvs/nginx

1. keepalived 100个IP, 一般一个keepalive有50个IP。一般另一个keepalived 50个IP。就平衡性能了
2. keepalive + lvs:  KA可以规则管理，检测健康状态。
3. keepalive + haproxy/nginx: 需要vrrp_script检测进程或网关状态，决定是否漂移VIP。

keepalived功能

1. vrrp协议完成漂移vip
2. 原生结合lvs, 生成规则，健康状态检测
3. 基于脚本，影响集群事务，支持nginx/haproxy等服务。(端口或进程不存在，降低优先)
4. 报警

运维keepalived

1. zabbix监控vip
2. 备份keepalived配置, nginx配置至gitlab或文件夹
3. 监控、优化、维护、代码升级

<!--more-->



# 1. 编译安装keepalived

https://keepalived.org/download.html

yum/apt安装的keepalived版本较旧

```bash
yum -y install keepalived
apt install keepalived -y
```



## 1.1 编译安装

### 1.1.1 依赖

centos

```bash
# yum -y  install gcc libnfnetlink-devel libnfnetlink ipvsadm  libnl libnl-devel  libnl3 libnl3-devel   lm_sensors-libs net-snmp-agent-libs net-snmp-libs  openssh-server openssh-clients  openssl openssl-devel automake iproute 
```

ubuntu [google: how to compile keepalived on ubuntu](https://readthedocs.org/projects/keepalived-pqa/downloads/pdf/latest/ )

```bash
# apt install pkg-config curl gcc autoconf automake libssl-dev libnl-3-dev libnl-genl-3-dev libsnmp-dev libnl-route-3-dev libnfnetlink-dev libipset-dev iptables-dev libsnmp-dev -y
```



### 1.1.2 编译

```bash
tar xvf keepalived-2.1.5.tar.gz -C /usr/local/src/
cd /usr/local/src/keepalived-2.1.5/
./configure --prefix=/apps/keepalived --disable-fwmark
make
make install
```

### 1.1.3 配置文件

```bash
install -dv /etc/keepalived/
cp keepalived/etc/keepalived/keepalived.conf /etc/keepalived/
```



### 1.1.4 unit脚本

先编辑脚本, 支持/etc/keepalived/keepalived.conf配置文件

```bash
[root@chengdu-chenghua-linux49-keepalived-0-162 keepalived-2.1.5]# cat keepalived/keepalived.service
[Unit]
Description=LVS and VRRP High Availability Monitor
After=network-online.target syslog.target 
Wants=network-online.target 

[Service]
Type=forking
#PIDFile=/run/keepalived.pid
#KillMode=process # 必须注释，否则停止不了keepalived
EnvironmentFile=-/apps/keepalived/etc/sysconfig/keepalived
ExecStart=/apps/keepalived/sbin/keepalived $KEEPALIVED_OPTIONS -f /etc/keepalived/keepalived.conf 
# -f指定配置文件
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
```

```bash
/bin/cp keepalived/keepalived.service /lib/systemd/system/
```

启动服务

```bash
systemctl daemon-reload 
systemctl enable keepalived
systemctl start keepalived
systemctl status keepalived
```

注意：启动后，ubuntu系统会添加防火墙规则, 需要注释`vrrp_strict`



### 1.1.5 日志查看

```bash
 journalctl  -xe -u keepalived
```

![image-20201022175853426](http://myapp.img.mykernel.cn/image-20201022175853426.png)

![image-20201022175909522](http://myapp.img.mykernel.cn/image-20201022175909522.png)

# 2. keepalive配置优化

注意: }} 

多个}同一行就会报错, 少了参数就会报错

```bash
! Configuration File for keepalived

global_defs {
   notification_email { # 邮件通知, 发给?
	1062670898@qq.com
   }
   notification_email_from 1062670898@qq.com # 邮件来自?
   smtp_server 192.168.200.1 # smtp服务器
   smtp_connect_timeout 30   # smtp超时
   router_id LVS_DEVEL       # 标识机器, 默认主机名
   vrrp_skip_check_adv_addr  # addr与上个报文同路由跳过检查
   vrrp_strict               # 遵守vrrp协议, 1)不能没有vip. 2)不能配置单播 3)vrrp v2版本有ipv6. v1版本不能使用ipv6. 4）ubuntu会添加iptables规则。
   vrrp_garp_interval 0      # arp报文发送延迟. 0没有延迟
   vrrp_gna_interval 0       # na报文发送延迟. 0没有延迟
   vrrp_mcast_group4 224.0.0.18 # 224.0.0.0-239.255.255.255 tcpdump -i eth0 -nn host 224.0.0.18 VIP节点才会发送心跳
   default_interface  eth0      # 默认接口eth0, 所以需要配置主机为eth0接口
   vrrp_iptables                # 不添加iptables规则. rpm安装会添加规则。编译安装则不会
}

vrrp_instance VI_1 { # 定义一个实例. 成为集群的实例, 实例名一样; virtual_route_id 一样。 VI_1可以自定义
    state MASTER     # 角色 MASTER|BACKUP. 角色并不是定义节点角色的必要条件。priority才是。 在配置非抢占模式和延迟抢占时, 需要所有节点使用BACKUP
    interface eth0   # 接口eth0
    virtual_router_id 51          # [0-255] 同一个集群应一样
    priority 100                  # 优先级, vip位于高优先级节点。 (100, 80)
    advert_int 1                  # 心跳检测master/backup是否存在
    authentication {              
        auth_type PASS            
        auth_pass 1111            # PASS认证的密码。同集群应一样
    }
    virtual_ipaddress {           # 可有多个, 与宿主机同一个网段
        192.168.0.248/24 dev eth0 label eth0:0
        192.168.0.249/24 dev eth0 label eth0:0 
    }
}
```



# 3. 配置MASTER/BACKUP

| ip            | 主机名                                    | vrrp_iptables | vrrp_mcast_group4 | vrrp_strict | VIP           | state  | priority | virtual_router_id | vrrp_instance        |
| ------------- | ----------------------------------------- | ------------- | ----------------- | ----------- | ------------- | ------ | -------- | ----------------- | -------------------- |
| 192.168.0.162 | chengdu-chenghua-linux49-keepalived-0-162 | 非编译才添加  | 224.0.0.18        | 默认打开    | 192.168.0.248 | MASTER | 100      | 254               | magedu-master-backup |
| 192.168.0.163 | chengdu-chenghua-linux49-keepalived-0-163 | 非编译才添加  | 224.0.0.18        | 默认打开    | 192.168.0.248 | BACKUP | 80       | 254               | magedu-master-backup |

## 3.1 192.168.0.162配置MASTER

### 3.1.1 编译安装

下载链接：http://myapp.img.mykernel.cn/install-keepalived-215.tar.xz

### 3.1.2 配置

```bash
[root@chengdu-chenghua-linux49-keepalived-0-162 ~]# cat /etc/keepalived/keepalived.conf 
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
   vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

vrrp_instance magedu-master-backup {
    state MASTER
    interface eth0
    virtual_router_id 254
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.0.248/24 dev eth0 label eth0:0
    }
}
```



## 3.2 192.168.0.163配置BACKUP

### 3.2.1 编译安装

下载链接：http://myapp.img.mykernel.cn/install-keepalived-215.tar.xz

### 3.2.2 配置BACKUP

```bash
[root@chengdu-chenghua-linux49-keepalived-0-162 ~]# scp /etc/keepalived/keepalived.conf 192.168.0.163:/etc/keepalived/keepalived.conf 


[root@chengdu-chenghua-linux49-keepalived-0-163 ~]# cat /etc/keepalived/keepalived.conf
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
   vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

vrrp_instance magedu-master-backup {
    state BACKUP
    interface eth0
    virtual_router_id 254
    priority 80
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.0.248/24 dev eth0 label eth0:0
    }
}
```

## 3.3 启动BACKUP

可以观察到vip在backup 163节点

```bash
systemctl enable keepalived
systemctl start keepalived
systemctl status keepalived
```

查看日志

```bash
[root@chengdu-chenghua-linux49-keepalived-0-163 ~]# tailf /var/log/messages 
Oct 23 08:44:05 chengdu-chenghua-linux49-keepalived-0-163 Keepalived_vrrp[17164]: Sending gratuitous ARP on eth0 for 192.168.0.248
Oct 23 08:44:05 chengdu-chenghua-linux49-keepalived-0-163 Keepalived_vrrp[17164]: Sending gratuitous ARP on eth0 for 192.168.0.248
```



查看Ip

```bash
[root@chengdu-chenghua-linux49-keepalived-0-163 ~]# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.163  netmask 255.255.255.0  broadcast 192.168.0.255
        inet6 fe80::20c:29ff:feed:4b4  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:ed:04:b4  txqueuelen 1000  (Ethernet)
        RX packets 1655612  bytes 101763828 (97.0 MiB)
        RX errors 0  dropped 306  overruns 0  frame 0
        TX packets 6497  bytes 687650 (671.5 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0:0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.248  netmask 255.255.255.0  broadcast 0.0.0.0
        ether 00:0c:29:ed:04:b4  txqueuelen 1000  (Ethernet)

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 1  bytes 152 (152.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1  bytes 152 (152.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

```

心跳

```bash
[root@chengdu-chenghua-linux49-keepalived-0-163 ~]# yum -y install tcpdump

[root@chengdu-chenghua-linux49-keepalived-0-163 ~]# tcpdump -i eth0 -nn -vv host 224.0.0.18
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
08:47:42.951729 IP (tos 0xc0, ttl 255, id 51549, offset 0, flags [none], proto VRRP (112), length 48)
    192.168.0.165 > 224.0.0.18: vrrp 192.168.0.165 > 224.0.0.18: VRRPv2, Advertisement, vrid 51, prio 100, authtype none, intvl 1s, length 28, addrs(3): 192.168.200.16,192.168.200.17,192.168.200.18
08:47:43.267405 IP (tos 0xc0, ttl 255, id 219, offset 0, flags [none], proto VRRP (112), length 40)
    192.168.0.163 > 224.0.0.18: vrrp 192.168.0.163 > 224.0.0.18: VRRPv2, Advertisement, vrid 254, prio 80, authtype simple, intvl 1s, length 20, addrs: 192.168.0.248 auth "1111^@^@^@^@"
```



## 3.4 启动MASTER

可以观察到master抢占vip （默认抢占模式）

心跳变成162发送

```bash
[root@chengdu-chenghua-linux49-keepalived-0-162 ~]# systemctl enable --now keepalived
```



心跳

```bash
08:49:42.432785 IP (tos 0xc0, ttl 255, id 338, offset 0, flags [none], proto VRRP (112), length 40)
    192.168.0.163 > 224.0.0.18: vrrp 192.168.0.163 > 224.0.0.18: VRRPv2, Advertisement, vrid 254, prio 80, authtype simple, intvl 1s, length 20, addrs: 192.168.0.248 auth "1111^@^@^@^@"
08:49:42.437692 IP (tos 0xc0, ttl 255, id 1, offset 0, flags [none], proto VRRP (112), length 40)
    192.168.0.162 > 224.0.0.18: vrrp 192.168.0.162 > 224.0.0.18: VRRPv2, Advertisement, vrid 254, prio 100, authtype none, intvl 1s, length 20, addrs: 192.168.0.248
08:49:43.438727 IP (tos 0xc0, ttl 255, id 2, offset 0, flags [none], proto VRRP (112), length 40)
    192.168.0.162 > 224.0.0.18: vrrp 192.168.0.162 > 224.0.0.18: VRRPv2, Advertisement, vrid 254, prio 100, authtype none, intvl 1s, length 20, addrs: 192.168.0.248
08:49:44.438798 IP (tos 0xc0, ttl 255, id 3, offset 0, flags [none], proto VRRP (112), length 40)

```

IP

```bash
[root@chengdu-chenghua-linux49-keepalived-0-162 ~]# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.162  netmask 255.255.255.0  broadcast 192.168.0.255
        inet6 fe80::20c:29ff:fe84:7ee  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:84:07:ee  txqueuelen 1000  (Ethernet)
        RX packets 1693604  bytes 105716254 (100.8 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 10889  bytes 1189029 (1.1 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0:0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.248  netmask 255.255.255.0  broadcast 0.0.0.0
        ether 00:0c:29:84:07:ee  txqueuelen 1000  (Ethernet)

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

```

日志

```bash
[root@chengdu-chenghua-linux49-keepalived-0-162 ~]# vim /var/log/messages
Oct 23 08:49:42 chengdu-chenghua-linux49-keepalived-0-162 Keepalived_vrrp[21582]: (magedu-master-backup) Entering MASTER STATE
Oct 23 08:49:42 chengdu-chenghua-linux49-keepalived-0-162 Keepalived_vrrp[21582]: (magedu-master-backup) setting VIPs.
Oct 23 08:49:42 chengdu-chenghua-linux49-keepalived-0-162 Keepalived_vrrp[21582]: (magedu-master-backup) Sending/queueing gratuitous ARPs on eth0 for 192.168.0.248
Oct 23 08:49:42 chengdu-chenghua-linux49-keepalived-0-162 Keepalived_vrrp[21582]: Sending gratuitous ARP on eth0 for 192.168.0.248
Oct 23 08:49:42 chengdu-chenghua-linux49-keepalived-0-162 Keepalived_vrrp[21582]: Sending gratuitous ARP on eth0 for 192.168.0.248
Oct 23 08:49:42 chengdu-chenghua-linux49-keepalived-0-162 Keepalived_vrrp[21582]: Sending gratuitous ARP on eth0 for 192.168.0.248
Oct 23 08:49:42 chengdu-chenghua-linux49-keepalived-0-162 Keepalived_vrrp[21582]: Sending gratuitous ARP on eth0 for 192.168.0.248
Oct 23 08:49:42 chengdu-chenghua-linux49-keepalived-0-162 Keepalived_vrrp[21582]: Sending gratuitous ARP on eth0 for 192.168.0.248
Oct 23 08:49:47 chengdu-chenghua-linux49-keepalived-0-162 Keepalived_vrrp[21582]: (magedu-master-backup) Sending/queueing gratuitous ARPs on eth0 for 192.168.0.248
Oct 23 08:49:47 chengdu-chenghua-linux49-keepalived-0-162 Keepalived_vrrp[21582]: Sending gratuitous ARP on eth0 for 192.168.0.248
Oct 23 08:49:47 chengdu-chenghua-linux49-keepalived-0-162 Keepalived_vrrp[21582]: Sending gratuitous ARP on eth0 for 192.168.0.248
Oct 23 08:49:47 chengdu-chenghua-linux49-keepalived-0-162 Keepalived_vrrp[21582]: Sending gratuitous ARP on eth0 for 192.168.0.248
Oct 23 08:49:47 chengdu-chenghua-linux49-keepalived-0-162 Keepalived_vrrp[21582]: Sending gratuitous ARP on eth0 for 192.168.0.248
Oct 23 08:49:47 chengdu-chenghua-linux49-keepalived-0-162 Keepalived_vrrp[21582]: Sending gratuitous ARP on eth0 for 192.168.0.248

```



## 3.5 两边安装nginx



### 3.5.1 编译安装Nginx

下载链接: http://myapp.img.mykernel.cn/install-nginx-1.14.2.tar

配置nginx的安装路径

```bash
[root@chengdu-chenghua-linux49-keepalived-0-162 src]# cat install-nginx.sh 
./configure --prefix=/apps/nginx --with-pcre-jit --with-http_ssl_module --with-http_v2_module --with-http_sub_module --with-stream --with-stream_ssl_module  --with-http_realip_module  --with-http_stub_status_module
```

安装...



### 3.5.2 配置和启动

```bash
echo $(hostname) > /apps/nginx/html/index.html 
```

```bash
/apps/nginx/sbin/nginx 
```

### 3.5.3 访问

访问node的IP

```bash
[root@chengdu-chenghua-linux49-keepalived-0-162 ~]# curl 192.168.0.162
chengdu-chenghua-linux49-keepalived-0-162.magedu.local
[root@chengdu-chenghua-linux49-keepalived-0-162 ~]# curl 192.168.0.163
chengdu-chenghua-linux49-keepalived-0-163.magedu.local

```

访问VIP

```bash
[root@chengdu-chenghua-linux49-keepalived-0-163 ~]# while true; do sleep 0.5; curl 192.168.0.248; done
chengdu-chenghua-linux49-keepalived-0-162.magedu.local
chengdu-chenghua-linux49-keepalived-0-162.magedu.local
chengdu-chenghua-linux49-keepalived-0-162.magedu.local
chengdu-chenghua-linux49-keepalived-0-162.magedu.local
chengdu-chenghua-linux49-keepalived-0-162.magedu.local
chengdu-chenghua-linux49-keepalived-0-162.magedu.local
chengdu-chenghua-linux49-keepalived-0-162.magedu.local
chengdu-chenghua-linux49-keepalived-0-162.magedu.local
chengdu-chenghua-linux49-keepalived-0-162.magedu.local

```

### 3.5.4 测试keepalived vip转移 用户无感知

在162主机poweroff

```bash
[root@chengdu-chenghua-linux49-keepalived-0-162 ~]# poweroff

Connection closed by foreign host.

Disconnected from remote host(192.168.0.162) at 09:54:10.

Type `help' to learn how to use Xshell prompt.
[c:\~]$ 
```

请求日志，可以看出没有任何报错

```bash
chengdu-chenghua-linux49-keepalived-0-162.magedu.local
chengdu-chenghua-linux49-keepalived-0-162.magedu.local
chengdu-chenghua-linux49-keepalived-0-163.magedu.local
chengdu-chenghua-linux49-keepalived-0-163.magedu.local
chengdu-chenghua-linux49-keepalived-0-163.magedu.local
```

# 4. 配置单播

| ip            | 主机名                                    | vrrp_iptables | vrrp_mcast_group4 | vrrp_strict    | VIP           | state  | priority | virtual_router_id | unicast_src_ip | unicast_peer  | vrrp_instance        |
| ------------- | ----------------------------------------- | ------------- | ----------------- | -------------- | ------------- | ------ | -------- | ----------------- | -------------- | ------------- | -------------------- |
| 192.168.0.162 | chengdu-chenghua-linux49-keepalived-0-162 | 非编译才添加  | 注释：因为单播    | 注释：因为单播 | 192.168.0.248 | MASTER | 100      | 254               | 192.168.0.162  | 192.168.0.163 | magedu-master-backup |
| 192.168.0.163 | chengdu-chenghua-linux49-keepalived-0-163 | 非编译才添加  | 注释：因为单播    | 注释：因为单播 | 192.168.0.248 | BACKUP | 80       | 254               | 192.168.0.163  | 192.168.0.162 | magedu-master-backup |

减少组播传输量

```bash
global_defs {
	#vrrp_strict                    # vrrp_strict指令强制不能使用单播
	#vrrp_mcast_group4 224.0.0.18   # 注释多播
}
vrrp_instance VI_1 {
    unicast_src_ip <IPADDR>         # 源IP, 本地地址
    unicast_peer {                  # 多个目标IP
        <IPADDR>
        ...
        ...
    }
}
```



## 4.1 192.168.0.162配置单播

```bash
[root@chengdu-chenghua-linux49-keepalived-0-162 ~]# cat /etc/keepalived/keepalived.conf
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
   #vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

vrrp_instance magedu-master-backup {
    state MASTER
    interface eth0
    virtual_router_id 254
    priority 100
    advert_int 1
    unicast_src_ip 192.168.0.162
    unicast_peer {
      192.168.0.163
    }

    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.0.248/24 dev eth0 label eth0:0
    }
}
```

## 4.2 192.168.0.163配置单播

```bash
[root@chengdu-chenghua-linux49-keepalived-0-163 ~]# cat /etc/keepalived/keepalived.conf
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
   #vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
   vrrp_iptables
}

vrrp_instance magedu-master-backup {
    state BACKUP
    interface eth0
    virtual_router_id 254
    priority 80
    advert_int 1
    unicast_src_ip 192.168.0.163
    unicast_peer {
      192.168.0.162
    }

    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.0.248/24 dev eth0 label eth0:0
    }
}

```

### 4.3 生效

```bash
[root@chengdu-chenghua-linux49-keepalived-0-162 ~]# systemctl restart keepalived
[root@chengdu-chenghua-linux49-keepalived-0-163 ~]# systemctl restart keepalived
```

查看VIP

```bash
[root@chengdu-chenghua-linux49-keepalived-0-162 ~]# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.162  netmask 255.255.255.0  broadcast 192.168.0.255
        inet6 fe80::20c:29ff:fe84:7ee  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:84:07:ee  txqueuelen 1000  (Ethernet)
        RX packets 19613  bytes 1188572 (1.1 MiB)
        RX errors 0  dropped 1  overruns 0  frame 0
        TX packets 550  bytes 52249 (51.0 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0:0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.248  netmask 255.255.255.0  broadcast 0.0.0.0
        ether 00:0c:29:84:07:ee  txqueuelen 1000  (Ethernet)

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

心跳

```bash
[root@chengdu-chenghua-linux49-keepalived-0-163 ~]# tcpdump -i eth0 -nn -vv host 192.168.0.162
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
10:13:45.331484 IP (tos 0xc0, ttl 255, id 63, offset 0, flags [none], proto VRRP (112), length 40)
    192.168.0.162 > 192.168.0.163: vrrp 192.168.0.162 > 192.168.0.163: VRRPv2, Advertisement, vrid 254, prio 100, authtype simple, intvl 1s, length 20, addrs: 192.168.0.248 auth "1111^@^@^@^@"
10:13:46.332245 IP (tos 0xc0, ttl 255, id 64, offset 0, flags [none], proto VRRP (112), length 40)
    192.168.0.162 > 192.168.0.163: vrrp 192.168.0.162 > 192.168.0.163: VRRPv2, Advertisement, vrid 254, prio 100, authtype simple, intvl 1s, length 20, addrs: 192.168.0.248 auth "1111^@^@^@^@"

```





# 5. 面试问题：如何让keepalived足够高可用？

1. 多vip
2. 多节点
3. 一个机器3个IP地址

 

## 5.1 配置单master多slave

VIP使用不多情况下

| ip            | 主机名                                    | vrrp_iptables | vrrp_mcast_group4 | vrrp_strict    | VIP                                       | state  | priority | virtual_router_id | unicast_src_ip | unicast_peer                | vrrp_instance        |
| ------------- | ----------------------------------------- | ------------- | ----------------- | -------------- | ----------------------------------------- | ------ | -------- | ----------------- | -------------- | --------------------------- | -------------------- |
| 192.168.0.162 | chengdu-chenghua-linux49-keepalived-0-162 | 非编译才添加  | 注释：因为单播    | 注释：因为单播 | 192.168.0.247 192.168.0.248 192.168.0.249 | MASTER | 100      | 254               | 192.168.0.162  | 192.168.0.163 192.168.0.164 | magedu-master-backup |
| 192.168.0.163 | chengdu-chenghua-linux49-keepalived-0-163 | 非编译才添加  | 注释：因为单播    | 注释：因为单播 | 192.168.0.247 192.168.0.248 192.168.0.249 | BACKUP | 80       | 254               | 192.168.0.163  | 192.168.0.162 192.168.0.164 | magedu-master-backup |
| 192.168.0.164 | chengdu-chenghua-linux49-keepalived-0-164 | 非编译才添加  | 注释：因为单播    | 注释：因为单播 | 192.168.0.247 192.168.0.248 192.168.0.249 | BACKUP | 60       | 254               | 192.168.0.164  | 192.168.0.162 192.168.0.163 | magedu-master-backup |

### 5.1 162 配置master

```bash
[root@chengdu-chenghua-linux49-keepalived-0-162 ~]# cat /etc/keepalived/keepalived.conf
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
   #vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

vrrp_instance magedu-master-backup {
    state MASTER
    interface eth0
    virtual_router_id 254
    priority 100
    advert_int 1
    unicast_src_ip 192.168.0.162
    unicast_peer {
      192.168.0.163
      192.168.0.164 # 添加此行
    }

    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.0.247/24 dev eth0 label eth0:0
        192.168.0.248/24 dev eth0 label eth0:1
        192.168.0.249/24 dev eth0 label eth0:2

    }
}

```

### 5.2 163 配置backup

```bash
[root@chengdu-chenghua-linux49-keepalived-0-163 ~]# cat /etc/keepalived/keepalived.conf
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
   #vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
   vrrp_iptables
}

vrrp_instance magedu-master-backup {
    state BACKUP
    interface eth0
    virtual_router_id 254
    priority 80
    advert_int 1
    unicast_src_ip 192.168.0.163
    unicast_peer {
      192.168.0.162
      192.168.0.164 # 添加此行
    }

    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.0.247/24 dev eth0 label eth0:0
        192.168.0.248/24 dev eth0 label eth0:1
        192.168.0.249/24 dev eth0 label eth0:2

    }
}

```

### 5.3 164配置backup

```bash
[root@chengdu-chenghua-linux49-keepalived-0-164 ~]# cat /etc/keepalived/keepalived.conf
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
   #vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

vrrp_instance magedu-master-backup {
    state BACKUP
    interface eth0
    virtual_router_id 254
    priority 60
    advert_int 1
    unicast_src_ip 192.168.0.164
    unicast_peer {
      192.168.0.162
      192.168.0.163
    }

    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.0.247/24 dev eth0 label eth0:0
        192.168.0.248/24 dev eth0 label eth0:1
        192.168.0.249/24 dev eth0 label eth0:2

    }
}
```

### 5.4 重启

```bash
systemctl restart keepalived
```

查看MASTER 162的VIP

```bash
[root@chengdu-chenghua-linux49-keepalived-0-162 ~]# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.162  netmask 255.255.255.0  broadcast 192.168.0.255
        inet6 fe80::20c:29ff:fe84:7ee  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:84:07:ee  txqueuelen 1000  (Ethernet)
        RX packets 180461  bytes 10861251 (10.3 MiB)
        RX errors 0  dropped 23  overruns 0  frame 0
        TX packets 3984  bytes 301213 (294.1 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0:0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.247  netmask 255.255.255.0  broadcast 0.0.0.0
        ether 00:0c:29:84:07:ee  txqueuelen 1000  (Ethernet)

eth0:1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.248  netmask 255.255.255.0  broadcast 0.0.0.0
        ether 00:0c:29:84:07:ee  txqueuelen 1000  (Ethernet)

eth0:2: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.249  netmask 255.255.255.0  broadcast 0.0.0.0
        ether 00:0c:29:84:07:ee  txqueuelen 1000  (Ethernet)

```


## 5.2 配置多master多slave

vip使用多的情况下

让每个节点运行一些VIP， 负载可以均衡, 一旦一个节点中断, VIP转移就需要运维及时知道, 不然VIP全在某个节点, 压力太大，用户访问慢。

| ip            | 主机名                                    | vrrp_iptables | vrrp_mcast_group4 | vrrp_strict    | VIP                      | state                | priority | virtual_router_id | unicast_src_ip | unicast_peer                | vrrp_instance |
| ------------- | ----------------------------------------- | ------------- | ----------------- | -------------- | ------------------------ | -------------------- | -------- | ----------------- | -------------- | --------------------------- | ------------- |
| 192.168.0.162 | chengdu-chenghua-linux49-keepalived-0-162 | 非编译才添加  | 注释：因为单播    | 注释：因为单播 | 192.168.0.3- 192.168.0.5 | MASTER/BACKUP/BACKUP | 100      | 253               | 192.168.0.162  | 192.168.0.163 192.168.0.164 | magedu-84     |
| 192.168.0.163 | chengdu-chenghua-linux49-keepalived-0-163 | 非编译才添加  | 注释：因为单播    | 注释：因为单播 | 192.168.0.6-192.168.0.8  | BACKUP/MASTER/BACKUP | 80       | 253               | 192.168.0.163  | 192.168.0.162 192.168.0.164 | magedu-84     |
| 192.168.0.164 | chengdu-chenghua-linux49-keepalived-0-164 | 非编译才添加  | 注释：因为单播    | 注释：因为单播 | 192.168.0.9-192.168.0.11 | BACKUP/BACKUP/MASTER | 60       | 253               | 192.168.0.164  | 192.168.0.162 192.168.0.163 | magedu-84     |

注意：以上的单master多slave不动的情况下，我们要新加配置



> 由于实验环境IP范围太小, 仅配置3个IP



### 5.2.1 配置192.168.0.3-5

#### 5.2.1.1 162配置3-5vip_master.conf(100)

```bash
install -dv /etc/keepalived/conf.d/


# 编辑配置文件
include /etc/keepalived/conf.d/*.conf
```



```bash
[root@chengdu-chenghua-linux49-keepalived-0-162 conf.d]# cat 3-5vip_master.conf 
vrrp_instance magedu-84 {
    state MASTER
    interface eth0
    virtual_router_id 253
    priority 100
    advert_int 1
    unicast_src_ip 192.168.0.162
    unicast_peer {
      192.168.0.163
      192.168.0.164
    }

    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.0.3/24 dev eth0 label eth0:5
        192.168.0.4/24 dev eth0 label eth0:6
        192.168.0.5/24 dev eth0 label eth0:7
    }
}
```

```bash
[root@chengdu-chenghua-linux49-keepalived-0-162 conf.d]# systemctl restart keepalived
```



#### 5.2.2 163配置1-10vip_slave.conf(80)

```bash
install -dv /etc/keepalived/conf.d/


# 编辑配置文件
include /etc/keepalived/conf.d/*.conf
```



```bash
[root@chengdu-chenghua-linux49-keepalived-0-162 conf.d]# scp 3-5vip_master.conf 192.168.0.163:/etc/keepalived/conf.d/3-5vip_master.conf 

```



```bash
[root@chengdu-chenghua-linux49-keepalived-0-163 conf.d]# cat 3-5vip_master.conf
vrrp_instance magedu-84 {
    state BACKUP
    interface eth0
    virtual_router_id 253
    priority 80
    advert_int 1
    unicast_src_ip 192.168.0.163
    unicast_peer {
      192.168.0.162
      192.168.0.164
    }

    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.0.3/24 dev eth0 label eth0:5
        192.168.0.4/24 dev eth0 label eth0:6
        192.168.0.5/24 dev eth0 label eth0:7
    }
}

```

```bash
[root@chengdu-chenghua-linux49-keepalived-0-163 conf.d]#  systemctl restart keepalived
```





#### 5.2.3 164配置1-10vip_slave.conf(60)

```bash
install -dv /etc/keepalived/conf.d/


# 编辑配置文件
include /etc/keepalived/conf.d/*.conf
```



```bash
[root@chengdu-chenghua-linux49-keepalived-0-162 ~]# scp /etc/keepalived/keepalived.conf 192.168.0.164:/etc/keepalived/keepalived.conf 

```

```bash
[root@chengdu-chenghua-linux49-keepalived-0-164 conf.d]# cat 3-5vip_master.conf 
vrrp_instance magedu-84 {
    state MASTER
    interface eth0
    virtual_router_id 253
    priority 60
    advert_int 1
    unicast_src_ip 192.168.0.164
    unicast_peer {
      192.168.0.163
      192.168.0.162
    }

    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.0.3/24 dev eth0 label eth0:5
        192.168.0.4/24 dev eth0 label eth0:6
        192.168.0.5/24 dev eth0 label eth0:7
    }
}

```

```bash
[root@chengdu-chenghua-linux49-keepalived-0-164 keepalived]#  systemctl restart keepalived
```

#### 5.2.1.4 验证

在162, 停止162, 转移至163，停止163转移至164

### 5.2.2 配置6-8

route_id

instance_id

#### 5.2.2.1 162配置6-8_backup.conf(80)

state, priority, src_Ip, peer_ip

```bash
vrrp_instance magedu-8 {
    state BACKUP
    interface eth0
    virtual_router_id 252
    priority 80
    advert_int 1
    unicast_src_ip 192.168.0.162
    unicast_peer {
      192.168.0.163
      192.168.0.164
    }

    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.0.6/24 dev eth0 label eth0:8
        192.168.0.7/24 dev eth0 label eth0:9
        192.168.0.8/24 dev eth0 label eth0:10
    }
}

```

vip在本机

#### 5.2.2.2 163配置6-8_master.conf (100)

```bash
[root@chengdu-chenghua-linux49-keepalived-0-162 conf.d]# scp 6-8_backup.conf 192.168.0.163:/etc/keepalived/conf.d/6-8_master.conf

[root@chengdu-chenghua-linux49-keepalived-0-163 conf.d]# cat 6-8_master.conf
vrrp_instance magedu-8 {
    state MASTER # 角色
    interface eth0
    virtual_router_id 252
    priority 100 # 优先级
    advert_int 1
    unicast_src_ip 192.168.0.163 # 源
    unicast_peer { # 目标
      192.168.0.162
      192.168.0.164
    }

    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.0.6/24 dev eth0 label eth0:8
        192.168.0.7/24 dev eth0 label eth0:9
        192.168.0.8/24 dev eth0 label eth0:10
    }
}


```

```bash
[root@chengdu-chenghua-linux49-keepalived-0-163 conf.d]# systemctl restart keepalived

[root@chengdu-chenghua-linux49-keepalived-0-163 conf.d]# ifconfig
# 观察到6-8在本机
```

vip漂移到本机

#### 5.2.2.3 164配置6-8_backup.conf(60)

```bash
[root@chengdu-chenghua-linux49-keepalived-0-162 conf.d]# scp 6-8_backup.conf 192.168.0.164:/etc/keepalived/conf.d/

[root@chengdu-chenghua-linux49-keepalived-0-164 conf.d]# cat 6-8_backup.conf 
vrrp_instance magedu-8 {
    state BACKUP
    interface eth0
    virtual_router_id 252
    priority 60
    advert_int 1
    unicast_src_ip 192.168.0.164
    unicast_peer {
      192.168.0.163
      192.168.0.162
    }

    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.0.6/24 dev eth0 label eth0:8
        192.168.0.7/24 dev eth0 label eth0:9
        192.168.0.8/24 dev eth0 label eth0:10
    }
}

[root@chengdu-chenghua-linux49-keepalived-0-164 conf.d]# systemctl restart keepalived

```



### 5.2.3 配置9-11

instance_id, route_id

#### 5.2.3.1 162配置9-11_backup.conf (60)

```bash
[root@chengdu-chenghua-linux49-keepalived-0-162 conf.d]# cat 9-11_backup.conf
vrrp_instance magedu-11 {
    state BACKUP
    interface eth0
    virtual_router_id 251
    priority 60
    advert_int 1
    unicast_src_ip 192.168.0.162
    unicast_peer {
      192.168.0.163
      192.168.0.164
    }

    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
	192.168.0.9/24 dev eth0 label eth0:11
	192.168.0.10/24 dev eth0 label eth0:12
	192.168.0.11/24 dev eth0 label eth0:13
    }
}
[root@chengdu-chenghua-linux49-keepalived-0-162 conf.d]# systemctl restart keepalived

```

vip在本机

#### 5.2.3.2 163配置9-11_backup.conf(80)

```bash
[root@chengdu-chenghua-linux49-keepalived-0-162 conf.d]# scp 9-11_backup.conf 192.168.0.163:/etc/keepalived/conf.d/

[root@chengdu-chenghua-linux49-keepalived-0-163 conf.d]# cat 9-11_backup.conf 
vrrp_instance magedu-11 {
    state BACKUP
    interface eth0
    virtual_router_id 251
    priority 80 #
    advert_int 1
    unicast_src_ip 192.168.0.163
    unicast_peer {
      192.168.0.162
      192.168.0.164
    }

    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
	192.168.0.9/24 dev eth0 label eth0:11
	192.168.0.10/24 dev eth0 label eth0:12
	192.168.0.11/24 dev eth0 label eth0:13
    }
}

[root@chengdu-chenghua-linux49-keepalived-0-163 conf.d]# systemctl restart keepalived

```

vip在本机

#### 5.2.3.3 164配置9-11_master.conf(100)

```bash
[root@chengdu-chenghua-linux49-keepalived-0-162 conf.d]# scp 9-11_backup.conf 192.168.0.164:/etc/keepalived/conf.d/9-11_master.conf 
[root@chengdu-chenghua-linux49-keepalived-0-164 conf.d]# cat 9-11_master.conf
vrrp_instance magedu-11 {
    state MASTER
    interface eth0
    virtual_router_id 251
    priority 100
    advert_int 1
    unicast_src_ip 192.168.0.164
    unicast_peer {
      192.168.0.163
      192.168.0.162
    }

    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
	192.168.0.9/24 dev eth0 label eth0:11
	192.168.0.10/24 dev eth0 label eth0:12
	192.168.0.11/24 dev eth0 label eth0:13
    }
}
[root@chengdu-chenghua-linux49-keepalived-0-164 conf.d]# systemctl restart keepalived

```

vip在本机



# 6. 在网络不稳定，心跳不稳的场景(有延迟)，怎么配置keepalive? (nopreemt)

默认keepalived是抢占模式，如果有延迟，可能两个节点同时配置VIP的情况发生, 出现脑裂。

需要配置非抢占模式

1. 仅在高优先级的节点加此选项(MASTER节点)
2. state为BACKUP, 由priority决定角色。
3. 高优先级节点恢复时, 不会抢占VIP。 即使另一个低优先级节点挂了，也不会抢占VIP。



| ip           | 主机名                                            | vrrp_iptables | vrrp_mcast_group4 | vrrp_strict    | VIP           | state  | priority | virtual_router_id | unicast_src_ip | unicast_peer  | vrrp_instance        | 非抢占    |
| ------------ | ------------------------------------------------- | ------------- | ----------------- | -------------- | ------------- | ------ | -------- | ----------------- | -------------- | ------------- | -------------------- | --------- |
| 172.20.0.162 | chengdu-chenghua-linux39-keepalived-haproxy-0-162 | 非编译才添加  | 注释：因为单播    | 注释：因为单播 | 192.168.0.248 | BACKUP | 100      | 239               | 192.168.0.162  | 192.168.0.163 | magedu-master-backup | nopreempt |
| 172.20.0.163 | chengdu-chenghua-linux39-keepalived-haproxy-0-163 | 非编译才添加  | 注释：因为单播    | 注释：因为单播 | 192.168.0.248 | BACKUP | 80       | 239               | 192.168.0.163  | 192.168.0.162 | magedu-master-backup |           |

配置前验证默认的抢占是正常的

```bash
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/etc/keepalived# vim keepalived.conf 
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/etc/keepalived# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.20.0.162  netmask 255.255.0.0  broadcast 172.20.255.255
        inet6 fe80::20c:29ff:fe10:8247  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:10:82:47  txqueuelen 1000  (Ethernet)
        RX packets 184675  bytes 61339212 (61.3 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 230462  bytes 22786928 (22.7 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0:0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.20.0.248  netmask 255.255.0.0  broadcast 0.0.0.0
        ether 00:0c:29:10:82:47  txqueuelen 1000  (Ethernet)

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 1402  bytes 122409 (122.4 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1402  bytes 122409 (122.4 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/etc/keepalived# systemctl stop keepalived

# # VIP漂移
root@chengdu-chenghua-linux48-keepalived-haproxy-0-163:~# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.20.0.163  netmask 255.255.0.0  broadcast 172.20.255.255
        inet6 fe80::20c:29ff:feaf:b47a  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:af:b4:7a  txqueuelen 1000  (Ethernet)
        RX packets 245605  bytes 59320349 (59.3 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 158800  bytes 21005118 (21.0 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0:0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.20.0.248  netmask 255.255.0.0  broadcast 0.0.0.0
        ether 00:0c:29:af:b4:7a  txqueuelen 1000  (Ethernet)

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 2345  bytes 3189042 (3.1 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 2345  bytes 3189042 (3.1 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0





root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/etc/keepalived# systemctl start keepalived


# VIP漂移
root@chengdu-chenghua-linux48-keepalived-haproxy-0-163:~# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.20.0.163  netmask 255.255.0.0  broadcast 172.20.255.255
        inet6 fe80::20c:29ff:feaf:b47a  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:af:b4:7a  txqueuelen 1000  (Ethernet)
        RX packets 245659  bytes 59325259 (59.3 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 158857  bytes 21011920 (21.0 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 2345  bytes 3189042 (3.1 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 2345  bytes 3189042 (3.1 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

```



## 6.1 配置162

```bash
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:~# cat /etc/keepalived/keepalived.conf 
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
}

vrrp_instance VI_1 {
    state BACKUP # 必须为BACKUP
    interface eth0
    virtual_router_id 239
    priority 100
	nopreempt # 非抢占
    advert_int 1
    unicast_src_ip 172.20.0.162
    unicast_peer {
      172.20.0.163
    }
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
		172.20.0.248/16 dev eth0 label eth0:0
    }
}


virtual_server 172.20.0.248 80 {      # vip用户入口(至少2个keepalive高可用, 由keepalived生成，可以多个keepalived节点上漂移)  port用户访问的端口
    delay_loop 6 # 检测后端服务器间隔
    lb_algo wrr
    lb_kind DR  
    # 注释相当于RR
    #persistence_timeout 50
    protocol TCP      #协议：一般tcp, 很少dns反向代理

    #sorry_server 192.168.200.200 1358

    real_server 172.20.0.151 80 {
        weight 1
        HTTP_GET {
            url {
              path /index.html ## 哪个URI是固定用于检测，要给开发发邮件并抄送给领导。如果开发忘记提这个URI. 连不到这个URI, 所有后端全部被拆下线。
            }
#            url {
#              path /testurl2/test.jsp
#              digest 640205b7b0fc66c1ea91c463fac6334d
#            }
#            url {
#              path /testurl3/test.jsp
#              digest 640205b7b0fc66c1ea91c463fac6334d
#            }
			# 链接超时
            connect_timeout 3
			# 重试几次
            retry 3
			# 重试前延迟
            delay_before_retry 3
        }
    }
    real_server 172.20.0.152 80 {
        weight 1
        HTTP_GET {
            url {
              path /index.html
            }
#            url {
#              path /testurl2/test.jsp
#              digest 640205b7b0fc66c1ea91c463fac6334d
#            }
#            url {
#              path /testurl3/test.jsp
#              digest 640205b7b0fc66c1ea91c463fac6334d
#            }
			# 链接超时
            connect_timeout 3
			# 重试几次
            retry 3
			# 重试前延迟
            delay_before_retry 3
        }
    }

}

#include /etc/keepalived/conf.d/*.conf

```

## 6.2 验证非抢占

```bash
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/etc/keepalived# systemctl stop keepalived
# 漂走VIP
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/etc/keepalived# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.20.0.162  netmask 255.255.0.0  broadcast 172.20.255.255
        inet6 fe80::20c:29ff:fe10:8247  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:10:82:47  txqueuelen 1000  (Ethernet)
        RX packets 185577  bytes 61424881 (61.4 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 231463  bytes 22907130 (22.9 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 1418  bytes 124145 (124.1 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1418  bytes 124145 (124.1 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# 启动高优先
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/etc/keepalived# systemctl start keepalived

# 连续观察不会抢占VIP
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/etc/keepalived# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.20.0.162  netmask 255.255.0.0  broadcast 172.20.255.255
        inet6 fe80::20c:29ff:fe10:8247  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:10:82:47  txqueuelen 1000  (Ethernet)
        RX packets 185618  bytes 61428063 (61.4 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 231489  bytes 22911266 (22.9 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 1422  bytes 124609 (124.6 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1422  bytes 124609 (124.6 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/etc/keepalived# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.20.0.162  netmask 255.255.0.0  broadcast 172.20.255.255
        inet6 fe80::20c:29ff:fe10:8247  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:10:82:47  txqueuelen 1000  (Ethernet)
        RX packets 185623  bytes 61428455 (61.4 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 231492  bytes 22912640 (22.9 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 1422  bytes 124609 (124.6 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1422  bytes 124609 (124.6 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

```



# 7. master恢复后, 是否立即抢占?(preempt_delay)

在非抢占模式(以上第6点配置基础)之上, 去掉nopreempt, 加上preempt_delay选项即可。

优点：在主节点启动后，等会运行稳定后才进行抢占

## 7.1 配置162 抢占延迟

基于第6节配置基础之上配置。

```bash
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:~# cat /etc/keepalived/keepalived.conf 
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
}

vrrp_instance VI_1 {
    state BACKUP      #
    interface eth0
    virtual_router_id 239
    priority 100
	preempt_delay 10 #
    advert_int 1
    unicast_src_ip 172.20.0.162
    unicast_peer {
      172.20.0.163
    }
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
		172.20.0.248/16 dev eth0 label eth0:0
    }
}


virtual_server 172.20.0.248 80 {      # vip用户入口(至少2个keepalive高可用, 由keepalived生成，可以多个keepalived节点上漂移)  port用户访问的端口
    delay_loop 6 # 检测后端服务器间隔
    lb_algo wrr
    lb_kind DR  
    # 注释相当于RR
    #persistence_timeout 50
    protocol TCP      #协议：一般tcp, 很少dns反向代理

    #sorry_server 192.168.200.200 1358

    real_server 172.20.0.151 80 {
        weight 1
        HTTP_GET {
            url {
              path /index.html ## 哪个URI是固定用于检测，要给开发发邮件并抄送给领导。如果开发忘记提这个URI. 连不到这个URI, 所有后端全部被拆下线。
            }
#            url {
#              path /testurl2/test.jsp
#              digest 640205b7b0fc66c1ea91c463fac6334d
#            }
#            url {
#              path /testurl3/test.jsp
#              digest 640205b7b0fc66c1ea91c463fac6334d
#            }
			# 链接超时
            connect_timeout 3
			# 重试几次
            retry 3
			# 重试前延迟
            delay_before_retry 3
        }
    }
    real_server 172.20.0.152 80 {
        weight 1
        HTTP_GET {
            url {
              path /index.html
            }
#            url {
#              path /testurl2/test.jsp
#              digest 640205b7b0fc66c1ea91c463fac6334d
#            }
#            url {
#              path /testurl3/test.jsp
#              digest 640205b7b0fc66c1ea91c463fac6334d
#            }
			# 链接超时
            connect_timeout 3
			# 重试几次
            retry 3
			# 重试前延迟
            delay_before_retry 3
        }
    }

}

#include /etc/keepalived/conf.d/*.conf

```

## 7.2 测试重启

```bash
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:/etc/keepalived# systemctl restart keepalived

```

查看日志

```bash
Oct 27 18:19:28 chengdu-chenghua-linux48-keepalive-haproxy-0-162 Keepalived_vrrp[29300]: (VI_1) start preempt delay (10.000000)
```



# 8. 在master切换为备用时，keepalived如何及时通知运维？

## 8.1 通知媒介

1. 邮件：可能看不到。
2. 短信：第3方平台，收费。阿里大鱼。
3. 微信：重要业务通知，免费语音：

## 8.2 配置所有keepalived切换角色可以发送报警

### 8.2.1 配置/etc/mail.rc

![image-20201201130622041](http://myapp.img.mykernel.cn/image-20201201130622041.png)![image-20201201130637457](http://myapp.img.mykernel.cn/image-20201201130637457.png)![image-20201201130752382](http://myapp.img.mykernel.cn/image-20201201130752382.png)



编辑/etc/mail.rc, 追加以下信息

```bash
set from="2192383945@qq.com"
set smtp=smtp.qq.com
set smtp-auth-user="2192383945@qq.com"       
set smtp-auth-password="tomujjpxtdegdjag"        
set smtp-auth=login
```
使用mail命令发送信息的方式
> ```bash
> # mail -s "test by mail" 2192383945@qq.com </etc/hosts
> # cat /etc/hosts | mail -s "test by mail" 2192383945@qq.com
> # mail -s "test by mail" 2192383945@qq.com <<EOF
> `cat /etc/hosts`
> EOF
> ```

![image-20201201131040267](http://myapp.img.mykernel.cn/image-20201201131040267.png)



### 8.2.2 配置keepalived



centos安装依赖

```bash
# yum install mailx -y
```

编辑 /etc/keepalived/scripts/notify.sh

```bash
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2020-12-01
#FileName：             /etc/keepalived/scripts/notify.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2020 All rights reserved
#********************************************************************
contact='2192383945@qq.com' # 发送给谁, 在生产使用中, 应该发送给一个组, python脚本读取到组名, 将消息推送给组内的运维人员。将来运维离职方便维护

notify() {
    local mailsubject="$(hostname) to be $1, vip floating"
    local mailbody="$(date +'%F %T'): vrrp transition, $(hostname) changed to be $1"  #时间状态改变
	# 此步骤可以修改为调用python脚本完成微信报警
	echo "$mailbody" | mail -s "$mailsubject" $contact
}

case $1 in
master)
    notify master
    # 当vip迁移到当前主机，可以做的一些事, 
    # 有状态：例如: keepalived + mysql, 启动mysql进程。 keepalived + nfs + rsyncd+inotify, 启动nfs +  rsyncd+inotify。 启动proxysql。
    # 无状态：例如：nginx/haproxy，可以一直运行，则不需要操作。
    ;;
backup)
    notify backup
    # 当vip从当前主机迁移走，可以做的一些事, 
    # 有状态：例如: keepalived + mysql, 停止mysql进程。 keepalived + nfs + rsyncd+inotify, 停止nfs +  rsyncd+inotify。 停止proxysql。
    # 无状态：例如：nginx/haproxy，可以一直运行，则不需要操作。
    ;;
fault)
    notify fault
    ;;
*)
    echo "Usage: $(basename $0) {master|backup|fault}"
    exit 1
    ;;
esac
```



测试功能

```bash
# /etc/keepalived/scripts/notify.sh master
```



![image-20201201131416505](http://myapp.img.mykernel.cn/image-20201201131416505.png)





