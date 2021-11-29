---
title: openstack搭建环境之controller与node通信的rabbitmq配置
date: 2020-12-25 03:33:40
tags:
---



可以单独安装至其他服务器：
各组件通过消息发送与接收是实现组件之间的通信：

生产环境可以如此部署, 使用2个高配置的物理机放以下组件。

![image-20201225113648911](http://myapp.img.mykernel.cn/image-20201225113648911.png)

本次由于实验环境资源不够，就和mysql部署在同一个主机。

![image-20201225104216105](http://myapp.img.mykernel.cn/image-20201225104216105.png)

<!--more-->

# 搭建rabbitmq

```bash
# rabbitmq必须主机名解析，并且前缀也需要解析，否则启动不了
cat  > /etc/hosts <<EOF
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.16.0.103 openstack-mysql-master.magedu.local openstack-mysql-master
EOF

# ping验证解析成功
#ping openstack-mysql-master
#ping openstack-mysql-master.magedu.local



# 安装rabbitmq
yum install rabbitmq-server -y

# 开机自启和启动
systemctl enable rabbitmq-server.service
systemctl start rabbitmq-server.service

# 验证端口25672 5672
# ss -tnl

# 验证vip端口通telnet 172.16.0.248 5672

#添加openstack用户：openstack123是密码
rabbitmqctl add_user openstack openstack123
# 允许openstack用户配置、写入和读取访问权限。
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```



