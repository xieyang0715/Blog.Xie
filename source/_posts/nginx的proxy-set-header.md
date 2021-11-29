---
title: nginx的proxy_set_header和日志收集
date: 2020-12-29 10:35:40
tags: 
- 生产小经验
- nginx
---



**向后端传递的X-Real-IP一定要是真实的客户端IP**，X-Forwarded-For看几级代理确定怎么传递变量

```bash
# 1级代理
proxy_set_header X-Real-IP $remote_addr
proxy_set_header X-Forwarded-For $remote_addr


# 2级代理
proxy_set_header X-Real-IP $http_x_real_ip
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for
```



日志收集结果看elk怎么需要首部

```bash
# 1级
 '"clientip":"$remote_addr",'
 '"xff":"$http_x_forwarded_for",'
 '"tcp_xff":"$proxy_protocol_addr",'


# 2级
 '"clientip":"$http_x_real_ip",'
 '"xff":"$http_x_forwarded_for",'
 '"tcp_xff":"$proxy_protocol_addr",'
```

