---

title: 'Ubuntu Server 18.04的安装, 优化系统'
date: 2020-09-29 16:05:48
tags: plan
toc: true
---



# 1. VMware配置安装ubuntu server

## 1.1 新建虚拟机 

文件 -> 新建虚拟机

![image-20200929161103116](http://myapp.img.mykernel.cn/image-20200929161103116.png)

## 1.2 典型安装

![image-20200929161142276](http://myapp.img.mykernel.cn/image-20200929161142276.png)

<!--more-->

## 1.3 稍后安装系统

![image-20200929161201963](http://myapp.img.mykernel.cn/image-20200929161201963.png)

## 1.4 选择Linux, Ubuntu 64

![image-20200929161223721](http://myapp.img.mykernel.cn/image-20200929161223721.png)

## 1.5 配置保存目录

![image-20200929161308969](http://myapp.img.mykernel.cn/image-20200929161308969.png)

## 1.6 配置磁盘

一般虚拟机只存放日志，40-60G即可，重要数据都在外置存储

存放为单个文件, 方便管理

![image-20200929161328145](http://myapp.img.mykernel.cn/image-20200929161328145.png)



## 1.7 自定义硬件

![image-20200929161425362](http://myapp.img.mykernel.cn/image-20200929161425362.png)



## 1.8 配置2C, 2G, 支持虚拟化 *

![image-20200929161519573](http://myapp.img.mykernel.cn/image-20200929161519573.png)



![image-20200929161533563](http://myapp.img.mykernel.cn/image-20200929161533563.png)

## 1.9 选择iso *

D:\迅雷下载\ubuntu-18.04.1-server-amd64.iso

服务器：http://cdimage.ubuntu.com/releases/

桌面：http://releases.ubuntu.com/

镜像的服务器镜像地址：https://mirrors.tuna.tsinghua.edu.cn/ubuntu-cdimage/releases/

![image-20200929161718044](http://myapp.img.mykernel.cn/image-20200929161718044.png)



## 1.10 配置网络 *

桥接：和宿主机在同一个交换机

NAT/仅主机: 宿主机为路由器，虚拟机属于其子网。 NAT可以与外部通信, 仅主机不可以。



![image-20200929161835438](http://myapp.img.mykernel.cn/image-20200929161835438.png)

![image-20200929162003611](http://myapp.img.mykernel.cn/image-20200929162003611.png)



## 1.11 完成硬件配置

![image-20200929162014394](http://myapp.img.mykernel.cn/image-20200929162014394.png)

![image-20200929162050174](http://myapp.img.mykernel.cn/image-20200929162050174.png)

## 1.12 归类模板为一个目录

![image-20200929162120467](http://myapp.img.mykernel.cn/image-20200929162120467.png)

![image-20200929162136041](http://myapp.img.mykernel.cn/image-20200929162136041.png)









# 2. Ubuntu server安装

## 2.1 启动虚拟机

![image-20200929162152173](http://myapp.img.mykernel.cn/image-20200929162152173.png)

![image-20200929162212948](http://myapp.img.mykernel.cn/image-20200929162212948.png)



## 2.2 配置网卡为ethX *

选择英语, F6，在ESC

![image-20200929162259720](http://myapp.img.mykernel.cn/image-20200929162259720.png)



## 2.3 在控制台传递内核参数 *

```bash
net.ifnames=0 biosdevname=0
```

输入完后，直接回车

![image-20200929162458574](http://myapp.img.mykernel.cn/image-20200929162458574.png)



## 2.4 英语

![image-20200929162544973](http://myapp.img.mykernel.cn/image-20200929162544973.png)



## 2.5 时区默认

![image-20200929162603821](http://myapp.img.mykernel.cn/image-20200929162603821.png)



## 2.6 配置键盘

![image-20200929162622941](http://myapp.img.mykernel.cn/image-20200929162622941.png)



选择英语键盘

![image-20200929162701221](http://myapp.img.mykernel.cn/image-20200929162701221.png)

布局为English(US)

![image-20200929162748774](http://myapp.img.mykernel.cn/image-20200929162748774.png)



## 2.7 加载组件

![image-20200929162828440](http://myapp.img.mykernel.cn/image-20200929162828440.png)



## 2.8 配置主机名

默认即可

![image-20200929163055588](http://myapp.img.mykernel.cn/image-20200929163055588.png)



## 2.9 配置普通用户

jack, jack

![image-20200929163111838](http://myapp.img.mykernel.cn/image-20200929163111838.png)

![image-20200929163129702](http://myapp.img.mykernel.cn/image-20200929163129702.png)



![image-20200929163205917](http://myapp.img.mykernel.cn/image-20200929163205917.png)



无论如何都使用弱密码, YES

![image-20200929163308148](http://myapp.img.mykernel.cn/image-20200929163308148.png)



![image-20200929163323301](http://myapp.img.mykernel.cn/image-20200929163323301.png)



## 2.10 自动获取到时区, 可以直接YES

![image-20200929163344471](http://myapp.img.mykernel.cn/image-20200929163344471.png)



## 2.11 手动配置磁盘

![image-20200929163434967](http://myapp.img.mykernel.cn/image-20200929163434967.png)

![image-20200929163446962](http://myapp.img.mykernel.cn/image-20200929163446962.png)

![image-20200929163457145](http://myapp.img.mykernel.cn/image-20200929163457145.png)

![image-20200929163505471](http://myapp.img.mykernel.cn/image-20200929163505471.png)

![image-20200929163516166](http://myapp.img.mykernel.cn/image-20200929163516166.png)

![image-20200929163524230](http://myapp.img.mykernel.cn/image-20200929163524230.png)

![image-20200929163532366](http://myapp.img.mykernel.cn/image-20200929163532366.png)



![image-20200929163659566](http://myapp.img.mykernel.cn/image-20200929163659566.png)

![image-20200929163711029](http://myapp.img.mykernel.cn/image-20200929163711029.png)



![image-20200929163728375](http://myapp.img.mykernel.cn/image-20200929163728375.png)

![image-20200929163607254](http://myapp.img.mykernel.cn/image-20200929163607254.png)



![image-20200929163742854](http://myapp.img.mykernel.cn/image-20200929163742854.png)





![image-20200929163753229](http://myapp.img.mykernel.cn/image-20200929163753229.png)



## 2.12 跳过代理

![image-20200929163902478](http://myapp.img.mykernel.cn/image-20200929163902478.png)

## 2.13 不自动更新

![image-20200929163950373](http://myapp.img.mykernel.cn/image-20200929163950373.png)

## 2.14 openssh-server

![image-20200929164017102](http://myapp.img.mykernel.cn/image-20200929164017102.png)

![image-20200929164039661](http://myapp.img.mykernel.cn/image-20200929164039661.png)

![image-20200929164710684](http://myapp.img.mykernel.cn/image-20200929164710684.png)

## 2.15 安装grub

![image-20200929164732837](http://myapp.img.mykernel.cn/image-20200929164732837.png)



## 2.16 完成安装

![image-20200929164803134](http://myapp.img.mykernel.cn/image-20200929164803134.png)





# 3. Ubuntu server 优化

## 3.1 重启进入控制台

![image-20200929164836342](http://myapp.img.mykernel.cn/image-20200929164836342.png)

## 3.2 以普通用户登陆

jack, 123456

![image-20200929164903375](http://myapp.img.mykernel.cn/image-20200929164903375.png)

## 3.3 openssh-server配置

### 3.3.1 查看网卡地址

```bash
ifconfig
```

以下可以看出地址为 192.168.0.194

![image-20200929165556131](http://myapp.img.mykernel.cn/image-20200929165556131.png)

### 3.3.2 xshell连接ubuntu主机

```bash
ssh 192.168.0.194
```

![image-20200929165720885](http://myapp.img.mykernel.cn/image-20200929165720885.png)

![image-20200929165735096](http://myapp.img.mykernel.cn/image-20200929165735096.png)

![image-20200929165749839](http://myapp.img.mykernel.cn/image-20200929165749839.png)

### 3.3.3 配置ssh服务

```bash
jack@ubuntu:~$ sudo vim /etc/ssh/sshd_config 
[sudo] password for jack: 

```

```bash
Port 22                           # openssh 监听端口, 为安全一般监听在非22号端口
# Authentication:

#LoginGraceTime 2m
#PermitRootLogin prohibit-password
PermitRootLogin yes               # ubuntu默认为 prohibit-password, 不允许root登陆。当为yes时, 即允许root登陆ssh. centos默认为yes.
#StrictModes yes
#MaxAuthTries 6
#MaxSessions 10

# To disable tunneled clear text passwords, change to no here!
PasswordAuthentication yes        # 打开密码认证
#PermitEmptyPasswords no

#UseDNS no
UseDNS no                         # 禁用dns, centos默认会打开

```

重启服务

```bash
sudo systemctl restart sshd
```

重置root密码

```bash
jack@ubuntu:~$ sudo su - root
root@ubuntu:~# echo 'root:123456' | chpasswd 
```



### 3.3.4 xshell连接root用户

```bash
[c:\~]$ ssh root@192.168.0.194
```



## 3.4 配置主机名

```bash
root@ubuntu:~# hostnamectl set-hostname 'ubuntu-template.magedu.local'
```

或

编辑/etc/hostname

```bash
ubuntu-template.magedu.local
```

## 3.5 网卡名称配置

### 3.5.1 验证网卡名

```bash
ifconfig
```

### 3.5.2 配置网卡名为eth0

如果安装过程中未向内核传递, 现在需要向内核传递参数

编辑/etc/default/grub

```bash
GRUB_CMDLINE_LINUX="net.ifnames=0 biosdevname=0"
```



ubuntu更新

```bash
root@ubuntu:~# sudo update-grub
Sourcing file `/etc/default/grub'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-4.15.0-29-generic
Found initrd image: /boot/initrd.img-4.15.0-29-generic
done
```



centos更新

```bash
# grub2-mkconfig -o /boot/grub2/grub.cfg
```



更新后，重启

```bash
reboot
```



## 3.6 网卡配置

Ubuntu从17年10月发版后，已经放弃之前的/etc/network/interfaces里固定IP的配置，而是改成netplan方式，配置文件是: /etc/netplan/01-netcfg.yaml

```bash
# vim /etc/netplan/01-netcfg.yaml 
# This file describes the network interfaces available on your system
# For more information, see netplan(5).
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
      addresses: [192.168.0.194/24]
      gateway4: 192.168.0.1
      nameservers:
        addresses: [223.6.6.6]
~                                       
```

更新配置

```bash
root@ubuntu:~# netplan apply
```





centos

```bash
> /etc/resolv.conf # 清空dns


[root@cd-ch-pxr-centos-0-198 ~]# cat /etc/sysconfig/network-scripts/ifcfg-eth0
TYPE=Ethernet
BOOTPROTO=static
DEVICE=eth0
ONBOOT=yes
IPADDR=192.168.0.198
NETMASK=255.255.255.0
GATEWAY=192.168.0.1
```

确保结果为, 没有search, 不然k8s中会出问题

to prevent Network Manager to overwrite your `resolv.conf` changes, remove the DNS1, DNS2, … lines from `/etc/sysconfig/network-scripts/ifcfg-*`  参考链接: https://ma.ttias.be/centos-7-networkmanager-keeps-overwriting-etcresolv-conf/

```bash
[root@cd-ch-pxr-centos-0-198 ~]# cat /etc/resolv.conf
nameserver 192.168.0.1
nameserver 223.6.6.6

```





## 3.7 安装系统常用命令

ubuntu

```bash
root@ubuntu:~# apt  purge ufw lxd lxd-client lxcfs lxc-common -y

root@ubuntu:~#  apt install iproute2 ntpdate tcpdump telnet traceroute nfs-kernel-server nfs-common lrzsz tree openssl libssl-dev libpcre3 libpcre3-dev zlib1g-dev ntpdate tcpdump telnet traceroute gcc make openssh-server lrzsz tree openssl libssl-dev libpcre3 libpcre3-dev zlib1g-dev ntpdate tcpdump telnet traceroute iotop unzip zip lsof make curl iputils-ping vim net-tools -y 
```

centos

```bash
yum remove -y NetworkManager* firewalld*
yum install -y vim wget tree  lrzsz gcc gcc-c++ automake pcre pcre-devel zlib zlib-devel openssl openssl-devel iproute net-tools iotop ntpdate lsof bzip2
```



## 3.8 系统资源限制优化

编辑/etc/security/limits.conf

```bash
root  soft  core      unlimited
root  hard  core      unlimited
root  soft  nproc     1000000
root  hard  nproc     1000000
root  soft  nofile    1000000
root  hard  nofile    1000000
root  soft  memlock   32000
root  hard  memlock   32000
root  soft  msgqueue  8192000
root  hard  msgqueue  8192000
*     soft  core      unlimited
*     hard  core      unlimited
*     soft  nproc     1000000
*     hard  nproc     1000000
*     soft  nofile    1000000
*     hard  nofile    1000000
*     soft  memlock   32000
*     hard  memlock   32000
*     soft  msgqueue  8192000
*     hard  msgqueue  8192000
```



## 3.9 内核参数优化

编辑/etc/sysctl.conf

```bash
# 1：开启严格的反向路径校验。对每个进来的数据包，校验其反向路径是否是最佳路径。如果反向路径不是最佳路径，则直接丢弃该数据包。
#  减少DDoS攻击,校验数据包的反向路径，如果反向路径不合适，则直接丢弃数据包，避免过多的无效连接消耗系统资源。
#  防止IP Spoofing,校验数据包的反向路径，如果客户端伪造的源IP地址对应的反向路径不在路由表中，或者反向路径不是最佳路径，则直接丢弃数据包，不会向伪造IP的客户端回复响应。
net.ipv4.conf.default.rp_filter = 1
# 监听非本机, 避免切换VIP后在启动服务, 用户就会中断请求。
net.ipv4.ip_nonlocal_bind = 1
# 转发
net.ipv4.ip_forward = 1
#处理无源路由的包
net.ipv4.conf.default.accept_source_route = 0
#关闭sysrq功能
kernel.sysrq = 0
#core文件名中添加pid作为扩展名
kernel.core_uses_pid = 1
# tcp_syncookies是一个开关，是否打开SYN Cookie功能，该功能可以防止部分SYN攻击。tcp_synack_retries和tcp_syn_retries定义SYN的重试次数。
net.ipv4.tcp_syncookies = 1
# docker
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-arptables = 1
# docker运行时，需要设置为1
fs.may_detach_mounts = 1
#修改消息队列长度
kernel.msgmnb = 65536
kernel.msgmax = 65536
#设置最大内存共享段大小bytes
kernel.shmmax = 68719476736
kernel.shmall = 4294967296

net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_sack = 1
net.core.wmem_default = 8388608
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.netdev_max_backlog = 262144
# net.core.somaxconn 是Linux中的一个kernel参数，表示socket监听（listen）的backlog上限。什么是backlog呢？backlog就是socket的监听队列，当一个请求（request）尚未被处理或建立时，他会进入backlog。而socket server可以一次性处理backlog中的所有请求，处理后的请求不再位于监听队列中。当server处理请求较慢，以至于监听队列被填满后，新来的请求会被拒绝。 
net.core.somaxconn = 20480
net.core.optmem_max = 81920
# tcp_max_syn_backlog 进入SYN包的最大请求队列.默认1024.对重负载服务器,增加该值显然有好处.
net.ipv4.tcp_max_syn_backlog = 262144
net.ipv4.tcp_syn_retries = 3
net.ipv4.tcp_retries1 = 3
net.ipv4.tcp_retries2 = 15
# 在使用 iptables 做 nat 时，发现内网机器 ping 某个域名 ping 的通，而使用 curl 测试不通, 原来是 net.ipv4.tcp_timestamps 设置了为 1 ，即启用时间戳
net.ipv4.tcp_timestamps = 0
# tw_reuse 只对客户端起作用，开启后客户端在1s内回收
net.ipv4.tcp_tw_reuse = 1
# recycle 同时对服务端和客户端启作用。如果服务端断开一个NAT用户“可能”会影响。 当TIMEOUT状态过多时，建议启用
net.ipv4.tcp_tw_recycle = 0
net.ipv4.tcp_fin_timeout = 1
# Nginx 之类的中间代理一定要关注这个值，因为它对你的系统起到一个保护的作用，一旦端口全部被占用，服务就异常了。 tcp_max_tw_buckets 能帮你降低这种情况的发生概率，争取补救时间。
net.ipv4.tcp_max_tw_buckets = 20000
# 这个值表示系统所能处理不属于任何进程的socket数量，当我们需要快速建立大量连接时，就需要关注下这个值了。 
net.ipv4.tcp_max_orphans = 327680
# 15. 现大量fin-wait-1
#首先，fin发送之后，有可能会丢弃，那么发送多少次这样的fin包呢？fin包的重传，也会采用退避方式，在2.6.358内核中采用的是指数退避，2s，4s，最后的重试次数是由
net.ipv4.tcp_synack_retries = 1
net.ipv4.tcp_syncookies = 1
# KeepAlive的空闲时长，或者说每次正常发送心跳的周期，默认值为7200s（2小时）
net.ipv4.tcp_keepalive_time = 300
# KeepAlive探测包的发送间隔，默认值为75s
net.ipv4.tcp_keepalive_intvl = 30
# 在tcp_keepalive_time之后，没有接收到对方确认，继续发送保活探测包次数，默认值为9（次）
net.ipv4.tcp_keepalive_probes = 3
# 允许超载使用内存，避免内存快到极限报错
# overcommit_memory 0 redis启动前向内核申请maxmemory, 够ok. 不够error.
# 1  redis启动前向内核申请maxmemory,内核不统计内存是否够用, 允许分配redis使用全部内存。
# 2  redis启动前向内核申请maxmemory,内核允许redis使用全部内存+swap空间内存。
vm.overcommit_memory = 1
#
# 0,内存不足启动oom killer. 1内存不足,kernel panic(系统重启) 或oom. 2. 内存不足, 强制kernel panic. (系统重启) 
vm.panic_on_oom=0
vm.swappiness = 10
#net.ipv4.conf.eth1.rp_filter = 0
#net.ipv4.conf.lo.arp_ignore = 1
#net.ipv4.conf.lo.arp_announce = 2
#net.ipv4.conf.all.arp_ignore = 1
#net.ipv4.conf.all.arp_announce = 2
net.ipv4.tcp_mem = 786432 1048576 1572864
net.ipv4.tcp_rmem = 4096 87380 4194304
net.ipv4.tcp_wmem = 4096 16384 4194304
# 随机端口的范围
net.ipv4.ip_local_port_range = 10001 65000

# inotify监听文件数量
fs.inotify.max_user_watches=89100

# 文件打开数量
# 所有进程 
fs.file-max=52706963
# 单个进程
fs.nr_open=52706963
```
## 3.10 定时任务

```bash
# crontab -e

*/5 * * * * /usr/sbin/ntpdate time1.aliyun.com &> /dev/null
```





## 3.11 禁用firewalld, selinux(centos)

```bash
systemctl disable firewalld
sed -Ei.bak '/SELINUX=/s/(SELINUX=)enforcing/\1disabled/' /etc/selinux/config
```



## 3.12 配置别名(option)

编辑 ~/.bashrc

```bash
# 获取普通配置文件
alias grepconf="grep -v -e '^$' -e '[[:space:]]*#'"
# 获取ini风格的配置文件
alias grepini="grep -v -e '^$' -e '^;'"
```

## 3.13 配置vim特性

编辑 ~/.vimrc

```bash
set nu
set cul
set tabstop=4
set ai
autocmd BufNewFile *.sh exec ":call SetTitle()"
function SetTitle()
        if expand("%:e") == 'sh'
        call setline(1,"#!/bin/bash")
        call setline(2,"#")
        call setline(3,"#********************************************************************")
        call setline(4,"#Author:                songliangcheng")
        call setline(5,"#QQ:                    2192383945")
        call setline(6,"#Date:                  ".strftime("%Y-%m-%d"))
        call setline(7,"#FileName：             ".expand("%"))
        call setline(8,"#URL:                   http://www.magedu.com")
        call setline(9,"#Description：          A test script")
        call setline(10,"#Copyright (C):        ".strftime("%Y")." All rights reserved")
        call setline(11,"#********************************************************************")
        call setline(12,"")
        endif
endfunc
autocmd BufNewFile * normal G

```



## 3.14 重启验证

```bash
# reboot
```

添加xshell选项, 利用重启的时间

![image-20200929174106007](http://myapp.img.mykernel.cn/image-20200929174106007.png)



![image-20200929174116628](http://myapp.img.mykernel.cn/image-20200929174116628.png)





没有search, centos有的话，k8s会出问题

```bash
[root@centos-template ~]# cat /etc/resolv.conf
nameserver 223.6.6.6
```





主机名

```bash
root@ubuntu-template:~# hostname
ubuntu-template.magedu.local
```

网络

```bash
root@ubuntu-template:~# ifconfig 
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.194  netmask 255.255.255.0  broadcast 192.168.0.255
        inet6 fe80::20c:29ff:fe8c:1e19  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:8c:1e:19  txqueuelen 1000  (Ethernet)
        RX packets 969  bytes 151596 (151.5 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 197  bytes 20819 (20.8 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

资源限制

```bash
root@ubuntu-template:~# ulimit -a
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 7644
max locked memory       (kbytes, -l) 16384
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1024
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 7644
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
```

查看内核参数

```bash
root@ubuntu-template:~# cat /etc/sysctl.conf 
```

查看定时任务

```bash
# crontab -l
```






## 3.15 关机

```bash
root@ubuntu-template:~#  poweroff
```





# 4. 快照和克隆主机

## 4.1 快照



![image-20200929174449463](http://myapp.img.mykernel.cn/image-20200929174449463.png)



![image-20200929174501594](http://myapp.img.mykernel.cn/image-20200929174501594.png)



## 4.2 克隆主机



![image-20200929174533922](http://myapp.img.mykernel.cn/image-20200929174533922.png)

![image-20200929174543298](http://myapp.img.mykernel.cn/image-20200929174543298.png)

![image-20200929174551305](http://myapp.img.mykernel.cn/image-20200929174551305.png)

![image-20200929174600553](http://myapp.img.mykernel.cn/image-20200929174600553.png)

主机名为：

![image-20200929174745601](http://myapp.img.mykernel.cn/image-20200929174745601.png)

![image-20200929180055841](http://myapp.img.mykernel.cn/image-20200929180055841.png)

## 4.3 进入虚拟机配置主机名和ip

![image-20200929180319289](http://myapp.img.mykernel.cn/image-20200929180319289.png)

![image-20200929180338042](http://myapp.img.mykernel.cn/image-20200929180338042.png)

## 4.4 关机快照，开机过程中，添加xshell固定选项



![image-20200929180803787](http://myapp.img.mykernel.cn/image-20200929180803787.png)