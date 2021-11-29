---
title: windows代理sstap
tags:
- windows
- sstap代理
photos:
- http://myapp.img.mykernel.cn/2020010310201365.jpg
date: 2021-11-11 11:57:25
---

# 前言

之前一直使用`ccproxy`代理(前同事推荐)，但是今天下载不了了，本来这个只是一个`http`协议的代理，所以使用`nginx stream module`来代理即可



以下介绍在windows平台上，安装`nginx`，然后配置`nginx`代理`sstap`

更新: 由于nginx不支持后端健康状态检测，所以更新为haproxy

<!--more-->



# 安装配置 windows nginx

## 安装nginx



1. 官方下载页面： http://nginx.org/en/download.html , 其中下载stable version的windows版本
2. 展开到windows安装目录
3. 双击nginx.exe即可启动



## 获取代理的端口

![image-20211111120050365](http://myapp.img.mykernel.cn/image-20211111120050365.png)

一般这个端口，可能会变，要么在25378，要么在25379



## 配置nginx

让2个端口均可以使用，只是8端口不可用时，就切换到9

```nginx
stream {
    upstream upstream_servers {
      server 127.0.0.1:25378;
      server 127.0.0.1:25379 backup;
    }
    server {
        listen 33000;
        proxy_connect_timeout 1s;
        proxy_timeout 1m;
        proxy_pass upstream_servers;
    }
}
```

检验配置

```nginx
nginx -t
nginx -s reload
```

# 安装配置haproxy服务

https://github.com/slcnx/haproxy-windows

1. 克隆到本地计算机

2. 在目录中准备haproxy配置文件

   ```haproxy
   global
       maxconn 100 
       spread-checks 2
       daemon               
   defaults
       timeout connect 10s             
       timeout client 1m       
       timeout server 1m   
   
   listen stats    
     bind :1001
     mode http
     stats enable
     log global 
     stats hide-version
     #stats refresh 3s
     stats uri /haproxy
     stats realm HAPorxy\ Stats\ Page
     stats auth admin:123456
     stats admin if TRUE
   
   
   listen http
     mode tcp
     bind *:1000
   
     server local_25378 127.0.0.1:25378  check
     server local_25379 127.0.0.1:25379  check
     server local_25380 127.0.0.1:25380  check
   ```

   

3. 使用nssm，配置haproxy为一个服务

   <strong style="color: red;">cmd 管理员模式下使用</strong>

   ```bat
   nssm install haproxy "D:\Program Files\haproxy-windows\haproxy.exe" -f """D:\Program Files\haproxy-windows\config.conf"""
   nssm set haproxy AppStdout "D:\Program Files\haproxy-windows\haproxy-stdout.log"
   nssm set haproxy AppStderr "D:\Program Files\haproxy-windows\haproxy-stdout.log"
   ```

   > nssm识别带空格的命令没有问题，但是对带空格的参数就有一点捉襟见肘了，所以需要`"""` 来引用每一个带空白的选项，例如配置文件名`D:\Program Files\haproxy-windows\config.conf`, 需要传递为`"""D:\Program Files\haproxy-windows\config.conf"""`

5. sstap管理页面: `localhost:1001/haproxy`

# 配置服务

http://nginx.org/en/docs/windows.html

官方说明中，只有启动nginx, 说日后会增加nginx配置为windows一个服务。

通过搜索, stackoverflow有一个回答： https://stackoverflow.com/questions/40846356/run-nginx-as-windows-service

https://nssm.cc/ 一个开源的工具，帮你管理服务。

Just stumbled here and managed to get things working with this free open source alternative: https://nssm.cc/

> NSSM is a service helper program similar to srvany and cygrunsrv.  It can 
> start any application as an NT service and will restart the service if it 
> fails for any reason.

It basically is just a GUI to help you create a service. Steps I used:

1. Download NGinx (http://nginx.org/en/download.html) and uzip to C:\foobar\nginx
2. Download nssm (https://nssm.cc/)
3. Run "nssm install nginx" from the command line
4. In NSSM gui do the following:
5. On the application tab: set path to C:\foobar\nginx\nginx.exe, set startup directory to C:\foorbar\nginx
6. On the I/O tab type "start nginx" on the Input slow. Optionally set C:\foobar\nginx\logs\service.out.log and C:\foobar\nginx\logs\service.err.log in the output and error slots.
7. Click "install service". Go to services, start "nginx". Hit [http://localhost:80](http://localhost/) and you should get the nginx logon. Turn off the service, disable browser cache and refresh, screen should now fail to load.

# 使用代理

- Linux

  ```bash
  root@d7a59c10a63e:~# export https_proxy=http://192.168.1.222:33000
  root@d7a59c10a63e:~# curl https://ifconfig.me
  14.199.5.123
  ```

- Windows

  1. 安装sstap 1.9.7
  2. 添加http代理





