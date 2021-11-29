---
title: windows配置openvpn
date: 2020-09-27 08:55:28
tags:  公司内部文档	
toc: true
---

# 1. windows 安装openvpn 2.4.8

下载链接：https://swupdate.openvpn.org/community/releases/openvpn-install-2.4.8-I602-Win10.exe

<!--more-->

## 1.1 点击Next

![image-20200927101757863](http://myapp.img.mykernel.cn/image-20200927101757863.png)

## 1.2 点击I Agree

![image-20200927101813841](http://myapp.img.mykernel.cn/image-20200927101813841.png)

## 1.3 点Next

![image-20200927101830725](http://myapp.img.mykernel.cn/image-20200927101830725.png)

## 1.4 点击Install

![image-20200927101844081](http://myapp.img.mykernel.cn/image-20200927101844081.png)

![image-20200927101856653](http://myapp.img.mykernel.cn/image-20200927101856653.png)

## 1.5 点击Next

![image-20200927101907496](http://myapp.img.mykernel.cn/image-20200927101907496.png)



## 1.6 点击Finish

![image-20200927101921162](http://myapp.img.mykernel.cn/image-20200927101921162.png)





# 2. 展开我提供的配置文件

![image-20200927101934138](http://myapp.img.mykernel.cn/image-20200927101934138.png)

# 3. 将所有文件复制到C:\Program Files\OpenVPN\config

![image-20200930140725305](http://myapp.img.mykernel.cn/image-20200930140725305.png)

# 4. 打开openvpnGUI

输入我提供的密码, 即可连接公司的网络. 192.168.0.0/24

![image-20200930140735430](http://myapp.img.mykernel.cn/image-20200930140735430.png)



# 5. 公司电脑打开openvpn

## 5.1 我的电脑，右键之后，选择属性

![image-20200930140756988](http://myapp.img.mykernel.cn/image-20200930140756988.png)

![image-20200930140810894](http://myapp.img.mykernel.cn/image-20200930140810894.png)

## 5.2 给自己电脑设置密码

![image-20200927102047697](http://myapp.img.mykernel.cn/image-20200927102047697.png)

![image-20200927102100421](http://myapp.img.mykernel.cn/image-20200927102100421.png)





## 5.3 电源设置

![image-20200927102112852](http://myapp.img.mykernel.cn/image-20200927102112852.png)

![image-20200927102124156](http://myapp.img.mykernel.cn/image-20200927102124156.png)





![image-20200927102140083](http://myapp.img.mykernel.cn/image-20200927102140083.png)

## 5.4 关闭防火墙

![image-20200927102152853](http://myapp.img.mykernel.cn/image-20200927102152853.png)

![image-20200927102207254](http://myapp.img.mykernel.cn/image-20200927102207254.png)

![image-20200927102220618](http://myapp.img.mykernel.cn/image-20200927102220618.png)

# 6. 使用rdp连接公司你的电脑的内网ip

![image-20200927102013605](http://myapp.img.mykernel.cn/image-20200927102013605.png)

点连接

![image-20200927102023332](http://myapp.img.mykernel.cn/image-20200927102023332.png)

