---
title: "python-django搭建hello world"
date: 2021-05-18 14:22:14
tags:
- python
- django
---

# django框架

django框架，开发的是TCP/IP协议的http协议（应用层协议），用于实现网站或者app的后端。

django开发有前后端一体，通过后端的数据直接把数据结合模板渲染成网页，传递给用户即可，目前不流行了，app一般只能接收json。

前后端分离，后端只提供序列化的数据json/xml(json主流)，前端使用前端框架(vue, react)，应用：网站、app。



我们在开发新闻站点时，考虑到新闻站点在页面框架上不会经常变动，只是页面上的数据会不停的变化，例如：

![image-20210518142759004](http://myapp.img.mykernel.cn/image-20210518142759004.png)

- 一般数据存放在数据库中

- 页面上的新闻需要有投稿人在后台定期投放

所以新闻站点，一般需要`model`来管理数据，`admin`管理更新新闻数据，每次只需要将变动的数据插入事先定义好的`html模板`，渲染出html文件，在页面上展示即可。

django就可以快速搭建web应用，开发只关心对数据的处理。



# http事务（请求、响应）

我在浏览器、app、ajax、python爬虫上，输入一个地址，发起请求。然后立马看到结果。

![image-20210518143950807](http://myapp.img.mykernel.cn/image-20210518143950807.png)

> 服务程序：接收http请求报文，按http协议解析http报文

django就是一个服务端

![image-20210518152227299](http://myapp.img.mykernel.cn/image-20210518152227299.png)

> 1. 前端请求：真实用户点浏览器、app、ajax、爬虫
> 2. 请求到达服务端(gunicorn/uwsgi, flask调试app.run() 部署 gunicorn, django调试, runserver 部署 uwsgi; )：WTBF
>    - 请求交给django的wsgi.py 文件，接收请求，解析请求，并把请求封装为request对象，传递给django框架。
>    - 由settings.py中的MIDDLEWARE中的中间件（请求前置勾子和后置勾子）处理。正向查找middleware.
>    - 将请求通过url.py路由，到app的urls.py, 再映射到views函数。
>    - views函数返回响应，再由后置中间件处理，反向查找settings.py中的MIDDLEWARE.
>    - 将请求交给wsgi.py，进行封装响应，发送响应给用户。
> 3. 用户接收响应

# web框架意义

搭建web应用程序

避免相同代码部分重复编写，只关心业务逻辑实现。 http请求由框架实现，我们只在具体地方填业务代码

# web本质 

接收请求解析 ，处理，响应

# django特点

1. 重量，框架支持大量的模块和功能，而flask不支持，轻量。
2. 由于框架原生支持大量的功能，所以可以减少你写代码量。
3. 功能：
   - 自动化脚本生成工程目录
   - 模板、表单、admin管理站点、文件管理、认证权限、session机制、缓存

4. MTV模式(module, template, view), 程序设计模式有MVC,  分工、解耦，代码块之间降低耦合，增强代码的可扩展性和可移值性。

   - mvc模式：model, view, controller, 对应jango的model, template, view
     - model 数据
     - view 展示，django是template
     - controller 控制逻辑，django是views.py文件里面均是从model拿数据，加工，入库或响应。

   > 可以从下面的hello world项目的文件目录结构可以看出MTV模式
   >
   > ```bash
   > (slc356) [root@afc5ccded6af django_project]# tree
   > .
   > |-- db.sqlite3
   > |-- django_project
   > |   |-- __init__.py
   > |   |-- settings.py
   > |   |-- urls.py
   > |   `-- wsgi.py
   > |-- manage.py
   > `-- users
   >     |-- admin.py
   >     |-- apps.py
   >     |-- __init__.py
   >     |-- migrations
   >     |   |-- __init__.py
   >     |-- models.py
   >     |-- tests.py
   >     |-- urls.py
   >     `-- views.py
   > ```

# 搭建django服务端

 ## 准备

docker环境、chrome浏览器、pycharm

## 搭建环境

进入cmd，输入以下命令，（windows会弹出`share it`通知），进入docker控制台

```bash
docker run --name django -d -v g:\django_data:/projects/django/django -p 32765:22 -p 32764:8000 slcnx/pyenv-python:3.6.5-virtualenv-django
```

![image-20210518144538320](http://myapp.img.mykernel.cn/image-20210518144538320.png)

## 准备一个项目

一个项目可包含1个或多个模块

![image-20210518144731871](http://myapp.img.mykernel.cn/image-20210518144731871.png)

直接通过django的内置命令完成准备django项目, 会自动生成一个目录结构。- 

- 执行命令给的项目名，会在执行命令的当前目录生成同名的目录，django_project
- 项目目录django_project目录下有与项目目录同名的目录，里面包含settings, urls, wsgi三个python文件，此目录作用是对项目进行配置，入口路由，wsgi是django的接收请求程序。
- 项目目录下一级的manage.py，是一个脚本，用来管理这个项目

## 准备一个app

项目只是容器，app才是真正工作的，app就是模块

示例：为这个项目准备一个users模块

![image-20210518145626032](http://myapp.img.mykernel.cn/image-20210518145626032.png)



## 启动项目

![image-20210518145701679](http://myapp.img.mykernel.cn/image-20210518145701679.png)

网页访问 localhost:32764

# 准备hello world

## pycharm加载项目

方便编写django代码

pycharm中File -> Open, 选择新建的项目名

![image-20210518145808765](http://myapp.img.mykernel.cn/image-20210518145808765.png)

![image-20210518145852200](http://myapp.img.mykernel.cn/image-20210518145852200.png)

然后File -> settings -> Project: django_project -> python interpreter -> add 添加一个python解释器

![image-20210518150134299](http://myapp.img.mykernel.cn/image-20210518150134299.png)



## 编写hello world

在pycharm中编写，由于上面已经在docker中运行，django默认机制，你一编写代码，就会热加载代码，即时生效，就不需要重启django.

### 用户请求地址

![image-20210518150336900](http://myapp.img.mykernel.cn/image-20210518150336900.png)

### 请求到达django, url

进入项目的URL

![image-20210518150457130](http://myapp.img.mykernel.cn/image-20210518150457130.png)

拷贝urls.py文件，至users目录下，如下图结果。

include users的urls, 点击 蓝色的`create function index() in module views.py`

![image-20210518150630425](http://myapp.img.mykernel.cn/image-20210518150630425.png)

> from . import views, 直接从当前位置导入views模块 

将会跳转到views.py, 并自动生成了函数. django框架会把这个请求url的请求封装成request对象，传递给函数，并且views中的函数只需要响应httpResponse对象，由wsgi.py 程序来将httpResponse对象转换成响应报文

![image-20210518151044590](http://myapp.img.mykernel.cn/image-20210518151044590.png)

### 现在导入应用到项目 

导入就是在settings.py 中的INSTALLED_APPS后面加一行，新模块的apps中的类名 

![image-20210518151205597](http://myapp.img.mykernel.cn/image-20210518151205597.png)

![image-20210518151220189](http://myapp.img.mykernel.cn/image-20210518151220189.png)

### 现在再次访问

![image-20210518151316533](http://myapp.img.mykernel.cn/image-20210518151316533.png)

