---
title: win10更新
date: 2021-10-28 15:41:17
tags:
- windows
---



# 前言

因为前一篇通过windows自带的更新工具，终究还是失败了，经过询问安全工程师，说自带的工具，会检测硬件、软件是否兼容，可能就更新不成功。最好的方式是通过ISO直接升级。

vmware 15.x 版本之后并且windows是21xx版本，用户空间隔离，可以与docker同时存在了。

> 20版本的windows, 使用docker需要hyper-V, 使用vmware 禁用hyper-V

模拟器：https://devblogs.microsoft.com/visualstudio/hyper-v-android-emulator-support/

> bluestacks 模拟器支持hyper-V

<!--more-->
# 如何获取ISO？

[获取ISO的视频](https://www.youtube.com/watch?v=-linHCoAE5w)



方法1

- baidu.com -> win10 下载 -> 修改user-agent 为ipad ios

方法二

- http://rufus.ie/zh/ -> 此工具portable 版本可以直接下载镜像



# 更新后的感悟

所有程序的数据丢失

每安装一个应用，都安装在其他盘，程序上一定要设置数据存放位置，也在其他盘。



# 迁移文档、下载

`windows徽标 + E`

以图片为例迁移到其它盘

![image-20211104100751543](http://myapp.img.mykernel.cn/image-20211104100751543.png)

