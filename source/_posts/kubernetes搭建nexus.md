---
title: kubernetes搭建nexus
date: 2021-03-03 05:34:33
tags:
- 生产小经验
- kubernetes
---



 

使用nexus可以节省公司出口带宽



# Docker安装nexus

下载：https://help.sonatype.com/repomanager3/download/download-archives---repository-manager-3

安装：https://help.sonatype.com/repomanager3/installation/installation-methods



```bash
root@node06:/data/weizhixiu/dockerfile/nexus-dockerfile# ls
Dockerfile  nexus-3.29.2-02-unix.tar.gz  nexus.vmoptions
```

准备Dockerfile

```dockerfile
FROM harbor.youwoyouqu.io/baseimage/jdk:8

# https://help.sonatype.com/repomanager3/download/download-archives---repository-manager-3

ADD nexus-3.29.2-02-unix.tar.gz /opt
ADD nexus.vmoptions /opt/nexus-3.29.2-02/bin/nexus.vmoptions

VOLUME /opt/sonatype-work/nexus3/
EXPOSE 8081

CMD ["/opt/nexus-3.29.2-02/bin/nexus","run"]
# admin admin123
# deployment deployment123
```

> 由于展开至/opt目录，所以数据就在 ` /opt/sonatype-work/nexus3/`

准备配置

```bash
`#-Xms2703m`
`#-Xmx2703m`
`#-XX:MaxDirectMemorySize=2703m`
-XX:+UnlockDiagnosticVMOptions
-XX:+LogVMOutput
-XX:LogFile=../sonatype-work/nexus3/log/jvm.log
-XX:-OmitStackTraceInFastThrow
-Djava.net.preferIPv4Stack=true
-Dkaraf.home=.
-Dkaraf.base=.
-Dkaraf.etc=etc/karaf
-Djava.util.logging.config.file=etc/karaf/java.util.logging.properties
-Dkaraf.data=../sonatype-work/nexus3
-Dkaraf.log=../sonatype-work/nexus3/log
-Djava.io.tmpdir=../sonatype-work/nexus3/tmp
-Dkaraf.startLocalConsole=false
#
# additional vmoptions needed for Java9+
#
# --add-reads=java.xml=java.logging
# --add-exports=java.base/org.apache.karaf.specs.locator=java.xml,ALL-UNNAMED
# --patch-module=java.base=lib/endorsed/org.apache.karaf.specs.locator-4.2.9.jar
# --patch-module=java.xml=lib/endorsed/org.apache.karaf.specs.java.xml-4.2.9.jar
# --add-opens=java.base/java.security=ALL-UNNAMED
# --add-opens=java.base/java.net=ALL-UNNAMED
# --add-opens=java.base/java.lang=ALL-UNNAMED
# --add-opens=java.base/java.util=ALL-UNNAMED
# --add-opens=java.naming/javax.naming.spi=ALL-UNNAMED
# --add-opens=java.rmi/sun.rmi.transport.tcp=ALL-UNNAMED
# --add-exports=java.base/sun.net.www.protocol.http=ALL-UNNAMED
# --add-exports=java.base/sun.net.www.protocol.https=ALL-UNNAMED
# --add-exports=java.base/sun.net.www.protocol.jar=ALL-UNNAMED
# --add-exports=jdk.xml.dom/org.w3c.dom.html=ALL-UNNAMED
# --add-exports=jdk.naming.rmi/com.sun.jndi.url.rmi=ALL-UNNAMED
#
# comment out this vmoption when using Java9+
#
-Djava.endorsed.dirs=lib/endorsed

```

构建运行Dockerfile

```bash
docker build -t nexus:3 ./
docker run --rm -v /opt/sonatype-work/nexus3/:/opt/sonatype-work/nexus3/ -p 8081:8081 nexus:3
```

网页访问：192.168.0.27:8081



# k8s部署nexus

需要将nexus固定在以上节点

这一次使用nodeAffinity 强亲和，上次jenkins使用nodeSelector

这一次使用hostPort，上次使用service

暴露服务

- [ ] hostport
- [ ] hostnetwork
- [ ] service

绑定主机

- [ ] nodename
- [ ] nodeSelector
- [ ] nodeaffinity
- [ ] podaffinity

## 节点标签

```bash
root@node06:~/weizhixiu/dockerfile/nexus-dockerfile# kubectl label node master01.youwoyouqu.io nexus="true"
```



## yaml

```bash
root@node06:~/weizhixiu/dockerfile/nexus-dockerfile# cat nexus-deployment.yaml 
apiVersion: v1
kind: Namespace
metadata:
    name: nexus
---
apiVersion: v1
data:
  .dockerconfigjson: eyJhdXRocyI6eyJoYXJib3IueW91d295b3VxdS5pbyI6eyJ1c2VybmFtZSI6ImFkbWluIiwicGFzc3dvcmQiOiJ3ZWl6aGl4aXUxMjMiLCJhdXRoIjoiWVdSdGFXNDZkMlZwZW1ocGVHbDFNVEl6In19fQ==
kind: Secret
metadata:
  name: registry
  namespace: nexus
type: kubernetes.io/dockerconfigjson
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nexus-deploy-label
  name: nexus-deploy
  namespace: nexus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nexus-deploy-selector
  template:
    metadata:
      labels:
        app: nexus-deploy-selector
    spec:
      tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: Equal
        effect: NoSchedule
      - key: "node.kubernetes.io/not-ready"
        operator: Exists
      affinity:
        nodeAffinity: 
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - {key: nexus, operator: In, values: ["true", "association_degree"]}
      imagePullSecrets:
      - name: registry
      volumes:
      - name: nexus-data
        hostPath:
          path: /opt/sonatype-work/nexus3/
          type: DirectoryOrCreate
      containers:
      - image: harbor.youwoyouqu.io/devops/nexus:3
        name: nexus
        volumeMounts:
        - name: nexus-data
          mountPath: /opt/sonatype-work/nexus3/
        ports:
        - containerPort: 8081
          protocol: TCP
          hostPort: 8081
          name: http
        resources: 
          limits:
            cpu: 500m
            memory: 500Mi
          requests:
            cpu: 200m
            memory: 200Mi
        livenessProbe: 
          initialDelaySeconds: 300
          successThreshold: 1
          failureThreshold: 2 
          periodSeconds: 3
          tcpSocket:
            port: 8081
        readinessProbe: 
          initialDelaySeconds: 60 
          successThreshold: 1
          failureThreshold: 2 
          periodSeconds: 3 
          tcpSocket:
            port: 8081
```

> 参考：http://blog.mykernel.cn/2021/02/02/Pod%E8%B5%84%E6%BA%90%E8%B0%83%E5%BA%A6/#%E7%A1%AC%E4%BA%B2%E5%92%8C-%E5%A4%9Akey

## build脚本

```bash
root@node06:~/weizhixiu/dockerfile/nexus-dockerfile# cat build.sh 
docker build -t harbor.youwoyouqu.io/devops/nexus:3 ./
```



## 验证端口

```bash
root@node06:~/weizhixiu/dockerfile/nexus-dockerfile# iptables -t nat -vnL | grep 8081
    0     0 CNI-HOSTPORT-SETMARK  tcp  --  *      *       10.244.0.0/24        0.0.0.0/0            tcp dpt:8081
    0     0 CNI-HOSTPORT-SETMARK  tcp  --  *      *       127.0.0.1            0.0.0.0/0            tcp dpt:8081
  187  9724 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8081 to:10.244.0.8:8081
  187  9724 CNI-DN-b6cdf5d13a4f43a41a6d0  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* dnat name: "cbr0" id: "46d1bac0522e80091c8787c25ebb7df3576cb7aa33284f7fb20d06661c7f44f2" */ multiport dports 8081
```











