---
title: haproxy编译安装及配置优化
date: 2020-10-16 01:43:58
tags: 
toc: true
---



下载haproxy: http://www.haproxy.org/download/

<!--more-->

# 1. lua

```bash
yum install libtermcap-devel ncurses-devel libevent-devel readline-devel wget make gcc -y

#wget http://www.lua.org/ftp/lua-5.3.5.tar.gz
tar xvf lua-5.3.5.tar.gz  -C /usr/local/src/
cd /usr/local/src/lua-5.3.5/src/
make linux
./lua -v
cd -
```



# 2. haproxy 安装配置及优化

安装

```bash
yum install gcc gcc-c++ glibc glibc-devel pcre pcre-devel openssl openssl-devel systemd-devel net-tools vim iotop bc zip unzip zlib-devel lrzsz tree screen lsof tcpdump wget ntpdate -y

#wget http://www.haproxy.org/download/2.0/src/haproxy-2.0.18.tar.gz
tar xvf haproxy-2.0.18.tar.gz  -C /usr/local/src/

#1.8或1.9
#cd /usr/local/src/
#make ARCH=x86_64 TARGET=linux2628 USE_PCRE=1 USE_OPENSSL=1 USE_ZLIB=1 USE_SYSTEMD=1 USE_CPU_AFFINITY=1 PREFIX=/usr/local/haproxy

#2.0 +
cd /usr/local/src/haproxy-2.0.18/
make ARCH=x86_64 TARGET=linux-glibc USE_PCRE=1 USE_OPENSSL=1 USE_ZLIB=1 USE_SYSTEMD=1 USE_CPU_AFFINITY=1 PREFIX=/usr/local/haproxy

make install PREFIX=/usr/local/haproxy
cp haproxy /usr/sbin/
haproxy -v
```



配置文件

工作模型: master（root运行) - worker(haproxy)

1. tcp代理(mode tcp + send_proxy -> nginx proxy_protocol)
2. http代理：日志、压缩、ssl、acl、增删除请求首部：一般不使用
   - acl: 基于首部名对应的值、前缀匹配、后缀匹配、src匹配、阻止、域名匹配：常用于开发和预发环境
4. 状态页、多用户、队列、session total、上线时间status、lastChk的状态
4. 会话保持：source, user-agent, cookie.  负载均衡: roundrobin(默认)



> 一句话概述： haproxy: tcp/http  优化连接数、进程绑定，错误处理，管理页，cookie粘性。

```bash

mkdir /etc/haproxy
mkdir /var/lib/haproxy
cat > /etc/haproxy/haproxy.cfg << EOF
global
    maxconn 100000 # 单个haproxy进程处理最多连接数. 监控入口这里总连接数达到5万时, 就加服务器。加服务器：给领导看连接数，访问量。
    #maxsslconn    # haproxy不配置ssl, 由后端nginx配置ssl.
    spread-checks 2 # 延迟检测和提前检测，不会同时向后端发起检查, 建议2-5. 数千后端非常必要配置.
    chroot /usr/local/haproxy # haproxy工作路径,haproxy进程被劫持了，只能在这个目录活动。

	#maxconnrate  # 每个进程每秒创建最大连接数，如果设定了，就大点。小了如果业务请求不多，没有问题。后期如果请求大了，用户请求就进不来。所以不设定
	
    #--- 进程绑定及每个进程的管理sock文件
    nbproc 4   #默认单进程启动。 haproxy只能工作为多进程单线程(进程和cpu绑定，此效率最好)。或单进程多线程。
    cpu-map 1 0 # cpu 进程 cpu.   因为服务器前2个核系统调用。 1，2核。
    cpu-map 2 1
    cpu-map 3 2
    cpu-map 4 3
    #--
    # # 非交互完成服务器热上线和下线
    # echo "help" | socat stdio  /var/lib/haproxy/haproxy.sock3 获取帮助
    # echo "enable server" 上线
    # disable server 下线; 或 set weight 0 
    # get weight, 获取获取
    # set weight 当是动态roundrobin或带选项的动态(uri, uri_param, hdr())支持动态调整weight
    stats socket /var/lib/haproxy/haproxy.sock1 mode 600 level admin process 1 
    stats socket /var/lib/haproxy/haproxy.sock2 mode 600 level admin process 2 
    stats socket /var/lib/haproxy/haproxy.sock3 mode 600 level admin process 3 
    stats socket /var/lib/haproxy/haproxy.sock4 mode 600 level admin process 4 
    user haproxy          # worker进程的用户
    group haproxy         # worker进程的组
    daemon                # 守护进程 

    pidfile /var/lib/haproxy/haproxy.pid
    log 127.0.0.1 local3 info 
    # 必须结合   log global 使用
    # /etc/rsyslog.conf 
    # 16 # provides UDP syslog reception
    # 17 module(load="imudp")
    # 18 input(type="imudp" port="514")  

    # /etc/rsyslog.d/50-default.conf
    # 40 local3.*              /var/log/haproxy.log  

    # root@ubuntu-template:~# systemctl restart haproxy rsyslog
    # root@ubuntu-template:~# tail -f /var/log/haproxy.log 

   
   
###################### Proxies配置 ######################
    
# defaults对frontend name (ngx server),backend name(ngx upstream),listen name生效。name只能- _ . :
defaults
    option http-keep-alive
    
    option forwardfor
    # mode http模式下，透传dnat的ip至后端. 默认X-Forwarded-For就是真实客户端的首部
    # option forwardfor header X-Real-IP # 表示不使用默认的首部，而是使用X-Real-IP来透传 
    
    # mode tcp模式下透传
    # mode tcp
    # server web1 blogs.studylinux.net:80 send-proxy check inter 3s fall 3 rise 5 
    # 后端nginx配置
    # server {
    #    listen 80 proxy_protocol;
    #    server_name blogs.studylinux.net;
    # }
    # nginx日志中的$proxy_protocol_addr
    
    option redispatch              # server id对应的服务器挂了, 强制定向请求至其他健康后端。
    # option abortonclose          # 负载高, 自动结束后端处理久的链接(千万不要加), 后端慢了直接加后端的IO和带宽。
    
    # 与后端建立长连接时长。如果时长长了，新连接进不来。
    #所以在请求量特别大的站点，此值应该尽可能的小(1ms), 这样与后端就不是长连接，都是短连接，来一个请求处理一个。 
    #请求量不大的站点，此值60/120s, 表示长连接2min。避免开销。
    timeout http-keep-alive 1ms       
    # frontend, backend, listen默认继承http模式 
    mode http
    
    # client -> haproxy -> realserver 建立连接超时
    timeout connect 10s             
    
    # client端打开网站多长时间不活跃就强制断开。60s-120s
    timeout client 1m       
    
	# haproxy响应超时, 一般60s.  用户最多等1分钟。如果在特殊情况后端有慢查询，建立连接后，就需要等数据返回，就调整此值： nginx: proxy_read_timeout, proxy_send_timeout
    timeout server 1m               
    
# 加注释: web监控页面, 宋亮成 N49
listen stats    
  #bind-process 1                    # 默认展示所有进程或 bind-process all
  bind :9999
  # 启用状态页
  stats enable
  
  # 启用日志
  log global 
  
  # 隐藏haproxy版本，安全
  stats hide-version
  
  # 自动刷新时间间隔
  #stats refresh 3s
  
  # 自定义状态页uri, 默认/haproxy?stats
  stats uri /haproxy
  
  # 账户认证时的提示信息，
  stats realm HAPorxy\ Stats\ Page
  
  # 认证时的用户和密码
  stats auth admin:123456
  stats auth haadmin:123456
  
  # 启用stats页面的管理功能
  stats admin if TRUE
  

#########################################################################
# pid = 793 (process #4, nbproc = 4, nbthread = 1) 
# uptime = 2d 20h11m56s
# system limits: memmax = unlimited; ulimit-n = 200034
# maxsock = 200034; maxconn = 100000; maxpipes = 0
# current conns = 1; current pipes = 0/0; conn rate = 0/sec; bit rate = 0.000 kbps
# Running tasks: 1/12; idle = 88 %
  ################
# pid = 793 (process #4, nbproc = 4, nbthread = 1) 
# pid = 793 当前worker进程的pid
# root@ubuntu-template:~# ps -q "`pidof haproxy`" axo pid,user,psr,cmd
#   PID USER     PSR CMD
#   793 haproxy    0 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /var/lib/haproxy/haproxy.pid
#   792 haproxy    0 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /var/lib/haproxy/haproxy.pid
#   791 haproxy    0 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /var/lib/haproxy/haproxy.pid
#   790 haproxy    0 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /var/lib/haproxy/haproxy.pid
#   736 root       0 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /var/lib/haproxy/haproxy.pid

# process #4, 当前是哪个进程的状态
# nbproc = 4, 总共的进程数
# nbthread = 1, 单个进程内多少个线程

# uptime = 2d 20h11m56s # 启动了多久时长
# system limits: memmax = unlimited; ulimit-n = 200034 # 系统资源限制; 最大内存，最大文件打开数
# maxsock = 200034; maxconn = 100000; maxpipes = 0 # 最大socket, 最大连接，最大管道
# current conns = 1; current pipes = 0/0; conn rate = 0/sec; bit rate = 0.000 kbps # 当前连接；当前管道；当前速度
# Running tasks: 1/12; idle = 88 % # 运行的任务，空间率
#########################################################################


#########################################################################
# 	active UP	 	backup UP
# active UP, going down		backup UP, going down
# active DOWN, going up		backup DOWN, going up
# active or backup DOWN  		not checked
# active or backup DOWN for maintenance (MAINT)  
# active or backup SOFT STOPPED for maintenance  
# Note: "NOLB"/"DRAIN" = UP with load-balancing disabled.
  ################
# 	active UP 在线的服务器	
#   backup UP 标记backup
# active UP, going down 从up服务器到离线。fall指定次数已经检测满足		
# backup UP, going down 从backup up到离线。
# active DOWN, going up down服务器到在线。rise指定次数已经检测满足		
# backup DOWN, going up backup服务器到在线。
# active or backup DOWN active或backup服务器已经down.
# not checked           标记不为监测的服务器
# active or backup DOWN for maintenance (MAINT)   active或backup服务器人为下线。进入维护模式
# active or backup SOFT STOPPED for maintenance  active或backup服务器软下线（人为将weight修改为0）
# Note: "NOLB"/"DRAIN" = UP with load-balancing disabled.
  
  
# 加注释: 官网页面, 维护人：宋亮成 ############################
# 如果注释的人离职，就问谁现在维护官网？然后更新姓名
listen http
  # 动态：1）慢启动：后端上线可以先来少量的流量接入，没有问题，才向后端转发大量的流量。2）可以socat(socket)命令来动态修改weight.
  # 静态，没有动态功能。weight不可以修改
  # 动态或静态算法(不加选项)
      # 会话保持, 但是SNAT中用户人不一样。web
      # balance source 
      # 最少连接, 可能一个连接重。常用于mysql
      # balance leastconn
  
  # 带选项的动态或静态。一般都加hash-type consistent.
      # haproxy后端是缓存的场景
      # balance uri hash-type consistent# nginx的: proxy_cache_key $request_uri
      # balance hdr(user-agent) hash-type consistent
      # balance url-parame 
	
  
  # 基于cookie会话保持，不在基于源地址完成会话保持，就算同一个SNAT用户同一个计算机，用户使用不同的浏览器有不同的cookie.  web场景
  # cookie SERVER-COOKIE insert indirect nocache
  # server web1 192.168.7.103:80 cookie check inter 3s fall 3 rise 5
  
  
  # 使用场景：
  # session粘性：source -> hdr(user-agent) -> cookie
  # session共享：balance static-rr
  # mysql leastconn
  
  # 通常绑定vip:port
  # bind vip1:port,vip2:port,vip3:port1-port2
  bind *:80 
  
  mode http # http转发可以收集日志
  
  # check #对指定real健康检测, 默认server id后的server的ip和port
      # addr IP 自定义ip
      # port num 自定义port.. 可以多个port. port 80 port 9000
      # inter num 检测间隔。默认ms, 可使用s单位。
      # fall num 成功到失败的检测次数
      # rise num 失败到成功的检测次数
  # weight 权重 ram/cpu 
  # backup 备用或sorry server
  # disabled 禁用服务器。ngx是down
  # redirect prefix http://magedu.net # ngx return 302或rewrite 
  # maxconn 后端的最大连接数，一般不指定
  # backlog server达到上限的后援队列长度
  
  server 172.16.0.221 172.16.0.221:80  check port 80 inter 3s fall 2 rise 5

# 加注释：https页面, 宋亮成 一般tls在nginx上做,这里只是转发。
listen https
  bind *:443 
  mode http
  server 172.16.0.221 172.16.0.221:443  check port 443 inter 3s fall 2 rise 5

# 记录nginx格式的日志, 宋亮成 ################################
listen nginx-log
	#bind vip:port
	bind 172.16.0.248:80 
	mode http
	balance roundrobin
	
	# 7层检测php-fpm php-fpm处理HEAD会占用大量CPU.
    option httpchk HEAD /php-status HTTP/1.1\r\nHost:\ 172.27.0.248
    # 7层检测nginx 后端应该关闭检测日志。
    option httpchk HEAD /index.php HTTP/1.1\r\nHost:\ 172.27.0.248
	# 用户请求插入cookie, 后续插入的后端的cookie始终定向到同一个后端
	cookie SERVER-COOKIE insert indirect nocache
	# 4层检测port
	# 只要fall了就不会调度用户了，所以失败次数3次，3*3=9s才不调度
	
	server web1 192.168.7.103:80 cookie web1 check addr 192.168.7.103 port 80 inter 3s fall 3 rise 5
	server web1 192.168.7.104:80 cookie web1 check addr 192.168.7.104 port 80 inter 3s fall 3 rise 5
	
	## realserver 删除首部-> haproxy 添加首部-> client  
    # HTTP/1.1 200 OK
    # server: nginx/1.16.1 隐藏
    # date: Fri, 11 Dec 2020 08:54:32 GMT
    # content-type: text/html; charset=UTF-8
    # transfer-encoding: chunked
    # x-powered-by: PHP/8.0.0 隐藏
    # X-Via: magedu linux49 slc/pxr HaProxy 添加
    # 删除后端传递来的
    http-response del-header server
    http-response del-header x-powered-by
    # 添加首部标记
    http-response add-header X-Via "magedu linux49 slc/pxr HaProxy"
	
	
	########################## 压缩 #######################################################
	# haproxy是入口 应该降低负载，不需要压缩
	#
	## 激活压缩
	# 请求首部包含以下首部，并且服务器配置压缩并支持压缩算法
	# Accept-Encoding: gzip, deflate
	##禁用压缩 
	# 真实服务器响应
	# 没有有Content-Length或Transfer-Encoding:chunked, status:不是200
	# 响应有Content-Type：multipart*
	# 响应有Cache-control:no-transform
	# User-Agent匹配“ Mozilla / 4”，除非它是具有XP SP2的MSIE 6或MSIE 7及更高版本。
	# 如果后端压缩了会响应Content-Encoding，haproxy将不会再压缩。
	# The response contains an invalid "ETag" header or multiple ETag headers
    #compression algo gzip deflate
    #compression type text/xml application/xml application/atom+xml application/rss+xml application/xhtml+xml image/svg+xml text/javascript application/javascript application/x-javascript text/x-json application/json application/x-web-app-manifest+json text/css text/plain text/x-component font/opentype font/ttf application/x-font-ttf application/vnd.ms-fontobject image/x-icon   
	#########################################################################################
	
	########################## 日志 #######################################################
	# haproxy是入口 一般工作在tcp模式，不需要记录日志，记录日志的问题：
	#  日志满了（磁盘、inode）就会导致用户请求不能请求。需要zabbix监控
	# 
	# 有特殊需求，就mode http.
	#
	# 当前listen所有日志和global配置的log输出位置一致
	log global
	# http://cbonte.github.io/haproxy-dconv/2.4/configuration.html#8.2.4
	log-format "%ci:%cp [%tr] %ft %b/%s %TR/%Tw/%Tc/%Tr/%Ta %ST %B %CC %CS %tsc %ac/%fc/%bc/%sc/%rc %sq/%bq %hr %hs %{+Q}r"
	# 日志格式选项
	option httplog
	# captured_request_cookie
		# Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
        # Accept-Encoding: gzip, deflate
        # Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
        # Cache-Control: no-cache
        # Connection: keep-alive
        # Cookie: wp-settings-time-1=1606816504; SERVER-COOKIE=192.168.101.248
	 
        # Haproxy配置
        # mode http
        # capture cookie wp-settings-time-1 len 32   

        # 请求日志 Dec 11 13:35:25 localhost haproxy[30702]: 172.27.3.1:64839 [11/Dec/2020:13:35:22.841] nginx-log nginx-log/192.168.101.248 0/0/1/2455/2486 200 22705 wp-settings-time-1=1606816504 - --VN 1/1/0/0/0 0/0 {|Mozilla/5.0 (Wi|172.27.0.248:8192|} {} "GET / HTTP/1.1"
	
	# %CC
	capture cookie wp-settings-time-1 len 512
	# captured_response_cookie
	# %hr
	#如果未启用任何捕获，则不会显示大括号，从而导致剩余字段的移位。
	# 上传时的内容长度：Content-length
	capture request header Content-length len 512
	# 快速区分真实用户和机器人的“用户代理” User-agent
	capture request header User-agent len 512
	# 在代理环境中查找请求的来源 X-Forwarded-For
	capture request header X-Forwarded-For len 512
	# 防盗链, 仅在定位到图片服务器使用。
	capture request header Referer len 512


	# %hs
	#如果未启用任何捕获，则不会显示大括号，从而导致剩余字段的移位。
	# 响应标头捕获的常见用法包括“ Content-length”标头，指示预期返回多少字节，
	capture response header Content-length len 512
	# “位置”标头可跟踪重定向。
	capture response header Location len 15
	# Field   Format                                Extract from the example above
	#       1   process_name '[' pid ']:'                            haproxy[14389]:
	#       2   client_ip ':' client_port                             10.0.1.2:33317
	#       3   '[' request_date ']'                      [06/Feb/2009:12:14:14.655]
	#       4   frontend_name                                                http-in
	#       5   backend_name '/' server_name                             static/srv1
	#       6   TR '/' Tw '/' Tc '/' Tr '/' Ta*                       10/0/30/69/109 client与haproxy建立连接/在队列中等待连接所花费的总时间/haproxy与后端建立连接/haproxy 建立连接后到real server响应/向haproxy发起请求到real server响应。(ms)
	#       7   status_code                                                      200 状态码
	#       8   bytes_read*                                                     2750 响应字节
	#       9   captured_request_cookie                                            - 抓请求的cookie
	#      10   captured_response_cookie                                           -
	#      11   termination_state                                               ---- NN表示插入cookie
	#      12   activeconn '/' frontendconn '/' backendconn '/' served_conn '/' retries*    1/1/1/1/0
	#      13   srv_queue '/' backend_queue                                      0/0
	#      14   '{' captured_request_headers* '}'                     {|Mozilla/5.0 (Wi|172.27.0.248:8192|} 抓请求头
	#      15   '{' captured_response_headers* '}'                                {} 抓响应头
	#      16   '"' http_request '"'                      "GET /index.html HTTP/1.1"
	
  ################  ACL

  # 域名匹配
  # 添加vip的hosts解析 172.27.0.248 www.magedu.net
  # -i 不区分大小写
  #acl jiege_acl hdr_dom(host) -i www.magedu.net

  # 源地址调度
  #acl src_acl src 172.27.1.1 10.21.0.0/16

  # 源地址的访问控制 curl 172.27.0.248:8192. 在172.27.1.1被拒绝
  #acl src_acl src 172.27.1.1 10.21.0.0/16
  # 方式1
  #block if src_acl
  # 方式2
  #http-request deny if src_acl
  #http-request allow


  # 基于浏览器的调度
  ## user-agent的header的值的起始
  #acl ie_acl hdr_beg(User-Agent) -i "Mozilla/5.0"
  # 匹配了302重定向
  #redirect prefix http://10.21.107.1 if ie_acl


  # 基于文件名 动静分离
        # 107准备
		#install static images -dv
		#echo 107.1 static >  static/index.html 
		#echo 107.1 imgs >  images/index.html
		#find / -name "*.png"
		#cp /apps/nginx/html/zabbix/assets/img/browser-sprite.png ./1.png
		# 请求107的3个url: 1.png/static/images均正常

  acl php_acl path_end -i .php
  acl static path_beg -i /static /images
  acl imgs  path_end -i .png .gif .jpeg
  ############### backend ###############
  # 源地址
  #use_backend  one if src_acl

  # 域名
  #use_backend  one if jiege_acl

  # 浏览器
  #use_backend one if ie_acl

  # 动静分离
  use_backend  default if php_acl
  ## 多条件
  #use_backend  default if ! php
  #use_backend  one if static  imgs
  use_backend  one if static || imgs

  
  ############### default backend ###############
  # 注意：此配置文件上面的server段配置了，就不需要默认backend。如果以下默认backend配置了，上面就不配置server了
  default_backend default

backend default
  option httpchk HEAD /php-status HTTP/1.1\r\nHost:\ 172.27.0.248
  option httpchk HEAD /index.php HTTP/1.1\r\nHost:\ 172.27.0.248
  server 10.21.101.1 10.21.101.1:80 check port 80 inter 3s fall 3 rise 5 
  server 10.21.101.2 10.21.101.2:80 check port 80 inter 3s fall 3 rise 5 


backend one
  server 10.21.107.1 10.21.107.1:80 check  inter 3s fall 3 rise 5


	
	
# 4层代理
listen tcp-http
  bind 172.27.0.248:8193
  mode tcp                                 
  # send-proxy完成ip透传
  server 10.21.101.1 10.21.101.1:80 send-proxy check port 80 inter 3s fall 3 rise 5 
  server 10.21.101.2 10.21.101.2:80 send-proxy check port 80 inter 3s fall 3 rise 5 
	
EOF
```

![image-20201211164250610](http://myapp.img.mykernel.cn/image-20201211164250610.png)



添加用户

```bash
groupadd -r haproxy
useradd -g haproxy -r -M -s /sbin/nologin haproxy
```

service文件

```bash
cat > /usr/lib/systemd/system/haproxy.service << EOF
[Unit]
Description=HAProxy Load Balancer
After=syslog.target network.target
#目录需对应安装目录
[Service]
ExecStartPre=/usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -c -q
ExecStart=/usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /var/lib/haproxy/haproxy.pid
ExecReload=/bin/kill -USR2 $MAINPID

[Install]
WantedBy=multi-user.target
EOF
```



启动

```bash
systemctl daemon-reload
systemctl enable --now haproxy
systemctl status haproxy
ss -tnlp | grep :80
```



# 3. haproxy状态页

![image-20201210150321581](http://myapp.img.mykernel.cn/image-20201210150321581.png)

由图知：后端服务器权重为1，活动链接1，备份服务器0，心跳检测0，后端DOWN机0，DOWN时间0s. 

![image-20201210151954970](http://myapp.img.mykernel.cn/image-20201210151954970.png)



1. queue过长，haproxy就慢了

2. session下面total有更详细描述

   ![image-20201214134611035](http://myapp.img.mykernel.cn/image-20201214134611035.png)



3. lastChk表明最近一次检测，4层或是7层，检测使用时长
4. status 表示服务器工作多长时长



# 4. haproxy 日志

![image-20201211164250610](http://myapp.img.mykernel.cn/image-20201211164250610.png)

配置日志

k8s中，java仅记录错误日志

nohup jar > /dev/null 2>> /apps/log/app.log &

```bash
# rsyslog.conf
udp 

local3.*   @192.168.0.123 # udp
local3.*   @@192.168.0.123 # tcp
local3.*   /var/log/haproxy.log # 本机文件


# haproxy.cfg
global
    local 127.0.0.1 local3
frontend http-in
    mode http
    default_backend bck
    
	# 当前listen所有日志和global配置的log输出位置一致
	log global
	# http://cbonte.github.io/haproxy-dconv/2.4/configuration.html#8.2.4
	log-format "%ci:%cp [%tr] %ft %b/%s %TR/%Tw/%Tc/%Tr/%Ta %ST %B %CC %CS %tsc %ac/%fc/%bc/%sc/%rc %sq/%bq %hr %hs %{+Q}r"
	# 日志格式选项
	option httplog
	# captured_request_cookie
		# Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
        # Accept-Encoding: gzip, deflate
        # Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
        # Cache-Control: no-cache
        # Connection: keep-alive
        # Cookie: wp-settings-time-1=1606816504; SERVER-COOKIE=192.168.101.248
	 
        # Haproxy配置
        # mode http
        # capture cookie wp-settings-time-1 len 32   

        # 请求日志 Dec 11 13:35:25 localhost haproxy[30702]: 172.27.3.1:64839 [11/Dec/2020:13:35:22.841] nginx-log nginx-log/192.168.101.248 0/0/1/2455/2486 200 22705 wp-settings-time-1=1606816504 - --VN 1/1/0/0/0 0/0 {|Mozilla/5.0 (Wi|172.27.0.248:8192|} {} "GET / HTTP/1.1"

	capture cookie wp-settings-time-1 len 32
	# captured_response_cookie
	# %hr
	#如果未启用任何捕获，则不会显示大括号，从而导致剩余字段的移位。
	# 上传时的内容长度：Content-length
	capture request header Content-length len 15
	# 快速区分真实用户和机器人的“用户代理” User-agent
	capture request header User-agent len 15
	# 在代理环境中查找请求的来源 X-Forwarded-For
	capture request header X-Forwarded-For len 15
	# 防盗链, 仅在定位到图片服务器使用。
	capture request header Referer len 15
backend static
	server srv1 127.0.0.1:8000
```

输出结果

遵循默认格式

```bash
 >>> Feb  6 12:14:14 localhost \
       haproxy[14389]: 10.0.1.2:33317 [06/Feb/2009:12:14:14.655] http-in \
       static/srv1 10/0/30/69/109 200 2750 - - ---- 1/1/1/1/0 0/0 {1wt.eu} \
       {} "GET /index.html HTTP/1.1"
       
       
Field   Format                                Extract from the example above
      1   process_name '[' pid ']:'                            haproxy[14389]:
      2   client_ip ':' client_port                             10.0.1.2:33317
      3   '[' request_date ']'                      [06/Feb/2009:12:14:14.655]
      4   frontend_name                                                http-in
      5   backend_name '/' server_name                             static/srv1
      6   TR '/' Tw '/' Tc '/' Tr '/' Ta*                       10/0/30/69/109
      7   status_code                                                      200
      8   bytes_read*                                                     2750
      9   captured_request_cookie                                            -
     10   captured_response_cookie                                           -
     11   termination_state                                               ----
     12   actconn '/' feconn '/' beconn '/' srv_conn '/' retries*    1/1/1/1/0
     13   srv_queue '/' backend_queue                                      0/0
     14   '{' captured_request_headers* '}'         {|Mozilla/5.0 (Wi|172.27.0.248:8192|}
     15   '{' captured_response_headers* '}'                                {}
     16   '"' http_request '"'                      "GET /index.html HTTP/1.1"
```

更详细的日志格式

```bash

#http://cbonte.github.io/haproxy-dconv/2.3/configuration.html#8.2.4
# "%ci:%cp [%tr] %ft %b/%s %TR/%Tw/%Tc/%Tr/%Ta %ST %B %CC %CS %tsc %ac/%fc/%bc/%sc/%rc %sq/%bq %hr %hs %{+Q}r"
# |   | %ci  | client_ip                 (accepted address)  | IP          | 与haproxy发起连接的ip. 172.27.3.1
# |   | %cp  | client_port               (accepted address)  | numeric     | 发起连接的客户端的TCP端口  55257
# | H | %tr  | date_time of HTTP request                     | date        | haproxy接收到HTTP请求的第一个字节的确切日期[10/Dec/2020:16:17:35.665]
# |   | %ft  | frontend_name_transport ('~' suffix for SSL)  | string      | 接收和处理连接的前端（或侦听器）的名称 nginx-log
# |   | %b   | backend_name                                  | string      | 被选择用来管理与服务器的连接的后端（或监听器）的名称 nginx-log
# |   | %s   | server_name                                   | string      | 连接发送到的最后一个服务器的名称 192.168.101.248
#
# | H | %TR  | time to receive the full request from 1st byte| numeric     | -“ TR”是接收到第一个字节后等待来自客户端（不包括正文）的完整HTTP请求所花费的总时间（以毫秒为单位）。如果在接收到完整的请求或接收到错误的请求之前中止连接，则可以为“-1”。它通常应该很小，因为一个请求通常适合一个数据包。此处的时间较长通常表示客户端与haproxy之间的网络问题或手动键入的请求.
# |   | %Tw  | Tw                                            | numeric     | 在队列中等待连接所花费的总时间 0
# |   | %Tc  | Tc                                            | numeric     | 从haproxy代理发送连接请求到real server服务器确认连接之间的时间 

# | H | %Tr  | Tr (response time)                            | numeric     | haproxy与real server服务器已经建立TCP连接 到 real server服务器发送其完整的响应头 之间的时间. 2498ms 即 2s 


# |   | %Ta  | Active time of the request (from TR to end)   | numeric     | 从haproxy收到请求标头的第一个字节到haproxy发出响应正文的最后一个字节之间的时间 2530ms, 
# Timings events in HTTP mode:

#                first request               2nd request
#     |<-------------------------------->|<-------------- ...
#     t         tr                       t    tr ...
#  ---|----|----|----|----|----|----|----|----|--
#     : Th   Ti   TR   Tw   Tc   Tr   Td : Ti   ...
#     :<---- Tq ---->:                   :
#     :<-------------- Tt -------------->:
#     :<--        -----Tu--------------->:
#               :<--------- Ta --------->:
#	所有值均以毫秒为单位报告。 在前端设置了“ option tcplog”的TCP模式下，以“ Tw / Tc / Tt”的形式报告3个控制点，而在HTTP模式下，以“ TR / Tw / Tc / Tr / Ta””的形式报告5个控制点。 另外，提供了三个其他度量，即“ Th”，“ Ti”和“ Tq”。

#	-TR：获取客户端请求的总时间（仅HTTP模式）。 这是从haproxy收到的第一个字节到haproxy代理收到标记HTTP标头结尾的空行之间的时间。 
#		值“ -1”表示从未看到标头的末尾。 当客户端过早关闭或超时时，会发生这种情况。 由于大多数请求都适合单个数据包，因此此时间通常很短。 较长的时间可能表明在测试过程中手动键入了请求。

#	Tw：在队列中等待连接插槽所花费的总时间。 它考虑了后端队列以及服务器队列，并且取决于队列大小以及服务器完成先前请求所需的时间。 
#		值“ -1”表示该请求在到达队列之前就被杀死，这通常是无效或拒绝请求时发生的情况。

#	Tc：与服务器建立TCP连接的总时间。 它是从haproxy代理发送连接请求到real server服务器确认连接之间的时间，或者是TCP SYN数据包与匹配的SYN/ACK数据包之间的时间。 
#		值“ -1”表示从未建立连接。 

#	Tr：服务器响应时间（仅HTTP模式）。 这是从haproxy与real server服务器已经建立TCP连接 到 real server服务器发送其完整的响应头 之间的时间。 它仅显示其请求处理时间，而不会由于数据传输而导致网络开销。值得注意的是，当客户端有数据要发送到服务器时（例如在POST请求期间），该时间已经在运行，这可能会明显歪曲的响应时间。 因此，对于从不受信任的网络后面的客户端发起的POST请求，不要过多地信任此字段。 
#		值“ -1”表示从未看到最后一个响应头（空行），这很可能是因为服务器设法处理请求之前的服务器超时行程。

#   Td: 从Ta字段中，我们可以通过减去有效的其他计时器来推断出数据传输时间“ Td”：Td = Ta-（TR + Tw + Tc + Tr）带有“ -1”值的计时器必须从该方程式中排除。 请注意，“ Ta”永远不能为负。

#	Ta：HTTP请求的总活动时间，从haproxy收到请求标头的第一个字节到haproxy发出响应正文的最后一个字节之间的时间。 指定了“ logasap”选项时除外。 在这种情况下，它仅等于（TR + Tw + Tc + Tr），并且以“ +”号作为前缀。 

# |   | %ST  | status_code                                   | numeric     | 响应状态码 200 返回给客户端的HTTP状态代码。此状态通常由服务器设置，但是当无法访问服务器503(建立)/502(r/w)或服务器的响应被haproxy阻止时，也可以由haproxy设置


# |   | %B   | bytes_read           (from server to client)  | numeric     | real server响应报文大小(包括http首部) 22705bytes. 22K
# | H | %CC  | captured_request_cookie                       | string      | - cookie名称及其最大长度由前端配置中的“捕获cookie”语句定义。未设置选项时，该字段为单破折号（'-'）。只能捕获一个cookie，它通常用于跟踪客户端和服务器之间的会话ID交换，以检测由于应用程序错误导致的客户端之间的会话交叉。
# Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
# Accept-Encoding: gzip, deflate
# Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
# Cache-Control: no-cache
# Connection: keep-alive
# Cookie: wp-settings-time-1=1606816504; SERVER-COOKIE=192.168.101.248

# Haproxy配置
# mode http
# capture cookie wp-settings-time-1 len 32   

# 请求日志 Dec 11 13:35:25 localhost haproxy[30702]: 172.27.3.1:64839 [11/Dec/2020:13:35:22.841] nginx-log nginx-log/192.168.101.248 0/0/1/2455/2486 200 22705 wp-settings-time-1=1606816504 - --VN 1/1/0/0/0 0/0 {|Mozilla/5.0 (Wi|172.27.0.248:8192|} {} "GET / HTTP/1.1"



# | H | %CS  | captured_response_cookie                      | string      | -
# | H | %tsc | termination_state with cookie status          | string      | --NI 
	# 会话结束时会话所处的条件。 这表明会话状态，是哪一侧导致了会话结束发生，是由于什么原因（超时，错误等），就像在TCP日志中一样，以及最后两个字符中关于cookie的持久性操作的信息。 正常标志应以“-”开头，表示会话被任一端关闭，缓冲区中没有剩余数据。
	#  -- 正常终止。
	# 参考http://cbonte.github.io/haproxy-dconv/2.3/configuration.html#8.2.4
	# NI 客户端未提供cookie，响应中插入了一个cookie。 对于“插入”模式下每个用户的首次请求，通常会发生这种插入情况，这使计数真实用户成为一种简便的方法。
	
# |   | %ac  | actconn                                       | numeric     | 1
	# 记录会话时进程上并发连接的总数。 当每进程系统限制per-process limit已经达到, 检测这个非常有用。 例如，如果在发生多个连接错误时actconn接近512，则很有可能系统限制进程最多使用1024个文件描述符，并且全部使用了它们。 请参阅第3节“全局参数”以了解如何调整系统。
	# ulimit -n 每进程限制配置
# |   | %fc  | feconn     (frontend concurrent connections)  | numeric     | 1
	# “ feconn”是记录会话时前端上的并发连接总数。 估计维持高负载所需的资源量，并检测何时达到前端的“ maxconn”，这很有用。 通常，当此值急剧增加时，这是因为后端服务器上存在拥塞，但有时可能是由于拒绝服务攻击引起的。
	
# |   | %bc  | beconn      (backend concurrent connections)  | numeric     | 0
	# “ beconn”是记录会话时后端处理的并发连接总数。 它包括服务器上活动的并发连接总数以及队列中暂挂的连接数。 估计支持给定应用程序的高负载所需的其他服务器的数量很有用。 通常，当此值急剧增加时，这是因为后端服务器上存在拥塞，但有时可能是由于拒绝服务攻击引起的。
# |   | %sc  | srv_conn     (server concurrent connections)  | numeric     | 0
	# “ srv_conn”是记录会话时服务器上仍处于活动状态的并发连接总数。 它永远不能超过服务器的已配置“ maxconn”参数。 如果此值通常接近或等于服务器的“ maxconn”，则表示流量调节涉及很多，这意味着服务器的maxconn值太低，或者没有足够的real server用最佳响应时间来处理负载。 当只有一个服务器的“ srv_conn”很高时，通常意味着该服务器有麻烦，导致连接处理的时间比其他服务器上的要长。
# |   | %rc  | retries                                       | numeric     | 0
	# -“重试”是尝试连接到服务器时此会话经历的连接重试次数。 通常，它必须为零，除非在尝试连接的同时停止服务器。 重试频率通常表明real server与haproxy之间存在网络问题，或者服务器上配置错误的系统积压工作阻止了新连接排队。 此字段可以选择以'+'前缀作为前缀，表示在初始服务器上已达到最大重试计数后，会话已进行了重新分派。 在这种情况下，出现在日志中的服务器名称是重新分配连接的服务器名称，而不是第一个，尽管例如在散列的情况下有时两者可能相同。 因此，作为一般经验法则，当重试计数前面有“ +”号时，不应将此计数归因于已记录的服务器。
# |   | %sq  | srv_queue                                     | numeric     | 0
	# -“ srv_queue”是服务器队列中在此请求之前处理的请求总数。 当请求尚未通过服务器队列时为零。 通过将花费在队列中的时间除以队列中的请求数，可以估算服务器的近似响应时间。 值得注意的是，如果会话重新分配并经过两个服务器队列，则它们的位置将是累积的。 除非发生重新分配，否则请求不应同时通过服务器队列和后端队列。
# |   | %bq  | backend_queue                                 | numeric     | 0
	#-“ backend_queue”是后端的全局队列中在此请求之前处理的请求总数。 当请求尚未通过全局队列时，该值为零。 这样就可以估计平均队列长度，将其平均除以服务器的“ maxconn”参数即可轻松转换为丢失的服务器数量。 值得注意的是，如果会话遇到重新分派，则它可能会在后端队列中通过两次，然后这两个位置都会累积。 除非发生重新分配，否则请求不应同时通过服务器队列和后端队列。
# |   | %hr  | captured_request_headers default style        | string      |  记录 capture reqeust header <name> len <length>
# 标头最后一次出现的完整值被捕获。该值将被添加到大括号（'{}'）之间的日志中。如果捕获了多个标题，则它们将由竖线（'|'）分隔，并按照在配置中声明的顺序显示。不存在的标头将被记录为一个空字符串。
#在捕获诸如“User-agent”之类的标头时，可能会记录一些空格，从而使日志分析更加困难。因此，如果您知道自己的日志解析器不够智能，无法依靠大括号，那么请谨慎选择要记录的内容。
#捕获的请求标头的数量及其长度没有限制，尽管明智的做法是将它们保持在较低的水平以限制每个会话的内存使用量。为了使同一前端的日志格式保持一致，标头捕获只能在前端声明。无法在“默认”部分中指定捕获。
#如果未启用任何捕获，则不会显示大括号，从而导致剩余字段的移位。
# 上传时的内容长度：Content-length
capture request header Content-length len 15
# 快速区分真实用户和机器人的“用户代理” User-agent
capture request header User-agent len 15
# 在代理环境中查找请求的来源 X-Forwarded-For
capture request header X-Forwarded-For len 15
# 防盗链, 仅在定位到图片服务器使用。
capture request header Referer len 15


# |   | %hs  | captured_response_headers default style       | string      |  记录capture response header <name> len <length>。 
#如果未启用任何捕获，则不会显示大括号，从而导致剩余字段的移位。

# 响应标头捕获的常见用法包括“ Content-length”标头，指示预期返回多少字节，
capture response header Content-length len 9
# “位置”标头可跟踪重定向。
capture response header Location len 15


# | H | %r   | http_request                                  | string      | "GET / HTTP/1.1" 完整的HTTP请求行，包括方法，请求和HTTP版本字符串。
# +---+------+-----------------------------------------------+-------------+
# | R | var  | field name (8.2.2 and 8.2.3 for description)  | type        |
# +---+------+-----------------------------------------------+-------------+
# |   | %o   | special variable, apply flags on all next var |             |
# +---+------+-----------------------------------------------+-------------+

# |   | %H   | hostname                                      | string      |
# | H | %HM  | HTTP method (ex: POST)                        | string      |
# | H | %HP  | HTTP request URI without query string (path)  | string      |
# | H | %HQ  | HTTP request URI query string (ex: ?bar=baz)  | string      |
# | H | %HU  | HTTP request URI (ex: /foo?bar=baz)           | string      |
# | H | %HV  | HTTP version (ex: HTTP/1.0)                   | string      |
# |   | %ID  | unique-id                                     | string      |
# |   | %T   | gmt_date_time                                 | date        |
# |   | %Td  | Td = Tt - (Tq + Tw + Tc + Tr)                 | numeric     |
# |   | %Tl  | local_date_time                               | date        |
# |   | %Th  | connection handshake time (SSL, PROXY proto)  | numeric     |
# | H | %Ti  | idle time before the HTTP request             | numeric     |
# | H | %Tq  | Th + Ti + TR                                  | numeric     |
# |   | %Ts  | timestamp                                     | numeric     |
# |   | %Tt  | Tt                                            | numeric     |
# |   | %U   | bytes_uploaded       (from client to server)  | numeric     |
# |   | %b   | backend_name                                  | string      |
# |   | %bi  | backend_source_ip       (connecting address)  | IP          |
# |   | %bp  | backend_source_port     (connecting address)  | numeric     |
# |   | %ci  | client_ip                 (accepted address)  | IP          |
# |   | %cp  | client_port               (accepted address)  | numeric     |
# |   | %f   | frontend_name                                 | string      |
# |   | %fi  | frontend_ip              (accepting address)  | IP          |
# |   | %fp  | frontend_port            (accepting address)  | numeric     |
# |   | %ft  | frontend_name_transport ('~' suffix for SSL)  | string      |
# |   | %lc  | frontend_log_counter                          | numeric     |
# |   | %hrl | captured_request_headers CLF style            | string list |
# |   | %hsl | captured_response_headers CLF style           | string list |
# |   | %ms  | accept date milliseconds (left-padded with 0) | numeric     |
# |   | %pid | PID                                           | numeric     |
# |   | %rt  | request_counter (HTTP req or TCP session)     | numeric     |
# |   | %s   | server_name                                   | string      |
# |   | %si  | server_IP                   (target address)  | IP          |
# |   | %sp  | server_port                 (target address)  | numeric     |
# | S | %sslc| ssl_ciphers (ex: AES-SHA)                     | string      |
# | S | %sslv| ssl_version (ex: TLSv1)                       | string      |
# |   | %t   | date_time      (with millisecond resolution)  | date        |
# | H | %tr  | date_time of HTTP request                     | date        |
# | H | %trg | gmt_date_time of start of HTTP request        | date        |
# | H | %trl | local_date_time of start of HTTP request      | date        |
# |   | %ts  | termination_state                             | string      |
# +---+------+-----------------------------------------------+-------------+

# R = Restrictions : H = mode http only ; S = SSL only

```













