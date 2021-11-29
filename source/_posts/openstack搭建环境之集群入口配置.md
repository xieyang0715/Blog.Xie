---
title: openstack搭建环境之集群入口配置
date: 2020-12-25 03:01:09
tags:
toc: true
---



由于集群的流量通过haproxy进入，所以先把haproxy配置OK

![image-20201225104216105](http://myapp.img.mykernel.cn/image-20201225104216105.png)

<!--more-->

# 1. 基于模板克隆haproxy

```bash
cd /etc/sysconfig/network-scripts/
# ifcfg-eth0
IPADDR=172.16.0.105
# ifcfg-eth1
IPADDR=10.0.0.105

# /etc/hostname
openstack-ha1.magedu.local
```



# 2. 安装配置keepalived+haproxy

```bash
# 安装软件 
yum -y install haproxy keepalived

# 配置keepalived
cat > /etc/keepalived/keepalived.conf <<'EOF'
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
   vrrp_iptables
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 147
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
		172.16.0.248/16 dev eth0 label eth0:0
    }
}
EOF

systemctl enable keepalived --now



while ! ifconfig eth0:0; do
    sleep 1
done 

# 在其他节点上验证可以ping通vip
ping 172.16.0.248


# 配置haproxy
 cat > /etc/haproxy/haproxy.cfg <<'EOF'
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/lib/haproxy/stats
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    # 以下4个选项的调整可以避免(2013, 'Lost connection to MySQL server during query'), 不过haproxy在mysql空闲时会断开连接，正常现象。
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    
    timeout check           10s
    maxconn                 100000
    
listen stats    
  bind :9999
  stats enable
  stats hide-version
  stats uri /haproxy
  stats realm HAPorxy\ Stats\ Page
  stats auth admin:123456
  stats auth haadmin:123456
  stats admin if TRUE



listen openstack-7.7-mysql-3306
  bind 172.16.0.248:3306
  mode tcp                                 
  server 172.16.0.103 172.16.0.103:3306  check port 3306 inter 2s fall 3 rise 5
  server 172.16.0.104 172.16.0.104:3306  check port 3306 inter 2s fall 3 rise 5 
  
listen openstack-7.7-rabbitmq-5672
  bind 172.16.0.248:5672
  mode tcp                                 
  server 172.16.0.103 172.16.0.103:5672  check port 5672 inter 2s fall 3 rise 5 
  
listen openstack-7.7-memcached-11211
  bind 172.16.0.248:11211
  mode tcp                                 
  server 172.16.0.103 172.16.0.103:11211  check port 11211 inter 3s fall 3 rise 5 

listen openstack-7.7-keystone-api-5000
  bind 172.16.0.248:5000
  mode tcp                                 
  server 172.16.0.101 172.16.0.101:5000  check port 5000 inter 3s fall 3 rise 5 
  server 172.16.0.102 172.16.0.102:5000  check port 5000 inter 3s fall 3 rise 5 
listen openstack-7.7-glance-api-9292
  bind 172.16.0.248:9292
  mode tcp                                 
  server 172.16.0.101 172.16.0.101:9292  check port 9292 inter 3s fall 3 rise 5 
  server 172.16.0.102 172.16.0.102:9292  check port 9292 inter 3s fall 3 rise 5 
   
listen openstack-7.7-placement-api-8778
  bind 172.16.0.248:8778
  mode tcp                                 
  server 172.16.0.101 172.16.0.101:8778  check port 8778 inter 3s fall 3 rise 5 
  server 172.16.0.102 172.16.0.102:8778  check port 8778 inter 3s fall 3 rise 5 
  
listen openstack-7.7-nova-api-8774
  bind 172.16.0.248:8774
  mode tcp                                 
  server 172.16.0.101 172.16.0.101:8774  check port 8774 inter 3s fall 3 rise 5
  server 172.16.0.102 172.16.0.102:8774  check port 8774 inter 3s fall 3 rise 5
  
listen openstack-7.7-nova-api-8775
  bind 172.16.0.248:8775
  mode tcp                                 
  server 172.16.0.101 172.16.0.101:8775  check port 8775 inter 3s fall 3 rise 5
  server 172.16.0.102 172.16.0.102:8775  check port 8775 inter 3s fall 3 rise 5
  #虚拟机通过请求nova API获取当前虚拟机的元数据，如公钥(获取到到的公钥会写入到虚拟机镜像中，用于ssh连接)、实例类型等，以实现让虚拟机进行磁盘空间拉伸
  
listen openstack-7.7-vnc-6080
  bind 172.16.0.248:6080
  mode tcp                                 
  server 172.16.0.101 172.16.0.101:6080  check port 6080 inter 3s fall 3 rise 5 
  server 172.16.0.102 172.16.0.102:6080  check port 6080 inter 3s fall 3 rise 5 
  
  
listen openstack-7.7-neutron-9696
  bind 172.16.0.248:9696
  mode tcp                                 
  server 172.16.0.101 172.16.0.101:9696  check port 9696 inter 3s fall 3 rise 5 
  server 172.16.0.102 172.16.0.102:9696  check port 9696 inter 3s fall 3 rise 5 
listen openstack-7.7-dashboard-80
  bind 172.16.0.248:80
  mode http
  server 172.16.0.101 172.16.0.101:80  check port 9696 inter 3s fall 3 rise 5   
  server 172.16.0.102 172.16.0.102:80  check port 9696 inter 3s fall 3 rise 5

  
EOF


systemctl enable haproxy --now

# 验证端口已经监听
ss -tnl


```

> 配置2个地址，后期会高可用。前期就算另一个不正常，因为这里haproxy会进行tcp检测，不正常会自动下线的。



# 3. 网页查看haproxy管理页面

 http://172.16.0.248:9999/haproxy



# 4. 高可用配置

监听非本机ip, 打开转发







