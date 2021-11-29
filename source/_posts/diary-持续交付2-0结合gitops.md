---
title: "持续交付2.0结合gitops"
date: 2021-04-29 17:40:36
tags:
- "个人日记"
---


# 为什么需要持续集成和部署
<!--more-->

单体应用，一个开发人员足够可以直接发版到服务器.

如果是多个应用，需要集成联合在一起进行调测，又需要部署到不同的环境，就比较麻烦了，所以就需要DevOps方法，以同一种方法部署多套应用，也叫CICD。

# CICD方法论

## 流水线

代码 -> 提交 -> build -> unit test -> integration test -> cd( review -> staging -> production)

##  CI/CD pipeline workflow with kubernetes

![image-20210423153434170](http://myapp.img.mykernel.cn/image-20210423153434170.png)

单个流水线，直接上k8s

CI流水线，可以直接操作kubernetes, 有风险。



## CI/CD with kubernetes and helm

![image-20210514132028631](http://myapp.img.mykernel.cn/image-20210514132028631.png)

k8s有3个环境

开发人员发布 开发分支，发布至开发环境。测试新特性。

测试人员发布， staging分支，staging环境；  master分支，生产环境。

## weave cloud的pull机制

![image-20210514132505614](http://myapp.img.mykernel.cn/image-20210514132505614.png)

上面是ci流水线，最终发布至容器镜像仓库。

下面的deploy，就是一个检测到CI完成自动拉配置仓库，进行部署。

优势：

1. 所有变更在仓库，git 仓库是事实来源，git仓库状态就是k8s之上的应有状态。controller的功能就是将部署清单定义的资源真正创建出组件。
2. ci和cd分离，ci过程中，是不能操作CD中的资源。

# 实践devops

- [x] 双仓库：CI仓库、CD仓库
- [x] OAM模型：devops团队协同管理**声明式配置清单**，CD的配置仓库里面全部是 声明式配置。
- [x] CI/CD**合理工具**
  - [x] CI流水线：jenkins
  - [x] 清单生成：kustomize/helm
  - [x] 发布的operator: jenkins， argocd/jenkinsX/spinnaker
  - [x] GIT仓库
- [x] 对新项目：直接开始。对老项目：变更频率高、易中断的应用程序开始做。



# 开发模型

版本控制系统(Version Control System)存储和追踪目录和文件的修订历史。提供了什么人在什么时间修改什么内容，以及git commit的为什么这样修改？

## 主干开发、主干发布

## 主干开发、分支发布

- 新版本功能开发完成，拉一个新分支
- 新分支完成测试，修复缺陷，达标合并回主干。

特点：

- 主干活动频繁，保障主干代码质量有较大的挑战。
- 分支只修复缺陷，不加新功能。

优势：

- 将发布新功能无关的人员，继续主干开发，不受版本发布影响。
- 新发布的版本出现缺陷，直接在自己的发布分支上进行修复，简单便捷。即使开发主干上的代码已经发生较大的变化，该分支也不会受到影响。

其不足在于：

- 主干上的代码，只能针对下一个新发布版本的功能开发。只要新发布版本的任何功能在主干上还没有开发完成，就不能创建版本发布分支，否则很有可能影响下一个发布的开发计划，开源项目在发布时间点以及特性功能方面的压力小一些，因此常用这种分支方式。
- 对发布分支数据不约束，并且分支周期长，很容易出现“分支地狱”倾向，这种倾向常见于“系列化产品簇+个性化定制”的项目，例如某硬件设备的软件产品研发的分支模式。

## 分支开发、主干发布

![image-20210429195540017](http://myapp.img.mykernel.cn/image-20210429195540017.png)

这种模式是指：

- 从主干上拉出分支，并在分支上开发软件新功能或修复缺陷。
- 某个分支开发完成之后，要对外发布版本时，才合入主干。删除分支？不删除，方便修复 。
-  从主干拉一个分支修复
  - hotfix修复：从主干拉一个分支修复，主干继续开发。

优势：

- 分支合并之前，每个分支之间的开发活动互相不受影响；
- 团队可以自由选择发布哪个分支上的特性。
- 如果新版本出现缺陷，可以直接主干修复或使用hotfix分支修复，简单便捷，无须考虑其他分支。

劣势：

- 分支太多，如果2个特性分支不及时合并到主干，最终2个分支将不能合并了。

规避劣势：

- 需要定开发周期，WPS方法拆分功能。使每个分支应该尽可能短
- 主干代码尽早与分支同步。
- 以主干为主，尽可能不要在特性分支之间合并代码。
- 只推release发布分支



开发、测试流程：王辉、闫鑫

```bash
开发人员：
1. 拉代码
2. 基于本次任务开一个新分支 dev
3. 开发完成后，先构建、单元测试
4. 合并到release分支，推送gitlab.

运维：
5. gitlab push事件触发jenkins完成构建代码、质量扫描。

测试：
6. 手工 发测试环境，测试人员进行冒烟测试：新开发的功能测试
7. 测试好了，测试报告完善，....
8. 当所有功能全部测试完了，进行 回归测试，测试所有登陆、支付、... 主要功能，OK后，告诉维护人员，合并master.

维护人员：
8. 合并至master, 合并前对release分支进行代码质量扫描。
9. 合并后，可以发布灰度。

测试：
10. 发灰度：将测试好的代码，retag

运维：
11. 发生产：将灰度retag.
```

![持续集成](http://myapp.img.mykernel.cn/持续集成.png)

# 敏捷开发、持续集成、持续部署、devops区别

敏捷开发：计划、代码、构建，质量扫描，构建结果反馈

持续集成：敏捷开发 + （自动化/手工）部署测试环境 + （自动化/ 手工 git changlog）测试，测试结果反馈

持续交付：测试完成之后，立即上传到工件库。

持续部署：将工件库的文件直接发版到灰度环境（取决于自动化的程度），部署生产并不是自动化，而是不是以往的半夜上线，可以随时发布。

devops:   部署后的运维、反馈

# 基础镜像准备

## nginx

启动脚本

```bash
bash-5.1# cat /usr/bin/run.sh 
#!/bin/bash
# 环境变量: ENVIRONMENT
# test 测试配置
# staging 灰度配置
# prod    生产配置

# 更新环境配置, DevOps 要求一次构建，不同环境可以直接运行，发版基于升级完成。
[-z "${ENVIRONMENT}" ] || install /apps/nginx/html/envConfig/${ENVIRONMENT}.env.js  /apps/nginx/html/config.js

# 更新cpu核心, 因为默认的pod的cpu是系统核心，并没有限制
[ -n "${CPU_LIMIT}" ] && sed -i "s@worker_processes.*@worker_processes ${CPU_LIMIT};@" /apps/nginx/conf/nginx.conf

# 启动nginx
/apps/nginx/sbin/nginx -g "daemon off;"

```

nginx配置

```bash
bash-5.1# cat /apps/nginx/conf/nginx.conf
user  root;                 
worker_processes  auto;       
worker_cpu_affinity auto; 
error_log  logs/error.log  error;
worker_priority 0;
worker_rlimit_nofile 10000; 
events {
    worker_connections  10000; 
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

    access_log logs/access.log  access_json;   
    directio 4m; 
    sendfile        on; 
    tcp_nopush     on;  
    keepalive_timeout  65 90; 
    keepalive_requests 1000; 
    keepalive_disable msie6; 
    tcp_nodelay on;     
	gzip on; 
	gzip_comp_level 5;
	gzip_min_length 1k;
	gzip_types text/plain application/javascript application/json application/x-javascript text/cssapplication/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;
	gzip_vary on;
    client_max_body_size 1024m; 
    open_file_cache max=10000 inactive=60s; 
    open_file_cache_valid 60s;
    open_file_cache_min_uses 5; 
    open_file_cache_errors on;
    server_tokens off; 
    add_header Cooperation "GraspYun";
    server_names_hash_bucket_size 512;
    server_names_hash_max_size 512;
    include       conf.d/*.conf;
}
```

dockerfile

```bash
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

# 编译nginx依赖                                                                        
RUN apk add --no-cache --virtual .build-deps   gcc   libc-dev   make   openssl-dev   pcre-dev   zlib-dev   linux-headers   curl   gnupg   libxslt-dev   gd-dev   geoip-dev   perl-dev 

# nginx包
ADD nginx-1.16.1.tar.gz ./ 
# 编译安装
RUN cd nginx-1.16.1 &&  ./configure  --prefix=/apps/nginx  --user=root  --group=root  --pid-path=/run/nginx.pid  --with-threads  --with-http_ssl_module  --with-http_v2_module  --with-http_realip_module  --with-http_image_filter_module  --with-http_geoip_module  --with-http_gzip_static_module  --with-http_auth_request_module  --with-http_stub_status_module  --with-http_perl_module  --with-stream  --with-stream_ssl_module  --with-stream_realip_module  --with-stream_geoip_module  --with-pcre &&     make -j `grep -c processor /proc/cpuinfo` && make install 

# 准备目录
RUN install -dv /apps/nginx/conf/conf.d/ 
# 准备配置
ADD nginx.conf /apps/nginx/conf/nginx.conf 
ADD run.sh /usr/bin/run.sh   

# 启动
CMD ["run.sh"]     
```



# jenkins

## 概念

项目类型：自由、流水线

流水线类型：脚本、申明式（面向云原生）

使用申明式流水线，可以将脚本放在Jenkinsfile中，保存在gitlab的仓库中，可以追溯应用变化，定义应用的期望状态，k8s只是负责将git定义的期望状态，实例化出对象。

## 自由脚本

ui接口配置，今天为申明式接口场景趋势场景下，很少使用自由风格的项目

![image-20210514133739633](http://myapp.img.mykernel.cn/image-20210514133739633.png)

## 流水线

![image-20210514134015365](http://myapp.img.mykernel.cn/image-20210514134015365.png)

### 脚本式

```groovy
Jenkinsfile (Scripted Pipeline)
node {  
    stage('Build') { 
        // 
    }
    stage('Test') { 
        // 
    }
    stage('Deploy') { 
        // 
    }
}
```

### 申明式

```groovy
pipeline { 
    agent any 
    options {
        skipStagesAfterUnstable()
    }
    stages {
        stage('Build') { 
            steps { 
                sh 'make' 
            }
        }
        stage('Test'){
            steps {
                sh 'make check'
                junit 'reports/**/*.xml' 
            }
        }
        stage('Deploy') {
            steps {
                sh 'make publish'
            }
        }
    }
}
```

## jenkins分布式构建

### 准备kubernetes

#### kubernetes架构

属于cs架构(master/node), 组件图如下

Master：主要由API Server、Controller-Manager和Scheduler这3个组件，以及一个用于集群状态存储的Etcd存储服务组成，它们构成整个集群的控制平面；

Node：主要包含Kubelet、Kube Proxy及容器运行时（Docker是最为常用的实现）3个组件，它们承载运行各类应用容器

![image-20210514135257924](http://myapp.img.mykernel.cn/image-20210514135257924.png)

#### k8s api资源

 Kubernetes内建的API资源类型大体可分为工作负载、发现与负载均衡、配置与存储、集群、元数据几个类别

![image-20210514135343701](http://myapp.img.mykernel.cn/image-20210514135343701.png)

#### pod存储 

存储卷用于为Pod提供独立于容器生命周期的数据存储及共享机制

每个工作节点基于本地内存或目录向Pod提供存储空间，也能够使用借助于驱动程序挂载的网络文件系统或附加的块设备，例如使用挂载至本地某路径上的NFS文件系统等；Kubernetes系统具体支持的存储卷类型要取决于存储卷插件的内置定义

Kubernetes系统内置了多种类型的存储卷插件，因而能够直接支持多种类型存储供给者（即存储服务方或存储预配），例如CephFS、NFS、RBD、iscsi和vsphereVolume等

![image-20210514135446574](http://myapp.img.mykernel.cn/image-20210514135446574.png)

#### pv和pvc

PV（PersistentVolume）与PVC（PersistentVolumeClaim）就是在用户与存储服务之间添加的一个中间层

管理员事先根据PV支持的存储插件及适配的存储供给（目标存储系统）细节定义好可以支撑存储卷的底层存储空间而后由用户通过PVC声明要使用的存储特性来绑定符合条件的最佳PV定义存储卷，从而使得存储系统的使用和管理职能的解耦，大大简化了用户使用存储的方式

PV和PVC的生命周期由Controller Manager中专用的PV控制器（PV Controller）独立管理，这种机制的存储卷不再依附并受限于Pod对象的生命周期，从而实现了用户和集群管理员的职责相分离

![image-20210514135614984](http://myapp.img.mykernel.cn/image-20210514135614984.png)

#### 控制器  

负责把API Server上存储的对象定义实例化到集群上的程序就是控制器

通过控制循环（control loop，也可称为控制回路）持续监视集群上由其负责管控的对象实例的实际状态，在因故障、更新或其他原因导致实际状态发生变化而不同于期望状态时，通过实时运行相应的程序代码尝试让对象的真实状态向期望状态迁移和逼近

![image-20210514140153345](http://myapp.img.mykernel.cn/image-20210514140153345.png)

![image-20210514140218030](http://myapp.img.mykernel.cn/image-20210514140218030.png)

标签择器：匹配并关联Pod对象，并据此完成受其管控的Pod对象的计数；

期望的副本数：期望在集群中精确运行着的受控Pod对象数量；

Pod模板：用于新建Pod对象使用的模板资源；

#### 资源和服务发现

标签匹配 pod

pod资源：配置有特定的标签及标签值（KV格式的数据）

Service资源：基于“标签选择器”过滤出符合条件的Pod标签

在该组Pod前端提供一个虚拟的组件作为该组Pod的访问入口，该组件

配置有IP地址及指定的服务端口

其地址位于一个专用的，称为“Cluster Network”的网络中，也可称为Service Network

支持负载均衡机制





![image-20210514140427376](http://myapp.img.mykernel.cn/image-20210514140427376.png)

##### clusterip

默认类型仅集群可达

![image-20210514140736486](http://myapp.img.mykernel.cn/image-20210514140736486.png)

##### nodeport

除了clusterip之外，还有节点端口

![image-20210514140825019](http://myapp.img.mykernel.cn/image-20210514140825019.png)

外部请求nodeY, 再到pod 1 必然有延迟， 确保pod1回复的是ndoeY,  nodeY转发时还需要SNAT

##### loadbalancer

依赖于部署在IaaS云计算服务之上并且能够调用其API接口创建软件负载均衡器的Kubernetes集群环境

建构在NodePort类型的基础上，通过云服务商（cloud provider）提供的软负载均衡器将服务暴露到集群外部，因此它也会具有NodePort和ClusterIP

![image-20210514141054815](http://myapp.img.mykernel.cn/image-20210514141054815.png)

#### 控制器更新策略

控制器编排的需要长期运行的应用，更新、回滚、扩缩容也是编排操作的核心，这类pod的变动会影响透过service来访问应用服务的客户端。显示生产环境中的应用编排过程通过不能影响或过度影响用户体验，达到该类目标也是应用程序的核心功能之一，事实上deployment允许用户自定义更新策略自定义应用升级过程。滚动更新

![image-20210514141436816](http://myapp.img.mykernel.cn/image-20210514141436816.png)

#### ingress

Ingress只是一组路由规则的清单文件

![image-20210514141606190](http://myapp.img.mykernel.cn/image-20210514141606190.png)

真正实现流量穿透，需要将路由规则配置在监听socket的应用之上。例如：Nginx、Envoy、HAProxy、Vulcand和Traefik等，通过解析路由规则转换为应用的配置，也叫Ingress控制器

![image-20210514141715690](http://myapp.img.mykernel.cn/image-20210514141715690.png)

ingress控制器如何接入流量 ？deploy: nodePort/ LoadBalaner. ds: hostPort, hostNetwork

![image-20210514141909123](http://myapp.img.mykernel.cn/image-20210514141909123.png)

#### kubernetes网络基础

3个网络，分别用于 节点、pod、service，3个网络在节点之上交汇，由节点的内核中的路由组件及iptables/netfilter和ipvs等完成网络间的报文转发。

![image-20210514142120565](http://myapp.img.mykernel.cn/image-20210514142120565.png)

#### kubernetes通信模型

k8s服务：api server和应用服务，客户端来自pod, 要么来自集群外。通过集群边界进入内部，叫：南北向流量。发现在pod与service网络之上叫东西向流量。

南北向流量 ： nodeport, loadbalancer。 ingress资源。

![image-20210514142351063](http://myapp.img.mykernel.cn/image-20210514142351063.png)

pod的容器间通信：共享network ns, 走内核协议栈

pod间通信：cni委托给第3方插件。cni专用为容器配置网络子系统，目前由RTK、docker、k8s、openshift和mesos相关容器运行环境所支持。

pod与service，iptables/ipvs

service或Pod与集群外：

- pod本地：hostPort, hostnetwork
- service通过特定IP：externalip
- service: nodeport, loadbalancer

#### cni基础

遵循cni规范的网络插件是可执行程序文件，由k8s调用，负责向容器的network ns插入一个网络接口，并在宿主机上配置虚拟网络，分配ip。因此常被称为网络插件。NetPlugin.

为了满足分布式pod通信，所有pod同一个平面，NetPlugin通常实现方案有：overlay(叠加网络)和Underlay network(承载网络)。

叠加网络：借助vxlan, udp, ipip和gre等隧道协议，通过隧道协议报文封装pod间的通信报文来构建虚拟网络。

承载网络：通过使用Direct routing直接路由，在pod与各子网间路由pod的IP报文，也使用bridge, maxvlan, ipvlan技术直接将容器暴露至外部网络中。

相对于承载网络，存在额外隧道报文封装，存在一定开销，但是用户需要跨越多个L2,L3层逻辑子网，此时只能借助隧道协议实现。

#### 常见cni插件

flannel: coreos创立，支持vxlan, udp, vxlan direct routing, host-gw，支持多种 不同虚拟网络模型。最简单，最受欢迎。

calico: 每机器运行一个vRouter基于BGP路由协议在节点之间路由数据包，calico支持网络策略，借助iptables实现访问控制。calico也支持ipip, vxlan模型的叠加网络。

#### 部署kubernetes

单机：minikube, 单master

部署工具：kubeadm, kops, kind, ..

### jenkins配置kubernetes

![image-20210425115431525](http://myapp.img.mykernel.cn/image-20210425115431525.png)

![image-20210514134435701](http://myapp.img.mykernel.cn/image-20210514134435701.png)

配置agent的端口

![image-20210514134520804](http://myapp.img.mykernel.cn/image-20210514134520804.png)

# 敏捷开发

192.168.1.22/songliangcheng/configcenter.git

## jenkins参数化构建

```bash
pipeline {
    agent any    
    environment {
        PROTOCOL="https://"
        REPOSITORY="registry.cn-hangzhou.aliyuncs.com/grasp_base_images"
        REPO_BASE_NAME="cmswebtest"
        GITCREDENTIAL="gitlab-documents-user-pass"
        GITHTTPURL="https://gitlab.mykernel.cn/songliangcheng/test-pipeline.git"
    }
    parameters {
        gitParameter name: 'BRANCH_TAG', 
                     type: 'PT_BRANCH_TAG',
                     branchFilter: 'origin/(.*)',
                     defaultValue: 'master',
                     selectedValue: 'DEFAULT',
                     sortMode: 'DESCENDING_SMART',
           description: 'Select your branch or tag.'
        // jenkins参数化构建
        choice(name: 'SonarQube', choices: ['False','True'],description: '')         
    }

    stages {
        stage("拉代码") {
          steps {
                script {
                      sh 'echo 参数化构建'
                      checkout([$class: 'GitSCM', 
                                branches: [[name: "${params.BRANCH_TAG}"]], 
                                doGenerateSubmoduleConfigurations: false, 
                                extensions: [], 
                                gitTool: 'Default', 
                                submoduleCfg: [], 
                                userRemoteConfigs: [[url: "${GITHTTPURL}",credentialsId: "${GITCREDENTIAL}",]]
                              ])
                      BRANCH_TAG = "${params.BRANCH_TAG}"
                }
          }
        }
    }
}
```

## jenkins配置GitLab触发

```bash
pipeline {
    // agent any    
    triggers {
      gitlab(triggerOnPush: true, triggerOnMergeRequest: true, branchFilterType: 'All', secretToken: '2a32f3c89c2d7eb9ca39eee4d1c2c569')
    }
    agent any 
    // jenkins正常指令
    environment {
        PROTOCOL="https://"
        REPOSITORY="registry.cn-hangzhou.aliyuncs.com/grasp_base_images"
        REPO_BASE_NAME="cmswebtest"
        GITCREDENTIAL="gitlab-documents-user-pass"
    }

    // pipeline正常指令
    stages {
        stage("拉代码") {
          steps {
            script {
                    // https://blog.csdn.net/nklinsirui/article/details/100521145
                    // https://www.twblogs.net/a/5d6fd26bbd9eee5327ff4b5d
                      sh 'echo 触发式'
                      checkout changelog: true, poll: true, scm: [
                                $class: 'GitSCM',
                                branches: [[name: "origin/${env.gitlabSourceBranch}"]],
                                doGenerateSubmoduleConfigurations: false,
                                extensions: [[
                                  $class: 'PreBuildMerge',
                                  options: [
                                    fastForwardMode: 'FF',
                                    mergeRemote: 'origin',
                                    mergeStrategy: 'default',
                                    mergeTarget: "${env.gitlabTargetBranch}"
                                  ]
                                ]],
                                extensions: [[
                                  $class: 'UserIdentity', 
                                  email: "${env.gitlabUserEmail}", 
                                  name: "${env.gitlabUserName}"
                                ]],
                                submoduleCfg: [],
                                userRemoteConfigs: [[
                                  credentialsId: "${GITCREDENTIAL}",
                                  name: 'origin',
                                  url: "${env.gitlabSourceRepoHttpUrl}"
                                ]]
                              ]
                      BRANCH_TAG = "merge ${env.gitlabSourceBranch} to ${env.gitlabTargetBranch}"
                }
          }
        }


    }
}

```

![image-20210506101214336](http://myapp.img.mykernel.cn/image-20210506101214336.png)

## 钉钉告警

```bash
pipeline {
  // agent any    
  triggers {
    gitlab(triggerOnPush: true, triggerOnMergeRequest: true, branchFilterType: 'All', secretToken: '2a32f3c89c2d7eb9ca39eee4d1c2c569')
  }
    agent {
        // jenkins对接kubernets
        // kubernetes的pod模板，不需要在jenkins中二次定义. 必须有nodeSelector, 否则不会调度
        kubernetes {
          yaml """\
            apiVersion: v1
            kind: Pod
            metadata:
              namespace: jenkins
              labels:
                some-label: some-label-value
            spec:
              nodeSelector:
                kubernetes.io/os: linux
              serviceAccountName: jenkins-build-sa
              containers:
              - name: npm
                image:  registry.cn-hangzhou.aliyuncs.com/grasp_base_images/npm:latest
                command:
                - cat
                tty: true
              - name: docker
                image: docker:19.03.12
                command:
                - cat
                tty: true
                securityContext:
                  privileged: true
                volumeMounts:
                  - name: dind-storage
                    mountPath: /var/run/docker.sock
              volumes:
              - name: dind-storage
                hostPath:
                  path: /var/run/docker.sock
            """.stripIndent()
        }
    }
    // jenkins正常指令
    environment {
        PROTOCOL="https://"
        REPOSITORY="registry.cn-hangzhou.aliyuncs.com/grasp_base_images"
        REPO_BASE_NAME="cmswebtest"
        GITCREDENTIAL="gitlab-documents-user-pass"

        // url
        GITHTTPURL="https://gitlab.mykernel.cn/songliangcheng/test-pipeline.git"
    }

    // jenkins的git参数化构建
    parameters {
        gitParameter name: 'BRANCH_TAG', 
                     type: 'PT_BRANCH_TAG',
                     branchFilter: 'origin/(.*)',
                     defaultValue: 'master',
                     selectedValue: 'DEFAULT',
                     sortMode: 'DESCENDING_SMART',
           description: 'Select your branch or tag.'
        // jenkins参数化构建
        choice(name: 'SonarQube', choices: ['False','True'],description: '')         
    }

    // pipeline正常指令
    stages {
        // pipeline正常指令，一个stages里面多个stage, 每个stage里面1个steps {}
        stage("拉代码") {
          // pipeline正常指令，每个stage里面1个steps {}, 1个steps{} 里面可以有多个指令
          steps {

  // 拉代码 参数化构建：master gitlab触发传递来的：origin/dev gitlab 用户名 slcnx gitlab merge的用户 songliangcheng gitlab用户邮箱 2192383945@qq.com gitlab merge标题 本地 t odev gitlab merge描述 dev 合并至 master
                echo "拉代码 参数化构建：${params.BRANCH_TAG} gitlab触发传递来的：origin/${env.gitlabSourceBranch} gitlab 用户名 ${env.gitlabUserName} gitlab merge的用户 ${env.gitlabMergedByUser} gitlab用户邮箱 ${env.gitlabUserEmail} gitlab merge标题 ${env.gitlabMergeRequestTitle} gitlab merge描述 ${env.gitlabMergeRequestDescription}"
                
                script {
                    // https://blog.csdn.net/nklinsirui/article/details/100521145
                    // https://www.twblogs.net/a/5d6fd26bbd9eee5327ff4b5d
                    if (env.gitlabUserName) {
                        sh 'echo 触发式'
                        checkout changelog: true, poll: true, scm: [
                                  $class: 'GitSCM',
                                  branches: [[name: "origin/${env.gitlabSourceBranch}"]],
                                  doGenerateSubmoduleConfigurations: false,
                                  extensions: [[
                                    $class: 'PreBuildMerge',
                                    options: [
                                      fastForwardMode: 'FF',
                                      mergeRemote: 'origin',
                                      mergeStrategy: 'default',
                                      mergeTarget: "${env.gitlabTargetBranch}"
                                    ]
                                  ]],
                                  extensions: [[
                                    $class: 'UserIdentity', 
                                    email: "${env.gitlabUserEmail}", 
                                    name: "${env.gitlabUserName}"
                                  ]],
                                  submoduleCfg: [],
                                  userRemoteConfigs: [[
                                    credentialsId: "${GITCREDENTIAL}",
                                    name: 'origin',
                                    url: "${env.gitlabSourceRepoHttpUrl}"
                                  ]]
                                ]
                        BRANCH_TAG = "merge ${env.gitlabSourceBranch} to ${env.gitlabTargetBranch}"
                    } else {
                        sh 'echo 参数化构建'
                        checkout([$class: 'GitSCM', 
                                  branches: [[name: "${params.BRANCH_TAG}"]], 
                                  doGenerateSubmoduleConfigurations: false, 
                                  extensions: [], 
                                  gitTool: 'Default', 
                                  submoduleCfg: [], 
                                  userRemoteConfigs: [[url: "${GITHTTPURL}",credentialsId: "${GITCREDENTIAL}",]]
                                ])
                        BRANCH_TAG = "${params.BRANCH_TAG}"
                      }

                }



          }
        }
        stage('发布前报警') {
            steps {
                 container('npm') {

                    // pipeline使用构建者名称
                    // 构建用户信息：id, 名，email
                    wrap([$class: 'BuildUser']) {
                       script {
                           BUILD_USER_ID = "${env.BUILD_USER_ID}"
                           BUILD_USER = "${env.BUILD_USER}"
                           BUILD_USER_EMAIL = "${env.BUILD_USER_EMAIL}"
                       }
                     }


                    // 钉钉的告警 https://plugins.jenkins.io/dingding-notifications/
                    script {
                        DCURTIME = sh(script: "date +%F-%H:%M:%S", returnStdout: true).trim()
                    }
                    dingtalk (
                            robot: 'aec16636-91cf-45a3-8083-b9376b4fb741',
                            type: 'MARKDOWN',
                            title: '你有新的消息，请注意查收',
                            text: [
                                "# ${env.JOB_NAME} CI ${currentBuild.durationString}",
                                "---",
                                "- 报告时间: ${DCURTIME}",
                                "- 任务日志: [${currentBuild.number} 构建日志](${currentBuild.absoluteUrl}console)",
                                "- 执行人:  ${BUILD_USER}",
                                "- 执行任务节点:  ${env.NODE_NAME}",
                                "- 你选择的分支：${BRANCH_TAG}"
                            ],
                            at: [
                              '17502890627'
                            ]
                        )
                }
            }
        }



    }

    post {
        always {
            container('npm') {
                // 构建用户信息：id, 名，email
                wrap([$class: 'BuildUser']) {
                   script {
                       BUILD_USER_ID = "${env.BUILD_USER_ID}"
                       BUILD_USER = "${env.BUILD_USER}"
                       BUILD_USER_EMAIL = "${env.BUILD_USER_EMAIL}"
                   }
                 }
              // 钉钉的告警
                    script {
                        DCURTIME = sh(script: "date +%F-%H:%M:%S", returnStdout: true).trim()
                    }
               dingtalk (
                        robot: 'aec16636-91cf-45a3-8083-b9376b4fb741',
                        type: 'MARKDOWN',
                        title: '你有新的消息，请注意查收',
                        text: [
                            "# ${env.JOB_NAME} ${currentBuild.currentResult}",
                            "---",
                            "- 报告时间: ${DCURTIME}",
                            "- 任务日志: [${currentBuild.number} 构建日志](${currentBuild.absoluteUrl}console)",
                            "- 持续时间:  ${currentBuild.durationString}",
                            "- 执行人:  ${env.gitlabUserName}",
                            "- 执行任务节点:  ${env.NODE_NAME}",
                            "- 你选择的分支：${BRANCH_TAG}"
                        ],
                        at: [
                          '17502890627'
                        ]
                    )
            }
        }

    }

}

```

## 质量扫描

```bash
pipeline {
	agent any
    parameters {
        choice(name: 'SonarQube', choices: ['False','True'],description: '')         
    }
	stages {
		stage('code quality') {
			// 与参数化构建一起使用，只有sonarqube环境变量为True才构建
            when {
                 expression { return env.sonarqube == "True" }
              }
            steps {
                container('sonarqube') {
                    sh 'sonar-scanner' +
                    '-Dsonar.sources=dist ' +
                    "-Dsonar.projectKey=${env.JOB_NAME}" +
                    "-Dsonar.projectName=${env.JOB_NAME}"
                }
            }
		}
	}
}
```

## sonarqube api

```bash
https://docs.sonarqube.org/6.7/MetricDefinitions.html
https://docs.sonarqube.org/latest/user-guide/metric-definitions/



echo jenkins获取bug
curl -u admin:12312321312321 -s 'http://1.1.1.1:9000/api/issues/search?projectKeys=test-pipeline1&types=BUG' | jq -r .total

 curl -u admin:12312321312321 -s 'http://1.1.1.1:9000/api/measures/component?component=test-pipeline1&metricKeys=bugs'  | jq -r .component.measures[].value

 curl -u admin:12312321312321 -s 'http://1.1.1.1:9000/api/measures/component?component=test-pipeline1&metricKeys=new_bugs'  | jq -r .component.measures[].value

echo 漏洞
curl -u admin:12312321312321 -s 'http://1.1.1.1:9000/api/issues/search?projectKeys=test-pipeline1&types=VULNERABILITY' | jq -r .total


echo code smell
curl -u admin:12312321312321 -s 'http://1.1.1.1:9000/api/issues/search?projectKeys=test-pipeline1&types=CODE_SMELL' | jq -r .total


echo 获取扫描结果 
curl -u admin:12312321312321 -s 'http://1.1.1.1:9000/api/ce/task?id=AXlBQYvP1iimO-vzeX2S' | jq -r .task.status


echo 获取代码重复代码率
curl -u admin:12312321312321 -s 'http://1.1.1.1:9000/api/measures/component?component=test-pipeline1&metricKeys=duplicated_lines_density' | jq -r .component.measures[].value


echo 代码总行数
curl -u admin:12312321312321 -s 'http://1.1.1.1:9000/api/measures/component?component=test-pipeline1&metricKeys=ncloc'  | jq -r .component.measures[].value


curl -u admin:12312321312321 -s 'http://1.1.1.1:9000/api/measures/component?component=test-pipeline1&metricKeys=complexity'
curl -u admin:12312321312321 -s 'http://1.1.1.1:9000/api/measures/component?component=test-pipeline1&metricKeys=violations'


curl -u admin:12312321312321 -s 'http://1.1.1.1:9000/api/measures/component?component=test-pipeline1&metricKeys=alert_status'



代码安全：
# 新代码的代码深层问题
curl -u admin:12312321312321 -s 'http://1.1.1.1:9000/api/measures/component?component=test-pipeline1&metricKeys=new_code_smells'  | jq -r .component.measures[].period.value
网址
http://1.1.1.1:9000/project/issues?id=test-pipeline1&resolved=false&sinceLeakPeriod=true&types=CODE_SMELL


# 新代码的漏洞
curl -u admin:12312321312321 -s 'http://1.1.1.1:9000/api/measures/component?component=test-pipeline1&metricKeys=new_vulnerabilities'  | jq -r .component.measures[].period.value
网址
http://1.1.1.1:9000/project/issues?id=test-pipeline1&resolved=false&sinceLeakPeriod=true&types=VULNERABILITY


可靠性
# 新代码的BUG
curl -u admin:12312321312321 -s 'http://1.1.1.1:9000/api/measures/component?component=test-pipeline1&metricKeys=new_bugs'  | jq -r .component.measures[].period.value
网址
http://1.1.1.1:9000/project/issues?id=test-pipeline1&resolved=false&sinceLeakPeriod=true&types=BUG

质量状态
curl -u admin:12312321312321 -s 'http://1.1.1.1:9000/api/measures/component?component=test-pipeline1&metricKeys=alert_status'  | jq -r .component.measures[].value
网址
http://1.1.1.1:9000/dashboard?id=test-pipeline1

质量未通过
curl -u token: -s 'http://1.1.1.1:9000/api/measures/component?component=test-pipeline1&metricKeys=quality_gate_details'  | jq -r .component.measures[].value
```



## 敏捷开发Jenkinsfile

### 有报警

```groovy
// AUTHOR: songliangcheng
// DESC：插件
// jenkins对接kubernets: https://plugins.jenkins.io/kubernetes/ , 文中搜索：Declarative Pipeline
// pipeline使用构建者名称：https://wiki.jenkins.io/display/JENKINS/Build+User+Vars+Plugin
// pipeline的 git参数化构建：https://plugins.jenkins.io/git-parameter/

// pipeline的gitlab自动触发：https://plugins.jenkins.io/gitlab-plugin/， 支持的变量：Defined variables，引用变量: ${env.gitlabSourceBranch}
    // 自由或pipeline在见面中定义。多分支pipeline可以yaml定义。
// pipeline钉钉告警：https://plugins.jenkins.io/dingding-notifications/
// docker piepline插件，docker相关的插件

// 示例：https://github.com/jenkinsci/pipeline-examples/tree/master/declarative-examples/simple-examples



pipeline {
  // agent any    
  triggers {
    // push事件，会触发，只对源分支为test的或目标分支为test触发。 开发本地合并，推送之后，源一定是test
    // merge打开请求的事件，在gitlab ci完成
    gitlab(triggerOnPush: true, triggerOnMergeRequest: false, branchFilterType: 'RegexBasedFilter', sourceBranchRegex: '.*test.*',targetBranchRegex: '.*test.*',secretToken: 'e1ed6d39b609d15f8ce27173675de080')
  }
    agent {
        // jenkins对接kubernets
        // kubernetes的pod模板，不需要在jenkins中二次定义. 必须有nodeSelector, 否则不会调度
        kubernetes {
          yaml """\
            apiVersion: v1
            kind: Pod
            metadata:
              namespace: jenkins
              labels:
                some-label: some-label-value
            spec:
              nodeSelector:
                kubernetes.io/os: linux
              serviceAccountName: jenkins-build-sa
              containers:
              
              - name: docker
                image: docker:19.03.12
                command:
                - cat
                tty: true
                env:
                - name: LANG
                  value: zh_CN.UTF-8
                - name: TZ
                  value: Asia/Shanghai
                securityContext:
                  privileged: true
                volumeMounts:
                  - name: dind-storage
                    mountPath: /var/run/docker.sock
              - name: sonar
                image: sonarsource/sonar-scanner-cli:4.6
                command:
                - cat
                tty: true
                env:
                - name: LANG
                  value: zh_CN.UTF-8
                - name: TZ
                  value: Asia/Shanghai
                volumeMounts:
                  - name: sonar-storage
                    mountPath:  /opt/sonar-scanner/.sonar/cache
              - name: jq
                image: registry.cn-hangzhou.aliyuncs.com/grasp_base_images/ubuntu-jq-curl:latest
                command:
                - cat
                tty: true
                env:
                - name: LANG
                  value: zh_CN.UTF-8
                - name: TZ
                  value: Asia/Shanghai
              - name: mysql
                image: registry.cn-hangzhou.aliyuncs.com/grasp_base_images/mysql:latest
                command:
                - cat
                tty: true
                env:
                - name: LANG
                  value: zh_CN.UTF-8
                - name: TZ
                  value: Asia/Shanghai
              volumes:
              - name: dind-storage
                hostPath:
                  path: /var/run/docker.sock
             
              - name: sonar-storage
                hostPath:
                  path: /opt/sonar-scanner/.sonar/cache
                  type: DirectoryOrCreate
            """.stripIndent()
        }
    }
    // jenkins正常指令
    environment {
        PROTOCOL="https://"
        REPOSITORY="registry.cn-hangzhou.aliyuncs.com/grasp_base_images"
        REPO_BASE_NAME="${gitlabSourceRepoName}"
        // git登陆 (参数化、非参数化均需要使用)
        GITCREDENTIAL="gitlab-documents-user-pass"

        // sonarqube 
        CRED = credentials('sonarqube') 
        // jenkins-self
        JENKINS_CRED = credentials('jenkins-self') 

        //mysql
        MYSQL_CRED = credentials('jenkins-to-mysql')

        // DINGDINGID = 'd1b9b632-cfbb-4ca9-966a-2c9534e14b59'
        DINGDINGID = 'aec16636-91cf-45a3-8083-b9376b4fb741'

        SONARURI = 'http://112.124.37.17:8599'
        SONARURL="${SONARURI}/dashboard?id=${env.JOB_NAME}"
    }


    // pipeline正常指令
    stages {
        // pipeline正常指令，一个stages里面多个stage, 每个stage里面1个steps {}
        stage("拉代码") {
          // pipeline正常指令，每个stage里面1个steps {}, 1个steps{} 里面可以有多个指令
          steps {
               // 构建用户信息：id, 名，email
                wrap([$class: 'BuildUser']) {
                   script {
                       BUILD_USER_ID = "${env.BUILD_USER_ID}"
                       BUILD_USER = "${env.BUILD_USER}"
                       BUILD_USER_EMAIL = "${env.BUILD_USER_EMAIL}"
                   }
                 }


  // 拉代码 参数化构建：master gitlab触发传递来的：origin/dev gitlab 用户名 slcnx gitlab merge的用户 songliangcheng gitlab用户邮箱 2192383945@qq.com gitlab merge标题 本地 t odev gitlab merge描述 dev 合并至 master
                echo "拉代码 参数化构建：${params.BRANCH_TAG} gitlab触发传递来的：origin/${env.gitlabSourceBranch} gitlab 用户名 ${env.gitlabUserName} gitlab merge的用户 ${env.gitlabMergedByUser} gitlab用户邮箱 ${env.gitlabUserEmail} gitlab merge标题 ${env.gitlabMergeRequestTitle} gitlab merge描述 ${env.gitlabMergeRequestDescription}"
                
                script {
                    sh 'printenv'
                    sh 'echo ============================================================'
                    // https://blog.csdn.net/nklinsirui/article/details/100521145
                    // https://www.twblogs.net/a/5d6fd26bbd9eee5327ff4b5d

                    sh 'echo 触发式'
                    checkout changelog: true, poll: true, scm: [
                              $class: 'GitSCM',
                              branches: [[name: "origin/${env.gitlabSourceBranch}"]],
                              doGenerateSubmoduleConfigurations: false,
                              extensions: [[
                                $class: 'PreBuildMerge',
                                options: [
                                  fastForwardMode: 'FF',
                                  mergeRemote: 'origin',
                                  mergeStrategy: 'default',
                                  mergeTarget: "${env.gitlabTargetBranch}"
                                ]
                              ]],
                              extensions: [[
                                $class: 'UserIdentity', 
                                email: "${env.gitlabUserEmail}", 
                                name: "${env.gitlabUserName}"
                              ]],
                              submoduleCfg: [],
                              userRemoteConfigs: [[
                                credentialsId: "${GITCREDENTIAL}",
                                name: 'origin',
                                url: "${env.gitlabSourceRepoHttpUrl}"
                              ]]
                            ]
                    BRANCH_TAG = "${env.gitlabSourceBranch}"

                    //本次提交信息
                    GIT_LAST_COMMIT= sh(script:"git rev-parse --short HEAD",returnStdout:true).trim()
                    GIT_LAST_COMMIT_MESSAGE=sh(script: "git  --no-pager show -s --format='%s'  ${GIT_LAST_COMMIT} | tr -d \"'\" ",returnStdout:true).trim()
                    BUILD_USER_EMAIL=sh(script: "git  --no-pager show -s --format='%s'  ${GIT_LAST_COMMIT} | tr -d \"'\" ",returnStdout:true).trim()
                    echo "提交信息 ${GIT_LAST_COMMIT_MESSAGE}  提交hash ${GIT_LAST_COMMIT} "
                }



          }
        }
        stage('发布前报警') {
            steps {
                 container('jq') {

                    // pipeline使用构建者名称
                    // 构建用户信息：id, 名，email
                    wrap([$class: 'BuildUser']) {
                       script {
                           BUILD_USER_ID = "${env.BUILD_USER_ID}"
                           BUILD_USER = "${env.BUILD_USER}"
                           BUILD_USER_EMAIL = "${env.BUILD_USER_EMAIL}"
                       }
                     }


                    // 钉钉的告警 https://plugins.jenkins.io/dingding-notifications/
                    script {
                        DCURTIME = sh(script: "date +'%Y年%m月%d日 %H时%M分%S秒'", returnStdout: true).trim()
                        
                        // 判断执行人
                        if (BUILD_USER == "SCM Change") {
                            // 如果构建人是SCM Change是触发构建
                            BUILD_USER ="${env.gitlabUserName}"
                        } 

                    }
                    dingtalk (
                            // robot: 'aec16636-91cf-45a3-8083-b9376b4fb741',
                            robot: "${DINGDINGID}",
                            type: 'MARKDOWN',
                            title: '你有新的消息，请注意查收',
                            text: [
                                " ${env.JOB_NAME} *开始*构建${BRANCH_TAG}分支 [${currentBuild.number}任务](${currentBuild.absoluteUrl}console) ${currentBuild.durationString}",
                                "---",
                                "- 执行人:  ${BUILD_USER}",
                                "- 本次构建的提交信息: ",
                                "> ${GIT_LAST_COMMIT_MESSAGE}",
                                "- git源码仓库位置: ",
                                "> ${gitlabSourceRepoName}",
                                "- [Jenkins job入口](https://jenkins.graspyun.com//job/${env.JOB_NAME}/)",
                                "- [Jenkins 发版位置](https://jenkins.graspyun.com/view/%E4%BA%91%E9%A1%B9%E7%9B%AE%E7%9A%84%E6%B5%8B%E8%AF%95%E5%92%8C%E7%81%B0%E5%BA%A6%E7%8E%AF%E5%A2%83/)",
                                "- [SonarQube代码质量入口](${SONARURL})",
                                "---",
                                "> ${DCURTIME}\n",
                                "> 如有疑问，请联系： 运维-宋亮成<songliangcheng@graspyun.com>"
                            ],
                            "atAll": true
                        )
                }

                container('mysql') {
                  script {
                      // 创建表的用户
                      // 分支
                      // 构建用户
                      // 构建邮箱
                      // 构建消息
                      LANG='zh_CN.UTF-8'
                      TZ='Asia/Shanghai'
                      NOWTIME  = sh(script: "date +'%F %H:%M'", returnStdout: true).trim()
                      CURTIME = sh(script: "date +%Y%m%d%H%M%S", returnStdout: true).trim()
                      sh script: "mysql -u${MYSQL_CRED_USR} -p${MYSQL_CRED_PSW} -P8600 -h112.124.37.17 -e \"insert into jenkins_webhook.jenkins(createtime,modifytime,creator,buildbranch,builduser,buildemail,buildcommit,node,taskurl,codescan,imageinfo,buildduration,githash,jobname,jobnum) values ('${NOWTIME}','${NOWTIME}','jenkins','${BRANCH_TAG}','${BUILD_USER}','${BUILD_USER_EMAIL}','${GIT_LAST_COMMIT_MESSAGE}','${env.NODE_NAME}','${currentBuild.absoluteUrl}console', '开始构建','开始构建','${currentBuild.durationString}','${GIT_LAST_COMMIT}','${env.JOB_NAME}','${currentBuild.number}')\"",returnStdout:true
                  }
                }

            }
        }


        stage('代码构建并打包镜像，并上传') {
            steps {
                container('docker') {
                    script {
                        // RESULT = sh(script: "basename ${BRANCH_TAG}", returnStdout: true).trim()
                        // docker piepline插件，docker相关的插件
                        docker.withRegistry("${PROTOCOL}${REPOSITORY}/${REPO_BASE_NAME}", 'docker-kubernetes') {
                            customImage = docker.build("${REPOSITORY}/${REPO_BASE_NAME}:${BRANCH_TAG}-${CURTIME}")
                            customImage.push("${BRANCH_TAG}-${CURTIME}")
                            customImage.push("${BRANCH_TAG}-latest")

                        } 

                        // 清理镜像
                        sh "docker rmi ${REPOSITORY}/${REPO_BASE_NAME}:${BRANCH_TAG}-${CURTIME}"
                        sh "docker rmi ${REPOSITORY}/${REPO_BASE_NAME}:${BRANCH_TAG}-latest"

                    }
                }
            }
        } 


      stage('镜像构建完成') {
        steps {

            container('jq') {

              // 钉钉的告警
                    script {
                        if (env.gitlabUserEmail) {
                          BUILD_USER_EMAIL="${env.gitlabUserEmail}"
                        }
                        DCURTIME = sh(script: "date +'%Y年%m月%d日 %H时%M分%S秒'", returnStdout: true).trim()
                        NOWTIME  = sh(script: "date +'%F %H:%M'", returnStdout: true).trim()
                    }
               dingtalk (
                        // robot: 'aec16636-91cf-45a3-8083-b9376b4fb741',
                        robot: "${DINGDINGID}",
                        type: 'MARKDOWN',
                        title: '你有新的消息，请注意查收',
                        text: [
                            " ${env.JOB_NAME} *结束*构建${BRANCH_TAG}分支 [${currentBuild.number}任务](${currentBuild.absoluteUrl}console) ${currentBuild.durationString}",
                            "```",
                            "本次构建结果: ${currentBuild.currentResult}",
                            "```",
                            "---",
                            "- 执行人:  ${BUILD_USER} ",
                            "- build邮箱: ${BUILD_USER_EMAIL}",
                            "- 本次构建的提交信息: ",
                            "> ${GIT_LAST_COMMIT_MESSAGE}",
                            "- git源码仓库位置: ",
                            "> ${gitlabSourceRepoName}",
                            "- [Jenkins job入口](https://jenkins.graspyun.com//job/${env.JOB_NAME}/)",
                            "- [Jenkins 发版位置](https://jenkins.graspyun.com/view/%E4%BA%91%E9%A1%B9%E7%9B%AE%E7%9A%84%E6%B5%8B%E8%AF%95%E5%92%8C%E7%81%B0%E5%BA%A6%E7%8E%AF%E5%A2%83/)",
                            "- [SonarQube代码质量入口](${SONARURL})",
                            "---",
                            "> ${DCURTIME}\n 对应镜像编号:${BRANCH_TAG}-${CURTIME}\n",
                            "> 如有疑问，请联系： 运维-宋亮成<songliangcheng@graspyun.com>"
                        ],
                        "atAll": true
                    )
            }

            container('mysql') {
              script {
                  // 创建表的用户
                  // 分支
                  // 构建用户
                  // 构建邮箱
                  // 构建消息
                  LANG='zh_CN.UTF-8'
                  TZ='Asia/Shanghai'
                  sh script: "mysql -u${MYSQL_CRED_USR} -p${MYSQL_CRED_PSW} -P8600 -h112.124.37.17 -e \"insert into jenkins_webhook.jenkins(createtime,modifytime,creator,buildbranch,builduser,buildemail,buildcommit,node,taskurl,imageinfo,currentResult,jobname,buildduration,githash,jobnum) values ('${NOWTIME}','${NOWTIME}','jenkins','${BRANCH_TAG}','${BUILD_USER}','${BUILD_USER_EMAIL}','${GIT_LAST_COMMIT_MESSAGE}','${env.NODE_NAME}','${currentBuild.absoluteUrl}console','${REPOSITORY}/${REPO_BASE_NAME}:${BRANCH_TAG}-${CURTIME}','${currentBuild.currentResult}','${env.JOB_NAME}','${currentBuild.durationString}','${GIT_LAST_COMMIT}','${currentBuild.number}')\"",returnStdout:true
              }
            }

        }



      }


        // https://stackoverflow.com/questions/45693418/sonarqube-quality-gate-not-sending-webhook-to-jenkins
        stage('QA Check') {
            steps {
                withCredentials([string(credentialsId: 'sonarqube', variable: 'SONAR_USER')]) {
                    container('sonar') {
                        sh "sonar-scanner -Dsonar.sources=./src -Dsonar.host.url=${SONARURI} -Dsonar.login=${SONAR_USER}  -Dsonar.projectKey=${env.JOB_NAME}  -Dsonar.projectName=${env.JOB_NAME}"
                        script {
                          SONAR_USER="${SONAR_USER}"
                        }
                    }
                }
            }
        }


        stage("Quality Gate") {
            steps {
                container('jq') {

                     script {
                            while(true){
                     
                                sh "sleep 2"
                                def url="https://jenkins.graspyun.com/job/${env.JOB_NAME.replaceAll('/','/job/')}/${currentBuild.number}/consoleText";
                                def sonarId = sh script: "wget -qO- --content-on-error --no-proxy --auth-no-challenge --http-user=${JENKINS_CRED_USR} --http-password=${JENKINS_CRED_PSW} '${url}'  | grep 'More about the report processing' | head -n1 ",returnStdout:true
                                sonarId = sonarId.substring(sonarId.indexOf("=")+1)
                                echo "sonarId ${sonarId}"


                                def sonarUrl = "${SONARURI}/api/ce/task?id=${sonarId}"

                                // 代码扫描执行的状态
                                def sonarStatus = sh script: "wget -qO- --content-on-error --no-proxy --auth-no-challenge --http-user=${SONAR_USER}  --http-password='' '${sonarUrl}' | jq -r '.task' | jq -r '.status' ",returnStdout:true
                                echo "Sonar status ... ${sonarStatus}"
                                if(sonarStatus.trim() == "SUCCESS"){
                                    sonarcodescanresult = 'OK'
                                    echo "BREAK";
                                } else if(sonarStatus.trim() == "FAILED"){
                                    echo "FAILED"
                                    sonarcodescanresult = 'FAILED'
                                } else if(sonarStatus.trim() == "IN_PROGRESS") {
                                  echo '继续' 
                                  continue
                                }
                                echo "代码扫描结果  ${sonarcodescanresult}"
                                // 代码质量状态
                                def sonarQualityGateUrl = "${SONARURI}/api/measures/component?component=${env.JOB_NAME}&metricKeys=alert_status"
                                def sonarQualityGateStatus = sh script: "wget -qO- --content-on-error --no-proxy --auth-no-challenge --http-user=${SONAR_USER} --http-password='' '${sonarQualityGateUrl}' | jq -r .component.measures[].value ",returnStdout:true
                                echo "sonar Quality Gate Status ... ${sonarQualityGateStatus}"
                                if(sonarQualityGateStatus.trim() == "OK"){
                                    echo "BREAK";
                                    sonarresult = 'OK'
                                    break;
                                }
                                if(sonarQualityGateStatus.trim() == "ERROR"){
                                    echo "ERROR"
                                    sonarresult = 'ERROR'
                                    break;
                                }
                            }
                        }
                }


                }
            }

    }

    post {
        always {
            container('jq') {

              // 钉钉的告警
                    script {
                        if (env.gitlabUserEmail) {
                          BUILD_USER_EMAIL="${env.gitlabUserEmail}"
                        }
                        DCURTIME = sh(script: "date +'%Y年%m月%d日 %H时%M分%S秒'", returnStdout: true).trim()
                        NOWTIME  = sh(script: "date +'%F %H:%M'", returnStdout: true).trim()
                    }
               dingtalk (
                        // robot: 'aec16636-91cf-45a3-8083-b9376b4fb741',
                        robot: "${DINGDINGID}",
                        type: 'MARKDOWN',
                        title: '你有新的消息，请注意查收',
                        text: [
                            " ${env.JOB_NAME} *结束*构建${BRANCH_TAG}分支 [${currentBuild.number}任务](${currentBuild.absoluteUrl}console) ${currentBuild.durationString}",
                            "```",
                            "执行扫描结果：${sonarcodescanresult}",
                            "代码质量结果：${sonarresult}",
                            "本次构建结果: ${currentBuild.currentResult}",
                            "```",
                            "---",
                            "- 执行人:  ${BUILD_USER} ",
                            "- build邮箱: ${BUILD_USER_EMAIL}",
                            "- 本次构建的提交信息: ",
                            "> ${GIT_LAST_COMMIT_MESSAGE}",
                            "- git源码仓库位置: ",
                            "> ${gitlabSourceRepoName}",
                            "- [SonarQube 本次构建代码质量入口](${SONARTASKURL})",
                            "---",
                            "> ${DCURTIME}\n 对应镜像编号:${BRANCH_TAG}-${CURTIME}\n",
                            "> 如有疑问，请联系： 运维-宋亮成<songliangcheng@graspyun.com>"
                        ],
                        "atAll": true
                    )
            }

            container('mysql') {
              script {
                  // 创建表的用户
                  // 分支
                  // 构建用户
                  // 构建邮箱
                  // 构建消息
                  LANG='zh_CN.UTF-8'
                  TZ='Asia/Shanghai'
                  sh script: "mysql -u${MYSQL_CRED_USR} -p${MYSQL_CRED_PSW} -P8600 -h112.124.37.17 -e \"insert into jenkins_webhook.jenkins(createtime,modifytime,creator,buildbranch,builduser,buildemail,buildcommit,node,taskurl,codescan,imageinfo,sonarcodescanresult,sonarresult,currentResult,jobname,buildduration,githash,jobnum) values ('${NOWTIME}','${NOWTIME}','jenkins','${BRANCH_TAG}','${BUILD_USER}','${BUILD_USER_EMAIL}','${GIT_LAST_COMMIT_MESSAGE}','${env.NODE_NAME}','${currentBuild.absoluteUrl}console', '${SONARTASKURL}','${REPOSITORY}/${REPO_BASE_NAME}:${BRANCH_TAG}-${CURTIME}','${sonarcodescanresult}','${sonarresult}','${currentBuild.currentResult}','${env.JOB_NAME}','${currentBuild.durationString}','${GIT_LAST_COMMIT}','${currentBuild.number}')\"",returnStdout:true
              }
            }

        }

    }

}


// CREATE TABLE `jenkins` (
//   `id` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '与业务无关的id',
//   `createtime` datetime NOT NULL COMMENT '创建时间',
//   `modifytime` datetime NOT NULL DEFAULT '0000-00-00 00:00:00' ON UPDATE current_timestamp() COMMENT '修改时间',
//   `creator` varchar(255) DEFAULT NULL COMMENT '创建者',
//   `buildbranch` varchar(255) DEFAULT NULL COMMENT '构建分支',
//   `builduser` varchar(255) DEFAULT NULL COMMENT '构建者用户',
//   `buildemail` varchar(255) DEFAULT NULL COMMENT '构建者邮箱',
//   `buildcommit` text DEFAULT NULL COMMENT '提交信息',
//   `node` varchar(255) DEFAULT '' COMMENT '执行节点',
//   `taskurl` varchar(255) DEFAULT NULL COMMENT '任务url',
//   `codescan` varchar(255) DEFAULT NULL COMMENT '代码质量url',
//   `imageinfo` varchar(255) DEFAULT NULL COMMENT '镜像信息',
//   `sonarcodescanresult` varchar(255) DEFAULT NULL COMMENT '执行sonar扫描的结果，成功或失败',
//   `sonarresult` varchar(255) DEFAULT NULL COMMENT '代码质量，sonarqube质量都达标时 成功',
//   `currentResult` varchar(255) DEFAULT NULL COMMENT 'Jenkins构建结果 ',
//   `jobname` varchar(255) DEFAULT NULL COMMENT 'jenkins构建的job名',
//   `buildduration` varchar(255) DEFAULT NULL COMMENT 'jenkins构建时长',
//   `githash` varchar(255) DEFAULT NULL COMMENT 'git提交的hash',
//   `jobnum` int(11) DEFAULT NULL COMMENT 'job的构建号',
//   PRIMARY KEY (`id`)
// ) ENGINE=InnoDB AUTO_INCREMENT=47 DEFAULT CHARSET=utf8mb4;

```

### 不报警

开发人员觉得烦

```groovy
// AUTHOR: songliangcheng
// DESC：插件
// jenkins对接kubernets: https://plugins.jenkins.io/kubernetes/ , 文中搜索：Declarative Pipeline
// pipeline使用构建者名称：https://wiki.jenkins.io/display/JENKINS/Build+User+Vars+Plugin
// pipeline的 git参数化构建：https://plugins.jenkins.io/git-parameter/

// pipeline的gitlab自动触发：https://plugins.jenkins.io/gitlab-plugin/， 支持的变量：Defined variables，引用变量: ${env.gitlabSourceBranch}
    // 自由或pipeline在见面中定义。多分支pipeline可以yaml定义。
// pipeline钉钉告警：https://plugins.jenkins.io/dingding-notifications/
// docker piepline插件，docker相关的插件

// 示例：https://github.com/jenkinsci/pipeline-examples/tree/master/declarative-examples/simple-examples



pipeline {
  // agent any    
  triggers {
    // push事件，会触发，只对源分支为test的或目标分支为test触发。 开发本地合并，推送之后，源一定是test
    // merge打开请求的事件，在gitlab ci完成
    gitlab(triggerOnPush: true, triggerOnMergeRequest: false, branchFilterType: 'RegexBasedFilter', sourceBranchRegex: '.*test.*',targetBranchRegex: '.*test.*',secretToken: 'e1ed6d39b609d15f8ce27173675de080')
  }
    agent {
        // jenkins对接kubernets
        // kubernetes的pod模板，不需要在jenkins中二次定义. 必须有nodeSelector, 否则不会调度
        kubernetes {
          yaml """\
            apiVersion: v1
            kind: Pod
            metadata:
              namespace: jenkins
              labels:
                some-label: some-label-value
            spec:
              nodeSelector:
                kubernetes.io/os: linux
              serviceAccountName: jenkins-build-sa
              containers:
              
              - name: docker
                image: docker:19.03.12
                command:
                - cat
                tty: true
                env:
                - name: LANG
                  value: zh_CN.UTF-8
                - name: TZ
                  value: Asia/Shanghai
                securityContext:
                  privileged: true
                volumeMounts:
                  - name: dind-storage
                    mountPath: /var/run/docker.sock
              - name: sonar
                image: sonarsource/sonar-scanner-cli:4.6
                command:
                - cat
                tty: true
                env:
                - name: LANG
                  value: zh_CN.UTF-8
                - name: TZ
                  value: Asia/Shanghai
                volumeMounts:
                  - name: sonar-storage
                    mountPath:  /opt/sonar-scanner/.sonar/cache
              - name: jq
                image: registry.cn-hangzhou.aliyuncs.com/grasp_base_images/ubuntu-jq-curl:latest
                command:
                - cat
                tty: true
                env:
                - name: LANG
                  value: zh_CN.UTF-8
                - name: TZ
                  value: Asia/Shanghai
              - name: mysql
                image: registry.cn-hangzhou.aliyuncs.com/grasp_base_images/mysql:latest
                command:
                - cat
                tty: true
                env:
                - name: LANG
                  value: zh_CN.UTF-8
                - name: TZ
                  value: Asia/Shanghai
              volumes:
              - name: dind-storage
                hostPath:
                  path: /var/run/docker.sock
             
              - name: sonar-storage
                hostPath:
                  path: /opt/sonar-scanner/.sonar/cache
                  type: DirectoryOrCreate
            """.stripIndent()
        }
    }
    // jenkins正常指令
    environment {
        PROTOCOL="https://"
        REPOSITORY="registry.cn-hangzhou.aliyuncs.com/grasp_base_images"
        REPO_BASE_NAME="${gitlabSourceRepoName}"
        // git登陆 (参数化、非参数化均需要使用)
        GITCREDENTIAL="gitlab-documents-user-pass"

        // sonarqube 
        CRED = credentials('sonarqube') 
        // jenkins-self
        JENKINS_CRED = credentials('jenkins-self') 

        //mysql
        MYSQL_CRED = credentials('jenkins-to-mysql')

        DINGDINGID = 'd1b9b632-cfbb-4ca9-966a-2c9534e14b59'
        // DINGDINGID = 'aec16636-91cf-45a3-8083-b9376b4fb741'

    }


    // pipeline正常指令
    stages {
        // pipeline正常指令，一个stages里面多个stage, 每个stage里面1个steps {}
        stage("拉代码") {
          // pipeline正常指令，每个stage里面1个steps {}, 1个steps{} 里面可以有多个指令
          steps {
              container('mysql') {
                script {
                CURTIME = sh(script: "date +%Y%m%d%H%M%S", returnStdout: true).trim()
                }
              }
               // 构建用户信息：id, 名，email
                wrap([$class: 'BuildUser']) {
                   script {
                       BUILD_USER_ID = "${env.BUILD_USER_ID}"
                       BUILD_USER = "${env.BUILD_USER}"
                       BUILD_USER_EMAIL = "${env.BUILD_USER_EMAIL}"
                   }
                 }


  // 拉代码 参数化构建：master gitlab触发传递来的：origin/dev gitlab 用户名 slcnx gitlab merge的用户 songliangcheng gitlab用户邮箱 2192383945@qq.com gitlab merge标题 本地 t odev gitlab merge描述 dev 合并至 master
                echo "拉代码 参数化构建：${params.BRANCH_TAG} gitlab触发传递来的：origin/${env.gitlabSourceBranch} gitlab 用户名 ${env.gitlabUserName} gitlab merge的用户 ${env.gitlabMergedByUser} gitlab用户邮箱 ${env.gitlabUserEmail} gitlab merge标题 ${env.gitlabMergeRequestTitle} gitlab merge描述 ${env.gitlabMergeRequestDescription}"
                
                script {

                        // 判断执行人
                        if (BUILD_USER == "SCM Change") {
                            // 如果构建人是SCM Change是触发构建
                            BUILD_USER ="${env.gitlabUserName}"
                        } 
        
                    sh 'printenv'
                    sh 'echo ============================================================'
                    // https://blog.csdn.net/nklinsirui/article/details/100521145
                    // https://www.twblogs.net/a/5d6fd26bbd9eee5327ff4b5d

                    sh 'echo 触发式'
                    checkout changelog: true, poll: true, scm: [
                              $class: 'GitSCM',
                              branches: [[name: "origin/${env.gitlabSourceBranch}"]],
                              doGenerateSubmoduleConfigurations: false,
                              extensions: [[
                                $class: 'PreBuildMerge',
                                options: [
                                  fastForwardMode: 'FF',
                                  mergeRemote: 'origin',
                                  mergeStrategy: 'default',
                                  mergeTarget: "${env.gitlabTargetBranch}"
                                ]
                              ]],
                              extensions: [[
                                $class: 'UserIdentity', 
                                email: "${env.gitlabUserEmail}", 
                                name: "${env.gitlabUserName}"
                              ]],
                              submoduleCfg: [],
                              userRemoteConfigs: [[
                                credentialsId: "${GITCREDENTIAL}",
                                name: 'origin',
                                url: "${env.gitlabSourceRepoHttpUrl}"
                              ]]
                            ]
                    BRANCH_TAG = "${env.gitlabSourceBranch}"

                    //本次提交信息
                    GIT_LAST_COMMIT= sh(script:"git rev-parse --short HEAD",returnStdout:true).trim()
                    GIT_LAST_COMMIT_MESSAGE=sh(script: "git  --no-pager show -s --format='%s'  ${GIT_LAST_COMMIT} | tr -d \"'\" ",returnStdout:true).trim()
                    BUILD_USER_EMAIL=sh(script: "git  --no-pager show -s --format='%s'  ${GIT_LAST_COMMIT} | tr -d \"'\" ",returnStdout:true).trim()
                    echo "提交信息 ${GIT_LAST_COMMIT_MESSAGE}  提交hash ${GIT_LAST_COMMIT} "
                }



          }
        }
        // stage('发布前报警') {
        //     steps {
        //          container('jq') {

        //             // pipeline使用构建者名称
        //             // 构建用户信息：id, 名，email
        //             wrap([$class: 'BuildUser']) {
        //                script {
        //                    BUILD_USER_ID = "${env.BUILD_USER_ID}"
        //                    BUILD_USER = "${env.BUILD_USER}"
        //                    BUILD_USER_EMAIL = "${env.BUILD_USER_EMAIL}"
        //                }
        //              }


        //             // 钉钉的告警 https://plugins.jenkins.io/dingding-notifications/
        //             script {
        //                 DCURTIME = sh(script: "date +'%Y年%m月%d日 %H时%M分%S秒'", returnStdout: true).trim()
                        
        //                 // 判断执行人
        //                 if (BUILD_USER == "SCM Change") {
        //                     // 如果构建人是SCM Change是触发构建
        //                     BUILD_USER ="${env.gitlabUserName}"
        //                 } 

        //             }
        //             dingtalk (
        //                     // robot: 'aec16636-91cf-45a3-8083-b9376b4fb741',
        //                     robot: "${DINGDINGID}",
        //                     type: 'MARKDOWN',
        //                     title: '你有新的消息，请注意查收',
        //                     text: [
        //                         " ${env.JOB_NAME} *开始*构建${BRANCH_TAG}分支 [${currentBuild.number}任务](${currentBuild.absoluteUrl}console) ${currentBuild.durationString}",
        //                         "---",
        //                         "- 执行人:  ${BUILD_USER}",
        //                         "- 本次构建的提交信息: ",
        //                         "> ${GIT_LAST_COMMIT_MESSAGE}",
        //                         "- git源码仓库位置: ",
        //                         "> ${gitlabSourceRepoName}",
        //                         "- [Jenkins job入口](https://jenkins.graspyun.com//job/${env.JOB_NAME}/)",
        //                         "- [Jenkins 发版位置](https://jenkins.graspyun.com/view/%E4%BA%91%E9%A1%B9%E7%9B%AE%E7%9A%84%E6%B5%8B%E8%AF%95%E5%92%8C%E7%81%B0%E5%BA%A6%E7%8E%AF%E5%A2%83/)",
        //                         "- [SonarQube代码质量入口](http://112.124.37.17:8599/dashboard?id=${env.JOB_NAME})",
        //                         "---",
        //                         "> ${DCURTIME}\n",
        //                         "> 如有疑问，请联系： 运维-宋亮成<songliangcheng@graspyun.com>"
        //                     ],
        //                     "atAll": false
        //                 )
        //         }

        //         container('mysql') {
        //           script {
        //               // 创建表的用户
        //               // 分支
        //               // 构建用户
        //               // 构建邮箱
        //               // 构建消息
        //               LANG='zh_CN.UTF-8'
        //               TZ='Asia/Shanghai'
        //               NOWTIME  = sh(script: "date +'%F %H:%M'", returnStdout: true).trim()
        //               CURTIME = sh(script: "date +%Y%m%d%H%M%S", returnStdout: true).trim()
        //               sh script: "mysql -u${MYSQL_CRED_USR} -p${MYSQL_CRED_PSW} -P8600 -h112.124.37.17 -e \"insert into jenkins_webhook.jenkins(createtime,modifytime,creator,buildbranch,builduser,buildemail,buildcommit,node,taskurl,codescan,imageinfo,buildduration,githash,jobname,jobnum) values ('${NOWTIME}','${NOWTIME}','jenkins','${BRANCH_TAG}','${BUILD_USER}','${BUILD_USER_EMAIL}','${GIT_LAST_COMMIT_MESSAGE}','${env.NODE_NAME}','${currentBuild.absoluteUrl}console', '开始构建','开始构建','${currentBuild.durationString}','${GIT_LAST_COMMIT}','${env.JOB_NAME}','${currentBuild.number}')\"",returnStdout:true
        //           }
        //         }

        //     }
        // }


        stage('代码构建并打包镜像，并上传') {
            steps {
                container('docker') {
                    script {
                        // RESULT = sh(script: "basename ${BRANCH_TAG}", returnStdout: true).trim()
                        // docker piepline插件，docker相关的插件
                        docker.withRegistry("${PROTOCOL}${REPOSITORY}/${REPO_BASE_NAME}", 'docker-kubernetes') {
                            customImage = docker.build("${REPOSITORY}/${REPO_BASE_NAME}:${BRANCH_TAG}-${CURTIME}")
                            customImage.push("${BRANCH_TAG}-${CURTIME}")
                            customImage.push("${BRANCH_TAG}-latest")

                        } 

                        // 清理镜像
                        sh "docker rmi ${REPOSITORY}/${REPO_BASE_NAME}:${BRANCH_TAG}-${CURTIME}"
                        sh "docker rmi ${REPOSITORY}/${REPO_BASE_NAME}:${BRANCH_TAG}-latest"
                    }
                }
            }
        } 


      // stage('镜像构建完成') {
      //   steps {

      //       container('jq') {

      //         // 钉钉的告警
      //               script {
      //                   if (env.gitlabUserEmail) {
      //                     BUILD_USER_EMAIL="${env.gitlabUserEmail}"
      //                   }
      //                   DCURTIME = sh(script: "date +'%Y年%m月%d日 %H时%M分%S秒'", returnStdout: true).trim()
      //                   NOWTIME  = sh(script: "date +'%F %H:%M'", returnStdout: true).trim()
      //               }
      //          dingtalk (
      //                   // robot: 'aec16636-91cf-45a3-8083-b9376b4fb741',
      //                   robot: "${DINGDINGID}",
      //                   type: 'MARKDOWN',
      //                   title: '你有新的消息，请注意查收',
      //                   text: [
      //                       " ${env.JOB_NAME} *结束*构建${BRANCH_TAG}分支 [${currentBuild.number}任务](${currentBuild.absoluteUrl}console) ${currentBuild.durationString}",
      //                       "```",
      //                       "本次构建结果: ${currentBuild.currentResult}",
      //                       "```",
      //                       "---",
      //                       "- 执行人:  ${BUILD_USER} ",
      //                       "- build邮箱: ${BUILD_USER_EMAIL}",
      //                       "- 本次构建的提交信息: ",
      //                       "> ${GIT_LAST_COMMIT_MESSAGE}",
      //                       "- git源码仓库位置: ",
      //                       "> ${gitlabSourceRepoName}",
      //                       "- [Jenkins job入口](https://jenkins.graspyun.com//job/${env.JOB_NAME}/)",
      //                       "- [Jenkins 发版位置](https://jenkins.graspyun.com/view/%E4%BA%91%E9%A1%B9%E7%9B%AE%E7%9A%84%E6%B5%8B%E8%AF%95%E5%92%8C%E7%81%B0%E5%BA%A6%E7%8E%AF%E5%A2%83/)",
      //                       "- [SonarQube代码质量入口](http://112.124.37.17:8599/dashboard?id=${env.JOB_NAME})",
      //                       "---",
      //                       "> ${DCURTIME}\n 对应镜像编号:${BRANCH_TAG}-${CURTIME}\n",
      //                       "> 如有疑问，请联系： 运维-宋亮成<songliangcheng@graspyun.com>"
      //                   ],
      //                   "atAll": false
      //               )
      //       }

      //       container('mysql') {
      //         script {
      //             // 创建表的用户
      //             // 分支
      //             // 构建用户
      //             // 构建邮箱
      //             // 构建消息
      //             LANG='zh_CN.UTF-8'
      //             TZ='Asia/Shanghai'
      //             sh script: "mysql -u${MYSQL_CRED_USR} -p${MYSQL_CRED_PSW} -P8600 -h112.124.37.17 -e \"insert into jenkins_webhook.jenkins(createtime,modifytime,creator,buildbranch,builduser,buildemail,buildcommit,node,taskurl,imageinfo,currentResult,jobname,buildduration,githash,jobnum) values ('${NOWTIME}','${NOWTIME}','jenkins','${BRANCH_TAG}','${BUILD_USER}','${BUILD_USER_EMAIL}','${GIT_LAST_COMMIT_MESSAGE}','${env.NODE_NAME}','${currentBuild.absoluteUrl}console','${REPOSITORY}/${REPO_BASE_NAME}:${BRANCH_TAG}-${CURTIME}','${currentBuild.currentResult}','${env.JOB_NAME}','${currentBuild.durationString}','${GIT_LAST_COMMIT}','${currentBuild.number}')\"",returnStdout:true
      //         }
      //       }

      //   }



      // }


        // https://stackoverflow.com/questions/45693418/sonarqube-quality-gate-not-sending-webhook-to-jenkins
        stage('QA Check') {
            steps {
                withCredentials([string(credentialsId: 'sonarqube', variable: 'SONAR_USER')]) {
                    container('sonar') {
                        sh "sonar-scanner -Dsonar.sources=./src -Dsonar.host.url=http://112.124.37.17:8599 -Dsonar.login=${SONAR_USER}  -Dsonar.projectKey=${env.JOB_NAME}  -Dsonar.projectName=${env.JOB_NAME}"
                        script {
                          SONAR_USER="${SONAR_USER}"
                        }
                    }
                }
            }
        }


        stage("Quality Gate") {
            steps {
                container('jq') {

                     script {
                            while(true){
                     
                                sh "sleep 2"
                                def url="https://jenkins.graspyun.com/job/${env.JOB_NAME.replaceAll('/','/job/')}/${currentBuild.number}/consoleText";
                                def sonarId = sh script: "wget -qO- --content-on-error --no-proxy --auth-no-challenge --http-user=${JENKINS_CRED_USR} --http-password=${JENKINS_CRED_PSW} '${url}'  | grep 'More about the report processing' | head -n1 ",returnStdout:true
                                sonarId = sonarId.substring(sonarId.indexOf("=")+1)
                                echo "sonarId ${sonarId}"


                                def sonarUrl = "http://112.124.37.17:8599/api/ce/task?id=${sonarId}"
                                SONARTASKURL = "http://112.124.37.17:8599/dashboard?id=${env.JOB_NAME}"

                                // 代码扫描执行的状态
                                def sonarStatus = sh script: "wget -qO- --content-on-error --no-proxy --auth-no-challenge --http-user=${SONAR_USER}  --http-password='' '${sonarUrl}' | jq -r '.task' | jq -r '.status' ",returnStdout:true
                                echo "Sonar status ... ${sonarStatus}"
                                if(sonarStatus.trim() == "SUCCESS"){
                                    sonarcodescanresult = 'OK'
                                    echo "BREAK";
                                } else if(sonarStatus.trim() == "FAILED"){
                                    echo "FAILED"
                                    sonarcodescanresult = 'FAILED'
                                } else if(sonarStatus.trim() == "IN_PROGRESS") {
                                  echo '继续' 
                                  continue
                                }
                                echo "代码扫描结果  ${sonarcodescanresult}"
                                // 代码质量状态
                                def sonarQualityGateUrl = "http://112.124.37.17:8599/api/measures/component?component=${env.JOB_NAME}&metricKeys=alert_status"
                                def sonarQualityGateStatus = sh script: "wget -qO- --content-on-error --no-proxy --auth-no-challenge --http-user=${SONAR_USER} --http-password='' '${sonarQualityGateUrl}' | jq -r .component.measures[].value ",returnStdout:true
                                echo "sonar Quality Gate Status ... ${sonarQualityGateStatus}"
                                if(sonarQualityGateStatus.trim() == "OK"){
                                    echo "BREAK";
                                    sonarresult = 'OK'
                                    break;
                                }
                                if(sonarQualityGateStatus.trim() == "ERROR"){
                                    echo "ERROR"
                                    sonarresult = 'ERROR'
                                    break;
                                }
                            }
                        }
                }


                }
            }

    }

    post {
        always {
            container('jq') {

              // 钉钉的告警
                    script {
                        if (env.gitlabUserEmail) {
                          BUILD_USER_EMAIL="${env.gitlabUserEmail}"
                        }
                        DCURTIME = sh(script: "date +'%Y年%m月%d日 %H时%M分%S秒'", returnStdout: true).trim()
                        NOWTIME  = sh(script: "date +'%F %H:%M'", returnStdout: true).trim()
                    }
            //    dingtalk (
            //             // robot: 'aec16636-91cf-45a3-8083-b9376b4fb741',
            //             robot: "${DINGDINGID}",
            //             type: 'MARKDOWN',
            //             title: '你有新的消息，请注意查收',
            //             text: [
            //                 " ${env.JOB_NAME} *结束*构建${BRANCH_TAG}分支 [${currentBuild.number}任务](${currentBuild.absoluteUrl}console) ${currentBuild.durationString}",
            //                 "```",
            //                 "执行扫描结果：${sonarcodescanresult}",
            //                 "代码质量结果：${sonarresult}",
            //                 "本次构建结果: ${currentBuild.currentResult}",
            //                 "```",
            //                 "---",
            //                 "- 执行人:  ${BUILD_USER} ",
            //                 "- build邮箱: ${BUILD_USER_EMAIL}",
            //                 "- 本次构建的提交信息: ",
            //                 "> ${GIT_LAST_COMMIT_MESSAGE}",
            //                 "- git源码仓库位置: ",
            //                 "> ${gitlabSourceRepoName}",
            //                 "- [SonarQube 本次构建代码质量入口](${SONARTASKURL})",
            //                 "---",
            //                 "> ${DCURTIME}\n 对应镜像编号:${BRANCH_TAG}-${CURTIME}\n",
            //                 "> 如有疑问，请联系： 运维-宋亮成<songliangcheng@graspyun.com>"
            //             ],
            //             "atAll": false
            //         )
            // }

            container('mysql') {
              script {
                  // 创建表的用户
                  // 分支
                  // 构建用户
                  // 构建邮箱
                  // 构建消息
                  LANG='zh_CN.UTF-8'
                  TZ='Asia/Shanghai'
                  sh script: "mysql -u${MYSQL_CRED_USR} -p${MYSQL_CRED_PSW} -P8600 -h112.124.37.17 -e \"insert into jenkins_webhook.jenkins(createtime,modifytime,creator,buildbranch,builduser,buildemail,buildcommit,node,taskurl,codescan,imageinfo,sonarcodescanresult,sonarresult,currentResult,jobname,buildduration,githash,jobnum) values ('${NOWTIME}','${NOWTIME}','jenkins','${BRANCH_TAG}','${BUILD_USER}','${BUILD_USER_EMAIL}','${GIT_LAST_COMMIT_MESSAGE}','${env.NODE_NAME}','${currentBuild.absoluteUrl}console', '${SONARTASKURL}','${REPOSITORY}/${REPO_BASE_NAME}:${BRANCH_TAG}-${CURTIME}','${sonarcodescanresult}','${sonarresult}','${currentBuild.currentResult}','${env.JOB_NAME}','${currentBuild.durationString}','${GIT_LAST_COMMIT}','${currentBuild.number}')\"",returnStdout:true
              }
            }

        }

    }

}
}

// CREATE TABLE `jenkins` (
//   `id` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '与业务无关的id',
//   `createtime` datetime NOT NULL COMMENT '创建时间',
//   `modifytime` datetime NOT NULL DEFAULT '0000-00-00 00:00:00' ON UPDATE current_timestamp() COMMENT '修改时间',
//   `creator` varchar(255) DEFAULT NULL COMMENT '创建者',
//   `buildbranch` varchar(255) DEFAULT NULL COMMENT '构建分支',
//   `builduser` varchar(255) DEFAULT NULL COMMENT '构建者用户',
//   `buildemail` varchar(255) DEFAULT NULL COMMENT '构建者邮箱',
//   `buildcommit` text DEFAULT NULL COMMENT '提交信息',
//   `node` varchar(255) DEFAULT '' COMMENT '执行节点',
//   `taskurl` varchar(255) DEFAULT NULL COMMENT '任务url',
//   `codescan` varchar(255) DEFAULT NULL COMMENT '代码质量url',
//   `imageinfo` varchar(255) DEFAULT NULL COMMENT '镜像信息',
//   `sonarcodescanresult` varchar(255) DEFAULT NULL COMMENT '执行sonar扫描的结果，成功或失败',
//   `sonarresult` varchar(255) DEFAULT NULL COMMENT '代码质量，sonarqube质量都达标时 成功',
//   `currentResult` varchar(255) DEFAULT NULL COMMENT 'Jenkins构建结果 ',
//   `jobname` varchar(255) DEFAULT NULL COMMENT 'jenkins构建的job名',
//   `buildduration` varchar(255) DEFAULT NULL COMMENT 'jenkins构建时长',
//   `githash` varchar(255) DEFAULT NULL COMMENT 'git提交的hash',
//   `jobnum` int(11) DEFAULT NULL COMMENT 'job的构建号',
//   PRIMARY KEY (`id`)
// ) ENGINE=InnoDB AUTO_INCREMENT=47 DEFAULT CHARSET=utf8mb4;

```



### 配置Jenkinsfile准备

#### yaml准备

```yaml
              serviceAccountName: jenkins-build-sa
              containers:
              - name: docker
                image: docker:19.03.12
              - name: sonar
                image: sonarsource/sonar-scanner-cli:4.6
              - name: jq
                image: registry.cn-hangzhou.aliyuncs.com/grasp_base_images/ubuntu-jq-curl:latest
              - name: mysql
                image: registry.cn-hangzhou.aliyuncs.com/grasp_base_images/mysql:latest
```

docker, sonar, 有现成的

ubuntu-jq-curl

```dockerfile
FROM ubuntu-baseimage:18.04.01


RUN apt install curl jq -y
```

mysql

```dockerfile
FROM ubuntu-baseimage:18.04.01
LABEL AUTHOR=songliangcheng QQ=2192383945@qq.com
RUN apt install mysql-client tzdata -y
```



serviceaccount包含registry认证信息

略

#### 安装插件

网址：

http://blog.mykernel.cn/2021/04/14/jenkins-%E8%BF%81%E7%A7%BB/#%E5%AE%89%E8%A3%85%E6%8F%92%E4%BB%B6



#### 添加job，并粘贴jenkinsfile

![image-20210512151921961](http://myapp.img.mykernel.cn/image-20210512151921961.png)

![image-20210512151945255](http://myapp.img.mykernel.cn/image-20210512151945255.png)

#### jenkins配置钉钉告警

jenkins -> system config

![image-20210512151732744](http://myapp.img.mykernel.cn/image-20210512151732744.png)



#### 调整环境变量

gitlab认证id

![image-20210512152303545](http://myapp.img.mykernel.cn/image-20210512152303545.png)

认证sonarqube的id

![image-20210512152424306](http://myapp.img.mykernel.cn/image-20210512152424306.png)

jenkins自己的用户及密码



sonar的地址 需要搭建sonarqube, 参考 [docker-compose启动sonarqube](http://blog.mykernel.cn/2021/03/22/docker-compose启动sonarqube/)

```groovy
    // jenkins正常指令
    environment {

        // 镜像保存在哪里？
        REPOSITORY="registry.cn-hangzhou.aliyuncs.com/grasp_base_images"
        // 生成镜像的仓库名, 此名要和发版的此名一致
        REPO_BASE_NAME="demoapp"
        
        
        // 登陆你的仓库的gitlab的认证id. 用户名和密码
        GITCREDENTIAL="gitlab-documents-user-pass"

        // 认证sonarqube的认证id  secret text
        CRED = credentials('sonarqube') 
        
        // jenkins-self jenkins自己的认证信息，获取jenkins的控制台 用户名和密码
        JENKINS_CRED = credentials('jenkins-self') 

        //mysql 用户名和密码
        MYSQL_CRED = credentials('jenkins-to-mysql')

        // 这个来自jenkins的钉钉告警自动生成的id
        // DINGDINGID = 'd1b9b632-cfbb-4ca9-966a-2c9534e14b59' 
        DINGDINGID = 'aec16636-91cf-45a3-8083-b9376b4fb741'
        
        
        // sonar的地址
        SONARURI = 'http://112.124.37.17:8599'
        SONARURL="${SONARURI}/dashboard?id=${env.JOB_NAME}"
    }
```

#### 发版位置

```groovy
                   dingtalk (

                                "- [Jenkins 发版位置](https://jenkins.graspyun.com/view/%E4%BA%91%E9%A1%B9%E7%9B%AE%E7%9A%84%E6%B5%8B%E8%AF%95%E5%92%8C%E7%81%B0%E5%BA%A6%E7%8E%AF%E5%A2%83/)",

                        )
```

#### 准备mysql

库

```bash
jenkins_webhook
```



表结构

```groovy
// CREATE TABLE `jenkins` (
//   `id` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '与业务无关的id',
//   `createtime` datetime NOT NULL COMMENT '创建时间',
//   `modifytime` datetime NOT NULL DEFAULT '0000-00-00 00:00:00' ON UPDATE current_timestamp() COMMENT '修改时间',
//   `creator` varchar(255) DEFAULT NULL COMMENT '创建者',
//   `buildbranch` varchar(255) DEFAULT NULL COMMENT '构建分支',
//   `builduser` varchar(255) DEFAULT NULL COMMENT '构建者用户',
//   `buildemail` varchar(255) DEFAULT NULL COMMENT '构建者邮箱',
//   `buildcommit` text DEFAULT NULL COMMENT '提交信息',
//   `node` varchar(255) DEFAULT '' COMMENT '执行节点',
//   `taskurl` varchar(255) DEFAULT NULL COMMENT '任务url',
//   `codescan` varchar(255) DEFAULT NULL COMMENT '代码质量url',
//   `imageinfo` varchar(255) DEFAULT NULL COMMENT '镜像信息',
//   `sonarcodescanresult` varchar(255) DEFAULT NULL COMMENT '执行sonar扫描的结果，成功或失败',
//   `sonarresult` varchar(255) DEFAULT NULL COMMENT '代码质量，sonarqube质量都达标时 成功',
//   `currentResult` varchar(255) DEFAULT NULL COMMENT 'Jenkins构建结果 ',
//   `jobname` varchar(255) DEFAULT NULL COMMENT 'jenkins构建的job名',
//   `buildduration` varchar(255) DEFAULT NULL COMMENT 'jenkins构建时长',
//   `githash` varchar(255) DEFAULT NULL COMMENT 'git提交的hash',
//   `jobnum` int(11) DEFAULT NULL COMMENT 'job的构建号',
//   PRIMARY KEY (`id`)
// ) ENGINE=InnoDB AUTO_INCREMENT=47 DEFAULT CHARSET=utf8mb4;
```

#### gitlab触发

获取地址

![image-20210512153718999](http://myapp.img.mykernel.cn/image-20210512153718999.png)



`https://jenkins.graspyun.com/project/cloudapi-ci`



获取token

```groovy
  triggers {
    // push事件，会触发，只对源分支为test的或目标分支为test触发。 开发本地合并，推送之后，源一定是test
    // merge打开请求的事件，在gitlab ci完成
    gitlab(triggerOnPush: true, triggerOnMergeRequest: false, branchFilterType: 'RegexBasedFilter', sourceBranchRegex: '.*test.*',targetBranchRegex: '.*test.*',secretToken: 'e1ed6d39b609d15f8ce27173675de080')
  }
```

配置gitlab

![image-20210512153858967](http://myapp.img.mykernel.cn/image-20210512153858967.png)

### 准备一个测试仓库

里面放测试的代码， sourcetree

![image-20210512154313754](http://myapp.img.mykernel.cn/image-20210512154313754.png)

![image-20210512154402001](http://myapp.img.mykernel.cn/image-20210512154402001.png)

现在可以下载你的开发的仓库的代码，挪到这个本地的自己的仓库， 并推上自己的仓库，并配置gitlab触发

### 准备dockerfile

因为在Jenkinsfile此步骤要求在git克隆的本地仓库有一个Dockerfile

```groovy
        stage('代码构建并打包镜像，并上传') {
            steps {
                container('docker') {

                        docker.withRegistry("${PROTOCOL}${REPOSITORY}/${REPO_BASE_NAME}", 'docker-kubernetes') {
                            customImage = docker.build("${REPOSITORY}/${REPO_BASE_NAME}:${BRANCH_TAG}-${CURTIME}") //这里

                    }
                }
            }
```

由于`sourceBranchRegex: '.*test.*',targetBranchRegex: '.*test.*',`在trigger中定义，所以在准备dockerfile前需要在本地新建一个test分支

```groovy
  triggers {
    // push事件，会触发，只对源分支为test的或目标分支为test触发。 开发本地合并，推送之后，源一定是test
    // merge打开请求的事件，在gitlab ci完成
    gitlab(triggerOnPush: true, triggerOnMergeRequest: false, branchFilterType: 'RegexBasedFilter', sourceBranchRegex: '.*test.*',targetBranchRegex: '.*test.*',secretToken: 'e1ed6d39b609d15f8ce27173675de080')
  }
```

![image-20210512155342473](http://myapp.img.mykernel.cn/image-20210512155342473.png)

#### npm的dockerfile

参考：[分层构建 — 深度优化npm + nginx – (高阶应用)](http://blog.mykernel.cn/2021/02/26/diary-%E8%AE%B0%E5%BD%95%E4%B8%80%E6%AC%A1%E9%83%A8%E7%BD%B2%E7%94%9F%E4%BA%A7%E7%8E%AF%E5%A2%83/#%E5%88%86%E5%B1%82%E6%9E%84%E5%BB%BA-%E2%80%94-%E6%B7%B1%E5%BA%A6%E4%BC%98%E5%8C%96npm-nginx-%E2%80%93-%E9%AB%98%E9%98%B6%E5%BA%94%E7%94%A8)

#### java的dockerfile

参考: [分层构建 — 深度优化 maven + jar – (高阶应用)](http://blog.mykernel.cn/2021/02/26/diary-%E8%AE%B0%E5%BD%95%E4%B8%80%E6%AC%A1%E9%83%A8%E7%BD%B2%E7%94%9F%E4%BA%A7%E7%8E%AF%E5%A2%83/#%E4%BC%98%E5%8C%96java-%E6%8F%90%E5%8D%87%E6%9E%84%E5%BB%BA%E9%80%9F%E5%BA%A6-%E2%80%93-%E9%AB%98%E9%98%B6%E5%BA%94%E7%94%A8)

#### other的dockerfile

开发人员自己准备好dockerfile, 目标是打包出生产可用的docker，并且传递不同的环境变量，就是不同的环境。

#### dotnet的sonarqube

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:5.0
ENV PATH $PATH:/root/.dotnet/tools
RUN dotnet tool install --global dotnet-sonarscanner
# https://docs.microsoft.com/zh-cn/dotnet/core/install/linux-centos
RUN wget https://packages.microsoft.com/config/ubuntu/18.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb && dpkg -i packages-microsoft-prod.deb && \
	 apt update && apt install -y apt-transport-https && apt update && apt install -y dotnet-sdk-5.0     aspnetcore-runtime-5.0 dotnet-runtime-5.0
	
	
# 交互进入容器，调整中文环境
# https://blog.csdn.net/newcong0123/article/details/106186807
	 
# 启动pod，使用中文变量
```



### 测试

在gitlab上切换分支，编辑自己的仓库任意文件，看看jenkinsfile执行情况





## 测试和灰度环境发布

命名 Jenkinsfile-deploy

```gro
// AUTHOR: songliangcheng
// DESC：插件
// jenkins对接kubernets: https://plugins.jenkins.io/kubernetes/ , 文中搜索：Declarative Pipeline
// pipeline使用构建者名称：https://wiki.jenkins.io/display/JENKINS/Build+User+Vars+Plugin
// pipeline的 git参数化构建：https://plugins.jenkins.io/git-parameter/

// pipeline的gitlab自动触发：https://plugins.jenkins.io/gitlab-plugin/， 支持的变量：Defined variables，引用变量: ${env.gitlabSourceBranch}
    // 自由或pipeline在见面中定义。多分支pipeline可以yaml定义。
// pipeline钉钉告警：https://plugins.jenkins.io/dingding-notifications/
// docker piepline插件，docker相关的插件

// 示例：https://github.com/jenkinsci/pipeline-examples/tree/master/declarative-examples/simple-examples



pipeline {

    agent {
        // jenkins对接kubernets
        // kubernetes的pod模板，不需要在jenkins中二次定义. 必须有nodeSelector, 否则不会调度
        kubernetes {
          yaml """\
            apiVersion: v1
            kind: Pod
            metadata:
              namespace: jenkins
              labels:
                some-label: some-label-value
            spec:
              nodeSelector:
                kubernetes.io/os: linux
              serviceAccountName: jenkins-build-sa
              containers:
            
              - name: docker
                image: docker:19.03.12
                command:
                - cat
                tty: true
                env:
                - name: LANG
                  value: zh_CN.UTF-8
                - name: TZ
                  value: Asia/Shanghai
                securityContext:
                  privileged: true
                volumeMounts:
                  - name: dind-storage
                    mountPath: /var/run/docker.sock
              - name: sonar
                image: sonarsource/sonar-scanner-cli:4.6
                command:
                - cat
                tty: true
                env:
                - name: LANG
                  value: zh_CN.UTF-8
                - name: TZ
                  value: Asia/Shanghai
                volumeMounts:
                  - name: sonar-storage
                    mountPath:  /opt/sonar-scanner/.sonar/cache
              - name: jq
                image: registry.cn-hangzhou.aliyuncs.com/grasp_base_images/ubuntu-jq-curl:latest
                command:
                - cat
                tty: true
                env:
                - name: LANG
                  value: zh_CN.UTF-8
                - name: TZ
                  value: Asia/Shanghai
              - name: mysql
                image: registry.cn-hangzhou.aliyuncs.com/grasp_base_images/mysql:latest
                command:
                - cat
                tty: true
                env:
                - name: LANG
                  value: zh_CN.UTF-8
                - name: TZ
                  value: Asia/Shanghai

              - name: kubectl
                image: registry.cn-hangzhou.aliyuncs.com/grasp_base_images/k8s-test:latest
                command:
                - cat
                tty: true

              volumes:
              - name: dind-storage
                hostPath:
                  path: /var/run/docker.sock
              
              - name: sonar-storage
                hostPath:
                  path: /opt/sonar-scanner/.sonar/cache
                  type: DirectoryOrCreate
            """.stripIndent()
        }
    }
    // jenkins正常指令
    environment {
        PROTOCOL="https://"
        REPOSITORY="registry.cn-hangzhou.aliyuncs.com/grasp_base_images"
        REPO_BASE_NAME="saas-customerweb"
        // git登陆 (参数化、非参数化均需要使用)
        GITCREDENTIAL="gitlab-documents-user-pass"
        // git url (参数化构建使用)
        GITHTTPURL="https://gitlab.mykernel.cn/graspyunplatform/saas-customerweb.git"

        // sonarqube 
        CRED = credentials('sonarqube') 
        // jenkins-self
        JENKINS_CRED = credentials('jenkins-self') 


        //mysql
        MYSQL_CRED = credentials('jenkins-to-mysql')

        // DINGDING ID
        DINGDINGID = 'd1b9b632-cfbb-4ca9-966a-2c9534e14b59'

        // 部署状态
        DEPLOY_STATUS="FAILURE"
    }

    // jenkins的git参数化构建
    parameters {
        gitParameter name: 'BRANCH_TAG', 
                     type: 'PT_BRANCH_TAG',
                     branchFilter: 'origin/(.*)',
                     defaultValue: 'master',
                     selectedValue: 'DEFAULT',
                     sortMode: 'DESCENDING_SMART',
                     // git url (参数化构建使用)
                     useRepository: 'https://gitlab.mykernel.cn/graspyunplatform/saas-customerweb.git',
           description: 'Select your branch or tag.'
        // jenkins参数化构建
        choice(name: 'deploy_env', choices: ['test','gray'],description: 'test 测试环境\ngray 灰度环境')         
        choice(name: 'deploy_method', choices: ['deploy','rollout'],description: 'deploy 部署\nrollout 回滚')         
    }

    // pipeline正常指令
    stages {
        // pipeline正常指令，一个stages里面多个stage, 每个stage里面1个steps {}
        stage("拉代码") {
          // pipeline正常指令，每个stage里面1个steps {}, 1个steps{} 里面可以有多个指令
          steps {
               // 构建用户信息：id, 名，email
                wrap([$class: 'BuildUser']) {
                   script {
                       BUILD_USER_ID = "${env.BUILD_USER_ID}"
                       BUILD_USER = "${env.BUILD_USER}"
                       BUILD_USER_EMAIL = "${env.BUILD_USER_EMAIL}"
                   }
                 }


                // 拉代码 参数化构建：master gitlab触发传递来的：origin/dev gitlab 用户名 slcnx gitlab merge的用户 songliangcheng gitlab用户邮箱 2192383945@qq.com gitlab merge标题 本地 t odev gitlab merge描述 dev 合并至 master
                echo "拉代码 参数化构建：${params.BRANCH_TAG} gitlab触发传递来的：origin/${env.gitlabSourceBranch} gitlab 用户名 ${env.gitlabUserName} gitlab merge的用户 ${env.gitlabMergedByUser} gitlab用户邮箱 ${env.gitlabUserEmail} gitlab merge标题 ${env.gitlabMergeRequestTitle} gitlab merge描述 ${env.gitlabMergeRequestDescription}"
                
                script {
                    sh 'printenv'
                    sh 'echo ============================================================'
                    // https://blog.csdn.net/nklinsirui/article/details/100521145
                    // https://www.twblogs.net/a/5d6fd26bbd9eee5327ff4b5d
                   
                    sh 'echo 参数化构建'
                    checkout([$class: 'GitSCM', 
                              branches: [[name: "${params.BRANCH_TAG}"]], 
                              doGenerateSubmoduleConfigurations: false, 
                              extensions: [], 
                              gitTool: 'Default', 
                              submoduleCfg: [], 
                              userRemoteConfigs: [[url: "${GITHTTPURL}",credentialsId: "${GITCREDENTIAL}",]]
                            ])

                    BRANCH_TAG = "${params.BRANCH_TAG}" 

       
                    //本次提交信息
                    GIT_LAST_COMMIT= sh(script:"git rev-parse --short HEAD",returnStdout:true).trim()
                    GIT_LAST_COMMIT_MESSAGE=sh(script: "git  --no-pager show -s --format='%s'  ${GIT_LAST_COMMIT} | tr -d \"'\" ",returnStdout:true).trim()
                    BUILD_USER_EMAIL=sh(script: "git  --no-pager show -s --format='%s'  ${GIT_LAST_COMMIT} | tr -d \"'\" ",returnStdout:true).trim()
                    echo "提交信息 ${GIT_LAST_COMMIT_MESSAGE}  提交hash ${GIT_LAST_COMMIT} "
                }
          }
        }
        stage('发布前报警') {
            steps {
                container('jq') {
                  script {
                    CURTIME = sh(script: "date +%Y%m%d%H%M%S", returnStdout: true).trim()

                  }
                }

                 container('jq') {

                    // pipeline使用构建者名称
                    // 构建用户信息：id, 名，email
                    wrap([$class: 'BuildUser']) {
                       script {
                           BUILD_USER_ID = "${env.BUILD_USER_ID}"
                           BUILD_USER = "${env.BUILD_USER}"
                           BUILD_USER_EMAIL = "${env.BUILD_USER_EMAIL}"
                       }
                     }


                    // 钉钉的告警 https://plugins.jenkins.io/dingding-notifications/
                    script {
                        DCURTIME = sh(script: "date +'%Y年%m月%d日 %H时%M分%S秒'", returnStdout: true).trim()
                        
                        // 判断执行人
                        if (BUILD_USER == "SCM Change") {
                            // 如果构建人是SCM Change是触发构建
                            BUILD_USER ="${env.gitlabUserName}"
                        } 

                    }
                    dingtalk (
                            robot: "${DINGDINGID}",
                            // robot: 'd1b9b632-cfbb-4ca9-966a-2c9534e14b59',
                            type: 'MARKDOWN',
                            title: '你有新的消息，请注意查收',
                            text: [
                                " ${env.JOB_NAME} 开始${deploy_method} ${deploy_env} ${BRANCH_TAG}分支 [${currentBuild.number}任务](${currentBuild.absoluteUrl}console) ${currentBuild.durationString}",
                                "---",
                                "- 执行人:  ${BUILD_USER}",
                                "- 本次构建的提交信息: ",
                                "> ${GIT_LAST_COMMIT_MESSAGE}",
                                "- [Jenkins job入口](https://jenkins.graspyun.com//job/${env.JOB_NAME}/)",
                                "- [SonarQube代码质量入口](http://112.124.37.17:8599)",
                                "---",
                                "> ${DCURTIME}\n",
                                "> 如有疑问，请联系： 运维-宋亮成<songliangcheng@graspyun.com>"
                            ],
                            "atAll": true
                        )
                }

                container('mysql') {
                  script {
                      // 创建表的用户
                      // 分支
                      // 构建用户
                      // 构建邮箱
                      // 构建消息
                      LANG='zh_CN.UTF-8'
                      TZ='Asia/Shanghai'
                      NOWTIME  = sh(script: "date +'%F %H:%M'", returnStdout: true).trim()
                      sh script: "mysql -u${MYSQL_CRED_USR} -p${MYSQL_CRED_PSW} -P8600 -h112.124.37.17 -e \"insert into jenkins_webhook.jenkins(createtime,modifytime,creator,buildbranch,builduser,buildemail,buildcommit,node,taskurl,codescan,imageinfo,buildduration,githash,jobname,jobnum) values ('${NOWTIME}','${NOWTIME}','jenkins','${BRANCH_TAG}','${BUILD_USER}','${BUILD_USER_EMAIL}','${GIT_LAST_COMMIT_MESSAGE}','${env.NODE_NAME}','${currentBuild.absoluteUrl}console', '开始构建','开始构建','${currentBuild.durationString}','${GIT_LAST_COMMIT}','${env.JOB_NAME}','${currentBuild.number}')\"",returnStdout:true
                  }
                }

            }
        }


        stage('代码构建并打包镜像，并上传') {
            when {
                expression { return  BRANCH_TAG == "master" && deploy_method == "deploy" }
            }
            steps {
               
                container('docker') {
                    script {
                        // RESULT = sh(script: "basename ${BRANCH_TAG}", returnStdout: true).trim()
                        // docker piepline插件，docker相关的插件
                        docker.withRegistry("${PROTOCOL}${REPOSITORY}/${REPO_BASE_NAME}", 'docker-kubernetes') {
                            customImage = docker.build("${REPOSITORY}/${REPO_BASE_NAME}:${BRANCH_TAG}-${CURTIME}")
                            customImage.push("${BRANCH_TAG}-${CURTIME}")
                            customImage.push("${BRANCH_TAG}-latest")
                        } 
                        // 清理镜像
                        sh "docker rmi ${REPOSITORY}/${REPO_BASE_NAME}:${BRANCH_TAG}-${CURTIME}"
                        sh "docker rmi ${REPOSITORY}/${REPO_BASE_NAME}:${BRANCH_TAG}-latest"
                    }
                }


            }
        } 

      // 升级镜像
      stage("环境升级镜像并发布") {
        steps {
          container('docker') {
          script {
              if (deploy_method == "deploy") {
                // 拉镜像，升级镜像
                docker.withRegistry("${PROTOCOL}${REPOSITORY}/${REPO_BASE_NAME}", 'docker-kubernetes') {
                  // 测试环境拉分支最新
                  if (deploy_env == "test") {
                    customImage = docker.image("${REPOSITORY}/${REPO_BASE_NAME}:${params.BRANCH_TAG}-latest")
                  // 灰度环境拉分支的test最新
                  } else if (deploy_env == "gray") {
                    customImage = docker.image("${REPOSITORY}/${REPO_BASE_NAME}:${params.BRANCH_TAG}_test-latest")
                  // 生产环境拉分支的gray最新
                  } else if (deploy_env == "prod") {
                    customImage = docker.image("${REPOSITORY}/${REPO_BASE_NAME}:${params.BRANCH_TAG}_gray-latest")
                  } 
                  customImage.pull()
                  customImage.push("${BRANCH_TAG}_${deploy_env}-${CURTIME}")
                  customImage.push("${BRANCH_TAG}_${deploy_env}-latest")
                }

                // 清理镜像
                  // 测试环境拉分支最新
                  if (deploy_env == "test") {
                    sh "docker rmi ${REPOSITORY}/${REPO_BASE_NAME}:${params.BRANCH_TAG}-latest"
                  // 灰度环境拉分支的test最新
                  } else if (deploy_env == "gray") {
                    sh "docker rmi ${REPOSITORY}/${REPO_BASE_NAME}:${params.BRANCH_TAG}_test-latest"
                  // 生产环境拉分支的gray最新
                  } else if (deploy_env == "prod") {
                    sh "docker rmi ${REPOSITORY}/${REPO_BASE_NAME}:${params.BRANCH_TAG}_gray-latest"
                  }
                sh "docker rmi ${REPOSITORY}/${REPO_BASE_NAME}:${BRANCH_TAG}_${deploy_env}-${CURTIME}"  
                sh "docker rmi ${REPOSITORY}/${REPO_BASE_NAME}:${BRANCH_TAG}_${deploy_env}-latest"
              }
            } 
          }

          script {
            container('kubectl') {
                checkout([$class: 'GitSCM', 
                          branches: [[name: "master"]], 
                          doGenerateSubmoduleConfigurations: false, 
                          extensions: [], 
                          gitTool: 'Default', 
                          submoduleCfg: [], 
                          userRemoteConfigs: [[url: "https://gitlab.mykernel.cn/songliangcheng/configcenter.git",credentialsId: "${GITCREDENTIAL}",]]
                        ])

                // 准备测试的k8s的config
                sh "cp jenkins-pipelines/kubernetes-kubeconfig/${deploy_env}-config  ~/.kube/config"

                // 测试发版
                script {
                  if (deploy_method == "deploy") {

                    sh "cd  jenkins-pipelines/${JOB_NAME}/${deploy_env} && sed -i 's@newName:.*@newName: ${REPOSITORY}/${REPO_BASE_NAME}@g' kustomization.yaml && sed  -i 's@newTag:.*@newTag: ${BRANCH_TAG}_${deploy_env}-${CURTIME}@g' kustomization.yaml  && kustomize build  > jenkins_${env.JOB_NAME}_${currentBuild.number}.yaml && kubectl apply -f jenkins_${env.JOB_NAME}_${currentBuild.number}.yaml  --record && cd ../../ && sh check_deploy_status.sh"
                    // 如果上面返回非0，jenkins自动退出。
                    // 如果返回0，继续走
                    DEPLOY_STATUS="OK"
                  }
                  else if (deploy_method == "rollout") {
                    sh "cd  jenkins-pipelines && sh rollout.sh"
                  }
                }
            }
          }

        }

      }


    }

    post {
        always {

            container('jq') {


              // 钉钉的告警
                    script {
                        if (env.gitlabUserEmail) {
                          BUILD_USER_EMAIL="${env.gitlabUserEmail}"
                        }
                        // CURTIME = sh(script: "date +%Y%m%d%H%M%S", returnStdout: true).trim()
                        DCURTIME = sh(script: "date +'%Y年%m月%d日 %H时%M分%S秒'", returnStdout: true).trim()
                        NOWTIME  = sh(script: "date +'%F %H:%M'", returnStdout: true).trim()
                    }


               script {
                  echo 'deploy ======='
                 dingtalk (
                          robot: "${DINGDINGID}",
                          // robot: 'd1b9b632-cfbb-4ca9-966a-2c9534e14b59',
                          type: 'MARKDOWN',
                          title: '你有新的消息，请注意查收',
                          text: [
                              " ${env.JOB_NAME} ${deploy_method} ${deploy_env} 完成 ${BRANCH_TAG}分支 [${currentBuild.number}任务](${currentBuild.absoluteUrl}console) ${currentBuild.durationString}",
                              "- 发布状态: ${DEPLOY_STATUS}",
                              "---",
                              "- [Jenkins job入口](https://jenkins.graspyun.com//job/${env.JOB_NAME}/)",
                              "- [SonarQube代码质量入口](http://112.124.37.17:8599)",
                              "---",
                              "> ${DCURTIME}\n 对应镜像编号:${BRANCH_TAG}-${CURTIME}\n",
                              "> 如有疑问，请联系： 运维-宋亮成<songliangcheng@graspyun.com>"
                          ],
                          "atAll": true
                      )
               }



            }

            container('mysql') {
              script {

    
                LANG='zh_CN.UTF-8'
                TZ='Asia/Shanghai'
                NOWTIME  = sh(script: "date +'%F %H:%M'", returnStdout: true).trim()
                sh script: "mysql -u${MYSQL_CRED_USR} -p${MYSQL_CRED_PSW} -P8600 -h112.124.37.17 -e \"insert into jenkins_webhook.jenkins(createtime,modifytime,creator,buildbranch,builduser,buildemail,buildcommit,node,taskurl,codescan,imageinfo,buildduration,githash,jobname,jobnum) values ('${NOWTIME}','${NOWTIME}','jenkins','${BRANCH_TAG}','${BUILD_USER}','${BUILD_USER_EMAIL}','${GIT_LAST_COMMIT_MESSAGE}','${env.NODE_NAME}','${currentBuild.absoluteUrl}console', '发布完成','${DEPLOY_STATUS}','${currentBuild.durationString}','${GIT_LAST_COMMIT}','${env.JOB_NAME}','${currentBuild.number}')\"",returnStdout:true
            }

        }

    }
}
}


// CREATE TABLE `jenkins` (
//   `id` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '与业务无关的id',
//   `createtime` datetime NOT NULL COMMENT '创建时间',
//   `modifytime` datetime NOT NULL DEFAULT '0000-00-00 00:00:00' ON UPDATE current_timestamp() COMMENT '修改时间',
//   `creator` varchar(255) DEFAULT NULL COMMENT '创建者',
//   `buildbranch` varchar(255) DEFAULT NULL COMMENT '构建分支',
//   `builduser` varchar(255) DEFAULT NULL COMMENT '构建者用户',
//   `buildemail` varchar(255) DEFAULT NULL COMMENT '构建者邮箱',
//   `buildcommit` text DEFAULT NULL COMMENT '提交信息',
//   `node` varchar(255) DEFAULT '' COMMENT '执行节点',
//   `taskurl` varchar(255) DEFAULT NULL COMMENT '任务url',
//   `codescan` varchar(255) DEFAULT NULL COMMENT '代码质量url',
//   `imageinfo` varchar(255) DEFAULT NULL COMMENT '镜像信息',
//   `sonarcodescanresult` varchar(255) DEFAULT NULL COMMENT '执行sonar扫描的结果，成功或失败',
//   `sonarresult` varchar(255) DEFAULT NULL COMMENT '代码质量，sonarqube质量都达标时 成功',
//   `currentResult` varchar(255) DEFAULT NULL COMMENT 'Jenkins构建结果 ',
//   `jobname` varchar(255) DEFAULT NULL COMMENT 'jenkins构建的job名',
//   `buildduration` varchar(255) DEFAULT NULL COMMENT 'jenkins构建时长',
//   `githash` varchar(255) DEFAULT NULL COMMENT 'git提交的hash',
//   `jobnum` int(11) DEFAULT NULL COMMENT 'job的构建号',
//   PRIMARY KEY (`id`)
// ) ENGINE=InnoDB AUTO_INCREMENT=47 DEFAULT CHARSET=utf8mb4;
```

### 准备job

![image-20210513103159810](http://myapp.img.mykernel.cn/image-20210513103159810.png)

同名，方便jenkins和git中对应

### 准备git的地址

```groovy
    environment {
        // git url (参数化构建使用)
        GITHTTPURL="https://gitlab.mykernel.cn/graspyunplatform/saas-customerweb.git"
    }


    parameters {
                     // git url (参数化构建使用)
                     useRepository: 'https://gitlab.mykernel.cn/graspyunplatform/saas-customerweb.git',
   }
```

这两个需要一致

### 钉钉发送位置

jenkins -> 配置 -> system config中的钉钉的ID

```bash
    environment {
      	// DINGDING ID
        DINGDINGID = 'd1b9b632-cfbb-4ca9-966a-2c9534e14b59'

    }
```

### 配置仓库

```groovy
            container('kubectl') {
                checkout([$class: 'GitSCM', 
                          branches: [[name: "master"]], 
                          doGenerateSubmoduleConfigurations: false, 
                          extensions: [], 
                          gitTool: 'Default', 
                          submoduleCfg: [], 
                          userRemoteConfigs: [[url: "https://gitlab.mykernel.cn/songliangcheng/configcenter.git",credentialsId: "${GITCREDENTIAL}",]]
                        ])

```



### 准备不同环境的k8s连接配置

```groovy

                // 准备测试的k8s的config
                sh "cp jenkins-pipelines/kubernetes-kubeconfig/${deploy_env}-config  ~/.kube/config"

```

要求仓库下有`jenkins-pipelines/kubernetes-kubeconfig/`目录，目录中有参数化构建不同名字-config的k8s的kubeconfig文件



### 准备一个demoapp的kustomize文件

```groovy
                // 测试发版
                script {
                  if (deploy_method == "deploy") {

                    sh "cd  jenkins-pipelines/${JOB_NAME}/${deploy_env} && sed -i 's@newName:.*@newName: ${REPOSITORY}/${REPO_BASE_NAME}@g' kustomization.yaml && sed  -i 's@newTag:.*@newTag: ${BRANCH_TAG}_${deploy_env}-${CURTIME}@g' kustomization.yaml  && kustomize build  > jenkins_${env.JOB_NAME}_${currentBuild.number}.yaml && kubectl apply -f jenkins_${env.JOB_NAME}_${currentBuild.number}.yaml  --record && cd ../../ && sh check_deploy_status.sh"
```

>  所以jenkins的job名称应该和此处的demoapp一致，这样直接进入JOB_NAME替换的目录
>
> deploy_env 这个就是参数化构建的可选参数，应该和应用的不同环境目录一致

发版过程：就是把新的镜像名和标签写入kustomize文件，然后生成指定构建id的yaml, 进程 apply --record, 可以在历史中查看应用哪个构建的yaml, 方便回溯这个历史发版哪些功能 

例如：

![image-20210513105146131](http://myapp.img.mykernel.cn/image-20210513105146131.png)

现在可以看到构建id

![image-20210513105233563](http://myapp.img.mykernel.cn/image-20210513105233563.png)

#### 基础目录

此基础目录，我是为web应用准备

```bash
root@ccea447f1caa:~/configcenter# tree jenkins-pipelines/base/
jenkins-pipelines/base/
├── deployment.yaml
├── ingress.yaml
├── kustomization.yaml
├── ns.yaml
└── service.yaml
```

##### nginx deploy 

```diff
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deploy-label
  name: deploy
  namespace: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deploy-selector
+  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: deploy-selector
    spec:
+      terminationGracePeriodSeconds: 90
      containers:
      - image: ikubernetes/deploy:v1
        name: deploy
        resources: 
          limits:
            cpu: 500
            memory: 500Mi
          requests:
            cpu: 200m
            memory: 200Mi
        ports:
        - containerPort: 80
          protocol: TCP
          name: http
+        livenessProbe:   
          initialDelaySeconds: 10
          successThreshold: 1
          failureThreshold: 2 
          periodSeconds: 3
          httpGet:
            port: 80
            scheme: HTTP
            path: /
+        readinessProbe:    
          initialDelaySeconds: 10  
          successThreshold: 1
          failureThreshold: 2 
          periodSeconds: 3 
          httpGet:
            port: 80
            scheme: HTTP
            path: /
```

> 重点：
>
> strategy 策略
>
> resources 资源
>
> livenessProbe 存活探针，失败就重启
>
> readinessProbe 就绪探针，失败就从service去掉
>
> terminationGracePeriodSeconds 回收前宽限，长连接一般90s, 所以宽限90s

##### ingress

```yaml
root@ccea447f1caa:~/configcenter# cat jenkins-pipelines/base/ingress.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress
```

> 每个应用对应一个ingress

##### ns

```diff
root@ccea447f1caa:~/configcenter# cat jenkins-pipelines/base/ns.yaml 
apiVersion: v1
kind: Namespace
metadata:
+  name: test
```

> 名称空间

##### service

```diff
apiVersion: v1
kind: Service
metadata:
  labels:
    app: service
  name: service
  namespace: test
spec:
  ports:
  - name: 80-80
+    port: 80
+    protocol: TCP
+    targetPort: 80
  selector:
    app: deploy-selector
+  type: ClusterIP
```

> port 集群端口
>
> targetport 容器端口
>
> type clusterip, 因为ingress中会直接反代到这个服务，不需要nodeport

##### kustomize

```bash
cd jenkins-pipelines/base/

# Create a new kustomization detecting resources in the current directory.
kustomize create --autodetect
```



#### demoapp发版不同环境的配置

```bash
root@ccea447f1caa:~/configcenter# tree jenkins-pipelines/demoapp/
jenkins-pipelines/demoapp/
├── gray
│   ├── ingress.yaml
│   ├── kustomization.yaml
│   ├── mount-patch.yaml
│   ├── nginx.conf
│   ├── replicas-patch.yaml
│   ├── resource-limit.yaml
│   ├── sa-patch.yaml
│   ├── update-patch.yaml
│   └── var-patch.yaml
├── Jenkinsfile-deploy
└── test
    ├── ingress.yaml
    ├── kustomization.yaml
    ├── mount-patch.yaml
    ├── nginx.conf
    ├── oldingress.yaml
    ├── replicas-patch.yaml
    ├── resource-limit.yaml
    ├── sa-patch.yaml
    ├── update-patch.yaml
    └── var-patch.yaml
```

##### ingress

```diff
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress 
  namespace: default
spec:
  rules:
+  - host: graycms.mykernel.cn
    http:
      paths:
      - backend:
          serviceName: service
          servicePort: 80
        path: /
  tls:
  - hosts:
+    - graycms.mykernel.cn
+    secretName: mykernel.cn

```

> 不同环境的app有不同域名
>
> secretName只需要一个通配的证书即可

##### mount-patch

```diff
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deploy-label
  name: deploy
  namespace: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deploy-selector
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: deploy-selector
    spec:
      volumes:
      - configMap:
          defaultMode: 400
          name: nginx-conf
        name: nginx-config

      containers:
      - name: deploy
+        volumeMounts:
+        - mountPath: /apps/nginx/conf/conf.d/
+          name: nginx-config
```

> 挂载nginx配置

##### nginx.conf

```diff
root@ccea447f1caa:~/configcenter# cat jenkins-pipelines/demoapp/gray/nginx.conf 
server {
+    listen       80 default_server;
    server_name  localhost;
    charset utf-8; 
    root   /apps/nginx/html;
    access_log logs/access.log access_json;
    error_log  logs/error.log  error;

    location / {
        index  index.html index.htm;
        rewrite ^/Home/Index(.*)$ /$1 redirect;
        rewrite ^/login/login(.*)$ /$1 redirect;
        rewrite ^/home/index(.*)$ /$1 redirect;

    }

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

每个环境的nginx配置不一样

```yaml
listen       80 default_server; 表示默认主机
access_log logs/access.log access_json; 表示引用json日志，方便统一收集日志

```

##### replicas

```diff
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deploy-label
  name: deploy
  namespace: test
spec:
+  replicas: 1

```

这个应用几个副本

##### resource

```diff
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deploy-label
  name: deploy
  namespace: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deploy-selector
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: deploy-selector
    spec:
      terminationGracePeriodSeconds: 90
      containers:
      - name: deploy
+        resources:
+          limits:
+            cpu: 100m
+            memory: 50Mi
+          requests:
+            cpu: 100m
+            memory: 50Mi

```

经过测试nginx在cpu为10m 时，qps为300多，cpu为1000m即 `limits.cpu: 1` 时qps为2000多。所以nginx只是占用cpu多，内存不多。1个副本处理200并发 * 每个并发100个请求 = 2万请求

如果在cpu非常低的情况，10m时，qps为300多，我启动replicas为50个时，也能处理200并发 * 每个并发100个请求 = 2万请求

##### sa-patch

```diff
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deploy-label
  name: deploy
  namespace: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deploy-selector
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: deploy-selector
    spec:
      terminationGracePeriodSeconds: 90
+      serviceAccountName: pipeline-serviceaccount
      containers:
      - name: deploy
```

添加sa名称，这样连接镜像的私有认证

##### update-patch

```diff
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deploy-label
  name: deploy
  namespace: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deploy-selector
+  strategy:
+    type: Recreate
```

由于是测试环境和灰度环境，避免资源不够，所以先删除后增加pod

##### var-patch

```diff
root@ccea447f1caa:~/configcenter# cat jenkins-pipelines/demoapp/gray/var-patch.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deploy-label
  name: deploy
  namespace: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deploy-selector
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: deploy-selector
    spec:
      terminationGracePeriodSeconds: 90
      containers:
      - image: ikubernetes/deploy:v1
        name: deploy
        env:
+        - name: PKG_RELEASE
+          value: "1"
        # Define the environment variable
+        - name: ENVIRONMENT
+           valueFrom:
+             fieldRef:
+               fieldPath: metadata.labels['env']
+         - name: CPU_LIMIT
+           valueFrom:
+             resourceFieldRef:
+               resource: limits.cpu
```

> metadata.labels['env'] 这个区别不同的环境
>
> limits.cpu 这个才是nginx work进程可以启动的个数，默认pod中可以看到nginx启动系统进程数，大并发场景有影响

其他变量，属于业务使用

##### kustomize

```diff
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    resources:
    - ../../base
+    namePrefix: saascmsweb-gray-
+    namespace: default
+    commonLabels:
      application: saascmsweb
      env: staging

+    images:
    - name: ikubernetes/deploy
      newName:
      newTag:


+    patchesStrategicMerge:
    - var-patch.yaml
    - sa-patch.yaml
    - mount-patch.yaml
    - resource-limit.yaml
    - ingress.yaml
    - update-patch.yaml
    - replicas-patch.yaml

+    configMapGenerator:
    - name: nginx-conf
      files:
      - nginx.conf
```

> namePrefix 如果test, staging, prod在同一个名称空间，相同的前缀会出问题
>
> namespace 全局名称空间
>
> commonLabels 所有应用添加标签 
>
> images 替换镜像
>
> patchesStrategicMerge 补丁文件列表
>
> configMapGenerator 生成配置文件

#### 准备dotnet的文件

##### base目录

基础目录，检测必须有，终止延迟90s

应用服务器，检测开始设置60s， 一般启动慢

```bash
root@ccea447f1caa:~/configcenter/jenkins-pipelines# cat base.net/deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deploy-label
  name: deploy
  namespace: test
spec:
  selector:
    matchLabels:
      app: deploy-selector
  template:
    metadata:
      labels:
        app: deploy-selector
    spec:
      terminationGracePeriodSeconds: 90
      containers:
      - image: ikubernetes/deploy:v1
        name: deploy
        livenessProbe:   
          initialDelaySeconds: 60
          successThreshold: 1
          failureThreshold: 2 
          periodSeconds: 3
          tcpSocket:
            port: 8000
        readinessProbe:    
          initialDelaySeconds: 60  
          successThreshold: 1
          failureThreshold: 2 
          periodSeconds: 3 
          tcpSocket:
            port: 8000

root@ccea447f1caa:~/configcenter/jenkins-pipelines# cat base.net/ingress.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress
root@ccea447f1caa:~/configcenter/jenkins-pipelines# cat base.net/kustomization.yaml 
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- ns.yaml
- service.yaml
- ingress.yaml
root@ccea447f1caa:~/configcenter/jenkins-pipelines# cat base.net/ns.yaml 
apiVersion: v1
kind: Namespace
metadata:
  name: test
root@ccea447f1caa:~/configcenter/jenkins-pipelines# cat base.net/service.yaml 
apiVersion: v1
kind: Service
metadata:
  labels:
    app: service
  name: service
  namespace: test

```

现在只需要准备test目录，其他目录复制test,修改一下即可

##### test目录

```bash
root@ccea447f1caa:~/configcenter/jenkins-pipelines# cat saasorder/test/ingress.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress 
  namespace: default
spec:
  rules:
  - host: testsaasorder.graspyun.com
    http:
      paths:
      - backend:
          serviceName: service
          servicePort: 8000
        path: /
  tls:
  - hosts:
    - testsaasorder.graspyun.com
    secretName: graspyun.com
root@ccea447f1caa:~/configcenter/jenkins-pipelines# cat saasorder/test/kustomization.yaml 
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base.net
namePrefix: saasorderapi-test-
namespace: default
commonLabels:
  application: saasorderapi
  env: Testing

images:
- name: ikubernetes/deploy
  newName: 
  newTag: 


patchesStrategicMerge:
- var-patch.yaml
- sa-patch.yaml
- resource-limit.yaml
- ingress.yaml
- service-patch.yaml
- port-patch.yaml
- update-patch.yaml
- replicas-patch.yaml
root@ccea447f1caa:~/configcenter/jenkins-pipelines# cat saasorder/test/port-patch.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deploy-label
  name: deploy
  namespace: test
spec:
  template:
    spec:
      containers:
      - name: deploy
        ports:
        - containerPort: 8000
          protocol: TCP
root@ccea447f1caa:~/configcenter/jenkins-pipelines# cat saasorder/test/replicas-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deploy-label
  name: deploy
  namespace: test
spec:
  replicas: 1
root@ccea447f1caa:~/configcenter/jenkins-pipelines# cat saasorder/test/resource-limit.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deploy-label
  name: deploy
  namespace: test
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 90
      containers:
      - name: deploy
        resources:
          limits:
            cpu: 250m
            memory: 512Mi
          requests:
            cpu: 250m
            memory: 512Mi
root@ccea447f1caa:~/configcenter/jenkins-pipelines# cat saasorder/test/sa-patch.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deploy-label
  name: deploy
  namespace: test
spec:
  template:
    metadata:
      labels:
        app: deploy-selector
    spec:
      serviceAccountName: pipeline-serviceaccount
      containers:
      - name: deploy
root@ccea447f1caa:~/configcenter/jenkins-pipelines# cat saasorder/test/service-patch.yaml 
apiVersion: v1
kind: Service
metadata:
  labels:
    app: service
  name: service
  namespace: test
spec:
  ports:
  - name: "8000-8000"
    port: 8000
    protocol: TCP
    targetPort: 8000
  selector:
    app: deploy-selector
  type: ClusterIP

root@ccea447f1caa:~/configcenter/jenkins-pipelines# cat saasorder/test/update-patch.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deploy-label
  name: deploy
  namespace: test
spec:
  replicas: 1
  strategy:
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
    type: RollingUpdate
root@ccea447f1caa:~/configcenter/jenkins-pipelines# cat saasorder/test/var-patch.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deploy-label
  name: deploy
  namespace: test
spec:
  template:
    spec:
      containers:
      - name: deploy
        env:
        - name: ASPNETCORE_URLS
          value: http://+:80
        - name: DOTNET_RUNNING_IN_CONTAINER
          value: "true"
        - name: LANG
          value: C.UTF-8
        # Define the environment variable
        - name: ASPNETCORE_ENVIRONMENT
          valueFrom:
            fieldRef:
              fieldPath: metadata.labels['env']
```

##### gray目录

```diff
root@ccea447f1caa:~/configcenter/jenkins-pipelines# diff -u saasorder/gray/ saasorder/test/
diff -u saasorder/gray/ingress.yaml saasorder/test/ingress.yaml
--- saasorder/gray/ingress.yaml	2021-05-13 09:06:03.663365000 +0000
+++ saasorder/test/ingress.yaml	2021-05-13 09:06:09.918365000 +0000
@@ -5,7 +5,7 @@
   namespace: default
 spec:
   rules:
-  - host: graysaasorder.graspyun.com
+  - host: testsaasorder.graspyun.com
     http:
       paths:
       - backend:
@@ -14,5 +14,5 @@
         path: /
   tls:
   - hosts:
-    - graysaasorder.graspyun.com
+    - testsaasorder.graspyun.com
     secretName: graspyun.com
diff -u saasorder/gray/kustomization.yaml saasorder/test/kustomization.yaml
--- saasorder/gray/kustomization.yaml	2021-05-13 09:02:26.637800000 +0000
+++ saasorder/test/kustomization.yaml	2021-05-13 08:50:51.127965000 +0000
@@ -2,11 +2,11 @@
 kind: Kustomization
 resources:
 - ../../base.net
-namePrefix: saasorderapi-gray-
+namePrefix: saasorderapi-test-
 namespace: default
 commonLabels:
   application: saasorderapi
-  env: Staging
+  env: Testing
 
 images:
 - name: ikubernetes/deploy

```

##### 生产目录

生产变更资源、副本、更新策略

```diff
root@ccea447f1caa:~/configcenter/jenkins-pipelines# diff -u saasorder/prod/ saasorder/test/
diff -u saasorder/prod/ingress.yaml saasorder/test/ingress.yaml
--- saasorder/prod/ingress.yaml	2021-05-13 09:11:33.622900000 +0000
+++ saasorder/test/ingress.yaml	2021-05-13 09:06:09.918365000 +0000
@@ -5,7 +5,7 @@
   namespace: default
 spec:
   rules:
-  - host: saasorder-2.graspyun.com
+  - host: testsaasorder.graspyun.com
     http:
       paths:
       - backend:
@@ -14,5 +14,5 @@
         path: /
   tls:
   - hosts:
-    - saasorder-2.graspyun.com
+    - testsaasorder.graspyun.com
     secretName: graspyun.com
diff -u saasorder/prod/kustomization.yaml saasorder/test/kustomization.yaml
--- saasorder/prod/kustomization.yaml	2021-05-13 09:02:38.455313000 +0000
+++ saasorder/test/kustomization.yaml	2021-05-13 08:50:51.127965000 +0000
@@ -2,11 +2,11 @@
 kind: Kustomization
 resources:
 - ../../base.net
-namePrefix: saasorderapi-prod-
+namePrefix: saasorderapi-test-
 namespace: default
 commonLabels:
   application: saasorderapi
-  env: Production
+  env: Testing
 
 images:
 - name: ikubernetes/deploy
只在 saasorder/test/ 存在：oldingress.yaml
diff -u saasorder/prod/replicas-patch.yaml saasorder/test/replicas-patch.yaml
--- saasorder/prod/replicas-patch.yaml	2021-05-13 09:10:14.337321000 +0000
+++ saasorder/test/replicas-patch.yaml	2021-05-13 08:41:14.668995000 +0000
@@ -6,4 +6,4 @@
   name: deploy
   namespace: test
 spec:
-  replicas: 2
+  replicas: 1
diff -u saasorder/prod/update-patch.yaml saasorder/test/update-patch.yaml
--- saasorder/prod/update-patch.yaml	2021-05-13 09:10:30.733907000 +0000
+++ saasorder/test/update-patch.yaml	2021-05-13 08:41:43.396849000 +0000
@@ -9,6 +9,6 @@
   replicas: 1
   strategy:
     rollingUpdate:
-      maxSurge: 1
-      maxUnavailable: 0
+      maxSurge: 0
+      maxUnavailable: 1
     type: RollingUpdate
```

##### 将发布脚本修正

```bash
REPO_BASE_NAME="saas" # 和ci脚本一样
 GITHTTPURL="http://18spy.git" # git地址
 useRepository:  # git地址
```



### 准备检查发布成功的脚本

```bash
root@ccea447f1caa:~/configcenter# cat jenkins-pipelines/check_deploy_status.sh 



# 来自jenkins的环境变量
cd ${JOB_NAME}/${deploy_env}
deploy=$(kustomize build  | sed -n '/Deployment/,/spec/p' | awk -F'name:' '/name:/{print $2}')
ns=$(kustomize build  | sed -n '/Deployment/,/spec/p' | awk -F'namespace:' '/namespace:/{print $2}')
kubectl rollout status --timeout=120s deploy $deploy  -n $ns
```

发布需要时间，检测也需要。nginx快，应用一般1分钟

### 准备回滚脚本

```bash
root@ccea447f1caa:~/configcenter# cat jenkins-pipelines/rollout.sh 



# 来自jenkins的环境变量
cd ${JOB_NAME}/${deploy_env}
deploy=$(kustomize build  | sed -n '/Deployment/,/spec/p' | awk -F'name:' '/name:/{print $2}')
ns=$(kustomize build  | sed -n '/Deployment/,/spec/p' | awk -F'namespace:' '/namespace:/{print $2}')
kubectl rollout undo deploy $deploy  -n $ns
kubectl rollout status --timeout=120s deploy $deploy  -n $ns
```

## 生产环境发布

仅需要修改 `REPO_BASE_NAME="saasagentweb"` ，只需要知道仓库名，每次仅发gray-latest分支

```groovy
// AUTHOR: songliangcheng
// DESC：插件
// jenkins对接kubernets: https://plugins.jenkins.io/kubernetes/ , 文中搜索：Declarative Pipeline
// pipeline使用构建者名称：https://wiki.jenkins.io/display/JENKINS/Build+User+Vars+Plugin
// pipeline的 git参数化构建：https://plugins.jenkins.io/git-parameter/

// pipeline的gitlab自动触发：https://plugins.jenkins.io/gitlab-plugin/， 支持的变量：Defined variables，引用变量: ${env.gitlabSourceBranch}
    // 自由或pipeline在见面中定义。多分支pipeline可以yaml定义。
// pipeline钉钉告警：https://plugins.jenkins.io/dingding-notifications/
// docker piepline插件，docker相关的插件

// 示例：https://github.com/jenkinsci/pipeline-examples/tree/master/declarative-examples/simple-examples



pipeline {

    agent {
        // jenkins对接kubernets
        // kubernetes的pod模板，不需要在jenkins中二次定义. 必须有nodeSelector, 否则不会调度
        kubernetes {
          yaml """\
            apiVersion: v1
            kind: Pod
            metadata:
              namespace: jenkins
              labels:
                some-label: some-label-value
            spec:
              nodeSelector:
                kubernetes.io/os: linux
              serviceAccountName: jenkins-build-sa
              containers:
            
              - name: docker
                image: docker:19.03.12
                command:
                - cat
                tty: true
                env:
                - name: LANG
                  value: zh_CN.UTF-8
                - name: TZ
                  value: Asia/Shanghai
                securityContext:
                  privileged: true
                volumeMounts:
                  - name: dind-storage
                    mountPath: /var/run/docker.sock
              - name: sonar
                image: sonarsource/sonar-scanner-cli:4.6
                command:
                - cat
                tty: true
                env:
                - name: LANG
                  value: zh_CN.UTF-8
                - name: TZ
                  value: Asia/Shanghai
                volumeMounts:
                  - name: sonar-storage
                    mountPath:  /opt/sonar-scanner/.sonar/cache
              - name: jq
                image: registry.cn-hangzhou.aliyuncs.com/grasp_base_images/ubuntu-jq-curl:latest
                command:
                - cat
                tty: true
                env:
                - name: LANG
                  value: zh_CN.UTF-8
                - name: TZ
                  value: Asia/Shanghai
              - name: mysql
                image: registry.cn-hangzhou.aliyuncs.com/grasp_base_images/mysql:latest
                command:
                - cat
                tty: true
                env:
                - name: LANG
                  value: zh_CN.UTF-8
                - name: TZ
                  value: Asia/Shanghai

              - name: kubectl
                image: registry.cn-hangzhou.aliyuncs.com/grasp_base_images/k8s-test:latest
                command:
                - cat
                tty: true

              volumes:
              - name: dind-storage
                hostPath:
                  path: /var/run/docker.sock
              
              - name: sonar-storage
                hostPath:
                  path: /opt/sonar-scanner/.sonar/cache
                  type: DirectoryOrCreate
            """.stripIndent()
        }
    }
    // jenkins正常指令
    environment {
        PROTOCOL="https://"
        REPOSITORY="registry.cn-hangzhou.aliyuncs.com/grasp_base_images"
        REPO_BASE_NAME="saasagentweb"
        // git登陆 (参数化、非参数化均需要使用)
        GITCREDENTIAL="gitlab-documents-user-pass"
        // sonarqube 
        CRED = credentials('sonarqube') 
        // jenkins-self
        JENKINS_CRED = credentials('jenkins-self') 


        //mysql
        MYSQL_CRED = credentials('jenkins-to-mysql')

        // DINGDING ID
        DINGDINGID = 'd1b9b632-cfbb-4ca9-966a-2c9534e14b59'

        // 部署状态
        DEPLOY_STATUS="FAILURE"


    }

    // jenkins的git参数化构建
    parameters {
        // jenkins参数化构建
        choice(name: 'deploy_env', choices: ['prod'],description: 'prod 生产环境')         
        choice(name: 'deploy_method', choices: ['deploy','rollout'],description: 'deploy 部署\nrollout 回滚')         
    }

    // pipeline正常指令
    stages {
        // pipeline正常指令，一个stages里面多个stage, 每个stage里面1个steps {}

        stage('发布前报警') {
            steps {


               // 构建用户信息：id, 名，email
                wrap([$class: 'BuildUser']) {
                   script {
                       BUILD_USER_ID = "${env.BUILD_USER_ID}"
                       BUILD_USER = "${env.BUILD_USER}"
                       BUILD_USER_EMAIL = "${env.BUILD_USER_EMAIL}"
                   }
                 }


                container('jq') {
                  script {
                    CURTIME = sh(script: "date +%Y%m%d%H%M%S", returnStdout: true).trim()
                    DCURTIME = sh(script: "date +'%Y年%m月%d日 %H时%M分%S秒'", returnStdout: true).trim()
                  }
                }

                 container('jq') {
                    dingtalk (
                            robot: "${DINGDINGID}",
                            // robot: 'd1b9b632-cfbb-4ca9-966a-2c9534e14b59',
                            type: 'MARKDOWN',
                            title: '你有新的消息，请注意查收',
                            text: [
                                " ${env.JOB_NAME} 开始${deploy_method} ${deploy_env} [${currentBuild.number}任务](${currentBuild.absoluteUrl}console) ${currentBuild.durationString}",
                                "---",
                                "- 执行人:  ${BUILD_USER}",
                                "- [Jenkins job入口](https://jenkins.graspyun.com//job/${env.JOB_NAME}/)",
                                "- [SonarQube代码质量入口](http://112.124.37.17:8599)",
                                "---",
                                "> ${DCURTIME}\n",
                                "> 如有疑问，请联系： 运维-宋亮成<songliangcheng@graspyun.com>"
                            ],
                            "atAll": false
                        )
                }

            }
        }



      // 升级镜像
      stage("环境升级镜像并发布") {
        steps {

          container('docker') {
        
          script {
              if (deploy_method == "deploy") {
                  docker.withRegistry("${PROTOCOL}${REPOSITORY}/${REPO_BASE_NAME}", 'docker-kubernetes') {
                    customImage = docker.image("${REPOSITORY}/${REPO_BASE_NAME}:gray-latest")

                    customImage.pull()
                    customImage.push("${deploy_env}-${CURTIME}")
                  }

                  sh "docker rmi ${REPOSITORY}/${REPO_BASE_NAME}:gray-latest"
              }

            } 
        


          script {
            container('kubectl') {
                checkout([$class: 'GitSCM', 
                          branches: [[name: "master"]], 
                          doGenerateSubmoduleConfigurations: false, 
                          extensions: [], 
                          gitTool: 'Default', 
                          submoduleCfg: [], 
                          userRemoteConfigs: [[url: "https://gitlab.mykernel.cn/songliangcheng/configcenter.git",credentialsId: "${GITCREDENTIAL}",]]
                        ])

                // 准备测试的k8s的config
                sh "cp jenkins-pipelines/kubernetes-kubeconfig/${deploy_env}-config  ~/.kube/config"

                // 测试发版
                script {
                  if (deploy_method == "deploy") {
                    if (deploy_env == "prod"){
                      JOB_NAME=sh(script: "echo ${JOB_NAME} | grep -E -o '^[^-]+'",returnStdout:true).trim()
                    }
                    sh """
                    cd  jenkins-pipelines/${JOB_NAME}/${deploy_env} && sed -i 's@newName:.*@newName: ${REPOSITORY}/${REPO_BASE_NAME}@g' kustomization.yaml && sed  -i 's@newTag:.*@newTag: ${deploy_env}-${CURTIME}@g' kustomization.yaml  && kustomize build  > jenkins_${env.JOB_NAME}_${currentBuild.number}.yaml && kubectl apply -f jenkins_${env.JOB_NAME}_${currentBuild.number}.yaml  --record && cd ../../ &&  bash check_deploy_status.sh
                    """
                    if (deploy_env == "prod"){
                      JOB_NAME="${JOB_NAME}-prod"
                    }
                    // 如果上面返回非0，jenkins自动退出。
                    // 如果返回0，继续走
                    DEPLOY_STATUS="OK"
                  }
                  else if (deploy_method == "rollout") {
                    sh "cd  jenkins-pipelines && bash rollout.sh"
                  }
                }
            }
          }

        }

      }


    }
  }

    post {
        always {

            container('jq') {


              // 钉钉的告警
                    script {
                        // CURTIME = sh(script: "date +%Y%m%d%H%M%S", returnStdout: true).trim()
                        DCURTIME = sh(script: "date +'%Y年%m月%d日 %H时%M分%S秒'", returnStdout: true).trim()
                        NOWTIME  = sh(script: "date +'%F %H:%M'", returnStdout: true).trim()
                    }


               script {
                  echo 'deploy ======='
                 dingtalk (
                          robot: "${DINGDINGID}",
                          // robot: 'd1b9b632-cfbb-4ca9-966a-2c9534e14b59',
                          type: 'MARKDOWN',
                          title: '你有新的消息，请注意查收',
                          text: [
                              " ${env.JOB_NAME} ${deploy_method} ${deploy_env} 完成 [${currentBuild.number}任务](${currentBuild.absoluteUrl}console) ${currentBuild.durationString}",
                              "- 发布状态: ${DEPLOY_STATUS}",
                              "---",
                              "- [Jenkins job入口](https://jenkins.graspyun.com//job/${env.JOB_NAME}/)",
                              "- [SonarQube代码质量入口](http://112.124.37.17:8599)",
                              "---",
                              "> ${DCURTIME}\n 对应镜像编号:${deploy_env}-${CURTIME}\n",
                              "> 如有疑问，请联系： 运维-宋亮成<songliangcheng@graspyun.com>"
                          ],
                          "atAll": false
                      )
               }



            }

            container('mysql') {
              script {

    
                LANG='zh_CN.UTF-8'
                TZ='Asia/Shanghai'
                NOWTIME  = sh(script: "date +'%F %H:%M'", returnStdout: true).trim()
                sh script: "mysql -u${MYSQL_CRED_USR} -p${MYSQL_CRED_PSW} -P8600 -h112.124.37.17 -e \"insert into jenkins_webhook.jenkins(createtime,modifytime,creator,builduser,buildemail,node,taskurl,imageinfo,buildduration,jobname,jobnum) values ('${NOWTIME}','${NOWTIME}','jenkins','${BUILD_USER}','${BUILD_USER_EMAIL}','${env.NODE_NAME}','${currentBuild.absoluteUrl}console','${DEPLOY_STATUS}','${currentBuild.durationString}','${env.JOB_NAME}','${currentBuild.number}')\"",returnStdout:true
            }

        }

    }
}
}


// CREATE TABLE `jenkins` (
//   `id` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '与业务无关的id',
//   `createtime` datetime NOT NULL COMMENT '创建时间',
//   `modifytime` datetime NOT NULL DEFAULT '0000-00-00 00:00:00' ON UPDATE current_timestamp() COMMENT '修改时间',
//   `creator` varchar(255) DEFAULT NULL COMMENT '创建者',
//   `buildbranch` varchar(255) DEFAULT NULL COMMENT '构建分支',
//   `builduser` varchar(255) DEFAULT NULL COMMENT '构建者用户',
//   `buildemail` varchar(255) DEFAULT NULL COMMENT '构建者邮箱',
//   `buildcommit` text DEFAULT NULL COMMENT '提交信息',
//   `node` varchar(255) DEFAULT '' COMMENT '执行节点',
//   `taskurl` varchar(255) DEFAULT NULL COMMENT '任务url',
//   `codescan` varchar(255) DEFAULT NULL COMMENT '代码质量url',
//   `imageinfo` varchar(255) DEFAULT NULL COMMENT '镜像信息',
//   `sonarcodescanresult` varchar(255) DEFAULT NULL COMMENT '执行sonar扫描的结果，成功或失败',
//   `sonarresult` varchar(255) DEFAULT NULL COMMENT '代码质量，sonarqube质量都达标时 成功',
//   `currentResult` varchar(255) DEFAULT NULL COMMENT 'Jenkins构建结果 ',
//   `jobname` varchar(255) DEFAULT NULL COMMENT 'jenkins构建的job名',
//   `buildduration` varchar(255) DEFAULT NULL COMMENT 'jenkins构建时长',
//   `githash` varchar(255) DEFAULT NULL COMMENT 'git提交的hash',
//   `jobnum` int(11) DEFAULT NULL COMMENT 'job的构建号',
//   PRIMARY KEY (`id`)
// ) ENGINE=InnoDB AUTO_INCREMENT=47 DEFAULT CHARSET=utf8mb4;
```







