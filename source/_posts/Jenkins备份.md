---
title: Jenkins备份
date: 2021-10-20 08:31:29
tags:
- jenkins
---



# 前言

虽然是我管理jenkins, 但是同事需要使用jenkins管理权限，添加一些功能。为了避免同事误删除job导致jenkins不可用，故采用定时备份的方法。



> 引用自百度: https://developer.aliyun.com/article/352665

配置

> 需要事先在jenkins主机准备目录: `/opt/jenkins_backup`
>
> ```bash
> install -dv /opt/jenkins_backup
> ```

![image-20211020083723550](http://myapp.img.mykernel.cn/image-20211020083723550.png)

<!--more-->
# 备份状态

过几天观察备份

![image-20211022173422323](http://myapp.img.mykernel.cn/image-20211022173422323.png)

