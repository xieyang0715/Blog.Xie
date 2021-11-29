---
title: rabbitmq使用示例
date: 2021-10-26 14:36:00
tags:
- rabbitmq
---

# 前言

https://www.rabbitmq.com/

rabbitmq是最广泛部署的开源的消息代理(broker), 拥有上万的用户，全世界范围内小公司和大企业均会使用rabbitmq.



rabbitmq很轻量、易于部署于云环境中。它支持多种协议。它可以分布式和联邦配置部署，来满足高可用、高扩展的需要。

rabbitmq可以运行在多种操作系统上和云环境中，并且为众多流行的开发语言提供了广泛的开发者工具。

看看别人如何使用rabbitmq: https://content.pivotal.io/rabbitmq



<!--more-->
# Downloading and Installing RabbitMQ

通过：[rabbitmq支持的时间线](https://www.rabbitmq.com/versions.html),  在明年1月31就停止支持发布补丁。故采用3.9.8

|         |                                                              |                                                              |                         |                          |                |
| :------ | :----------------------------------------------------------- | :----------------------------------------------------------- | :---------------------- | :----------------------- | :------------- |
| Version | Latest Patch                                                 | First Release                                                | End of General Support1 | End of Extended Support2 | In service for |
| 3.9     | [3.9.8](https://github.com/rabbitmq/rabbitmq-server/releases/tag/v3.9.8) | [26 July 2021](https://github.com/rabbitmq/rabbitmq-server/releases/tag/v3.9.0) |                         |                          |                |
| 3.8     | [3.8.23](https://github.com/rabbitmq/rabbitmq-server/releases/tag/v3.8.23) | [1 October 2019](https://github.com/rabbitmq/rabbitmq-server/releases/tag/v3.8.0) | 31 January 2022         |                          |                |

git: https://github.com/rabbitmq/rabbitmq-server/releases/tag/v3.9.8

依赖: 

- Erlang 23.2+



部署方式

- docker  
- linux
- windows
- mac
- RabbitMQ Cluster Kubernetes Operator
- RabbitMQ Topology Kubernetes Operator

```dockerfile
FROM ubuntu:18.04

# 安装ERLANG 23.2+ https://www.rabbitmq.com/install-debian.html#apt-cloudsmith
#RUN apt-get install gpg curl gnupg apt-transport-https -y
RUN sed -i  -e 's@archive.ubuntu.com@mirrors.aliyun.com@g' -e 's@security.ubuntu.com@mirrors.aliyun.com@g' /etc/apt/sources.list && \
	apt update && \
	apt-get install curl gnupg apt-transport-https -y

## Team RabbitMQ's main signing key
RUN curl -1sLf "https://keys.openpgp.org/vks/v1/by-fingerprint/0A9AF2115F4687BD29803A206B73A36E6026DFCA" |  gpg --dearmor |  tee /usr/share/keyrings/com.rabbitmq.team.gpg > /dev/null
## Cloudsmith: modern Erlang repository
RUN curl -1sLf https://dl.cloudsmith.io/public/rabbitmq/rabbitmq-erlang/gpg.E495BB49CC4BBE5B.key |  gpg --dearmor |  tee /usr/share/keyrings/io.cloudsmith.rabbitmq.E495BB49CC4BBE5B.gpg > /dev/null
## Cloudsmith: RabbitMQ repository
RUN curl -1sLf https://dl.cloudsmith.io/public/rabbitmq/rabbitmq-server/gpg.9F4587F226208342.key |  gpg --dearmor |  tee /usr/share/keyrings/io.cloudsmith.rabbitmq.9F4587F226208342.gpg > /dev/null

## Add apt repositories maintained by Team RabbitMQ
copy rabbitmq.list  /etc/apt/sources.list.d/rabbitmq.list
## Update package indices
RUN apt-get update -y

## Install Erlang packages
RUN apt-get install -y erlang-base \
                        erlang-asn1 erlang-crypto erlang-eldap erlang-ftp erlang-inets \
                        erlang-mnesia erlang-os-mon erlang-parsetools erlang-public-key \
                        erlang-runtime-tools erlang-snmp erlang-ssl \
                        erlang-syntax-tools erlang-tftp erlang-tools erlang-xmerl

## 语言
RUN apt-get install -y locales  \
    && localedef -i zh_CN -c -f UTF-8 -A /usr/share/locale/locale.alias zh_CN.UTF-8

ENV LANG zh_CN.utf8

## Install rabbitmq-server and its dependencies
#RUN apt install rabbitmq-server=3.9.8-1 -y --fix-missing
ADD     rabbitmq-server-generic-unix-3.9.8.tar.xz /


## 安装网络工具
RUN apt install net-tools -y

## 清理不需要的包
RUN  apt autoremove  -y &&  rm -rf /var/lib/apt/lists/* 

## 添加环境变量
ENV PATH="/rabbitmq_server-3.9.8/sbin:${PATH}"

## 添加插件
RUN rabbitmq-plugins enable rabbitmq_mqtt rabbitmq_management


## web management ,  MQTT client lib connect port.
EXPOSE 15672 
EXPOSE 1883


COPY enabled_plugins /rabbitmq_server-3.9.8/etc/rabbitmq/enabled_plugins
COPY start.sh start.sh

ENTRYPOINT [ "./start.sh" ]
```

`enabled_plugins`

```bash
[rabbitmq_management,rabbitmq_mqtt].
```

`start.sh`

```bash
#!/bin/bash -x

set -e

(
count=0;
# Execute list_users until service is up and running
until timeout 5 rabbitmqctl list_users >/dev/null 2>/dev/null || (( count++ >= 60 )); do sleep 1; done;
   if rabbitmqctl list_users | grep guest > /dev/null
   then
      # Delete default user and create new users
      rabbitmqctl delete_user guest
      # Specific commands for Application
      rabbitmqctl add_user $RABBITMQ_USER $RABBITMQ_PASSWORD 2>/dev/null ; \
      rabbitmqctl set_user_tags $RABBITMQ_USER administrator ; \
      rabbitmqctl set_permissions -p / $RABBITMQ_USER  ".*" ".*" ".*" ; \

      # add virtualhost and create new uesrs 
      rabbitmqctl add_vhost $RABBITMQ_MQTT_VHOST; \
      rabbitmqctl add_user $RABBITMQ_MQTT_USER $RABBITMQ_MQTT_PASSWORD; \
      rabbitmqctl set_user_tags $RABBITMQ_MQTT_USER management; \
      rabbitmqctl set_permissions -p $RABBITMQ_MQTT_VHOST $RABBITMQ_MQTT_USER ".*" ".*" ".*" ;  \

      echo "setup completed"
   else
      echo "already setup"
   fi
) &

exec /rabbitmq_server-3.9.8/sbin/rabbitmq-server
```

构建镜像

```bash
docker build -t registry.cn-hangzhou.aliyuncs.com/ishop-pos/rabbitmq -t rabbitmq ./
docker push registry.cn-hangzhou.aliyuncs.com/ishop-pos/rabbitmq
```



# 配置rabbitmq

https://www.rabbitmq.com/install-debian.html#configuration

## 资源需求

节点至少保证任何时间都有256MB可用。

建议：`vm_memory_high_watermark.relative` 0.4-0.7 40%-70%内存可用，当高于70%时，应该需要稳定的 [memory usage](https://www.rabbitmq.com/memory-use.html) 和基础设施的[monitoring](https://www.rabbitmq.com/monitoring.html) 。

`disk_free_limit.relative = 1.0 `磁盘空间必须大于内存空间

`disk_free_limit.relative = 2.0` 磁盘空间必须大于2倍内存空间，这个最建议。

确保至少5万的文件描述符对rabbitmq可用。 https://www.rabbitmq.com/install-debian.html#kernel-resource-limits

```bash
docker run --rm --name rq -it rabbitmq
docker exec rq rabbitmqctl status
File Descriptors

Total: 2, limit: 1048479
Sockets: 0, limit: 943629

cat /proc/1/limits
Limit                     Soft Limit           Hard Limit           Units
Max cpu time              unlimited            unlimited            seconds
Max file size             unlimited            unlimited            bytes
Max data size             unlimited            unlimited            bytes
Max stack size            8388608              unlimited            bytes
Max core file size        0                    unlimited            bytes
Max resident set          unlimited            unlimited            bytes
Max processes             unlimited            unlimited            processes
Max open files            1048576              1048576              files
Max locked memory         65536                65536                bytes
Max address space         unlimited            unlimited            bytes
Max file locks            unlimited            unlimited            locks
Max pending signals       15595                15595                signals
Max msgqueue size         819200               819200               bytes
Max nice priority         0                    0
Max realtime priority     0                    0
Max realtime timeout      unlimited            unlimited            us
```

## 日志收集

统一收集处理rabbitmq日志, 非常建议

## 监控

当应用程序无限制的打开rabbitmq连接 ，而不关闭，就会导致新的连接不能处理，特别是在非常复杂的场景中，很难定位这样的问题，所以需要监控rabbitmq.

> channel leak, 连接池的文件描述符使用，...

## 连接管理

rabbitmq会通过心跳检测连接是否还在用，如果心跳超时过低(小于5s), 可能因为网络拥塞或系统负载上升导致误判。

## 防火墙配置

[Ports used by RabbitMQ](https://www.rabbitmq.com/networking.html#ports) 使用2种端口

- 客户端库连接的端口(AMQP 0-9-1, AMQP 1.0, MQTT, STOMP, HTTP API)
- 其他端口（节点间通信 、cli工具通信的端口）

## 网络配置

### 端口配置

https://www.rabbitmq.com/networking.html#interfaces

- 1883, 8883: [MQTT clients](http://mqtt.org/) without and with TLS, if the [MQTT plugin](https://www.rabbitmq.com/mqtt.html) is enabled

## 插件管理 

https://www.rabbitmq.com/rabbitmq-plugins.8.html

## 环境变量配置

https://www.rabbitmq.com/configure.html#supported-environment-variables

## web管理 

https://www.rabbitmq.com/management.html#getting-started

```bash
# create a user
rabbitmqctl add_user root root_pass
# tag the user with "administrator" for full management UI and HTTP API access
rabbitmqctl set_user_tags root administrator
```

![image-20211029114442476](http://myapp.img.mykernel.cn/image-20211029114442476.png)

# mqtt

https://www.rabbitmq.com/mqtt.html

```bash
rabbitmq-plugins enable rabbitmq_mqtt
```

授权, mqtt-test用户，对默认的虚拟主机有完全的访问控制权限。

```bash
# username and password are both "mqtt-test"
rabbitmqctl add_user mqtt-admin mqtt-admin_pass
rabbitmqctl set_permissions -p / mqtt-admin ".*" ".*" ".*"
rabbitmqctl set_user_tags mqtt-admin management
```

```bash
 docker run --rm --name rq  -e RABBITMQ_NODENAME=rabbitmq-node01 -e RABBITMQ_MNESIA_DIR=/data/rabbitmq -v G:/data/rabbitmq:/data/rabbitmq -e RABBITMQ_LOGS=- -p 15672:15672 -p 18883:1883 -it rabbitmq
```

配置文件 :  `/rabbitmq_server-3.9.8/etc/rabbitmq/rabbitmq.conf` 

关闭匿名

```bash
mqtt.allow_anonymous = false
```



# kubernetes statefulset

由于，rabbitmq基于主机名的，所以要使用statefulset.

```bash
kubectl create sts rabbitmq-deploy --image=rabbitmq 
```

> - 定位在单个主机上
> - 时间
> - 健康接口
>
> ```diff
>  18 spec:
>  19   nodeName: cn-hangzhou.192.168.2.5      
> 
>  47         volumeMounts:
>  48         - name: data
>  49           mountPath: /data/rabbitmq
>  50         - name: volume-localtime
>  51           mountPath: /etc/localtime                                                                                                                                                                                                                                                                                    
>  52       volumes:
>  53       - hostPath:
>  54           path: /etc/localtime
>  55           type: ""
>  56         name: volume-localtime
>  57       - hostPath:
>  58           path: /data/rabbitmq
>  59           type: DirectoryOrCreate
>  60         name: data
> 
> ```
>
> - headless-service
>
> ```diff
> spec:
>   replicas: 1
>   selector:
>     matchLabels:
>       app: rabbitmq-sts
> +  serviceName: 'rabbitmq-headless-svc'
>   template:
> ```
>
> - 配置插件
>
> ```diff
>           volumeMounts:
>             - mountPath: /data/rabbitmq
>               name: data
>             - mountPath: /etc/localtime
>               name: volume-localtime
> +            - mountPath: /rabbitmq_server-3.9.8/etc/rabbitmq/rabbitmq.conf
> +              name: volume-1635727981738
> +              subPath: rabbitmq.conf
>       nodeName: cn-hangzhou.192.168.2.5
>       restartPolicy: Always
>       volumes:
>         - hostPath:
>             path: /data/rabbitmq
>             type: DirectoryOrCreate
>           name: data
>         - hostPath:
>             path: /etc/localtime
>             type: ''
>           name: volume-localtime
> +        - configMap:
> +            defaultMode: 420
> +            name: rabbitmq-mqtt-conf
> +          name: volume-1635727981738
> ```
>
> ```yaml
> ---
> # 插件配置
> apiVersion: v1
> data:
>   rabbitmq.conf: |-
>     mqtt.listeners.tcp.default = 1883
>     ## Default MQTT with TLS port is 8883
>     # mqtt.listeners.ssl.default = 8883
> 
>     # anonymous connections, if allowed, will use the default
>     # credentials specified here
>     mqtt.allow_anonymous  = true
>     mqtt.default_user     = guest
>     mqtt.default_pass     = guest
> 
>     mqtt.vhost            = ishop_pos
>     mqtt.exchange         = amq.topic
>     # 24 hours by default
>     mqtt.subscription_ttl = 86400000
>     mqtt.prefetch         = 10
> kind: ConfigMap
> metadata:
>   name: rabbitmq-mqtt-conf
>   namespace: default
> ```

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: rabbitmq-sts
  name: rabbitmq-sts
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rabbitmq-sts
  serviceName: 'rabbitmq-headless-svc'
  template:
    metadata:
      labels:
        app: rabbitmq-sts
    spec:
      containers:
        - env:
            - name: RABBITMQ_MNESIA_DIR
              value: /data/rabbitmq
            - name: RABBITMQ_LOGS
              value: '-'
          image: 'rabbitmq:latest'
          imagePullPolicy: Always
          livenessProbe:
            failureThreshold: 1
            periodSeconds: 10
            successThreshold: 1
            tcpSocket:
              port: 1883
            timeoutSeconds: 1
          name: rabbitmq
          readinessProbe:
            failureThreshold: 1
            periodSeconds: 10
            successThreshold: 1
            tcpSocket:
              port: 1883
            timeoutSeconds: 1
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
          startupProbe:
            failureThreshold: 30
            periodSeconds: 10
            successThreshold: 1
            tcpSocket:
              port: 1883
            timeoutSeconds: 1
          volumeMounts:
            - mountPath: /data/rabbitmq
              name: data
            - mountPath: /etc/localtime
              name: volume-localtime
            - mountPath: /rabbitmq_server-3.9.8/etc/rabbitmq/rabbitmq.conf
              name: volume-1635727981738
              subPath: rabbitmq.conf
      nodeName: cn-hangzhou.192.168.2.5
      restartPolicy: Always
      volumes:
        - hostPath:
            path: /data/rabbitmq
            type: DirectoryOrCreate
          name: data
        - hostPath:
            path: /etc/localtime
            type: ''
          name: volume-localtime
        - configMap:
            defaultMode: 420
            name: rabbitmq-mqtt-conf
          name: volume-1635727981738
  updateStrategy:
    rollingUpdate:
      partition: 0
    type: RollingUpdate
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: rabbitmq-sts
  name: rabbitmq-sts
  namespace: default
spec:
  ports:
    - name: client
      nodePort: 32613
      port: 61613
      protocol: TCP
      targetPort: 1883
    - name: web
      nodePort: 32614
      port: 15672
      protocol: TCP
      targetPort: 15672
  selector:
    app: rabbitmq-sts
  type: NodePort
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: rabbitmq-sts
  name: rabbitmq-headless-svc
  namespace: default
spec:
  clusterIP: None
  clusterIPs:
    - None
  ports:
    - name: web
      port: 15672
      protocol: TCP
      targetPort: 15672
    - name: mqtt
      port: 1883
      protocol: TCP
      targetPort: 1883
  selector:
    app: rabbitmq-sts
  type: ClusterIP
---
# 插件配置
apiVersion: v1
data:
  rabbitmq.conf: |-
    mqtt.listeners.tcp.default = 1883
    ## Default MQTT with TLS port is 8883
    # mqtt.listeners.ssl.default = 8883

    # anonymous connections, if allowed, will use the default
    # credentials specified here
    mqtt.allow_anonymous  = true
    mqtt.default_user     = guest
    mqtt.default_pass     = guest

    mqtt.vhost            = /
    mqtt.exchange         = amq.topic
    # 24 hours by default
    mqtt.subscription_ttl = 86400000
    mqtt.prefetch         = 10
kind: ConfigMap
metadata:
  name: rabbitmq-mqtt-conf
  namespace: default
```



# 测试MQTT连通性

下载工具：https://mqttx.app/zh

## 连接配置

因为MQTT没有虚拟主机的概念，rabbitmq-mqq官方有2种方式连接指定虚拟机。

![image-20211029111950775](http://myapp.img.mykernel.cn/image-20211029111950775.png)

> - 指定用户名，密码。哪个虚拟主机
>   - 由官方配置文件默认的vhost决定 。
>   - 或者由全局的port映射连接哪个端口，就由哪个虚拟主机处理。
> - 指定虚拟主机:用户名。此优先级高于上面的指定

## 订阅topic

官方说，rabbitmq-mqtt中的topic是以`/`分隔的，然后这个插件会将`/`转化 成`.`

![image-20211029112407722](http://myapp.img.mykernel.cn/image-20211029112407722.png)

然后选择好之后，

![image-20211029112442620](http://myapp.img.mykernel.cn/image-20211029112442620.png)

> 此处已接收，左侧高亮，即选择这个topic. 中间就是别人发布的内容。

## 发布

![image-20211029112605711](http://myapp.img.mykernel.cn/image-20211029112605711.png)

