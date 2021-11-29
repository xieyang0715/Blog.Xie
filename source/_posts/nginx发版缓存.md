---
title: nginx发版缓存
date: 2021-08-18 08:53:43
tags:
- nginx
---

# 前言

nginx发版之后，需要手工刷新缓存



<!--more-->
# 配置

在nginx配置中添加以下高亮行

通过location匹配优先级：= ^~ ~ ~*

- = 精确匹配
- ^~ 前缀匹配
- ~ 大小写敏感
- ~* 大小写敏感
- 长前缀匹配

```diff
server {
    listen       80 default_server;
    server_name  localhost;
    charset utf-8; 
    root   /apps/nginx/html;
    access_log logs/access.log access_json;
    error_log  logs/error.log  error;

    location / {
        index  index.html index.htm;
       # hello 
    }
+    location = /index.html {
+        add_header Cache-Control "private, no-store, no-cache, must-revalidate, proxy-revalidate";
+    }

    location /nginx_status {
        stub_status;
        allow 127.0.0.1;
        deny all;
    }

    error_page  404              /index.html;
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
    }
}
```

> 这样index.html优先走
