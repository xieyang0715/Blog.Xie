---
title: 安装配置kvm虚拟机，并创建虚拟机
date: 2020-11-11 06:54:49
tags: 
- plan
- kvm
toc: true
---




# 1. 前提

KVM需要宿主机CPU必须支持虚拟化功能，因此如果是在vmware workstation上使用虚拟机做宿主机，那么必须
要在虚拟机配置界面的处理器选项中开启虚拟机化功能。内存不能太小, 磁盘不能太小。要在其中开多个虚拟主机。

![image-20201111150149725](http://myapp.img.mykernel.cn/image-20201111150149725.png)







linux验证

```bash
# 每核对应一个
# vmx, 是inter, svm是amd.
root@ubuntu-template:~# grep -E "vmx|svm" /proc/cpuinfo | wc -l
2
```

<!--more-->

# 2. 安装kvm工具包

centos

```bash
yum install qemu-kvm qemu-kvm-tools libvirt libvirt-client virt-manager virt-install

systemctl enable --now libvirtd


# 检验是否生成NAT网卡及地址
# ifconfig virbr0
```

ubuntu

```bash
apt install qemu-kvm virt-manager libvirt-daemon-system

# 检验是否支持kvm
root@ubuntu-template:~# kvm-ok 
INFO: /dev/kvm exists
KVM acceleration can be used

```



# 3. 准备磁盘, ISO

```bash
root@ubuntu-template:~# qemu-img create -f qcow2 /var/lib/libvirt/images/centos.qcow2 10G
Formatting '/var/lib/libvirt/images/centos.qcow2', fmt=qcow2 size=10737418240 cluster_size=65536 lazy_refcounts=off refcount_bits=16
```

```bash
root@tomcat1:~# mv CentOS-7-x86_64-Minimal-1908.iso /var/lib/libvirt/images/
```



# 4. 启动并安装虚拟机

## 4.1 启动虚拟机

```bash
root@tomcat1:~# virt-install --virt-type kvm --name centos7 --ram 1024 --vcpus 2 --cdrom=/var/lib/libvirt/images/CentOS-7-x86_64-Minimal-1908.iso --disk path=/var/lib/libvirt/images/centos.qcow2 --network network=default --graphics vnc,listen=0.0.0.0 --noautoconsole --autostart


Starting install...
Domain installation still in progress. You can reconnect to 
the console to complete the installation process.

```

> ```bash
> --virt-type kvm 
> 
> --name centos7
> 
> --ram 1024 
> 
> --vcpus 2 
> 
> --cdrom=/var/lib/libvirt/images/CentOS-7-x86_64-Minimal-1908.iso 
> 
> --disk path=/var/lib/libvirt/images/centos.qcow2 # 稀疏格式
> 
> --network network=default  # 默认网络virbr0网桥下 /etc/libvirt/qemu/networks/default.xml
> 
> --graphics vnc,listen=0.0.0.0 # 默认本地5900 /etc/libvirt/qemu.conf:#remote_display_port_min = 5900
> 
> --noautoconsole   # 不自动附加到终端
> 
> --autostart              # 虚拟机随着系统启动而启动
> ```



## 4.2 连接虚拟机

```bash
root@tomcat1:~# virt-manager # 略
```

![image-20201112145623389](http://myapp.img.mykernel.cn/image-20201112145623389.png)



## 4.3 安装虚拟机

![image-20201112150059958](http://myapp.img.mykernel.cn/image-20201112150059958.png)



# 7. 安装桥接网卡 *

生产环境都使用桥接

生产环境先绑定实现[冗余或带宽提升](http://liangcheng.mykernel.cn/2020/09/29/Ubuntu%E5%8F%8C%E7%BD%91%E5%8D%A1%E7%BB%91%E5%AE%9Abond0-%E5%8F%8C%E7%BD%91%E5%8D%A1%E6%A1%A5%E6%8E%A5/#5-%E7%BD%91%E5%8D%A1%E9%85%8D%E7%BD%AE%E5%B8%A6%E5%AE%BD%E6%8F%90%E5%8D%87%E5%92%8C%E9%AB%98%E5%8F%AF%E7%94%A8)

## 7.1 [配置桥接](http://liangcheng.mykernel.cn/2020/09/29/Ubuntu%E5%8F%8C%E7%BD%91%E5%8D%A1%E7%BB%91%E5%AE%9Abond0-%E5%8F%8C%E7%BD%91%E5%8D%A1%E6%A1%A5%E6%8E%A5/#1-2-%E6%A1%A5%E6%8E%A5bridge%E9%85%8D%E7%BD%AE) 略

启动时网络指定为`--network bridge=br0`

```bash

[root@centos-mq01 ~]# ls -lh /usr/local/src/CentOS-7-x86_64-Minimal-1908.iso 
-rw-r--r--. 1 root root 942M Nov 12 15:35 /usr/local/src/CentOS-7-x86_64-Minimal-1908.iso

[root@centos-mq01 ~]# qemu-img create -f qcow2 /var/lib/libvirt/images/centos7.qcow2 10G 
[root@centos-mq01 ~]# systemctl enable --now libvirtd

[root@centos-mq01 ~]# virt-install --virt-type kvm --name centos7-bridge --ram 1024 --vcpus 2 --cdrom=/usr/local/src/CentOS-7-x86_64-Minimal-1908.iso  --disk path=/var/lib/libvirt/images/centos7.qcow2 --network bridge=br0 --graphics vnc,listen=0.0.0.0 --noautoconsole --autostart

```



## 7.2 vnc连接

> 依次点Option-->Advanced-->Expert找到ColourLevel，默认值是pal8，修改为rgb222或full。

通过安装界面已经获取到桥接地址

![image-20201112165655871](http://myapp.img.mykernel.cn/image-20201112165655871.png)

安装过程略

# 8. 制作基础镜像

在系统安装完成后，启动系统，进行优化系统

```bash
[root@localhost ~]# curl -O http://myapp.img.mykernel.cn/linux_template_install.sh

# 初始化, 避免模板ip与其他ip冲突, 应该在一个固定的ip
[root@localhost ~]# bash linux_template_install.sh --port=22 --allow-root-login=yes --allow-pass-login=yes --root-password=123456 --hostname=chengdu-huayang-centos-template.magedu.local --ipaddr=192.168.0.6 --netmask=255.255.255.0 --gateway=192.168.0.1 --dns=223.6.6.6 --author=songliangcheng --qq=2192383945 --desc="A test toy"

# 关机
poweroff


```





```bash
#创建一个快照
[root@centos-mq01 ~]# virsh snapshot-create centos7-bridge

```



## 8.1 基于快照创建新虚拟机

```bash
#恢复快照
[root@centos-mq01 ~]# virsh snapshot-list centos7-bridge
 Name                 Creation Time             State
------------------------------------------------------------
 1605173204           2020-11-12 17:26:44 +0800 shutoff

[root@centos-mq01 ~]# virsh snapshot-revert centos7-bridge 1605173204
```

```bash
#生成带快照的模板
[root@centos-mq01 ~]# cp /var/lib/libvirt/images/centos7.qcow2 /var/lib/libvirt/images/centos7-template.qcow2
#克隆模板
[root@centos-mq01 ~]# cp /var/lib/libvirt/images/centos7-template.qcow2  /var/lib/libvirt/images/centos7-1.qcow2

#基于镜像启动虚拟机
# --import 导入镜像
# --boot hd  启动项由hd镜像

# 创建主机不带dvd
[root@centos-mq01 ~]# virt-install --virt-type kvm --name centos7-1 --ram 1024 --vcpus 2 --import --disk path=/var/lib/libvirt/images/centos7-1.qcow2 --network bridge=br0 --graphics vnc,listen=0.0.0.0 --noautoconsole --autostart

```

![image-20201112183250439](http://myapp.img.mykernel.cn/image-20201112183250439.png)



## 8. 常用virsh命令

```bash
# virsh --help     #获取帮助
# virsh list --all #列出所有虚拟机
# virsh shutdown CentOS-7-x86_64 #正常关机
# virsh start CentOS-7-x86_64 #正常开机
# virsh destroy centos7 #强制停止/关机
# virsh undefine Win_2008_r2-x86_64 #强制删除
# virsh autostart centos7 #设置当前虚拟机开机自启动
# 快照相关命令
# virsh snapshot-list haproxy-12
# virsh snapshot-revert --snapshotname NewOS haproxy-12 
# virsh snapshot-delete centos7-bridge --snapshotname NewOS
# virsh snapshot-create-as --name NewOS --domain centos7-template
```



# 9. 搭建生产环境的lnmp

![](http://myapp.img.mykernel.cn/linux_architecture.png)







## 9.1 搭建一个拥有br0, br1的kvm

参考: [centos/ubuntu 冗余bond+桥接](http://liangcheng.mykernel.cn/2020/09/29/Ubuntu%E5%8F%8C%E7%BD%91%E5%8D%A1%E7%BB%91%E5%AE%9Abond0-%E5%8F%8C%E7%BD%91%E5%8D%A1%E6%A1%A5%E6%8E%A5/#2-2-centos-ubuntu-%E5%86%97%E4%BD%99bond-%E6%A1%A5%E6%8E%A5)

```bash
# ifconfig
br0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.7  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::20c:29ff:fe5b:b031  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:5b:b0:31  txqueuelen 1000  (Ethernet)
        RX packets 195  bytes 38226 (37.3 KiB)
        RX errors 0  dropped 17  overruns 0  frame 0
        TX packets 60  bytes 7913 (7.7 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

br1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.20.10.2  netmask 255.255.0.0  broadcast 10.20.255.255
        inet6 fe80::20c:29ff:fe5b:b045  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:5b:b0:45  txqueuelen 1000  (Ethernet)
        RX packets 78  bytes 22698 (22.1 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 14  bytes 908 (908.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# 验证bond模式
[root@mgr1 ~]# cat /proc/net/bonding/bond{0,1} | grep 'Bonding Mode'
Bonding Mode: fault-tolerance (active-backup) (fail_over_mac active)
Bonding Mode: fault-tolerance (active-backup) (fail_over_mac active)

```



## 9.2 安装kvm

脚本：http://myapp.img.mykernel.cn/install_kvm.sh

如果需要在kvm虚拟机二次启动虚拟机，就需要打开虚拟化支持

```bash
[root@centos7-iaas ~]# cat /sys/module/kvm_intel/parameters/nested
N

# /etc/modprobe.d/kvm-nested.conf
options kvm-intel nested=1
options kvm-intel enable_shadow_vmcs=1
options kvm-intel enable_apicv=1
options kvm-intel ept=1     

# modprobe -r kvm_intel
# modprobe -a kvm_intel
# cat /sys/module/kvm_intel/parameters/nested
Y
```







## 9.3 创建kvm模板

脚本: http://myapp.img.mykernel.cn/safe_create_kvm.sh

```bash
# ls /usr/local/src/CentOS-7-x86_64-Minimal-1804.iso

# qemu-img create -f qcow2 /var/lib/libvirt/images/centos7-1804-template-1C1G.qcow2 10G 


# 启动模板
# virt-install --virt-type kvm --name centos7-template --ram 512 --vcpus 1 --cdrom=/usr/local/src/CentOS-7-x86_64-Minimal-1804.iso  --disk path=/var/lib/libvirt/images/centos7-1804-template-1C1G.qcow2 --network bridge=br0 --graphics vnc,listen=0.0.0.0 --noautoconsole --autostart

# 赶紧连接，或vnc连接
# virt-manager
```

安装略

```bash
#优化如下
# bash linux_template_install.sh --port=22 --allow-root-login=yes --allow-pass-login=yes --root-password=123456 --hostname=centos7-template.magedu.local --ipaddr=192.168.1.2 --netmask=255.255.255.0 --gateway=192.168.1.1 --dns=223.6.6.6 --author=songliangcheng --qq=2192383945 --desc="A test toy"

# 这一步必须做, 确保ip配置正常
# cat /etc/resolv.conf
nameserver 223.6.6.6

#关机
# poweroff

#取消开机自启
# virsh autostart --disable centos7-template


#快照
# virsh snapshot-create-as --name NewOS --domain centos7-template

```



## 9.4 创建虚拟机

### 9.4.1 haproxy

#### 9.4.1.1 克隆并启动主机

```bash
# pwd
/var/lib/libvirt/images
# cp -a centos7-1804-template-1C1G.qcow2 haproxy-11.qcow2
# cp -a centos7-1804-template-1C1G.qcow2 haproxy-12.qcow2

#启动haproxy11
# virt-install --name haproxy-11  \
--import  --disk path=/var/lib/libvirt/images/haproxy-11.qcow2 \
--virt-type kvm \
--ram 512 \
--vcpus 1 \
--network bridge=br0 --network bridge=br1 \
--graphics vnc,listen=0.0.0.0 \
--noautoconsole --autostart


#启动haproxy12
# virt-install --name haproxy-12  \
--import  --disk path=/var/lib/libvirt/images/haproxy-12.qcow2 \
--virt-type kvm \
--ram 512 \
--vcpus 1 \
--network bridge=br0 --network bridge=br1 \
--graphics vnc,listen=0.0.0.0 \
--noautoconsole --autostart
```

克隆脚本: http://myapp.img.mykernel.cn/safe_clone_kvm.sh

> 基于脚本创建示例: `bash safe_clone_kvm.sh -i /var/lib/libvirt/images/cenots7-1810-template-1C-512M.qcow2 -d /vms/haproxy2-1-2.qcow2 -t kvm -n haproxy2-1-2 -r 512 -v 1 -b br0 -f br1`

```bash
# virt-manager #连接验证网卡和配置
```



![image-20201119162055227](http://myapp.img.mykernel.cn/image-20201119162055227.png)

![image-20201119162110054](http://myapp.img.mykernel.cn/image-20201119162110054.png)



#### 9.4.1.2 配置网络和路由并快照

```bash
# vnc或virt-manager连接上 配置主机名网卡
# echo 'chengdu-huayang-linux48-haproxy1-1-1.magedu.local' > /etc/hostname
#ifcfg-eth0
TYPE=Ethernet
BOOTPROTO=static
DEVICE=eth0
ONBOOT=yes
IPADDR=172.31.1.1
NETMASK=255.255.0.0
GATEWAY=172.31.0.253
#DNS1=192.168.0.1



#ifcfg-eth1
TYPE=Ethernet
BOOTPROTO=static
DEVICE=eth1
ONBOOT=yes
IPADDR=10.20.1.1
NETMASK=255.255.0.0

# cat route-eth1
192.168.100.0/24 via 10.20.0.254 dev eth1

#重启, 过程添加xshell
# reboot

#验证网络通了
ping www.baidu.com -I eth0 # 通
ping www.baidu.com -I eth1 # 不通
#验证没有被networkmanager覆盖
cat /etc/resolv.conf

# 路由还存在
route -n

# 关机 
poweroff

#  virsh snapshot-create-as --name NewOS --domain haproxy1-1-1
Domain snapshot NewOS created



# echo 'chengdu-huayang-linux48-haproxy2-1-2.magedu.local' > /etc/hostname
#ifcfg-eth0
TYPE=Ethernet
BOOTPROTO=static
DEVICE=eth0
ONBOOT=yes
IPADDR=172.31.1.2
NETMASK=255.255.0.0
GATEWAY=172.31.0.253



#ifcfg-eth1
TYPE=Ethernet
BOOTPROTO=static
DEVICE=eth1
ONBOOT=yes
IPADDR=10.20.1.2
NETMASK=255.255.0.0

# cat route-eth1
192.168.100.0/24 via 10.20.0.254 dev eth1

#重启, 过程添加xshell
# reboot

#验证网络通了
ping www.baidu.com -I eth0 # 通
ping www.baidu.com -I eth1 # 不通
#验证没有被networkmanager覆盖
cat /etc/resolv.conf
# 路由还存在
route -n

# 关机 
poweroff

# virsh snapshot-create-as --name NewOS --domain haproxy2-1-2
Domain snapshot NewOS created
```





#### 9.4.1.3 启动虚拟机

```bash
# virsh start haproxy1-1-1
# virsh start haproxy2-1-2
```

现在在xshell中创建haproxy目录, 生成两个haproxy文件

![image-20201119170602458](http://myapp.img.mykernel.cn/image-20201119170602458.png)



#### 9.4.1.4 安装配置2个haproxy+keepalived

略: [实现haproxy+keepalived高可用集群转发](http://liangcheng.mykernel.cn/2020/10/26/%E5%AE%9E%E7%8E%B0haproxy-keepalived%E9%AB%98%E5%8F%AF%E7%94%A8%E9%9B%86%E7%BE%A4%E8%BD%AC%E5%8F%91/)

1. keepalived vip漂移ok
2. haproxy结合httpd配置
3. 请求vip过程中，haproxy停止vip自动漂移, 不影响用户



### 9.4.2 nginx

#### 9.4.2.1 kvm启动虚拟机

> 注意：仅需要内网 br1

```bash
# NAME=nginx1-101-1; bash  safe_clone_kvm.sh -i /var/lib/libvirt/images/cenots7-1810-template-1C-512M.qcow2 -d /vms/${NAME}.qcow2 -t kvm -n ${NAME} -r 512 -v 1 -b br1


# NAME=nginx2-101-2; bash  safe_clone_kvm.sh -i /var/lib/libvirt/images/cenots7-1810-template-1C-512M.qcow2 -d /vms/${NAME}.qcow2 -t kvm -n ${NAME} -r 512 -v 1 -b br1
```

#### 9.4.2.2  配置网络和路由并快照

```bash
#配置nginx1
[root@centos7-template network-scripts]# cat ifcfg-eth0 
TYPE=Ethernet
BOOTPROTO=static
DEVICE=eth0
ONBOOT=yes
IPADDR=10.20.101.1
NETMASK=255.255.0.0
[root@centos7-template network-scripts]# cat route-eth0 
192.168.100.0/24 via 10.20.0.254 dev eth0
# echo 'chengdu-huayang-linux48-nginx1-101-1.magedu.local' > /etc/hostname
# reboot


############ dns, 外网, 路由 
# 不需要打通外网，所以ping不通很正常

# cat /etc/resolv.conf 
nameserver 223.6.6.6

# ping www.baidu.com
ping: www.baidu.com: 未知的名称或服务

# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.20.0.0       0.0.0.0         255.255.0.0     U     0      0        0 eth0
169.254.0.0     0.0.0.0         255.255.0.0     U     1002   0        0 eth0
192.168.100.0   10.20.0.254     255.255.255.0   UG    0      0        0 eth0

# poweroff
# 关机快照

# virsh snapshot-create-as --name NewOS --domain nginx1-101-1



#配置nginx2
[root@centos7-template network-scripts]# echo 'chengdu-huayang-linux48-nginx2-101-2.magedu.local' > /etc/hostname
[root@centos7-template network-scripts]# cat ifcfg-eth0 
TYPE=Ethernet
BOOTPROTO=static
DEVICE=eth0
ONBOOT=yes
IPADDR=10.20.101.2
NETMASK=255.255.0.0
[root@centos7-template network-scripts]# cat route-eth0 
192.168.100.0/24 via 10.20.0.254 dev eth0

#poweroff
# virsh snapshot-create-as --name NewOS --domain nginx2-101-2
```

#### 9.4.2.3 配置后启动虚拟机

```bash
# virsh start nginx2-101-2
# virsh start nginx1-101-1
```

xshell创建目录

![image-20201124102122162](http://myapp.img.mykernel.cn/image-20201124102122162.png)

#### 9.4.2.4 安装配置2个nginx

略: [nginx编译安装及配置优化](http://liangcheng.mykernel.cn/2020/10/14/nginx%E7%BC%96%E8%AF%91%E5%AE%89%E8%A3%85%E5%8F%8A%E9%85%8D%E7%BD%AE%E4%BC%98%E5%8C%96/)




#### 9.4.2.5 haproxy指向这2个nginx

```bash
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
  bind 172.27.0.248:80 # VIP
  mode tcp
  server 10.21.101.1  10.21.101.1:80  check port 80 inter 3s fall 3 rise 5
  server 10.21.101.2  10.21.101.2:80  check port 80 inter 3s fall 3 rise 5
```

#### 9.4.2.6 在一个主机请求vip地址, 测试宕机一个haproxy和nginx对业务几乎无影响

![lnmp](http://myapp.img.mykernel.cn/lnmp.gif)

问simon老师。

nginx -s reload你觉得对用户有影响没有？

其实是有的。只是把影响降到了最低。reload一次，nginx的主进程重载配置文件，同时worker进程也会重载

**高可用并不是100%可用，他们的目的都是把影响降到最低。服务之间迁移依靠集群之间探测，探测的那段时间都是不可用状态。然后集群状态会很快切好**

reload我们之前测过，用户已经建立的连接都断了，只是断了后很快恢复，继续可以建立连接。中间不超过1s

Worker进程的pid号都变了，你可以看看

### 9.4.3 php

编译安装脚本: http://myapp.img.mykernel.cn/php.tar.gz

#### 9.4.3.1 nginx配置反代至php-fpm

```nginx
http {
    server {
        listen       80;
        server_name  localhost;
        
        
        index  index.php index.html index.htm; # index.php
        
        
        location ~ \.php$ { # php结尾
            root           /apps/nginx/html; # php主机上的文档位置
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name; #引用root
            include        fastcgi_params;
        }
        location ~ /\.ht {
            deny  all;
        }
    }
}

```

同步至另一个nginx

```bash
# scp nginx.conf 10.21.101.2:/apps/nginx/conf/
# /apps/nginx/sbin/nginx  -t
# /apps/nginx/sbin/nginx  -s reload

```

网页访问: http://172.27.0.248/,  phpinfo页面; 可以查询mysql, mysql模块均已经加载



### 9.4.4 mysql

#### 9.4.4.1 准备虚拟机

```bash
# NAME=mysql1-101-11; bash  safe_clone_kvm.sh -i /VMs/template/centos7-1804-1C1G.qcow2 -d /VMs/mysql/${NAME}.qcow2 -t kvm -n ${NAME} -r 1024 -v 1 -b vmnet2

# NAME=mysql22-101-12; bash  safe_clone_kvm.sh -i /VMs/template/centos7-1804-1C1G.qcow2 -d /VMs/mysql/${NAME}.qcow2 -t kvm -n ${NAME} -r 1024 -v 1 -b vmnet2


# 安装服务
# 参考：https://www.jianshu.com/p/9c01300d8c9e
```

![mysql-master-slave](http://myapp.img.mykernel.cn/mysql-master-slave.gif)

注意：flush tables with read lock 公当前连接有效，root用户在次登陆无效。在登陆slave时，一定是非root用户。

#### 9.4.4.2  配置master

id, 二进制日志

```bash
[mysqld]
datadir=/data/mysql
socket=/data/mysql/mysql.sock
symbolic-links=0
# 节点ID，确保唯一
server-id = 1        
#控制数据库的binlog刷到磁盘上去 , 0 不控制，性能最好，1每次事物提交都会刷到日志文件中，性能最差，最安全
# 一般是开发连接mysql, 都会在事务内提交, 所以这个禁用后，减少binlog刷写到日志文件
sync_binlog=1
#autocommit=0 
#开启mysql的binlog日志功能
log-bin = mysql-bin     
binlog_format = mixed   #binlog日志格式，mysql默认采用statement，建议使用mixed
expire_logs_days = 7                           #binlog过期清理时间
max_binlog_size = 100m                    #binlog每个日志文件大小
binlog_cache_size = 4m                        #binlog缓存大小
max_binlog_cache_size= 512m              #最大binlog缓存大
auto-increment-offset = 1     # 自增值的偏移量
auto-increment-increment = 1  # 自增值的自增量
slave-skip-errors = all #跳过从库错误
# 慢查询日志文件
slow_query_log=ON
log_output=FILE
long_query_time=3
slow_query_log_file=/data/mysql/slow.log
log_queries_not_using_indexes=ON
# 错误日志
log_error=/data/mysql/error.log
skip_name_resolve = ON
innodb_file_per_table = ON
innodb-flush-log-at-trx-commit=1
# 主节点的binlog已经复制到哪个位置，从节点会对其记录。
sync_master_info=1
sync_relay_log_info=1
#open_files_limit =  65535
[mysqld_safe]
log-error=/data/mysql/mariadb.log
pid-file=/data/mysql/mariadb.pid
!includedir /etc/my.cnf.d
```

> 慢日志
>
> ```bash
> MariaDB [(none)]> show variables where variable_name rlike 'log_slow_queries|slow_query_log|log_output|long_query_time|log_queries_not_using_indexes|log_throttle_queries_not_using_indexes';
> +-------------------------------+----------------------+
> | Variable_name                 | Value                |
> +-------------------------------+----------------------+
> | log_output                    | FILE                 | 输出至 文件。TABLE时记录至mysql.general
> | log_queries_not_using_indexes | OFF                  | 不使用索引是否记录慢日志
> | long_query_time               | 3.000000             | 查询时间阈值
> | slow_query_log                | ON                   | 开启慢日志
> | slow_query_log_file           | /data/mysql/slow.log | 日志文件位置，
> +-------------------------------+----------------------+
> 
> # log_slow_queries|slow_query_log 启用
> 
> ```
>





#### 9.4.4.3  配置slave

配置id, 中继日志

只读？二进制日志

```bash
[mysqld]
datadir=/data/mysql
socket=/data/mysql/mysql.sock
symbolic-links=0
###########################################################
# 节点ID，确保唯一
server-id = 2
# IO -> relay-log ->  SQL: 执行relay-log
relay-log = mysql-relay-bin          
# 提升主使用
# 开启二进制日志功能，以备Slave作为其它Slave的Master时使用
log-bin=mysql-bin
# 只读
read_only = 1      
###########################################################



#控制数据库的binlog刷到磁盘上去 , 0 不控制，性能最好，1每次事物提交都会刷到日志文件中，性能最差，最安全
# 一般是开发连接mysql, 都会在事务内提交, 所以这个禁用后，减少binlog刷写到日志文件
sync_binlog=1
#autocommit=0 
#开启mysql的binlog日志功能
binlog_format = mixed   #binlog日志格式，mysql默认采用statement，建议使用mixed
expire_logs_days = 7                           #binlog过期清理时间
max_binlog_size = 100m                    #binlog每个日志文件大小
binlog_cache_size = 4m                        #binlog缓存大小
max_binlog_cache_size= 512m              #最大binlog缓存大
auto-increment-offset = 1     # 自增值的偏移量
auto-increment-increment = 1  # 自增值的自增量
slave-skip-errors = all #跳过从库错误
# 慢查询日志文件
slow_query_log=ON
log_output=FILE
long_query_time=3
slow_query_log_file=/data/mysql/slow.log
log_queries_not_using_indexes=ON
# 错误日志
log_error=/data/mysql/error.log
skip_name_resolve = ON
innodb_file_per_table = ON
innodb-flush-log-at-trx-commit=1
# 主节点的binlog已经复制到哪个位置，从节点会对其记录。
sync_master_info=1
sync_relay_log_info=1
#open_files_limit =  65535
[mysqld_safe]
log-error=/data/mysql/mariadb.log
pid-file=/data/mysql/mariadb.pid
!includedir /etc/my.cnf.d
```

### 9.4.5 nfs

#### 9.4.5.1 准备虚拟机

```bash
# NAME=nfs1-101-1; bash  safe_clone_kvm.sh -i /VMs/template/centos7-1804-1C1G.qcow2 -d /VMs/mysql/${NAME}.qcow2 -t kvm -n ${NAME} -r 1024 -v 1 -b vmnet2

# NAME=nfs2-101-2; bash  safe_clone_kvm.sh -i /VMs/template/centos7-1804-1C1G.qcow2 -d /VMs/mysql/${NAME}.qcow2 -t kvm -n ${NAME} -r 1024 -v 1 -b vmnet2

```



![image-20201201133053621](http://myapp.img.mykernel.cn/image-20201201133053621.png)



#### 9.4.5.2 配置rsync + inotify 

##### 9.4.5.2.1 安装rsync

```bash
yum -y install epel-release
yum -y install rsync inotify-tools
 cp /etc/rsyncd.conf /etc/rsyncd.conf_bak
 >/etc/rsyncd.conf
echo 'uid = root
gid = root
use chroot = 0
port = 873
hosts allow = 192.168.101.0/24  #允许ip访问设置，可以指定ip或ip段
max connections = 0
timeout = 300
pid file = /var/run/rsyncd.pid
lock file = /var/run/rsyncd.lock
log file = /var/log/rsyncd.log
log format = %t %a %m %f %b
transfer logging = yes
syslog facility = local3
# 模块名而已, 对端依据这个来传输文件. 如果当前是master, 就配置read only=yes, 不允许其他向本机同步 
[slave]
path = /data/nfs
comment = master
ignore errors
read only = no   #是否允许客户端上传文件
list = no
auth users = rsync  #指定由空格或逗号分隔的用户名列表，只有这些用户才允许连接该模块
secrets file = /etc/rsyncd.passwd  #保存密码和用户名文件，需要自己生成
read only=no # 是否允许其他rsync 向本机同步' | sed 's@#.*$@@g' > /etc/rsyncd.conf
useradd -s /sbin/nologin rsync
cat > /etc/rsyncd.passwd <<'EOF'
rsync:123456
EOF
cat > /opt/rsyncd.passwd << 'EOF'
123456
EOF
chmod 600 /etc/rsyncd.passwd
chmod 600 /opt/rsyncd.passwd
systemctl enable rsyncd --now
```

地址：http://myapp.img.mykernel.cn/install-rsync-inotify-1.1.sh

注意，具体怎么配置查看脚本。

测试同步

```bash
# rsync -avzpP --delete /data/nfs/ rsync@192.168.101.1::slave_web --password-file=/opt/rsyncd.passwd


a recursive, symlinks,permissions,modification times,group,owner,device files,special files
v verbose
z compress
P  show progress during transfer, 
--delete delete extraneous files from destination dirs
/data/nfs/ 当前主机的目录
rsync 用户
192.168.101.1::slave_web 对方IP和对方配置的模块
--password-file=/opt/rsyncd.passwd 当前主机保存对方密码的文件
```

##### 9.4.5.2.2 准备inotify脚本

监听/data/nfs

目标主机的模块是slave

目标主机的认证密码存储在/opt/rsyncd.passwd

以`recursive, symlinks,permissions,modification times,group,owner,device files,special files,verbose,compress, show progress during transfer,,delete extraneous files from destination dirs`这些特性完成rsync同步。

编辑/etc/keepalived/scripts/inotify_rsync.sh文件

```bash
/usr/bin/inotifywait -mrq  /data/nfs/  --format "%w%f event %e"  -e modify,delete,create,attrib | while read line; do
        FILEPATH=$(echo $line | awk '{print $1}')
        EVENT=$(echo $line | awk '{print $3}')

		/usr/bin/rsync -avzpP --delete  /data/nfs/ rsync@192.168.101.2::slave --password-file=/opt/rsyncd.passwd
done  &> /tmp/inotify.log
```

#### 9.4.5.3 配置nfs

```bash
yum -y install nfs-utils
mkdir -pv /data/nfs
echo '/data/nfs *(rw,sync,no_root_squash)' > /etc/exports
exportfs -arv
systemctl enable nfs --now
rpcinfo -p localhost
showmount -e
ls /data/nfs/
```

测试挂载

```bash
mount -t nfs ip:/data/nfs /mnt
touch /mnt/123
ls /data/nfs # nfs server
umount /mnt
```



#### 9.4.5.4 安装配置keepalived

脚本：http://myapp.img.mykernel.cn/keepalived-2.1.5-noarch.tar.xz

##### 9.4.5.4.1 master配置

编辑 /etc/keepalived/keepalived.conf

```bash
! Configuration File for keepalived
vrrp_script magedu {
   script "/etc/keepalived/scripts/vrrp_script.sh"
   interval 1
   timeout 10
   weight -30 # 注意, 可以看上面的图理解
   rise 5
   fall 2
   user root
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
   vrrp_strict
   vrrp_iptables
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 23
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.101.9/24 dev eth0 label eth0:0
    }
    track_script {
		magedu	
	}
      notify_master "/etc/keepalived/scripts/notify.sh master"
      notify_backup "/etc/keepalived/scripts/notify.sh backup"
      notify_fault "/etc/keepalived/scripts/notify.sh fault"
}
```

配置状态转换邮件通知和管理有状态服务, 编辑 /etc/keepalived/scripts/notify.sh

```bash
contact='2192383945@qq.com' 
notify() {
    local mailsubject="$(hostname) to be $1, vip floating"
    local mailbody="$(date +'%F %T'): vrrp transition, $(hostname) changed to be $1"  
	echo "$mailbody" | mail -s "$mailsubject" $contact
}
case $1 in
master)
    notify master
		ps -ef | grep  rsync_inotify   | grep -vq grep   ||	 nohup /opt/inotify_rsync.sh	& 
    ;;
backup)
    notify backup
	 	 ps -ef | grep inotifywait | awk '{printf "kill -9 %s\n",$2}' | bash 2> /dev/null
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

编辑/etc/keepalived/scripts/vrrp_script.sh

```bash
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2020-11-30
#FileName：             /etc/keepalived/scripts/vrrp_script.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2020 All rights reserved
#********************************************************************


PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin

ps -ef | grep nfsd | grep -vq grep
# $? 负数时会降低优先级

```



##### 9.4.5.4.2 backup配置

编辑 /etc/keepalived/keepalived.conf

```bash
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
   vrrp_iptables
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 23
    priority 80
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.101.9/24 dev eth0 label eth0:0
    }
      notify_master "/etc/keepalived/scripts/notify.sh master"
      notify_backup "/etc/keepalived/scripts/notify.sh backup"
      notify_fault "/etc/keepalived/scripts/notify.sh fault"
}
```

backup没有配置检测权重, 但是一定要配置通知

```bash
contact='2192383945@qq.com' 
notify() {
    local mailsubject="$(hostname) to be $1, vip floating"
    local mailbody="$(date +'%F %T'): vrrp transition, $(hostname) changed to be $1"  
	echo "$mailbody" | mail -s "$mailsubject" $contact
}
case $1 in
master)
    notify master
	ps -ef | grep  rsync_inotify   | grep -vq grep   ||	 nohup /opt/inotify_rsync.sh	& 
    ;;
backup)
    notify backup
	 ps -ef | grep inotifywait | awk '{printf "kill -9 %s\n",$2}' | bash 2> /dev/null
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

#### 9.4.5.5 验证

一台主机挂载, 停止master的nfs，不会有影响。

```bash
yum -y install nfs-utils
showmount -e 192.168.101.9
mount -t nfs 192.168.101.9:/data/k8s_storage /mnt
touch /mnt/1111

# 过程中停止master主机的nfs, inotify会停止, vip迁移, 另一台主机的vip会启动。inotify会启动。
touch /mnt/11112
umount /mnt
```

### 9.4.6 内网haproxy+keepalived

暴露各个内网服务, nginx, php, mysql, 方便统一管理

#### 9.4.6.1 安装配置虚拟机

```bash
# NAME=haproxy1-101-71; bash  safe_clone_kvm.sh -i /VMs/template/centos7-1804-1C1G.qcow2 -d /VMs/mysql/${NAME}.qcow2 -t kvm -n ${NAME} -r 1024 -v 1 -b vmnet2

# NAME=haproxy2-101-72; bash  safe_clone_kvm.sh -i /VMs/template/centos7-1804-1C1G.qcow2 -d /VMs/mysql/${NAME}.qcow2 -t kvm -n ${NAME} -r 1024 -v 1 -b vmnet2
```





#### 9.4.6.2  安装haproxy + keepalived

略: [实现haproxy+keepalived高可用集群转发](http://liangcheng.mykernel.cn/2020/10/26/%E5%AE%9E%E7%8E%B0haproxy-keepalived%E9%AB%98%E5%8F%AF%E7%94%A8%E9%9B%86%E7%BE%A4%E8%BD%AC%E5%8F%91/)

1. keepalived vip漂移ok
2. haproxy结合httpd配置
3. 请求vip过程中，haproxy停止vip自动漂移, 不影响用户



master

```bash
# /etc/keepalived/keepalived.conf 

! Configuration File for keepalived
vrrp_script magedu {
   script "/etc/keepalived/scripts/vrrp_script.sh"
   interval 1
   timeout 10
   weight -30 # 注意, 可以看上面的图理解
   rise 5
   fall 2
   user root
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
   vrrp_strict
   vrrp_iptables
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 25
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.101.248/24 dev eth0 label eth0:0
        192.168.101.249/24 dev eth0 label eth0:1
    }
    track_script {
		magedu	
	}
      notify_master "/etc/keepalived/scripts/notify.sh master"
      notify_backup "/etc/keepalived/scripts/notify.sh backup"
      notify_fault "/etc/keepalived/scripts/notify.sh fault"
}

[root@centos-template ~]# cat /etc/keepalived/scripts/notify.sh 
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
contact='root@localhost' 
notify() {
    local mailsubject="$(hostname) to be $1, vip floating"
    local mailbody="$(date +'%F %T'): vrrp transition, $(hostname) changed to be $1"  
	echo "$mailbody" | mail -s "$mailsubject" $contact
}
case $1 in
master)
    notify master
		ps -ef | grep  rsync_inotify   | grep -vq grep   ||	 nohup /opt/inotify_rsync.sh	& 
    ;;
backup)
    notify backup
	 	 ps -ef | grep inotifywait | awk '{printf "kill -9 %s\n",$2}' | bash 2> /dev/null
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

backup

```bash
[root@centos-template keepalived]# cat /etc/keepalived/keepalived.conf 
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
   vrrp_iptables
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 25
    priority 80
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.101.248/24 dev eth0 label eth0:0
        192.168.101.249/24 dev eth0 label eth0:1
    }
      notify_master "/etc/keepalived/scripts/notify.sh master"
      notify_backup "/etc/keepalived/scripts/notify.sh backup"
      notify_fault "/etc/keepalived/scripts/notify.sh fault"
}
[root@centos-template keepalived]# cat /etc/keepalived/scripts/
notify.sh       vrrp_script.sh  
[root@centos-template keepalived]# cat /etc/keepalived/scripts/notify.sh 
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
contact='root@localhost' 
notify() {
    local mailsubject="$(hostname) to be $1, vip floating"
    local mailbody="$(date +'%F %T'): vrrp transition, $(hostname) changed to be $1"  
	echo "$mailbody" | mail -s "$mailsubject" $contact
}
case $1 in
master)
    notify master
		ps -ef | grep  rsync_inotify   | grep -vq grep   ||	 nohup /opt/inotify_rsync.sh	& 
    ;;
backup)
    notify backup
	 	 ps -ef | grep inotifywait | awk '{printf "kill -9 %s\n",$2}' | bash 2> /dev/null
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





haproxy 配置

两个haproxy一样的配置

```bash
[root@centos-template keepalived]# cat /etc/haproxy/haproxy.cfg 
global
maxconn 100000
chroot /usr/local/haproxy
stats socket /var/lib/haproxy/haproxy.sock1 mode 600 level admin process 1 # 非交互完成服务器热上线和下线
stats socket /var/lib/haproxy/haproxy.sock2 mode 600 level admin process 2 # 非交互完成服务器热上线和下线
stats socket /var/lib/haproxy/haproxy.sock3 mode 600 level admin process 3 # 非交互完成服务器热上线和下线
stats socket /var/lib/haproxy/haproxy.sock4 mode 600 level admin process 4 # 非交互完成服务器热上线和下线
user haproxy
group haproxy
daemon
#---
nbproc 4   #默认单进程启动
#nbthread 4  #可设置为单进程多线程或者多进程单线程，以及针对进程进程cpu绑定
cpu-map 1 3 # 因为服务器前2个核系统调用
cpu-map 2 4
cpu-map 3 5
cpu-map 4 6
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



# Nginx
listen http
  bind 192.168.101.248:80,192.168.101.249:80
  mode tcp
  server 10.21.101.1 10.21.101.1:80  check port 80 inter 3s fall 3 rise 5
  server 10.21.101.2 10.21.101.2:80  check port 80 inter 3s fall 3 rise 5

# mariadb
listen mariadb
  bind 192.168.101.248:3306,192.168.101.249:3306
  mode tcp
  server 192.168.101.11 192.168.101.11:3306  check port 3306 inter 3s fall 3 rise 5

```



#### 9.4.6.3 更新接了外网的haproxy的nginx配置

```bash
listen https
  #bind 172.27.0.248:443 ssl crt /usr/local/haproxy/certs/slc.mykernel.cn.pem #  cat 4847483_slc.mykernel.cn.pem 4847483_slc.mykernel.cn.key > slc.mykernel.cn.pem
  bind 172.27.0.248:8192
  mode tcp
  server 192.168.101.248  192.168.101.248:80  check port 80 inter 3s fall 3 rise 5
```

```bash
systemctl restart haproxy
```

访问网页正常http://172.27.0.248:8192/











### 9.4.7 安装wordpress

#### 9.4.7.1 放置wordpress

下载地址：https://wordpress.org/download/releases/

将wordpress站点页面放在nginx上和php上 /app/nginx/html目录下, 并完成以下授权

```bash
chown -R nginx.nginx /apps/nginx/html/*
```



#### 9.4.7.2 mysql master授权wordpress用户及密码

```bash
CREATE DATABASE wordpress CHARACTER SET 'utf8mb4';
GRANT ALL PRIVILEGES ON  wordpress.* TO 'wpuser'@'%' IDENTIFIED BY 'wppass';
```

测试连接从库

```bash
mysql -uwpuser -pwppass -h192.168.101.12
```

在wordpress指定数据库

```bash
cp wp-config-sample.php wp-config.php 
```



编辑wp-config.php 

```bash
define( 'DB_NAME', 'wordpress' );

/** MySQL database username */
define( 'DB_USER', 'wpuser' );

/** MySQL database password */
define( 'DB_PASSWORD', 'wppass' );

/** MySQL hostname */
define( 'DB_HOST', '192.168.101.249' );

/** Database Charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8' );

/** The Database Collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );


# 访问https://api.wordpress.org/secret-key/1.1/salt/获得如下内容
define('AUTH_KEY',         '<e$.o{vM0:WhsSf4}q#JpP#D5W)26gC*v{RZ<>CIoZ+/J8DaKD;]Hekvu5N/9ASS');
define('SECURE_AUTH_KEY',  'UZ;5r@iuLah$Mk`2%/Fz6A3%9I4Y&NtN%w<63e^k/,Rn!!~l5,irehd0:G[&7(Xy');
define('LOGGED_IN_KEY',    '33KJ3}?F!.QEE[`rHkI=>rKg(2RX]6`$6vp3s0c<-nlB:F`T9~:VDcvKNWvs,Q3d');
define('NONCE_KEY',        'yisj^D?#_V~53tp i3#{Xe0>6;GSS+/tZJ.ck<=e!90M@lKk&!`<2odFcc1:`!e ');
define('AUTH_SALT',        '<#^E[dln115}q)=F^^fj(6Ld1jY:u+)xL+a@dREI7b!(y<MUF2!ER.1:%A*#>w:G');
define('SECURE_AUTH_SALT', '#5=XcV}df$eps?S=3c^@B@S)+G_KW@|(.00$/pq+iX?3lPaiC|n7e>~Dw#|=P?w]');
define('LOGGED_IN_SALT',   'wUPV=;,*wMZG4pt CqF,QP;3Lh=lWJGy/=SZ_o9{6=SP,294h4 1,E:6w00}+*hb');
define('NONCE_SALT',       'S6%[1+x>-@s+f?q1l.wBEdJ-e%NH(Cb|cI067GN-uYk?s_H:-b]4R:%uU|>{pZx[');

```



#### 9.4.6.3 wordpress网页配置

![image-20201201162646155](http://myapp.img.mykernel.cn/image-20201201162646155.png)



![image-20201201162718761](http://myapp.img.mykernel.cn/image-20201201162718761.png)





![image-20201201162840753](http://myapp.img.mykernel.cn/image-20201201162840753.png)



安装完成后可以查看mysql的wordpress库有数据生成

```bash
[root@chengdu-huayang-linux48-mysql1-101-11 ~]# mysql 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 10
Server version: 10.5.8-MariaDB-log MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| hidb               |
| information_schema |
| mysql              |
| performance_schema |
| wordpress          |
+--------------------+
5 rows in set (0.001 sec)

MariaDB [(none)]> use wordpress;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [wordpress]> show tables;
+-----------------------+
| Tables_in_wordpress   |
+-----------------------+
| wp_commentmeta        |
| wp_comments           |
| wp_links              |
| wp_options            |
| wp_postmeta           |
| wp_posts              |
| wp_term_relationships |
| wp_term_taxonomy      |
| wp_termmeta           |
| wp_terms              |
| wp_usermeta           |
| wp_users              |
+-----------------------+
12 rows in set (0.000 sec)

MariaDB [wordpress]> 
```

#### 9.4.6.4 发布博客

![image-20201201163010947](http://myapp.img.mykernel.cn/image-20201201163010947.png)



#### 9.4.6.4 图片使用nfs共享

进入php主机

```bash
cd /apps/nginx/html/
[root@chengdu-huayang-linux48-php-108-1 html]# find . -name "1594*.png"
./wp-content/uploads/2020/12/15946919751.png
备份到tmp目录
[root@chengdu-huayang-linux48-php-108-1 wp-content]# mv uploads/2020/ /tmp/
[root@chengdu-huayang-linux48-php-108-1 wp-content]# readlink -f uploads/
/apps/nginx/html/wp-content/uploads
[root@chengdu-huayang-linux48-php-108-1 wp-content]# ll -dl uploads/
drwxr-xr-x 2 nginx nginx 6 12月  1 16:32 uploads/

```

挂载nfs至php-fpm, nginx 三个主机都需要执行

```bash
showmount -e 192.168.101.9
install -dv -o nginx -g nginx /apps/nginx/html/wp-content/uploads
mount -t nfs 192.168.101.9:/data/k8s_storage /apps/nginx/html/wp-content/uploads
echo '192.168.101.9:/data/k8s_storage /apps/nginx/html/wp-content/uploads nfs defaults        0 0' >> /etc/fstab 
```

现在将备份的文件挪回去

```bash
[root@chengdu-huayang-linux48-php-108-1 wp-content]# mv /tmp/2020/ uploads/
```

刷新网页

![image-20201201163831856](http://myapp.img.mykernel.cn/image-20201201163831856.png)



再上传一张图片, 正常获取









### 9.4.8 测试整个架构的高可用

停止一个服务

1. 停止外网主haproxy，正常

2. 停止内网主haproxy，正常

3. 停止一个nginx, 正常

4. 停止拥有VIP的nfs节点, 异常

   ```bash
   # nfs出现文件句柄不可用情况
   http://slc.mykernel.cn/2020/12/03/nginx%E4%BD%BF%E7%94%A8NFS%E7%9A%84VIP%E6%9D%A5%E6%8C%82%E8%BD%BD-%E5%BD%93VIP%E5%88%87%E6%8D%A2%E6%9C%89%E9%97%AE%E9%A2%98/
   ```

   ```bash
   nginx主机编写检测脚本
   # /etc/fstab里面有以下行
   # 192.168.101.9:/data/k8s_storage /apps/nginx/html/wp-content/uploads nfs defaults        0 0
   
   
   #!/bin/bash
   #
   #********************************************************************
   #Author:                songliangcheng
   #QQ:                    2192383945
   #Date:                  2020-12-03
   #FileName：             check_nfs.sh
   #URL:                   http://www.magedu.com
   #Description：          A test toy
   #Copyright (C):        2020 All rights reserved
   #********************************************************************
   for ((i=0;i<57;i=(i+1)));do
   	sleep 1
   	df -TH &> /dev/null
   	install -dv -o nginx -g nginx /apps/nginx/html/wp-content/uploads
   	[ $? -ne 0 ] && umount -tl /apps/nginx/html/wp-content/uploads && mount -t nfs 192.168.101.9:/data/k8s_storage /apps/nginx/html/wp-content/uploads
   	df -TH | grep -e nfs4  -e /data/k8s_storage || mount -t nfs 192.168.101.9:/data/k8s_storage /apps/nginx/html/wp-content/uploads
done
   
   ```
   
   

现在发现即使中断了服务，我们也无从得知，就上面配置了nfs的keepalived会发个邮件通知，以下将安装zabbix



### 9.4.9 zabbix + ansible

监控所有节点, 并使用当前主机管理所有节点

#### 9.4.9.1 准备ansible

之前所有kvm密码是123456, 使用ddns+dmz, 被攻击连续2天了。

今天先搭建ansible管理所有主机，周期性修改密码，并通知管理员

准备虚拟机同zabbix

```bash
# apt install ansible -y
```



所有主机使用免密公钥免密认证

```bash
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2020-12-02
#FileName：             sshpass.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2020 All rights reserved
#********************************************************************
hosts=(
# frontend
10.21.3.1
10.21.1.1
10.21.1.2

# middle
10.21.107.1
10.21.101.1
10.21.101.2

# end
192.168.101.11
192.168.101.12
192.168.101.1
192.168.101.2
192.168.101.71
192.168.101.72

)


for host in ${hosts[@]}; do
	sshpass -p 123456 ssh-copy-id  root@$host -f
	[ $? -eq 0 ] && echo $host add success. ||  echo $host add failure.
done

```



将所有主机加入ansible的hosts

```bash
[iaas:children]
frontend
middle
end
###########
[frontend:children]
external-haproxy
#k8s
#jumpserver

[external-haproxy]
10.21.1.[1:2]
[k8s]
10.21.2.1
10.21.2.101
[jumpserver]
10.21.4.1

###########
[middle:children]
nginx
#gitlab
#elk
zabbix

[nginx]
10.21.101.[1:2]
[gitlab]
10.21.102.1
[jenkins]
10.21.103.1
[elk]
10.21.10[4:6].1
[zabbix]
10.21.107.1

###########
[end:children]
nfs
mysql
#redis
inter-haproxy
#rabbitmq

[nfs]
192.168.101.[1:2]
[mysql]
192.168.101.[11:12]
[inter-haproxy]
192.168.101.[71:72]
[redis]
192.168.101.[31:33]
[rabbitmq]
192.168.101.5[1:3]

```

准备修改密码的role

```bash
ansible-galaxy init roles/chpasswd
```

准备修改密码的脚本chpassword.sh 

```bash
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2020-12-02
#FileName：             chpassword.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2020 All rights reserved
#********************************************************************

PASSWORD="$1"
if [ -z "$PASSWORD" ]; then
	echo password must be exists!
	exit 123
fi
if which apt &> /dev/null; then
	echo "root:$PASSWORD"  | chpasswd
else
	echo "$PASSWORD"  | passwd --stdin root
fi
```

role文件

```bash
root@chengdu-huayang-linux48-zabbix-ansible-107-1:/etc/ansible# tree roles/chpasswd/
roles/chpasswd/
├── defaults # 变量
│   └── main.yml
├── files     # copy 依赖的文件
│   └── chpassword.sh
├── handlers # 触发器
│   └── main.yml
├── meta 
│   └── main.yml
├── README.md
├── tasks   # 主任务
│   └── main.yml
├── templates # template 依赖的文件
├── tests 
│   ├── inventory
│   └── test.yml
└── vars # 变量
    └── main.yml


```



roles/chpasswd/tasks/main.yml

```bash
---
# tasks file for roles/haproxy
# src以/结尾, 表示复制src目录中的所有文件
# If `dest' is a nonexistent path and if either `dest' ends with "/" or `src' is a directory, `dest' is created. 
- name: 推送脚本
  copy: src=chpassword.sh dest=/tools/downloads/

# 安装
- name: chpassword
  shell: cd /tools/downloads/ && bash chpassword.sh {{password}}
```

role 7.chpasswd.yaml 

```bash
- hosts: external-haproxy
  remote_user: root
  roles:
  - { name: chpasswd } 

```

##### 9.4.9.1.1 执行完成密码更新

```bash
# password=$(openssl rand -base64 11)
# echo $password
VdlFk7fbjrlZiZk=
# ansible-playbook -e password="$password" 7.chpasswd.yaml 
```

##### 9.4.9.1.2 通知管理员

安装邮件客户端

```bash
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2020-12-02
#FileName：             install.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2020 All rights reserved
#********************************************************************

qq=${1:-"2192383945@qq.com"}
smtp=${2:-smtp.qq.com}
shouquanma=${3:-tomujjpxtdegdjag}



if which apt &> /dev/null; then
		apt install s-nail -y
		cat  >> /etc/s-nail.rc <<EOF
set from="${qq}"
set smtp=${smtp}
set smtp-auth-user="${qq}"       
set smtp-auth-password="${shouquanma}"        
set smtp-auth=login
EOF
		s-nail -s "$(date +%F_%T): $(hostname) test email on ubuntu" ${qq} </etc/issue

else
		yum -y install mailx
		cat  >> /etc/mail.rc <<EOF
set from="${qq}"
set smtp=${smtp}
set smtp-auth-user="${qq}"       
set smtp-auth-password="${shouquanma}"        
set smtp-auth=login

EOF
		mail -s "$(date +%F_%T): $(hostname) test email on centos" ${qq} </etc/issue
fi

```

将更新密码和邮件结合起来组成脚本，加入crontab

```bash
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2020-12-02
#FileName：             install-inotify.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2020 All rights reserved
#********************************************************************
tar Pxvf ansible_chpasswd_role.tar.gz
install -Dv chpasswd_inotify_administrator.sh /tools/softwares/chpasswd_inotify_administrator.sh


if which apt &> /dev/null; then
echo '*/5 * * * * /tools/softwares/chpasswd_inotify_administrator.sh' >>  /var/spool/cron/crontabs/root
else
echo '*/5 * * * * /tools/softwares/chpasswd_inotify_administrator.sh' >> /var/spool/cron/root
fi


echo 查看crontab
crontab -l



echo
echo 现在是每5分钟就会更新密码, 方便查看是否正常配置, 正常后修改crontab, 建议3个月换一次: 00 00 30 * * /tools/softwares/chpasswd_inotify_administrator.sh
```



#### 9.4.9.2 准备zabbix server



```bash
# NAME=zabbix1-ansible1-107-1; bash  safe_clone_kvm.sh -i /VMs/template/ubuntu-1804-1C1G.qcow2 -d /VMs/mysql/${NAME}.qcow2 -t kvm -n ${NAME} -r 1024 -v 1 -b vmnet1
```

```bash
MariaDB [(none)]> create database zabbix character set utf8 collate utf8_bin;
Query OK, 1 row affected (0.040 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON  zabbix.* TO 'zbxuser'@'%' IDENTIFIED BY 'zbxpass';

```

执行zabbix_server一键安装脚本

```bash
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2020-12-07
#FileName：             install.sh
#URL:                   http://www.magedu.com
#Description：          编译安装zabbix 5.0, zabbix-agent, zabbix-proxy, zabbix-server
#Copyright (C):        2020 All rights reserved
#********************************************************************

#########
PROGNAME=${1:-agent}
# 以下
# mysql授权
# 1. GRANT ALL ON zabbix.* TO 'zbxuser'@'%' IDENTIFIED BY 'zbxpass';
# 2. GRANT ALL ON zabbix_proxy.* TO 'zbxuser'@'%' IDENTIFIED BY 'zbxpass';
# 安装nginx/php
zbx_user=${2:-zbxuser}
zbx_pass=${3:-zbxpass}
zbx_host=${4:-192.168.101.11}
zbx_dbname=${5:-zabbix}
zbx_proxy_dbname=${6:-zabbix_proxy}
#######################
CURRENT_PWD=$(readlink -f $(dirname "$0"))




eval $(grep  '^ID=' /etc/os-release)
os_family=${ID,,}


# 安装go环境
wget http://myapp.img.qiniu.mykernel.cn/go1.15.6.linux-amd64.tar.gz
tar xvf ./go1.15.6.linux-amd64.tar.gz   -C /usr/local/  
export PATH=$PATH:/usr/local/go/bin
go version

# 安装zabbix-agent, zabbix-proxy, zabbix-server
wget https://cdn.zabbix.com/zabbix/sources/stable/5.0/zabbix-5.0.6.tar.gz
tar xvf zabbix-5.0.6.tar.gz -C /usr/local/src/
groupadd -r zabbix
useradd -r -g zabbix -s /sbin/nologin -M -c 'zabbix monitor system' zabbix

if [ "$os_family" == "ubuntu" ]; then
# ubuntu安装
apt update
apt  install build-essential libmysqlclient-dev libssl-dev libsnmp-dev libevent-dev pkg-config libopenipmi-dev libcurl4-openssl-dev libxml2-dev libssh2-1-dev libpcre3-dev libldap2-dev libiksemel-dev libcurl4-openssl-dev libgnutls28-dev -y
cd /usr/local/src/zabbix-5.0.6/
./configure --prefix=/apps/zabbix5 --enable-server --enable-agent2 --with-mysql --enable-ipv6 --with-net-snmp --with-libcurl --with-libxml2  --enable-proxy  --with-ssh2 --with-openipmi --with-ldap --with-openssl

go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.cn,direct
make -s install
install misc/init.d/debian/zabbix-agent /etc/init.d/zabbix_agentd 
sed -i -e 's@NAME=.*$@NAME=zabbix_agentd@g' -e "s@DAEMON=.*\$@DAEMON=/apps/zabbix5/sbin/\${NAME}@g" /etc/init.d/zabbix_agentd
install /etc/init.d/zabbix_agentd /etc/init.d/zabbix_agent2
sed -i -e 's@NAME=.*$@NAME=zabbix_agent2@g' -e "s@DAEMON=.*\$@DAEMON=/apps/zabbix5/sbin/\${NAME}@g" /etc/init.d/zabbix_agent2
install /etc/init.d/zabbix_agentd /etc/init.d/zabbix_proxy
sed -i  's@NAME=.*$@NAME=zabbix_proxy@g' /etc/init.d/zabbix_proxy

install misc/init.d/debian/zabbix-server /etc/init.d/zabbix_server 
sed -i "s@DAEMON=.*\$@DAEMON=/apps/zabbix5/sbin/\${NAME}@g" /etc/init.d/zabbix_server 

systemctl daemon-reload



echo '
/etc/init.d/zabbix_agent2 restart
/etc/init.d/zabbix_agentd restart
/etc/init.d/zabbix_proxy  restart
/etc/init.d/zabbix_server  restart
'

elif [ "$os_family" == "centos" ]; then
# centos安装
yum -y install libxml2-devel libcurl-devel libcurl-devel net-snmp-devel libcurl-devel net-snmp-devel libevent-devel  openssl-devel  fping  unixODBC-devel  OpenIPMI-devel java-devel mysql-devel libssh2-devel openldap-devel
cd /usr/local/src/zabbix-5.0.6/
./configure --prefix=/apps/zabbix5 --enable-server --enable-agent2 --with-mysql --enable-ipv6 --with-net-snmp --with-libcurl --with-libxml2  --enable-proxy  --with-ssh2 --with-openipmi --with-ldap --with-openssl
go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.cn,direct
make -s install
install misc/init.d/fedora/core/zabbix_agentd /etc/init.d/
sed -i  's@BASEDIR=.*$@BASEDIR=/apps/zabbix5@g' /etc/init.d/zabbix_agentd
install /etc/init.d/zabbix_agentd /etc/init.d/zabbix_agent2
sed -i  's@BINARY_NAME=.*$@BINARY_NAME=zabbix_agent2@g' /etc/init.d/zabbix_agent2
install /etc/init.d/zabbix_agentd /etc/init.d/zabbix_proxy
sed -i  's@BINARY_NAME=.*$@BINARY_NAME=zabbix_proxy@g' /etc/init.d/zabbix_proxy

install misc/init.d/fedora/core/zabbix_server /etc/init.d/
sed -i  's@BASEDIR=.*$@BASEDIR=/apps/zabbix5@g' /etc/init.d/zabbix_server


systemctl daemon-reload


echo '
/etc/init.d/zabbix_agent2 restart
/etc/init.d/zabbix_agentd restart
/etc/init.d/zabbix_proxy  restart
/etc/init.d/zabbix_server  restart
'
fi

if [ "$PROGNAME" == "agent" ]; then
exit 

else

		######################## zabbix server安装 ##############################
		# echo 手工调整以下完成操作. tail -n +91 install.sh | bash


		## 1. 确保安装过nginx/php-fpm
		if ! [ -d /apps/nginx -o -d /apps/php7 ]; then
			echo 'please install nginx/php7'
			exit 125
		fi


		# web(依赖nginx/php已经安装)
		cp -a  ui/           /apps/nginx/html/zabbix
		chown nginx.nginx -R /apps/nginx/html/zabbix/
		cp    ${CURRENT_PWD}/simkai.ttf     /apps/nginx/html/zabbix/assets/fonts



		if [ "$os_family" == "centos" ]; then
			yum -y install mariadb
			# 中文
			yum -y install kde-l10n-Chinese  glibc-common
			localedef -c -f UTF-8 -i zh_CN zh_CN.utf8
			echo 'export LANG=zh_CN.utf8' > /etc/profile.d/chinese.sh
			systemctl restart php-fpm
		elif [ "$os_family" == "ubuntu" ]; then
			apt -y install mysql-client-core-5.7   
			# 中文
			apt-get install language-pack-zh* -y
			echo 'LANG="zh_CN.UTF-8"' > /etc/default/locale
			dpkg-reconfigure --frontend=noninteractive locales
			update-locale LANG=zh_CN.UTF-8
			systemctl restart php-fpm

		fi


		if [ "$PROGNAME" == "server" ]; then
		# 导入server的mysql(依赖mysql端已经授权以上用户)
		mysql -u${zbx_user} -p${zbx_pass} -h${zbx_host} -e "create database ${zbx_dbname} character set utf8 collate utf8_bin;"
		mysql -u${zbx_user} -p${zbx_pass} -h${zbx_host} -D ${zbx_dbname} < database/mysql/schema.sql 
		mysql -u${zbx_user} -p${zbx_pass} -h${zbx_host} -D ${zbx_dbname} < database/mysql/images.sql 
		mysql -u${zbx_user} -p${zbx_pass} -h${zbx_host} -D ${zbx_dbname} < database/mysql/data.sql 
		fi
		if [ "$PROGNAME" == "proxy" ]; then
		# 导入proxy的mysql(依赖mysql端已经授权以上用户)
		mysql -u${zbx_user} -p${zbx_pass} -h${zbx_host} -e "create database ${zbx_proxy_dbname} character set utf8 collate utf8_bin;" 
		mysql -u${zbx_user} -p${zbx_pass} -h${zbx_host} -D ${zbx_proxy_dbname} < database/mysql/schema.sql 
		fi

sed -i.bak -e "s,^# DBHost=localhost,DBHost=${zbx_host},g" -e "s@^DBUser=.*\$@DBUser=${zbx_user}@g" -e "s@^# DBPassword=@DBPassword=${zbx_pass}@g" -e "s@# DBPort=@DBPort=3306@g" /apps/zabbix5/etc/zabbix_server.conf
sed -i.bak -e "s,^# DBHost=localhost,DBHost=${zbx_host},g" -e "s@^DBUser=.*\$@DBUser=${zbx_user}@g" -e "s@^# DBPassword=@DBPassword=${zbx_pass}@g" -e "s@# DBPort=@DBPort=3306@g" /apps/zabbix5/etc/zabbix_proxy.conf


# php配置
echo '
server {
	listen 8080;
    root /apps/nginx/html;
	index index.php;

    location ~ \.php$ {
		fastcgi_pass   127.0.0.1:9000;
		fastcgi_index  index.php;
		fastcgi_param  SCRIPT_FILENAME  /apps/nginx/html/$fastcgi_script_name;
		include        fastcgi_params;
	}  

	location ~ /\.ht {
		deny  all;
	}  
}
' > /apps/nginx/conf/conf.d/zabbix.conf 



/apps/nginx/sbin/nginx -s reload
fi

```

```bash
bash install.sh server zbxuser zbxpass 192.168.101.11 zabbix zabbix_proxy
```



访问http://10.21.107.1/zabbix/setup.php

![image-20201204142013825](http://myapp.img.mykernel.cn/image-20201204142013825.png)

#### 9.4.9.3 所有主机安装agent

```bash
- hosts: iaas
  remote_user: root
  roles:
  - { name: zabbix, progname: agent }
```

```bash
ansible-playbook zabbix.yaml
```



网页使用自动注册，客户端工作在主动模式

![image-20201214154432707](http://myapp.img.mykernel.cn/image-20201214154432707.png)

agentd配置

```bash
#agent配置主动或被动。
PidFile=/tmp/zabbix_agentd.pid
LogType=file
LogFile=/tmp/zabbix_agentd.log
# 0不滚动日志
# 1-255,滚动大小M
LogFileSize=1
# 0 基本信息; 1 critical info; 2 error info; 3 warning; 4 debuging; 5 扩展debug.
DebugLevel=3
############# 被动监控配置
#指定server
# 跨网段是网关的ip
Server=10.21.107.1,172.27.1.107,192.168.101.254,10.21.107.1
ListenPort=10050
# 如果在server关联被动监控模板，server就依据此ip来获取数据
ListenIP=192.168.101.2
StartAgents=5
############主动监控配置
# 参数是用于在自动注册和主动监控(监控项)用的参数，设置为zabbix server或者是zabbix proxy的 IP。
ServerActive=10.21.107.1
# 如果在server关联主动监控模板，此为主机名
Hostname=192.168.101.2
# 主动检查刷新list的间隔
RefreshActiveChecks=120

# 无论agent是主动或被动, zabbix都会根据这个来判断是否符合linux
HostMetadataItem=system.uname
#  UserParameter=<key>,<shell command>
# See 'zabbix_agentd' directory for examples.
# 脚本执行的超时
Timeout=30
# agent运行为root?
AllowRoot=0
User=zabbix
TLSConnect=unencrypted
# Include=/usr/local/etc/zabbix_agentd.userparams.conf
# Include=/usr/local/etc/zabbix_agentd.conf.d/*.conf
# Include=/usr/local/etc/zabbix_agentd.conf.d/
Include=/var/lib/zabbix/percona/templates/userparameter_percona_mysql.conf

# 报警及配置
# https://blog.51cto.com/11555417/2044672

```

数据图![image-20201214154919639](http://myapp.img.mykernel.cn/image-20201214154919639.png)



### 9.4.10 spug运维管理平台

上课一会儿就睡觉，一会儿中午吃个饭，然后年薪30万。哈哈哈。

> 使用https://github.com/openspug/spug 工具完成自动化运维平台管理



安装https://spug.dev/docs/install/

https://spug.dev/docs/deploy-product-script/

```bash
$ curl https://spug.dev/installer/spug-installer | bash
```



配置主机

![image-20201214172412271](http://myapp.img.mykernel.cn/image-20201214172412271.png)

配置联系人

![image-20201214172421516](http://myapp.img.mykernel.cn/image-20201214172421516.png)

配置所在组

![image-20201214172439229](http://myapp.img.mykernel.cn/image-20201214172439229.png)

配置监控

![image-20201214172628455](http://myapp.img.mykernel.cn/image-20201214172628455.png)

![image-20201214172643859](http://myapp.img.mykernel.cn/image-20201214172643859.png)



配置环境

![image-20201214174232203](http://myapp.img.mykernel.cn/image-20201214174232203.png)



配置app应用

![image-20201214174247112](http://myapp.img.mykernel.cn/image-20201214174247112.png)

新建发布

常规发布：https://spug.dev/docs/deploy-config/

![image-20201214174425497](http://myapp.img.mykernel.cn/image-20201214174425497.png)

代码检出前、代码clone后、

发布前、发布后





发布申请

由于我配置github的仓库，是nginx代理，代理连接github慢，默认超时60, 会报504，就直接添加代理超时

![image-20201214175754157](http://myapp.img.mykernel.cn/image-20201214175754157.png)



通过

![image-20201214175812334](http://myapp.img.mykernel.cn/image-20201214175812334.png)





命令批量执行模板

![image-20201214180739512](http://myapp.img.mykernel.cn/image-20201214180739512.png)

执行任务，选择主机和模板

![image-20201214180809395](http://myapp.img.mykernel.cn/image-20201214180809395.png)





添加保存wordpress配置文件的目录

```bash
install -dv /data/conf

# 备份
cp /apps/nginx/html/wp-config.php /data/conf/
```

![image-20201216164337278](http://myapp.img.mykernel.cn/image-20201216164337278.png)

![image-20201216181617995](http://myapp.img.mykernel.cn/image-20201216181617995.png)

```bash
#检出后
ssh 10.21.101.1 "df -TH | grep -e nfs4  -e 192.168.101.9:/data/k8s_storage && umount /apps/nginx/html/wp-content/uploads" || true
ssh 10.21.101.2 "df -TH | grep -e nfs4  -e 192.168.101.9:/data/k8s_storage && umount /apps/nginx/html/wp-content/uploads" || true

#发布前
echo 卸载nfs
df -TH | grep -e nfs4  -e "192.168.101.9:/data/k8s_storage" && umount /apps/nginx/html/wp-content/uploads || true

#发布后
install -dv -o nginx -g nginx /apps/nginx/html/wp-content/uploads
echo 挂载nfs
df -TH | grep -e nfs4  -e "192.168.101.9:/data/k8s_storage" || mount -t nfs 192.168.101.9:/data/k8s_storage /apps/nginx/html/wp-content/uploads
echo 添加wordpress配置文件至$SPUG_DST_DIR
cp /data/conf/wp-config.php $SPUG_DST_DIR
```



 发布

![image-20201216181657600](http://myapp.img.mykernel.cn/image-20201216181657600.png)



但是二次发布时，会出现图片找不到的情况

spug原来的位置在1827(18:27分发布)，期望新挂载在1830(18:30分发布)，但是spug没有。

![image-20201217092941902](http://myapp.img.mykernel.cn/image-20201217092941902.png)



意识到发布的web主站的图片和主站应该分离

访问图片的URLhttp://lccnx.tpddns.cn:8192/wp-content/uploads/2020/12/15946919751-1024x525.png

所以将对/wp-content/uploads的请求重写到/data/images路径 

```bash
# 挂载脚本
sed -i 's@/apps/nginx/html/wp-content/uploads@/data/images@g' /tools/downloads/nginx/check_nfs.sh
# ansible重新配置检测脚本
root@chengdu-huayang-zabbix1-107-1:/opt/linux_scripts/ansible# ansible-playbook 2.nginx-php.yaml 

# nginx配置重写
[root@chengdu-huayang-linux48-nginx1-101-1 ~]# grep '^[[:space:]]*[^ #]' /apps/nginx/conf/nginx.conf
user nginx;
worker_processes  1;
pid         /apps/nginx/run/nginx.pid;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    log_format  main  '$http_x_forwarded_for - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" ';
    access_log  logs/access.log  main;
    sendfile        on;
    keepalive_timeout  65;
    server {
        listen       80;
        server_name  localhost;
        # 进入此location
		location /images {
			root /data/;
		}
		
        location / {
            root   html;
            index  index.php index.html index.htm;
            # 重写对上传图片位置的请求至/images, last跳出当前location。 
			rewrite /wp-content/uploads(.*)? /images$1 last;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
        location ~ \.php$ {
            root           /apps/nginx/html;
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi_params;
		    if ( $request_method = 'HEAD' ) {
				access_log  off;
			}
        }
        location ~ /(php-status|ping) {
			rewrite ^/php-status(.*)$ /status$2 break;
            root           /apps/nginx/html;
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi_params;
			allow 172.16.0.0/16;
			allow 10.21.0.0/16;
			allow 192.168.101.0/24;
			allow 127.0.0.1;
			deny all;                                    
			access_log  off;
        }
        location ~ /\.ht {
            deny  all;
        }
    }
}

```

![image-20201217095226305](http://myapp.img.mykernel.cn/image-20201217095226305.png)



正常访问，现在发版，不需要考虑位置了。



两次发布还是上传到源目录，添加链接

```bash
echo 添加wordpress配置文件至$SPUG_DST_DIR
cp /data/conf/wp-config.php $SPUG_DST_DIR
echo 添加权限 
chown -R nginx.nginx $SPUG_DST_DIR/
echo 链接上传路径
ln -sv /data/images/ /apps/nginx/html/wp-content/uploads
```





### 9.4.11 配置zabbix的mysql模板，监控mysql

https://git.zabbix.com/projects/ZBX/repos/zabbix/browse/templates/db/mysql_agent

将用户和密码写在/etc/my.cnf.d/mysql-clients.cnf才成功

编辑agentd配置

```bash
Include=/apps/zabbix5/etc/zabbix_agentd.conf.d/*.conf
```



编辑/apps/zabbix5/etc/zabbix_agentd.conf.d/template_db_mysql.conf

```bash
UserParameter=mysql.ping[*], mysqladmin -h"$1" -P"$2" ping
#UserParameter=mysql.ping[*], /apps/zabbix5/share/zabbix/alertscripts/ping.sh "$1" "$2"
UserParameter=mysql.get_status_variables[*], mysql -h"$1" -P"$2" -sNX -e "show global status"
UserParameter=mysql.version[*], mysqladmin -s -h"$1" -P"$2" version
UserParameter=mysql.db.discovery[*], mysql -h"$1" -P"$2" -sN -e "show databases"
UserParameter=mysql.dbsize[*], mysql -h"$1" -P"$2" -sN -e "SELECT COALESCE(SUM(DATA_LENGTH + INDEX_LENGTH),0) FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA='$3'"
UserParameter=mysql.replication.discovery[*], mysql -h"$1" -P"$2" -sNX -e "show slave status"
UserParameter=mysql.slave_status[*], mysql -h"$1" -P"$2" -sNX -e "show slave status"
```

重启

```bash
root@chengdu-huayang-zabbix1-107-1:/opt/linux_scripts# /apps/zabbix5/bin/zabbix_get -s 192.168.101.11 -k 'mysql.ping["localhost","3306"]'
mysqld is alive
```

网页直接启动

![image-20201215150127932](http://myapp.img.mykernel.cn/image-20201215150127932.png)





zabbix监控一切的解决方案：https://www.zabbix.com/cn/integrations?cat=

zabbix报警仅在扩展中显示：![image-20201215153147149](http://myapp.img.mykernel.cn/image-20201215153147149.png)

### 9.4.12 elk收集nginx日志

```bash
[root@centos7-iaas kvm]# NAME=elk1;  ./safe_clone_kvm.sh -i /VMs/template/ubuntu-1804-1C1G.qcow2 -d /VMs/elk/${NAME}.qcow2  -t kvm  -n ${NAME} -r 1024  -v 1 -b vmnet1
开始克隆 /VMs/template/ubuntu-1804-1C1G.qcow2 -> /VMs/elk/elk1.qcow2 
启动虚拟机
virt-install --virt-type kvm --name elk1 --ram 1024 --vcpus 1 --disk path=/VMs/elk/elk1.qcow2 --graphics vnc,listen=0.0.0.0 --noautoconsole --autostart --network bridge=vmnet1 --import
WARNING  未检测到操作系统，虚拟机性能可能会受到影响。使用 --os-variant 选项指定操作系统以获得最佳性能。

开始安装......
域创建完成。

[root@centos7-iaas kvm]# NAME=elk2;  ./safe_clone_kvm.sh -i /VMs/template/ubuntu-1804-1C1G.qcow2 -d /VMs/elk/${NAME}.qcow2  -t kvm  -n ${NAME} -r 1024  -v 1 -b vmnet1
开始克隆 /VMs/template/ubuntu-1804-1C1G.qcow2 -> /VMs/elk/elk2.qcow2 
启动虚拟机
virt-install --virt-type kvm --name elk2 --ram 1024 --vcpus 1 --disk path=/VMs/elk/elk2.qcow2 --graphics vnc,listen=0.0.0.0 --noautoconsole --autostart --network bridge=vmnet1 --import
WARNING  未检测到操作系统，虚拟机性能可能会受到影响。使用 --os-variant 选项指定操作系统以获得最佳性能。

开始安装......
域创建完成。

您在 /var/spool/mail/root 中有新邮件
[root@centos7-iaas kvm]# NAME=elk3;  ./safe_clone_kvm.sh -i /VMs/template/ubuntu-1804-1C1G.qcow2 -d /VMs/elk/${NAME}.qcow2  -t kvm  -n ${NAME} -r 1024  -v 1 -b vmnet1
开始克隆 /VMs/template/ubuntu-1804-1C1G.qcow2 -> /VMs/elk/elk3.qcow2 
启动虚拟机
virt-install --virt-type kvm --name elk3 --ram 1024 --vcpus 1 --disk path=/VMs/elk/elk3.qcow2 --graphics vnc,listen=0.0.0.0 --noautoconsole --autostart --network bridge=vmnet1 --import
WARNING  未检测到操作系统，虚拟机性能可能会受到影响。使用 --os-variant 选项指定操作系统以获得最佳性能。

开始安装......
域创建完成。

```



配置主机名解析和主机名

```bash
# /etc/hosts
127.0.0.1	localhost
127.0.1.1	ubuntu

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters



10.21.104.1 node1.magedu.com node1
10.21.105.1 node2.magedu.com node2
10.21.106.1 node3.magedu.com node3
```



https://www.elastic.co/cn/downloads/elasticsearch 安装步骤

https://www.elastic.co/cn/downloads/past-releases#elasticsearch 下载位置

```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.10.1-amd64.deb
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.10.1-amd64.deb.sha512
shasum -a 512 -c elasticsearch-7.10.1-amd64.deb.sha512 
sudo dpkg -i elasticsearch-7.10.1-amd64.deb
```

配置

```yaml
root@node1:/opt# cat /etc/elasticsearch/elasticsearch.yml
cluster.name: myblog
node.name: node1
path.data: /data/lib/elasticsearch
path.logs: /data/log/elasticsearch
bootstrap.memory_lock: true
network.host: 0.0.0.0
http.port: 9200
discovery.seed_hosts: ["node1", "node2","node3"]
cluster.initial_master_nodes: ["node1", "node2","node3"]
gateway.recover_after_nodes: 2
action.destructive_requires_name: true
http.cors.enabled: true
http.cors.allow-origin: "*"
```

> 其他主机node.name为node2, node3

```bash
# /etc/security/limits.conf
elasticsearch soft memlock unlimited
elasticsearch hard memlock unlimited

```



启动

```bash
install -dv -o elasticsearch -g elasticsearch /data/{lib,log}/elasticsearch


# /usr/lib/systemd/system/elasticsearch.service
44 LimitLOCK=infinity     


systemctl daemon-reload
systemctl restart elasticsearch
```



管理端

https://github.com/lmenezes/cerebro

```
wget https://github.com/lmenezes/cerebro/releases/download/v0.9.2/cerebro-0.9.2.tgz
bin/cerebro -Dhttp.port=1234 -Dhttp.address=0.0.0.0
```

![image-20201216102347246](http://myapp.img.mykernel.cn/image-20201216102347246.png)



安装kibana

https://www.elastic.co/cn/downloads/past-releases#kibana

```bash
root@node1:~# grep '^[a-Z]' /etc/kibana/kibana.yml
server.port: 5601
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://node1:9200","http://node2:9200","http://node3:9200"]
i18n.locale: "zh-CN"
```

http://10.21.104.1:5601/



nginx安装filebeat

![image-20201216110732645](http://myapp.img.mykernel.cn/image-20201216110732645.png)



![image-20201216110750624](http://myapp.img.mykernel.cn/image-20201216110750624.png)

```bash
[root@chengdu-huayang-linux48-nginx2-101-2 ~]# grep -n '^[[:space:]]*[a-Z-]' /etc/filebeat/filebeat.yml 
15:filebeat.inputs:
21:- type: log
24:  enabled: true
27:  paths:
28:    - /var/log/*.log
67:- type: filestream
70:  enabled: false
73:  paths:
74:    - /var/log/*.log
97:filebeat.config.modules:
99:  path: ${path.config}/modules.d/*.yml
102:  reload.enabled: false
109:setup.template.settings:
110:  index.number_of_shards: 1
146:setup.kibana:
# es地址
177:output.elasticsearch:
178:  hosts: ["10.21.104.1:9200","10.21.105.1:9200","10.21.106.1:9200"]
# kibana地址 setup写入dashboard
179:setup.kibana:
180:  host: "10.21.104.1:5601"
198:processors:
199:  - add_host_metadata:
200:      when.not.contains.tags: forwarded
201:  - add_cloud_metadata: ~
202:  - add_docker_metadata: ~
203:  - add_kubernetes_metadata: ~

  
  
[root@chengdu-huayang-linux48-nginx2-101-2 ~]# grep -n '^[[:space:]]*[a-Z-]' /etc/filebeat/modules.d/nginx.yml 
4:- module: nginx
6:  access:
7:    enabled: true
# nginx日志
11:    var.paths: ["/apps/nginx/logs/*.log"]
14:  error:
15:    enabled: true
22:  ingress_controller:
23:    enabled: false

```

```bash
sudo filebeat modules enable nginx

#sudo filebeat setup # 配置kibana仪表盘，这个只需要在第一个filebeat进程配置

sudo systemctl enable --now filebeat
```

![image-20201216144504921](http://myapp.img.mykernel.cn/image-20201216144504921.png)

### 9.4.13  elk网络分析

https://www.elastic.co/cn/beats/packetbeat

```bash
yum -y install  libpcap
yum -y install ./packetbeat-7.10.1-x86_64.rpm 

[root@chengdu-huayang-linux48-nginx2-101-2 ~]# grep -n '^[[:space:]]*[a-Z-]' /etc/packetbeat/packetbeat.yml
# 收集的接口
14:packetbeat.interfaces.device: eth0
19:packetbeat.flows:
22:  timeout: 30s
25:  period: 10s
29:packetbeat.protocols:
30:- type: icmp
32:  enabled: true
34:- type: amqp
37:  ports: [5672]
39:- type: cassandra
41:  ports: [9042]
43:- type: dhcpv4
45:  ports: [67, 68]
47:- type: dns
50:  ports: [53]
52:- type: http
55:  ports: [80, 8080, 8000, 5000, 8002]
57:- type: memcache
60:  ports: [11211]
62:- type: mysql
65:  ports: [3306,3307]
67:- type: pgsql
70:  ports: [5432]
72:- type: redis
75:  ports: [6379]
77:- type: thrift
80:  ports: [9090]
82:- type: mongodb
85:  ports: [27017]
87:- type: nfs
90:  ports: [2049]
92:- type: tls
95:  ports:
96:    - 443   # HTTPS
97:    - 993   # IMAPS
98:    - 995   # POP3S
99:    - 5223  # XMPP over SSL
100:    - 8443
101:    - 8883  # Secure MQTT
102:    - 9243  # Elasticsearch
104:- type: sip
106:  ports: [5060]
110:setup.template.settings:
111:  index.number_of_shards: 1
# kibana地址方便完成 setup加载dashboard
147:setup.kibana:
153:  host: "10.21.104.1:5601"
# es地址写日志
178:output.elasticsearch:
180:  hosts: ["10.21.104.1:9200","10.21.105.1:9200","10.21.106.1:9200"]
207:processors:
208:  - # Add forwarded to tags when processing data from a network tap or mirror.
209:    if.contains.tags: forwarded
210:    then:
211:      - drop_fields:
212:          fields: [host]
213:    else:
214:      - add_host_metadata: ~
215:  - add_cloud_metadata: ~
216:  - add_docker_metadata: ~



 #查看可用设备的列表，请运行：
 # packetbeat devices
 
 
 # 测试配置
 # packetbeat test config -e
 2020-12-16T15:04:07.066+0800	WARN	[cfgwarn]	sip/plugin.go:64	BETA: packetbeat SIP protocol is used
Config OK

#此步骤加载推荐的索引模板用于写入ElasticSearch并部署示例仪表板，以可视化Kibana中的数据。
# packetbeat setup -e

# systemctl enable --now packetbeat
```

直接从协议分析出mysql的慢日志

![image-20201216153002766](http://myapp.img.mykernel.cn/image-20201216153002766.png)



elk收集mysql日志



### 9.4.14 mysql读写分离

proxysql + keepalived --> mysql-master,       mysql-slave ....

略....



从库备份

 

# 10. kvm迁移

virt-manager添加新主机，直接点迁移

