---
title: nginx编译安装及配置优化
date: 2020-10-14 09:53:43
tags: 
- plan
- nginx
toc: true
---

# 1. nginx编译安装

## 1.1 依赖安装

```bash
yum install -y vim lrzsz tree screen psmisc lsof tcpdump wget ntpdate gcc gcc-c++ glibc glibc-devel pcre pcre-devel openssl openssl-devel systemd-devel net-tools iotop bc zip unzip zlib-devel bash-completion nfs-utils automake libxml2 libxml2-devel libxslt libxslt-devel perl perl-ExtUtils-Embed gd-devel GeoIP GeoIP-devel GeoIP-data
```

## 1.2 官方stable源码包下载

https://nginx.org/en/download.html

![image-20201013163557054](http://myapp.img.mykernel.cn/image-20201013163557054.png)

<!--more-->

## 1.3 编译安装

```bash
[root@cd-ch-pxr-centos-0-198 ~]# cd /usr/local/src
[root@cd-ch-pxr-centos-0-198 src]# wget https://nginx.org/download/nginx-1.18.0.tar.gz
[root@cd-ch-pxr-centos-0-198 src]# tar xvf nginx-1.18.0.tar.gz 
[root@cd-ch-pxr-centos-0-198 src]# cd nginx-1.18.0/
[root@cd-ch-pxr-centos-0-198 nginx-1.18.0]# ./configure --prefix=/apps/nginx \
--user=nginx \
--group=nginx \
--pid-path=/run/nginx.pid  \
--with-threads \
--with-file-aio \
--with-http_ssl_module \
--with-http_v2_module \
--with-http_realip_module \
--with-http_image_filter_module \
--with-http_geoip_module \
--with-http_gzip_static_module \
--with-http_auth_request_module \
--with-http_stub_status_module \
--with-http_perl_module \
--with-stream \
--with-stream_ssl_module \
--with-stream_realip_module \
--with-stream_geoip_module \
--with-pcre
[root@cd-ch-pxr-centos-0-198 nginx-1.18.0]# make -j `grep -c processor /proc/cpuinfo`
[root@cd-ch-pxr-centos-0-198 nginx-1.18.0]# make install
[root@cd-ch-pxr-centos-0-198 nginx-1.18.0]# groupadd -g 2020 nginx
[root@cd-ch-pxr-centos-0-198 nginx-1.18.0]# useradd -m -s /sbin/nologin -r -u 2020 -g nginx nginx
[root@cd-ch-pxr-centos-0-198 nginx-1.18.0]# id nginx
uid=2020(nginx) gid=2020(nginx) groups=2020(nginx)

[root@cd-ch-pxr-centos-0-198 nginx-1.18.0]# chown -R nginx.nginx /apps/nginx/

```

## 1.4 验证版本和编译参数

```bash
[root@cd-ch-pxr-centos-0-198 nginx-1.18.0]# /apps/nginx/sbin/nginx -V
nginx version: nginx/1.18.0
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-39) (GCC) 
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/apps/nginx --with-threads --with-file-aio --with-http_ssl_module --with-http_v2_module --with-http_realip_module --with-http_image_filter_module --with-http_geoip_module --with-http_gzip_static_module --with-http_auth_request_module --with-http_stub_status_module --with-http_perl_module --with-stream --with-stream_ssl_module --with-stream_realip_module --with-stream_geoip_module --with-pcre
```

## 1.5 访问编译安装的web

```bash
 /apps/nginx/sbin/nginx 
```

![image-20201013165102446](http://myapp.img.mykernel.cn/image-20201013165102446.png)

## 1.6 创建 nginx服务脚本

配置pid路径 `/apps/nginx/conf/nginx.conf`

```nginx
user  nginx; # 运行nginx的用户
pid        /run/nginx.pid; # nginx的MAINPID位置
```

配置`/usr/lib/systemd/system/nginx.service`

```ini
[Unit]
Description=The nginx HTTP and reverse proxy server
After=network.target remote-fs.target nss-lookup.target
[Service]
Type=forking
PIDFile=/run/nginx.pid # 这个PID，保存了MAINPID
# Nginx will fail to start if /run/nginx.pid already exists but has the wrong
# SELinux context. This might happen when running `nginx -t` from the cmdline.
# https://bugzilla.redhat.com/show_bug.cgi?id=1268621
ExecStartPre=/usr/bin/rm -f /run/nginx.pid # 删除pid
ExecStartPre=/apps/nginx/sbin/nginx -t     # 测试配置
ExecStart=/apps/nginx/sbin/nginx           # 启动
ExecReload=/bin/kill -s HUP $MAINPID       # 给主PID发送HUP信号 
KillSignal=SIGQUIT
TimeoutStopSec=5
KillMode=process
PrivateTmp=true
[Install]
WantedBy=multi-user.target
```

启动

```bash
[root@cd-ch-pxr-centos-0-198 nginx-1.18.0]# systemctl daemon-reload
[root@cd-ch-pxr-centos-0-198 nginx-1.18.0]# systemctl enable nginx
Created symlink from /etc/systemd/system/multi-user.target.wants/nginx.service to /usr/lib/systemd/system/nginx.service.
[root@cd-ch-pxr-centos-0-198 nginx-1.18.0]# systemctl start nginx
[root@cd-ch-pxr-centos-0-198 nginx-1.18.0]# systemctl status nginx
```

![image-20201013170102005](http://myapp.img.mykernel.cn/image-20201013170102005.png)



# 2. nginx优化

1. IP透传
2. 全栈SSL
3. 传输压缩、持久连接、缓存、自定义变量、下载服务、上传服务器、root/alias
4. ngx_http_upstream_module
5. location匹配
   - = 精确
   - ^~ 左前缀 
   - ~ 区分大小写        正则
   - ~* 不区分大小写  正则
   - /  request_uri 
6. 防盗链
7. rewrite  https://xuexb.github.io/learn-nginx/example/proxy_pass.html#%E9%87%8D%E5%86%99%E4%BB%A3%E7%90%86%E9%93%BE%E6%8E%A5-url-rewrite
   - redirect, permanent重新请求
   - last, 跳出当前location(不管后续的last/break), 从server {} 重新匹配
   - break, 从当前Location响应
8. tcp传输配置

> 一句话概述：nginx优化：进程绑定、连接数，日志（错误，访问），缓存（打开文件，代理缓存），状态页，模块（ssl, gzip, 隐藏版本，重写，代理，upstream, 限速，ip控制，上传下载)

```nginx
[root@cd-ch-pxr-centos-0-198 conf]# grep -v '^$' nginx.conf
user  nginx;                   # 运行nginx work的用户及组 user user [group]
worker_processes  auto;        # 工作进程数
worker_cpu_affinity auto;      # 工作进程与cpu绑定。 ps axo pid,cmd,psr,user | grep nginx
#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;
error_log  logs/error.log  error;
pid        /run/nginx.pid;
worker_priority 0;
worker_rlimit_nofile 10000; 
#ulimit 对单一程序的限制, 进程级别的。默认1024. nginx官方文档说nginx 2-4万并发，所以ulimit必须65535，而且上面的值要在2-4万之间或1万。
# 如果worker_rlimit_nofile < ulimit会存在nginx启动不了的情况
#/proc/sys/fs/file-max
#file-max是所有进程最大的文件数
daemon off; # docker
events {
    worker_connections  10000; # 单个进程最大并发, 总并发 = worker_connections * worker_processes
    use epoll; # 边缘触发, 哈希表，最大并发没有限制, mmap机制
    accept_mutex on; # 一个请求过来是否唤醒所有进程;  on为防止被同时唤醒 
    multi_accept on; # 每个工作作进程可以同时接受多个新的网络连接; on为一个进程同一时刻接受多个连接;
}
http {
    include       mime.types; #导入支持的文件类型
    default_type  application/octet-stream; # 默认类型, 当文件不属于mime.types时，默认识别的类型。 application/octet-stream会自动下载文件.
    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';
    #access_log  logs/access.log  main;
    log_format access_json escape=json '{"@timestamp":"$time_iso8601",'
     '"host":"$server_addr",'
     '"clientip":"$remote_addr",'
     '"size":$body_bytes_sent,'
     '"responsetime":$request_time,'
     '"upstreamtime":"$upstream_response_time",'
     '"upstreamhost":"$upstream_addr",'
     '"http_host":"$host",'
     '"url":"$uri",'
     '"domain":"$host",'
     '"xff":"$http_x_forwarded_for",'
     '"tcp-xff":"$proxy_protocol_addr",'
     '"referer":"$http_referer",'
     '"user-agent":"$http_user_agent",'
     '"status":"$status"}';

    access_log  /var/log/nginx/access.log  access_json;   
    aio on; # --with-file-aio
    directio 4m; # 在aio打开情况下， 当小于4m直接使用sendfile. 大于4m使用directio;
    sendfile        on;  # 内核空间直接封装响应
    tcp_nopush     on;   # 打开sendfile，合并请求统一发送。 略有延迟但是减少封装次数, 减少开销
    #keepalive_timeout  0; # 0表示不打开会话保持
    keepalive_timeout  65 90; # 前面是实际保持时长, 后者是用户所见保持时长
    keepalive_requests 1000;  # 长连接之上处理多少个请求或达到65s就立即断开。
    keepalive_disable msie6; # 禁用ie6不建立长连接;
    tcp_nodelay off;      # keepalived模式下的连接是否启TCP_NODELAY选项，当为off时，延迟0.2s发送，默认On时，不延迟发送，立即发送相应报文。
	## 压缩
	gzip on; # cp  /var/log/messages  /apps/nginx/html/message.html; chown nginx.nginx /apps/nginx/html/message.html
	gzip_comp_level 5;
	gzip_min_length 1k; # 至少1k文件才压缩
	gzip_types text/plain application/javascript application/x-javascript text/cssapplication/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;
	gzip_vary on;
	##
    # 上传服务器优化
    client_max_body_size 1024m; #设置允许客⼾端上传单个⽂件的最⼤值，默认值为1m
    #
    open_file_cache max=10000 inactive=60s; #最大缓存10000个文件，非活动数据超时时长60s
    open_file_cache_valid 60s; #每间隔60s检查⼀下缓存数据有效性
    open_file_cache_min_uses 5; #60秒内⾄少被命中访问5次才被标记为活动数据
    open_file_cache_errors on; #缓存错误信息 
    server_tokens off; #隐藏Nginx server版本。 指定服务名，通过编译编译目录的src/http/ngx_http_header_filter_module.c 49 static u_char ngx_http_server_string[] = "Server: nginx" CRLF;
    server {
        listen       80;
        server_name  localhost;
        charset utf-8; # 字符集配置
        #access_log  logs/host.access.log  main;
        location / {
            root   html;
            index  index.html index.htm;
            #try_files $uri $uri/index.html $uri.html /about/default.html; # 查找文件的顺序, 如果没有找到返回/about/default.html文件
            #try_files $uri $uri/index.html $uri.html =489;  # 没有找到返回489状态码
        }
        location /download {
            autoindex on; #自动索引功能
            autoindex_exact_size on; # 精确大小(单位bytes），off非精确, 显示gb, mb, kb）
            autoindex_localtime on; #显示本机时间, 非GMT(格林威治时间)
            root /apps/nginx/html/; #install -dv /apps/nginx/html/download;   cp /var/log/messages  /apps/nginx/html/download
        }
        location /upload {
            root /apps/nginx/html/;
            index index.html;
            # 始终允许HEAD方法
            limit_except GET {
                allow 172.18.200.101;
                deny all;
            }
        }
            # 状态页配置
            # Active connections: 291 同时在线用户数
            # reading 当前状态，正在读client请求报文首部的连接的连接数。 越高，服务压力越大 
            # Writing：当前状态，正在向客⼾端发送响应报⽂过程中的连接数。
        location /nginx_status {
            stub_status;
            allow 192.168.0.0/16;
            allow 127.0.0.1;
            deny all;
        }
        
        location /images {
            # referer为以下时允许访问当前 locahost/images
            # 因为这个地址过来全是引用图片, 添加防盗链
            valid_referers none blocked server_names
                *.mykernel.cn
                ~\.google\.
                ~\.bing\.com
                ~\.baidu\.(cn|com);
            if ($invalid_referer) {
                return 502;
            }
            # break, last都是本站返回
            ## break 只能在当前location返回 $server_name$request_uri
            ## last  第一个last直接跳出location, 然后在server {} 中从上到下匹配新的$request_uri前缀
            rewrite /(.*)$ http://myapp.img.qiniu.mykernel.cn/$1 permanent;
        }
        

        
        #error_page  404              /404.html;
        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
		
        
        ################## ngx_fastcgi_http_module ##################
        # php部署方式    
        # 1. nginx和php-fpm在同一个主机，前端接入nginx，可以知道nginx状态，但是php-fpm挂了呢？
        	# 所以需要在nginx/php-fpm主机上写检测脚本，php中断了就停nginx,前端就自动摘了nginx.
    		# 或者前端只检测php-fpm的端口. check port 9000, 但是nginx挂了？需要检测nginx.
        # 2. php不同主机：lvs/haproxy tcp模式 反代 多个php. nginx从统一的入口来请求php, php的在线检测交给lvs/haproxy.
        # 方式1: root + 变量
        location ~ \.php$ {
            root /scripts
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            # 表示php-fpm不知道php文件在哪，需要nginx通过SCRIPT_FILE传递给php. 后端就是php脚本的位置。
            # 配置Php-fpm: 上传的文件用户和组和worker进程的用户一样，不然ngx 403. ping页，状态页。 php-fpm的工作模型 master - work, 每个worker处理php请求。 
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name; 
            # param SERVER_PORT $server_port这些通过include param指令引入，会传递给php, 默认就有，不传递可能运行出错 
            include        fastcgi_params;
            # 调用
            fastcgi_cache fastcgicache;   
			# Haproxy 7层检查时，会有日志，关闭7层检测的日志
            if ( $request_method = 'HEAD' ) {
    			access_log  off;
			}
        }
        
        # 方式2: 直接路径
        location ~ \.php$ {
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
            include        fastcgi_params;
            # 调用
            fastcgi_cache fastcgicache;
        }
        
        ## php状态页
        location ~ /(php-status|ping) {
			rewrite ^/php-status(.*)$ /status$2 break;
            root           /apps/nginx/html;
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi_params;
			# 访问控制
            allow 172.16.0.0/16;
			allow 10.21.0.0/16;
			allow 192.168.101.0/24;
			allow 127.0.0.1;
			deny all;
            # 状态页不记录日志
			access_log  off;
        }
		###
        #    ngx调用Php时，有缓存的话，就调用缓存，不转发了‘
    	#1. 配置长连接
    	#2. 不同响应码缓存时长
    	#3. 隐藏字段，
    	#4. 响应字段
        #
        
    }
	# fastcgi缓存
    fastcgi_cache_key $request_uri;
    fastcgi_cache_path /var/cache/nginx/fastcgi_cache levels=1:2:2 keys_zone=fastcgicache:20m inactive=120s max_size=1g;
    #fastcgi_cache_stale
    fastcgi_cache_valid 200 302 301 10m;
    fastcgi_cache_valid any 1m;
    fastcgi_headers_hash_bucket_size 512;
    fastcgi_headers_hash_max_size 512;
    fastcgi_set_header  X-Forwarded-For $remote_addr;


    
    server {
        listen 80;
        listen 8080;
        listen somename:8080;
        
        location /proxy {
            #proxy_pass http://test.magedu.com:80; # 此proxy_pass没有高级功能
            proxy_pass http://upstream_servers; # 此proxy_pass有高级功能
            # 调用
            proxy_cache proxycache;
            
        } 
    }
    
    # 以指定key缓存。默认$scheme$proxy_host$request_uri;
    proxy_cache_key $request_uri;

    # 定义对特定响应码的响应内容的缓存时⻓.
    # any表示除了200，302，301缓存1m.
    proxy_cache_valid 200 302 301 10m; #指定的状态码返回的数据缓存多⻓时间
    proxy_cache_valid any 1m;

    # 后端哪些不可用，就使用缓存？
    #  一般不使用，
    # 1. 有监控系统，后端挂了还不知道，监控多差。
    # 2. ngx_http_upstream_module 模块定义的分组有健康状态检测，后端不可用，直接可以自动下线。
    #proxy_cache_stale 


    ## client --> proxy ##
    # 设置nginx提供代理服务的HTTP协议的版本
    proxy_http_version 1.0; 
    # 客户端发起请求后，主动断开。
    # 默认off表示nginx会中断客户端请求返回499,并记录日志。 
    # on表示nginx会忽略客户中断，一直等后端响应。
    # 大量的499表示后端响应慢，用户等的不耐烦了。后端太慢
    proxy_ignore_client_abort off;


    ### proxy --> realserver ###
    # 建立连接。建立失败503
    # 建立连接后。如果后端mysql查询时间太长, 默认值就会让nginx给用户响应502.
    proxy_connect_timeout 60s;
    ## 读请求的超时
    proxy_read_timeout 60s;
    ## 写请求的超时
    proxy_send_timeout 60s;
    ################ realserver -> proxy -> client，F12查看response header  ################
    # 默认隐藏后端传递给客户端 “Date”, “Server”, “X-Pad”, and “X-Accel-...” 首部
    # 也可以自定义隐藏后端给客户端响应的首部。
    #proxy_hide_header field;
    # 许可传递后端的首部，从代理服务器传递给客户端
    #proxy_pass_header field;
    # 缓存是否命令(通常添加上表示Proxy_cache是否生效) 
    add_header X-cache $upstream_cache_status;

    ################ client -> proxy -> realserver  ################
    # 将client HTTP请求体转发给realserver。默认on. http/server或location
    proxy_pass_request_body on;
    # 将client HTTP请求首部转发给realserver。 默认on. http/server或location
    proxy_pass_request_headers on;
    # 向后端传递自定义首部。真实IP
    # 后端apache: /etc/httpd/conf/httpd.conf      LogFormat "%{X-Forwarded-For}i %h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{UserAgent}i\"" combined  其中%{}i 表示引用HTTP首部
    # 后端nginx: /apps/nginx/conf/nginx.conf "$http_x_forwarded_for"' #默认⽇志格式就有此配置 http_开头表示引用HTTP首部
	
    
    proxy_set_header X-Real-IP $remote_addr;
    
    proxy_set_header X-Forwarded-For $remote_addr; # 第一层代理
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; # 不是第1层代理

    # 后端nginx接收到请求会查找请求首部的Host来定位哪个虚拟主机， 如果不设定，后端不知道对哪个虚拟主机请求。
    proxy_set_header Host            $host; 
    
    ## 代理模式下，大量转发时，Ngx直接读缓存，直接转发。
    # 字节,当设定 proxy_set_header或hide_header，用于保存http报文header的hash表上限。
    proxy_headers_hash_bucket_size 128;
    #  设定 hash bucket size最大空间。字节
    proxy_headers_hash_max_size 512;
    # 保存server_name的hash表及上限。字节
    server_names_hash_bucket_size 512;
    server_names_hash_max_size 512;

    
    # 高级功能, 负载均衡算法, 心跳检测
    ##   1. proxy_pass到一个组
  	##   2. 多个后端有检测功能
    ##   3. 高可用，所有的挂了，还有backup, 通常也叫sorry server.
    upstream upstream_servers {
         #server 
        #unix/域名/ip 
        # weight(高配置cpu/ram高权限) 
        # backup(高可用) 
        # maxconns(后端最大连接，一般不 定义) 
        # fail_timeout(服务器从可用到不可用，检测时间，默认10s。) 
        # max_fails 3(在fail_timeout即10s内出现3次失败就将这个server标记为失败。) 
        # down(不用他，假如这个server即将下线，还未下线时，将标记down) 
        # resolve (server定义A记录变化时，自动应用新ip, 不重启nginx)
        
        # 算法
        # 源地址hash
        ## ip_hash;
        ## hash $remote_addr consistent;
        server 192.168.7.103:80 weight=1 fail_timeout=5s max_fails=3;
    }
 	# proxy_pass的缓存 "线上代码上线, 就需要清理缓存"
    # /var/cache/nginx/proxy_cache 缓存位置: install -dv -o nginx -g nginx /var/cache/nginx/proxy_cache
    # levels=1:2:2 缓存目录。1表示16进制0-f. 将文件md5从后向前取，第1个为1级子目录，第2-3为2级目录，第4-5为三级目录。加速查找
    # proxycache:20m 缓存区域名，大小（主要用来存放key和metadata）
    # inactive 缓存有效期
    # max_size 占用磁盘空间大小，在inactive时间内，看会有多少缓存。
    proxy_cache_path /var/cache/nginx/proxy_cache levels=1:2:2 keys_zone=proxycache:20m inactive=120s max_size=1g;


############## 配置SSL ############
server {
      listen 80;
      listen 443 ssl;
      server_name www.mykernel.cn mykernel.cn;
      # 证书
      ssl_certificate /apps/nginx/certs/4899578_mykernel.cn.pem;
      # 私钥
      ssl_certificate_key /apps/nginx/certs/4899578_mykernel.cn.key;
      # ssl缓存
      ssl_session_cache shared:sslcache:20m;
      # 缓存失效
      ssl_session_timeout 10m;

      location / {
          if ( $scheme ~* "http$") {
            rewrite ^(.*)$ https://www.mykernel.cn$1 permanent;
          }
          root /blog/public;
      }
}

}

stream {
    upstream redis_server {
        hash $remote_addr consistent;
        server 192.168.7.104:6379 max_fails=3 fail_timeout=30s;
    }
    server {
        listen 192.168.7.102:6379;
        proxy_connect_timeout 3s;
        proxy_timeout 3s;
        proxy_pass redis_server;
    }
    
    upstream mysql_salve_server {
        least_conn;
        server 192.168.7.104:3306 max_fails=3 fail_timeout=30s;
    }
    server {
        listen 192.168.7.102:3306;
        # 建立连接
        proxy_connect_timeout 3s;
        # 转发超时
        proxy_timeout 3s;
        proxy_pass mysql_salve_server;
    }
    
}

[root@cd-ch-pxr-centos-0-198 conf]# 

```





# 优化补充

https://github.com/qiurunze123/miaosha/blob/master/docs/ngnix-good.md

```nginx
        #用于清除缓存
        location ~ /purge(/.*)
        {
            allow 127.0.0.1;
            allow 192.168.220.133;
            deny all;
            proxy_cache_purge cache_one $host$1$is_args$args;
        }
        # 静态文件加缓存
        location ~ .*\.(gif|jpg|jpeg|png|bmp|swf|js|css|html|htm)?$
        {
            expires 1d;
            proxy_cache cache_one;
            proxy_cache_valid 200 304 1d;
            proxy_cache_valid any 1m;
            proxy_cache_key $host$uri$is_args$args;
            proxy_pass http://server_pool;
        }
```

```nginx
    #反向代理服务器集群
    upstream server_pool{
        server localhost:8080 weight=1 max_fails=2 fail_timeout=30s;
        server localhost:8081 weight=1 max_fails=2 fail_timeout=30s; 
        keepalive 200; # 最大的空闲的长连接数 
    }

		location / {
            #长连接
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            #Tomcat获取真实用户ip
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $remote_addr;
            proxy_set_header X-Forwarded-Proto  $scheme;
            proxy_pass http://server_pool;
        }
```



#  F12分析

![image-20201210104751457](http://myapp.img.mykernel.cn/image-20201210104751457.png)

![image-20201210104823419](http://myapp.img.mykernel.cn/image-20201210104823419.png)

Server: nginx. 表示`server_tokens off`指令生效

Connection: keep-alive  Keep-Alive: timeout=90  表示`keepalive_timeout  65 90;`指令生效

Vary: Accept-Encoding 表示`gzip on`生效

X-cache: HIT 表示`add_header X-cache $upstream_cache_status`生效

 