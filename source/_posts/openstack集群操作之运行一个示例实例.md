---
title: openstack集群操作之运行一个示例实例
date: 2020-12-25 10:02:44
tags:
toc: true
---

https://docs.openstack.org/install-guide/launch-instance-networks-provider.html



# 1. 配置网络

```bash
. admin-openrc
 openstack network create  --share --external \
  --provider-physical-network external \
  --provider-network-type flat external-net
  
#第一行指定共享网络，openstack中的所有用户都可以使用这个网络；--external，声明是外部网络
第二行，指定提供者物理网络名称，需要是neutron配置文件中指定的名称，在这个定义的网络基础之上创建一个自己的网络
第三行指定提供者网络类型，指定flat桥接，指定网络名称
#安装openstack时，只是把网络的关联关系以及网络名称配置好，但还未创建网络


# 由于是桥接所以子网和宿主机网络一样，为了避免地址冲突，分配地址池自已决定
openstack subnet create --network external-net \  #基于所指定名称的网络创建子网
  --allocation-pool start=172.16.10.50,end=172.16.10.100 \ #指定子网地址池的开始及结束地址
  --dns-nameserver 223.6.6.6 --gateway 172.16.0.2 \ #指定DNS及宿主机网关
  --subnet-range 172.16.0.0/16 external-sub  #指定子网范围以及子网名称

#openstack subnet create --network external-net   --allocation-pool start=172.16.10.50,end=172.16.10.100   --dns-nameserver 223.6.6.6 --gateway 172.16.0.2  --subnet-range 172.16.0.0/16 external-sub
  
  
# 确保controller和node的brq桥均已经添加了eth0设备
#openstack T版本创建完子网后，网桥设备不会自动关联宿主机物理网卡的接口，所以需要强制指定桥接设备所关联的物理网卡，controller及每个node节点都需要强制关联
brctl show     #查看桥接设备，桥接设备brqxxxx必须对应一个物理网卡才能通讯



tail  /var/log/neutron/*.log  #日志中不能有任何报错
```



# 2. 添加实例类型

```bash
openstack flavor create --id 0 --vcpus 1 --ram 512 --disk 10 m1.nano
#创建虚拟机类型；
# id默认auto
# vcpus
# ram 默认256M
# disk 默认G单位

# 注意如果类型的cpu/ram/disk小于disk需要的，就运行不起来。
# kernel panic VFS not found.. 这样的错误


openstack flavor list  #列出虚拟机类型
```



# 3. 安装组配置

```bash
openstack security group rule create --proto icmp default
#给default安全组创建一些规则，指定icmp协议，允许其他机器去 ping 使用这个安全组的虚拟机

openstack security group rule create --proto tcp --dst-port 22 default
#给default安全组创建规则，协议为tcp，目标端口为22，允许其他机器通过 ssh 连接此安全组下的服务器


# . admin-openrc 
# openstack security group rule  list
```



# 4. ssh密钥

```bash
# 生成密钥
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -P ''


# 生成密钥对，当镜像启动时 cloud-init服务会连接openstack的api, 获取公钥写入配置指定用户的家目录authorized_keys文件中，用户可以直接基于
# 私钥登入创建的虚拟主机
openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey


9、openstack keypair list   #验证openstack中当前用户的公钥
```







# 5. 创建虚拟机

```bash
openstack flavor list  #列出实例类型名称
openstack image list   #列出可用的镜像， 镜像名
openstack network list  #列出openstack的网络, 获取网络ID


openstack server create --flavor m1.nano --image cirros-0.5.1 \
	--nic net-id=96a81a85-85b9-4222-af30-ee1983b70102 --security-group default --key-name mykey\
	linux-vm1 
# 实例类型名，镜像名，网络ID, 安全组

# 创建后node断网, 就是桥网卡和eth0未绑定上
# 如果没有添加上需要手工添加
#brctl addif brqxxx eth0 && systemctl restart network
# 解决Train版本不自动绑定的问题
#==controller节点和node节点==修改python文件实现brq网桥自动绑定宿主机eth0网卡
#vim /usr/lib/python2.7/site-#packages/neutron/plugins/ml2/drivers/linuxbridge/agent/linuxbridge_neutron_agent.py
#   metric = 100   #就让metric的值为100
#   #if 'metric' in gateway:  #注释掉这两行
#   #    metric = gateway['metric'] - 1
#如果不注释掉这两行，if判断gateway中没有metric的值时，则metric的值为空，这样做计算会报错，所以注释掉这两个值之后重启neutron服务，brq网桥设备就会自动绑定宿主机的eth0网卡；但也有可能造成所有虚拟机创建完后字段值都是100(目前没发生过)；
#宿主机eth0网卡默认是管理整个openstack的物理网络(管理网)，此网段不会被虚拟机所绑定，所以有可能是此原因造成eth0网卡无法被brq网桥设备自动绑定；
#如果brq网桥设备没有与宿主机eth0网卡做绑定，通过 brctl show 命令会发现，brq网桥设备的tap接口只与虚拟机网卡做绑定，未与宿主机的eth0做绑定，这样会造成虚拟机无法访问外网；
# 重启neutron


#NetlinkError: (13, 'Permission denied')
# 内核参数
# echo "net.ipv6.conf.all.disable_ipv6=1" >>/etc/sysctl.d/openstack.conf 
# sysctl -p /etc/sysctl.d/openstack.conf
#

# 重启neutron


#重建虚拟机
openstack server create --flavor m1.nano --image cirros-0.5.1 --nic net-id=96a81a85-85b9-4222-af30-ee1983b70102 --security-group default linux-vm2



openstack server list   #列出当前用户下所有虚拟机，虚拟机的状态必须是ACTIVE，并且必须拿到IP地址  

openstack console url show VM_NAME  #获取到指定虚拟机名称的控制台的URL
   
   
# node节点 确保日志没有错误
# tail  /var/log/neutron/*.log  /var/log/nova/*.log -f
#2020-12-26 10:19:31.871 20741 INFO neutron.plugins.ml2.drivers.agent._common_agent [req-caa5e89d-5df9-4c2b-98f1-3b676e76ce3d - - - - -] Port tapecfa02ec-92 updated. Details: {u'profile': {}, u'network_qos_policy_id': None, u'propagate_uplink_status': False, u'qos_policy_id': None, u'allowed_address_pairs': [], u'admin_state_up': True, u'network_id': u'96a81a85-85b9-4222-af30-ee1983b70102', u'segmentation_id': None, u'mtu': 1500, u'device_owner': u'compute:nova', u'physical_network': u'external', u'mac_address': u'fa:16:3e:8f:4c:a6', u'device': u'tapecfa02ec-92', u'port_security_enabled': True, u'port_id': u'ecfa02ec-92c7-48ce-b9e6-99e5445e86ca', u'fixed_ips': [{u'subnet_id': u'58b63459-9f64-4529-9ef2-f5dac5c8ffe8', u'ip_address': u'172.16.10.89'}], u'network_type': u'flat'}
# 输出了新虚拟机的ip

```

浏览器访问URL，宿主机需要配置hosts解析；openstack T版本在访问时会出现问题，虚拟机启动失败，需要修改node节点的一些配置，如下：

```bash
virsh capabilities  #查看系统支持的虚拟化类型；使用：pc-i440fx-rhel7.2.0

#vim /etc/nova/nova.conf    
[libvirt]
virt_type=kvm
#cpu_mode=host-passthrough                    #CPU模式使用主机透传模式，让虚拟机直接使用宿主机CPU模式，性能较好
hw_machine_type = x86_64=pc-i440fx-rhel7.2.0 #如果使用qemu或KVM虚拟化，需要指定执行的虚拟化类型


bash compute-node.sh  #重启nova服务

tail -f /var/log/nova/*.log /var/log/neutron/*.log  #日志中不能有任何报错
```

重试连接虚拟机

ping 172.31.7.89   #测试能否ping通创建的虚拟机IP




# 6. 关闭安全组

```bash
# controller节点 和 node节点都需要执行
sed -i 's@enable_security_group = true@enable_security_group = false@g' /etc/neutron/plugins/ml2/linuxbridge_agent.ini 

# 重启neutron
```



# 7. 虚拟机自动启动

```bash
# controller节点 和 node节点都需要修改
# /etc/nova/nova.conf
[DEFAULT]
resume_guests_state_on_host_boot = true

# 重启nova
```

