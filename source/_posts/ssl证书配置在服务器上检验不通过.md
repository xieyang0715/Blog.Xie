---
title: ssl证书配置在服务器上检验不通过
date: 2021-08-20 13:33:38
tags:
- 生产小经验
---



# 前言

"request to https://* failed, reason: unable to verify the first certificate",



调出请求这个域名对应的header， 由于开发是二级转发，将前端的header全部传递过来，host首部也传递过来了，导致waf认证证书失败。


<!--more-->

