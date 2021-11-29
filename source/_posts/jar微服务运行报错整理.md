---
title: jar微服务运行报错整理
date: 2020-12-18 01:43:59
tags: 
- 生产小经验
- java
---





所有报错均来自elk查询message: "exception"

![image-20201218095359841](http://myapp.img.mykernel.cn/image-20201218095359841.png)

<!--more-->

# 1. mysql连接问题

![image-20201218094447591](http://myapp.img.mykernel.cn/image-20201218094447591.png)

1. 查看jar文件读取的java代码配置文件是哪个环境

   ```bash
   [root@k8s-master1 admin]# cat run.sh 
   #!/bin/bash
   
   /usr/share/filebeat/bin/filebeat -e -c /etc/filebeat/filebeat.yml -path.home /usr/share/filebeat -path.config /etc/filebeat -path.data /var/lib/filebeat -path.logs /var/log/filebeat > /dev/null &
   nohup /usr/local/jdk/bin/java -jar /apps/app/admin-0.0.1-SNAPSHOT.jar --spring.profiles.active=pro &>> /apps/logs/app.log &
   tail -f /etc/hosts
   ```

   > 可以看到是spring.profiles.active=pro, 即pro配置文件
   >
   > ![image-20201218094623540](http://myapp.img.mykernel.cn/image-20201218094623540.png)

2. 配置中的mysql用户及密码是否已经授权

   ```sql
   GRANT ALL ON ynyq.* TO 'ynyq'@'%' IDENTIFIED BY 'user_pass_in_properties';
   ```



# 2. 读取配置文件找到配置

![image-20201218094924562](http://myapp.img.mykernel.cn/image-20201218094924562.png)

最后的 Could not resolve placeholder 'alipay.notify_url' in value "${alipay.notify_url}"

![image-20201218095141169](http://myapp.img.mykernel.cn/image-20201218095141169.png)

通过发布仓库的源码可以grep找到这个关键字，@value表示读配置文件的key, 而仓库的配置文件读取见 标题1 `1. mysql连接问题`

通知开发添加配置即可。

