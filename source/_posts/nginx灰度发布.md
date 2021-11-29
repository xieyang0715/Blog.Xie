---
title: nginx灰度发布
date: 2021-03-12 01:53:53
tags:
- nginx
- 生产小经验
- 个人日记
---



# 如何发布不影响生产用户？
<!--more-->

基于header控制访问不同的主机, 这样即使在公司内部访问相同app, 加了首部和不加首部都不一样的环境

一个新版本在预发测试OK之后，在jenkins之上发版至灰度的流程：

- [x] 点发到灰度
- [x] 构建出的jar包，直接发布至灰度环境，只是加载生产环境的配置文件
- [x] app端打包生产环境的app, 里面有请求每个URL时添加canary=always首部，这样所有流量走了灰度环境。

## 全局变量

入口

```diff
# 匹配$http_canary, 表示canary首部的值
# 如果always表示group变量的值为http_request_grayscale_release
# 如果为never表示group变量的值为http_requests
# 如果值为非always非nerver，表示group变量的值为http_requests
+map $http_canary $group {
+    '~always' http_request_grayscale_release;
+    '~never' http_requests;
+    default http_requests;
+}


upstream http_requests {
        server 172.16.0.221:30024 weight=1 fail_timeout=5s max_fails=3;
        server 172.16.0.222:30024 weight=1 fail_timeout=5s max_fails=3;
        server 172.16.0.223:30024 weight=1 fail_timeout=5s max_fails=3;
        server 172.16.0.224:30024 weight=1 fail_timeout=5s max_fails=3;
        server 172.16.0.225:30024 weight=1 fail_timeout=5s max_fails=3;
}



+upstream http_request_grayscale_release { # 是生产的其中一台主机，生产环境连接mongo,mysql走内网放在外网上连不上
+        server  172.16.0.221:30062 weight=1 fail_timeout=5s max_fails=3;
+}


server {
    server_name live.youninyouqu.com;
    listen 80 ;
    location / {
    	rewrite ^(.*)$ https://$host$1 last;
    }
}
server {
        listen 443 ssl http2 ;
        server_name live.youninyouqu.com;
        client_max_body_size 100M;
        proxy_set_header   Host    $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;

        ssl_certificate     /usr/local/nginx/certs/live-ssl/5310252_live.youninyouqu.com.pem;
        ssl_certificate_key /usr/local/nginx/certs/live-ssl/5310252_live.youninyouqu.com.key;
        ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;

        location / {
+             proxy_pass  http://$group;
        }
}


```

## 仅中location中

有些流量太细了，所以临时走内部nginx

```bash
    # 任务中心
    location /taskcenter {
             set $group weizhixiu-nginx-taskcenter-service.weizhixiu.svc.weizhixiu.local;
             if ($http_canary ~ "always") {
                set $group drayscale-nginx-taskcenter-service.drayscale.svc.weizhixiu.local;
             } 
             proxy_pass  http://$group;
    }
```



```bash
[root@izpxkqmt2zlej4z ~]# /usr/local/nginx/sbin/nginx -t
nginx: the configuration file /usr/local/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful
[root@izpxkqmt2zlej4z ~]# /usr/local/nginx/sbin/nginx -s reload
```

# 不加首部请求

![image-20210331141410581](http://myapp.img.mykernel.cn/image-20210331141410581.png)

# 添加awlays首部请求

结果到灰度主机

![image-20210331141445564](http://myapp.img.mykernel.cn/image-20210331141445564.png)

# 添加nerver首部请求

走生产

![image-20210331141544174](http://myapp.img.mykernel.cn/image-20210331141544174.png)

# 其他首部

全部走生产

![image-20210331141619212](http://myapp.img.mykernel.cn/image-20210331141619212.png)

# 发布完成之后

关闭灰度环境

```bash
map $http_canary $group {
#    '~always' http_request_grayscale_release;
    '~never' http_requests;
```

