---
title: kubernetes中api接口慢排查
date: 2021-03-04 03:48:11
tags:
- 个人日记
---

今天当把nginx-controller副本数量提升成2个时，前端在访问网页过程中就出现了时而慢，时而快的响应。

# 网页请求
<!--more-->

这个是网页请求F12的结果，可以看到20.14s

![image-20210304114944246](http://myapp.img.mykernel.cn/image-20210304114944246.png)



## kibana查询

![image-20210310110540785](http://myapp.img.mykernel.cn/image-20210310110540785.png)



# postman请求

可以看到右下角也是20.14s

![image-20210304115033509](http://myapp.img.mykernel.cn/image-20210304115033509.png)

现在postman也异常，所以直接上nginx-controller请求本地，验证 本地环境到nginx的网络问题

导出curl命令

![image-20210304115417235](http://myapp.img.mykernel.cn/image-20210304115417235.png)

![image-20210304115421771](http://myapp.img.mykernel.cn/image-20210304115421771.png)

因为在pod请求本地使用localhost, 所以请求方法有2种

- [x] 添加hosts解析，请求域名
- [x] 直接请求localhost, 添加headers Host, 这个header就是通过这个header决定使用哪个虚拟主机

```bash
curl --location --request POST --headers 'Host:test.youwoyouqu.io' 'http://localhost/admin/video/verifyPass' \
--header 'Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ3YW5nc2hlbiIsImNyZWF0ZWQiOjE2MTQ4Mjc0MTE1NjMsImV4cCI6MTYxNDkxMzgxMX0.vm3EKRZF7Vn1DtRGKSqeRtHSC9rh4Och4SUTHI_Zenc' \
--header 'Content-Type: application/json' \
--data-raw '{"pageIndex":1,"pageSize":10}'
```

# 入口容器请求

容器中直接请求nginx本地，因为是2个副本，所以2个pod均要测试

容器1

![img](http://myapp.img.mykernel.cn/wps1.jpg)

容器2

![image-20210304115721611](http://myapp.img.mykernel.cn/image-20210304115721611.png)

可以查看是pod2慢





# Nginx日志定位

容器2到后端慢

通过日志定位到这个IP的8080响应

![image-20210304115756243](http://myapp.img.mykernel.cn/image-20210304115756243.png)

> upstreamtime proxy到后端一次事务的时间
>
> reponsetime    client请求nginx一次事务的时间

所以确定是nginx连接后端慢，查看后端的地址，通过svc知道哪个服务

![image-20210304115809051](http://myapp.img.mykernel.cn/image-20210304115809051.png)

## kibana查询

​     ![image-20210310112138372](http://myapp.img.mykernel.cn/image-20210310112138372.png)

> upstreamhost后端哪个service
>
> host.hostname即哪个Pod名

![image-20210310110924250](http://myapp.img.mykernel.cn/image-20210310110924250.png)

先获取这个接口的curl命令

![image-20210310111505540](http://myapp.img.mykernel.cn/image-20210310111505540.png)

> 保存日志，不缓存，测试登陆，复制curl

postman请求，先导入curl

![image-20210310111657979](http://myapp.img.mykernel.cn/image-20210310111657979.png)

导入之后请求, 发现多次请求时间非常小

![image-20210310111730771](http://myapp.img.mykernel.cn/image-20210310111730771.png)

body均有

![image-20210310111739461](http://myapp.img.mykernel.cn/image-20210310111739461.png)

进入容器，需要命令前加`time <COMMAND>` 多次请求正常

![image-20210310111856256](http://myapp.img.mykernel.cn/image-20210310111856256.png)





## nginx添加记录POST的request_body （only测试环境）

生产打开，日志量太多了



所以本地环境分析接口，需要打开POST方法的request_body， 即nginx日志加入以下字段

需要token才可以认证

```diff
root@master01:/data/weizhixiu# cat dockerfile/nginx-baseimage-dockerfile/nginx.conf
worker_processes  auto;
worker_cpu_affinity auto;
error_log  logs/error.log  error;
events {
		worker_connections  100000;
		use epoll;
		accept_mutex on;
		multi_accept on;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    log_format access_json escape=json '{"@timestamp":"$time_iso8601",'
     '"host":"$server_addr",'
     '"clientip":"$remote_addr",'
     '"size":$body_bytes_sent,'
     '"responsetime":$request_time,'
+     # http://nginx.org/en/docs/http/ngx_http_upstream_module.html#var_upstream_response_time
+     # https://juejin.cn/post/6844903887757901832
+     '"upstream_addr":"$upstream_addr",'
+     '"upstream_bytes_received":"$upstream_bytes_received",'
+     '"upstream_bytes_sent":"$upstream_bytes_sent",'
+     '"upstream_connect_time":"$upstream_connect_time",'
+     '"upstream_header_time":"$upstream_header_time",'
+     '"upstreamtime":"$upstream_response_time",'
+     '"upstreamhost":"$upstream_addr",'

     '"http_host":"$host",'
     '"url":"$uri",'
     '"domain":"$host",'
     '"xff":"$http_x_forwarded_for",'
     '"tcp-xff":"$proxy_protocol_addr",'
     '"referer":"$http_referer",'
     '"user-agent":"$http_user_agent",'
+     '"request-method":"$request",'
+     '"request-body":"$request_body",'
+     '"token":"$http_Authorization",'

     '"status":"$status"}';

    access_log  /apps/nginx/logs/access.log  access_json;  

    #aio             on;
    #directio 4m; 
    sendfile        on;  
    tcp_nopush     on;   
    keepalive_timeout  65 90; 
    keepalive_requests 1000;  
    keepalive_disable msie6; 
    tcp_nodelay off;      
	gzip on; 
	gzip_comp_level 5;
	gzip_min_length 1k; 
	gzip_types text/plain application/javascript application/x-javascript text/cssapplication/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;
	gzip_vary on;
    client_max_body_size 1024m; 
    open_file_cache max=10000 inactive=60s; 
    open_file_cache_valid 60s; 
    open_file_cache_min_uses 5; 
    open_file_cache_errors on; 
    server_tokens off; 
    server {
        listen       80;
        server_name  localhost;
        location / {
            root   html;
            index  index.html index.htm;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
    include /apps/nginx/conf/conf.d/*.conf;
}

```

![img](http://myapp.img.mykernel.cn/16be1235edca7f93)

> - 程序真正的运行时间 = $upstream_header_time - $upstream_connect_time
> - $request_time 中包含了数据返回时间
> - $request_time 中包含了日志打印的时间



```bash
root@master01:/data/weizhixiu/dockerfile/nginx-baseimage-dockerfile# bash build.sh 
```

重新发版nginx

```bash
root@master01:/data/weizhixiu/stage-builder/admin-web-nginx# bash build.sh 202103101127
```

更新kibana索引

![image-20210310113517227](http://myapp.img.mykernel.cn/image-20210310113517227.png)

观察新的kibana日志是否有请求体, token, 请求方法

![image-20210310135814862](http://myapp.img.mykernel.cn/image-20210310135814862.png)



之后在观察kibana分析url

```bash
# 响应application/json没有压缩, nginx修正
gzip_types text/plain application/json application/javascript application/x-javascript text/cssapplication/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;
```



## 查看慢日志

```bash
# mysql
tail -f /data/mysql/slow.log

# 配置为
[mysqld]
slow_query_log=ON
log_output=FILE
long_query_time=1
slow_query_log_file=/data/mysql/slow.log
#log_queries_not_using_indexes=ON             #  加索引的也有必要性，所以不使用索引不是惟一检测标准               
log_error=/data/mysql/error.log
```

```bash
# redis
SLOWLOG LEN
SLOWLOG GET
```

## 部署soar-web分析慢查询

```bash
docker run --rm -d -p 5077:5077 becivells/soar-web
```



![image-20210310144036648](http://myapp.img.mykernel.cn/image-20210310144036648.png)

# nginx结合微服务排查问题

2个网页打开2个gateway的日志

2个网页打开第2个nginx pod

现在在nginx请求这个接口，可以发现后端一下有日志输出

> 说明 ：nginx -> 微服务1 正常

微服务1会调用微服务2，在其上，通过开发所述的方法请求，发现时间正常。

> 说明 ：微服务1 -> 微服务2 正常



现在就是微服务1内部问题，需要开发，更新版本，打印日志查看哪层慢了。



# 解决问题

因为开发在网关调用后端逻辑，什么批量或顺序的，修改了非常快。