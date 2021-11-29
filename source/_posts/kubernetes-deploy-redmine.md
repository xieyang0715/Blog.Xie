---
title: kubernetes-deploy-redmine
date: 2021-03-05 03:23:49
tags:
- kubernetes
---





# Docker

https://hub.docker.com/_/redmine?tab=tags&page=1&ordering=last_updated

```bash
root@master01:/data/weizhixiu/dockerfile/redmine-dockerfile# docker pull redmine:alpine
```

新建库

```bash
CREATE DATABASE redmine CHARACTER SET utf8;
GRANT ALL PRIVILEGES ON redmine.* TO 'redmine'@'%' IDENTIFIED BY 'redminep@ss';
```

启动

```bash
root@master01:/data/weizhixiu/dockerfile/redmine-dockerfile# docker run -it -e REDMINE_DB_MYSQL="192.168.0.171" -e REDMINE_DB_PASSWORD="redminep@ss" -e REDMINE_DB_USERNAME="redmine" -e REDMINE_DB_DATABASE="redmine"  -v /opt/redmine_data:/usr/src/redmine/files -p 3000:3000 redmine:alpine

```

验证监听

```bash
root@master01:/data/weizhixiu/dockerfile/redmine-dockerfile# ss -tnl | grep 3000
LISTEN   0         20480               0.0.0.0:3000             0.0.0.0:*       
```

访问 http://192.168.0.246:3000/

账号：admin

密码：admin

一定要先加载默认配置，否则不能新建问题

![img](http://myapp.img.mykernel.cn/7175ad50-f418-11e4-96ed-1369b38280bd.png)

# K8S deployment

## 准备镜像

https://docs.webfaction.com/software/redmine.html

https://guides.rubyonrails.org/action_mailer_basics.html#action-mailer-configuration



```bash
docker tag redmine:alpine harbor.youwoyouqu.io/devops/redmine:alpine
root@master01:/data/weizhixiu/dockerfile/redmine-dockerfile# docker push harbor.youwoyouqu.io/devops/redmine:alpine
```

先在一个容器中测试通过可以发邮件

### 准备配置

`root@master01:/data/weizhixiu/dockerfile/redmine-dockerfile# cat configuration.yml `

```bash
default:
  email_delivery:
    delivery_method: :smtp
    smtp_settings:
+      tls: true
      enable_starttls_auto: true
      address: smtp.exmail.qq.com
      port: 465
      domain: youninyouqu.com
      authentication: :login
      user_name: angole@youninyouqu.com
      password: xxxxxxx
  attachments_storage_path:
  autologin_cookie_name:
  autologin_cookie_path:
  autologin_cookie_secure:
  scm_subversion_command:
  scm_mercurial_command:
  scm_git_command:
  scm_cvs_command:
  scm_bazaar_command:
  scm_subversion_path_regexp:
  scm_mercurial_path_regexp:
  scm_git_path_regexp:
  scm_cvs_path_regexp:
  scm_bazaar_path_regexp:
  scm_filesystem_path_regexp:
  scm_stderr_log_file:
  database_cipher_key:
  minimagick_font_path:
production:
development:
```

> http://m.itboth.com/d/7NZVfa/eoferror:-end-of-file-reached-mail-file-redmine

```bash
root@master01:/data/weizhixiu/dockerfile/redmine-dockerfile# cat database.yml
production:
  adapter: mysql2
  database: redmine1
  host: 192.168.0.171
  port: 3306
  username: redmine
  password: redminep@ss
  encoding: utf8mb4
development:
  adapter: mysql2
  database: redmine1_development
  host: 192.168.0.171
  port: 3306
  username: redmine
  password: redminep@ss
  encoding: utf8mb4
test: 
  adapter: postgresql
  database: redmine1_test
  host: 192.168.0.171
  port: 3306
  username: redmine
  password: redminep@ss
  encoding: utf8mb4
```

### 启动镜像

```bash
# 获取配置目录
root@master01:/data/weizhixiu/dockerfile/redmine-dockerfile# docker run  --rm -it  harbor.youwoyouqu.io/devops/redmine:alpine sh
/usr/src/redmine # readlink -f config/configuration.yml
/usr/src/redmine/config/configuration.yml
/usr/src/redmine # readlink -f config/database.yml
/usr/src/redmine/config/database.yml
/usr/src/redmine # readlink -f files/
/usr/src/redmine/files


# 当前dockerfile目录的文件
root@master01:/data/weizhixiu/dockerfile/redmine-dockerfile# readlink -f database.yml 
/data/weizhixiu/dockerfile/redmine-dockerfile/database.yml
root@master01:/data/weizhixiu/dockerfile/redmine-dockerfile# readlink -f configuration.yml 
/data/weizhixiu/dockerfile/redmine-dockerfile/configuration.yml

# 启动
root@master01:/data/weizhixiu/dockerfile/redmine-dockerfile# docker run -v /opt/redmine_data1/:/usr/src/redmine/files -v /data/weizhixiu/dockerfile/redmine-dockerfile/configuration.yml:/usr/src/redmine/config/configuration.yml -v /data/weizhixiu/dockerfile/redmine-dockerfile/database.yml:/usr/src/redmine/config/database.yml -p 3000:3000  --rm -it  harbor.youwoyouqu.io/devops/redmine:alpine
```



### 配置邮件标签

![image-20210309131156857](http://myapp.img.mykernel.cn/image-20210309131156857.png)

### 测试发送邮件

![image-20210309131222104](http://myapp.img.mykernel.cn/image-20210309131222104.png)

### 收邮件

企业邮箱添加收集规则

![image-20210309130957803](http://myapp.img.mykernel.cn/image-20210309130957803.png)![image-20210309131012338](http://myapp.img.mykernel.cn/image-20210309131012338.png)

![image-20210309131132256](http://myapp.img.mykernel.cn/image-20210309131132256.png)

## 写dockerfile

```bash
root@master01:/data/weizhixiu/dockerfile/redmine-dockerfile# cat Dockerfile
FROM harbor.youwoyouqu.io/devops/redmine:alpine

ADD configuration.yml  database.yml /usr/src/redmine/config/

```

构建

```bash
docker build -t harbor.youwoyouqu.io/devops/redmine:alpine-email ./
root@master01:/data/weizhixiu/dockerfile/redmine-dockerfile# docker push harbor.youwoyouqu.io/devops/redmine:alpine-email
```



## yaml

```diff
root@master01:/data/weizhixiu/dockerfile/redmine-dockerfile# cat redmine-deployment.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: redmine
---
apiVersion: v1
data:
+  TZ: "Asia/Shanghai"
kind: ConfigMap
metadata:
  name: redmine-config
  namespace: redmine
---
apiVersion: v1
data:
  .dockerconfigjson: eyJhdXRocyI6eyJoYXJib3IueW91d295b3VxdS5pbyI6eyJ1c2VybmFtZSI6ImFkbWluIiwicGFzc3dvcmQiOiJ3ZWl6aGl4aXUxMjMiLCJhdXRoIjoiWVdSdGFXNDZkMlZwZW1ocGVHbDFNVEl6In19fQ==
kind: Secret
metadata:
  name: registry
  namespace: redmine
type: kubernetes.io/dockerconfigjson
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: redmine-deploy-label
  name: redmine-deploy
  namespace: redmine
spec:
+  strategy:
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
    type: RollingUpdate
  replicas: 1
  selector:
    matchLabels:
      app: redmine-deploy-selector
  template:
    metadata:
      labels:
        app: redmine-deploy-selector
    spec:
      tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: Equal
        effect: NoSchedule
      imagePullSecrets:
      - name: registry
      volumes:
      - name: redmine-data
        hostPath:
          path: /opt/redmine_data1
          type: DirectoryOrCreate
      nodeSelector:
        redmine: "true"
      containers:
+      - image: harbor.youwoyouqu.io/devops/redmine:alpine-email
        name: redmine-pod
        envFrom: 
        - configMapRef:
            name: redmine-config
            optional: true
        volumeMounts:
        - name: redmine-data
          mountPath: /usr/src/redmine/files
        ports:
+        - containerPort: 3000
          name: http
+          hostPort: 3000
        resources: 
          limits:
            cpu: 500m
            memory: 500Mi
          requests:
            cpu: 200m
            memory: 200Mi
+        livenessProbe: 
          initialDelaySeconds: 300
          successThreshold: 1
          failureThreshold: 2 
          periodSeconds: 3
          tcpSocket:
            port: 3000
+        readinessProbe: 
          initialDelaySeconds: 60 
          successThreshold: 1
          failureThreshold: 2 
          periodSeconds: 3 
          tcpSocket:
            port: 3000
```



提交 

```bash
root@master01:/data/weizhixiu/dockerfile/redmine-dockerfile# kubectl apply -f redmine-deployment.yaml
```

标签

```bash
root@master01:/data/weizhixiu/dockerfile/redmine-dockerfile# kubectl label node master01.youwoyouqu.io redmine="true"
```

## 验证

http://192.168.0.246:3000/

删除pod, 查看是否可以重新登陆



## 配置nginx proxy

```bash
[root@wzx conf.d]# cat redmine.conf 
server {
	listen 80;
	server_name redmine.youwoyouqu.io;
	location / {
            proxy_set_header Host $host;
            proxy_pass http://192.168.0.27:3000;
            proxy_connect_timeout 900s;
            proxy_read_timeout 900s;    
            proxy_send_timeout 900s;
        }
}
```

![image-20210305183947282](http://myapp.img.mykernel.cn/image-20210305183947282.png)



# 用户管理

现在开发人员需要添加账号，需要事先准备各位开发人员的邮件账号

假设开发人员邮箱：如下

![image-20210309132545083](http://myapp.img.mykernel.cn/image-20210309132545083.png)

查看邮件

![image-20210309132619192](http://myapp.img.mykernel.cn/image-20210309132619192.png)

登陆后，重置密码后，不能修改登陆名了

![image-20210309132706486](http://myapp.img.mykernel.cn/image-20210309132706486.png)

## 获取开发人员的邮件

写邮件处，写名字就出来了

![image-20210309132856655](http://myapp.img.mykernel.cn/image-20210309132856655.png)

创建账号即可，通知开发更新基本信息即可

![image-20210309133250970](http://myapp.img.mykernel.cn/image-20210309133250970.png)

## 配置redmine权限

今天发现测试人员不能更新问题，指派给哪个开发

![image-20210312100747923](http://myapp.img.mykernel.cn/image-20210312100747923.png)

![image-20210312100728076](http://myapp.img.mykernel.cn/image-20210312100728076.png)

# 备份数据库

```bash
[root@wzx redmine]# cat /weizhixiu/redmine/redmine.sh
#!/bin/bash
install -dv /weizhixiu/redmine/backup/
mysqldump -uredmine -h192.168.0.171 -predminep@ss --single-transaction  --skip-add-locks --skip-lock-tables redmine1 | gzip > /weizhixiu/redmine/backup/redmine-$(date +'%F').gz
find /weizhixiu/redmine/backup/ -mtime +3 -name "*.gz" -delete


# 添加crontab
crontab -e
#Backup Redmine
00 02 * * * /weizhixiu/redmine/redmine.sh > /dev/null
```

测试恢复，新建 redmine

```bash
[root@wzx redmine]# cat recover.sh 
gunzip -c backup/redmine-....gz  | mysql  -uredmine -h192.168.0.171 -predminep@ss -Dxxxx
```

