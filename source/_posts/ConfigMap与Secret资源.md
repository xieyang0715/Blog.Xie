---
title: ConfigMap与Secret资源
date: 2021-01-28 06:26:41
tags:
---



- configMap (名称空间中资源)

  非敏感信息场景：nginx配置

  - Pod中容器提供配置信息；
    - env：      不可动态修改容器中配置
    - volumes: 可动态修改容器中配置，需要外部发送命令来重载，或高级工具完成, ....。
      - pod.spec.volumes.configMap
  - 一组类似Pod应用充当配置中心；
  - 非容器化应用，使用ansible/puppet动态修改配置
  - 创建cm
    - 命令行
      - --from-literal
      - --from-file 不指键名，为文件基名为key名
    - 配置清单 yaml，没有spec字段，`kubectl create cm test -n test  --dry-run -o yaml`

- Secret

  敏感信息场景：mysql初始化时设定密码

  - generic
    - env
    - volumes
  - tls
    - volumes
    - env 一般认证使用文件
  - docker-registry
    - 指定docker-server, docker-username,docker-password,docker-email

# 为什么使用configmap/secret？

容器中使用配置？

- 硬编码
  - 存储卷：nginx加载配置文件模块，
- 环境变量
  - 云原生应用，直接传递环境变量作为配置参数。
  - kubernetes环境变量

> 以上两个配置方式，宿主机修改配置，要送到容器中配置文件或变量更改需要重启容器。



- kuberentes的configmap/secret集中配置(配置中心)

  - 5个pod，hpa更多的pod，都在修改配置后重载pod。
  - 更新配置后，热加载。不需要重启pod

  如果cm/secret以**env方式**传入pod中，当cm/secret修改, 里面的env不会对应修改，**重建pod才生效**。

  如果cm/secret以**volumes方式**传入pod, 容器挂载volumes之后，cm/secret修改，**里面的文件会对应修改**。

  >  以上配置能及时送到容器中生效，**但是进程不能重读容器中的配置**。

  



<!--more-->

# configmap和secret区别 

configmap保存非敏感

secret保存敏感信息，base64编码



都提供数据：

- data
  - key1: value1
  - key2: value2

>  文件：名为键，内容为值



都属于名称空间



# 调用configmap/secret

调用configmap

- key/value映射为环境变量。  配置**不能及时**送到容器中生效

  ![image-20210128144024669](http://myapp.img.mykernel.cn/image-20210128144024669.png)

- key/value 以Key为名挂载到某个目录中。 配置**能及时送到容器中生效**，但是进程不能重读容器中的配置。

  ![image-20210128144035505](http://myapp.img.mykernel.cn/image-20210128144035505.png)



​	



# 创建configmap

```bash
[root@master chapter7]# kubectl create cm -h
#1. 命令行直接执行键和值
kubectl create configmap my-config --from-literal=key1=config1 --from-literal=key2=config2
#2. 从文件加载
## 指定键
kubectl create configmap my-config --from-file=key1=/path/to/bar/file1.txt --from-file=key2=/path/to/bar/file2.txt
## 不指定键, 文件名当作键
kubectl create configmap my-config --from-file=path/to/bar
```



示例为filebeat创建2个键redis_host、loglevel

```bash
[root@master chapter7]# kubectl create ns config
namespace/config created
[root@master chapter7]# kubectl create cm filebeat-cfg -n config  --from-literal=redis_host="redis.default.svc.cluster.local"  --from-literal=log_level="Info"
configmap/filebeat-cfg created


# 显示
[root@master chapter7]# kubectl get cm -n config
NAME               DATA   AGE
filebeat-cfg       2      28s
kube-root-ca.crt   1      98s
[root@master chapter7]# kubectl get cm -n config filebeat-cfg -o yaml
apiVersion: v1
data: # 数据，以下是变变量
  log_level: Info 
  redis_host: redis.default.svc.cluster.local
kind: ConfigMap

```



pod引用configmap

## env引用configmap

修改configmap容器中的变量值不会变

与docker --env无异，不能及时送到容器中，需要重建

```bash
[root@master configmap]# kubectl explain pod.spec.containers | grep '>'
RESOURCE: containers <[]Object>
   args	<[]string>
   command	<[]string>
   env	<[]Object>   # 指定环境变量
   envFrom	<[]Object> # 从另一个位置加载环境变量
   image	<string>
   imagePullPolicy	<string>
   lifecycle	<Object>
   livenessProbe	<Object>
   name	<string> -required-
   ports	<[]Object>
   readinessProbe	<Object>
   resources	<Object>
   securityContext	<Object>
   startupProbe	<Object>
   stdin	<boolean>
   stdinOnce	<boolean>
   terminationMessagePath	<string>
   terminationMessagePolicy	<string>
   tty	<boolean>
   volumeDevices	<[]Object>
   volumeMounts	<[]Object>
   workingDir	<string>
 
 
 
 [root@master configmap]# kubectl explain pod.spec.containers.env | grep '>'
RESOURCE: env <[]Object>
   name	<string> -required-
   value	<string> # 直接给值
   valueFrom	<Object> # 从另一个位置引用值


[root@master configmap]# kubectl explain pod.spec.containers.env.valueFrom | grep '>'
RESOURCE: valueFrom <Object>
   configMapKeyRef	<Object> # 引用来自cm
   fieldRef	<Object>
     `metadata.labels['<KEY>']`, `metadata.annotations['<KEY>']`, spec.nodeName,
   resourceFieldRef	<Object>
   secretKeyRef	<Object>  # 引用来自secret

[root@master configmap]# kubectl explain pod.spec.containers.env.valueFrom.configMapKeyRef | grep '>'
RESOURCE: configMapKeyRef <Object>
   key	<string> -required- # 引用configmap的key名 redis_host, log_level
   name	<string>            # 引用的cm的名
   optional	<boolean>     # 引用的key是否可选。默认false必须有key, 没有就报错。true表示key可选，key没有也不会报错


```









```bash
[root@master configmap]# kubectl run pod-cfg-demo -n config --image=ikubernetes/filebeat:5.6.5-alpine --dry-run -oyaml>pod-cfg.yaml

```

```diff
[root@master configmap]# cat pod-cfg.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-cfg-demo
  namespace: config
spec:
  containers:
  - image: ikubernetes/filebeat:5.6.5-alpine
    name: filebeat
    resources: {}
+    env:
+    - name: REDIS_HOST
+      valueFrom:
+        configMapKeyRef:
+          name: filebeat-cfg
+          key: redis_host
		   # 没有加optional 说明 是必须存在key
+    - name: LOG_LEVEL
+      valueFrom:
+        configMapKeyRef:
+          name: filebeat-cfg
+          key: log_level
  dnsPolicy: ClusterFirst
  restartPolicy: Always
[root@master configmap]# kubectl apply -f pod-cfg.yaml
pod/pod-cfg-demo created

```

```bash
# 显示
[root@master configmap]# kubectl get pod -n config
NAME           READY   STATUS    RESTARTS   AGE
pod-cfg-demo   1/1     Running   0          69s
[root@master configmap]# kubectl exec -it pod-cfg-demo -n config -- printenv
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=pod-cfg-demo 
TERM=xterm
REDIS_HOST=redis.default.svc.cluster.local # redis地址
LOG_LEVEL=Info # log_level
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
KUBERNETES_SERVICE_HOST=10.96.0.1
FILEBEAT_VERSION=5.6.5
HOME=/root

```

### 验证修改configmap 

```bash
[root@master configmap]# kubectl edit cm/filebeat-cfg  -n config
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  log_level: Notice
  redis_host: redis.default.svc.cluster.local
kind: ConfigMap
metadata:
  creationTimestamp: "2021-01-28T06:46:24Z"
  name: filebeat-cfg
  namespace: config
  resourceVersion: "1282419"
  uid: cfb9150a-3996-4191-899e-f195d923d3ba
                                                                                                                                 
configmap/filebeat-cfg edited
[root@master configmap]# kubectl exec -it pod-cfg-demo -n config -- printenv
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=pod-cfg-demo
TERM=xterm
REDIS_HOST=redis.default.svc.cluster.local 
LOG_LEVEL=Info # 未改变 
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
KUBERNETES_SERVICE_HOST=10.96.0.1
FILEBEAT_VERSION=5.6.5
HOME=/root

```

重建pod

```bash
 kubectl delete  pod/pod-cfg-demo -n config

[root@master configmap]# kubectl apply -f pod-cfg.yaml 
pod/pod-cfg-demo created
[root@master configmap]# kubectl exec -it pod-cfg-demo -n config -- printenv
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=pod-cfg-demo
TERM=xterm
REDIS_HOST=redis.default.svc.cluster.local
LOG_LEVEL=Notice # 重建后，容器中配置变动
```

## 存储卷引用configmap

多个pod使用存储卷的cm, 当修改存储卷时，所有pod中的配置都会改变

但是进程不会自动重载配置，需要手工重载。有些情况下Pod自动重载。

即cm为K8S的配置中心



非容器化多个应用使用相同的配置，ansible完成修改配置，批量同步。Apollo, 携程。 Distconf, 百度。

也可以**让程序支持配置中心**，应用程序不再从本地读取配置，而是从配置中心服务器获取配置。



准备2个nginx配置文件

```bash
[root@master configmap]# cat server1.conf 
server {
	server_name www.magedu.com;
	listen 80;
	location / {
		root /html/magedu;
	}
}


[root@master configmap]# cat server2.conf 
server {
	listen 80;
	server_name www.ilinux.io;
	location / {
		root /html/ilinux;
	}
}

```

创建configmap

```bash
kubectl create cm nginx-cfg --from-file=./server1.conf --from-file=server-second.conf=./server2.conf  -n config



[root@master configmap]# kubectl get cm -n config
NAME               DATA   AGE
filebeat-cfg       2      23m
kube-root-ca.crt   1      25m
nginx-cfg          2      19s


# configmap
[root@master configmap]# kubectl get cm -n config nginx-cfg -oyaml
apiVersion: v1
data:
  server-second.conf: "server {\n\tlisten 80;\n\tserver_name www.ilinux.io;\n\tlocation
    / {\n\t\troot /html/ilinux;\n\t}\n}\n\n\n"
  server1.conf: "server {\n\tserver_name www.magedu.com;\n\tlisten 80;\n\tlocation
    / {\n\t\troot /html/magedu;\n\t}\n}\n\n\n"
kind: ConfigMap
metadata:
  name: nginx-cfg
  namespace: config


```

pod挂载configmap

```bash
[root@master ~]# kubectl explain pod.spec.volumes.configMap | grep '>'
RESOURCE: configMap <Object>
   defaultMode	<integer> # 文件 644，目录默认755
   items	<[]Object> # 哪个key映射为文件
   name	<string> # cm名
   optional	<boolean> # 默认false, 必须存在cm的key

[root@master ~]# kubectl explain pod.spec.volumes.configMap.items | grep '>'
RESOURCE: items <[]Object> 
   key	<string> -required- # cm的key
   mode	<integer>　#权限，没有就是使用defaultMode
   path	<string> -required- #映射的文件名

```

```diff
[root@master configmap]# cat myapp-pod.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
+  namespace: config
spec:
+  volumes:
+  - name: config
+    configMap:
+      name: nginx-cfg
+      items:
+      - key: server1.conf
+        path: server-first.conf
+      - key: server-second.conf # cm的key名
+        path: server-second.conf # 容器中的名称
  containers:
  - image: ikubernetes/myapp:v1
    name: myapp
    resources: {}
+    volumeMounts:
+    - name: config # volumes的名
+      mountPath: /etc/nginx/conf.d/ # 确保没有与volumes定义文件一样，如果一样使用子路径挂载, ...
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  
  
[root@master configmap]# kubectl apply -f myapp-pod.yaml
pod/myapp-pod created
[root@master configmap]# kubectl get pod -n config
NAME           READY   STATUS    RESTARTS   AGE
myapp-pod      1/1     Running   0          7s
pod-cfg-demo   1/1     Running   0          51m

```

验证配置

```bash
[root@master configmap]# kubectl exec -it myapp-pod -n config -- sh
/ # ls -l /etc/nginx/conf.d/
total 0
lrwxrwxrwx    1 root     root            24 Jan 28 07:54 server-first.conf -> ..data/server-first.conf
lrwxrwxrwx    1 root     root            25 Jan 28 07:54 server-second.conf -> ..data/server-second.conf

```

### 验证修改cm

```diff
[root@master ~]# kubectl edit cm -n config nginx-cfg
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  server-second.conf: "server {\n\tlisten 80;\n\tserver_name www.ilinux.io;\n\tlocation
    / {\n\t\troot /html/ilinux;\n\t}\n}\n\n\n"
+  server1.conf: "server {\n\tserver_name www.magedu.com;\n\tlisten 8080;\n\tlocation 
    / {\n\t\troot /html/magedu;\n\t}\n}\n\n\n"
kind: ConfigMap
metadata:
  creationTimestamp: "2021-01-28T07:54:28Z"
  name: nginx-cfg
  namespace: config
  resourceVersion: "1289237"
  uid: b558030f-828e-4373-8d18-d402a82dc34d

```

>  修改8080端口

查看配置

```bash
/ # [root@master configmap]# kubectl exec -it myapp-pod -n config -- sh
/ # cat /etc/nginx/conf.d/server-first.conf
server {
	server_name www.magedu.com;
	listen 8080;
	location / {
		root /html/magedu;
	}
}
```

## flannel

https://github.com/coreos/flannel

https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: | # |表示多行配置
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",   # 为什么是10.244.0.0/16，不应该修改。应该启动k8s和flannel指定相同的地址。
      "Backend": {
        "Type": "vxlan"
      }
    }
```



# 创建secret

避免配置信息泄漏



## secret类型

```bash
[root@master ~]# kubectl create secret -h
Create a secret using specified subcommand.

Available Commands:
  docker-registry 连接docker-registry的认证信息，在`pod.spec.imagePullSecrets` 定义
  					如果使用docker-registry不利于修改, `pod.spec.serviceAccountName`内部调用secret
  generic         非tls
  tls             X509格式打包为secret, 不管原名，统统叫tls.crt, tls.key
```



```bash
kubectl create secret generic -h
  kubectl create secret generic NAME [--type=string] [--from-file=[key=]source] [--from-literal=key1=value1]

```



## generic类型

```bash
kubectl create secret generic mysql-root-password -n config --from-literal=password=magedu

[root@master ~]# kubectl get secret mysql-root-password -n config -o yaml
apiVersion: v1
data:
  password: bWFnZWR1
kind: Secret
metadata:
  name: mysql-root-password
  namespace: config
type: Opaque # generic

[root@master ~]# echo bWFnZWR1 | base64 -d
magedu[root@master ~]# 


[root@master ~]# kubectl get secret  -n config
NAME                  TYPE                                  DATA   AGE
default-token-mrpp2   kubernetes.io/service-account-token   3      115m
mysql-root-password   Opaque                                1      74s

```



### env启动mysql

```bash
[root@master configmap]# kubectl explain pod.spec.containers.env.valueFrom.secretKeyRef
KIND:     Pod
VERSION:  v1

RESOURCE: secretKeyRef <Object>

DESCRIPTION:
     Selects a key of a secret in the pod's namespace

     SecretKeySelector selects a key of a Secret.

FIELDS:
   key	<string> -required-  # secret key
   name	<string > # secret名称
   optional	<boolean>

[root@master configmap]# kubectl run mysql -n config --image=mysql:5.6 --dry-run -o yaml > mysql.yaml

```



```diff
"mysql.yaml" 18L, 371C written                                                                                                                                                                                                                                                                           
[root@master configmap]# cat mysql.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql
  namespace: config
spec:
  containers:
  - image: mysql:5.6
    name: mysql
+    env:
+    - name: MYSQL_ROOT_PASSWORD
+      valueFrom:
+        secretKeyRef:
+          key: password             # secret key
+          name: mysql-root-password # secret名
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
[root@master configmap]# kubectl apply -f mysql.yaml
pod/mysql created

[root@master configmap]# kubectl get pod -n config -o wide 
NAME           READY   STATUS    RESTARTS   AGE    IP            NODE                NOMINATED NODE   READINESS GATES
myapp-pod      1/1     Running   0          52m    10.244.2.24   node02.magedu.com   <none>           <none>
mysql          1/1     Running   0          113s   10.244.1.18   node01.magedu.com   <none>           <none>
pod-cfg-demo   1/1     Running   0          103m   10.244.2.22   node02.magedu.com   <none>           <none>

```

连接mysql

```bash
[root@master configmap]# kubectl exec -it mysql -n config -- sh
# mysql -uroot -pmagedu
Warning: Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 1
Server version: 5.6.51 MySQL Community Server (GPL)

Copyright (c) 2000, 2021, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
# printenv
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.96.0.1:443
HOSTNAME=mysql
MYSQL_MAJOR=5.6
HOME=/root
MYSQL_ROOT_PASSWORD=magedu # 查看变量，还是实际密码
TERM=xterm
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
MYSQL_VERSION=5.6.51-1debian9
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
GOSU_VERSION=1.12
KUBERNETES_SERVICE_HOST=10.96.0.1
PWD=/

```



### NMT

mysql

- svc名
- deploy
- secret传入密码



### 存储卷引用secret generic

key/value映射指定路径下的文件名

私有信息，映射为文件

```bash
[root@master configmap]# kubectl explain pod.spec.volumes.secret | grep '>'
RESOURCE: secret <Object>
   defaultMode	<integer> # 敏感数据640/600
   items	<[]Object>  # 引用key和文件名
   optional	<boolean>   # 默认必须有key
   secretName	<string> # secret名
[root@master configmap]# kubectl explain pod.spec.volumes.secret.items | grep '>'
RESOURCE: items <[]Object>
   key	<string> -required- # secret的key
   mode	<integer> # 敏感数据640/600
   path	<string> -required- # 相对的文件名

```

比如tomcat的配置文件，`tomcat-user.xml`保存内建认证的账号名密码

redis密码使用配置文件提供



## tls

```bash
[root@master configmap]# kubectl create secret tls -h
  kubectl create secret tls NAME --cert=path/to/cert/file --key=path/to/key/file [--dry-run=server|client|none]
```

> 没有让我们指定cert/key的key，无论什么名，统统映射为tls.crt, tls.key

使用ingress时，使用过

```diff
[root@master ingress]# kubectl create secret tls mysql-cert --cert=./tomcat.crt --key=./tomcat.key -n config
[root@master ingress]# kubectl get secret -n config
NAME                  TYPE                                  DATA   AGE
mysql-cert            kubernetes.io/tls                     2      19s
[root@master ingress]# kubectl get secret mysql-cert  -o yaml -n config
apiVersion: v1
data:
+  tls.crt: LS0tLS1CRUdJ  # 可以看出是tls.crt,base64编码
+  tls.key:   xxxxx # 可以看出是tls.key,base64编码
kind: Secret
metadata:
  name: mysql-cert
  namespace: config
type: kubernetes.io/tls

```

> secret清单一般不会修改, 而且保存下来，在应用k8s前会被看见。所以secret一般不会使用清单创建

创建后，ingress直接引用secret tls，会自动使用证书。

```diff
[root@master ingress]# cat myapp-tls-ingress-example.yaml 
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-tls
  namespace: myns
  annotations:
    kubernetes.io/ingress.class: "nginx" # nginx controller解析
spec:
+  tls: # 表示规则对应https
+  - hosts:
+      - www.ilinux.io
+    secretName: ilinux-cert
  rules:
  - host: www.ilinux.io
    http: # https会话卸载器，面向后端是http
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 80

```

### 存储卷引用secret tls

```diff
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  namespace: config
spec:
  volumes:
  - name: config
    configMap:
      name: nginx-cfg
      items:
      - key: server1.conf
        path: server-first.conf
      - key: server-second.conf # cm的key名
        path: server-second.conf # 容器中的名称
+  - name: tls
+    secret:
+      secretName: mysql-cert
+      items:    # 不写全部引用
+      - key: tls.crt
+        path: myapp.crt
+      - key: tls.key
+        path: myapp.key
+        mode: 0600
  containers:
  - image: ikubernetes/myapp:v1
    name: myapp
    resources: {}
    volumeMounts:
    - name: config # volumes的名
      mountPath: /etc/nginx/conf.d/ # 确保没有与volumes定义文件一样，如果一样使用子路径挂载, ...
+    - name: tls
+      mountPath: /etc/nginx/certs/ # 不存在，自动创建
  dnsPolicy: ClusterFirst
  restartPolicy: Always

```

```bash
[root@master configmap]# kubectl apply -f myapp-pod-tls.yaml

[root@master configmap]# kubectl get pod -n config
NAME           READY   STATUS    RESTARTS   AGE
# 运行
myapp-pod      1/1     Running   0          44s
mysql          1/1     Running   0          25m
pod-cfg-demo   1/1     Running   0          127m
[root@master configmap]# kubectl exec -it myapp-pod -n config -- sh
/ # ls /etc/nginx/certs/
myapp.crt  myapp.key
/ # ls -l /etc/nginx/certs/
total 0
# 权限 是777, 原文件权限是600
lrwxrwxrwx    1 root     root            16 Jan 28 09:10 myapp.crt -> ..data/myapp.crt
lrwxrwxrwx    1 root     root            16 Jan 28 09:10 myapp.key -> ..data/myapp.key


# 原文件权限
/etc/nginx/certs # ls -l -h ..data/myapp.crt 
-rw-r--r--    1 root     root        1.2K Jan 28 09:10 ..data/myapp.crt
/etc/nginx/certs # ls -l -h ..data/myapp.key 
-rw-------    1 root     root        1.6K Jan 28 09:10 ..data/myapp.key

# 查看证书, 已经正常
/etc/nginx/certs # cat myapp.key 
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEAxk/bf72uRC8tWbBNMm0chCKzdZCve0dB8i4+yYAbuetFFVBu
ivY97muyhAHg9g7r18I+jvXMjDSoG4Ou5ePVjxbD131GOynXevyJORk2Q0pd9P7j
KsJOizhFcLQoT6lG4ykZaJPVeVtltBs3q1PQV8a+CkKlv70JjWj4ataZ34zIR6P5
1zLFLA9iJ4JkUG1uY7wD8M+2RmBQQ+0fB8URKNQDWq27PKsSao4gsMoej0l/wzjH
MXpgpwES9HVs32A3Pwp7oeRO3m1js0HajTaCOmYZW60ayFnVm43V39D5XaabkrZK
XckR1wR1W6c1QzjnG9xxAW8tPjCIC6wMu2xaowIDAQABAoIBAHsJzuCpebaaIqPz
y2GO6tNciEVX2Fg/NL4iTRhNoGYwfzMjLQKQlooXTbGzTLS9OzwpKxEdlaQjg21W
vSuquLRHZoiLFAjfA+8tQaIob08+k57OiXjdB0g/SG4NiLksCGwl8rq8hgT+XNJq
1JY6sRfUmdHZ2eZlTcjrqLz4mo1kPIoQEctrCmsKM0IG1A6Ehg+0+t+WnFK6BvlC
CCK77pyt1l/YooK8VwdICmdexOht6vfeTf4yQ1xfcWv5Y9zwVq2bmD6V5hzmYcPU
POBDRtPFSc5pexHzz4d+Bab7yIXdHy4VVfVDh/CdV8TgSrBneuHZM1RzFQHkvisx
q0CE2uECgYEA/5YH5qaRu2O/liHQHlzSZkpb899KFRllihGtyQgCgRoagwVqRbT0
qDwA4kU09cowoHvphxJL2ILNyVGdPkzrfLbuKmgPxZ8woKEeRvwIHurTJpyODrsw
HAmXf6KReWcbchjAWvTGlhDMrbCNh8yP2JZD8utqrpXuFPF975Y0JjECgYEAxqIU
etMdv+Yc5geIxQ77qifZcfksJufyz8u68sUZojeu7HsSGLZ3oGyFvbP8L0DlTKp4
HlDODijUQF2nLnaGnJAxYm5+uV5JMTx5FW5DSjo5Q9Pr257Dm5FpOYEkE+hhQxsg
sNftx/X1V7MbvtLJv/V0mR+AwvMevq3qdGTDlRMCgYEAk64xKokcs9ZTIYCwLJsd
x5U3xJZEzCQ8k6bbb8l9CPP4VbSPT2/b3kmtiRDMJSmLJ2/x4+YihRwvpB/QZ+sy
NoHM5Bv04Q+2nVn7kLCYUKUHFMxpGQH4LnssWseonymAplC+9M9y38sdOU9GuCzv
AQrygC6fGfnv85IGXqW/xEECgYBfbSyDmXs4XxfRFxuI+FrFc2GO1NN2WYaYd9r3
mONowHGkILgf8UFla92QtrBYD0hZ3afZgJ6NxOW7ioKv2rdu7gMbs9PjwD1Pjyro
tdFUDsbGJECygQKecWxo+PbZLZHUiGrbKtGMeEiG+oBA28mbFBQRIEZe4igKGUmC
44nmywKBgQCKbzcwjAcLh0klyz0LNi19POcBYvJi/D2jh48eRK2T/yEKtiSa7nyd
sG8xEbGqaDXauXLpDhP0bGdgPSCXRN2GMZNQ+z19/KDi3AZM3PsUbZu7QsDqq0yz
DrYAy08tse2EnGL+yrESh7eBXFzY1JQXjmVsWexPpiB1qExV3o/X9Q==
-----END RSA PRIVATE KEY-----

```



## docker registry

连接docker-registry的认证信息，在`pod.spec.imagePullSecrets` 定义
如果使用docker-registry不利于修改, `pod.spec.serviceAccountName`内部调用secret

```bash
[root@master configmap]# kubectl create secret docker-registry -h
  kubectl create secret docker-registry NAME --docker-username=user --docker-password=password --docker-email=email
[--docker-server=string] [--from-literal=key1=value1] [--dry-run=server|client|none] [options]

  kubectl create secret docker-registry my-secret --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER
--docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL
```

>  --docker-username=user 用户名
>
> --docker-password=password 密码
>
> --docker-email=email email信息
> [--docker-server=string]  docker server自已的标识符
>
> [--from-literal=key1=value1]  #
>
> [--dry-run=server|client|none] [options]





# 如果flannel不想使用10.244.0.0/16

- [x] 部署k8s集群--pod-network 指定网段
- [x] flannel的configmap中的net.json配置文件的network指定网段





