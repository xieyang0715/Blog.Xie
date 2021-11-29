---
title: openstack集群操作之kvm制作qcow2镜像原生版制作和官方版二次制作
date: 2020-12-29 00:39:41
tags:
toc: true
---





通过KVM 安装虚 Centos 和 Windwos 2008 R2_x86_64 操作系统步骤并将磁盘文件作为镜像上传到 openstack glance ，作为批量创建虚拟机的镜像文件，其中 windowsn 2008 安装 virtio半虚拟化驱动，以实现网络 IO 和磁盘 IO 的半虚拟化提升速度， Centos 7 默认即支持半虚拟化，不需要安装驱动.



# 1. 制作步骤

- [x] 基于kvm
- [x] 虚拟机连接网络需要brX桥接网卡。openstack的node节点则不需要桥接，自动生成brqX桥
- [x] 虚拟机最小化安装 系统 并配置 优化，做完 配置之后将虚拟机关机，然后将虚拟机 磁盘 文件 上传至 glance 即可启动虚拟机

https://docs.openstack.org/image-guide/create-images-manually.html



# 2. 准备kvm环境

[桥接配置](http://blog.mykernel.cn/2020/09/29/Ubuntu%E5%8F%8C%E7%BD%91%E5%8D%A1%E7%BB%91%E5%AE%9Abond0-%E5%8F%8C%E7%BD%91%E5%8D%A1%E6%A1%A5%E6%8E%A5/#1-2-%E6%A1%A5%E6%8E%A5bridge%E9%85%8D%E7%BD%AE)



或者命令行添加物理桥

```bash
# 添加开机启动
brctl addbr br0                         #添加虚拟桥
brctl addif br0 eth0                    #桥连接物理网卡
ifconfig br0 172.16.0.100/16 up         #ip
ip route add default via 172.16.0.2     #route
ip addr flush eth0                      #清空物理网卡地址

# 查看ip
[root@openstack-template ~]# ifconfig br0
br0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.16.0.100  netmask 255.255.0.0  broadcast 172.16.255.255
        inet6 fe80::20c:29ff:fe66:c350  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:66:c3:50  txqueuelen 1000  (Ethernet)
        RX packets 543  bytes 28075 (27.4 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 69  bytes 11067 (10.8 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

```

查看virsh默认NAT桥

```bash
[root@openstack-template ~]# virsh net-list
 Name                 State      Autostart     Persistent
----------------------------------------------------------
 default              active     yes           yes
 
 
[root@openstack-template ~]# ifconfig virbr0
virbr0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 192.168.122.1  netmask 255.255.255.0  broadcast 192.168.122.255
        ether 52:54:00:8c:e1:ec  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

```



# 3. 启动镜像

以下以centos镜像作示例，ubuntu/window一样的道理来启动

## 原生基于iso创建镜像

```bash
# 准备目录 
install -dv /vms

# 虽然镜像只有10G, openstack基于实例类型启动, 后期可以拉伸。
qemu-img create -f qcow2  /vms/centos-template.qcow2 10G 

# 准备镜像
wget https://mirrors.aliyun.com/centos-vault/7.5.1804/isos/x86_64/CentOS-7-x86_64-Minimal-1804.iso


# 基于ISO
virt-install --name centos-7-1804-template  \
--location=/vms/CentOS-7-x86_64-Minimal-1804.iso  --disk path=/vms/centos-template.qcow2,format=qcow2 \
--virt-type kvm \
--ram 2048 \
--vcpus 4 \
--network bridge=br0 \  # 基于桥启动。 --network network=default 基于默认桥启动
--graphics vnc,listen=0.0.0.0 \
--noautoconsole #--autostart
```

连接

```bash
# 命令
virt-manager

# vnc
# 直接连接宿主机的ip:5900
```

- [x] 语言支持：英文，不配置中文，避免openstack是python2出问题
- [x] 时区：Asia/Shanghai
- [x] 分区：**标准分区必须是XFS**，/boot: 512M, swap: 1024M, /: 留空 **不能使用LVM分区**，LVM就自已扩展。
- [x] 网络确定为eth0即可，不要打开
- [x] root密码

安装完成后关机



## 官方

```bash
# 下载镜像
# http://cloud.centos.org/centos/7/images/
# 选择xz压缩，空间小的，传输快，本地 xz -d 解压
cd /vms
wget http://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud-1708.qcow2.xz


# 基于qcow2镜像直接导入虚拟机
virt-install --name centos-genericcloud  \
--import  --disk path=/vms/CentOS-7-x86_64-GenericCloud-20150628_01.qcow2,format=qcow2 \
--virt-type kvm \
--ram 2048 \
--vcpus 4 \
--network bridge=br0 \  # 基于桥启动。 --network network=default 基于默认桥启动
--graphics vnc,listen=0.0.0.0 \
--noautoconsole #--autostart
```

# 4. 虚拟机优化

https://docs.openstack.org/image-guide/create-images-manually.html



## 4.1 基于配置

无论是官方镜像或是原生镜像都建议进行以下优化

- [x] ssh配置root登陆

  [ssh配置](http://blog.mykernel.cn/2020/09/29/Ubuntu-Server-18-04%E7%9A%84%E5%AE%89%E8%A3%85-%E4%BC%98%E5%8C%96%E7%B3%BB%E7%BB%9F/#3-3-3-%E9%85%8D%E7%BD%AEssh%E6%9C%8D%E5%8A%A1)

- [x] root密码

  ```bash
  # centos
  echo 'magedu' | passwd --stdin root
  # ubuntu
  echo 'root:magedu' | chpasswd
  ```

- [x] 主机名

  ```bash
  hostnamectl set-hostname "template-1804.magedu.local"
  ```

- [x] ${HOME}/.vimrc

  ```bash
  # ~/.vimrc
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

- [x] 内核参数，用户级资源限制

  [资源限制](http://blog.mykernel.cn/2020/09/29/Ubuntu-Server-18-04%E7%9A%84%E5%AE%89%E8%A3%85-%E4%BC%98%E5%8C%96%E7%B3%BB%E7%BB%9F/#3-8-%E7%B3%BB%E7%BB%9F%E8%B5%84%E6%BA%90%E9%99%90%E5%88%B6%E4%BC%98%E5%8C%96)

  [内核参数](http://blog.mykernel.cn/2020/09/29/Ubuntu-Server-18-04%E7%9A%84%E5%AE%89%E8%A3%85-%E4%BC%98%E5%8C%96%E7%B3%BB%E7%BB%9F/#3-9-%E5%86%85%E6%A0%B8%E5%8F%82%E6%95%B0%E4%BC%98%E5%8C%96)

- [x] iptables, selinux, NetworkManager禁用

  ```bash
  # systemctl disable NetworkManager firewalld
  # sed -Ei.bak '/SELINUX=/s/(SELINUX=)enforcing/\1disabled/' /etc/selinux/config
  ```

- [x] 基础包

  ```bash
  # centos
  yum install -y vim wget tree  lrzsz gcc gcc-c++ automake pcre pcre-devel zlib zlib-devel openssl openssl-devel iproute net-tools iotop ntpdate lsof
  
  # ubuntu
  apt install iproute2 nfs-kernel-server nfs-common lrzsz tree openssl libssl-dev libpcre3 libpcre3-dev zlib1g-dev ntpdate tcpdump telnet traceroute gcc make openssh-server iotop unzip zip lsof make curl iputils-ping vim -y
  ```

- [x] 配置中文, 别配置语言

  ```bash
  # localectl
  en_US.UTF-8
  ```

- [x] 配置网络，让其自已dhcp. 网络配置为dhcp, 因为openstack基于这个镜像启动的ip由openstack来分配。

  ```bash
  # cat > /etc/sysconfig/network-scripts/ifcfg-eth0 <<EOF
  BOOTPROTO=dhcp
  DEVICE=eth0
  ONBOOT=yes
  TYPE=Ethernet
  USERCTL=no
  EOF
  
  # cat /etc/netplan/01-netcfg.yaml 
  # This file describes the network interfaces available on your system
  # For more information, see netplan(5).
  network:
    version: 2
    renderer: networkd
    ethernets:
      eth0:
        dhcp4: dhcp
  ```

- [x] 时间同步

  ```bash
  # centos
  echo '*/5 * * * * /usr/sbin/ntpdate time1.aliyun.com &> /dev/null' > /var/spool/cron/root
  
  # ubuntu
  echo '*/5 * * * * /usr/sbin/ntpdate time1.aliyun.com &> /dev/null' > /var/spool/cron/crontabs/root
  ```

- [x] 时区

  ```bash
  timedatectl set-timezone "Asia/Shanghai"
  ```



网络是通的

```bash
[root@chengdu-huayang-centos-template ~]# ping www.baidu.com
PING www.a.shifen.com (14.215.177.39) 56(84) bytes of data.
64 bytes from 14.215.177.39 (14.215.177.39): icmp_seq=1 ttl=128 time=37.7 ms
```



## 4.2 原生镜像(必须进行以下优化)

### centos原生镜像

https://docs.openstack.org/image-guide/centos-image.html 最下方

```bash
# 使用cloud-init来获取公钥
yum install cloud-init -y

# 注意：以下是直接复制官方镜像的cloud.cfg配置文件。
[root@linux-vm7 ~]# cat /etc/cloud/cloud.cfg
users:
 - root                  # 配置ssh密钥放置账户
disable_root: 0          # 开放root ssh登陆。1表示为禁用，为0表示不禁用（部分OS的Cloud-Init配置使用true表示禁用，false表示不禁用）
ssh_pwauth:   1          # root支持密码登陆

mount_default_fields: [~, ~, 'auto', 'defaults,nofail', '0', '2']
resize_rootfs_tmp: /dev
ssh_deletekeys:   0
ssh_genkeytypes:  ~
syslog_fix_perms: ~

cloud_init_modules:
 - migrator
 - bootcmd
 - write-files
 - growpart
 - resizefs
 - set_hostname
 - update_hostname
 - update_etc_hosts
 - rsyslog
 - users-groups
 - ssh

cloud_config_modules:
 - mounts
 - locale
 - set-passwords
 - rh_subscription
 - yum-add-repo
 - package-update-upgrade-install
 - timezone
 - puppet
 - chef
 - salt-minion
 - mcollective
 - disable-ec2-metadata
 - runcmd

cloud_final_modules:
 - rightscale_userdata
 - scripts-per-once
 - scripts-per-boot
 - scripts-per-instance
 - scripts-user
 - ssh-authkey-fingerprints
 - keys-to-console
 - phone-home
 - final-message
 - power-state-change

system_info:
  default_user:
    name: centos
    lock_passwd: true
    gecos: Cloud User
    groups: [wheel, adm, systemd-journal]
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
    shell: /bin/bash
  distro: rhel
  paths:
    cloud_dir: /var/lib/cloud
    templates_dir: /etc/cloud/templates
  ssh_svcname: sshd

# vim:syntax=yaml


# 安装cloud-utils-growpart以允许分区调整大小
yum install cloud-utils-growpart -y

# 编写脚本以获取公钥（如果没有cloud-init）

# 禁用zeroconf路由
echo "NOZEROCONF=yes" >> /etc/sysconfig/network

# 配置控制台, 为了使nova console-log命令在CentOS 7上正常工作，您可能需要执行以下步骤：
#edit the /etc/default/grub file and configure the GRUB_CMDLINE_LINUX option. Delete the rhgb quiet and add console=tty0 console=ttyS0,115200n8 to the option.
grub2-mkconfig -o /boot/grub2/grub.cfg

# 关机, 保证数据一致性
poweroff

# 清理（​​删除MAC地址的详细信息）可以不清理
#virt-sysprep -d centos-7-1804-template


# 不销毁镜像，后期可能需要在镜像上二次制作


# 禁用cloud-init
systemctl disable cloud-init
systemctl enable  cloud-config.service  cloud-final.service    cloud-init.service         cloud-init-local.service   cloud-init.target

```



### ubuntu原生镜像

 https://docs.openstack.org/image-guide/ubuntu-image.html 最下方

```bash
# 使用cloud-init来获取公钥
apt install cloud-init

# 元数据源，使用EC2源
dpkg-reconfigure cloud-init

# /etc/cloud/cloud.cfg
users:
  - name: root # 配置ssh密钥放置账户
 
# 关机, 保证数据一致性
/sbin/shutdown -h now

# 清理（​​删除MAC地址的详细信息）可以不清理
#virt-sysprep -d ubuntu-1804-template

# 不销毁镜像，后期可能需要在镜像上二次制作
```



# 5. 基于镜像创建虚拟机

```bash
# admin用户
. admin-openrc 

# 将kvm镜像复制到controller主机
# scp centos-template.qcow2 172.16.0.102:/vms
# 导入glance
glance image-create --name "centos-7.5-1804"   --file /vms/centos-template.qcow2   --disk-format qcow2 --container-format bare   --visibility=public

# 查看镜像
glance image-list
+--------------------------------------+-----------------+
| ID                                   | Name            |
+--------------------------------------+-----------------+
| f1238576-d0d5-4fc5-b3f4-07023900620e | centos-7.5-1804 |
+--------------------------------------+-----------------+


# 创建flavor
openstack flavor create --vcpus 2 --ram 2048 --disk 60 2C-2G-60G

# 启动虚拟机
openstack flavor list  #列出实例类型名称
openstack image list   #列出可用的镜像， 镜像名
openstack network list  #列出openstack的网络, 获取网络ID

# 不需要安全组，http://blog.mykernel.cn/2020/12/25/openstack%E9%9B%86%E7%BE%A4%E6%93%8D%E4%BD%9C%E4%B9%8B%E8%BF%90%E8%A1%8C%E4%B8%80%E4%B8%AA%E7%A4%BA%E4%BE%8B%E5%AE%9E%E4%BE%8B/
# 此文章完成关闭安全组和启用虚拟机自动启动
openstack server create --flavor 2C-2G-60G --image centos-7.5-1804 \
	--nic net-id=96a81a85-85b9-4222-af30-ee1983b70102    --key-name mykey \
	linux-vm1
	
	
# 查看连接
openstack server list
openstack console url show linux-vm1


# 在被调度的机器上查看ip
tail -f /var/log/neutron/*.log
2020-12-29 11:20:24.705 1162 INFO neutron.plugins.ml2.drivers.agent._common_agent [req-fe70e736-1775-46f4-9844-a11c3fdd3117 - - - - -] Port tapa6f8110c-7f updated. Details: {u'profile': {}, u'network_qos_policy_id': None, u'propagate_uplink_status': False, u'qos_policy_id': None, u'allowed_address_pairs': [], u'admin_state_up': True, u'network_id': u'96a81a85-85b9-4222-af30-ee1983b70102', u'segmentation_id': None, u'mtu': 1500, u'device_owner': u'compute:nova', u'physical_network': u'external', u'mac_address': u'fa:16:3e:d0:f4:ae', u'device': u'tapa6f8110c-7f', u'port_security_enabled': True, u'port_id': u'a6f8110c-7f69-4eda-9a35-662854b72e9b', u'fixed_ips': [{u'subnet_id': u'58b63459-9f64-4529-9ef2-f5dac5c8ffe8', u'ip_address': u'172.16.10.99'}], u'network_type': u'flat'}


# 在被调度的机器上查看虚拟主机
virsh list


# 在被调度的机器上登陆控制台, 因为模板已经制作好了，可以登陆。
virsh console #
CentOS Linux 7 (Core)
Kernel 3.10.0-862.el7.x86_64 on an x86_64

linux-vm1 login: 



```



# 6. 磁盘拉伸

访问api

```bash
[root@linux-vm1 ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.16.0.2      0.0.0.0         UG    0      0        0 eth0
169.254.169.254 172.16.10.50    255.255.255.255 UGH   0      0        0 eth0
172.16.0.0      0.0.0.0         255.255.0.0     U     0      0        0 eth0

[root@linux-vm1 ~]# curl http://169.254.169.254
1.0
2007-01-19
2007-03-01
2007-08-29
2007-10-10
2007-12-15
2008-02-01
2008-09-01
2009-04-04

```

查看磁盘空间

```bash
[root@linux-vm7 ~]# df -TH
Filesystem     Type      Size  Used Avail Use% Mounted on
/dev/vda3      xfs        63G  1.4G   62G   3% /       #  --flavor 2C-2G-60G
devtmpfs       devtmpfs  953M     0  953M   0% /dev
tmpfs          tmpfs     964M     0  964M   0% /dev/shm
tmpfs          tmpfs     964M  8.9M  956M   1% /run
tmpfs          tmpfs     964M     0  964M   0% /sys/fs/cgroup
/dev/vda1      xfs       534M  134M  400M  26% /boot
tmpfs          tmpfs     193M     0  193M   0% /run/user/0
```



>  本文手工下载centos镜像，安装cloudinit服务，使用官方镜像的配置文件启动后，正常实现磁盘拉伸



