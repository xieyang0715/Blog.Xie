---
title: harbor主从复制
date: 2021-03-24 06:03:37
tags:
- harbor
- docker-compose
---


# 环境

192.168.0.83是harbor01,    192.168.0.84是harbor02

每个harbor的域名是 harbor.youwoyouqu.io

进行复制时，harbor02会通过配置的主机名去认证，去拉镜像，所以在harbor中需要有这个域名的解析记录。

# 基于之前搭建的harbor

## 解压

```bash
scp harbor-offline-installer-v2.2.0.tgz 192.168.0.84:
tar xvf harbor-offline-installer-v2.2.0.tgz -C /opt/
```

## 复制配置

```bash
scp /opt/harbor/harbor.yml 192.168.0.84:/opt/harbor/
scp -rp /opt/cert/ 192.168.0.84:/opt
```

## 启动

```bash
cd /opt/harbor
./install.sh
```

编辑docker-compose.yaml

```bash
#在每个container_name: 同级添加可以解析harbor.yml的host字段的域名的dns
	dns:
	- 192.168.0.77     # 此处添加能解析harbor.yml中的host的域名的dns
```

```bash
docker-compose kill 
docker-compose rm -f
docker-compose up -d
```



# 界面配置复制 

## haproxy下线84

```bash
 88 listen harbor-443
 89     bind 192.168.0.249:443
 90     mode tcp
 91     server 192.168.0.83  192.168.0.83:443 check inter 3s fall 2 rise 5                                                                                                                                                                                                                                                 
 92     #server 192.168.0.84  192.168.0.84:443 check inter 3s fall 2 rise 5   


root@haproxy01:~# systemctl restart haproxy
```



## 84拉83

### 配置拉单个库

![image-20210324140815792](http://myapp.img.mykernel.cn/image-20210324140815792.png)

复制管理 - 添加规则，由于之前83已经有生产运行一段时间的镜像，所以先拉过来。

注意：先测试拉一个库，并且触发为手动

![image-20210324145718589](http://myapp.img.mykernel.cn/image-20210324145718589.png)





手工触发复制

![image-20210324150128994](http://myapp.img.mykernel.cn/image-20210324150128994.png)

点162查看进程，由于网速较快，已经看到完成一个。

![image-20210324150156863](http://myapp.img.mykernel.cn/image-20210324150156863.png)

### 配置拉所有库

#### 手工

先让触发为手动，后期同步完了，再修改定时，不然在同步周期内同步不完，会一直启动进程

![image-20210324150504903](http://myapp.img.mykernel.cn/image-20210324150504903.png)

![image-20210324151946113](http://myapp.img.mykernel.cn/image-20210324151946113.png)

#### 现在才配置自动

![image-20210324152203058](http://myapp.img.mykernel.cn/image-20210324152203058.png)

## 83拉84

一样的新建仓库，添加一个复制规则



### compose配置

配置83的docker-compose和84一致

```bash
root@harbor01:~# scp 192.168.0.84:/opt/harbor/docker-compose.yml     /opt/harbor/
root@harbor01:~# cd /opt/harbor/
root@harbor01:/opt/harbor# docker-compose kill
root@harbor01:/opt/harbor# docker-compose rm -f
Going to remove harbor-jobservice, nginx, harbor-core, redis, registry, registryctl, harbor-db, harbor-portal, harbor-log
Removing harbor-jobservice ... done
Removing nginx             ... done
Removing harbor-core       ... done
Removing redis             ... done
Removing registry          ... done
Removing registryctl       ... done
Removing harbor-db         ... done
Removing harbor-portal     ... done
Removing harbor-log        ... done
root@harbor01:/opt/harbor# docker-compose up -d
```

### haproxy切换镜像仓库为84

```bash
 88 listen harbor-443
 89     bind 192.168.0.249:443
 90     mode tcp
 91     #server 192.168.0.83  192.168.0.83:443 check inter 3s fall 2 rise 5                                                                                                                                                                                                                                                
 92     server 192.168.0.84  192.168.0.84:443 check inter 3s fall 2 rise 5   
```

```bash
root@haproxy01:~# systemctl restart haproxy
```

正常推镜像

```bash
root@master01:/data/weizhixiu/yaml/prometheus# docker push harbor.youwoyouqu.io/pub/grafana:7.4.3
The push refers to repository [harbor.youwoyouqu.io/pub/grafana]
31e37bf28d35: Layer already exists 
0448b810b7f3: Layer already exists 
0835354182ac: Layer already exists 
8f0f1539b4c9: Layer already exists 
ff51b7928895: Layer already exists 
af0dc88d00a2: Layer already exists 
b76036dc8ce6: Layer already exists 
cb381a32b229: Layer already exists 
7.4.3: digest: sha256:0baac55c486161598f0d263cd5c148cb45beeff0a311c7ee82800f7250fd4338 size: 2004

```

### 网页复制

![image-20210324152902355](http://myapp.img.mykernel.cn/image-20210324152902355.png)



### 添加规则

![image-20210324154118077](http://myapp.img.mykernel.cn/image-20210324154118077.png)



## 测试推送镜像

### haproxy指向

指向84

![image-20210324154403359](http://myapp.img.mykernel.cn/image-20210324154403359.png)

### 验证83没有镜像 pub/test

![image-20210324154339940](http://myapp.img.mykernel.cn/image-20210324154339940.png)

### 推送至镜像

```bash
root@master01:/data/weizhixiu/yaml/prometheus# docker tag harbor.youwoyouqu.io/pub/grafana:7.4.3  harbor.youwoyouqu.io/pub/test:20210324
root@master01:/data/weizhixiu/yaml/prometheus# docker push harbor.youwoyouqu.io/pub/test:20210324
The push refers to repository [harbor.youwoyouqu.io/pub/test]
31e37bf28d35: Mounted from pub/grafana 
0448b810b7f3: Mounted from pub/grafana 
0835354182ac: Mounted from pub/grafana 
8f0f1539b4c9: Mounted from pub/grafana 
ff51b7928895: Mounted from pub/grafana 
af0dc88d00a2: Mounted from pub/grafana 
b76036dc8ce6: Mounted from pub/grafana 
cb381a32b229: Mounted from pub/grafana 
20210324: digest: sha256:0baac55c486161598f0d263cd5c148cb45beeff0a311c7ee82800f7250fd4338 size: 2004
```

### 验证84仓库

![image-20210324154516816](http://myapp.img.mykernel.cn/image-20210324154516816.png)

### 验证83仓库

会有一定延迟，这个是因为同步是1min一次的原因

![image-20210324154557130](http://myapp.img.mykernel.cn/image-20210324154557130.png)



## 配置haproxy

### 主备haproxy

由于推送有延迟，所以不建议直接负载均衡，所以haproxy上面使用主备即可

```diff
listen harbor-443 
    bind 192.168.0.249:443
    mode tcp
+    server 192.168.0.83  192.168.0.83:443 check inter 3s fall 2 rise 5  backup
    server 192.168.0.84  192.168.0.84:443 check inter 3s fall 2 rise 5
```

```bash
root@haproxy01:~# systemctl restart haproxy
```

![image-20210324154907327](http://myapp.img.mykernel.cn/image-20210324154907327.png)



### rr轮循

这个harbor同步有一定延迟，直接负载均衡有可能在构建镜像推送数据后，在slave同步master完成前, 负载均衡的效果会有访问不到资源的情况，一般推送镜像后，就是更新kubernetes api, 各个kubelet，就算拉不到镜像，一会会重新尝试拉镜像，所以不存在问题的。

```bash
listen harbor-443 
    bind 192.168.0.249:443
    mode tcp
    server 192.168.0.83  192.168.0.83:443 check inter 3s fall 2 rise 5  
    server 192.168.0.84  192.168.0.84:443 check inter 3s fall 2 rise 5
```

```bash
root@haproxy01:~# systemctl restart haproxy
```

![image-20210324155345011](http://myapp.img.mykernel.cn/image-20210324155345011.png)