---
title: java环境配置
date: 2022-04-06 11:40:46
categories: java
toc: true
---

#### 一、java SDK下载

地址：https://www.oracle.com/java/technologies/downloads/

#### 二、环境变量配置

##### 1.打开 环境变量窗口

右键 This PC(此电脑) -> Properties（属性） -> Advanced system settings（高级系统设置） -> Environment Variables（环境变量）

<!--more-->

##### 2.新建JAVA_HOME 变量

新建系统变量（一般在路径C:\Program Files\Java下，具体路径需要看安装在C:\Program Files\Java路径下的版本）

变量名：JAVA_HOME

变量值（JDK安装路径）：C:\Program Files\Java\jdk-17.0.1

![image-20220406112701377](http://r61ygz0f5.hn-bkt.clouddn.com/image-20220406112701377.png)

##### 3.新建/修改 CLASSPATH 变量

变量名：CLASSPATH
变量值：.;%JAVA_HOME%\lib\dt.jar;%JAVA_HOME%\lib\tools.jar;

![image-20220406113254122](http://r61ygz0f5.hn-bkt.clouddn.com/image-20220406113254122.png)

##### 4. 修改Path 变量

新建两条路径：

%JAVA_HOME%\bin
%JAVA_HOME%\jre\bin

![image-20220406113516902](http://r61ygz0f5.hn-bkt.clouddn.com/image-20220406113516902.png)

##### 5.检查Java环境

cmd窗口输入java 或者 java -version

![image-20220406113656678](http://r61ygz0f5.hn-bkt.clouddn.com/image-20220406113656678.png)
