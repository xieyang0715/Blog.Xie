---
title: openstack搭建环境之验证集群
date: 2020-12-25 09:46:43
tags:
---





# 1. nova 检验

当nova的controller和node配置成功后，node会自动通过rabbitmq注册到nova controller, 可以通过以下命令在controller端以admin用户权限查看 

```bash
. admin-openrc

# 列出注册的主机
openstack compute service list --service nova-compute

#扫描当前openstack中有哪些计算节点可用，发现后会把计算节点创建到cell中，后面就可以在cell中创建虚拟机；相当于openstack内部对计算节点进行分组，把计算节点分配到不同的cell中
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova

```

每次添加一个计算节点至cell，在控制端就需要执行一次扫描，这样会很麻烦，所以可以修改控制端nova的主配置文件：

```bash
# /etc/nova/nova.conf
[scheduler]
discover_hosts_in_cells_interval = 300  #设置每5分钟自动扫描一次可用计算节点
```



检查

```bash
#检查 nova 的各个服务是否都是正常，以及 compute 服务是否注册成功
openstack compute service list
 #查看各个组件的 api 是否正常
openstack catalog list
#查看是否能够拿到镜像
openstack image list
#查看cell的api和placement的api是否正常，只要其中一个有误，后期无法创建虚拟机
placement-status upgrade check


#https://ask.openstack.org/en/question/103325/ah01630-client-denied-by-server-configuration-usrbinnova-placement-api/
cat >> /etc/httpd/conf.d/00-placement-api.conf <<EOF
<Directory /usr/bin>
    <IfVersion >= 2.4>
        Require all granted
    </IfVersion>
    <IfVersion < 2.4>
        Order allow,deny
        Allow from all
    </IfVersion>
</Directory>
EOF
httpd -t
systemctl restart httpd 

nova-status upgrade check


#检查 nova 的各个服务是否都是正常
nova service-list 
```

# 2. neutron检验

```bash
. admin-openrc 
openstack extension list --network 
openstack network agent list  #验证网络客户端；状态必须是UP

nova service-list   #验证nova状态，必须都是UP
```

