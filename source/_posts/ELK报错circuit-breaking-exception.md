---
title: ELK报错circuit_breaking_exception
date: 2020-12-25 07:07:22
tags: 
- 生产小经验
- elk
---



今天突然kibana打开报错

TransportError(429, 'circuit_breaking_exception', '[parent] Data too large, data for [<http_request>] would be [993837424/947.7mb], which is larger than  the limit of [986061209/940.3mb], real usage: [984848776/939.2mb], new bytes reserved: [8988648/8.5mb], usages [request=9224248/8.7mb, fielddata=0/0b,  in_flight_requests=88005626/83.9mb, accounting=1591187/1.5mb]')

链接: https://juejin.cn/post/6844903953512005646

解决办法如下：
提升 config/jvm.options 中的 Xms 和 Xmx 的值 

但是公司的ecs服务器的规格较小，不需要这么多日志，所以写了以下脚本完成定期清理日志

```bash
# 清理前一个月的日志
 curl -XGET 'http://127.0.0.1:9200/_cat/indices/?v' | grep 2020.11 | awk '{printf  "/usr/bin/curl -XDELETE http://127.0.0.1:9200/%s\n",$3}' | bash

# 清理本月前20天日志
for i in {01..20}; do  curl -XGET 'http://127.0.0.1:9200/_cat/indices/?v' | grep 2020.12.${i} | awk '{printf  "/usr/bin/curl -XDELETE http://127.0.0.1:9200/%s\n",$3}' | bash; done

# crontab清理过去第5天的日志
 curl -XGET 'http://127.0.0.1:9200/_cat/indices/?v' | grep $(date -d'-5 days' +%Y.%m.%d)  | awk '{printf  "/usr/bin/curl -XDELETE http://127.0.0.1:9200/%s\n",$3}' | bash
```







脚本

```bash
[root@k8s-master1 cerebro-0.9.2]# cat /etc/zabbix/scripts/delete_es_index.sh
#!/bin/bash
DATE=$(date +%y.%m.%d)
#curl -XGET 'http://127.0.0.1:9200/_cat/indices/?v'
curl -XGET 'http://127.0.0.1:9200/_cat/indices/?v' | grep $(date -d'-4 days' +%Y.%m.%d)  | awk '{printf  "/usr/bin/curl -XDELETE http://127.0.0.1:9200/%s\n",$3}' | bash

# chmod +x /etc/zabbix/scripts/delete_es_index.sh



```

crontab

```bash
# crontab -e
# 4点清理索引
00 04 * * * /etc/zabbix/scripts/delete_es_index.sh 
```





