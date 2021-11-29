---
title: systemd-journald日志配置
date: 2021-11-01 11:52:40
tags:
- systemd
---



# 前言

进程内存异常，分析系统日志、业务/系统访问流量、业务请求、业务连接





<!--more-->
# systemd-journald日志配置

默认是1个月的日志，日志丢失，重新配置

`/etc/systemd/journald.conf `

```bash
# 保留时间
MaxRetentionSec=1week
# 单个日志大小
SystemMaxFileSize=200M
# 日志总量
SystemMaxUse=10G
```

```bash
systemctl restart systemd-journald
```

> 引用自：https://www.cnblogs.com/vincenshen/p/10751566.html
>
> 清理日志：https://www.cxyzjd.com/article/GX_1_11_real/100974324

