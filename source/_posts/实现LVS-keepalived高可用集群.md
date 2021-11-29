---
title: 实现LVS+keepalived高可用集群
date: 2020-10-26 13:34:10
tags: plan
toc: true
---



# 1. 规则集群架构

![image-20201026213928743](http://myapp.img.mykernel.cn/image-20201026213928743.png)

<!--more-->

基于上篇文章

http://liangcheng.mykernel.cn/2020/10/26/%E5%AE%9E%E7%8E%B0haproxy-keepalived%E9%AB%98%E5%8F%AF%E7%94%A8%E9%9B%86%E7%BE%A4%E8%BD%AC%E5%8F%91/, 停止haproxy, 去掉检测脚本, 让VIP恢复至master





>  keepalive + lvs
>
>  自动生成Lvs规则
>  自带lvs健康检查
>
>  一个VIP可以对应多个real server(2个或2个以上）新加后端，健康检测成功才会加入lvs.
>
>  keepalived +lvs 只 消耗CPU
>
>  **VIP迁移过程中，用户无感知**



> 主要DR模式
> 优点：
> 		请求经过LVS, 响应不经过LVS。节省带宽，和提升响应能力。
>
> 缺点：
> 	同局域网，不能跨网段（一个公司20位掩码，不存在跨网段）

>LVS DR配置后
>需要使用lvs-dr脚本绑定virtual server ip 到后端机器的lo:0接口



# 2. 配置keepalived

## 2.1 162配置

反代dns protocol UDP

web -> keepalive(vip)+lvs(vip+udp) -> dns

web需要使用dnsmasq，这个服务专门作为dns缓存, 默认10分钟。

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
   vrrp_iptables
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 239
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
		172.20.0.248/16 dev eth0 label eth0:0
    }
}




virtual_server 172.16.0.248 80 {      # vip用户入口(至少2个keepalive高可用, 由keepalived生成，可以多个keepalived节点上漂移)  port用户访问的端口
    delay_loop 6 # 检测后端服务器间隔
    lb_algo wrr
    lb_kind DR  
    # 注释相当于RR
    #persistence_timeout 50
    protocol TCP      #协议：一般tcp, 很少dns反向代理

    #sorry_server 192.168.200.200 1358

    real_server 172.20.0.151 80 { #vip同网段(真实的后端的地址)，端口和vip port一样
        weight 1 # 权重
        # notify_up <STRING>|<QUOTED-STRING> [username [groupname]] #rs上线通知脚本
        # notify_down <STRING>|<QUOTED-STRING> [username [groupname]] #rs下线通知脚本 

        HTTP_GET {
            url {
              path /index.html ## 哪个URI是固定用于检测，要给开发发邮件并抄送给领导。如果开发忘记提这个URI. 连不到这个URI, 所有后端全部被拆下线。
              # status_code #返回响应状态码，默认200-299, 都正常。
            }
            #connect_ip 对哪个ip检测, 默认realserver ip
            #connect_port 对哪个port检测, 默认realserver port
			#bindto  源地址
			#bind_if 源接口, 走哪个接口出去
			#bind_port 源端口
			
			
			
			# 连接realserver超时，默认5s
            connect_timeout 3
			# 重试几次, 默认失败
            retry 3
			# 重试延迟或重试间隔
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

```

```bash
apt install ipvsadm 

root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:~# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  172.16.0.248:80 wrr
  -> 172.20.0.151:80              Route   1      0          0         
  -> 172.20.0.152:80              Route   1      0          0         

```

### 2.1.1 后端配置

2个或多个后端均需要配置

`lvs-dr.sh`

```bash
#!/bin/bash
#
# Script to start LVS DR real server.
# description: LVS DR real server
#
.  /etc/rc.d/init.d/functions

VIP=172.16.0.248

case "$1" in
start)
       # Start LVS-DR real server on this machine.
        /sbin/ifconfig lo down
        /sbin/ifconfig lo up
        echo 1 > /proc/sys/net/ipv4/conf/lo/arp_ignore
        echo 2 > /proc/sys/net/ipv4/conf/lo/arp_announce
        echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
        echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce

        /sbin/ifconfig lo:0 $VIP broadcast $VIP netmask 255.255.255.255 up
        /sbin/route add -host $VIP dev lo:0

;;
stop)

        # Stop LVS-DR real server loopback device(s).
        /sbin/ifconfig lo:0 down
        echo 0 > /proc/sys/net/ipv4/conf/lo/arp_ignore
        echo 0 > /proc/sys/net/ipv4/conf/lo/arp_announce
        echo 0 > /proc/sys/net/ipv4/conf/all/arp_ignore
        echo 0 > /proc/sys/net/ipv4/conf/all/arp_announce

;;
status)

        # Status of LVS-DR real server.
        islothere=`/sbin/ifconfig lo:0 | grep $VIP`
        isrothere=`netstat -rn | grep "lo:0" | grep $VIP`
        if [ ! "$islothere" -o ! "isrothere" ];then
            # Either the route or the lo:0 device
            # not found.
            echo "LVS-DR real server Stopped."
        else
            echo "LVS-DR real server Running."
        fi
;;
*)
            # Invalid entry.
            echo "$0: Usage: $0 {start|status|stop}"
            exit 1
;;
esac
```



```bash
root@chengdu-chenghua-linux48-nginx-0-152:~# bash lvs-dr.sh start
  
root@chengdu-chenghua-linux48-nginx-0-152:~# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.20.0.152  netmask 255.255.0.0  broadcast 172.20.255.255
        inet6 fe80::20c:29ff:fe48:e5b3  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:48:e5:b3  txqueuelen 1000  (Ethernet)
        RX packets 4000  bytes 1479325 (1.4 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 3121  bytes 433116 (433.1 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 230  bytes 18471 (18.4 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 230  bytes 18471 (18.4 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo:0: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 172.16.0.248  netmask 255.255.255.255
        loop  txqueuelen 1000  (Local Loopback)


```



## 3. 163配置

```bash
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:~# scp /etc/keepalived/keepalived.conf 172.20.163:/etc/keepalived/keepalived.conf

root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:~# cat /etc/keepalived/keepalived.conf 
vrrp_instance VI_1 {
    state BACKUP #

    priority 80 #

}





root@chengdu-chenghua-linux48-keepalived-haproxy-0-163:/etc/keepalived# systemctl restart keepalived

```



# 4. 测试 vip切换用户无感知

```bash
root@chengdu-chenghua-linux48-keepalived-haproxy-0-163:~# curl 172.20.0.248
chengdu-chenghua-linux48-nginx-0-152
root@chengdu-chenghua-linux48-keepalived-haproxy-0-163:~# curl 172.20.0.248
chengdu-chenghua-linux48-nginx-0-151
root@chengdu-chenghua-linux48-keepalived-haproxy-0-163:~# curl 172.20.0.248
chengdu-chenghua-linux48-nginx-0-152
root@chengdu-chenghua-linux48-keepalived-haproxy-0-163:~# curl 172.20.0.248
chengdu-chenghua-linux48-nginx-0-151
root@chengdu-chenghua-linux48-keepalived-haproxy-0-163:~# curl 172.20.0.248
chengdu-chenghua-linux48-nginx-0-152

```

```bash
root@chengdu-chenghua-linux48-keepalive-haproxy-0-162:~# systemctl stop keepealived

root@chengdu-chenghua-linux48-keepalived-haproxy-0-163:/etc/keepalived# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.20.0.163  netmask 255.255.0.0  broadcast 172.20.255.255
        inet6 fe80::20c:29ff:feaf:b47a  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:af:b4:7a  txqueuelen 1000  (Ethernet)
        RX packets 36439  bytes 29157456 (29.1 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 29227  bytes 11197841 (11.1 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0:0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.20.0.248  netmask 255.255.0.0  broadcast 0.0.0.0
        ether 00:0c:29:af:b4:7a  txqueuelen 1000  (Ethernet)

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 1365  bytes 3104122 (3.1 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1365  bytes 3104122 (3.1 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0


root@chengdu-chenghua-linux48-nginx-0-152:~# while true; do sleep 1; curl 172.20.0.248; done
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-152
chengdu-chenghua-linux48-nginx-0-152

```



# 附录

仅配置单个前端负载时，可以使用这个脚本简化

```bash
#!/bin/bash
#
# LVS script for VS/DR
#
. /etc/rc.d/init.d/functions
#
VIP=192.168.80.229
RIP1=192.168.80.221
RIP2=192.168.80.222
PORT=80

#
case "$1" in
start)           

  /sbin/ifconfig eth0:1 $VIP broadcast $VIP netmask 255.255.255.255 up
  /sbin/route add -host $VIP dev eth0:1

# Since this is the Director we must be able to forward packets
  echo 1 > /proc/sys/net/ipv4/ip_forward

# Clear all iptables rules.
  /sbin/iptables -F

# Reset iptables counters.
  /sbin/iptables -Z

# Clear all ipvsadm rules/services.
  /sbin/ipvsadm -C

# Add an IP virtual service for VIP 192.168.80.229 port 80
# In this recipe, we will use the round-robin scheduling method.        # wrr
# In production, however, you should use a weighted, dynamic scheduling method. # wlc
  /sbin/ipvsadm -A -t $VIP:80 -s wrr

# Now direct packets for this VIP to
# the real server IP (RIP) inside the cluster
  /sbin/ipvsadm -a -t $VIP:80 -r $RIP1 -g -w 1
  /sbin/ipvsadm -a -t $VIP:80 -r $RIP2 -g -w 1

  /bin/touch /var/lock/subsys/ipvsadm &> /dev/null
;; 

stop)
# Stop forwarding packets
  echo 0 > /proc/sys/net/ipv4/ip_forward

# Reset ipvsadm
  /sbin/ipvsadm -C

# Bring down the VIP interface
  /sbin/ifconfig eth0:1 down
  /sbin/route del $VIP
  
  /bin/rm -f /var/lock/subsys/ipvsadm
  
  echo "ipvs is stopped..."
;;

status)
  if [ ! -e /var/lock/subsys/ipvsadm ]; then
    echo "ipvsadm is stopped ..."
  else
    echo "ipvs is running ..."
    ipvsadm -L -n
  fi
;;
*)
  echo "Usage: $0 {start|stop|status}"
;;
esac
```

