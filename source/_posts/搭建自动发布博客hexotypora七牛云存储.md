---
title: '搭建自动发布博客: hexo + typora + 七牛云存储'
date: 2020-09-27 10:33:55
tags: hexo
toc: true
---




![image-20201219073407682](http://myapp.img.mykernel.cn/image-20201219073407682.png)

# 下载代码

https://code.aliyun.com/1062670898/blogs.git

![image-20200927105816110](http://myapp.img.mykernel.cn/image-20200927105816110.png)

<!--more-->

# 镜像分层

1. centos镜像: 基于centos 7,安装基础包。

2. nodejs镜像: 安装nodejs。
3. hexo镜像: 安装hexo, 添加镜像配置为cnpm
   手动commit镜像：基于hexo, 添加公钥
4. client镜像:  registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto:2020092201
   client镜像是commit后添加了inotify脚本完成监听source目录变化, 而后先pull代码, 再提交
5. server镜像:  registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:2020092201
   基于client镜像, 里面运行一个flask, 当阿里云收到push请求, 就会发起请求. flask运行命令git pull, hexo g --config=hexo-config.yaml
   里面一个nginx就直接展示生成的html

# 配置站点
## 阿里云添加博客仓库
## 构建容器
构建基础容器

```bash
cd 1.centos/      ## 基础cenos
bash build-command.sh
cd ../2.nodejs    ## 基础nodejs
bash build-command.sh
cd ../3.hexo      ## 生成一个新站点, hexo配置和主题配置在此步骤进行
bash build-command.sh
```

运行站点, 做配置
```bash
docker run --rm -it --name hexo1 hexo:2020092201 bash
```
##  阿里云添加公钥
```
ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa -P ''
cat /root/.ssh/id_rsa.pub
```
## 推送网站

**注意，运行此步骤前一定要先备份博客**

```bash
git config --global user.name "loneiei"
git config --global user.email "1062670898@qq.com"
git init
git remote add origin git@code.aliyun.com:1062670898/2020-09-16-myblog.git
git pull origin master # 推送前一定要先拉code.aliyun.com上的代码
git add .
git commit -m 'init'
git push -u origin master # 不加-f, 避免覆盖git仓库的代码
```
阿里云查看博客已经生成


## 生成镜像
```bash
docker commit -p hexo1 hexo:2020092202
```
## 将制作的带公钥的镜像推到阿里云的私有仓库
```bash
docker login --username=loneiei registry.cn-hangzhou.aliyuncs.com
docker tag hexo:2020092202 registry.cn-hangzhou.aliyuncs.com/slcnx/2020-09-16-myblog:2020092202
docker push registry.cn-hangzhou.aliyuncs.com/slcnx/2020-09-16-myblog:2020092202
```

##  制作开发环境的启动镜像
作用：启动时，自动拉代码. 运行过程中可以根据source目录变化, 来将所有文件推送到aliyun


修改Dockerfile的FROM指令, 和build-command.sh
```Dockerfile
FROM registry.cn-hangzhou.aliyuncs.com/slcnx/2020-09-16-myblog:2020092202
```
```bash
cd 4.aliyun ## 添加一个脚本, 在编辑文件后触发自动推送. 在文件修改前pull代码.
bash build-command.sh
```
测试自动拉、推代码
```bash
docker run -d --restart=always -it --name writer -v /source/:/apps/source registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto:2020101403 

```
添加一个文章
```bash
docker exec writer addarticle 'hello'
INFO  Created: /apps/source/_posts/hello.md
INFO  Total precache size is about 0 B for 0 resources.
source/
`-- _posts
    `-- hello.md

1 directory, 1 file
阿里云代码仓库已经出现此代码
ctrl+c退出
```

客户端启动，请看第4步。
注意：客户端镜像为: registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto:2020101403



注意：挂载后, 外部空目录会覆盖里面的文件

```bash
docker exec -it writer bash
rm -fr source/*
git checkout source/
```

##  启动客户端
编辑docker-compose.yaml, 修改镜像
```bash
docker-compose up -d
docker-compose logs -f

# windows
docker run -d --restart=always -it --name writer -v c:/source/:/apps/source registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto:2020123101
```

##  写文章
```bash
docker exec writer addarticle 'title'
```
###  添加页面
```bash
docker exec writer addpage ''
```

### 在线编辑文章

![image-20200927103949128](http://myapp.img.mykernel.cn/image-20200927103949128.png)

![image-20200927104014452](http://myapp.img.mykernel.cn/image-20200927104014452.png)

![image-20200927104031554](http://myapp.img.mykernel.cn/image-20200927104031554.png)

###  停止客户端

注意，一定要先将自己的代码，保存之后。确保已经上传至阿里云。之后在停止客户端

否则会有问题

```bash
docker-compose kill
docker-compose rm -f
```

删除/source/目录

```bash
rm -fr /source/
```

注意：如果不删除/source/目录，在启动后，这些文件为新文件和aliyun的冲突. 就会出问题



重启容器和重启服务器，不会有此类问题

原因容器中的.git暂存仓库一直在



###  服务端配置

```bash
cd 5.nginx-pro
```

修改FROM镜像为客户端成品镜像:  registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto:2020101403
```bash
## cat 5.nginx-pro/Dockerfile 
FROM  registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto:2020101403  ## 修改镜像
```

### 制作镜像

```bash
cd 5.nginx-pro


root@songliangcheng:~/blogs/5.nginx-pro# cat build-command.sh
# 10-14添加目录
docker build -t registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:2020101404 ./
docker push registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:2020101404



bash build-command.sh
```

生产服务器编辑docker-compose.yaml的镜像为build后的镜像

```bash
[root@alyy 5.nginx-pro]# docker login --username loneiei registry.cn-hangzhou.aliyuncs.com
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded

[root@alyy 5.nginx-pro]# docker pull registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:2020101404

```

```yaml
version: "3.5"
services:
    nginx-pro:
        restart: always
        image: registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:2020101404 # 这个是build后的镜像
        build:
            context: .
            dockerfile: Dockerfile
            network: "host"
        environment:
        - TZ=Asia/Shanghai
        container_name: nginx-pro
        ports:
        # nginx html
        - 127.0.0.1:5001:80
        # git webhook: git pull, hexo g
        - 5000:5000
        networks:
        - nginx-pro
networks:
  nginx-pro:

```

### 启动服务端 

```bash
docker-compose kill
docker-compose rm -f
docker-compose up -d
docker-compose logs -f


# linux
docker run -d --restart=always -it --name nginx-pro -p 12.0.0.1:5001:80 -p 5000:5000 registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:2020123101 
```

### 配置nginx代理至本地的5001

```bash

/etc/nginx/conf.d/blog.conf
#################### HTTP
server {
   server_name liangcheng.mykernel.cn;
   listen 80;
   location / {
      proxy_pass http://localhost:5001;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Real-IP $remote_addr;
   }
}

# 七牛图片myapp.img.mykernel.cn





```

全栈SSL

```bash
###################### HTTPS
server {
   server_name slc.mykernel.cn;
   listen 443 ssl;
   ssl_certificate /etc/nginx/ssl/4847483_slc.mykernel.cn.pem;  #将domain name.pem替换成您证书的文件名称。
   ssl_certificate_key /etc/nginx/ssl/4847483_slc.mykernel.cn.key; #将domain name.key替换成您证书的密钥文件名称。
   ssl_session_timeout 5m;
   ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4; #使用此加密套件。
   ssl_protocols TLSv1 TLSv1.1 TLSv1.2; #使用该协议进行配置。
   ssl_prefer_server_ciphers on;

   location / {
      proxy_pass http://127.0.0.1:5001;
      #proxy_pass http://lccnx.tpddns.cn:8192;
      proxy_set_header Host $host;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Real-IP $remote_addr;
   }
}
server {
   server_name liangcheng.mykernel.cn;
   listen 80;
   # redirect, permanent
   # last, break
   location / {
    rewrite /(.*)$  https://slc.mykernel.cn/$1 permanent;
   }
}


# 七牛是http, 需要https会话卸载
server {
   server_name myapp.img.mykernel.cn;
   listen 443 ssl;
   ssl_certificate /etc/nginx/ssl/4860773_myapp.img.mykernel.cn.pem;  #将domain name.pem替换成您证书的文件名称。
   ssl_certificate_key /etc/nginx/ssl/4860773_myapp.img.mykernel.cn.key; #将domain name.key替换成您证书的密钥文件名称。
   ssl_session_timeout 5m;
   ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4; #使用此加密套件。
   ssl_protocols TLSv1 TLSv1.1 TLSv1.2; #使用该协议进行配置。
   ssl_prefer_server_ciphers on;
   # redirect, permanent
   # last, break
   location / {
    proxy_pass http://myapp.img.qiniu.mykernel.cn;
   }
}

# 不得行
```



liangcheng -> slc

```bash
# 七牛图片myapp.img.mykernel.cn  重写 myapp.img.qiniu.mykernel.cn
# 由于七牛https要钱，代理过去又代慢， https 重写http不行
server {
   server_name slc.mykernel.cn;
   listen 80;
#   listen 443 ssl;
#   ssl_certificate /etc/nginx/ssl/4847483_slc.mykernel.cn.pem;  #将domain name.pem替换成您证书的文件名称。
#   ssl_certificate_key /etc/nginx/ssl/4847483_slc.mykernel.cn.key; #将domain name.key替换成您证书的密钥文件名称。
#   ssl_session_timeout 5m;
#   ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4; #使用此加密套件。
#   ssl_protocols TLSv1 TLSv1.1 TLSv1.2; #使用该协议进行配置。
#   ssl_prefer_server_ciphers on;

   location / {
      proxy_pass http://127.0.0.1:5001;
      #proxy_pass http://lccnx.tpddns.cn:8192;
      proxy_set_header Host $host;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Real-IP $remote_addr; 
   }
}
server {
   server_name liangcheng.mykernel.cn;
   listen 80;
   # redirect, permanent
   # last, break
   location / {
    rewrite /(.*)$  http://slc.mykernel.cn/$1 permanent;
   }
}


# 七牛是http, 需要https会话卸载
server {
   server_name myapp.img.mykernel.cn;
   listen 80;
#   listen 443 ssl;
#   ssl_certificate /etc/nginx/ssl/4860773_myapp.img.mykernel.cn.pem;  #将domain name.pem替换成您证书的文件名称。
#   ssl_certificate_key /etc/nginx/ssl/4860773_myapp.img.mykernel.cn.key; #将domain name.key替换成您证书的密钥文件名称。
#   ssl_session_timeout 5m;
#   ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4; #使用此加密套件。
#   ssl_protocols TLSv1 TLSv1.1 TLSv1.2; #使用该协议进行配置。
#   ssl_prefer_server_ciphers on;
   # redirect, permanent
   # last, break
   location / {
    # 因为这个地址过来全是引用图片, 添加防盗链
    valid_referers none blocked server_names
                   *.mykernel.cn
                   ~\.google\.
                   ~\.bing\.com
                   ~\.baidu\.(cn|com);
    if ($invalid_referer) {
        return 502;
    }
    add_header Access-Control-Allow-Origin *;
    add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
    add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache
-Control,Content-Type,Authorization';
    # break, last都是本站返回
    ## break 只能在当前location返回 $server_name$request_uri
    ## last  第一个last直接跳出location, 然后在server {} 中从上到下匹配新的$request_uri前缀
    rewrite /(.*)$ http://myapp.img.qiniu.mykernel.cn/$1 permanent;
   }
}

# none 直接请求
# blocked 不合法referer
# server_names 网站的server_name来请求
# 其他域名
# invalid_referer表示
```

slc/liangcheng -> blog

```bash
[root@alyy nginx]# cat conf.d/blog.conf
server {
   server_name blog.mykernel.cn;
   listen 80;
#   listen 443 ssl;
#   ssl_certificate /etc/nginx/ssl/4847483_slc.mykernel.cn.pem;  #将domain name.pem替换成您证书的文件名称。
#   ssl_certificate_key /etc/nginx/ssl/4847483_slc.mykernel.cn.key; #将domain name.key替换成您证书的密钥文件名称。
#   ssl_session_timeout 5m;
#   ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4; #使用此加密套件。
#   ssl_protocols TLSv1 TLSv1.1 TLSv1.2; #使用该协议进行配置。
#   ssl_prefer_server_ciphers on;

   location / {
      proxy_pass http://127.0.0.1:5001;
      #proxy_pass http://lccnx.tpddns.cn:8192;
      proxy_set_header Host $host;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Real-IP $remote_addr; 
   }
}
server {
   server_name liangcheng.mykernel.cn slc.mykernel.cn;
   listen 80;
   # redirect, permanent
   # last, break
   location / {
    rewrite /(.*)$  http://blog.mykernel.cn/$1 permanent;
   }
}

# 七牛是http, 需要https会话卸载
server {
   server_name myapp.img.mykernel.cn;
   listen 80;
#   listen 443 ssl;
#   ssl_certificate /etc/nginx/ssl/4860773_myapp.img.mykernel.cn.pem;  #将domain name.pem替换成您证书的文件名称。
#   ssl_certificate_key /etc/nginx/ssl/4860773_myapp.img.mykernel.cn.key; #将domain name.key替换成您证书的密钥文件名称。
#   ssl_session_timeout 5m;
#   ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4; #使用此加密套件。
#   ssl_protocols TLSv1 TLSv1.1 TLSv1.2; #使用该协议进行配置。
#   ssl_prefer_server_ciphers on;
   # redirect, permanent
   # last, break
   location / {
    # 因为这个地址过来全是引用图片, 添加防盗链
    valid_referers none blocked server_names
                   *.mykernel.cn
                   ~\.google\.
                   ~\.bing\.com
                   ~\.baidu\.(cn|com);
    if ($invalid_referer) {
        return 502;
    }
    add_header Access-Control-Allow-Origin *;
    add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
    add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache
-Control,Content-Type,Authorization';
    # break, last都是本站返回
    ## break 只能在当前location返回 $server_name$request_uri
    ## last  第一个last直接跳出location, 然后在server {} 中从上到下匹配新的$request_uri前缀
    # redirect, permanent 可以本站, 可以其他站返回.
    rewrite /(.*)$ http://myapp.img.qiniu.mykernel.cn/$1 permanent;
   }
}

```

配置个人日记走nginx的认证

> hexo new "diary-<日记名>"

```diff
[root@alyy ~]# cat /apps/nginx/conf/conf.d/blog.conf
server {
  server_name lc.mykernel.cn;
  listen 80;
  index index.php;

  location / {
    root /opt/nginx/html;
  }
  location ~ \.php$ {
      fastcgi_pass   127.0.0.1:9000;
      fastcgi_index  index.php;
      fastcgi_param  SCRIPT_FILENAME  /opt/nginx/html$fastcgi_script_name;
      include        fastcgi_params;
  }
}


server {
   server_name blog.mykernel.cn;
   listen 80;
#   listen 443 ssl;
#   ssl_certificate /etc/nginx/ssl/4847483_slc.mykernel.cn.pem;  #将domain name.pem替换成您证书的文件名称。
#   ssl_certificate_key /etc/nginx/ssl/4847483_slc.mykernel.cn.key; #将domain name.key替换成您证书的密钥文件名称。
#   ssl_session_timeout 5m;
#   ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4; #使用此加密套件。
#   ssl_protocols TLSv1 TLSv1.1 TLSv1.2; #使用该协议进行配置。
#   ssl_prefer_server_ciphers on;

   location / {
      #return 302 http://www.magedu.com;
      # 拒绝爬虫
      if ($http_user_agent ~* (scrapy|curl|httpclient|baidu|google)) {
           return 403;
      }
      proxy_pass http://upstream_servers; # 此proxy_pass有高级功能
      
      ## 缓存 
      
      # 调用
      #proxy_cache proxycache;
      proxy_cache off;
      
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
      
      
   }
+   location ~* "/diary.*/" {
+     auth_basic "Administrator’s Area";
+     auth_basic_user_file /apps/nginx/conf/.htpasswd;
+     proxy_pass http://upstream_servers; # 此proxy_pass有高级功能
+   }
   location /status {
        stub_status;
        #allow 192.168.0.0/16;
        allow 127.0.0.1;
        #deny all;
   }
}
server {
   server_name liangcheng.mykernel.cn slc.mykernel.cn;
   listen 80;
   # redirect, permanent
   # last, break
   location / {
    rewrite /(.*)$  http://blog.mykernel.cn/$1 permanent;
   }
}

# 七牛是http, 需要https会话卸载
server {
   server_name myapp.img.mykernel.cn;
   listen 80;
#   listen 443 ssl;
#   ssl_certificate /etc/nginx/ssl/4860773_myapp.img.mykernel.cn.pem;  #将domain name.pem替换成您证书的文件名称。
#   ssl_certificate_key /etc/nginx/ssl/4860773_myapp.img.mykernel.cn.key; #将domain name.key替换成您证书的密钥文件名称。
#   ssl_session_timeout 5m;
#   ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4; #使用此加密套件。
#   ssl_protocols TLSv1 TLSv1.1 TLSv1.2; #使用该协议进行配置。
#   ssl_prefer_server_ciphers on;
   # redirect, permanent
   # last, break
   location / {
    # 因为这个地址过来全是引用图片, 添加防盗链
    valid_referers none blocked server_names
                   *.mykernel.cn
                   ~\.google\.
                   ~\.bing\.com
                   ~\.baidu\.(cn|com);
    if ($invalid_referer) {
        return 502;
    }
    add_header Access-Control-Allow-Origin *;
    add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
    add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache
-Control,Content-Type,Authorization';
    # break, last都是本站返回
    ## break 只能在当前location返回 $server_name$request_uri
    ## last  第一个last直接跳出location, 然后在server {} 中从上到下匹配新的$request_uri前缀
    # redirect, permanent 可以本站, 可以其他站返回.
    rewrite /(.*)$ http://myapp.img.qiniu.mykernel.cn/$1 permanent;
   }
}

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
    server 127.0.0.1:5001 weight=1 fail_timeout=5s max_fails=3;
}
# proxy_pass的缓存 "线上代码上线, 就需要清理缓存"
# /var/cache/nginx/proxy_cache 缓存位置: install -dv -o nginx -g nginx /var/cache/nginx/proxy_cache
# levels=1:2:2 缓存目录。1表示16进制0-f. 将文件md5从后向前取，第1个为1级子目录，第2-3为2级目录，第4-5为三级目录。加速查找
# proxycache:20m 缓存区域名，大小（主要用来存放key和metadata）
# inactive 缓存有效期
# max_size 占用磁盘空间大小，在inactive时间内，看会有多少缓存。
proxy_cache_path /var/cache/nginx/proxy_cache levels=1:2:2 keys_zone=proxycache:20m inactive=120s max_size=1g;

```





### 自定义配置

```bash
docker run --rm -it --name test registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:2020101404 bash


docker commit test registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:2020101605
```



配置Dockerfile-property

```bash
FROM registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:2020101605
```

制作新镜像

```bash
bash build-command-property.sh
# registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:20201016
```

编辑docker-compose.yaml

```yaml
version: "3.5"
services:
    nginx-pro:
        restart: always
        image: registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:20201016     
```

```bash
docker pull  registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:20201016    

cd 5.nginx-pro/

docker-compose kill; docker-compose rm -f ; docker-compose up -d
```



## 配置七牛存储 

https://juejin.im/post/6844904169745154062

### 在线上传图片

ctrl + alt + a  qq支持在线截图

![image-20200927105409515](http://myapp.img.mykernel.cn/image-20200927105409515.png)

粘贴到博客后，右键。会显示上传按键

![image-20200927105513721](http://myapp.img.mykernel.cn/image-20200927105513721.png)

上传后图片的URL会自动变成七年的CDN加速的域名

![image-20200927105537807](http://myapp.img.mykernel.cn/image-20200927105537807.png)

# 后期维护

## 客户端启动

```bash
# windows
docker run -d --restart=always -it --name writer -v c:/source/:/apps/source registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto:2020123101

# linux
docker run -d --restart=always -it --name writer -v /source/:/apps/source registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto:2020123101
```

实现关机前重启，专用一个脚本来关机，脚本前面是推送博客

> 来源：重庆小传哥

### 失败

windows的shell脚本直接拉`git`

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
```

添加脚本`push-before-shutdown.ps1`

```powershell
param($BLOGDIR)
Set-Location -Path "$BLOGDIR"
Invoke-Expression -Command .\push.sh
```

`gpedit.msc` 这个服务中关机，引用脚本

```powershell
.\2020-09-16-myblog\push-before-shutdown.ps1  C:\2020-09-16-myblog
```



![image-20211103090819227](http://myapp.img.mykernel.cn/image-20211103090819227.png)

> 引用https://www.cnblogs.com/jackadam/p/11311437.html

![image-20211103093024468](http://myapp.img.mykernel.cn/image-20211103093024468.png)

> 1. 将脚本复制到目录下
>
> 2. 添加脚本在这个powershell脚本窗口中
>
>    ![image-20211103092929262](http://myapp.img.mykernel.cn/image-20211103092929262.png)

同时，在左边窗口也添加相同的，添加的是相对文件名

![image-20211103092313933](http://myapp.img.mykernel.cn/image-20211103092313933.png)



引用 http://woshub.com/running-powershell-startup-scripts-using-gpo/

![image-20211103103546528](http://myapp.img.mykernel.cn/image-20211103103546528.png)

![image-20211103103616135](http://myapp.img.mykernel.cn/image-20211103103616135.png)

## 服务端启动

```bash
# linux
docker run -d --restart=always -it --name nginx-pro -p 127.0.0.1:5001:80 -p 5000:5000 registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:2021011403
```

# 备份

```bash
# 服务端
[root@alyy ~]# docker commit -p 705aee015b4b registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:2020123101
[root@alyy ~]# docker login registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:2020123101
[root@alyy ~]# docker push registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:2020123101



# 客户端
C:\Users\Administrator>docker commit -p 0acfd926c4b3    registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto:2020123101
C:\Users\Administrator>docker push registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto:2020123101
```

> 未知

```bash

# 客户端备份
PS C:\Users\Administrator> docker commit -p 2c2a67c644b7 registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto:2021011403
sha256:d13965ec51db0cebe57f7a71e37fbc5af47d830a270f168392344931233df0b6
PS C:\Users\Administrator> docker push registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto:2021011403

# 服务端备份
docker commit -m "toc全部展开" -p 705aee015b4b registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:2021011403
docker push registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:2021011403
```

> 2021年1月14日 周4，添加next主题的备份



此条不要，由于这种加密影响toc显示

```bash
# 服务端备份
[root@alyy ~]# docker commit -m "post加密, only phone" -p 43c506e2b84f registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:20210220
sha256:05ff1fe76d019dd4cd30a51c9229b7d7c58159f27831e15848915d8de96f9803
[root@alyy ~]# docker push registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:20210220
```

> 2021年2月20号，添加文章加密



## 更新主题

```bash
# server
docker run -d --privileged -e NODE_OPTIONS=--max_old_space_size=200 --restart=always -it --name nginx-pro -p 127.0.0.1:5001:80 -p 5000:5000 --cpus=1 -m 512MB registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:20210924
```

> 2021年9月24号，更新所有组件及next. 参考文章  [hexo更新及next主题更新](http://blog.mykernel.cn/2021/09/24/hexo更新及next主题更新/)



## 添加注释插件和资源限制

**影响目录点击**, 废弃

```bash
docker commit -p e91bfa447fb6 registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:202109271148


docker run -d  -e NODE_OPTIONS=--max_old_space_size=200 --restart=always -it --name nginx-pro -p 127.0.0.1:5001:80 -p 5000:5000 --cpus=1 -m 512MB registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:202109271148
```

> 添加插件`admontion` 插件
>
> 由于服务器渲染会出现killed, docker可以使用以下选项限制资源
>
> `--cpus` 表示限制使用1核,`cpu`可压缩资源，<s>不存在被kill</s>。
>
> `-m` 限制500MB内存, 而且应用，<strong>只限制200MB</strong>, <ins>验证给500MB时，也会被kill, 可能有其他资源需求</ins>
>
> 通过系统工具查看被kill原因
>
> ```bash
> [3932741.816369] Out of memory: Kill process 7767 (node) score 87 or sacrifice child
> [3932741.818807] Killed process 7767 (node) total-vm:956396kB, anon-rss:161384kB, file-rss:0kB, shmem-rss:0kB
> [root@alyy ~]# dmesg | tail -n 2000 | grep -i -B 100 'killed process' 
> ```

!!! note ""
    在下次使用此命令前需要进入容器删除`html`，重新渲染
    `command = ' cd /apps/ && git pull origin master && ok=false; until $ok; do   if ! ps -ef | grep -v grep  | grep hexo; then ok=true; echo "no hexo process" ; else echo "sleep 10s, has hexo process"; exit 1; fi; done  && hexo g --config=hexo-config.yaml'` 为避免提交多次，使用此命令

让启动的进程不要被oom kill

```bash
hexo g --config=hexo-config.yaml & echo -17 > /proc/$!/oom_adj
```

配置swap, 让内存不足时，走swap

```bash
dd if=/dev/zero of=/swapfile bs=1M count=4096
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo "/swapfile swap swap defaults 0 0" >> /etc/fstab
```



## 避免启动多次hxo

```bash
docker run -d --privileged -e NODE_OPTIONS=--max_old_space_size=200 --restart=always -it --name nginx-pro -p 127.0.0.1:5001:80 -p 5000:5000 --cpus=1 -m 512MB registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:2021100801
```

更新run.sh

```diff
#!/bin/bash

/usr/local/nginx/sbin/nginx
+ ps -ef | grep -v grep | grep '/usr/bin/python3 /app.py' ||  /usr/bin/python3 /app.py
```

更新app.py

```diff
import subprocess
from flask import Flask, request

app = Flask(__name__)


@app.route('/', methods=['POST', 'GET'])
def hello_world():
    headers = request.headers
    path = request.path
    data = request.data
    
+    command = '''cd /apps/ && git pull origin master && ps -eo cmd,pid | grep '^hexo' | awk '{print $2}' | xargs -I {} kill -9 {} && hexo g --config=hexo-config.yaml'''
    handle = subprocess.Popen(command, shell=True, stdout=subprocess.PIPE)
    print(handle.stdout.read())
    return 'ok'


if __name__ == '__main__':
    app.run(host="0.0.0.0", port=5000, debug=True)
```

## 避免hexo因为kill, 生成大量的core文件

```bash
 docker run -d --privileged -e NODE_OPTIONS=--max_old_space_size=200 --restart=always -it --name nginx-pro -p 127.0.0.1:5001:80 -p 5000:5000 --cpus=1 -m 512MB registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:2021100803
```

更新app.py

```diff
import subprocess
from flask import Flask, request

app = Flask(__name__)


@app.route('/', methods=['POST', 'GET'])
def hello_world():
    headers = request.headers
    path = request.path
    data = request.data
    
+    command = '''cd /apps/ && git pull origin master && ps -eo cmd,pid | grep '^hexo' | awk '{print $2}' | xargs -I {} kill -9 {} && \
+              rm -fr core.* && \
+              hexo g --config=hexo-config.yaml'''
    handle = subprocess.Popen(command, shell=True, stdout=subprocess.PIPE)
    print(handle.stdout.read())
    return 'ok'


if __name__ == '__main__':
    app.run(host="0.0.0.0", port=5000, debug=True)
```

## hexo的`>` 渲染时，css更新

```bash
 docker run -d --privileged -e NODE_OPTIONS=--max_old_space_size=400 --restart=always -it --name nginx-pro -p 127.0.0.1:5001:80 -p 5000:5000 --cpus=1 -m 512MB  registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:20211009
```

## aliyun升级企业版之后，原来的url不可用

```bash
 docker run -d --privileged -e NODE_OPTIONS=--max_old_space_size=400 --restart=always -it --name nginx-pro -p 127.0.0.1:5001:80 -p 5000:5000 --cpus=1 -m 512MB   registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:20211027
```

## 中文支持

```bash
docker run -d --privileged -e NODE_OPTIONS=--max_old_space_size=400 --restart=always -it --name nginx-pro -p 127.0.0.1:5001:80 -p 5000:5000 --cpus=1 -m 1024MB  registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:20211029
```

## 更新网站名和关键词

```bash
docker run -d --privileged -e NODE_OPTIONS=--max_old_space_size=400 --restart=always -it --name nginx-pro -p 127.0.0.1:5001:80 -p 5000:5000 --cpus=1 -m 1024MB  registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:202111052026
```

> ok

## 网站添加搜索

local_serach

```bash
docker run -d --privileged -e NODE_OPTIONS=--max_old_space_size=400 --restart=always -it --name nginx-pro -p 127.0.0.1:5001:80 -p 5000:5000 --cpus=1 -m 1024MB  registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:2021110501
```

> 会影响目录跳转

## 添加来必力评论

支持QQ、微信

```bash
docker run -d --privileged -e NODE_OPTIONS=--max_old_space_size=400 --restart=always -it --name nginx-pro -p 127.0.0.1:5001:80 -p 5000:5000 --cpus=1 -m 1024MB  registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:2021110507
```

> 添加了评论，每个文章会直接跳转到评论位置

## 添加valine评论

```bash
docker run -d --privileged -e NODE_OPTIONS=--max_old_space_size=400 --restart=always -it --name nginx-pro -p 127.0.0.1:5001:80 -p 5000:5000 --cpus=1 -m 1024MB registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:2021110601
```

这个评论也有来必力的问题，但是我发现切换成buttons就正常了

```diff
# Multiple Comment System Support
comments:
  # Available values: tabs | buttons
+  style: buttons
  # Choose a comment system to be displayed by default.
  # Available values: disqus | disqusjs | changyan | livere | gitalk | utterances
+  active:  #livere
  # Setting `true` means remembering the comment system selected by the visitor.
  storage: true
  # Lazyload all comment systems.
  lazyload: false
  # Modify texts or order for any naves, here are some examples.
  nav:
```

## 还原主页

首页加图，首页应该有图片

普通博客前言结束加more, 认证博客，第1个标题紧接后加more

```bash
docker run -d --privileged -e NODE_OPTIONS=--max_old_space_size=400 --restart=always -it --name nginx-pro -p 127.0.0.1:5001:80 -p 5000:5000 --cpus=1 -m 1024MB  registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:2021110602
```

## 生成头像图和引用友链和生成博客存活时长

```bash
docker run -d --privileged -e NODE_OPTIONS=--max_old_space_size=400 --restart=always -it --name nginx-pro -p 127.0.0.1:5001:80 -p 5000:5000 --cpus=1 -m 1024MB   registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:20211108
```

## 添加搜索功能

`hexo-generator-search`

`local_search`

`unescape`

```bash
 docker run -d --privileged -e NODE_OPTIONS=--max_old_space_size=400 --restart=always -it --name nginx-pro -p 127.0.0.1:5001:80 -p 5000:5000 -p 6000:6000 --cpus=1 -m 1024MB   registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:2021110802
```





