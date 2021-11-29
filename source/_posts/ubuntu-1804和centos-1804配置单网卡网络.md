---
title: ubuntu 1804和centos 1804配置单网卡网络
date: 2020-12-18 07:57:52
tags: 
- network-config
---



ubuntu

```
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
      addresses: [192.168.0.196/24]
      gateway4: 192.168.0.1
      nameservers:
        addresses: [223.6.6.6]
```

```bash
#启动
# netplan apply
```



centos

```
[root@centos-mq01 network-scripts]# cat ifcfg-eth0 
DEVICE=eth0
TYPE=Ethernet
BOOTPROTO=none
ONBOOT=yes
IPADDR=192.168.0.167
NETMASK=255.255.255.0
GATEWAY=192.168.0.1
```

```bash
# cat resolv.conf
nameserver 223.6.6.6
```

设备

类型

协议

- none/static 静态
- dhcp 

开机启动接口

地址

掩码

网关

```bash
#启动
# systemctl restart network
```

