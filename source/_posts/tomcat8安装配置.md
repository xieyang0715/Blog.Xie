---
title: tomcat8安装配置
tags:
  - java
photos:
  - http://myapp.img.mykernel.cn/tomcat.png
date: 2021-11-13 13:57:06
---

# 前言

tomcat 7 在web管理上的配置正常，参考: https://github.com/slcnx/tomcat/commits/master

但是在tomcat 8 就异常了，原因是 [tomcat-8.5](https://tomcat.apache.org/tomcat-8.5-doc/changelog.html)

> The Manager and Host Manager applications are now only accessible via `localhost` by default. (markt)

详细安装位置: https://tomcat.apache.org/tomcat-8.5-doc/deployer-howto.html#Installation

<!--more-->



# 安装与配置

1. 进入下载网址 https://tomcat.apache.org/download-80.cgi

2. 下载/解压/展开/进入tomcat目录

3. 配置`conf/tomcat-users.xml`

   ```xml
     <role rolename="manager-gui"/>
     <role rolename="manager-status"/>
     <role rolename="admin-gui"/>
     <role rolename="tomcat"/>
     <user username="tomcat" password="tomcatpass" roles="manager-gui,tomcat,manager-status,admin-gui"/>
   </tomcat-users>
   ```

4. 配置页面可以远程访问

   `webapps/manager/META-INF/context.xml`

   ```xml
     <Valve className="org.apache.catalina.valves.RemoteAddrValve"
            allow=".*|::1|0:0:0:0:0:0:0:1" />
   ```

   `webapps/host-manager/META-INF/context.xml`

   ```xml
     <Valve className="org.apache.catalina.valves.RemoteAddrValve"
            allow=".*|::1|0:0:0:0:0:0:0:1" />
   ```

