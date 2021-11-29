---
title: php8 状态页输出
date: 2020-12-14 05:00:52
tags: 日常挖掘
---









今天在配置haproxy检测后端同时安装nginx+php的主机时，因为4层检测只能有一个条件，只能检测一个端口。但是7层检测可以检测多个URL，返回2XX,3XX均为正常。想着nginx直接访问主机。php-fpm访问ping页面或状态页面，所以制作如下的状态页配置过程。

编译安装php 8之后配置状态页和ping页面

# 1. php-fpm的status.html

```bash
# find /usr/local/src/php-8.0.0/ -name "status.html"
/usr/local/src/php-8.0.0/sapi/fpm/status.html

# 复制到nginx的位置
# cp /usr/local/src/php-8.0.0/sapi/fpm/status.html /apps/nginx/html/status.html 
```

<!--more-->

# 2.配置php-fpm状态页

```bash
[root@chengdu-huayang-linux48-nginx2-101-2 php8]# cat /apps/php8/etc/php-fpm.d/www.conf
[www]
user = nginx
group = nginx
listen = 0.0.0.0:9000
pm = dynamic
pm.max_children = 1000
pm.start_servers = 5
pm.min_spare_servers = 3
pm.max_spare_servers = 25
pm.status_path = /status
ping.path = /ping
ping.response = pong
access.log = var/log/$pool.access.log              
php_flag[display_errors] = on
php_admin_value[error_log] = var/log/fpm-php.www.log
php_admin_flag[log_errors] = on
php_admin_value[memory_limit] = 32M
rlimit_files = 65535
```



# 3. 配置nginx

```bash
server {
        location ~ \.php$ { # 动态网页
            root           /apps/nginx/html;
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }
        location ~ /(php-status|ping) { # 状态页
			rewrite ^/php-status(.*)$ /status$2 break; # break当前url返回
            root           /apps/nginx/html;
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi_params;
			allow 172.16.0.0/16; # 安全
			allow 10.21.0.0/16;
			allow 192.168.101.0/24;
			allow 127.0.0.1;
			deny all;
        }
}
```

输出

http://10.21.101.2/php-status

```
pool:                 www
process manager:      dynamic
start time:           14/Dec/2020:13:06:09 +0800
start since:          55
accepted conn:        1
listen queue:         0
max listen queue:     0
listen queue len:     511
idle processes:       4
active processes:     1
total processes:      5
max active processes: 1
max children reached: 0
slow requests:        0
```

http://10.21.101.2/ping

pong

