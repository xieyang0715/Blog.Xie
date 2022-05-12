---

title: docker安装mysql8
date: 2022-05-12 17:21:00
categories: 
   - [docker] 
toc: true
---



docker安装mysql8

<!--more-->

查看是否存在mysql镜像

```
docker images
```

拉取mysql镜像

```
docker pull mysql:latest
```

运行镜像

```
docker run -itd --name mysql -p 3306:3306 --restart=always -e TZ=Asia/Shanghai -e MYSQL_ROOT_PASSWORD=111111 mysql --character-set-server=utf8 --collation-server=utf8_general_ci
```

```
--restart always 容器退出时总是重启容器
-e MYSQL_ROOT_PASSWORD=admin 配置root用户的密码为 “admin”
-e TZ=Asia/Shanghai 设置容器时区为 亚洲/上海
-p 3306:3306 暴露容器的端口给主机，前面是主机端口，后面是容器端口
--character-set-server=utf8 --collation-server=utf8_general_ci 设置默认编码格式为utf8，解决中文乱码问题
```

查看日志

```
docker logs mysql
```

#进入容器

```
docker exec -it mysql bash
```

#登录mysql

```
mysql -u root -p
```

#添加远程登录用户

```
CREATE USER '用户名'@'%' IDENTIFIED WITH mysql_native_password BY '111111';
GRANT ALL PRIVILEGES ON *.* TO '用户名'@'%';
```



[MySQL8.0 创建用户及授权]: https://blog.csdn.net/baidu_25986059/article/details/104042858

