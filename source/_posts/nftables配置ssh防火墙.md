---
title: nftables配置ssh防火墙
date: 2020-11-25 06:41:11
tags: 日常挖掘
toc: true
---

它可以在Linux内核>=3.13中使用。

它附带了一个新的命令行实用程序。*NFT*其语法与iptable不同。

它还附带了一个兼容层，允许您在新的nftable内核框架上运行iptables命令。

它提供了通用集基础结构，允许您构造映射和连接。您可以使用这个新特性将规则集安排在多维树中。**大幅度**减少需要检查的规则的数量，直到在数据包上找到最后的操作为止。



与iptables区别

- 删除预定义链
- ipv4和ipv6统一使用inet规则

<!--more-->

# 1. 安装nftables

```bash
apt install nftables
```



# 2. 环境准备

使用nftables前，需要禁用默认的firewalld

```bash
# systemctl disable --now firewalld
# systemctl mask firewalld
```



# 3. 创建表

保存链和规则

```bash
#　nft add table inet filter
# 表示添加可以过滤inet类型(ipv4和ipv6)系列的数据包, 表名为filter
```



# 4. 创建链

```bash
#默认输入
# nft add chain inet filter input { type filter hook input priority 0 \; policy accept \; }

```

input 链名, 任意

type filter 链类型: route,nat

IPv4/IPv6/Inet地址簇中，hook可以是: prerouting , forward, input, output, postrouting, ingress

priority: 数字较小的链优先处理，并且可以是负数。

\; 避免被bash解释, \转义分号

policy accept： 默认情况下所有流量 accept



```bash
#lo接口
# nft add chain inet filter input { iif lo accept \; }
```



# 5. 所有已经建立连接和相关连接

>  *iptables-translate*实用程序何以将[iptables](https://wiki.archlinux.org/index.php/Iptables)规则转换成nftables格式。

```bash
# nft add rule inet filter input ct state established,related accept # 响应出站
# nft add rule inet filter input tcp dport 22 accept # 入站tcp 22
```

# 6. 拒绝其他流量 

```bash
# nft add rule inet filter input counter drop # 删除剩余流量 
```



# 7. 列出规则集

```bash
# nft list ruleset
table inet filter {
	chain input {
		type filter hook input priority 0; policy accept;
		ct state established,related accept
		tcp dport ssh accept
	}

	chain forward {
		type filter hook forward priority 0; policy accept;
	}

	chain output {
		type filter hook output priority 0; policy accept;
	}
}

```



# 8. 保存规则链

```bash
# nft list ruleset > /etc/sysconfig/nftables
# systemctl enable --now nftables.service
```



# 9. 清空和重载

```bash
# nftables flush ruleset

# nft list ruleset

# nft -f /etc/sysconfig/nftables
```



# 10. 配置示例

参考: https://wiki.archlinux.org/index.php/Nftables_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)

/etc/nftables.conf

---

```bash
flush ruleset

table inet filter {
        chain input {
                type filter hook input priority 0;

                # accept any localhost traffic
                iif lo accept

                # accept traffic originated from us
                ct state established,related accept

		# accept ICMP & IGMP
		ip6 nexthdr icmpv6 icmpv6 type { destination-unreachable, packet-too-big, time-exceeded, parameter-problem, mld-listener-query, mld-listener-report, mld-listener-reduction, nd-router-solicit, nd-router-advert, nd-neighbor-solicit, nd-neighbor-advert, ind-neighbor-solicit, ind-neighbor-advert, mld2-listener-report } accept
		ip protocol icmp icmp type { destination-unreachable, router-solicitation, router-advertisement, time-exceeded, parameter-problem } accept
		ip protocol igmp accept

                # activate the following line to accept common local services
                #tcp dport { 22, 80, 443 } ct state new accept

                # count and drop any other traffic
                counter drop
        }
}
```



firewall.rules

---

```bash
# A simple firewall

flush ruleset

table inet filter {
	chain input {
		type filter hook input priority 0; policy drop;

		# established/related connections
		ct state established,related accept

		# invalid connections
		ct state invalid drop
		
		# loopback interface
		iif lo accept

		# ICMP & IGMP
		ip6 nexthdr icmpv6 icmpv6 type { destination-unreachable, packet-too-big, time-exceeded, parameter-problem, mld-listener-query, mld-listener-report, mld-listener-reduction, nd-router-solicit, nd-router-advert, nd-neighbor-solicit, nd-neighbor-advert, ind-neighbor-solicit, ind-neighbor-advert, mld2-listener-report } accept
		ip protocol icmp icmp type { destination-unreachable, router-solicitation, router-advertisement, time-exceeded, parameter-problem } accept
		ip protocol igmp accept

		# SSH (port 22)
		tcp dport ssh accept

		# HTTP (ports 80 & 443)
		tcp dport { http, https } accept
	}

	chain forward {
		type filter hook forward priority 0; policy drop;
	}

	chain output {
		type filter hook output priority 0; policy accept;
	}

}
```

