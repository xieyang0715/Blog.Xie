---
title: openstack搭建环境之部署dashboard组件
date: 2020-12-25 10:03:12
tags:
---



horizon是 openstack 的图形显示和操作界面，通过 API 和其他服务进行通讯 如镜像服务、计算服务和网络服务等结合使用 

horizon 基于 python django 开发，通过Apache 的 wsgi 模块进行 web 访问通信， Horizon 只需要更改配置文件连接到 keyston 即可

train版本部署步骤

https://docs.openstack.org/horizon/train/install/install-rdo.html

<!--more-->

```bash
yum install openstack-dashboard -y   #安装在controller节点

cat >> /etc/openstack-dashboard/local_settings <<EOF
OPENSTACK_HOST = "172.16.0.248"   
#指定为本机的监听地址

ALLOWED_HOSTS = ['172.16.0.248', 'openstack-vip.magedu.local']
#只允许通过列表中指定的域名访问dashboard；允许通过指定的IP地址及域名访问dahsboard；['*']表示允许所有域名

SESSION_ENGINE = 'django.contrib.sessions.backends.cache'  #指定session引擎
CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'openstack-vip.magedu.local:11211',  #指定memcache地址及端口
    } 
}
#配置session信息存放到memcache中；session信息不仅可以存放到memcache中，也可以存放到其他地方，参考文档：https://docs.openstack.org/horizon/latest/admin/sessions.html

OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
#配置keystone认证的API版本为v3

OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
#让dashboard支持域

OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
}
#配置openstack的API版本

OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
#设置keystone的默认域为default

OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
#指定通过dashboard创建用户的权限为user role的权限，而不是admin role权限

OPENSTACK_NEUTRON_NETWORK = {
    'enable_router': False,
    'enable_quotas': False,     
    'enable_distributed_router': False,
    'enable_ha_router': False,
    'enable_lb': False,
    'enable_firewall': False,
    'enable_vpn': False,
    'enable_fip_topology_check': False,
}
#如果使用的网络为二层网络，则关闭3层网络服务

TIME_ZONE = "Asia/Shanghai"
#修改时区

WEBROOT = '/dashboard'
#指定跟目录；如果不配置此项，则无法通过apache访问dashboard；当通过浏览器访问 http://172.16.0.248/dashboard 时会报404错误；告诉django程序，根目录为/dashboard，否则就会到 / 目录下做认证
EOF

#dashboard会把配置文件放到apache的配置文件目录，即 /etc/httpd/conf.d/openstack-dashboard.conf,并监听在80
cat >> /etc/httpd/conf.d/openstack-dashboard.conf <<EOF
WSGIApplicationGroup %{GLOBAL}
EOF


systemctl restart httpd.service  

```

浏览器访问；域为配置文件中设置的default，用户名和密码可以使用admin用户登录或者myuser用户登录

http://172.16.0.248/dashboard

http://openstack-vip.magedu.local/dashboard        

在dashboard上创建个user角色，否则无法创建项目

