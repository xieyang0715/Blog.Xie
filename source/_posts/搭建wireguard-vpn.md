---
title: 搭建wireguard-vpn
date: 2020-11-26 01:05:23
tags: 日常挖掘
toc: true
---

https://blogs.oracle.com/linux/wireguard-and-uek6u1

https://www.wireguard.com/install/



环境准备

```bash
	$ sudo yum install epel-release elrepo-release
	$ sudo yum install yum-plugin-elrepo
	$ sudo yum install kmod-wireguard wireguard-tools
```

<!--more-->

# 1. server端配置，内网地址是10.0.0.1

```bash
echo '
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
' > /etc/sysctl.d/wg-vpn.conf
sysctl -p /etc/sysctl.d/wg-vpn.conf
```

```bash
cd /etc/wireguard
(umask 077; wg genkey | tee privatekey | wg pubkey > publickey)


cat > wg0.conf <<EOF
[Interface]
Address = 192.168.2.1/24
SaveConfig = true
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; ip6tables -A FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE; ip6tables -D FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
ListenPort = 51820
PrivateKey = $(cat privatekey)
 
[Peer]
PublicKey = $(cat publickey)
AllowedIPs = 0.0.0.0/0, ::/0
#Endpoint = 10.0.0.2:60477
EOF
```


# 2. client端配置，内网地址是10.0.0.2
```bash
install -dv /etc/wireguard/client
 /etc/wireguard/client
(umask 077; wg genkey | tee privatekey | wg pubkey > publickey)
cat > wg0.conf <<EOF
[Interface]
Address = 192.168.2.2/24
SaveConfig = true
ListenPort = 60477
PrivateKey = $(cat privatekey)
 
[Peer]
PublicKey = $(cat publickey)
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = 公网IP:51820 # 示例: 10.0.0.1:51820
EOF
```