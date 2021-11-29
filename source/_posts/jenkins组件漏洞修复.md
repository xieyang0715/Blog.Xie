---
title: jenkins组件漏洞修复
tags:
  - jenkins
photos:
  - http://myapp.img.mykernel.cn/image-20211112101513490.png
date: 2021-11-12 10:10:55
---

# 前言

阿里分析出服务器的`jenkins`中组件依赖的文件有漏洞

1. 升级`jenkins`
2. 替换组件依赖的文件

<!--more-->

# 升级`jenkins`

当我升级到[Jenkins 2.277.3](https://jenkins.io/)遇到一个问题，解决方案如下

https://www.ycblog.top/article?articleId=142&pageNum=1



# 替换文件

## 解决方案来源

https://issues.jenkins.io/browse/JENKINS-59708?page=com.atlassian.jira.plugin.system.issuetabpanels%3Aall-tabpanel

这里提到，替换文件

> Deployed applications can be hardened by replacing the commons-fileupload jar file in WEB-INF/lib with the fixed jar.

## 异常文件来源

通过阿里云的告警如图:

![image-20211112101513490](http://myapp.img.mykernel.cn/image-20211112101513490.png)

> 阿里云会详细告知哪个文件有问题

## 获取指定jar文件最新版本

参考: https://www.dev2qa.com/how-to-download-jars-from-maven-repository/



<strong  style="color: red">接下来就是替换文件，原文件可以移动到/tmp目录, 并启动jenkins了, 略...</strong>

