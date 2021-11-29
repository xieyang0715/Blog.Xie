---
title: "kubernetes deploy jenkins"
date: 2021-03-01 03:37:26
tags:
- "生产小经验"
- "kubernetes"
- "jenkins"
---



需要jenkins自动触发构建，所以就制作Jenkins镜像完成部署k8s, 参考官方文档：https://www.jenkins.io/doc/book/installing/war-file/

# 方法1

## 制作kubectl

也可以jenkins安装`kubernetes插件`,  配置认证如下

https://blog.csdn.net/winterfeng123/article/details/110041772

先确保k8s可以连接

```bash
root@harbor01:~/weizhixiu/dockerfile/jenkins-dockerfile# cat Dockerfile 
FROM harbor.youwoyouqu.io/baseimage/jdk@sha256:617e32a473fea1c26a208ed4a9c4101b2a3c7c5b0e352a0e115fc4a06be03fbf

ADD kubectl /usr/bin/
ADD config  /root/.kube/

CMD ["kubectl","get","ns"]

root@harbor01:~/weizhixiu/dockerfile# docker run --rm -it --dns 192.168.0.77 harbor.youwoyouqu.io/devops/jenkins:2021030107  
NAME              STATUS   AGE
default           Active   2d23h
jenkins           Active   49m
kube-node-lease   Active   2d23h
kube-public       Active   2d23h
kube-system       Active   2d23h
weizhixiu         Active   2d19h
```

## 启动jenkins

再通过war包启动jenkins

https://www.jenkins.io/download/

```bash
root@node06:~/weizhixiu/dockerfile/jenkins-dockerfile# cat Dockerfile 
FROM harbor.youwoyouqu.io/baseimage/jdk:8

ADD kubectl /usr/bin/
ADD config  /root/.kube/

ADD jenkins.war .
ENV JENKINS_HOME=/opt/data


# git
RUN apk add --no-cache git openssh

CMD ["java", "-jar", "/jenkins.war", "--httpPort=9090","-server -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=5"]
```

> - kubectl，config属于集群，现成的, 运行在集群中，所以使用内部地址与APIserver通信     10.96.0.1:443
> - war文件：` wget https://get.jenkins.io/war-stable/2.263.4/jenkins.war`
> - jenkins数据目录在  JENKINS_HOME 指定的目录
> - jenkins配置时需要git,openssh

build

```bash
root@harbor01:~/weizhixiu/dockerfile/jenkins-dockerfile# cat build.sh 
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


docker build -t harbor.youwoyouqu.io/devops/jenkins:2.263.4-alpine ./
docker push harbor.youwoyouqu.io/devops/jenkins:2.263.4-alpine

```

测试jenkins

```bash
#docker run --rm -it -p 9090:9090 --dns 192.168.0.77 -v /opt/jenkins_data:/opt/data  harbor.youwoyouqu.io/devops/jenkins:2.263.4-alpine 

#jenkins启动成功后, 进入镜像进行如下操作
##然后修改镜像
## sed -i -e 's@https://updates.jenkins.io/download@https://mirrors.tuna.tsinghua.edu.cn/jenkins@g' -e 's@google.com@baidu.com@g' /opt/jenkins_data/updates/default.json 
#再启动jenkins

# 启动jenkins
#docker run --rm -it -p 9090:9090 --dns 192.168.0.77 -v /opt/jenkins_data:/opt/data  harbor.youwoyouqu.io/devops/jenkins:2.263.4-alpine 

# h.m.UpdateCenter$CompleteBatchJob#run: Completed installation of 79 plugins in 2 分 33 秒
```

> jenkins加速参考: 
>
> https://www.cnblogs.com/richiewlq/p/11919081.html        ok
>
> https://blog.csdn.net/qq_34556414/article/details/109594110  not ok

## 编写k8s yaml

由于jenkins有数据，需要pv或storageclass, 但是nfs不合适生产使用，所以需要将pods固定在某个节点之上，方案有以下

- [ ] nodeName, 不需要调度了
- [ ] nodeSelect + node label
- [x] nodeaffinity + node label + node taint

```bash
root@harbor01:~# kubectl label node node03.youwoyouqu.io jenkins="true"
kubectl taint nodes  node03.youwoyouqu.io node.kubernetes.io/unschedulable:NoSchedule
```

```bash
root@harbor01:~/weizhixiu/dockerfile/jenkins-dockerfile# cat jenkins-ns.yaml 
apiVersion: v1
kind: Namespace
metadata:
  name: jenkins
```

生成deployment

```bash
root@harbor01:~/weizhixiu/dockerfile/jenkins-dockerfile# kubectl create deployment jenkins -n jenkins --image=harbor.youwoyouqu.io/devops/jenkins:2.263.4-alpine --port=9090 --replicas=1 --dry-run -o yaml > jenkins-deployment.yaml
root@harbor01:~/weizhixiu/dockerfile/jenkins-dockerfile# kubectl create service nodeport jenkins --tcp=9090:9090 --dry-run -o yaml >> jenkins-deployment.yaml 
```

```diff
root@harbor01:~/weizhixiu/dockerfile/jenkins-dockerfile# cat jenkins-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: jenkins-deploy-label
  name: jenkins-deploy
  namespace: jenkins
spec:
+  strategy:
    rollingUpdate:
      maxSurge: 0
+      maxUnavailable: 1 # 占用端口，所以先停后创建
    type: RollingUpdate
  replicas: 1
  selector:
    matchLabels:
      app: jenkins-deploy-selector
  template:
    metadata:
      labels:
        app: jenkins-deploy-selector
    spec:
+      tolerations:
      - key: "node.kubernetes.io/unschedulable"
        operator: Equal
        effect: NoSchedule
+      imagePullSecrets:
      - name: registry
+      volumes:
      - name: jenkins-data
        hostPath:
          path: /opt/jenkins_data
          type: DirectoryOrCreate
+      nodeSelector:
        jenkins: "true"
      containers:
      - image: harbor.youwoyouqu.io/devops/jenkins:2.263.4-alpine
        name: jenkins
+        volumeMounts:
        - name: jenkins-data
          mountPath: /opt/data
        ports:
+        - containerPort: 9090
          protocol: TCP
          name: http
+        resources: 
          limits:
            cpu: 2
            memory: 2048Mi
          requests:
            cpu: 1000m
            memory: 500Mi
+        livenessProbe: 
+          initialDelaySeconds: 300
          successThreshold: 1
          failureThreshold: 2 
          periodSeconds: 3
          tcpSocket:
            port: 9090
+        readinessProbe: 
          initialDelaySeconds: 60 
          successThreshold: 1
          failureThreshold: 2 
          periodSeconds: 3 
          tcpSocket:
            port: 9090
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: jenkins-service-label
  name: jenkins
  namespace: jenkins
spec:
  ports:
  - name: 9090-9090
    port: 9090
    protocol: TCP
    targetPort: 9090
+    nodePort: 32090
  selector:
    app: jenkins-deploy-selector
  type: NodePort

```

```bash
root@harbor01:~/weizhixiu/dockerfile/jenkins-dockerfile# kubectl apply -f jenkins-deployment.yaml 
```

build脚本

```bash
root@master01:/data/weizhixiu/dockerfile/jenkins-dockerfile# cat build.sh 
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
image=harbor.youwoyouqu.io/devops/jenkins:${1}
docker build -t ${image} ./
docker push ${image}

kubectl set image deploy -n jenkins jenkins-deploy jenkins=${image}
sed -i "s@- image: .*@- image: ${image}@g"  jenkins-deployment.yaml
```



添加secret

```bash
root@harbor01:~/weizhixiu/dockerfile/jenkins-dockerfile# kubectl create -n jenkins secret docker-registry registry --docker-username=admin --docker-password=xxx  --docker-server=harbor.youwoyouqu.io 
```

验证pod

```bash
root@harbor01:~/weizhixiu/dockerfile/jenkins-dockerfile# kubectl get pods -n jenkins -o wide
NAME                             READY   STATUS    RESTARTS   AGE     IP            NODE                   NOMINATED NODE   READINESS GATES
jenkins-deploy-b968457c6-c47sw   1/1     Running   0          8m25s   10.244.5.34   node03.youwoyouqu.io   <none>           <none>
# node03上
```

node03之上修改镜像

```bash
# sed -i -e 's@https://updates.jenkins.io/download@https://mirrors.tuna.tsinghua.edu.cn/jenkins@g' -e 's@google.com@baidu.com@g' /opt/jenkins_data/updates/default.json
```

重启jenkins

```bash
kubectl delete pods -n jenkins --all
```

当jenkins在次进入正常状态时，访问node节点的 32090 http://192.168.0.87:32090

```bash
root@node03:~# cat /opt/jenkins_data/secrets/initialAdminPassword 
cc6bd42d99814493b770747cd624c8e6
```

用密码登陆jenkins

![image-20210301155844686](http://myapp.img.mykernel.cn/image-20210301155844686.png)

![image-20210301160132083](http://myapp.img.mykernel.cn/image-20210301160132083.png)

## 使用jenkins

参考：https://www.mykernel.cn/jenkins_v1.html#more

# 方法二

获取镜像: https://hub.docker.com/_/jenkins?tab=description

官方文档：https://github.com/jenkinsci/docker/blob/master/README.md

准备deploy

```bash
kubectl create deploy jenkins  --image=jenkins/jenkins:lts-jdk11
```

> 之后依据文档提供数据,  运行用户，时间
>
> ```bash
> kubectl edit deploy jenkins
>     spec:
>       containers:
>       - image: jenkins/jenkins:lts-jdk11
>         imagePullPolicy: IfNotPresent
>         name: jenkins
>         resources: {}
>         volumeMounts:
>         - mountPath: /etc/localtime
>           name: volume-localtime
>         - mountPath: /var/jenkins_home
>           name: data
>       restartPolicy: Always
>       securityContext:
>         runAsUser: 0
> 
>       volumes:
>       - hostPath:
>           path: /etc/localtime
>           type: ""
>         name: volume-localtime
>       - hostPath:
>           path: /data/jenkins_home
>           type: DirectoryOrCreate
>         name: data
> ```
>
> 日志在控制台，默认
>
> java选项

暴露clusterip

```bash
kubectl expose deploy jenkins --type=ClusterIP --port=8080 
```

> 提供agent端口
>
> ```bash
> kubectl edit svc jenkins 
>  23   ports:
>  24   - name: web
>  25     port: 8080
>  26     protocol: TCP
>  27     targetPort: 8080
>  28   - name: tcp
>  29     port: 50000
>  30     protocol: TCP
>  31     targetPort: 50000
> ```

暴露ingress

```bash
kubectl create ingress jenkins --rule="jenkins.mykernel.cn/=jenkins:8080"
```

访问 ingress svc nodeport.

