---
title: 实现基于域名PC端和移动端
date: 2020-10-13 07:27:53
tags: plan
toc: true
---

基于之前[Ubuntu Server 18.04的安装, 优化系统](http://liangcheng.mykernel.cn/2020/09/29/Ubuntu-Server-18-04%E7%9A%84%E5%AE%89%E8%A3%85-%E4%BC%98%E5%8C%96%E7%B3%BB%E7%BB%9F/)文章, 重新制作centos模板, 并克隆出新主机`cd-ch-pxr-centos-0-198`(成都，成华区，蒲小锐的centos，192.168.0.198主机), 并做了快照

<!--more-->

# 1. 安装nginx

## 1.1 进入nginx官网下载页面

http://nginx.org/en/download.html

Legacy versions: 旧的稳定版

Stable version： 当前稳定版, 建议使用

Mainline version： 主线开发版

![image-20201013153726242](http://myapp.img.mykernel.cn/image-20201013153726242.png)

下面有提供linux包，上面是源码包, 我们先使用包安装。



## 1.2 包安装

![image-20201013153925669](http://myapp.img.mykernel.cn/image-20201013153925669.png)



当我们点击包安装之后，出现支持的指定版本的系统





### 1.2.1 centos安装

因为我们使用centos, 直接点击centos/rehl, 并翻译一下。

![image-20201013154140466](http://myapp.img.mykernel.cn/image-20201013154140466.png)



我们直接执行上面的命令

```bash
sudo yum install yum-utils
```

![image-20201013154246802](http://myapp.img.mykernel.cn/image-20201013154246802.png)

![image-20201013154258198](http://myapp.img.mykernel.cn/image-20201013154258198.png)

```bash
[root@cd-ch-pxr-centos-0-198 ~]# cat /etc/yum.repos.d/nginx.repo # 配置仓库
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true

[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/$releasever/$basearch/
gpgcheck=1
enabled=0
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
```

![image-20201013154426407](http://myapp.img.mykernel.cn/image-20201013154426407.png)

```bash
sudo yum-config-manager --enable nginx-stable # 使用稳定版

[root@cd-ch-pxr-centos-0-198 ~]# cat /etc/yum.repos.d/nginx.repo
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1 # 已经启用
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true

[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/$releasever/$basearch/
gpgcheck=1
enabled=0
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true

```

```bash
sudo yum install nginx
```

![image-20201013154606256](http://myapp.img.mykernel.cn/image-20201013154606256.png)

![image-20201013154618696](http://myapp.img.mykernel.cn/image-20201013154618696.png)

### 1.2.2 ubuntu安装

注意，使用稳定的nginx包配置的apt存储库

![image-20201013154713881](http://myapp.img.mykernel.cn/image-20201013154713881.png)

### 1.2.3 安装后检验

```bash
[root@cd-ch-pxr-centos-0-198 ~]# nginx -V
nginx version: nginx/1.18.0 # nginx是1.18.0版本
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-39) (GCC) 
built with OpenSSL 1.0.2k-fips  26 Jan 2017 # openssl 1.0.2k-fips
TLS SNI support enabled       # 单IP多域名支持HTTPS

# 下面是编辑参数
configure arguments: --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module 
# http认证
--with-http_auth_request_module 
#
--with-http_dav_module --with-http_flv_module --with-http_gunzip_module 
# gzip压缩
--with-http_gzip_static_module
#
--with-http_mp4_module --with-http_random_index_module 
# realip
--with-http_realip_module --with-http_secure_link_module --with-http_slice_module
# https
--with-http_ssl_module 
# substatus页面
--with-http_stub_status_module --with-http_sub_module 
#
--with-http_v2_module --with-mail --with-mail_ssl_module 
# tcp代理
--with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -fPIC' --with-ld-opt='-Wl,-z,relro -Wl,-z,now -pie'
```

# 2. 规划pc和app端

| ip            | 域名                                    | uri  | 网站路径                 | 日志路径                 |
| ------------- | --------------------------------------- | ---- | ------------------------ | ------------------------ |
| 192.168.0.198 | [www.magedu.net](http://www.magedu.net) | /    | /apps/magedu/html/pc     | /apps/magedu/logs/pc     |
| 192.168.0.198 | mobile.magedu.net                       | /    | /apps/magedu/html/mobile | /apps/magedu/logs/mobile |



## 2.1 配置pc端

```nginx
[root@cd-ch-pxr-centos-0-198 conf.d]# cat pc.conf 
server {
	listen 80;
	server_name www.magedu.net;
	charset utf-8;
	access_log  /apps/magedu/logs/pc/pc.access_log  main;
	location / {
		root /apps/magedu/html/pc;
		index index.html index.htm;
	}
	error_page   500 502 503 504  /50x.html;
	location = /50x.html {
		root   /usr/share/nginx/html;
	}
	error_page  404              /404.html;
	location = /404.html {
		root   /apps/magedu/html/pc;
	}
}
```

```bash
install -dv /apps/magedu/logs/pc/
install -dv /apps/magedu/html/pc
chown -R nginx.nginx /apps/
echo '404 erorr www.magedu.com' > /apps/magedu/html/pc/404.html
echo 'welcome to magedu' > /apps/magedu/html/pc/index.html
```

解析域名

```bash
[root@cd-ch-pxr-centos-0-198 conf.d]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4 www.magedu.net
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

```

```bash
# nginx
[root@cd-ch-pxr-centos-0-198 conf.d]# curl  www.magedu.net/21341243
404 erorr www.magedu.com
[root@cd-ch-pxr-centos-0-198 conf.d]# curl  www.magedu.net
welcome to magedu
```

## 2.2 配置mobile端

```nginx
[root@cd-ch-pxr-centos-0-198 conf.d]# cat mobile.conf
server {
	listen 80;
	server_name mobile.magedu.net;
	charset utf-8;
	access_log  /apps/magedu/logs/mobile/mobile.access_log  main;
	location / {
		root /apps/magedu/html/mobile;
		index index.html index.htm;
	}
	error_page   500 502 503 504  /50x.html;
	location = /50x.html {
		root   /usr/share/nginx/html;
	}
	error_page  404              /404.html;
	location = /404.html {
		root   /apps/magedu/html/mobile;
	}
}
```

```bash
install -dv /apps/magedu/html/mobile /apps/magedu/logs/mobile/
echo '404 erorr www.magedu.com on mobile' > /apps/magedu/html/mobile/404.html
echo 'welcome to magedu on mobile' > /apps/magedu/html/mobile/index.html
```

```bash
# vim /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4 www.magedu.net mobile.magedu.net

```

```bash
# nginx -s reload
[root@cd-ch-pxr-centos-0-198 conf.d]# curl mobile.magedu.net
welcome to magedu on mobile
[root@cd-ch-pxr-centos-0-198 conf.d]# curl mobile.magedu.net/123
404 erorr www.magedu.com on mobile
[root@cd-ch-pxr-centos-0-198 conf.d]#  tail /apps/magedu/logs/mobile/mobile.access_log 
127.0.0.1 - - [13/Oct/2020:16:17:19 +0800] "GET / HTTP/1.1" 200 18 "-" "curl/7.29.0" "-"
127.0.0.1 - - [13/Oct/2020:16:17:21 +0800] "GET /123 HTTP/1.1" 404 25 "-" "curl/7.29.0" "-"
127.0.0.1 - - [13/Oct/2020:16:18:18 +0800] "GET / HTTP/1.1" 200 28 "-" "curl/7.29.0" "-"
127.0.0.1 - - [13/Oct/2020:16:18:24 +0800] "GET /123 HTTP/1.1" 404 35 "-" "curl/7.29.0" "-"
```




