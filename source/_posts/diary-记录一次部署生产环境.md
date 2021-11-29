---
title: "记录一次部署生产环境"
date: 2021-02-26 11:39:58
tags:
- "个人日记"
---


本地开发环境，使用kubeadm部署kubernetes集群参考: [kubeadm高可用kubernetes](http://blog.mykernel.cn/2021/02/07/%E5%9F%BA%E4%BA%8Ekubeadm%E5%AE%9E%E7%8E%B0%E9%AB%98%E5%8F%AF%E7%94%A8kubernetes%E9%9B%86%E7%BE%A4/)

![image-20210226201646703](http://myapp.img.mykernel.cn/image-20210226201646703.png)

# 部署要点
<!--more-->

- 事先规划master3节点，haproxy2节点，harbor2节点，之后ip递增为node节点

- 环境：kvm使用virtio [k8s异常](http://blog.mykernel.cn/2021/02/26/kubernetes%E5%BC%82%E5%B8%B8%E5%B4%A9%E6%BA%83/)

- master必须使用SSD：使用更多master节点：5个master

  ```bash
  root@node06:/data# sysbench --test=fileio --num-threads=50 --file-total-size=2G --file-test-mode=rndrw parepare
   2147483648 bytes written in 10.72 seconds (190.97 MiB/sec).
   
  root@node06:/data# sysbench --test=fileio --num-threads=50 --file-total-size=2G --file-test-mode=rndrw run
  File operations:
      reads/s:                      6318.74
      writes/s:                     4212.33
      fsyncs/s:                     13468.88
  
  #至少这个配置才可以上master, 否则就单机docker
  
  
  # 清理
  root@node06:/data# sysbench --test=fileio --num-threads=50 --file-total-size=2G --file-test-mode=rndrw cleanup
  ```
  
- kubeadm init使用命令时，需要注意 apiserver的域名



# 部署步骤 

1. containerd + runc

2. kubeadm + kubelet + kubectl

3. flannel + directrouting

   

# 镜像制作

## 基础镜像

### centos

```bash
FROM centos:7.5.1804
LABEL AUTHOR=2192383945@qq.com

ENV LANG=zh_CN.UTF-8 TZ=Asia/Shanghai

# 基础环境
RUN yum install -y vim wget tree  lrzsz gcc gcc-c++ automake pcre pcre-devel zlib zlib-devel openssl openssl-devel iproute net-tools iotop ntpdate lsof bzip2 tzdata

# 中文
RUN yum -y install kde-l10n-Chinese  glibc-common && localedef -c -f UTF-8 -i zh_CN zh_CN.utf8
	
# ssh服务
RUN yum install -y openssh-server openssh-clients
RUN sed -i 's@#PermitRootLogin prohibit-password@PermitRootLogin yes@g' /etc/ssh/sshd_config && ssh-keygen -A

# 密码
RUN echo 'MBZh/TAmW9CynO0' | passwd --stdin root

# 前台命令
ENTRYPOINT  /usr/sbin/sshd -D
```

### ubuntu

```bash
FROM ubuntu:18.04
LABEL AUTHOR=2192383945@qq.com

ENV LANG=zh_CN.UTF-8 TZ=Asia/Shanghai

# 基础环境
RUN sed -i.bak -e 's@archive.ubuntu.com@mirrors.aliyun.com@g'  -e 's@security.ubuntu.com@mirrors.aliyun.com@g' /etc/apt/sources.list && \
	apt update && \
	apt install iproute2 ntpdate tcpdump telnet traceroute nfs-kernel-server nfs-common lrzsz tree openssl libssl-dev libpcre3 libpcre3-dev zlib1g-dev ntpdate tcpdump telnet traceroute gcc make openssh-server lrzsz tree openssl libssl-dev libpcre3 libpcre3-dev zlib1g-dev ntpdate tcpdump telnet traceroute iotop unzip zip lsof make curl iputils-ping net-tools vim tzdata -y && \
	apt-get install language-pack-zh* -y && echo 'LANG="zh_CN.UTF-8"' > /etc/default/locale && dpkg-reconfigure --frontend=noninteractive locales && update-locale LANG=zh_CN.UTF-8


# 添加ssh服务 docker -p xx:22
RUN sed -i 's@#PermitRootLogin prohibit-password@PermitRootLogin yes@g' /etc/ssh/sshd_config && \
	service ssh restart

# 密码
RUN echo 'root:MBZh/TAmW9CynO0' | chpasswd

# 前台命令
ENTRYPOINT  /usr/sbin/sshd -D
```

### alpine

我选择alpine, 大小小

```bash
root@harbor01:~/weizhixiu# cat dockerfile/baseimage-dockerfile/Dockerfile 
FROM alpine

LABEL AUTHOR=songliangcheng QQ=1062670898

#更改源
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories
#安装常用软件
RUN apk update && apk add bash tzdata iotop gcc gcc libgcc libc-dev libcurl libc-utils pcre-dev zlib-dev libnfs make pcre pcre2 zip unzip net-tools pstree wget libevent libevent-dev iproute2 tcpdump vim
# 这里是安装glibc,参考https://github.com/sgerrand/alpine-pkg-glibc 官方文档
RUN apk --no-cache add ca-certificates && \
wget -q -O /etc/apk/keys/sgerrand.rsa.pub https://alpine-pkgs.sgerrand.com/sgerrand.rsa.pub 

COPY glibc-2.33-r0.apk  glibc-bin-2.33-r0.apk   glibc-i18n-2.33-r0.apk  ./
RUN apk add glibc-2.33-r0.apk  glibc-bin-2.33-r0.apk  glibc-i18n-2.33-r0.apk && rm -f glibc-2.33-r0.apk  glibc-bin-2.33-r0.apk  glibc-i18n-2.33-r0.apk
RUN echo 'en_AG en_AU en_BW en_CA en_DK en_GB en_HK en_IE en_IN en_NG en_NZ en_PH en_SG en_US en_ZA en_ZM en_ZW zh_CN zh_HK zh_SG zh_TW zu_ZA' | xargs -n1 | xargs -I {} /usr/glibc-compat/bin/localedef -i {} -f UTF-8 {}.UTF-8

# 语言 时区
ENV LANG=zh_CN.UTF-8 TZ=Asia/Shanghai

CMD ["/bin/bash"]
```

> 常用包
>
> 中文
>
> glibc, 运行java命令
>
> 时区，java开发需要借助系统时间：参考：https://blog.csdn.net/isea533/article/details/87261764

之后直接构建推送harbor

```bash
root@harbor01:~/weizhixiu# cat dockerfile/baseimage-dockerfile/build.sh 
docker build -t harbor.youwoyouqu.io/baseimage/alpine:base ./


docker push harbor.youwoyouqu.io/baseimage/alpine:base
```

验证时间和语言

```bash
root@master01:~# docker run --rm -it harbor.youwoyouqu.io/baseimage/alpine:base sh
/ # date
Fri Mar  5 10:09:23 CST 2021
/ # echo $LANG
zh_CN.UTF-8
```



## jdk镜像

```bash
root@harbor01:~/weizhixiu# cat dockerfile/jdk-dockerfile/Dockerfile 
FROM harbor.youwoyouqu.io/baseimage/alpine:base


LABEL AUTHOR=songliangcheng QQ=1062670898

ENV JAVA_HOME=/usr/local/jdk  
ENV TOMCAT_HOME=/apps/tomcat 
ENV PATH=\$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$TOMCAT_HOME/bin:$PATH 
ENV CLASSPATH=.$CLASSPATH:$JAVA_HOME/lib:$JAVA_HOME/jre/lib:$TOMCAT_HOME/lib/tools.jar

ADD jdk-8u221-linux-x64.tar.gz /usr/local/src/
RUN ln -sv /usr/local/src/jdk1.8.0_221 /usr/local/jdk 
CMD ["java","-version"]



root@harbor01:~/weizhixiu# cat dockerfile/jdk-dockerfile/build.sh 
docker build -t harbor.youwoyouqu.io/baseimage/jdk:8 .
docker run --rm  harbor.youwoyouqu.io/baseimage/jdk:8

docker push harbor.youwoyouqu.io/baseimage/jdk:8
```

验证日期和时间

```bash
root@master01:~# docker run --rm -it harbor.youwoyouqu.io/baseimage/jdk:8 sh
/ # date
Fri Mar  5 10:11:40 CST 2021
/ # echo $LANG
zh_CN.UTF-8
```

## nginx部署

### 镜像

#### 基础镜像

```bash
root@harbor01:~/weizhixiu/dockerfile/nginx-baseimage-dockerfile# cat Dockerfile 
FROM harbor.youwoyouqu.io/baseimage/alpine:base

LABEL AUTHOR=songliangcheng QQ=1062670898

RUN apk add --no-cache --virtual .build-deps \
  gcc \
  libc-dev \
  make \
  openssl-dev \
  pcre-dev \
  zlib-dev \
  linux-headers \
  curl \
  gnupg \
  libxslt-dev \
  gd-dev \
  geoip-dev \
  perl-dev 
  
ADD nginx-1.16.1.tar.gz ./
RUN cd nginx-1.16.1 && \
 ./configure --prefix=/apps/nginx \
 --user=root \
 --group=root \
 --pid-path=/run/nginx.pid \
 --with-threads \
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
 --with-pcre && \
    make -j `grep -c processor /proc/cpuinfo` && make install 

RUN install -dv /apps/nginx/conf/conf.d/
ADD nginx.conf /apps/nginx/conf/nginx.conf

CMD ["/apps/nginx/sbin/nginx", "-g", "daemon off;"]
```

准备优化后nginx配置

```bash
root@harbor01:~/weizhixiu/dockerfile/nginx-baseimage-dockerfile# cat nginx.conf 
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

```bash
root@harbor01:~/weizhixiu/dockerfile/nginx-baseimage-dockerfile# cat build.sh 
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-02-26
#FileName：             build.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************


docker build -t harbor.youwoyouqu.io/baseimage/nginx:base ./
docker push harbor.youwoyouqu.io/baseimage/nginx:base

```

#### k8s依赖的nginx

```bash
root@harbor01:~/weizhixiu/dockerfile/nginx-controller-dockerfile# cat Dockerfile 
FROM harbor.youwoyouqu.io/baseimage/nginx:base

LABEL AUTHOR=songliangcheng QQ=1062670898


ADD default.conf /apps/nginx/conf/conf.d/
ADD dist /apps/home/
```



### 构建和更新脚本

```bash
root@harbor01:~/weizhixiu# cat dockerfile/nginx-controller-dockerfile/build.sh 
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-02-26
#FileName：             build.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************


[ -z "$1" ] && echo "$0 <tag>" && exit 1

docker build -t harbor.youwoyouqu.io/weizhixiu/nginx-controller:${1} .
docker push harbor.youwoyouqu.io/weizhixiu/nginx-controller:${1}

# 更新1
#kubectl set image deploy -n weizhixiu nginx-controller-deploy nginx-controller-pod=harbor.youwoyouqu.io/weizhixiu/nginx-controller:${1} --record

# 修改
sed -i "s@- image: .*@- image: harbor.youwoyouqu.io/weizhixiu/nginx-controller:${1}@g" /root/weizhixiu/yaml/nginx-controller-yaml/app.yaml

# 更新2
kubectl apply -f /root/weizhixiu/yaml/nginx-controller-yaml/app.yaml --record


kubectl apply -f /root/weizhixiu/yaml/nginx-controller-yaml/hpa.yaml --record
```

> 前期只有build和推送，但是后期构建之后，结合yaml文件就需要更新镜像了

### 回滚脚本

```bash
root@harbor01:~/weizhixiu# cat dockerfile/nginx-controller-dockerfile/rollout.sh 
kubectl rollout -n weizhixiu undo  deployments      nginx-controller-deploy
```

### yaml文件

```diff
root@harbor01:~/weizhixiu# cat  /root/weizhixiu/yaml/nginx-controller-yaml/app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-controller-deploy-label
  name: nginx-controller-deploy
  namespace: weizhixiu
spec:
+  strategy:        # 完成滚动升级，先加后删除
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  replicas: 1
  selector:
    matchLabels:
      app: nginx-controller-deploy-selector
  template:
    metadata:
      labels:
        app: nginx-controller-deploy-selector
    spec:
+      imagePullSecrets: # 私有harbor认证
      - name: registry
#      nodeSelector:
#        group: weizhixiu
      containers:
      - image: harbor.youwoyouqu.io/weizhixiu/nginx-controller:202102261911
        name: nginx-controller-pod
+        resources: # 资源限制
          limits:
            cpu: 10m
            memory: 50Mi
          requests:
            cpu: 10m
            memory: 50Mi
+        ports:      # 容器端口，由于nginx接入k8s流量，所以在这里https会话卸载
        - containerPort: 80
          protocol: TCP
          name: http
        - containerPort: 443
          protocol: TCP
          name: https
+        livenessProbe:   # nginx不正常自动重启
          initialDelaySeconds: 10
          successThreshold: 5
          failureThreshold: 2 
          periodSeconds: 3
          httpGet:
            port: 80
            scheme: HTTP
            path: /
+        readinessProbe:    # nginx不正常，应该不接入service
+          initialDelaySeconds: 10   # nginx启动多久开始检测是否可以接入svc, 应该是估计nginx正常的时间
          successThreshold: 5
          failureThreshold: 2 
          periodSeconds: 3 
          httpGet:
            port: 80
            scheme: HTTP
            path: /
+        lifecycle:     # 停止前等待一下，可能还有流量没有处理完。filebeat日志也没有收集完
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 20"]
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx-controller-label
  name: nginx-controller
  namespace: weizhixiu
spec:
  ports:
  - name: 443-443
    port: 443
    protocol: TCP
    targetPort: 443
+    nodePort: 30443     # 前端接入haproxy -> nginx controller
  - name: 80-80
    port: 80
    protocol: TCP
    targetPort: 80
+    nodePort: 30088
  selector:
    app: nginx-controller-deploy-selector
  type: NodePort
```

> [initial时间配置](http://blog.mykernel.cn/2020/11/13/%E7%94%9F%E4%BA%A7k8s%E7%9A%84deploy%E6%A3%80%E6%B5%8B%E9%85%8D%E7%BD%AE/)
>
> 内存 不需要太多，nginx1M可以维护1万并发。
>
> cpu nginx非常吃，访问慢，直接升cpu

> 后端：请求资源需要给相对大一些， 2C4G/4C8G. 1）容器预留资源，防止因资源不足被其他进程抢用，导致服务不可用。2）schedule调度使用。也需要限制。
>
> nginx: 请求资源可以100m, 70Mi. 不限制。

### 配置自动伸缩

```bash
root@harbor01:~/weizhixiu# cat  /root/weizhixiu/yaml/nginx-controller-yaml/hpa.yaml 
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  creationTimestamp: null
  name: nginx-controller-deploy
spec:
  maxReplicas: 2
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-controller-deploy
  targetCPUUtilizationPercentage: 80
status:
  currentReplicas: 0
  desiredReplicas: 0
```

## java微服务部署

### 镜像

基于上面制作的jdk制作镜像

```bash
root@harbor01:~/weizhixiu# cat  /root/weizhixiu/dockerfile/auth-server-dockerfile/Dockerfile 
FROM harbor.youwoyouqu.io/baseimage/jdk:8
LABEL LABEL AUTHOR=songliangcheng QQ=1062670898

ADD auth-server-1.0.0-SNAPSHOT.jar run.sh /apps/
WORKDIR /apps/
CMD ["/apps/run.sh"]
```

### 启动脚本

准备启动脚本

```bash
root@harbor01:~/weizhixiu# cat  /root/weizhixiu/dockerfile/auth-server-dockerfile/run.sh 
#!/bin/sh
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-02-26
#FileName：             run.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************





install -d /apps/logs
#nohup  java -jar auth-server-1.0.0-SNAPSHOT.jar --spring.profiles.active=test -server  -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=5  	&>> /apps/logs/app.log &
#tail -f /etc/hosts
# test
 java -jar auth-server-1.0.0-SNAPSHOT.jar --spring.profiles.active=test -server  -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=5 
```

> 前期跑起来，方便看日志。所以直接使用这个
>
> 后期需要收集日志，参考[pod日志](http://blog.mykernel.cn/2020/12/11/k8s-filebeat%E6%94%B6%E9%9B%86pod%E6%97%A5%E5%BF%97/)
>
> java的选项参考 [jvm调优](http://blog.mykernel.cn/2020/12/17/tomcat%E7%9A%84jvm%E8%B0%83%E4%BC%98/)

### 构建和更新脚本

准备构建脚本

```bash
root@harbor01:~/weizhixiu# cat  /root/weizhixiu/dockerfile/auth-server-dockerfile/build.sh 
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-02-26
#FileName：             build.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************


[ -z "$1" ] && echo "$0 <tag>" && exit 1

docker build -t harbor.youwoyouqu.io/weizhixiu/auth-server:${1} .
docker push harbor.youwoyouqu.io/weizhixiu/auth-server:${1}


# 更新
#kubectl set image deploy -n weizhixiu auth-server-deploy auth-server-pod=harbor.youwoyouqu.io/weizhixiu/auth-server:${1} --record
# 修改
sed -i "s@- image: .*@- image: harbor.youwoyouqu.io/weizhixiu/auth-server:${1}@g" /root/weizhixiu/yaml/auth-server-yaml/app.yaml
 kubectl apply -f /root/weizhixiu/yaml/auth-server-yaml/app.yaml --record

kubectl apply -f /root/weizhixiu/yaml/auth-server-yaml/hpa.yaml --record
```

> 前期使用"修改下面的内容"
>
> 后期正常后，使用更新下面的，最下面的就注释，保留`kubectl set image + sed -i "s@- i....修改镜像(避免k8s集群宕机，直接apply yaml还原即可)`。原因： 之前发现
>
> ```bash
> kubectl apply -f file --record 会一下删除所有pod, 生产流量一下就中断
> 
> kubectl set image --record 就会一个一个的更新镜像
> ```

认证镜像

```bash
 kubectl create -n weizhixiu secret docker-registry registry --docker-username=admin --docker-password=xxx --docker-server=harbor.youwoyouqu.io
```



### 回滚脚本

准备回滚脚本

```bash
root@harbor01:~/weizhixiu# cat  /root/weizhixiu/dockerfile/auth-server-dockerfile/rollout.sh 
kubectl rollout -n weizhixiu undo  deployments      auth-server-deploy
```

### deploy yaml

准备yaml

```bash
root@harbor01:~/weizhixiu# cat  /root/weizhixiu/yaml/auth-server-yaml/app.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: auth-server-deploy-label
  name: auth-server-deploy
  namespace: weizhixiu
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  replicas: 2
  selector:
    matchLabels:
      app: auth-server-deploy-selector
  template:
    metadata:
      labels:
        app: auth-server-deploy-selector
    spec:
      imagePullSecrets:
      - name: registry
#      nodeSelector:
#        group: weizhixiu
      containers:
      - image: harbor.youwoyouqu.io/weizhixiu/auth-server:2021022637
        name: auth-server-pod
        resources:
          limits:
            cpu: 500
            memory: 500Mi
          requests:
            cpu: 200m
            memory: 200Mi
        ports:
        - containerPort: 8080
          protocol: TCP
          name: http
        livenessProbe: # 重启
          initialDelaySeconds: 60       # java启动比较慢
          periodSeconds: 3
          httpGet:
            port: 8080
            scheme: HTTP
            path: /healthz
#          tcpSocket:
#            port: 8080
        readinessProbe: # 接入service
          initialDelaySeconds: 60
          successThreshold: 10
          failureThreshold: 1 
          periodSeconds: 3 
          httpGet:
            port: 8080
            scheme: HTTP
            path: /healthz
#          tcpSocket:
#            port: 8080
        lifecycle:
          preStop: # 停止前等待一下，可能还有流量没有处理完。filebeat日志也没有收集完
            exec:
              command: ["/bin/sh", "-c", "sleep 20"]  # 一定要使用sh, 因为alpine基础镜像
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: auth-server-label
  name: auth-server
  namespace: weizhixiu
spec:
  ports:
  - name: 8080-8080
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: auth-server-deploy-selector
  type: ClusterIP
```

> 与nginx几乎一样，只是
>
> 1. http接口是/healthz,  要求开发加上，本身java对内存消耗大，可能会出现内存泄漏，端口在，但是服务不可用。那个时候如果readinessprobe基于端口检测是否正常，用户访问过来就是502。所以使用url探测的结果来决定是否接入service.
>2. 端口一般为8080，如果没有就tcpsocket检测避免开发监听端口有问题，容器也正常

！！！！千万不能给大内存，否则集群起来，抗不住

### hpa

```bash
root@harbor01:~/weizhixiu# cat  /root/weizhixiu/yaml/auth-server-yaml/hpa.yaml 
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  creationTimestamp: null
  name: auth-server-deploy
spec:
  maxReplicas: 2
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: auth-server-deploy
  targetCPUUtilizationPercentage: 80
status:
  currentReplicas: 0
  desiredReplicas: 0
```

### 部署下一个微服务

假如下一个叫`user-server`

```bash
root@harbor01:~/weizhixiu/dockerfile# pwd
/root/weizhixiu/dockerfile
# 准备dockerfile
cp -a auth-server-dockerfile/ user-server-dockerfile/
# 更新名称
cd user-server-dockerfile/
sed -i 's@auth-server@user-server@gp' build.sh Dockerfile rollout.sh run.sh 


# 准备yaml
cp -a auth-server-yaml/ user-server-yaml/
sed -i 's@auth-server@user-server@g' app.yaml hpa.yaml
```

# 部署结果

haproxy(4层) -> nginx(7层) -> java微服务

![image-20210226202331467](http://myapp.img.mykernel.cn/image-20210226202331467.png)

# node节点管理镜像

```bash
# 拉
crictl pull --creds admin:xxxx xyz-harbor.com:7443/redis-test/nginx:latest
```



# 部署dashboard

开发需要看日志

```bash
root@harbor01:~/weizhixiu/yaml/dashboard# wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.2.0/aio/deploy/recommended.yaml
root@harbor01:~/weizhixiu/yaml/dashboard# grep image: recommended.yaml 
          image: harbor.youwoyouqu.io/pub/dashboard:v2.2.0
          image: harbor.youwoyouqu.io/pub/metrics-scraper:v1.0.6
root@harbor01:~/weizhixiu/yaml/dashboard# 

```

准备拉镜像

```bash
root@harbor01:~/weizhixiu/dockerfile/dashboard# cat image.sh 
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-03-01
#FileName：             image.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************
docker pull  kubernetesui/metrics-scraper:v1.0.6 
docker tag  kubernetesui/metrics-scraper:v1.0.6 harbor.youwoyouqu.io/pub/metrics-scraper:v1.0.6
docker pull kubernetesui/dashboard:v2.2.0
docker tag kubernetesui/dashboard:v2.2.0 harbor.youwoyouqu.io/pub/dashboard:v2.2.0
```

更新yaml后apply

```diff
    spec:
      containers:
        - name: kubernetes-dashboard
          image: harbor.youwoyouqu.io/pub/dashboard:v2.2.0
          imagePullPolicy: Always
+          resources:
            limits:
              cpu: 500                               
              memory: 500Mi
            requests:
              cpu: 200m
              memory: 200Mi
          ports:
            - containerPort: 8443
              protocol: TCP 
          args:
            - --auto-generate-certificates
            - --namespace=kubernetes-dashboard 
+            - --token-ttl=86400                                                                                                                                                                                                                                                                                            
            # Uncomment the following line to manually specify Kubernetes API server Host
            # If not specified, Dashboard will attempt to auto discover the API server and connect
            # to it. Uncomment only if the default does not work.
            # - --apiserver-host=http://my-address:port
          volumeMounts:
            - name: kubernetes-dashboard-certs
              mountPath: /certs
              # Create on-disk volume to store exec logs
            - mountPath: /tmp
              name: tmp-volume

```

> 避免经常token过期

编辑dashboard的service, 添加nodeport http://192.168.0.87:32091/

```bash
root@harbor01:~/weizhixiu/yaml/dashboard# kubectl edit svc -n kubernetes-dashboard  kubernetes-dashboard 
 22   ports:
 23   - port: 443
 24     protocol: TCP
 25     targetPort: 8443
 `26     nodePort: 32091`
 27   selector:
 28     k8s-app: kubernetes-dashboard
 29   sessionAffinity: None
 `30   type: NodePort`

```

![image-20210301171952755](http://myapp.img.mykernel.cn/image-20210301171952755.png)

添加开发用户，参考：http://blog.mykernel.cn/2021/01/29/%E8%AE%A4%E8%AF%81%E3%80%81%E6%8E%88%E6%9D%83%E5%8F%8A%E5%87%86%E5%85%A5%E6%8E%A7%E5%88%B6/

授权开发对名称空间只读

```bash
root@harbor01:~/weizhixiu/yaml/dashboard# kubectl create clusterrole ns-reader --verb=get,list,watch --resource=namespace 
```

授权开发对weizhixiu名称空间有管理权限

添加用户

```bash
curl -L https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -o cfssl; chmod +x cfssl; curl -L https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -o cfssljson; chmod +x cfssljson; curl -L https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -o cfssl-certinfo; chmod +x cfssl-certinfo;
mkdir cert
cd cert
root@harbor01:~/weizhixiu/yaml/dashboard/cert# cat common-config.json 
{
    "signing": {
        "default": {
            "expiry": "876000h" 
        },
        "profiles": {
            "kubernetes-ca": { 
                "expiry": "876000h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            },
            "etcd-ca": {
                "expiry": "87600h",  
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            },
            "kubernetes-front-proxy-ca": {
                "expiry": "87600h", 
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}


#准备ca证书
root@node06:~/weizhixiu/yaml/dashboard/cert# pwd
/root/weizhixiu/yaml/dashboard/cert
root@node06:~/weizhixiu/yaml/dashboard/cert# cp /etc/kubernetes/pki/ca.* ./


root@harbor01:~/weizhixiu/yaml/dashboard/cert# cat client-csr.json 
{
  "CN": "devops",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names":[{
    "C": "China",
    "ST": "SiChuan",
    "L": "ChengDu",
    "O": "system:Huayang",
    "OU": "system"
  }],
    "hosts": [
	]
}


../cfssl gencert -ca=ca.crt -ca-key=ca.key      --config=common-config.json -profile=kubernetes-ca      client-csr.json | ../cfssljson -bare devops
```

> 用户名 CN=devops

```bash
# /root/weizhixiu/yaml/dashboard/cert
root@harbor01:~/weizhixiu/yaml/dashboard/cert# kubectl config set-cluster  mykube --server="https://kubernetes-api.youwoyouqu.io:6443" --certificate-authority=ca.crt   --kubeconfig=../login/devops.kubeconfig
Cluster "mykube" set.
root@harbor01:~/weizhixiu/yaml/dashboard/cert# kubectl config set-credentials devops --client-certificate=devops.pem --client-key=devops-key.pem --username=devops --embed-certs=true --kubeconfig=../login/devops.kubeconfig
User "devops" set.
root@harbor01:~/weizhixiu/yaml/dashboard/cert# kubectl config set-context devops@mykube --cluster=mykube --user=devops --kubeconfig=../login/devops.kubeconfig
Context "devops@mykube" created.
root@harbor01:~/weizhixiu/yaml/dashboard/cert# kubectl config use-context devops@mykube --kubeconfig=../login/devops.kubeconfig
Switched to context "devops@mykube".


root@harbor01:~/weizhixiu/yaml/dashboard# kubectl config view --kubeconfig ../login/devops.kubeconfig 
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /root/weizhixiu/yaml/dashboard/cert/ca.crt
    server: https://kubernetes-api.youwoyouqu.io:6443
  name: mykube
contexts:
- context:
    cluster: mykube
    user: devops
  name: devops@mykube
current-context: devops@mykube
kind: Config
preferences: {}
users:
- name: devops
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
    username: devops

```

用户读pod

```bash
root@node06:~/weizhixiu/yaml/dashboard/cert# kubectl get pods --kubeconfig ../login/devops.kubeconfig 
Error from server (Forbidden): pods is forbidden: User "devops" cannot list resource "pods" in API group "" in the namespace "default"
```

> 失败

授权证书申请的CN=devops，即devops用户有读ns权限 

```bash
harbor01:~/weizhixiu/yaml/dashboard/cert# kubectl create clusterrolebinding devops-read-ns --clusterrole=ns-reader   --user=devops


root@node06:~/weizhixiu/yaml/dashboard/cert# kubectl get ns --kubeconfig ../login/devops.kubeconfig 
NAME                   STATUS   AGE
default                Active   14h
jenkins                Active   12h
kube-node-lease        Active   14h
kube-public            Active   14h
kube-system            Active   14h
kubernetes-dashboard   Active   12h
weizhixiu              Active   13h

root@node06:~/weizhixiu/yaml/dashboard/cert# kubectl get pods --kubeconfig ../login/devops.kubeconfig 
Error from server (Forbidden): pods is forbidden: User "devops" cannot list resource "pods" in API group "" in the namespace "default"
root@node06:~/weizhixiu/yaml/dashboard/cert# kubectl get pods --kubeconfig ../login/devops.kubeconfig  -n weizhixiu
Error from server (Forbidden): pods is forbidden: User "devops" cannot list resource "pods" in API group "" in the namespace "weizhixiu"

```

> 用户可以查看ns, 但是不能读pod

授权用户对weizhixiu名称空间有admin权限

```bash
root@harbor01:~/weizhixiu/yaml/dashboard/cert# kubectl create -n weizhixiu rolebinding devops-admin-weizhixiu --clusterrole=admin   --user=devops
rolebinding.rbac.authorization.k8s.io/devops-admin-weizhixiu created
root@node06:~/weizhixiu/yaml/dashboard/cert# kubectl get pods --kubeconfig ../login/devops.kubeconfig  -n weizhixiu
NAME                                           READY   STATUS    RESTARTS   AGE
auth-server-deploy-6bfd865c68-nss42            1/1     Running   0          13h
auth-server-deploy-6bfd865c68-s7pfc            1/1     Running   0          18m
nginx-controller-deploy-5b5ff8f86d-mfll4       1/1     Running   0          56m
nginx-controller-deploy-5b5ff8f86d-th2h2       1/1     Running   0          18m
user-server-deploy-5bfd97965f-55jgw            1/1     Running   0          18m
user-server-deploy-5bfd97965f-mt2v9            1/1     Running   0          13h
video-comment-server-deploy-5655d8fc9d-h5g9d   1/1     Running   0          18m
video-comment-server-deploy-5655d8fc9d-q5kv6   1/1     Running   0          13h
video-server-deploy-75bff48bc-7prf5            1/1     Running   0          18m
video-server-deploy-75bff48bc-8f5b7            1/1     Running   0          13h
wzx-admin-gateway-deploy-6fcbf4785-cg5m2       1/1     Running   0          13h
wzx-admin-gateway-deploy-6fcbf4785-wvpcs       1/1     Running   0          18m
wzx-mobile-gateway-deploy-78bfc4fb5-ggzzx      1/1     Running   0          18m
wzx-mobile-gateway-deploy-78bfc4fb5-pk5v9      1/1     Running   0          13h
```

现在将这个文件测试kubernetes-dashboard加载。。。。。

测试结果，失败



## 基于ns读+ns admin创建开发的认证kubeconfig

证书生成的结果，测试此kubeconfig, dashboard不得行，所以使用serviceaccount的token生成kubeconfig

```bash
root@harbor01:~/weizhixiu/yaml/dashboard# kubectl create sa  -n kube-system devops
TOKEN=$(kubectl get secret -n kube-system $(kubectl describe sa  -n kube-system devops | awk '/Tokens:/{print $2}') -o jsonpath='{.data.token}' | base64 -d)
root@harbor01:~/weizhixiu/yaml/dashboard# kubectl config set-credentials devops --token=${TOKEN}   --kubeconfig=./login/devops.kubeconfig 

# 现在添加了token, kubeconfig文件可以加载了，但是没有权限

# 绑定读名称空间
root@harbor01:~/weizhixiu/yaml/dashboard# kubectl create  clusterrolebinding devops-read-ns-2 --clusterrole=ns-reader --serviceaccount=kube-system:devops

# 现在网页可以读ns

# 有weizhixiu管理权限
root@harbor01:~/weizhixiu/yaml/dashboard# kubectl create -n weizhixiu rolebinding devops-admin-weizhixiu-2 --clusterrole=admin --serviceaccount=kube-system:devops



#下载root@node06:~/weizhixiu/yaml/dashboard/cert# sz ../login/devops.kubeconfig 
```

## 基于cluster-admin的token认证dashboard

```bash
kubectl create sa  -n kube-system admin
TOKEN=$(kubectl get secret -n kube-system $(kubectl describe sa  -n kube-system admin | awk '/Tokens:/{print $2}') -o jsonpath='{.data.token}' | base64 -d)
root@harbor01:~/weizhixiu/yaml/dashboard/cert# pwd
/root/weizhixiu/yaml/dashboard/cert
kubectl config set-credentials admin --token=${TOKEN} --kubeconfig=../login/admin.kubeconfig

root@harbor01:~/weizhixiu/yaml/dashboard/cert# kubectl config set-cluster  mykube --server="https://kubernetes-api.youwoyouqu.io:6443" --certificate-authority=ca.crt   --kubeconfig=../login/admin.kubeconfig 
Cluster "mykube" set.
root@harbor01:~/weizhixiu/yaml/dashboard# kubectl config set-context admin@mykube --cluster=mykube --user=admin --kubeconfig=../login/admin.kubeconfig 
Context "admin@mykube" created.
root@harbor01:~/weizhixiu/yaml/dashboard# kubectl config use-context admin@mykube --kubeconfig=../login/admin.kubeconfig 
Switched to context "admin@mykube".

# 查看权限
root@node06:~/weizhixiu/yaml/dashboard/cert# kubectl get pods --kubeconfig=../login/admin.kubeconfig 
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:kube-system:admin" cannot list resource "pods" in API group "" in the namespace "default"


# 绑定admin-cluster
root@harbor01:~/weizhixiu/yaml/dashboard# kubectl create  clusterrolebinding admin-cluster-yunwei --clusterrole=cluster-admin --serviceaccount=kube-system:admin

# 再次查看权限

#下载root@node06:~/weizhixiu/yaml/dashboard/cert# sz ../login/admin.kubeconfig 

```



# 快速添加node --- 本地集群抗不住压力，重置集群的问题

## 初始化主机 -- 无论添加master/node均需要先执行

添加之前需要初始化机器

```bash
root@master02:~# cat cleankubeadm.sh
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-03-03
#FileName：             cleankubeadm.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************



# 停止容器
docker stop $(docker ps -a -q)
docker rm $(docker ps -a -q)
crictl ps  | awk '{print $1}' | xargs -I {} crictl stop {}
crictl ps -a | awk '{print $1}' | xargs -I {} crictl rm {}

# kubernetes
rm -rf /etc/kubernetes  

# etcd
rm -fr /etc/etcd.env  /var/lib/etcd  /var/etcd/

# flannel
rm -fr  /var/lib/cni/ /etc/cni /etc/flanneld  /var/run/flannel/
ifconfig flannel.1 down

# kubelet kube-proxy
systemctl stop kubelet
mount | grep '/var/lib/kubelet'| awk '{print $3}'|xargs sudo umount
rm -fr /var/lib/kubelet /var/libe/kube-proxy


# 网卡
ifconfig docker0 down
ip link del docker0
ifconfig cni0 down
ip link delete cni0
ifconfig flannel.1 down
ip link delete flannel.1

# 清理容器数据
#rm -fr /data/containerd /var/lib/docker /var/lib/containerd
systemctl restart containerd

# 清理iptables
sudo iptables -F && sudo iptables -X && sudo iptables -F -t nat && sudo iptables -X -t nat

# 重启
reboot
```

## 初始化node环境

准备一个脚本，方便后期快速添加node节点

需要事先在master节点完成

```bash
ssh-keygen
ssh-copy-id localhost
```

直接通过链接`http://myapp.img.qiniu.mykernel.cn/kubeadm-tools.tar.xz`下载所有文件，核心脚本如下：

```bash
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-03-02
#FileName：             addnode.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************

[ -z "$1" -o -z "$2" ] && echo "$0 <ip> <password>" && exit

if ! ping -c 1 -w 1 $1 &> /dev/null ; then
	echo $1 not online....
	exit
fi


if ! which sshpass &> /dev/null ; then
	echo please install sshpass...
	exit
fi



sshpass  -p ${2} scp -o StrictHostKeyChecking=no -rp ~/.ssh ${1}:




# 添加node
scp -rp containerd/  ${1}:/etc/
scp -rp crictl.yaml   ${1}:/etc/
scp -rp containerd.service   ${1}:/etc/systemd/system/containerd.service
scp containerd-1.4.1-linux-amd64.tar.gz ${1}:
scp kubelet ${1}:/etc/default/kubelet
scp hosts ${1}:/etc/hosts
scp -rp runc ${1}:/usr/local/bin/runc


ssh ${1} "tar --no-overwrite-dir -C /usr/local/ -xvzf containerd-1.4.1-linux-amd64.tar.gz"
# 添加harbor证书
scp harbor-ca.pem ${1}:/usr/share/ca-certificates/harbor-ca.crt
ssh ${1} "if ! grep -q harbor-ca.crt /etc/ca-certificates.conf; then  echo 'harbor-ca.crt' >>  /etc/ca-certificates.conf; fi"
ssh ${1} update-ca-certificates
ssh ${1} systemctl restart containerd
ssh ${1} systemctl enable containerd

# 安装kubeadm
ssh ${1} apt-get install -y apt-transport-https
ssh ${1} "curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -"
scp kubernetes.list ${1}:/etc/apt/sources.list.d/kubernetes.list
ssh ${1} apt update
ssh ${1} apt install kubeadm=1.20.1-00 kubelet=1.20.1-00  -y


# 配置ipvs
ssh ${1} apt install -y ipset ipvsadm
scp 10-k8s-modules.conf ${1}:/etc/modules-load.d/10-k8s-modules.conf
for i in br_netfilter ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh nf_conntrack_ipv4; do ssh ${1} "modprobe -v $i"; done


# 手工加入
echo manual join master
```

准备目录结构如下 

```bash
root@harbor01:~/linux_scripts/kubeadm-tools# ls
10-k8s-modules.conf  addnode.sh  containerd  containerd-1.4.1-linux-amd64.tar.gz  containerd.service  crictl.yaml  harbor-ca.pem  hosts  kubelet  kubernetes.list  README.md  runc

root@harbor01:~/linux_scripts/kubeadm-tools# cat 10-k8s-modules.conf
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack_ipv4
```

## 快速添加node/master角色

添加前确认apiserver地址正常

```bash
root@master01:~# cat /etc/hosts # 至少与master节点有相同的api解析记录
127.0.0.1	localhost
127.0.1.1	ubuntu

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

192.168.0.248 kubernetes-api.mykernel.cn



root@haproxy01:~# cat /etc/haproxy/haproxy.cfg
# 加注释: web监控页面, 宋亮成 N49
listen stats    
  #bind-process 1                    # 默认展示所有进程或 bind-process all
  bind :9999
  # 启用状态页
  stats enable
  
  # 隐藏haproxy版本，安全
  stats hide-version
  
  # 自动刷新时间间隔
  stats refresh 3s
  
  # 自定义状态页uri, 默认/haproxy?stats
  stats uri /haproxy
  
  # 账户认证时的提示信息，
  stats realm HAPorxy\ Stats\ Page
  
  # 认证时的用户和密码
  stats auth admin:123456
  stats auth haadmin:123456
  
  # 启用stats页面的管理功能
  stats admin if TRUE
  
listen apiserver-6443
    bind 192.168.0.248:6443
    mode tcp
    server 192.168.0.4 192.168.0.4:6443  check inter 3s fall 2 rise 5
    server 192.168.0.5 192.168.0.5:6443  check inter 3s fall 2 rise 5
    server 192.168.0.6 192.168.0.6:6443  check inter 3s fall 2 rise 5
    server 192.168.0.8 192.168.0.8:6443  check inter 3s fall 2 rise 5
    server 192.168.0.10 192.168.0.10:6443  check inter 3s fall 2 rise 5
```

访问：http://192.168.0.248:9999/haproxy

![image-20210304104025536](http://myapp.img.mykernel.cn/image-20210304104025536.png)

准备master加入node的token

```bash
# 查看token
root@master01:~# kubeadm token list
TOKEN                     TTL         EXPIRES                     USAGES                   DESCRIPTION                                                EXTRA GROUPS
abcdef.0123456789abcdef   14h     

# 删除token
root@master01:~# kubeadm token delete abcdef.0123456789abcdef

# 加入命令
root@master01:~# kubeadm token create --print-join-command --config ./kubeadm-v1.20.1.yaml  
```

添加master

```bash
# kubeadm init phase upload-certs --upload-certs

c19c27a0b345756a213c414d2c20b093e494ad48fd2cf3428fbd3c9afa38d60b


kubeadm join kubernetes-api.youwoyouqu.io:6443 --token abcdef.0123456789abcdef     --discovery-token-ca-cert-hash sha256:44a120669f327d259fac5ca874307029e117b5c9f9f1a5127471505ccebee135  --ignore-preflight-errors=swap \
--control-plane --certificate-key 1f4bb92e479d51bb592fd66466cce64f0665283917c4690d068e3a7c23ec19a8
```

> 由于本地环境的k8s集群 etcd性能差，所以使用5节点

注意：在升级certs证书后

```bash
error execution phase preflight: couldn't validate the identity of the API Server: cluster CA found in cluster-info ConfigMap is invalid: none of the public keys "sha256:44a120669f327d259fac5ca874307029e117b5c9f9f1a5127471505ccebee135" are pinned

#将错误提示的hash值代替原先获得的hash值即可
https://cloud.tencent.com/developer/news/268376
```





# 部署hpa依赖的组件

metrics-server

prometheus, adaptor也提供

```bash
 wget https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.4.2/components.yaml

docker pull k8s.gcr.io/metrics-server/metrics-server:v0.4.2
# 需要代理解决

docker tag k8s.gcr.io/metrics-server/metrics-server:v0.4.2 harbor.youwoyouqu.io/pub/metrics-server:v0.4.2
```

编辑 components.yaml 

```diff
  template:
    metadata:
      labels:
        k8s-app: metrics-server
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
+        - --kubelet-insecure-tls
+        #image: k8s.gcr.io/metrics-server/metrics-server:v0.4.2                                                                                                                                                                                                                                                            
+        image:  harbor.youwoyouqu.io/pub/metrics-server:v0.4.2
        imagePullPolicy: IfNotPresent
```

```bash
kubectl apply -f components.yaml  --record

 kubectl get apiservice
v1beta1.metrics.k8s.io                 kube-system/metrics-server   True        3m17s
```

获取top信息

```bash
root@node06:/data/weizhixiu/yaml/metrics-server# kubectl top node
NAME                     CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
master01.youwoyouqu.io   260m         13%    1142Mi          60%       
master02.youwoyouqu.io   195m         9%     1107Mi          58%       
master03.youwoyouqu.io   190m         9%     1132Mi          59%       
node01.youwoyouqu.io     95m          1%     2480Mi          64%       
node02.youwoyouqu.io     59m          0%     2230Mi          58%       
node03.youwoyouqu.io     59m          0%     2453Mi          63%       
node04.youwoyouqu.io     132m         3%     1855Mi          26%       
node06.youwoyouqu.io     43m          2%     1467Mi          19%       
```



# CD部署---Jenkins

[kubernetes部署jenkins](http://blog.mykernel.cn/2021/03/01/kubernetes-deploy-jenkins/)

[使用jenkins](https://www.mykernel.cn/jenkins_v1.html#more)

https://www.cnblogs.com/kevingrace/p/6479813.html

 Build when a change is pushed to GitLab. GitLab webhook  -> 高级 -> Secret token -> generate



首先得调整jenkins的使用内存限制

```bash
 41         resources: 
 42           limits:
 43             cpu: 500m
 44             memory: 500Mi
 45           requests:
 46             cpu: 200m
 47             memory: 200Mi
```

发版

```bash
root@harbor01:~/weizhixiu/dockerfile/jenkins-dockerfile# bash build.sh 202103032116
```





## webhook配置

安装以下插件

```bash
Gitlab Authentication plugin
Gitlab Hook Plugin
Role-based Authorization Strategy #权限管理
Blue Ocean #流水线形式
Authorize Project Plugin
```

### gitlab配置

![image-20210302144043159](http://myapp.img.mykernel.cn/image-20210302144043159.png)

![image-20210302144110769](http://myapp.img.mykernel.cn/image-20210302144110769.png)

通过点击测试push，在jenkins上完成项目构建

![image-20210302144141295](http://myapp.img.mykernel.cn/image-20210302144141295.png)

### jenkins上配置

![image-20210302144433970](http://myapp.img.mykernel.cn/image-20210302144433970.png)

#### 描述字段

业务 ，哪个开发名

![image-20210302144512456](http://myapp.img.mykernel.cn/image-20210302144512456.png)

#### 配置gitlab地址

![image-20210302144550691](http://myapp.img.mykernel.cn/image-20210302144550691.png)

先点添加

![image-20210302144758816](http://myapp.img.mykernel.cn/image-20210302144758816.png)

#### 构建触发器

通过gitlab webhook触发

![image-20210302145033184](http://myapp.img.mykernel.cn/image-20210302145033184.png)

#### 构建环境

添加时间和构建前删除工作区

![image-20210302145142288](http://myapp.img.mykernel.cn/image-20210302145142288.png)

#### 构建过程

增加构建步骤中选shell构建即可，以下是一段测试构建语句

```bash
printenv       # 查看可用的环境变量
echo hello     # 验证命令 
pwd            # 验证当前目录
ls             # 验证内容
kubectl get ns # 验证k8s通
```

![image-20210302145228761](http://myapp.img.mykernel.cn/image-20210302145228761.png)

#### 新项目

复制即可

![image-20210302150111636](http://myapp.img.mykernel.cn/image-20210302150111636.png)

## 结合gitlab + jenkins

在gitlab界面点push

![image-20210302145428151](http://myapp.img.mykernel.cn/image-20210302145428151.png)

![image-20210302145438625](http://myapp.img.mykernel.cn/image-20210302145438625.png)

在jenkins界面查看 

![image-20210302145504248](http://myapp.img.mykernel.cn/image-20210302145504248.png)

点击蓝色圆圈，即可查看日志

![image-20210302145536858](http://myapp.img.mykernel.cn/image-20210302145536858.png)

## 自动发布微服务

### java/npm适用项目

一个gitlab项目对应一个一级目录

![image-20210304132636658](http://myapp.img.mykernel.cn/image-20210304132636658.png)

先在jenkins shell方框测试执行 `npm run build`, 测试发布, 正常后在

```bash
printenv
echo hello
pwd
kubectl get ns
# 生成node_modules
npm  --registry=https://registry.npm.taobao.org install
npm run build
```

成功之后，需要将打包的文件复制到registry主机



### jenkins与镜像服务器免密密钥互信

需要把自已的公钥的复制到registry主机，为了方便，直接在registry主机进行如下操作

```bash
ssh-keygen -t rsa -b 4096 -P '' -f ~/.ssh/id_rsa
ssh-copyid localhost
```

直接将，~/.ssh目录复制到jenkins容器的家目录即可



jenkins的Dockerfile

```bash
root@node06:~/weizhixiu/dockerfile/jenkins-dockerfile# cat Dockerfile 
FROM harbor.youwoyouqu.io/baseimage/jdk:8

ADD kubectl /usr/bin/
ADD config  /root/.kube/

ADD jenkins.war .
ENV JENKINS_HOME=/opt/data


# git
RUN apk add --no-cache git openssh 

# vue项目
RUN apk add --no-cache npm maven

# 打包结果复制到镜像主机
COPY .ssh /root/.ssh/

CMD ["java", "-jar", "/jenkins.war", "--httpPort=9090","-server -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=5"]
```

> 其他文件参考：http://blog.mykernel.cn/2021/03/01/kubernetes-deploy-jenkins/



jenkins的pod, 测试复制到镜像主机的dockerfile目录

```bash
 ssh 192.168.0.27 "cd /data/weizhixiu/dockerfile/nginx-controller-dockerfile && unzip dist.zip && echo $(date +'%Y%m%d%H%M%S')"
```

现在可以在网页端项目的配置中写脚本了

```bash
# 生成node_modules
npm  --registry=https://registry.npm.taobao.org install
# 生成网页
npm run build
# 发布版本
ssh 192.168.0.27 "cd /data/weizhixiu/dockerfile/nginx-controller-dockerfile && bash clean.sh && unzip dist.zip && bash build.sh $(date +'%Y%m%d%H%M%S')"
```

> 在镜像目录先准备一个清理文件的脚本
>
> ```bash
> root@node06:~/weizhixiu/dockerfile/nginx-controller-dockerfile# cat clean.sh 
> rm -fr dist*
> ```



### 修改k8s coredns的配置支持解析不了的域名走内部DNS --- 重要

制作镜像的主机IP可能会变，所以使用域名, 域名已经在外部DNS解析，所以直接将K8S中的DNS上游修改为外部DNS

```diff
  5 apiVersion: v1
  6 data:
  7   Corefile: |
  8     .:53 {
  9         errors
 10         health {
 11            lameduck 5s
 12         }
 13         ready
 14         kubernetes youwoyouqu.local in-addr.arpa ip6.arpa {
 15            pods insecure
 16            fallthrough in-addr.arpa ip6.arpa
 17            ttl 30
 18         }
 19         prometheus :9153
- 20         #forward . /etc/resolv.conf {
+ 21         forward . 192.168.0.77 {                                                                                                                                                                                                                                                                                       
 22            max_concurrent 1000
 23         }
 24         cache 30
 25         loop
 26         reload
 27         loadbalance
 28     } 
 29 kind: ConfigMap
```

> 配置coredns使用外部dns
>
> 可能需要删除dns pods来加载配置

需要删除k8s中的coredns pod

```bash\
kubectl delete pods -n kube-system -l k8s-app=kube-dns
```



### 脚本调用启动jenkins

为了避免使用jenkins的开发乱修改jenkins中的配置，使用脚本调用，先准备脚本

```bash
# /root/weizhixiu/dockerfile/jenkins-dockerfile/scripts
install -dv scripts
cd scripts
root@node06:~/weizhixiu/dockerfile/jenkins-dockerfile/scripts# cat admin-web-new.sh 
#!/bin/sh
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-03-02
#FileName：             admin-web-new.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************

# 生成node_modules
npm  --registry=https://registry.npm.taobao.org install
# 生成网页
npm run build
# 发布版本，先清理
ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd /data/weizhixiu/dockerfile/nginx-controller-dockerfile  && bash clean.sh"
scp -o StrictHostKeyChecking=no dist.zip make-registry.youwoyouqu.io:/data/weizhixiu/dockerfile/nginx-controller-dockerfile
ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd /data/weizhixiu/dockerfile/nginx-controller-dockerfile  && unzip dist.zip && bash build.sh $(date +'%Y%m%d%H%M%S')"

```

> 由于alpine没有bash, 所以写#!/bin/sh
>
> ```bash
> / # ping make-registry.youwoyouqu.io
> PING make-registry.youwoyouqu.io (192.168.0.27): 56 data bytes
> 64 bytes from 192.168.0.27: seq=0 ttl=63 time=0.694 ms
> 64 bytes from 192.168.0.27: seq=1 ttl=63 time=0.833 ms
> ^C
> --- make-registry.youwoyouqu.io ping statistics ---
> 2 packets transmitted, 2 packets received, 0% packet loss
> round-trip min/avg/max = 0.694/0.763/0.833 ms
> ```

```bash
chmod +x admin-web-new.sh
```

 配置jenkins Dockerfile加载脚本

```diff
root@node06:~/weizhixiu/dockerfile/jenkins-dockerfile# cat Dockerfile 
FROM harbor.youwoyouqu.io/baseimage/jdk:8

ADD kubectl /usr/bin/
ADD config  /root/.kube/

ADD jenkins.war .
ENV JENKINS_HOME=/opt/data


# git
RUN apk add --no-cache git openssh 

# vue项目
RUN apk add --no-cache npm maven

# 打包结果复制到镜像主机
COPY .ssh /root/.ssh/

+# 准备项目脚本
+# scripts/* -> /opt/scripts/
+ADD scripts /opt/scripts/

CMD ["java", "-jar", "/jenkins.war", "--httpPort=9090","-server -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=5"]
```

配置jenkins使用脚本

```bash
cd ${WORKSPACE}
sh  /opt/scripts/admin-web-new.sh
```

![image-20210302183850582](http://myapp.img.mykernel.cn/image-20210302183850582.png)

### 优化npm, 提升构建速度 -- (高阶应用)

优化npm, 参考：https://zhuanlan.zhihu.com/p/83793004

所以使用专用的镜像来制作npm, 在镜像服务器完成

```bash
root@node06:~/weizhixiu/deploy/npm-dockerfile# ls
admin-web-new  build.sh  Dockerfile

```

构建Dockerfile

```bash
root@node06:~/weizhixiu/deploy/npm-dockerfile# cat Dockerfile 
FROM harbor.youwoyouqu.io/baseimage/alpine:base
# vue项目
RUN apk add --no-cache npm 

# 1. 只有部分文件如 package.json 发生变更的时候，才重新执行 npm install
COPY package.json .
RUN npm install --registry=https://registry.npm.taobao.org

# 2. 剩余的文件
COPY . .
RUN npm run build

```

构建脚本

```bash
root@node06:~/weizhixiu/deploy/npm-dockerfile# cat build.sh 
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-02-26
#FileName：             build.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************


[ -z "$1" ] && echo "$0 <tag>" && exit 1
image=harbor.youwoyouqu.io/devops/npm:${1}

# 先清理网页包
rm -rf dist*
ls
sleep 2

cp -a Dockerfile admin-web-new/
docker build -t ${image}  admin-web-new/
#不需要推送只是构建的临时容器
#docker push     ${image}


# 克隆镜像构建结果，并打包
CONTAINERNAME=$(openssl rand -hex 5)
docker run --rm -itd --name ${CONTAINERNAME}  ${image} sh
docker cp   ${CONTAINERNAME}:dist ./
zip -r dist.zip dist

# 删除容器,并清理这个镜像
docker kill ${CONTAINERNAME}
docker rmi   ${image}
```

现在Jenkins脚本修改，为先把拉下来的文件上传至镜像服务器，然后构建出结果（可以利用缓存），将结果发布为nginx-controller

```bash
root@node06:~/weizhixiu/dockerfile/jenkins-dockerfile/scripts# cat admin-web-new.sh
#!/bin/sh
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-03-02
#FileName：             admin-web-new.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************

# 先将镜像服务器网页文件清理
ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd /data/weizhixiu/deploy/npm-dockerfile && rm -fr admin-web-new"

# 上传网页
cd ..
rm -fr admin-web-new/node_modules/ admin-web-new/dist.zip
zip -r admin-web-new.zip admin-web-new
scp -o StrictHostKeyChecking=no admin-web-new.zip make-registry.youwoyouqu.io:/data/weizhixiu/deploy/npm-dockerfile
ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd /data/weizhixiu/deploy/npm-dockerfile && unzip admin-web-new.zip && rm -fr admin-web-new.zip"

# 构建镜像
ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd /data/weizhixiu/deploy/npm-dockerfile && bash build.sh"

# 下载网页
rm -fr admin-web-new.zip                 
cd admin-web-new
scp -o StrictHostKeyChecking=no make-registry.youwoyouqu.io:/data/weizhixiu/deploy/npm-dockerfile/dist.zip ./


# 发布版本，先清理
ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd /data/weizhixiu/dockerfile/nginx-controller-dockerfile  && bash clean.sh"
scp -o StrictHostKeyChecking=no dist.zip make-registry.youwoyouqu.io:/data/weizhixiu/dockerfile/nginx-controller-dockerfile
ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd /data/weizhixiu/dockerfile/nginx-controller-dockerfile  && unzip dist.zip && bash build.sh $(date +'%Y%m%d%H%M%S')"
```

> 每一行，都事先在k8s jenkins pods中一条条测试执行
>
> ```bash
> /opt/data/workspace # ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.i
> o "cd /data/weizhixiu/deploy/npm-dockerfile && bash build.sh"                   
>         
> admin-web-new
> build.sh
> Dockerfile
> Sending build context to Docker daemon  12.28MB
> Step 1/6 : FROM harbor.youwoyouqu.io/baseimage/alpine:base
>  ---> 4a1e171841b5
> Step 2/6 : RUN apk add --no-cache npm
>  ---> Using cache
>  ---> 26d5820ec3c4
> Step 3/6 : COPY package.json .
>  ---> Using cache # 只要这个文件不变，就一直使用缓存
>  ---> 2548ff104085
> Step 4/6 : RUN npm install --registry=https://registry.npm.taobao.org
>  ---> Using cache # 可以看到使用了缓存
>  ---> 91ecbd8b1ba7
> Step 5/6 : COPY . .
>  ---> 6ae2bb146edd
> Step 6/6 : RUN npm run build
>  ---> Running in 9e5d8e2114bd
> 
> > wxz-admin@2.0.0 build /
> > vue-cli-service build
> 
>  WARN  "baseUrl" option in vue.config.js is deprecated now, please use "publicPath" instead.
> 
> -  Building for production..
> ```

现在发布jenkins

```bash
root@node06:/data/weizhixiu/dockerfile/jenkins-dockerfile# bash build.sh 202103021231
```

现在通过gitlab的webhook来发起请求



### 分层构建 --- 深度优化npm + nginx -- (高阶应用)

上面先构建出镜像销毁，之后把结果部署到nginx, 通过分层构建完成

```bash
root@node06:/data/weizhixiu/nginx-controller-dockerfile# cat Dockerfile 
FROM harbor.youwoyouqu.io/baseimage/alpine:base as builder
# vue项目
RUN apk add --no-cache npm 

# 1. 只有部分文件如 package.json 发生变更的时候，才重新执行 npm install
COPY package.json .
RUN npm install --registry=https://registry.npm.taobao.org

# 2. 剩余的文件
COPY . .
RUN npm run build


FROM harbor.youwoyouqu.io/baseimage/nginx:base

LABEL AUTHOR=songliangcheng QQ=1062670898


ADD default.conf /apps/nginx/conf/conf.d/
COPY --from=build /dist /apps/home/
#ADD dist /apps/home/
```

1. 上层镜像也会在镜像中？

   ```bash
   root@node06:~/weizhixiu/deploy/npm-dockerfile# bash build.sh  # 上层镜像单独打包
   root@node06:~/weizhixiu/deploy/npm-dockerfile# docker images harbor.youwoyouqu.io/devops/npm:b97680e69ab8
   REPOSITORY                        TAG            IMAGE ID       CREATED          SIZE
   harbor.youwoyouqu.io/devops/npm   b97680e69ab8   789e3294426f   13 seconds ago   681MB
   ```

   ```bash
   root@node06:~/weizhixiu/dockerfile/nginx-controller-dockerfile# docker build -t harbor.youwoyouqu.io/weizhixiu/nginx-controller:2021030311 . # 下层镜像单独打包
    root@node06:~/weizhixiu/dockerfile/nginx-controller-dockerfile# docker images harbor.youwoyouqu.io/weizhixiu/nginx-controller:2021030311
   REPOSITORY                                        TAG          IMAGE ID       CREATED        SIZE
   harbor.youwoyouqu.io/weizhixiu/nginx-controller   2021030311   ed65b3a025db   12 hours ago   381MB
   ```

   分层镜像结果只有下层镜像

   ```bash
   root@node06:/data/weizhixiu/nginx-controller-dockerfile# docker build -t harbor.youwoyouqu.io/weizhixiu/nginx-controller:20210303  admin-web-new
   root@node06:/data/weizhixiu/nginx-controller-dockerfile# docker images harbor.youwoyouqu.io/weizhixiu/nginx-controller:20210303
   REPOSITORY                                        TAG        IMAGE ID       CREATED          SIZE
   harbor.youwoyouqu.io/weizhixiu/nginx-controller   20210303   14c34277b00b   12 seconds ago   381MB
   ```

2. 下层镜像会使用缓存？

   通过多次构建发现

   ```bash
   Removing intermediate container 8f6a0ebeb122 # 移除中间容器
    ---> fdfb0a432551
   Step 8/11 : FROM harbor.youwoyouqu.io/baseimage/nginx:base
    ---> bf527d5a41f3
   Step 9/11 : LABEL AUTHOR=songliangcheng QQ=1062670898
    ---> Using cache
    ---> cef4e35623de
   Step 10/11 : ADD default.conf /apps/nginx/conf/conf.d/
    ---> Using cache # 使用缓存
    ---> 465d9c7507ad
   Step 11/11 : COPY --from=builder /dist /apps/home/
    ---> 14c34277b00b
   Successfully built 14c34277b00b
   Successfully tagged harbor.youwoyouqu.io/weizhixiu/nginx-controller:20210303
   ```

#### 重新编排文件

```bash
root@node06:~/weizhixiu/stage-builder# tree . -L 1
.
├── admin-web-new       # admin-web-new项目
├── build.sh            # 构建
├── default.conf        # 默认配置
├── Dockerfile          # Dockerfile
├── nginx-1.16.1.tar.gz # nginx源码
└── rollout.sh          # 回滚脚本
```

```bash
root@node06:~/weizhixiu/stage-builder/admin-web-nginx# cat Dockerfile 
############################# admin-web-new项目, vue前端：彭茂云 #############################
FROM harbor.youwoyouqu.io/baseimage/alpine:base as admin-web-new
# vue项目
RUN apk add --no-cache npm 

# 1. 只有部分文件如 package.json 发生变更的时候，才重新执行 npm install
COPY admin-web-new/package.json .
RUN npm install --registry=https://registry.npm.taobao.org

# 2. 剩余的文件
COPY admin-web-new .
RUN npm run build


############################# 入口, 运维：宋亮成 #############################
FROM harbor.youwoyouqu.io/baseimage/nginx:base

LABEL AUTHOR=songliangcheng QQ=1062670898


ADD default.conf /apps/nginx/conf/conf.d/
COPY --from=admin-web-new /dist /apps/home/
```

准备Dockerfile依赖的文件

```bash
root@node06:~/weizhixiu/stage-builder/admin-web-nginx# cat build.sh 
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-02-26
#FileName：             build.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************


[ -z "$1" ] && echo "$0 <tag>" && exit 1

docker build -t harbor.youwoyouqu.io/weizhixiu/nginx-controller:${1} .
docker push harbor.youwoyouqu.io/weizhixiu/nginx-controller:${1}

# 更新
kubectl set image deploy -n weizhixiu nginx-controller-deploy nginx-controller-pod=harbor.youwoyouqu.io/weizhixiu/nginx-controller:${1} --record
# 修改
sed -i "s@- image: .*@- image: harbor.youwoyouqu.io/weizhixiu/nginx-controller:${1}@g" /root/weizhixiu/yaml/nginx-controller-yaml/app.yaml
#kubectl apply -f /root/weizhixiu/yaml/nginx-controller-yaml/app.yaml --record


#kubectl apply -f /root/weizhixiu/yaml/nginx-controller-yaml/hpa.yaml --record
```

```bash
root@node06:~/weizhixiu/stage-builder/admin-web-nginx# cat rollout.sh 
kubectl rollout -n weizhixiu undo  deployments      nginx-controller-deploy
```

```bash
root@node06:~/weizhixiu/stage-builder/admin-web-nginx# cat default.conf 
server {
    listen       80;
    server_name  test.youwoyouqu.io;
    root   /apps/home;

    location / {
        root   /apps/home;
        index  index.html index.htm;
    }
    location /login {
        root   /apps/home;
		rewrite /login/(.*) /$1 last;
    }

   location /admin {
     proxy_hide_header Access-Control-Allow-Origin;
	add_header Access-Control-Allow-Origin '*';
	add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
	add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization,authorization';

	if ($request_method = 'OPTIONS') {
		return 204;
	}
	proxy_pass http://wzx-admin-gateway:8080;
   }
   location /api {
	proxy_pass http://wzx-mobile-gateway:8080;
   }

    error_page   500 502 503 504  /50x.html;
    error_page   404    /index.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

}
```

#### jenkins配置 分层构建

查看分层构建目录

```bash
root@node06:~/weizhixiu/stage-builder/admin-web-nginx# readlink -f .
/data/weizhixiu/stage-builder/admin-web-nginx

```

编辑jenkins的脚本

```bash
root@node06:~/weizhixiu/dockerfile/jenkins-dockerfile# cat scripts/admin-web-new.sh 
#!/bin/sh
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-03-02
#FileName：             admin-web-new.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************

# 先将镜像服务器网页文件清理
ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd /data/weizhixiu/stage-builder/admin-web-nginx && rm -fr admin-web-new"

# 上传网页
cd ..
rm -fr admin-web-new/node_modules/ admin-web-new/dist.zip
zip -r admin-web-new.zip admin-web-new
scp -o StrictHostKeyChecking=no admin-web-new.zip make-registry.youwoyouqu.io:/data/weizhixiu/stage-builder/admin-web-nginx
ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd /data/weizhixiu/stage-builder/admin-web-nginx && unzip admin-web-new.zip && rm -fr admin-web-new.zip"

# 构建镜像
ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd /data/weizhixiu/stage-builder/admin-web-nginx && bash build.sh $(date +%Y%m%d%H%M%S)"
```

发布jenkins

```bash
Sending build context to Docker daemon  107.5MB
Step 1/10 : FROM harbor.youwoyouqu.io/baseimage/jdk:8
 ---> 6db3df587075
Step 2/10 : ADD kubectl /usr/bin/
 ---> Using cache
 ---> 832fde1b9a97
Step 3/10 : ADD config  /root/.kube/
 ---> Using cache
 ---> e5e698f51858
Step 4/10 : ADD jenkins.war .
 ---> Using cache
 ---> 879c2447d74c
Step 5/10 : ENV JENKINS_HOME=/opt/data
 ---> Using cache
 ---> 532468886ace
Step 6/10 : RUN apk add --no-cache git openssh
 ---> Using cache
 ---> f8a4af1647da
Step 7/10 : RUN apk add --no-cache npm maven
 ---> Using cache
 ---> 498f848ee80a
Step 8/10 : COPY .ssh /root/.ssh/
 ---> Using cache
 ---> 27b7f5e74c2b
Step 9/10 : ADD scripts /opt/scripts/
 ---> 43e49090187e
Step 10/10 : CMD ["java", "-jar", "/jenkins.war", "--httpPort=9090","-server -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=5"]
 ---> Running in f94af38f277d
Removing intermediate container f94af38f277d
 ---> 9b7db0157589
Successfully built 9b7db0157589
Successfully tagged harbor.youwoyouqu.io/devops/jenkins:202103031640
The push refers to repository [harbor.youwoyouqu.io/devops/jenkins]
b2a5ca600fcd: Pushed 
3422c007b3d0: Layer already exists 
5c72be0faaeb: Layer already exists 
8d823f381235: Layer already exists 
f1c06d21086a: Layer already exists 
f7ad265a75b6: Layer already exists 
af59ecdb9671: Layer already exists 
d029e1ed6555: Layer already exists 
fd49a9f05e15: Layer already exists 
ead9b93a3d0e: Layer already exists 
41032bf81c26: Layer already exists 
8f0f1539b4c9: Layer already exists 
a1d9d45fa7df: Layer already exists 
4ddc0fb2452c: Layer already exists 
9111e18d92dc: Layer already exists 
cb381a32b229: Layer already exists 
202103031640: digest: sha256:3a3e601e748eddd470e3c5a0e212d1efac96c73dde456a86add7cf2e2e8b200a size: 3682
deployment.apps/jenkins-deploy image updated
root@node06:/data/weizhixiu/dockerfile/jenkins-dockerfile# 
```

#### jenkins配置job

![image-20210304130056653](http://myapp.img.mykernel.cn/image-20210304130056653.png)

jenkins webhook测试, OK





### 优化java, 提升构建速度 -- (高阶应用)

```bash
root@node06:~/weizhixiu/deploy/maven-dockerfile# ls
build.sh  Dockerfile  settings.xml  weizhixiu2.0
```

配置nexus仓库：

> 安装 http://blog.mykernel.cn/2021/03/03/kubernetes%E6%90%AD%E5%BB%BAnexus/
>
> 配置快照 http://blog.mykernel.cn/2021/03/03/nexus%E4%B8%8A%E4%BC%A0%E5%BF%AB%E7%85%A7/

配置java代码所有项目，走nexus代理仓库 `settings.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>

<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
    <mirrors>
        <mirror>
            <id>nexus</id>
            <mirrorOf>central</mirrorOf>
            <name>weizhixiu internal nexus</name>
            <url>http://192.168.0.246/repository/maven-public/</url>
        </mirror>
    </mirrors>

    <servers>
        <server>
            <id>nexus</id>  <!--//id要与pom.xml中关联私服的id相同, 认证这个用来拉镜像**-->
            <username>admin</username>
            <password>admin123</password>
        </server>
        <server>
            <id>releases-wzx</id>  <!--//id要与pom.xml中关联私服的id相同, 认证这个用来推送快照**-->
            <username>admin</username>
            <password>admin123</password>
        </server>
        <server>
            <id>snapshots-wzx</id>
            <username>admin</username>
            <password>admin123</password>
        </server>
    </servers>
    <profiles>
        <profile>
            <id>nexus</id>
            <repositories>
                <repository>
                    <id>nexus</id>
                    <name>weizhixiu internal nexus</name>
                    <url>http://192.168.0.246/repository/maven-public/</url>
                    <releases>
                        <!-- true表示开启仓库发布版本下载，false表示禁止 -->
                        <enabled>true</enabled>
                    </releases>
                    <snapshots>
                        <!-- true表示开启仓库快照版本下载，false表示禁止 -->
                        <enabled>true</enabled>
                    </snapshots>
                </repository>
            </repositories>

            <pluginRepositories>
                <pluginRepository>
                    <id>nexus</id>
                    <url>http://192.168.0.246/repository/maven-public/</url>
                    <releases>
                        <enabled>true</enabled>
                    </releases>
                    <snapshots>
                        <!-- 禁止快照版本，防止不稳定的插件影响项目构建 -->
                        <enabled>false</enabled>
                    </snapshots>
                </pluginRepository>
            </pluginRepositories>
        </profile>
    </profiles>
    <!-- 激活nexus私服 -->
    <activeProfiles>
        <activeProfile>nexus</activeProfile>
    </activeProfiles>
</settings>
```

> 注意：
>
> - [x] 修改成域名，假设nexus不存在了，可以修改dns,而不用动配置。“失败”
>
> - [x] 修改成VIP，走haproxy反代至nexus。“成功”
>
> 使用Maven部署构件到Nexus私服上日常开发的快照版本部署到Nexus中策略为Snapshot的宿主仓库中，正式项目部署到策略为Release的宿主仓库中，POM的配置方式如下：`<distributionManagement> .... </distributionManagement>` , 替换页尾的`</project>`

```bash
<!-- 发布构件至私服nexus -->
    <distributionManagement>
        <repository>
            <id>nexus-releases</id>
            <name>corp nexus-releases</name>
            <url>http://你的nexusIP:8081/nexus/content/repositories/releases/</url>
        </repository>
        <snapshotRepository>
            <id>nexus-snapshot</id>
            <name>corp nexus-snapshot</name>
            <url>http://你的nexusIP:8081/nexus/content/repositories/snapshots/</url>
        </snapshotRepository>
    </distributionManagement>
</project>
```

并且需要配置推送快照的用户,在**setting.xml**中设置：

```bash
<servers>

    <!-- 发布Releases版的账号，ID要与distributionManagement中的Releases ID一致 -->
    <server>
      <id>nexus-releases</id>
      <username>admin</username>
      <password>******</password>
    </server>
    <!-- 发布snapshot版的账号，ID要与distributionManagement中的snapshot ID一致 -->
    <server>
      <id>nexus-snapshot</id>
      <username>admin</username>
      <password>******</password>
    </server>

 </servers>
```

[maven使用指南](https://www.ibofine.com/mavenbook/chapter8/chapter8.html)

配置构建脚本

```bash
root@node06:~/weizhixiu/deploy/maven-dockerfile# cat build.sh
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-02-26
#FileName：             build.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************


TAGNAME=$(openssl rand -hex 6)
image=harbor.youwoyouqu.io/devops/jdk:$TAGNAME

docker build -t ${image}  ./
```

配置Dockerfile

```bash
root@node06:~/weizhixiu/deploy/maven-dockerfile# cat Dockerfile 
FROM harbor.youwoyouqu.io/baseimage/jdk:8
# java项目
RUN apk add --no-cache maven



# 1. 只有部分文件如 package.json 发生变更的时候，才重新执行 npm install
COPY  weizhixiu2.0/pom.xml                                    weizhixiu2.0/
COPY  weizhixiu2.0/module/pom.xml                             weizhixiu2.0/module/
COPY  weizhixiu2.0/cloud-server/pom.xml                       weizhixiu2.0/cloud-server/
COPY  weizhixiu2.0/cloud-server/user-server/pom.xml           weizhixiu2.0/cloud-server/user-server/
COPY  weizhixiu2.0/cloud-server/wzx-mobile-gateway/pom.xml    weizhixiu2.0/cloud-server/wzx-mobile-gateway/
COPY  weizhixiu2.0/cloud-server/wzx-admin-gateway/pom.xml     weizhixiu2.0/cloud-server/wzx-admin-gateway/
COPY  weizhixiu2.0/cloud-server/payment-server/pom.xml        weizhixiu2.0/cloud-server/payment-server/
COPY  weizhixiu2.0/cloud-server/video-server/pom.xml          weizhixiu2.0/cloud-server/video-server/
COPY  weizhixiu2.0/cloud-server/auth-server/pom.xml           weizhixiu2.0/cloud-server/auth-server/
COPY  weizhixiu2.0/cloud-server/video-comment-server/pom.xml  weizhixiu2.0/cloud-server/video-comment-server/

# 添加配置, 配置mirrors
COPY settings.xml ./
RUN cd weizhixiu2.0 && mvn  dependency:go-offline --settings /settings.xml


# 2. 剩余的文件
###################### auth-server ######################
COPY weizhixiu2.0 weizhixiu2.0/

# 编译auth-server
RUN cd weizhixiu2.0/cloud-server/auth-server/ && mvn clean package -DskipTests --settings /settings.xml
```

> 只要pom.xml和settings.xml不修改都会一直使用缓存
>
> 添加服务时，先添加pom.xml, 再配置编译命令

通过构建可以看到可以重复利用缓存

```diff
+root@node06:~/weizhixiu/deploy/maven-dockerfile# echo 1 > weizhixiu2.0/1111 # 添加一个文件
root@node06:~/weizhixiu/deploy/maven-dockerfile# bash build.sh 
Sending build context to Docker daemon  1.871MB
Step 1/16 : FROM harbor.youwoyouqu.io/baseimage/jdk:8
...
Step 13/16 : COPY settings.xml ./
 ---> Using cache
 ---> d529cb6c38e3
Step 14/16 : RUN cd weizhixiu2.0 && mvn  dependency:go-offline --settings /settings.xml
 ---> Using cache
 ---> c691ff41db86
Step 15/16 : COPY weizhixiu2.0 weizhixiu2.0/
 ---> cd247d775cc8
Step 16/16 : RUN cd weizhixiu2.0/cloud-server/auth-server/ && mvn clean package -DskipTests --settings /settings.xml
+ ---> Running in 2392990fb919 # 使用缓存 

...
[INFO] Replacing main artifact with repackaged archive
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  20.102 s
[INFO] Finished at: 2021-03-03T08:45:39Z
[INFO] ------------------------------------------------------------------------
Removing intermediate container 2392990fb919
 ---> 47e417095e94
Successfully built 47e417095e94
Successfully tagged harbor.youwoyouqu.io/devops/jdk:263695eb2f7b

```

### 分层构建 --- 深度优化 maven + jar -- (高阶应用)

```bash
root@node06:/data/weizhixiu/stage-builder/auth-server# cat Dockerfile
############################# JAVA 所有项目 编译方法, 王申，谢如华 #############################
FROM harbor.youwoyouqu.io/baseimage/jdk:8 AS auth-server

# java项目
RUN apk add --no-cache maven



# 1. 只有部分文件如 package.json 发生变更的时候，才重新执行 npm install
COPY  weizhixiu2.0/pom.xml                                    weizhixiu2.0/
COPY  weizhixiu2.0/module/pom.xml                             weizhixiu2.0/module/
COPY  weizhixiu2.0/cloud-server/pom.xml                       weizhixiu2.0/cloud-server/
COPY  weizhixiu2.0/cloud-server/user-server/pom.xml           weizhixiu2.0/cloud-server/user-server/
COPY  weizhixiu2.0/cloud-server/wzx-mobile-gateway/pom.xml    weizhixiu2.0/cloud-server/wzx-mobile-gateway/
COPY  weizhixiu2.0/cloud-server/wzx-admin-gateway/pom.xml     weizhixiu2.0/cloud-server/wzx-admin-gateway/
COPY  weizhixiu2.0/cloud-server/payment-server/pom.xml        weizhixiu2.0/cloud-server/payment-server/
COPY  weizhixiu2.0/cloud-server/video-server/pom.xml          weizhixiu2.0/cloud-server/video-server/
COPY  weizhixiu2.0/cloud-server/auth-server/pom.xml           weizhixiu2.0/cloud-server/auth-server/
COPY  weizhixiu2.0/cloud-server/video-comment-server/pom.xml  weizhixiu2.0/cloud-server/video-comment-server/

# 添加配置, 配置mirrors
COPY settings.xml ./
RUN cd weizhixiu2.0 && mvn  dependency:go-offline --settings /settings.xml


# 2. 剩余的文件
COPY weizhixiu2.0 weizhixiu2.0/

# 编译auth-server
RUN cd weizhixiu2.0/cloud-server/auth-server/ && mvn clean package -DskipTests --settings /settings.xml


############################# jar, 运维：宋亮成 #############################
FROM harbor.youwoyouqu.io/baseimage/jdk:8
LABEL LABEL AUTHOR=songliangcheng QQ=1062670898

ADD run.sh /apps/
COPY --from=auth-server /weizhixiu2.0/cloud-server/auth-server/target/auth-server-1.0.0-SNAPSHOT.jar /apps/

WORKDIR /apps/
CMD ["/apps/run.sh"]

```

其他依赖文件, 直接复制过来

```bash
root@node06:/data/weizhixiu/stage-builder/auth-server# cat build.sh 
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-02-26
#FileName：             build.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************


[ -z "$1" ] && echo "$0 <tag>" && exit 1

docker build -t harbor.youwoyouqu.io/weizhixiu/auth-server:${1} .
docker push harbor.youwoyouqu.io/weizhixiu/auth-server:${1}


# 更新
#kubectl set image deploy -n weizhixiu auth-server-deploy auth-server-pod=harbor.youwoyouqu.io/weizhixiu/auth-server:${1}
# 修改
sed -i "s@- image: .*@- image: harbor.youwoyouqu.io/weizhixiu/auth-server:${1}@g" /root/weizhixiu/yaml/auth-server-yaml/app.yaml
kubectl apply -f /root/weizhixiu/yaml/auth-server-yaml/app.yaml

kubectl apply -f /root/weizhixiu/yaml/auth-server-yaml/hpa.yaml
```

```bash
root@node06:/data/weizhixiu/stage-builder/auth-server# cat rollout.sh 
kubectl rollout -n weizhixiu undo  deployments      auth-server-deploy
```

```bash
root@node06:/data/weizhixiu/stage-builder/auth-server# cat run.sh 
#!/bin/sh
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-02-26
#FileName：             run.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************





install -d /apps/logs
nohup java -jar auth-server-1.0.0-SNAPSHOT.jar --spring.profiles.active=test -server  -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=5       &> /apps/logs/app.log     &
tail -f /etc/hosts
# test
# java -jar auth-server-1.0.0-SNAPSHOT.jar --spring.profiles.active=test -server  -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=5 
```

```bash
root@node06:/data/weizhixiu/stage-builder/auth-server# cat settings.xml 
<?xml version="1.0" encoding="UTF-8"?>

<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
    <mirrors>
        <mirror>
            <id>nexus</id>
            <mirrorOf>central</mirrorOf>
            <name>weizhixiu internal nexus</name>
            <url>http://192.168.0.246/repository/maven-public/</url>
        </mirror>
    </mirrors>

    <servers>
        <server>
            <id>nexus</id>  <!--//id要与pom.xml中关联私服的id相同, 认证这个用来拉镜像**-->
            <username>admin</username>
            <password>admin123</password>
        </server>
        <server>
            <id>releases-wzx</id>  <!--//id要与pom.xml中关联私服的id相同, 认证这个用来推送快照**-->
            <username>admin</username>
            <password>admin123</password>
        </server>
        <server>
            <id>snapshots-wzx</id>
            <username>admin</username>
            <password>admin123</password>
        </server>
    </servers>
    <profiles>
        <profile>
            <id>nexus</id>
            <repositories>
                <repository>
                    <id>nexus</id>
                    <name>weizhixiu internal nexus</name>
                    <url>http://192.168.0.246/repository/maven-public/</url>
                    <releases>
                        <!-- true表示开启仓库发布版本下载，false表示禁止 -->
                        <enabled>true</enabled>
                    </releases>
                    <snapshots>
                        <!-- true表示开启仓库快照版本下载，false表示禁止 -->
                        <enabled>true</enabled>
                    </snapshots>
                </repository>
            </repositories>

            <pluginRepositories>
                <pluginRepository>
                    <id>nexus</id>
                    <url>http://192.168.0.246/repository/maven-public/</url>
                    <releases>
                        <enabled>true</enabled>
                    </releases>
                    <snapshots>
                        <!-- 禁止快照版本，防止不稳定的插件影响项目构建 -->
                        <enabled>false</enabled>
                    </snapshots>
                </pluginRepository>
            </pluginRepositories>
        </profile>
    </profiles>
    <!-- 激活nexus私服 -->
    <activeProfiles>
        <activeProfile>nexus</activeProfile>
    </activeProfiles>
</settings>
```

#### jenkins配置分层构建

确定当前目录

```bash
root@node06:/data/weizhixiu/stage-builder/auth-server# readlink -f .
/data/weizhixiu/stage-builder/auth-server
```

脚本

```bash
root@node06:/data/weizhixiu/dockerfile/jenkins-dockerfile/scripts# cat auth-server.sh 
#!/bin/sh
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-03-02
#FileName：             ${CODE_DIR}.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************

SERVICE_DIR=/data/weizhixiu/stage-builder/auth-server
CODE_DIR=weizhixiu2.0

# 先将镜像服务器网页文件清理
ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd ${SERVICE_DIR} && rm -fr ${CODE_DIR}"

# 上传网页
cd ..
zip -r ${CODE_DIR}.zip ${CODE_DIR}
scp -o StrictHostKeyChecking=no ${CODE_DIR}.zip make-registry.youwoyouqu.io:${SERVICE_DIR}
ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd ${SERVICE_DIR} && unzip ${CODE_DIR}.zip && rm -fr ${CODE_DIR}.zip"

# 构建镜像
ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd ${SERVICE_DIR} && bash build.sh $(date +%Y%m%d%H%M%S)"
```

发版jenkins

```bash
root@node06:/data/weizhixiu/dockerfile/jenkins-dockerfile# bash build.sh 202103031731
```

#### jenkins配置job

```bash
cd ${WORKSPACE}
sh  /opt/scripts/auth-server.sh
```



![image-20210304130238529](http://myapp.img.mykernel.cn/image-20210304130238529.png)



### 发布多个项目包含在一个目录 --  jenkins参数化构建

如果还是以上方式，需要每个都维护一个脚本, 使用参数化构建直接传递不同名字，调用不同的构建参数即可

![image-20210304132908502](http://myapp.img.mykernel.cn/image-20210304132908502.png)

#### jenkins参数化构建

利用jenkins的参数化构建`Build with params`

![image-20210304133153766](http://myapp.img.mykernel.cn/image-20210304133153766.png)

配置项目

![image-20210304134355941](http://myapp.img.mykernel.cn/image-20210304134355941.png)

第一个是变量

第二个是变量的值

第3个是描述

![image-20210304135121556](http://myapp.img.mykernel.cn/image-20210304135121556.png)

将变量传递给脚本

![image-20210304135234973](http://myapp.img.mykernel.cn/image-20210304135234973.png)

测试使用

![image-20210304135421659](http://myapp.img.mykernel.cn/image-20210304135421659.png)



#### jenkins脚本

传递不同的参数，调用脚本传递参数

```diff
root@node06:~/weizhixiu/dockerfile/jenkins-dockerfile/scripts# cat weizhixiu2.0.sh 
#!/bin/sh
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-03-02
#FileName：             ${CODE_DIR}.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************

SERVICE_DIR=/data/weizhixiu/stage-builder/weizhixiu2.0
CODE_DIR=weizhixiu2.0
+APP_NAME=$1

# 先将镜像服务器网页文件清理
echo "清理代码"
ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd ${SERVICE_DIR} && rm -fr ${CODE_DIR}"
echo "清理代码完毕"
sleep 2


# 上传网页
echo "上传网页"
cd ..
zip -r ${CODE_DIR}.zip ${CODE_DIR}
scp -o StrictHostKeyChecking=no ${CODE_DIR}.zip make-registry.youwoyouqu.io:${SERVICE_DIR}
ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd ${SERVICE_DIR} && unzip ${CODE_DIR}.zip && rm -fr ${CODE_DIR}.zip"
echo "上传网页完毕"
sleep 2

# 构建镜像
echo "构建镜像"
+ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd ${SERVICE_DIR} && bash build.sh $(date +%Y%m%d%H%M%S) ${APP_NAME}"
echo "构建镜像完毕"
```

#### 镜像服务器的Dockerfile -- 参数化构建

构建时传递不同的参数

```diff
root@node06:/data/weizhixiu/stage-builder/weizhixiu2.0# cat Dockerfile
############################# JAVA 所有项目 编译方法, 王申，谢如华 #############################
+FROM harbor.youwoyouqu.io/baseimage/jdk:8 AS weizhixiu2.0

# java项目
RUN apk add --no-cache maven



# 1. 只有部分文件如 package.json 发生变更的时候，才重新执行 npm install
COPY  weizhixiu2.0/pom.xml                                    weizhixiu2.0/
COPY  weizhixiu2.0/module/pom.xml                             weizhixiu2.0/module/
COPY  weizhixiu2.0/cloud-server/pom.xml                       weizhixiu2.0/cloud-server/
COPY  weizhixiu2.0/cloud-server/user-server/pom.xml           weizhixiu2.0/cloud-server/user-server/
COPY  weizhixiu2.0/cloud-server/wzx-mobile-gateway/pom.xml    weizhixiu2.0/cloud-server/wzx-mobile-gateway/
COPY  weizhixiu2.0/cloud-server/wzx-admin-gateway/pom.xml     weizhixiu2.0/cloud-server/wzx-admin-gateway/
COPY  weizhixiu2.0/cloud-server/payment-server/pom.xml        weizhixiu2.0/cloud-server/payment-server/
COPY  weizhixiu2.0/cloud-server/video-server/pom.xml          weizhixiu2.0/cloud-server/video-server/
COPY  weizhixiu2.0/cloud-server/auth-server/pom.xml           weizhixiu2.0/cloud-server/auth-server/
COPY  weizhixiu2.0/cloud-server/video-comment-server/pom.xml  weizhixiu2.0/cloud-server/video-comment-server/

# 添加配置, 配置mirrors
COPY settings.xml ./
RUN cd weizhixiu2.0 && mvn  dependency:go-offline --settings /settings.xml


# 2. 剩余的文件
COPY weizhixiu2.0 weizhixiu2.0/

# 编译${APP_NAME}
+ARG APP_NAME=auth-server
+RUN cd weizhixiu2.0/cloud-server/${APP_NAME}/ && mvn clean package -DskipTests --settings /settings.xml

############################# jar, 运维：宋亮成 #############################
FROM harbor.youwoyouqu.io/baseimage/jdk:8
LABEL LABEL AUTHOR=songliangcheng QQ=1062670898

ADD run.sh /apps/
+RUN sed -i "s@auth-server@${APP_NAME}@g" /apps/run.sh
+ARG APP_NAME=auth-server
+COPY --from=weizhixiu2.0 /weizhixiu2.0/cloud-server/${APP_NAME}/target/${APP_NAME}-1.0.0-SNAPSHOT.jar /apps/

WORKDIR /apps/
CMD ["/apps/run.sh"]
```

#### 镜像服务器的build.sh脚本

配置build.sh脚本

```diff
root@node06:/data/weizhixiu/stage-builder/weizhixiu2.0# cat build.sh
#!/bin/bash
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-02-26
#FileName：             build.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************


[ -z "$1" -o -z "$2" ] && echo "$0 <tag> <app_name>" && exit 1

+APP_NAME=$2
+docker build --build-arg APP_NAME=${APP_NAME} -t harbor.youwoyouqu.io/weizhixiu/${APP_NAME}:${1} .
docker push harbor.youwoyouqu.io/weizhixiu/${APP_NAME}:${1}


# 更新
#kubectl set image deploy -n weizhixiu ${APP_NAME}-deploy ${APP_NAME}-pod=harbor.youwoyouqu.io/weizhixiu/${APP_NAME}:${1}
# 修改
sed -i "s@- image: .*@- image: harbor.youwoyouqu.io/weizhixiu/${APP_NAME}:${1}@g" /root/weizhixiu/yaml/${APP_NAME}-yaml/app.yaml
kubectl apply -f /root/weizhixiu/yaml/${APP_NAME}-yaml/app.yaml

kubectl apply -f /root/weizhixiu/yaml/${APP_NAME}-yaml/hpa.yaml
```



发版jenkins

```bash
root@node06:/data/weizhixiu/dockerfile/jenkins-dockerfile# bash build.sh 202103041340
```





#### 测试跑通

在jenkins的pod中传递参数启动脚本

```bash
/opt/data/workspace/weizhixiu2.0 # sh -x /opt/scripts/weizhixiu2.0.sh video-comment-server
```

跑正常了，在jenkins上点一下

![image-20210304140412464](http://myapp.img.mykernel.cn/image-20210304140412464.png)

然后在gitlab webhook点一下

![image-20210304140601147](http://myapp.img.mykernel.cn/image-20210304140601147.png)

可以发现会自动构建第一个 `auth-server`, 为了简单，直接删除webhook

#### 项目回滚

![image-20210304141430368](http://myapp.img.mykernel.cn/image-20210304141430368.png)

![image-20210304141459651](http://myapp.img.mykernel.cn/image-20210304141459651.png)

修改jenkins脚本

```diff
root@node06:/data/weizhixiu/dockerfile/jenkins-dockerfile# cat scripts/weizhixiu2.0.sh
#!/bin/sh
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-03-02
#FileName：             ${CODE_DIR}.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************

SERVICE_DIR=/data/weizhixiu/stage-builder/weizhixiu2.0
CODE_DIR=weizhixiu2.0
APP_NAME=$1
+dev_operation=$2


if [ "$dev_operation" == "deploy" ]; then
        # 先将镜像服务器网页文件清理
        ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd ${SERVICE_DIR} && rm -fr ${CODE_DIR}"

        # 上传网页
        cd ..
        zip -r ${CODE_DIR}.zip ${CODE_DIR}
        scp -o StrictHostKeyChecking=no ${CODE_DIR}.zip make-registry.youwoyouqu.io:${SERVICE_DIR}
        ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd ${SERVICE_DIR} && unzip ${CODE_DIR}.zip && rm -fr ${CODE_DIR}.zip"

        # 构建镜像
        ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd ${SERVICE_DIR} && bash build.sh $(date +%Y%m%d%H%M%S) ${APP_NAME}"
else
+        ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd ${SERVICE_DIR} && bash rollout.sh ${APP_NAME}"
fi
```

配置rollout.sh

```bash
root@node06:/data/weizhixiu/stage-builder/weizhixiu2.0# cat rollout.sh 
APP_NAME=$1
kubectl rollout -n weizhixiu undo  deployments      ${APP_NAME}-deploy
```

发版jenkins

```bash
root@node06:~/weizhixiu/dockerfile/jenkins-dockerfile# bash build.sh 202103041451
```

网页测试回滚

## jenkins发布落库

年终将mysql库统计一下给老板看jenkins每天什么人发版数量



安装`user build vars plugin`插件获取发布的用户名

> https://www.cnblogs.com/honeybee/p/6525322.html
>
> https://plugins.jenkins.io/build-user-vars-plugin/

离线安装：https://www.cnblogs.com/yy-cola/p/10162062.html

本次测试 1.6版本正常

![image-20210304161157230](http://myapp.img.mykernel.cn/image-20210304161157230.png)

测试跑出结果

```bash
16:12:05 + printenv
16:12:05 JENKINS_HOME=/opt/data
16:12:05 KUBERNETES_PORT=tcp://10.96.0.1:443
16:12:05 GIT_PREVIOUS_SUCCESSFUL_COMMIT=4ac040057a26c82a9c139afde692522b8bf8553c
16:12:05 KUBERNETES_SERVICE_PORT=443
16:12:05 RUN_CHANGES_DISPLAY_URL=http://192.168.0.87:32090/job/test/2/display/redirect?page=changes
16:12:05 HOSTNAME=jenkins-deploy-6cb7d6c888-hbvvr
16:12:05 BUILD_USER=微知秀
16:12:05 NODE_LABELS=master
16:12:05 HUDSON_URL=http://192.168.0.87:32090/
16:12:05 GIT_COMMIT=4ac040057a26c82a9c139afde692522b8bf8553c
16:12:05 SHLVL=1
16:12:05 HOME=/root
16:12:05 BUILD_USER_ID=weizhixiu
16:12:05 BUILD_URL=http://192.168.0.87:32090/job/test/2/
16:12:05 TOMCAT_HOME=/apps/tomcat
16:12:05 HUDSON_COOKIE=f9d84c3f-7edd-4f4d-9a0b-f883b7e80b45
16:12:05 JENKINS_SERVER_COOKIE=e6d191a2e198272f
16:12:05 APP_NAME=auth-server
16:12:05 WORKSPACE=/opt/data/workspace/test
16:12:05 JENKINS_SERVICE_PORT_9090_9090=9090
16:12:05 NODE_NAME=master
16:12:05 RUN_ARTIFACTS_DISPLAY_URL=http://192.168.0.87:32090/job/test/2/display/redirect?page=artifacts
16:12:05 GIT_BRANCH=origin/master
16:12:05 EXECUTOR_NUMBER=0
16:12:05 BUILD_USER_FIRST_NAME=微知秀
16:12:05 RUN_TESTS_DISPLAY_URL=http://192.168.0.87:32090/job/test/2/display/redirect?page=tests
16:12:05 BUILD_DISPLAY_NAME=#2
16:12:05 KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
16:12:05 HUDSON_HOME=/opt/data
16:12:05 JOB_BASE_NAME=test
16:12:05 PATH=/usr/local/jdk/bin:/usr/local/jdk/jre/bin:/apps/tomcat/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
16:12:05 JENKINS_SERVICE_HOST=10.100.145.249
16:12:05 BUILD_ID=2
16:12:05 KUBERNETES_PORT_443_TCP_PORT=443
16:12:05 JENKINS_PORT_9090_TCP_ADDR=10.100.145.249
16:12:05 BUILD_TAG=jenkins-test-2
16:12:05 KUBERNETES_PORT_443_TCP_PROTO=tcp
16:12:05 JENKINS_URL=http://192.168.0.87:32090/
16:12:05 LANG=zh_CN.UTF-8
16:12:05 JOB_URL=http://192.168.0.87:32090/job/test/
16:12:05 GIT_URL=git@gitlab.youwoyouqu.io:weizhixiu/weizhixiu2.0.git
16:12:05 JENKINS_PORT_9090_TCP_PORT=9090
16:12:05 BUILD_NUMBER=2
16:12:05 dev_operation=deploy
16:12:05 JENKINS_PORT_9090_TCP_PROTO=tcp
16:12:05 RUN_DISPLAY_URL=http://192.168.0.87:32090/job/test/2/display/redirect
16:12:05 JENKINS_PORT=tcp://10.100.145.249:9090
16:12:05 JENKINS_SERVICE_PORT=9090
16:12:05 HUDSON_SERVER_COOKIE=e6d191a2e198272f
16:12:05 JOB_DISPLAY_URL=http://192.168.0.87:32090/job/test/display/redirect
16:12:05 KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
16:12:05 BUILD_USER_EMAIL=1062670898@qq.com
16:12:05 KUBERNETES_SERVICE_PORT_HTTPS=443
16:12:05 JOB_NAME=test
16:12:05 JAVA_HOME=/usr/local/jdk
16:12:05 KUBERNETES_SERVICE_HOST=10.96.0.1
16:12:05 PWD=/opt/data/workspace/test
16:12:05 JENKINS_PORT_9090_TCP=tcp://10.100.145.249:9090
16:12:05 GIT_PREVIOUS_COMMIT=4ac040057a26c82a9c139afde692522b8bf8553c
16:12:05 WORKSPACE_TMP=/opt/data/workspace/test@tmp
```

> BUILD_USER  --> username
>
> dev_operation -> dev_operation
>
> JOB_NAME -> job
>
> APP_NAME -> service

设计表

| id   | username | dev_operation | service     | job          | create_time |
| ---- | -------- | ------------- | ----------- | ------------ | ----------- |
| 1    | 宋亮成   | deploy        | auth-server | weizhixiu2.0 | 时间        |

```sql
CREATE TABLE jenkins_table (
	id INTEGER NOT NULL COMMENT '自增ID' AUTO_INCREMENT, 
	username VARCHAR(64) NOT NULL COMMENT '用户名', 
	dev_operation VARCHAR(64) COMMENT '发布或回滚', 
	service VARCHAR(64) COMMENT '发布什么服务', 
	job VARCHAR(64) NOT NULL COMMENT '什么job名', 
	create_time DATETIME NOT NULL COMMENT '什么时间发布', 
	PRIMARY KEY (id)
)
```

先准备一个SQL语句, 在k8s的pod中执行成功

```bash
mysql -h192.168.0.171 -ujenkins -pjenkins -Djenkins -e "INSERT INTO jenkins_table (username, dev_operation, service, job, create_time) VALUES ('宋亮成','deploy','auth-server','weizhixiu2.0','2020-04-03 16:24:00')"
```

![image-20210304163839762](http://myapp.img.mykernel.cn/image-20210304163839762.png)

配置一个test项目

![image-20210304163918417](http://myapp.img.mykernel.cn/image-20210304163918417.png)

在pod中准备这个脚本

```bash
mysql -h192.168.0.171 -ujenkins -pjenkins -Djenkins -e "INSERT INTO jenkins_table (username, dev_operation, service, job, create_time) VALUES ('${BUILD_USER}','${dev_operation}','${JOB_NAME}','${APP_NAME}','$(date +'%F %T')')"
```

![image-20210304163948816](http://myapp.img.mykernel.cn/image-20210304163948816.png)

现在jenkins构建运行

![image-20210304164436064](http://myapp.img.mykernel.cn/image-20210304164436064.png)

成功，所以仅需要将这段语句粘贴到相应脚本中

### nginx 完整脚本

```diff
root@master01:/data/weizhixiu/dockerfile/jenkins-dockerfile/scripts# cat admin-web-new.sh
#!/bin/sh
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-03-02
#FileName：             admin-web-new.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************

# 确保任何一个命令的错误均退出，jenkins界面才显示红色
set -e

echo 先将镜像服务器网页文件清理
ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd /data/weizhixiu/stage-builder/admin-web-nginx && rm -fr admin-web-new"
sleep 2

echo 上传网页
cd ..
rm -fr admin-web-new/node_modules/ admin-web-new/dist.zip
zip -r admin-web-new.zip admin-web-new
scp -o StrictHostKeyChecking=no admin-web-new.zip make-registry.youwoyouqu.io:/data/weizhixiu/stage-builder/admin-web-nginx
ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd /data/weizhixiu/stage-builder/admin-web-nginx && unzip admin-web-new.zip && rm -fr admin-web-new.zip"
sleep 2

echo 构建镜像
ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd /data/weizhixiu/stage-builder/admin-web-nginx && bash build.sh $(date +%Y%m%d%H%M%S)"
sleep 2

+echo 落库
+mysql -h192.168.0.171 -ujenkins -pjenkins -Djenkins -e "INSERT INTO jenkins_table (username, dev_operation, service, job, create_time) VALUES ('${BUILD_USER}','${dev_operation}','${JOB_NAME}','${APP_NAME}','$(date +'%F %T')')"
```

### java完整脚本

```diff
root@master01:/data/weizhixiu/dockerfile/jenkins-dockerfile/scripts# cat weizhixiu2.0.sh
#!/bin/sh
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-03-02
#FileName：             ${CODE_DIR}.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************
# 确保任何一个命令的错误均退出，jenkins界面才显示红色
set -e

SERVICE_DIR=/data/weizhixiu/stage-builder/weizhixiu2.0
CODE_DIR=weizhixiu2.0
APP_NAME=$1
dev_operation=$2


if [ "$dev_operation" == "deploy" ]; then
        echo  先将镜像服务器网页文件清理
        ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd ${SERVICE_DIR} && rm -fr ${CODE_DIR}"
        sleep 2

        echo  上传网页
        cd ..
        zip -r ${CODE_DIR}.zip ${CODE_DIR}
        scp -o StrictHostKeyChecking=no ${CODE_DIR}.zip make-registry.youwoyouqu.io:${SERVICE_DIR}
        ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd ${SERVICE_DIR} && unzip ${CODE_DIR}.zip && rm -fr ${CODE_DIR}.zip"
        sleep 2

        echo 构建镜像
        ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd ${SERVICE_DIR} && bash build.sh $(date +%Y%m%d%H%M%S) ${APP_NAME}"
        sleep 2
else
        echo 回滚
        ssh -o StrictHostKeyChecking=no make-registry.youwoyouqu.io "cd ${SERVICE_DIR} && bash rollout.sh ${APP_NAME}"
        sleep 2
fi


+echo 落库
+mysql -h192.168.0.171 -ujenkins -pjenkins -Djenkins -e "INSERT INTO jenkins_table (username, dev_operation, service, job, create_time) VALUES ('${BUILD_USER}','${dev_operation}','${JOB_NAME}','${APP_NAME}','$(date +'%F %T')')"
```

配置jenkins添加mysql程序

```diff
root@node06:/data/weizhixiu/dockerfile/jenkins-dockerfile# cat Dockerfile
FROM harbor.youwoyouqu.io/baseimage/jdk:8

ADD kubectl /usr/bin/
ADD config  /root/.kube/

ADD jenkins.war .
ENV JENKINS_HOME=/opt/data


# git
RUN apk add --no-cache git openssh 

# vue项目
RUN apk add --no-cache npm maven

# 打包结果复制到镜像主机
COPY .ssh /root/.ssh/

# 准备项目脚本
# scripts/* -> /opt/scripts/
ADD scripts /opt/scripts/

+# 添加mysql
+RUN apk add --no-cache mysql mysql-client

CMD ["java", "-jar", "/jenkins.war", "--httpPort=9090","-server -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=5"]
```

配置所有项目启动

![image-20210304164815839](http://myapp.img.mykernel.cn/image-20210304164815839.png)

发版jenkins

```bash
root@node06:/data/weizhixiu/dockerfile/jenkins-dockerfile# bash build.sh 20210309
```



## jenkins授权项目给开发

### 创建用户  --- 后期维护：1

![image-20210302175216471](http://myapp.img.mykernel.cn/image-20210302175216471.png)

![image-20210302175227337](http://myapp.img.mykernel.cn/image-20210302175227337.png)

### 配置权限

![image-20210303092149970](http://myapp.img.mykernel.cn/image-20210303092149970.png)

![image-20210303092232467](http://myapp.img.mykernel.cn/image-20210303092232467.png)

### 管理角色  --- 后期维护：2

![image-20210303092252245](http://myapp.img.mykernel.cn/image-20210303092252245.png)

![image-20210303092650162](http://myapp.img.mykernel.cn/image-20210303092650162.png)

开发人员：

![image-20210615135532080](http://myapp.img.mykernel.cn/image-20210615135532080.png)

测试人员：看所有job，只读或有构建操作

![image-20210615135749495](http://myapp.img.mykernel.cn/image-20210615135749495.png)

### 用户分配角色 -- 用户授权  --- 后期维护：3

![image-20210303092751866](http://myapp.img.mykernel.cn/image-20210303092751866.png)

### 测试登陆

![image-20210303092942995](http://myapp.img.mykernel.cn/image-20210303092942995.png)

接下来指导开发使用吧

注意：管理员的视图如下

![image-20210303093018353](http://myapp.img.mykernel.cn/image-20210303093018353.png)



# 部署日志收集平台ELK

现在CICD跑通了，但是前端写的代码访问后台接口可能慢，需要我们去分析哪些慢，为什么慢，所以需要收集nginx来分析

分析过程：http://blog.mykernel.cn/2021/03/04/diary-kubernetes%E4%B8%ADapi%E6%8E%A5%E5%8F%A3%E6%85%A2%E6%8E%92%E6%9F%A5/

这个还是前端反馈说慢，但是测试人员未必每个接口都能处理到，所以现在就直接使用kibana分析

分析图形参考：http://blog.mykernel.cn/2020/10/30/kibana%E6%B7%BB%E5%8A%A0%E5%B8%B8%E7%94%A8%E5%9B%BE%E5%BD%A2/



## 部署es

es分为data, master, ingest节点，我们只需要使用默认配置，集群内部实现这些逻辑

由于本地没有足够的主机，所以使用单机的es

### 单机es 7.11.1

https://www.elastic.co/cn/downloads/past-releases#elasticsearch

直接选择最新的  [RPM X86_64](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.11.1-x86_64.rpm)[sha](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.11.1-x86_64.rpm.sha512)

```bash
[root@elk ~]# rpm -ivh elasticsearch-7.11.1-x86_64.rpm 
```

```bash
[root@elk ~]# install -dv /data/elasticsearch/
[root@elk ~]# mount /dev/sda5 /data/elasticsearch/
[root@elk ~]# ls /data/elasticsearch/
lost+found

```

```diff
[root@elk ~]# cat /etc/fstab

#
# /etc/fstab
# Created by anaconda on Thu Jun 18 21:44:15 2020
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=12d1c266-1a85-46dd-8a9e-165bc5a8b9cc /boot                   xfs     defaults        0 0
/dev/mapper/centos-home /home                   xfs     defaults        0 0
+/dev/sda5               /data/elasticsearch/    ext4    defaults        0 0
```

配置

```bash
cluster.name: elk-cluster
node.name: 192.168.0.171 # node.name必须存在在  discovery.seed_hosts cluster.initial_master_nodes 中，主机名就也写主机名
path.data: /data/elasticsearch/es_data
path.logs: /data/elasticsearch/es_logs  
bootstrap.memory_lock: true
network.host: 0.0.0.0
http.port: 9200
discovery.seed_hosts: ["192.168.0.171"]
cluster.initial_master_nodes: ["192.168.0.171"]
gateway.recover_after_nodes: 1
action.destructive_requires_name: true
http.cors.enabled: true #开启支持跨域访问
http.cors.allow-origin: "*" #指定允许访问范围
```

```bash
[root@elk ~]# install -dv -o elasticsearch -g elasticsearch /data/elasticsearch/es_data /data/elasticsearch/es_logs
```

启动

```bash
[root@elk ~]# systemctl restart elasticsearch
```

测试访问9200

```bash
[root@elk ~]# curl localhost:9200
{
  "name" : "192.168.0.171",
  "cluster_name" : "elk-cluster",
  "cluster_uuid" : "1AxrlLZNQmml8YNf4zKE1Q",
  "version" : {
    "number" : "7.11.1",
    "build_flavor" : "default",
    "build_type" : "rpm",
    "build_hash" : "ff17057114c2199c9c1bbecc727003a907c0db7a",
    "build_date" : "2021-02-15T13:44:09.394032Z",
    "build_snapshot" : false,
    "lucene_version" : "8.7.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

查看Elasticsearch健康状态 (*表示ES集群的master主节点)

```bash
[root@elk ~]# curl -XGET 'http://192.168.0.171:9200/_cat/nodes?v'
ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role  master name
10.244.6.0           21          59   3    0.47    0.46     0.35 cdhilmrstw *      192.168.0.171



[root@elk ~]# curl -XGET 'http://192.168.0.171:9200/_cat/health'
1614942494 11:08:14 elk-cluster green 1 1 6 6 0 0 0 0 - 100.0%


[root@elk ~]# curl -XGET 'http://192.168.0.171:9200/_cluster/health?pretty'
{
  "cluster_name" : "elk-cluster",
  "status" : "green",       #为 green 则代表健康没问题，如果是 yellow 或者 red 则是集群有问题
  "timed_out" : false,        #是否有超时
  "number_of_nodes" : 1,         #集群中的节点数量
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 6,
  "active_shards" : 6,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0 #集群分片的可用性百分比，如果为0则表示不可用
}

```

> https://www.cnblogs.com/kevingrace/p/10671063.html

#### 部署ElasticSearch Web管理工具

https://github.com/lmenezes/cerebro

```bash
# releases处下载
[root@elk ~]# rpm -ivh cerebro-0.9.3-1.noarch.rpm 
准备中...                          ################################# [100%]
Creating system group: cerebro
Creating system user: cerebro in cerebro with cerebro user-daemon and shell /bin/false
正在升级/安装...
   1:cerebro-0.9.3-1                  ################################# [100%]
Created symlink from /etc/systemd/system/multi-user.target.wants/cerebro.service to /usr/lib/systemd/system/cerebro.service.


[root@elk ~]# java -Duser.dir=/usr/share/cerebro  -jar /usr/share/cerebro/lib/cerebro.cerebro-0.9.3-launcher.jar
[info] p.c.s.AkkaHttpServer - Listening for HTTP on /0:0:0:0:0:0:0:0:9000
```

首页填 http://192.168.0.171:9200

![image-20210305191927931](http://myapp.img.mykernel.cn/image-20210305191927931.png)





### 集群es

直接拿下生成的es的yaml

| ip           | 角色 |
| ------------ | ---- |
| 172.16.0.222 | es1  |
| 172.16.0.223 | es2  |
| 172.16.0.224 | es3  |

```diff
# 222
[root@k8s-master1 ~]# cat /etc/elasticsearch/elasticsearch.yml 
cluster.name: weizhixiu
+node.name: node-1
path.data: /mnt/elasticsearch/data
path.logs: /mnt/elasticsearch/log
bootstrap.memory_lock: true
network.host: 0.0.0.0
http.port: 9200
discovery.seed_hosts: ["172.16.0.222", "172.16.0.223","172.16.0.224"]
cluster.initial_master_nodes: ["172.16.0.222", "172.16.0.223","172.16.0.224"]
gateway.recover_after_nodes: 2
action.destructive_requires_name: true
http.cors.enabled: true
http.cors.allow-origin: "*"
```

```diff
# 223
[root@k8s-master2 ~]# cat /etc/elasticsearch/elasticsearch.yml 
cluster.name: weizhixiu
+node.name: node-2
path.data: /mnt/elasticsearch/data
path.logs: /mnt/elasticsearch/log
bootstrap.memory_lock: true
network.host: 0.0.0.0
http.port: 9200
discovery.seed_hosts: ["172.16.0.222", "172.16.0.223","172.16.0.224"]
cluster.initial_master_nodes: ["172.16.0.222", "172.16.0.223","172.16.0.224"]
gateway.recover_after_nodes: 2
action.destructive_requires_name: true
http.cors.enabled: true #开启支持跨域访问
http.cors.allow-origin: "*" #指定允许访问范围
```

```diff
[root@k8s-master3 ~]# cat /etc/elasticsearch/elasticsearch.yml 
cluster.name: weizhixiu
+node.name: node-3
path.data: /mnt/elasticsearch/data
path.logs: /mnt/elasticsearch/logs
bootstrap.memory_lock: true
network.host: 0.0.0.0
http.port: 9200
discovery.seed_hosts: ["172.16.0.222", "172.16.0.223","172.16.0.224"]
cluster.initial_master_nodes: ["172.16.0.222", "172.16.0.223","172.16.0.224"]
gateway.recover_after_nodes: 2
action.destructive_requires_name: true
http.cors.enabled: true #开启支持跨域访问
http.cors.allow-origin: "*" #指定允许访问范围
```



## 部署logstash

### 单机logstash

#### 部署jdk

```bash
#!/bin/bash
#此版本适合Ubuntu 18.04.4 LTS，其他版本未测试



#下载
cd /usr/local/src

[[ ! -f jdk-8u221-linux-x64.tar.gz ]] && wget http://qiniu.mykernel.cn/jdk-8u221-linux-x64.tar.gz

cd /usr/local/src && rm -rf  jdk1.8.0_221

#解压
tar xvf jdk-8u221-linux-x64.tar.gz

#软连接
ln -sv /usr/local/src/jdk1.8.0_221 /usr/local/jdk

#编辑配置文件
echo "
export JAVA_HOME=/usr/local/jdk
export TOMCAT_HOME=/apps/tomcat
export PATH=\$JAVA_HOME/bin:\$JAVA_HOME/jre/bin:\$TOMCAT_HOME/bin:\$PATH
export CLASSPATH=.\$CLASSPATH:\$JAVA_HOME/lib:\$JAVA_HOME/jre/lib:\$TOMCAT_HOME/lib/tools.jar">>/etc/profile

#读取配置文件
source /etc/profile

#版本检测
java -version
```

#### 部署logstash 7.11.1

- [RPM X86_64](https://artifacts.elastic.co/downloads/logstash/logstash-7.11.1-x86_64.rpm)[sha](https://artifacts.elastic.co/downloads/logstash/logstash-7.11.1-x86_64.rpm.sha512)

```bash
[root@elk ~]# rpm -ivh logstash-7.11.1-x86_64.rpm 
```

#### 写入数据

```bash
/usr/share/logstash/bin/logstash -e 'input { stdin{} } output { elasticsearch { hosts => [" 192.168.0.171:9200"] index => "mytest-%{+YYYY.MM.dd}" } }'
```

验证数据

![image-20210305195229441](http://myapp.img.mykernel.cn/image-20210305195229441.png)

```bash
[root@elk ~]# ls /data/elasticsearch/es_data/nodes/0/indices/
-2rQPgsURfOUb92NM3GjbQ  imBHodTsSp6d7T05Mj3aLQ  LzxADPztTleC4eu3fHimeg  w2-0ff4hS4aCZrSix3185w  z1Ug-8v0TgueFWWDKzaJ2A
FVy58KrbQrWLRxoHr-RL3A  lA6f2U9IQ0KSK_V_BQfbxQ  -MLJP-UbSqSEB1ekIu8TLg  x6bWEDYhTg2WchNIrbzCBQ
```

### 集群 logstash

和logstash相同配置即可



## 部署kibana 7.11.1

https://www.elastic.co/cn/downloads/past-releases#kibana

- [RPM 64-BIT](https://artifacts.elastic.co/downloads/kibana/kibana-7.11.1-x86_64.rpm)[sha](https://artifacts.elastic.co/downloads/kibana/kibana-7.11.1-x86_64.rpm.sha512)

### 单机kibana

```bash
[root@elk ~]# rpm -ivh kibana-7.11.1-x86_64.rpm 
```

配置

```bash
[root@elk ~]# cat /etc/kibana/kibana.yml
server.port: 5601
server.host: "192.168.0.171"
elasticsearch.hosts: ["http://192.168.0.171:9200"]
#elasticsearch.username: ""
#elasticsearch.password: ""
i18n.locale: "zh-CN"
```

查看状态

http://192.168.0.171:5601/status

访问kibana

![image-20210305192200836](http://myapp.img.mykernel.cn/image-20210305192200836.png)

添加上一步的数据

![image-20210305195636374](http://myapp.img.mykernel.cn/image-20210305195636374.png)

![image-20210305195658982](http://myapp.img.mykernel.cn/image-20210305195658982.png)

查看数据

![image-20210305195917607](http://myapp.img.mykernel.cn/image-20210305195917607.png)

### 集群kibana

由于是http请求，就和nginx一样启动多个即可



## 规划生产日志收集

所有容器应用走filebeat写中间redis，logstash从redis读，然后写

需要收集haproxy入口的日志

filebeats -> 192.168.0.171(logstash 解析) -> 192.168.0.79(redis) -> 192.168.0.85(logstash 传输) -> 192.168.0.171(es存储)

### 准备redis

略

参考：http://liangcheng.mykernel.cn/2020/11/03/redis%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AE%E5%8F%8A%E4%BC%98%E5%8C%96/

### 配置nginx Docker支持日志收集

#### 配置filebeat->logstash->redis

修改nginx基础镜像即可

```bash
# 日志到控制台
/usr/local/filebeat-7.11.1-linux-x86_64/filebeat  -e --path.config /usr/local/filebeat-7.11.1-linux-x86_64/ -c  /usr/local/filebeat-7.11.1-linux-x86_64/filebeat.yml --path.data /usr/local/filebeat-7.11.1-linux-x86_64/data/ --path.home /usr/local/filebeat-7.11.1-linux-x86_64/ --path.logs /usr/local/filebeat-7.11.1-linux-x86_64/logs

# 日志到文件
/usr/local/filebeat-7.11.1-linux-x86_64/filebeat   --path.config /usr/local/filebeat-7.11.1-linux-x86_64/ -c  /usr/local/filebeat-7.11.1-linux-x86_64/filebeat.yml --path.data /usr/local/filebeat-7.11.1-linux-x86_64/data/ --path.home /usr/local/filebeat-7.11.1-linux-x86_64/ --path.logs /usr/local/filebeat-7.11.1-linux-x86_64/logs

```

```bash
root@master01:/data/weizhixiu/dockerfile/nginx-baseimage-dockerfile# cat Dockerfile 
FROM harbor.youwoyouqu.io/baseimage/alpine:base

LABEL AUTHOR=songliangcheng QQ=1062670898

RUN apk add --no-cache --virtual .build-deps \
  gcc \
  libc-dev \
  make \
  openssl-dev \
  pcre-dev \
  zlib-dev \
  linux-headers \
  curl \
  gnupg \
  libxslt-dev \
  gd-dev \
  geoip-dev \
  perl-dev 
  
ADD nginx-1.16.1.tar.gz ./
RUN cd nginx-1.16.1 && \
 ./configure --prefix=/apps/nginx \
 --user=root \
 --group=root \
 --pid-path=/run/nginx.pid \
 --with-threads \
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
 --with-pcre && \
    make -j `grep -c processor /proc/cpuinfo` && make install 

RUN install -dv /apps/nginx/conf/conf.d/


# 配置json日志
ADD nginx.conf /apps/nginx/conf/nginx.conf

# 添加filebeat
ADD filebeat-7.11.1-linux-x86_64.tar.gz /usr/local/

# 清理
RUN rm -fr /nginx-1.16.1

#CMD ["/apps/nginx/sbin/nginx", "-g", "daemon off;"]
ADD run.sh /usr/local/bin/
CMD ["run.sh"]
```

配置启动脚本

```bash
root@master01:/data/weizhixiu/dockerfile/nginx-baseimage-dockerfile# cat run.sh
#!/bin/sh
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-03-08
#FileName：             run.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************


nohup /usr/local/filebeat-7.11.1-linux-x86_64/filebeat   --path.config /usr/local/filebeat-7.11.1-linux-x86_64/ -c  /usr/local/filebeat-7.11.1-linux-x86_64/filebeat.yml --path.data /usr/local/filebeat-7.11.1-linux-x86_64/data/ --path.home /usr/local/filebeat-7.11.1-linux-x86_64/ --path.logs /usr/local/filebeat-7.11.1-linux-x86_64/logs &
/apps/nginx/sbin/nginx
tail -f /etc/hosts
```

构建

```bash
root@master01:/data/weizhixiu/dockerfile/nginx-baseimage-dockerfile# bash build.sh 
```

准备haproxy

```bash
listen logstash-5045
	bind 192.168.0.246:5045
	mode tcp
	server 192.168.0.171 192.168.0.171:5045 check inter 3s fall 2 rise 5                                        
root@haproxy01:~# systemctl restart haproxy
```

启动logstash

```bash
[root@elk ~]# cat /etc/logstash/conf.d/beats-nginx.conf
input {
  beats {
	port => 5045
	codec => "json"  # 将json的每个元互相转换为一个事件
  }
}

output {
	if [fields][type] == "nginx-accesslog-filebeat" {
	    redis {
			data_type => "list"
			key => "nginxlog-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "12"
			password => "linux48"       
		}  
		stdout{ codec => rubydebug }
	}
	if [fields][type] == "nginx-errorlog-filebeat" {
	    redis {
			data_type => "list"
			key => "nginxlog-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "12"
			password => "linux48"
		}  
		stdout{ codec => rubydebug }
	}
}


[root@elk ~]# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/beats-nginx.conf 
```

验证logstash已经正常

![image-20210309142515321](http://myapp.img.mykernel.cn/image-20210309142515321.png)

测试运行

```bash
root@master01:/data/weizhixiu/dockerfile/nginx-baseimage-dockerfile# docker run -it -p 30876:80 --rm harbor.youwoyouqu.io/baseimage/nginx:base sh
/ # cat /usr/local/bin/run.sh # 查看运行内容
nohup /usr/local/filebeat-7.11.1-linux-x86_64/filebeat   --path.config /usr/local/filebeat-7.11.1-linux-x86_64/ -c  /usr/local/filebeat-7.11.1-linux-x86_64/filebeat.yml --path.data /usr/local/filebeat-7.11.1-linux-x86_64/data/ --path.home /usr/local/filebeat-7.11.1-linux-x86_64/ --path.logs /usr/local/filebeat-7.11.1-linux-x86_64/logs &
/apps/nginx/sbin/nginx
tail -f /etc/hosts

/ # /apps/nginx/sbin/nginx # 启动nginx
```

连接redis, 清理key

```bash
root@master03:~# redis-cli  -h 127.0.0.1
127.0.0.1:6379> auth linux48
OK
127.0.0.1:6379> SELECT 13
OK
127.0.0.1:6379[13]> DBSIZE
(integer) 0
127.0.0.1:6379[13]> FLUSHDB
OK
127.0.0.1:6379[13]> KEYS *
(empty list or set)
127.0.0.1:6379[13]> 
```

```diff
/ # cat /usr/local/filebeat-7.11.1-linux-x86_64/filebeat.yml 
filebeat.inputs:
+- type: log
  enabled: true
  paths:
+    - /apps/nginx/logs/access.log
  exclude_lines: ['^DBG']
  exclude_files: ['.gz$']
  fields:
    type: nginx-accesslog-filebeat
    level: debug
    review: 1
+- type: log
  enabled: true
  paths:
+    - /apps/nginx/logs/error.log
  exclude_lines: ['^DBG']
  exclude_files: ['.gz$']
  fields:
    type: nginx-errorlog-filebeat
    level: debug
    review: 1
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
setup.template.settings:
  index.number_of_shards: 1
setup.kibana:
+output.logstash:
  hosts: ["192.168.0.246:5045"]
  worker: 1
  compression_level: 3
  loadbalance: true
processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~
```

> VIP



现在收集日志

```bash
/ # /usr/local/filebeat-7.11.1-linux-x86_64/filebeat   --path.config /usr/local/filebeat-7.11.1-linux-x86_64/ -c  /usr/local/filebeat-7.11.1-linux-x86_64/filebeat.yml --path.data /usr/local/filebeat-7.11.1-linux-x86_64/data/ --path.home /usr/local/filebeat-7.11.1-linux-x86_64/ --path.logs /usr/local/filebeat-7.11.1-linux-x86_64/logs -e # 打印日志

# 另打开一个终端请求nginx
root@master01:/data/weizhixiu/dockerfile/redmine-dockerfile# curl localhost:30876/1111111111
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>



# 观察日志输出logstash已经建立
2021-03-09T14:31:59.498+0800	INFO	log/harvester.go:302	Harvester started for file: /apps/nginx/logs/access.log
2021-03-09T14:32:00.498+0800	INFO	[publisher_pipeline_output]	pipeline/output.go:143	Connecting to backoff(async(tcp://192.168.0.246:5045))
2021-03-09T14:32:00.498+0800	INFO	[publisher]	pipeline/retry.go:219	retryer: send unwait signal to consumer
2021-03-09T14:32:00.499+0800	INFO	[publisher]	pipeline/retry.go:223	  done
2021-03-09T14:32:00.518+0800	INFO	[publisher_pipeline_output]	pipeline/output.go:151	Connection to backoff(async(tcp://192.168.0.246:5045)) established


# logstash查看日志,2个 错误日志会显示json解析失败。正确日志，正常解析。
[ERROR] 2021-03-09 14:43:14.544 [defaultEventExecutorGroup-4-1] json - JSON parse error, original data now in message field {:error=>#<LogStash::Json::ParserError: Unexpected character ('/' (code 47)): Expected space separating root-level values
 at [Source: (String)"2021/03/09 14:43:02 [error] 9#9: *3 open() "/apps/nginx/html/1111111111" failed (2: No such file or directory), client: 172.17.0.1, server: localhost, request: "GET /1111111111 HTTP/1.1", host: "localhost:30876""; line: 1, column: 6]>, :data=>"2021/03/09 14:43:02 [error] 9#9: *3 open() \"/apps/nginx/html/1111111111\" failed (2: No such file or directory), client: 172.17.0.1, server: localhost, request: \"GET /1111111111 HTTP/1.1\", host: \"localhost:30876\""}
{

       "message" => "2021/03/09 14:43:02 [error] 9#9: *3 open() \"/apps/nginx/html/1111111111\" failed (2: No such file or directory), client: 172.17.0.1, server: localhost, request: \"GET /1111111111 HTTP/1.1\", host: \"localhost:30876\""
}
{
 
    "upstreamtime" => "",
    "responsetime" => 0.0,
    "upstreamhost" => "",
}




# redis查看日志
127.0.0.1:6379[12]> LLEN nginxlog-filebeat
(integer) 2
127.0.0.1:6379[12]> LPOP nginxlog-filebeat
"{\"@timestamp\":\"2021-03-09T06:46:17.094Z\",\"size\":146,\"@version\":\"1\",\"xff\":\"\",\"agent\":{\"name\":\"e28979c38859\",\"ephemeral_id\":\"3b01776f-3ae2-4272-9cfc-b0ad9fe3db4b\",\"id\":\"faefd026-4b8c-4816-92ce-72626674bce5\",\"type\":\"filebeat\",\"hostname\":\"e28979c38859\",\"version\":\"7.11.1\"},\"host\":{\"name\":\"e28979c38859\",\"containerized\":true,\"ip\":[\"172.17.0.3\"],\"mac\":[\"02:42:ac:11:00:03\"],\"hostname\":\"e28979c38859\",\"os\":{\"name\":\"Alpine Linux\",\"platform\":\"alpine\",\"version\":\"\",\"kernel\":\"4.15.0-29-generic\",\"family\":\"\"},\"architecture\":\"x86_64\"},\"upstreamtime\":\"\",\"responsetime\":0.0,\"upstreamhost\":\"\",\"user-agent\":\"curl/7.58.0\",\"domain\":\"localhost\",\"fields\":{\"level\":\"debug\",\"type\":\"nginx-accesslog-filebeat\",\"review\":1},\"clientip\":\"172.17.0.1\",\"ecs\":{\"version\":\"1.6.0\"},\"http_host\":\"localhost\",\"input\":{\"type\":\"log\"},\"status\":\"404\",\"tags\":[\"beats_input_codec_json_applied\"],\"tcp-xff\":\"\",\"log\":{\"file\":{\"path\":\"/apps/nginx/logs/access.log\"},\"offset\":891},\"referer\":\"\",\"url\":\"/1111111111\"}"
127.0.0.1:6379[12]> LPOP nginxlog-filebeat
"{\"@timestamp\":\"2021-03-09T06:46:22.093Z\",\"log\":{\"offset\":424,\"file\":{\"path\":\"/apps/nginx/logs/error.log\"}},\"@version\":\"1\",\"input\":{\"type\":\"log\"},\"agent\":{\"name\":\"e28979c38859\",\"ephemeral_id\":\"3b01776f-3ae2-4272-9cfc-b0ad9fe3db4b\",\"id\":\"faefd026-4b8c-4816-92ce-72626674bce5\",\"type\":\"filebeat\",\"hostname\":\"e28979c38859\",\"version\":\"7.11.1\"},\"tags\":[\"_jsonparsefailure\",\"beats_input_codec_json_applied\"],\"fields\":{\"level\":\"debug\",\"type\":\"nginx-errorlog-filebeat\",\"review\":1},\"ecs\":{\"version\":\"1.6.0\"},\"host\":{\"name\":\"e28979c38859\",\"containerized\":true,\"ip\":[\"172.17.0.3\"],\"mac\":[\"02:42:ac:11:00:03\"],\"os\":{\"name\":\"Alpine Linux\",\"platform\":\"alpine\",\"version\":\"\",\"family\":\"\",\"kernel\":\"4.15.0-29-generic\"},\"hostname\":\"e28979c38859\",\"architecture\":\"x86_64\"},\"message\":\"2021/03/09 14:46:15 [error] 9#9: *4 open() \\\"/apps/nginx/html/1111111111\\\" failed (2: No such file or directory), client: 172.17.0.1, server: localhost, request: \\\"GET /1111111111 HTTP/1.1\\\", host: \\\"localhost:30876\\\"\"}"
127.0.0.1:6379[12]> LPOP nginxlog-filebeat
(nil)

# 可以看到一条是来自错误日志,一条来自正常
```

#### redis -> logstash -> es 

再配置logstash从redis读数据，写入es

```bash
root@master05:~# cat /etc/logstash/conf.d/nginx.conf
input {
	    redis {
			data_type => "list"
			key => "nginxlog-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "12"
			password => "linux48"
			codec => json                   # 将json的每个元互相转换为一个事件
		}  
}

output {
	if [fields][type] == "nginx-accesslog-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "logstash-nginx-accesslog-%{+YYYY.MM.dd}"
		}
		stdout{ codec => rubydebug }
	}
	if [fields][type] == "nginx-errorlog-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "logstash-nginx-errorlog-%{+YYYY.MM.dd}"
		}
		stdout{ codec => rubydebug }
	}
}

```

```bash
[root@elk ~]# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/nginx.conf  


# 现在再次测试请求
root@master01:/data/weizhixiu/dockerfile/redmine-dockerfile# curl localhost:30876/1111111111
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>


# 此处发现2条日志, 并且也不会报错
{
    "@timestamp" => 2021-03-09T07:26:32.149Z,
           "ecs" => {
        "version" => "1.6.0"
    },
           "log" => {
        "offset" => 1696,
          "file" => {
            "path" => "/apps/nginx/logs/error.log"
        }
    },
          "host" => {
        "containerized" => true,
             "hostname" => "e28979c38859",
                  "mac" => [
            [0] "02:42:ac:11:00:03"
        ],
                 "name" => "e28979c38859",
                   "os" => {
              "family" => "",
              "kernel" => "4.15.0-29-generic",
             "version" => "",
                "name" => "Alpine Linux",
            "platform" => "alpine"
        },
         "architecture" => "x86_64",
                   "ip" => [
            [0] "172.17.0.3"
        ]
    },
         "agent" => {
            "hostname" => "e28979c38859",
                "type" => "filebeat",
                "name" => "e28979c38859",
        "ephemeral_id" => "3b01776f-3ae2-4272-9cfc-b0ad9fe3db4b",
             "version" => "7.11.1",
                  "id" => "faefd026-4b8c-4816-92ce-72626674bce5"
    },
          "tags" => [
        [0] "_jsonparsefailure",
        [1] "beats_input_codec_json_applied"
    ],
        "fields" => {
        "review" => 1,
         "level" => "debug",
          "type" => "nginx-errorlog-filebeat"
    },
      "@version" => "1",
       "message" => "2021/03/09 15:26:22 [error] 9#9: *10 open() \"/apps/nginx/html/1111111111\" failed (2: No such file or directory), client: 172.17.0.1, server: localhost, request: \"GET /1111111111 HTTP/1.1\", host: \"localhost:30876\"",
         "input" => {
        "type" => "log"
    }
}
{
             "xff" => "",
            "host" => {
        "containerized" => true,
             "hostname" => "e28979c38859",
                  "mac" => [
            [0] "02:42:ac:11:00:03"
        ],
                 "name" => "e28979c38859",
                   "os" => {
              "family" => "",
              "kernel" => "4.15.0-29-generic",
             "version" => "",
                "name" => "Alpine Linux",
            "platform" => "alpine"
        },
         "architecture" => "x86_64",
                   "ip" => [
            [0] "172.17.0.3"
        ]
    },
      "user-agent" => "curl/7.58.0",
       "http_host" => "localhost",
         "tcp-xff" => "",
          "status" => "404",
    "upstreamhost" => "",
           "input" => {
        "type" => "log"
    },
    "responsetime" => 0.0,
             "ecs" => {
        "version" => "1.6.0"
    },
         "referer" => "",
             "log" => {
        "offset" => 2673,
          "file" => {
            "path" => "/apps/nginx/logs/access.log"
        }
    },
        "clientip" => "172.17.0.1",
          "domain" => "localhost",
        "@version" => "1",
    "upstreamtime" => "",
             "url" => "/1111111111",
           "agent" => {
            "hostname" => "e28979c38859",
                "type" => "filebeat",
                "name" => "e28979c38859",
        "ephemeral_id" => "3b01776f-3ae2-4272-9cfc-b0ad9fe3db4b",
             "version" => "7.11.1",
                  "id" => "faefd026-4b8c-4816-92ce-72626674bce5"
    },
            "size" => 146,
          "fields" => {
        "review" => 1,
         "level" => "debug",
          "type" => "nginx-accesslog-filebeat"
    },
      "@timestamp" => 2021-03-09T07:26:32.150Z,
            "tags" => [
        [0] "beats_input_codec_json_applied"
    ]
}

```

#### kibana添加访问日志

![image-20210309145252606](http://myapp.img.mykernel.cn/image-20210309145252606.png)

![image-20210309145307588](http://myapp.img.mykernel.cn/image-20210309145307588.png)





![image-20210309152747251](http://myapp.img.mykernel.cn/image-20210309152747251.png)

#### kibana添加错误日志

![image-20210309152816291](http://myapp.img.mykernel.cn/image-20210309152816291.png)![image-20210309152822691](http://myapp.img.mykernel.cn/image-20210309152822691.png)

![image-20210309152847676](http://myapp.img.mykernel.cn/image-20210309152847676.png)

> 此处是左侧按类型筛选下的message点了加号

#### 准备filebeat.yaml至Dockerfile

```bash
root@master01:/data/weizhixiu/dockerfile/nginx-baseimage-dockerfile# cat filebeat.yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /apps/nginx/logs/access.log
  exclude_lines: ['^DBG']
  exclude_files: ['.gz$']
  fields:
    type: nginx-accesslog-filebeat
    level: debug
    review: 1
- type: log
  enabled: true
  paths:
    - /apps/nginx/logs/error.log
  exclude_lines: ['^DBG']
  exclude_files: ['.gz$']
  fields:
    type: nginx-errorlog-filebeat
    level: debug
    review: 1
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
setup.template.settings:
  index.number_of_shards: 1
setup.kibana:
output.logstash:
  hosts: ["192.168.0.246:5045"]
  worker: 1
  compression_level: 3
  loadbalance: true
processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~
```

```diff
root@master01:/data/weizhixiu/dockerfile/nginx-baseimage-dockerfile# cat Dockerfile
FROM harbor.youwoyouqu.io/baseimage/alpine:base

LABEL AUTHOR=songliangcheng QQ=1062670898

RUN apk add --no-cache --virtual .build-deps \
  gcc \
  libc-dev \
  make \
  openssl-dev \
  pcre-dev \
  zlib-dev \
  linux-headers \
  curl \
  gnupg \
  libxslt-dev \
  gd-dev \
  geoip-dev \
  perl-dev 
  
ADD nginx-1.16.1.tar.gz ./
RUN cd nginx-1.16.1 && \
 ./configure --prefix=/apps/nginx \
 --user=root \
 --group=root \
 --pid-path=/run/nginx.pid \
 --with-threads \
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
 --with-pcre && \
    make -j `grep -c processor /proc/cpuinfo` && make install 

RUN install -dv /apps/nginx/conf/conf.d/


# 配置json日志
ADD nginx.conf /apps/nginx/conf/nginx.conf

# 添加filebeat
ADD filebeat-7.11.1-linux-x86_64.tar.gz /usr/local/
+ADD filebeat.yml /usr/local/filebeat-7.11.1-linux-x86_64/

# 清理
RUN rm -fr /nginx-1.16.1

#CMD ["/apps/nginx/sbin/nginx", "-g", "daemon off;"]
ADD run.sh /usr/local/bin/
CMD ["run.sh"]

```

构建

```bash
root@master01:/data/weizhixiu/dockerfile/nginx-baseimage-dockerfile# bash build.sh
```



#### 将nginx线上pod重新构建

jenkins pod中执行

```bash
/ # cd /opt/data/workspace/admin-web-new
/opt/data/workspace/admin-web-new # sh -x /opt/scripts/admin-web-new.sh 
```

进入nginx-controller-pod 可以观察到filebeat日志

```bash
2021-03-09T15:34:28.469+0800    INFO    [publisher_pipeline_output]     pipeline/output.go:151  Connection to backoff(async(tcp://192.168.0.246:5045)) established
```

查看nginx日志

```bash
/ #  cat /apps/nginx/logs/access.log 
{"@timestamp":"2021-03-09T15:40:11+08:00","host":"10.244.5.22","clientip":"10.244.5.1","size":612,"responsetime":0.000,"upstreamtime":"","upstreamhost":"","http_host":"10.244.5.22","url":"/index.html","domain":"10.244.5.22","xff":"","tcp-xff":"","referer":"","user-agent":"kube-probe/1.20","status":"200"}
{"@timestamp":"2021-03-09T15:40:14+08:00","host":"10.244.5.22","clientip":"10.244.5.1","size":612,"responsetime":0.000,"upstreamtime":"","upstreamhost":"","http_host":"10.244.5.22","url":"/index.html","domain":"10.244.5.22","xff":"","tcp-xff":"","referer":"","user-agent":"kube-probe/1.20","status":"200"}
{"@timestamp":"2021-03-09T15:40:14+08:00","host":"10.244.5.22","clientip":"10.244.5.1","size":612,"responsetime":0.000,"upstreamtime":"","upstreamhost":"","http_host":"10.244.5.22","url":"/index.html","domain":"10.244.5.22","xff":"","tcp-xff":"","referer":"","user-agent":"kube-probe/1.20","status":"200"}
{"@timestamp":"2021-03-09T15:40:17+08:00","host":"10.244.5.22","clientip":"10.244.5.1","size":612,"responsetime":0.000,"upstreamtime":"","upstreamhost":"","http_host":"10.244.5.22","url":"/index.html","domain":"10.244.5.22","xff":"","tcp-xff":"","referer":"","user-agent":"kube-probe/1.20","status":"200"}
```



通过解析logstash和写es的logstash可以查看到全部是与业务无关的nginx liveProbe, readinessProbe的日志，所以需要排除他们

![image-20210309153807168](http://myapp.img.mykernel.cn/image-20210309153807168.png)

redis侧可以看到，key出现了，一下又被拿走

```bash
127.0.0.1:6379[12]> KEYS *
1) "nginxlog-filebeat"
127.0.0.1:6379[12]> KEYS *
(empty list or set)
```

> 所以一旦logstash宕机，redis可能key的长度打满，redis机器内存打满，死机





#### Pod中修正filebeat 规则

由于以上的日志会记录与业务无关的nginx liveProbe, readinessProbe的日志，需要排除

为了方便直接在pod中使用，由于是2个pod, 先缩容pod为一个

```bash
root@master01:/data/weizhixiu/dockerfile/nginx-baseimage-dockerfile# kubectl scale deploy -n weizhixiu nginx-controller-deploy --replicas=1
```

进入pod

```diff
/ # cat /usr/local/filebeat-7.11.1-linux-x86_64/filebeat.yml 
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /apps/nginx/logs/access.log
  exclude_lines: ['^DBG']
+  exclude_lines: ['kube-probe/1.20']
  exclude_files: ['.gz$']
  fields:
    type: nginx-accesslog-filebeat
    level: debug
    review: 1
- type: log
  enabled: true
  paths:
    - /apps/nginx/logs/error.log
  exclude_lines: ['^DBG']
+  exclude_lines: ['kube-probe/1.20']
  exclude_files: ['.gz$']
  fields:
    type: nginx-errorlog-filebeat
    level: debug
    review: 1
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
setup.template.settings:
  index.number_of_shards: 1
setup.kibana:
output.logstash:
  hosts: ["192.168.0.246:5045"]
  worker: 1
  compression_level: 3
  loadbalance: true
processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~
/ # 
```

重启filebeat

```bash
/ # ps -ef
PID   USER     TIME  COMMAND
    7 root      0:02 /usr/local/filebeat-7.11.1-linux-x86_64/filebeat --path.co
/ # kill -9 7

# 验证redis没有日志
127.0.0.1:6379[12]> FLUSHDB
OK
127.0.0.1:6379[12]> KEYS *
(empty list or set)
127.0.0.1:6379[12]> KEYS *
(empty list or set)
127.0.0.1:6379[12]> KEYS *
(empty list or set)
127.0.0.1:6379[12]> KEYS *
(empty list or set)
127.0.0.1:6379[12]> KEYS *
(empty list or set)

   
#先停止第二个写es的logstash


# 启动
/ # /usr/local/filebeat-7.11.1-linux-x86_64/filebeat   --path.config /usr/local/filebeat-7.11.1-linux-x86_64/ -c  /usr/local/filebeat-7.11.1-linux-x86_64/filebeat.yml --path.data /usr/local/filebeat-7.11.1-linux-x86_64/data/ --path.home /usr/local/filebeat-7.11.1-linux-x86_64/ --path.logs /usr/local/filebeat-7.11.1-linux-x86_64/logs


# 验证redis无日志
127.0.0.1:6379[12]> KEYS *
(empty list or set)
127.0.0.1:6379[12]> KEYS *
(empty list or set)
127.0.0.1:6379[12]> KEYS *
(empty list or set)
```

现在请求nginx http://test.youwoyouqu.io

![image-20210309094742047](http://myapp.img.mykernel.cn/image-20210309094742047.png)

> 一定要使用chrome浏览器，其他浏览器可能会加载很多其他资源

```bash
# 验证redis日志
127.0.0.1:6379[12]> KEYS *
1) "nginxlog-filebeat"
127.0.0.1:6379[12]> LLEN nginxlog-filebeat
(integer) 34
```

接上第2个logstash

```bash
/usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/nginx.conf
```

查看es日志, 可以查看到这34个日志

![image-20210309155025776](http://myapp.img.mykernel.cn/image-20210309155025776.png)



#### 发版basenginx及nginx controller (完整版filebeat.yaml)

先修正basenginx的filebeatyaml

```diff
root@master01:/data/weizhixiu/dockerfile/nginx-baseimage-dockerfile# cat filebeat.yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /apps/nginx/logs/access.log
  exclude_lines: ['^DBG']
+  exclude_lines: ['kube-probe/1.20']
  exclude_files: ['.gz$']
  fields:
    type: nginx-accesslog-filebeat
    level: debug
    review: 1
- type: log
  enabled: true
  paths:
    - /apps/nginx/logs/error.log
  exclude_lines: ['^DBG']
+  exclude_lines: ['kube-probe/1.20']
  exclude_files: ['.gz$']
  fields:
    type: nginx-errorlog-filebeat
    level: debug
    review: 1
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
setup.template.settings:
  index.number_of_shards: 1
setup.kibana:
output.logstash:
  hosts: ["192.168.0.246:5045"]
  worker: 1
  compression_level: 3
  loadbalance: true
processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~

```

```bash

root@master01:/data/weizhixiu/dockerfile/nginx-baseimage-dockerfile# bash build.sh 

# jenkins Pod中发版nginx
/opt/data/workspace/admin-web-new # /opt/scripts/admin-web-new.sh 
```

```bash
root@master01:/data/weizhixiu/dockerfile/nginx-baseimage-dockerfile# cat Dockerfile 
FROM harbor.youwoyouqu.io/baseimage/alpine:base

LABEL AUTHOR=songliangcheng QQ=1062670898

RUN apk add --no-cache --virtual .build-deps \
  gcc \
  libc-dev \
  make \
  openssl-dev \
  pcre-dev \
  zlib-dev \
  linux-headers \
  curl \
  gnupg \
  libxslt-dev \
  gd-dev \
  geoip-dev \
  perl-dev 
  
ADD nginx-1.16.1.tar.gz ./
RUN cd nginx-1.16.1 && \
 ./configure --prefix=/apps/nginx \
 --user=root \
 --group=root \
 --pid-path=/run/nginx.pid \
 --with-threads \
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
 --with-pcre && \
    make -j `grep -c processor /proc/cpuinfo` && make install 

RUN install -dv /apps/nginx/conf/conf.d/


# 配置json日志
ADD nginx.conf /apps/nginx/conf/nginx.conf

# 添加filebeat
ADD filebeat-7.11.1-linux-x86_64.tar.gz /usr/local/
ADD filebeat.yml /usr/local/filebeat-7.11.1-linux-x86_64/

# 清理
RUN rm -fr /nginx-1.16.1

#CMD ["/apps/nginx/sbin/nginx", "-g", "daemon off;"]
ADD run.sh /usr/local/bin/
CMD ["run.sh"]
```

```bash
root@master01:/data/weizhixiu/dockerfile/nginx-baseimage-dockerfile# cat run.sh 
#!/bin/sh
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-03-08
#FileName：             run.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************


nohup /usr/local/filebeat-7.11.1-linux-x86_64/filebeat   --path.config /usr/local/filebeat-7.11.1-linux-x86_64/ -c  /usr/local/filebeat-7.11.1-linux-x86_64/filebeat.yml --path.data /usr/local/filebeat-7.11.1-linux-x86_64/data/ --path.home /usr/local/filebeat-7.11.1-linux-x86_64/ --path.logs /usr/local/filebeat-7.11.1-linux-x86_64/logs &
/apps/nginx/sbin/nginx
tail -f /etc/hosts
```



现在先停了写es的logstash, 查看redis的key状态

```bash
127.0.0.1:6379[12]> LLEN nginxlog-filebeat
(integer) 0
127.0.0.1:6379[12]> LLEN nginxlog-filebeat
(integer) 0


# Chrome重新请求
#略, 还是34个请求


# 查看redis
127.0.0.1:6379[12]> LLEN nginxlog-filebeat
(integer) 34
```

接上logstash

```bash
root@master05:~# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/nginx.conf  
```

kibana查询

![image-20210309155747930](http://myapp.img.mykernel.cn/image-20210309155747930.png)



现在扩容nginx

```bash
root@master01:/data/weizhixiu/dockerfile/nginx-baseimage-dockerfile# kubectl scale deploy -n weizhixiu nginx-controller-deploy --replicas=2
deployment.apps/nginx-controller-deploy scaled

# 正常后
nginx-controller-deploy-d7cd749c-gtnw5         1/1     Running   0          76s
nginx-controller-deploy-d7cd749c-mnwrw         1/1     Running   0          3m18s


# 浏览器请求

# kibana查询

```

##### haproxy透传ip

查询过程中发现clientip地址是192.168.0.27, tcp-xff, xff为空。

haproxy(81) -> nginx , 需要haproxy传递源地址,参考：[haproxy编译安装与配置优化](http://blog.mykernel.cn/2020/10/16/haproxy%E7%BC%96%E8%AF%91%E5%AE%89%E8%A3%85%E5%8F%8A%E4%BC%98%E5%8C%96/)

现在修改haproxy配置

```bash
defaults
    timeout connect 10s
    timeout client  1m
    timeout server  1m     
    option forwardfor      # 转发源地址
    option http-keep-alive # 长连接
    option redispatch    # server1不处理的请求转发给其他server

listen ingress-80
	bind 192.168.0.247:80
	mode http # http
	#log global # 不记录日志
    #option httplog 
    option forwardfor # xforwaredfor字段
    http-response del-header Server # 删除server信息
    http-response del-header x-powered-by
    http-response add-header X-Via "linux49 WeiZhiXiu HaProxy" # 添加字段
	server 192.168.0.27 192.168.0.27:30088 check inter 3s fall 2 rise 5

```

```bash
root@haproxy01:~# systemctl restart haproxy

# 请求验证
root@haproxy01:~# curl -I test.youwoyouqu.io
HTTP/1.1 200 OK
Date: Tue, 09 Mar 2021 08:22:01 GMT
Content-Type: text/html
Content-Length: 2479
Last-Modified: Tue, 09 Mar 2021 01:04:52 GMT
Keep-Alive: timeout=90         # nginx长连接
Vary: Accept-Encoding          # nginx压缩
ETag: "6046c9b4-9af"
Accept-Ranges: bytes
X-Via: linux49 WeiZhiXiu HaProxy # x-via

```

再次请求网站，查看kibana

> 参考：http://blog.mykernel.cn/2020/12/29/nginx%E7%9A%84proxy-set-header/

![image-20210309162651629](http://myapp.img.mykernel.cn/image-20210309162651629.png)



### 配置java Docker支持日志收集

#### java-filebeat基础镜像

由于所有java我均把日志写在`/apps/logs/app.log`

所以可以在jdk基础镜像，做好filebeat及yaml

```bash
root@master01:/data/weizhixiu/dockerfile# cp -a jdk-dockerfile jar-baseimage-dockerfile
```

```diff
root@master01:/data/weizhixiu/dockerfile/jar-baseimage-dockerfile# cat filebeat.yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /apps/logs/app.log
  exclude_lines: ['^DBG']
  exclude_files: ['.gz$']
  fields:
+    type: APP_NAME-filebeat
    level: debug
    review: 1
  ### Multiline options

  # Multiline can be used for log messages spanning multiple lines. This is common
  # for Java Stack Traces or C-Line Continuation

  # The regexp Pattern that has to be matched. The example pattern matches all lines starting with [
+  multiline.pattern: ^\[

  # Defines if the pattern set under pattern should be negated or not. Default is false.
+  multiline.negate: false

  # Match can be set to "after" or "before". It is used to define if lines should be append to a pattern
  # that was (not) matched before or after or as long as a pattern is not matched based on negate.
  # Note: After is the equivalent to previous and before is the equivalent to to next in Logstash
+  multiline.match: after

filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
setup.template.settings:
  index.number_of_shards: 1
setup.kibana:
output.logstash:
+  hosts: ["192.168.0.246:5046"]
  worker: 1
  compression_level: 3
  loadbalance: true
processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~

```

> 5045会做json解析
>
> 5046不会，还需要加haproxy
>
> ```bash
> listen logstash-5046
> 	bind 192.168.0.246:5046
> 	mode tcp
> 	#server 192.168.0.83 192.168.0.83:8081 check inter 3s fall 2 rise 5
> 	server 192.168.0.171 192.168.0.171:5046 check inter 3s fall 2 rise 5
> root@haproxy01:~# systemctl restart haproxy
> ```

准备dockerfile

```diff
root@master01:/data/weizhixiu/dockerfile/jar-baseimage-dockerfile# cat Dockerfile 
FROM harbor.youwoyouqu.io/baseimage/alpine:base


LABEL AUTHOR=songliangcheng QQ=1062670898

ENV JAVA_HOME=/usr/local/jdk  
ENV TOMCAT_HOME=/apps/tomcat 
ENV PATH=\$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$TOMCAT_HOME/bin:$PATH 
ENV CLASSPATH=.$CLASSPATH:$JAVA_HOME/lib:$JAVA_HOME/jre/lib:$TOMCAT_HOME/lib/tools.jar

ADD jdk-8u221-linux-x64.tar.gz /usr/local/src/
RUN ln -sv /usr/local/src/jdk1.8.0_221 /usr/local/jdk 



+# 添加filebeat
+ADD filebeat-7.11.1-linux-x86_64.tar.gz /usr/local/
+ADD filebeat.yml /usr/local/filebeat-7.11.1-linux-x86_64/

#CMD ["java","-version"]
```

准备build.sh

```bash
root@master01:/data/weizhixiu/dockerfile/jar-baseimage-dockerfile# cat build.sh
docker build -t harbor.youwoyouqu.io/baseimage/jdk:8-filebeat .
docker run --rm  harbor.youwoyouqu.io/baseimage/jdk:8-filebeat

docker push harbor.youwoyouqu.io/baseimage/jdk:8-filebeat
```

```bash
root@master01:/data/weizhixiu/dockerfile/jar-baseimage-dockerfile# bash build.sh 
```



#### 测试一个服务

```bash
root@master01:/data/weizhixiu/dockerfile# cp -a auth-server-dockerfile auth-server-dockerfile-test
```

```diff
root@master01:/data/weizhixiu/dockerfile/auth-server-dockerfile-test# cat Dockerfile
FROM harbor.youwoyouqu.io/baseimage/jdk:8-filebeat
LABEL LABEL AUTHOR=songliangcheng QQ=1062670898

ADD auth-server-1.0.0-SNAPSHOT.jar run.sh /apps/


+# 修正filebeat.yml
+RUN sed -i "s@type: APP_NAME-filebeat@type: auth-server-filebeat@g" /usr/local/filebeat-7.11.1-linux-x86_64/filebeat.yml

WORKDIR /apps/
CMD ["/apps/run.sh"]
```

之后run.sh脚本修改

```diff
root@master01:/data/weizhixiu/dockerfile/auth-server-dockerfile-test# cat run.sh
#!/bin/sh
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-02-26
#FileName：             run.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************





install -d /apps/logs
+nohup /usr/local/filebeat-7.11.1-linux-x86_64/filebeat   --path.config /usr/local/filebeat-7.11.1-linux-x86_64/ -c  /usr/local/filebeat-7.11.1-linux-x86_64/filebeat.yml --path.data /usr/local/filebeat-7.11.1-linux-x86_64/data/ --path.home /usr/local/filebeat-7.11.1-linux-x86_64/ --path.logs /usr/local/filebeat-7.11.1-linux-x86_64/logs &
nohup java -jar auth-server-1.0.0-SNAPSHOT.jar --spring.profiles.active=test -server  -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=5       &> /apps/logs/app.log     &
tail -f /etc/hosts
# test
# java -jar auth-server-1.0.0-SNAPSHOT.jar --spring.profiles.active=test -server  -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=5 
```

测试build

```bash
root@master01:/data/weizhixiu/dockerfile/auth-server-dockerfile-test# docker build -t harbor.youwoyouqu.io/weizhixiu/auth-server:202103091649 .
```

测试运行

```bash
root@master01:/data/weizhixiu/dockerfile/auth-server-dockerfile-test# docker run  --rm -it  harbor.youwoyouqu.io/weizhixiu/auth-server:202103091654 sh


# 先后台运行java
/apps # install -d /apps/logs
/apps # nohup java -jar auth-server-1.0.0-SNAPSHOT.jar --spring.profiles.active=test -server  -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=5       &> /apps/logs/app.log     &

# 观察日志
2021-03-09 17:10:12.508  INFO 91 --- [           main] c.y.p.cloud.auth.AuthServerApplication   : The following profiles are active: test

#修正配置
/apps # cat /usr/local/filebeat-7.11.1-linux-x86_64/filebeat.yml 
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /apps/logs/app.log
  exclude_lines: ['^DBG']
  exclude_files: ['.gz$']
  fields:
    type: auth-server-filebeat
    level: debug
    review: 1
  
  # 参考：https://www.elastic.co/guide/en/beats/filebeat/current/multiline-examples.html
  multiline.pattern: ^\d+-\d+-\d+
  multiline.negate: true
  multiline.match: after

filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
setup.template.settings:
  index.number_of_shards: 1
setup.kibana:
output.logstash:
  hosts: ["192.168.0.246:5046"]
  worker: 1
  compression_level: 3
  loadbalance: true
processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~

# 重启java
/apps # ps -ef
PID   USER     TIME  COMMAND
    1 root      0:00 sh
   91 root      0:34 java -jar auth-server-1.0.0-SNAPSHOT.jar --spring.profiles.active=test -server -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=5
  157 root      0:00 ps -ef
/apps # kill -9 91
nohup java -jar auth-server-1.0.0-SNAPSHOT.jar --spring.profiles.active=test -server  -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=5       &> /apps/logs/app.log     &


#再启动日志
/apps # /usr/local/filebeat-7.11.1-linux-x86_64/filebeat   --path.config /usr/local/filebeat-7.11.1-linux-x86_64/ -c  /usr/local/filebeat-7.11.1-linux-x86_64/filebeat.yml --path.data /usr/local/filebeat-7.11.1-linux-x86_64/data/ --path.home /usr/local/filebeat-7.11.1-linux-x86_64/ --path.logs /usr/local/filebeat-7.11.1-linux-x86_64/logs -e

2021-03-09T16:57:12.471+0800	INFO	[publisher_pipeline_output]	pipeline/output.go:151	Connection to backoff(async(tcp://192.168.0.246:5046)) established
2021-03-09T16:57:12.478+0800	ERROR	[logstash]	logstash/async.go:280	Failed to publish events caused by: EOF
2021-03-09T16:57:12.478+0800	INFO	[publisher]	pipeline/retry.go:219	retryer: send unwait signal to consumer

```

#### 配置filebeat -> logstash ->redis

在第一个logstash

```bash
[root@elk ~]# cat /etc/logstash/conf.d/beats-java.conf
input {
  beats {
	port => 5046
  }
}

output {
	if [fields][type] == "auth-server-filebeat" {
	    redis {
			data_type => "list"
			key => "java-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "11"
			password => "linux48"
		}  
		stdout{ codec => rubydebug }
	}
}

```

```bash
[root@elk ~]# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/beats-java.conf 

# 启动之后，可以观察到filebeat不在报错
2021-03-09T17:00:29.993+0800	INFO	[publisher_pipeline_output]	pipeline/output.go:151	Connection to backoff(async(tcp://192.168.0.246:5046)) established

# 并且输出日志
{
         "agent" => {
                "name" => "27c450ba8976",
                  "id" => "ff1f4731-cf69-4852-b9f5-23054bd80d79",
            "hostname" => "27c450ba8976",
        "ephemeral_id" => "8e984b05-14d2-46cc-a7e2-b10cf0f6ce1b",
                "type" => "filebeat",
             "version" => "7.11.1"
```

查看redis

```bash
127.0.0.1:6379[11]> LLEN java-filebeat
(integer) 37
127.0.0.1:6379[11]> LPOP java-filebeat
"{\"fields\":{\"review\":1,\"type\":\"auth-server-filebeat\",\"level\":\"debug\"},\"ecs\":{\"version\":\"1.6.0\"},\"log\":{\"offset\":0,\"flags\":[\"multiline\"],\"file\":{\"path\":\"/apps/logs/app.log\"}},\"@timestamp\":\"2021-03-09T09:20:33.284Z\",\"@version\":\"1\",\"host\":{\"name\":\"27c450ba8976\",\"mac\":[\"02:42:ac:11:00:03\"],\"ip\":[\"172.17.0.3\"],\"architecture\":\"x86_64\",\"os\":{\"family\":\"\",\"name\":\"Alpine Linux\",\"version\":\"\",\"kernel\":\"4.15.0-29-generic\",\"platform\":\"alpine\"},\"hostname\":\"27c450ba8976\",\"containerized\":true},\"tags\":[\"beats_input_codec_plain_applied\"],\"input\":{\"type\":\"log\"},\"message\":\"  .   ____          _            __ _ _\\n /\\\\\\\\ / ___'_ __ _ _(_)_ __  __ _ \\\\ \\\\ \\\\ \\\\\\n( ( )\\\\___ | '_ | '_| | '_ \\\\/ _` | \\\\ \\\\ \\\\ \\\\\\n \\\\\\\\/  ___)| |_)| | | | | || (_| |  ) ) ) )\\n  '  |____| .__|_| |_|_| |_\\\\__, | / / / /\\n =========|_|==============|___/=/_/_/_/\\n :: Spring Boot ::        (v2.3.2.RELEASE)\\n\",\"agent\":{\"name\":\"27c450ba8976\",\"type\":\"filebeat\",\"id\":\"ff1f4731-cf69-4852-b9f5-23054bd80d79\",\"ephemeral_id\":\"39f57610-58a5-4d14-8c01-94cf8676006d\",\"version\":\"7.11.1\",\"hostname\":\"27c450ba8976\"}}"
127.0.0.1:6379[11]> LPOP java-filebeat
"{\"fields\":{\"type\":\"auth-server-filebeat\",\"review\":1,\"level\":\"debug\"},\"ecs\":{\"version\":\"1.6.0\"},\"log\":{\"offset\":775,\"file\":{\"path\":\"/apps/logs/app.log\"}},\"@timestamp\":\"2021-03-09T09:20:36.285Z\",\"host\":{\"name\":\"27c450ba8976\",\"mac\":[\"02:42:ac:11:00:03\"],\"ip\":[\"172.17.0.3\"],\"architecture\":\"x86_64\",\"os\":{\"name\":\"Alpine Linux\",\"family\":\"\",\"kernel\":\"4.15.0-29-generic\",\"version\":\"\",\"platform\":\"alpine\"},\"containerized\":true,\"hostname\":\"27c450ba8976\"},\"@version\":\"1\",\"tags\":[\"beats_input_codec_plain_applied\"],\"input\":{\"type\":\"log\"},\"agent\":{\"name\":\"27c450ba8976\",\"type\":\"filebeat\",\"id\":\"ff1f4731-cf69-4852-b9f5-23054bd80d79\",\"ephemeral_id\":\"39f57610-58a5-4d14-8c01-94cf8676006d\",\"version\":\"7.11.1\",\"hostname\":\"27c450ba8976\"},\"message\":\"2021-03-09 17:20:35.694  INFO 304 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 477ms. Found 1 JPA repository interfaces.\"}"


127.0.0.1:6379[11]> RPOP java-filebeat
"{\"fields\":{\"type\":\"auth-server-filebeat\",\"review\":1,\"level\":\"debug\"},\"ecs\":{\"version\":\"1.6.0\"},\"log\":{\"offset\":8092,\"file\":{\"path\":\"/apps/logs/app.log\"}},\"@timestamp\":\"2021-03-09T09:20:48.293Z\",\"@version\":\"1\",\"host\":{\"name\":\"27c450ba8976\",\"mac\":[\"02:42:ac:11:00:03\"],\"ip\":[\"172.17.0.3\"],\"architecture\":\"x86_64\",\"os\":{\"family\":\"\",\"name\":\"Alpine Linux\",\"kernel\":\"4.15.0-29-generic\",\"version\":\"\",\"platform\":\"alpine\"},\"containerized\":true,\"hostname\":\"27c450ba8976\"},\"tags\":[\"beats_input_codec_plain_applied\"],\"input\":{\"type\":\"log\"},\"agent\":{\"name\":\"27c450ba8976\",\"type\":\"filebeat\",\"id\":\"ff1f4731-cf69-4852-b9f5-23054bd80d79\",\"ephemeral_id\":\"39f57610-58a5-4d14-8c01-94cf8676006d\",\"version\":\"7.11.1\",\"hostname\":\"27c450ba8976\"},\"message\":\"2021-03-09 17:20:47.336  INFO 304 --- [           main] c.y.p.cloud.auth.AuthServerApplication   : Started AuthServerApplication in 17.831 seconds (JVM running for 19.116)\"}"
```

> 注意：LPOP表示第一行，可以发现已经汇聚成一行

#### 配置redis->logstash->es

```bash
root@master05:~# cat /etc/logstash/conf.d/java.conf
input {
	    redis {
			data_type => "list"
			key => "java-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "11"
			password => "linux48"
		}  
}

output {
	if [fields][type] == "auth-server-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "auth-server-filebeat-%{+YYYY.MM.dd}"
		}
		stdout{ codec => rubydebug }
	}
}

```

```bash
[root@elk ~]# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/java.conf  
```

可以查看终端输出日志

kibana添加

![image-20210309173735326](http://myapp.img.mykernel.cn/image-20210309173735326.png)

![image-20210309173744038](http://myapp.img.mykernel.cn/image-20210309173744038.png)

![image-20210309173800164](http://myapp.img.mykernel.cn/image-20210309173800164.png)

#### 发版jar-baseimage及auth-server(完整版filebeat.yaml)

```diff
root@master01:/data/weizhixiu/dockerfile/jar-baseimage-dockerfile# cat filebeat.yml 
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /apps/logs/app.log
  exclude_lines: ['^DBG']
  exclude_files: ['.gz$']
  fields:
+    type: APP_NAME-filebeat
    level: debug
    review: 1
   # 参考：https://www.elastic.co/guide/en/beats/filebeat/current/multiline-examples.html
+  multiline.pattern: ^\d+-\d+-\d+
+  multiline.negate: true
+  multiline.match: after

filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
setup.template.settings:
  index.number_of_shards: 1
setup.kibana:
output.logstash:
  hosts: ["192.168.0.246:5046"]
  worker: 1
  compression_level: 3
  loadbalance: true
processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~
```

```diff
root@master01:/data/weizhixiu/dockerfile/jar-baseimage-dockerfile# cat Dockerfile 
FROM harbor.youwoyouqu.io/baseimage/alpine:base


LABEL AUTHOR=songliangcheng QQ=1062670898

ENV JAVA_HOME=/usr/local/jdk  
ENV TOMCAT_HOME=/apps/tomcat 
ENV PATH=\$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$TOMCAT_HOME/bin:$PATH 
ENV CLASSPATH=.$CLASSPATH:$JAVA_HOME/lib:$JAVA_HOME/jre/lib:$TOMCAT_HOME/lib/tools.jar

ADD jdk-8u221-linux-x64.tar.gz /usr/local/src/
RUN ln -sv /usr/local/src/jdk1.8.0_221 /usr/local/jdk 



+# 添加filebeat
+ADD filebeat-7.11.1-linux-x86_64.tar.gz /usr/local/
+ADD filebeat.yml /usr/local/filebeat-7.11.1-linux-x86_64/

#CMD ["java","-version"]
```

```diff
root@master01:/data/weizhixiu/dockerfile/jar-baseimage-dockerfile# cat build.sh 
+docker build -t harbor.youwoyouqu.io/baseimage/jdk:8-filebeat .
+docker run --rm  harbor.youwoyouqu.io/baseimage/jdk:8-filebeat

+docker push harbor.youwoyouqu.io/baseimage/jdk:8-filebeat
```

```bash
root@master01:/data/weizhixiu/dockerfile/jar-baseimage-dockerfile# bash build.sh 
```

auth-server

```diff
root@master01:/data/weizhixiu/dockerfile/auth-server-dockerfile-test# cat Dockerfile 
FROM harbor.youwoyouqu.io/baseimage/jdk:8-filebeat
LABEL LABEL AUTHOR=songliangcheng QQ=1062670898

ADD auth-server-1.0.0-SNAPSHOT.jar run.sh /apps/


+# 修正filebeat.yml
+RUN sed -i "s@type: APP_NAME-filebeat@type: auth-server-filebeat@g" /usr/local/filebeat-7.11.1-linux-x86_64/filebeat.yml

WORKDIR /apps/
CMD ["/apps/run.sh"]

```

```diff
root@master01:/data/weizhixiu/dockerfile/auth-server-dockerfile-test# cat run.sh 
#!/bin/sh
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-02-26
#FileName：             run.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************





install -d /apps/logs
+nohup /usr/local/filebeat-7.11.1-linux-x86_64/filebeat   --path.config /usr/local/filebeat-7.11.1-linux-x86_64/ -c  /usr/local/filebeat-7.11.1-linux-x86_64/filebeat.yml --path.data /usr/local/filebeat-7.11.1-linux-x86_64/data/ --path.home /usr/local/filebeat-7.11.1-linux-x86_64/ --path.logs /usr/local/filebeat-7.11.1-linux-x86_64/logs &
nohup java -jar auth-server-1.0.0-SNAPSHOT.jar --spring.profiles.active=test -server  -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=5       &> /apps/logs/app.log     &
tail -f /etc/hosts
# test
# java -jar auth-server-1.0.0-SNAPSHOT.jar --spring.profiles.active=test -server  -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=5 
```



测试auth-server

```bash
root@master01:/data/weizhixiu/dockerfile/auth-server-dockerfile-test# docker build -t harbor.youwoyouqu.io/weizhixiu/auth-server:202103091730 .

# 先清空redis
127.0.0.1:6379[11]> FLUSHDB
OK
127.0.0.1:6379[11]> KEYS *
(empty list or set)
127.0.0.1:6379[11]> KEYS *
(empty list or set)

root@master01:/data/weizhixiu/dockerfile/auth-server-dockerfile-test# docker run  --rm -it  harbor.youwoyouqu.io/weizhixiu/auth-server:202103091654 
```

观察logstash有日志输出

查看redis

```bash
127.0.0.1:6379[11]> KEYS *
1) "java-filebeat"
127.0.0.1:6379[11]> LLEN java-filebeat
(integer) 44
127.0.0.1:6379[11]> LLEN java-filebeat
(integer) 44
127.0.0.1:6379[11]> LLEN java-filebeat
(integer) 44
127.0.0.1:6379[11]> LLEN java-filebeat
(integer) 44
```

#### 修改之前的分层构建脚本

```diff
root@master01:/data/weizhixiu/stage-builder/weizhixiu2.0# cat Dockerfile 
############################# JAVA 所有项目 编译方法, 王申，谢如华 #############################
FROM harbor.youwoyouqu.io/baseimage/jdk:8 AS weizhixiu2.0

# java项目
RUN apk add --no-cache maven



# 1. 只有部分文件如 package.json 发生变更的时候，才重新执行 npm install
COPY  weizhixiu2.0/pom.xml                                    weizhixiu2.0/
COPY  weizhixiu2.0/module/pom.xml                             weizhixiu2.0/module/
COPY  weizhixiu2.0/cloud-server/pom.xml                       weizhixiu2.0/cloud-server/
COPY  weizhixiu2.0/cloud-server/user-server/pom.xml           weizhixiu2.0/cloud-server/user-server/
COPY  weizhixiu2.0/cloud-server/wzx-mobile-gateway/pom.xml    weizhixiu2.0/cloud-server/wzx-mobile-gateway/
COPY  weizhixiu2.0/cloud-server/wzx-admin-gateway/pom.xml     weizhixiu2.0/cloud-server/wzx-admin-gateway/
COPY  weizhixiu2.0/cloud-server/payment-server/pom.xml        weizhixiu2.0/cloud-server/payment-server/
COPY  weizhixiu2.0/cloud-server/video-server/pom.xml          weizhixiu2.0/cloud-server/video-server/
COPY  weizhixiu2.0/cloud-server/auth-server/pom.xml           weizhixiu2.0/cloud-server/auth-server/
COPY  weizhixiu2.0/cloud-server/video-comment-server/pom.xml  weizhixiu2.0/cloud-server/video-comment-server/

# 添加配置, 配置mirrors
COPY settings.xml ./
RUN cd weizhixiu2.0 && mvn  dependency:go-offline --settings /settings.xml


# 2. 剩余的文件
COPY weizhixiu2.0 weizhixiu2.0/

# 编译${APP_NAME}
ARG APP_NAME=auth-server
RUN cd weizhixiu2.0/cloud-server/${APP_NAME}/ && mvn clean package -DskipTests --settings /settings.xml

############################# jar, 运维：宋亮成 #############################
FROM harbor.youwoyouqu.io/baseimage/jdk:8-filebeat
LABEL LABEL AUTHOR=songliangcheng QQ=1062670898

ARG APP_NAME=auth-server

ADD run.sh /apps/
RUN sed -i "s@auth-server@${APP_NAME}@g" /apps/run.sh
+# 修正filebeat.yml
+RUN sed -i "s@type: APP_NAME-filebeat@type: ${APP_NAME}-filebeat@g" /usr/local/filebeat-7.11.1-linux-x86_64/filebeat.yml

COPY --from=weizhixiu2.0 /weizhixiu2.0/cloud-server/${APP_NAME}/target/${APP_NAME}-1.0.0-SNAPSHOT.jar /apps/

WORKDIR /apps/
CMD ["/apps/run.sh"]
```

```diff
root@master01:/data/weizhixiu/stage-builder/weizhixiu2.0# cat run.sh 
#!/bin/sh
#
#********************************************************************
#Author:                songliangcheng
#QQ:                    2192383945
#Date:                  2021-02-26
#FileName：             run.sh
#URL:                   http://www.magedu.com
#Description：          A test toy
#Copyright (C):        2021 All rights reserved
#********************************************************************





install -d /apps/logs
+nohup /usr/local/filebeat-7.11.1-linux-x86_64/filebeat   --path.config /usr/local/filebeat-7.11.1-linux-x86_64/ -c  /usr/local/filebeat-7.11.1-linux-x86_64/filebeat.yml --path.data /usr/local/filebeat-7.11.1-linux-x86_64/data/ --path.home /usr/local/filebeat-7.11.1-linux-x86_64/ --path.logs /usr/local/filebeat-7.11.1-linux-x86_64/logs &
nohup java -jar auth-server-1.0.0-SNAPSHOT.jar --spring.profiles.active=test -server  -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=5       &> /apps/logs/app.log     &
tail -f /etc/hosts
# test
# java -jar auth-server-1.0.0-SNAPSHOT.jar --spring.profiles.active=test -server  -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=5 
```

jenkins pod发版

```bash
/opt/data/workspace/weizhixiu2.0 #  /opt/scripts/weizhixiu2.0.sh  auth-server deploy
/opt/data/workspace/weizhixiu2.0 #  /opt/scripts/weizhixiu2.0.sh  user-server deploy #
/opt/data/workspace/weizhixiu2.0 #  /opt/scripts/weizhixiu2.0.sh  video-comment-server deploy
/opt/data/workspace/weizhixiu2.0 #  /opt/scripts/weizhixiu2.0.sh  video-server deploy
/opt/data/workspace/weizhixiu2.0 #  /opt/scripts/weizhixiu2.0.sh  wzx-admin-gateway deploy
/opt/data/workspace/weizhixiu2.0 #  /opt/scripts/weizhixiu2.0.sh  wzx-mobile-gateway deploy


```

配置java的logstash

```bash
[root@elk ~]# cat /etc/logstash/conf.d/beats-java.conf
input {
  beats {
	port => 5046
  }
}

output {
	if [fields][type] == "auth-server-filebeat" {
	    redis {
			data_type => "list"
			key => "java-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "11"
			password => "linux48"
		}  
	#	stdout{ codec => rubydebug }
	}
	if [fields][type] == "user-server-filebeat" {
	    redis {
			data_type => "list"
			key => "java-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "11"
			password => "linux48"
		}  
	#	stdout{ codec => rubydebug }
	}
	if [fields][type] == "video-comment-server-filebeat" {
	    redis {
			data_type => "list"
			key => "java-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "11"
			password => "linux48"
		}  
	#	stdout{ codec => rubydebug }
	}
	if [fields][type] == "video-server-filebeat" {
	    redis {
			data_type => "list"
			key => "java-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "11"
			password => "linux48"
		}  
	#	stdout{ codec => rubydebug }
	}
	if [fields][type] == "wzx-admin-gateway-filebeat" {
	    redis {
			data_type => "list"
			key => "java-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "11"
			password => "linux48"
		}  
	#	stdout{ codec => rubydebug }
	}
	if [fields][type] == "wzx-mobile-gateway-filebeat" {
	    redis {
			data_type => "list"
			key => "java-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "11"
			password => "linux48"
		}  
	#	stdout{ codec => rubydebug }
	}
}
```

```bash
[root@elk ~]# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/beats-java.conf 
```

配置查看redis

```bash
127.0.0.1:6379[11]> DBSIZE
(integer) 1
127.0.0.1:6379[11]> KEYS *
1) "java-filebeat"
127.0.0.1:6379[11]> LLEN java-filebeat
(integer) 151
127.0.0.1:6379[11]> LLEN java-filebeat
(integer) 155
```

配置logstash 2写不同的索引

```bash
root@localhost:~# cat /etc/logstash/conf.d/java.conf 
input {
	    redis {
			data_type => "list"
			key => "java-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "11"
			password => "linux48"
		}  
}

output {
	if [fields][type] == "auth-server-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "auth-server-filebeat-%{+YYYY.MM.dd}"
		}
		#stdout{ codec => rubydebug }
	}
	if [fields][type] == "user-server-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "user-server-filebeat-%{+YYYY.MM.dd}"
		}
		#stdout{ codec => rubydebug }
	}
	if [fields][type] == "video-comment-server-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "video-comment-server-filebeat-%{+YYYY.MM.dd}"
		}
		#stdout{ codec => rubydebug }
	}
	if [fields][type] == "video-server-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "video-server-filebeat-%{+YYYY.MM.dd}"
		}
		#stdout{ codec => rubydebug }
	}
	if [fields][type] == "wzx-admin-gateway-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "wzx-admin-gateway-filebeat-%{+YYYY.MM.dd}"
		}
		#stdout{ codec => rubydebug }
	}
	if [fields][type] == "wzx-mobile-gateway-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "wzx-mobile-gateway-filebeat-%{+YYYY.MM.dd}"
		}
		#stdout{ codec => rubydebug }
	}
}
```

启动

```bash
root@localhost:~# /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/java.conf 
```

查看索引  http://192.168.0.171:9000

![image-20210310094637635](http://myapp.img.mykernel.cn/image-20210310094637635.png)

kibana添加索引

略

### 完整logstash配置

#### 192.168.0.171 解析json

```bash
[root@elk ~]# ls /etc/logstash/conf.d/
beats-java.conf  beats-nginx.conf 
```

配置java

```diff
[root@elk ~]# cat /etc/logstash/conf.d/beats-java.conf 
input {
  beats {
	port => 5046
  }
}

output {
	if [fields][type] == "auth-server-filebeat" {
	    redis {
			data_type => "list"
			key => "java-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "11"
			password => "linux48"
		}  
+	#	stdout{ codec => rubydebug }
	}
	if [fields][type] == "user-server-filebeat" {
	    redis {
			data_type => "list"
			key => "java-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "11"
			password => "linux48"
		}  
+	#	stdout{ codec => rubydebug }
	}
	if [fields][type] == "video-comment-server-filebeat" {
	    redis {
			data_type => "list"
			key => "java-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "11"
			password => "linux48"
		}  
+	#	stdout{ codec => rubydebug }
	}
	if [fields][type] == "video-server-filebeat" {
	    redis {
			data_type => "list"
			key => "java-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "11"
			password => "linux48"
		}  
+	#	stdout{ codec => rubydebug }
	}
	if [fields][type] == "wzx-admin-gateway-filebeat" {
	    redis {
			data_type => "list"
			key => "java-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "11"
			password => "linux48"
		}  
+	#	stdout{ codec => rubydebug }
	}
	if [fields][type] == "wzx-mobile-gateway-filebeat" {
	    redis {
			data_type => "list"
			key => "java-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "11"
			password => "linux48"
		}  
+	#	stdout{ codec => rubydebug }
	}
}
```

配置nginx

```diff
[root@elk ~]# cat /etc/logstash/conf.d/beats-nginx.conf 
input {
  beats {
	port => 5045
	codec => "json"  # 将json的每个元互相转换为一个事件
  }
}

output {
	if [fields][type] == "nginx-accesslog-filebeat" {
	    redis {
			data_type => "list"
			key => "nginxlog-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "12"
			password => "linux48"
		}  
+		#stdout{ codec => rubydebug }
	}
	if [fields][type] == "nginx-errorlog-filebeat" {
	    redis {
			data_type => "list"
			key => "nginxlog-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "12"
			password => "linux48"
		}  
+		#stdout{ codec => rubydebug }
	}
}
```



配置service

```diff
[root@elk ~]# systemctl cat logstash
# /etc/systemd/system/logstash.service
[Unit]
Description=logstash

[Service]
Type=simple
+User=root
+Group=root
# Load env vars from /etc/default/ and /etc/sysconfig/ if they exist.
# Prefixing the path with '-' makes it try to load, but if the file doesn't
# exist, it continues onward.
EnvironmentFile=-/etc/default/logstash
EnvironmentFile=-/etc/sysconfig/logstash
ExecStart=/usr/share/logstash/bin/logstash "--path.settings" "/etc/logstash"
Restart=always
WorkingDirectory=/
Nice=19
LimitNOFILE=16384

# When stopping, how long to wait before giving up and sending SIGKILL?
# Keep in mind that SIGKILL on a process can cause data loss.
TimeoutStopSec=infinity

[Install]
WantedBy=multi-user.target
```

启动

```bash
[root@elk ~]# systemctl start logstash
[root@elk ~]# systemctl enable logstash
Created symlink from /etc/systemd/system/multi-user.target.wants/logstash.service to /etc/systemd/system/logstash.service.
```

#### 192.168.0.85 写es/mysql

```bash
root@localhost:~# ls /etc/logstash/conf.d/
java.conf  nginx.conf
```

配置java

```diff
root@localhost:~# cat /etc/logstash/conf.d/java.conf 
input {
	    redis {
			data_type => "list"
			key => "java-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "11"
			password => "linux48"
		}  
}

output {
	if [fields][type] == "auth-server-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "java-logstash-auth-server-filebeat-%{+YYYY.MM.dd}"
		}
+		#stdout{ codec => rubydebug }
	}
	if [fields][type] == "user-server-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "java-logstash-user-server-filebeat-%{+YYYY.MM.dd}"
		}
+		#stdout{ codec => rubydebug }
	}
	if [fields][type] == "video-comment-server-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "java-logstash-video-comment-server-filebeat-%{+YYYY.MM.dd}"
		}
+		#stdout{ codec => rubydebug }
	}
	if [fields][type] == "video-server-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "java-logstash-video-server-filebeat-%{+YYYY.MM.dd}"
		}
+		#stdout{ codec => rubydebug }
	}
	if [fields][type] == "wzx-admin-gateway-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "java-logstash-wzx-admin-gateway-filebeat-%{+YYYY.MM.dd}"
		}
+		#stdout{ codec => rubydebug }
	}
	if [fields][type] == "wzx-mobile-gateway-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "java-logstash-wzx-mobile-gateway-filebeat-%{+YYYY.MM.dd}"
		}
+		#stdout{ codec => rubydebug }
	}
}

```

配置nginx

```diff
root@localhost:~# cat /etc/logstash/conf.d/nginx.conf 
input {
	    redis {
			data_type => "list"
			key => "nginxlog-filebeat"
			host => "redis.youwoyouqu.io"
			port => 6379
			db => "12"
			password => "linux48"
			codec => json                   # 将json的每个元互相转换为一个事件
		}  
}

filter {
	if [fields][type] == "nginx-accesslog-filebeat" {
		geoip {
+			source => "xff"
			target => "geoip"
			database => "/etc/logstash/GeoLite2-City_20210302/GeoLite2-City.mmdb"
			add_field => ["[geoip][coordinates]","%{[geoip][longitude]}"]
			add_field => ["[geoip][coordinates]","%{[geoip][latitude]}"]
		}
		mutate {
			convert => ["[geoip][coordinates]","float"]
		}
	}
}

output {
	if [fields][type] == "nginx-accesslog-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "logstash-nginx-accesslog-%{+YYYY.MM.dd}"
		}
		jdbc {
		    driver_jar_path => "/usr/share/java/mysql-connector-java-8.0.23.jar"
			connection_string => "jdbc:mysql://192.168.0.171/elk?user=elk&password=123456&useUnicode=true&characterEncoding=UTF8"
+			statement => ["INSERT INTO elklog(host,clientip,status,UserAgent) VALUES(?,?,?,?)", "http_host","xff","status","user-agent"]
		}
+		#stdout{ codec => rubydebug }
	}
	if [fields][type] == "nginx-errorlog-filebeat" {
		elasticsearch {
			hosts => ["192.168.0.171:9200"]
			index => "logstash-nginx-errorlog-%{+YYYY.MM.dd}"
		}
+		#stdout{ codec => rubydebug }
	}
}
```

> 由于工作在haproxy之后的nginx, 真实客户端ip是xff

配置service

```diff
root@localhost:~# systemctl cat logstash.service 
# /etc/systemd/system/logstash.service
[Unit]
Description=logstash

[Service]
Type=simple
+User=root
+Group=root
# Load env vars from /etc/default/ and /etc/sysconfig/ if they exist.
# Prefixing the path with '-' makes it try to load, but if the file doesn't
# exist, it continues onward.
EnvironmentFile=-/etc/default/logstash
EnvironmentFile=-/etc/sysconfig/logstash
ExecStart=/usr/share/logstash/bin/logstash "--path.settings" "/etc/logstash"
Restart=always
WorkingDirectory=/
Nice=19
LimitNOFILE=16384

# When stopping, how long to wait before giving up and sending SIGKILL?
# Keep in mind that SIGKILL on a process can cause data loss.
TimeoutStopSec=infinity

[Install]
WantedBy=multi-user.target
```

启动

```bash
root@localhost:~# systemctl restart logstash.service 
root@localhost:~# systemctl enable logstash.service 
Created symlink /etc/systemd/system/multi-user.target.wants/logstash.service → /etc/systemd/system/logstash.service.
```

#### 验证日志

真实地址

![image-20210310101359067](http://myapp.img.mykernel.cn/image-20210310101359067.png)

![image-20210310101425170](http://myapp.img.mykernel.cn/image-20210310101425170.png)

通过chrome F12无缓存对首页请求,34个请求

![image-20210310101521666](http://myapp.img.mykernel.cn/image-20210310101521666.png)



## 日志仪表盘

参考：http://blog.mykernel.cn/2020/10/30/kibana%E6%B7%BB%E5%8A%A0%E5%B8%B8%E7%94%A8%E5%9B%BE%E5%BD%A2/

优势

- qps衡量访问指标：可以查看业务并发状态
- pv: 总url数, url请求比
- uv: 总源地址，源地址地图索引，每个ip请求次数：防攻击
- status, client事务时长，nginx事务时长
- user-agent, 按比重兼容浏览器

![image-20210325165405562](http://myapp.img.mykernel.cn/image-20210325165405562.png)

> 1. uv是ip个数。2. 真实是基于token统计
> 2. 监控 qps, 每个ip的连接数，响应时间，响应状态码

分析url慢，通过此文章：1. [url慢](http://blog.mykernel.cn/2021/03/04/diary-kubernetes%E4%B8%ADapi%E6%8E%A5%E5%8F%A3%E6%85%A2%E6%8E%92%E6%9F%A5/) 2. [排查网站慢](http://blog.mykernel.cn/2020/10/28/%E6%8E%92%E6%9F%A5%E7%BD%91%E7%AB%99%E6%85%A2/)

### kibana统一入口

```bash
[root@wzx ~]# cat /etc/nginx/conf.d/kibana.conf 
server {
	server_name kibana.youwoyouqu.io;
	listen 80;

	location / {
		proxy_pass http://192.168.0.171:5601;
		proxy_set_header Host $host;
	}
}
```

```bash
nginx -t
nginx -s reload
```

#### 短url

![image-20210310120215283](http://myapp.img.mykernel.cn/image-20210310120215283.png)

# 监控

## prometheus监控

### 监控k8s

https://prometheus.io/download/

第三方：https://prometheus.io/docs/instrumenting/exporters/

haproxy: https://github.com/prometheus/haproxy_exporter

elasticsearch: https://github.com/justwatchcom/elasticsearch_exporter

redis: https://github.com/oliver006/redis_exporter

mysql: https://github.com/prometheus/mysqld_exporter

自定义指标：https://blog.csdn.net/lihao21/article/details/106988390

需要监控的项目



### 监控部署

[监控配置过程](http://blog.mykernel.cn/2021/03/15/diary-prometheus-%E8%87%AA%E5%8A%A8%E5%8F%91%E7%8E%B0%E7%BC%96%E5%86%99%E6%8E%A2%E5%AF%BB/)

[Prometheus+Grafana+Alertmanager监控部署](http://blog.mykernel.cn/2021/03/15/prometheus%E7%9B%91%E6%8E%A7kubernetes/)

[高可用的prometheus](https://yunlzheng.gitbook.io/prometheus-book/part-ii-prometheus-jin-jie/readmd/prometheus-and-high-availability)

[高可用的alertmanager](https://www.modb.pro/db/46819)

## zabbix监控

[zabbix监控部署](http://blog.mykernel.cn/2020/11/11/%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AEkvm%E8%99%9A%E6%8B%9F%E6%9C%BA%EF%BC%8C%E5%B9%B6%E5%88%9B%E5%BB%BA%E8%99%9A%E6%8B%9F%E6%9C%BA/#9-4-9-2-%E5%87%86%E5%A4%87zabbix-server)



## kibana监控日志

### 日志流配置 

统一的java索引名，方便引用java日志

![image-20210327072650482](http://myapp.img.mykernel.cn/image-20210327072650482.png)

### 配置告警

![image-20210327072735633](http://myapp.img.mykernel.cn/image-20210327072735633.png)





![image-20210327072801701](http://myapp.img.mykernel.cn/image-20210327072801701.png)

> 至开发人员邮件，主题就是**测试环境java日志报警**



### 测试告警

![image-20210327072927632](http://myapp.img.mykernel.cn/image-20210327072927632.png)

#### 配置发送给开发，并通知开发查看日志的方式

![image-20210327073405375](http://myapp.img.mykernel.cn/image-20210327073405375.png)

## logstash监控java日志

```bash
input {
  beats {
    port => 5046
  }
}

output {
    if [message] =~ /(exception|ERROR)/ {
        email {
            port => 587
            address => "smtp.qq.com"
            username => "1062670898@qq.com" 
            password => "scyhdklulyrmbdci"
            authentication => "plain"
            contenttype => ""
            from => "1062670898@qq.com"
            subject => "测试环境 %{[fields][type]}错误告警"
            to => "2192383945@qq.com,wangshenaperlin@163.com,1028402351@qq.com"       # 开发邮箱                                
            use_tls => true
            via => "smtp"
            domain => "smtp.qq.com"
            body => "%{[message]}"
            debug => true
        }
    }
}
```

