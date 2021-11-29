---
title: 学习django框架方法
date: 2021-05-18 17:16:13
tags:
- python
- django
---

> http://liushen.xyz/%E8%AF%BE%E4%BB%B6/DRF%E6%A1%86%E6%9E%B6%E8%AE%B2%E4%B9%89/
>
> 前文：http://blog.mykernel.cn/2021/05/18/python-django%E6%90%AD%E5%BB%BAhello-world/

# web框架学习方法

	框架基础：搭建、路由、视图
	如何获取请求数据
	如何构造响应数据
	如何使用类视图和中间层
	框架提供的其他功能组件的使用：数据库、模板、表单、admin、认证、限流、文档、异常处理、过滤



# 框架基础

## 搭建

### docker运行虚拟环境

```bash
docker run --name django-2 -d -v g:\django_data_2:/projects/django/django -p 32765:22 -p 32764:8000 slcnx/pyenv-python-virtualenv:3.6.5-virtualenv-django

# /projects/django/django 此目录就是镜像中制作的虚拟环境的子目录。 /projects/django/ 虚拟环境是python 3.6.5
# g:\django_data 共享出这个目录，方便pycharm加载目录，进行编写代码。
# 暴露22，可以登陆。暴露8000，django启动的调试页面在8000端口。
```

### ssh连接

```bash
[c:\~]$ ssh 127.0.0.1 32765

[root@2e2144919378 ~]# cd /projects/django/django/
(slc365) [root@2e2144919378 django]# 

```

### 准备项目

```bash
(slc365) [root@2e2144919378 django]# django-admin startproject demo
```

### windows中加载项目

![image-20210527110750844](http://myapp.img.mykernel.cn/image-20210527110750844.png)

### 准备模块

```bash
(slc365) [root@2e2144919378 django]# cd demo/
(slc365) [root@2e2144919378 demo]# python manage.py startapp users
Traceback (most recent call last):
  File "manage.py", line 22, in <module>
    main()
  File "manage.py", line 18, in main
    execute_from_command_line(sys.argv)
  File "/root/.pyenv/versions/slc365/lib/python3.6/site-packages/django/core/management/__init__.py", line 419, in execute_from_command_line
    utility.execute()
  File "/root/.pyenv/versions/slc365/lib/python3.6/site-packages/django/core/management/__init__.py", line 395, in execute
    django.setup()
  File "/root/.pyenv/versions/slc365/lib/python3.6/site-packages/django/__init__.py", line 24, in setup
    apps.populate(settings.INSTALLED_APPS)
  File "/root/.pyenv/versions/slc365/lib/python3.6/site-packages/django/apps/registry.py", line 114, in populate
    app_config.import_models()
  File "/root/.pyenv/versions/slc365/lib/python3.6/site-packages/django/apps/config.py", line 301, in import_models
    self.models_module = import_module(models_module_name)
  File "/root/.pyenv/versions/3.6.5/lib/python3.6/importlib/__init__.py", line 126, in import_module
    return _bootstrap._gcd_import(name[level:], package, level)
  File "<frozen importlib._bootstrap>", line 994, in _gcd_import
  File "<frozen importlib._bootstrap>", line 971, in _find_and_load
  File "<frozen importlib._bootstrap>", line 955, in _find_and_load_unlocked
  File "<frozen importlib._bootstrap>", line 665, in _load_unlocked
  File "<frozen importlib._bootstrap_external>", line 678, in exec_module
  File "<frozen importlib._bootstrap>", line 219, in _call_with_frames_removed
  File "/root/.pyenv/versions/slc365/lib/python3.6/site-packages/django/contrib/auth/models.py", line 3, in <module>
    from django.contrib.auth.base_user import AbstractBaseUser, BaseUserManager
  File "/root/.pyenv/versions/slc365/lib/python3.6/site-packages/django/contrib/auth/base_user.py", line 48, in <module>
    class AbstractBaseUser(models.Model):
  File "/root/.pyenv/versions/slc365/lib/python3.6/site-packages/django/db/models/base.py", line 122, in __new__
    new_class.add_to_class('_meta', Options(meta, app_label))
  File "/root/.pyenv/versions/slc365/lib/python3.6/site-packages/django/db/models/base.py", line 326, in add_to_class
    value.contribute_to_class(cls, name)
  File "/root/.pyenv/versions/slc365/lib/python3.6/site-packages/django/db/models/options.py", line 207, in contribute_to_class
    self.db_table = truncate_name(self.db_table, connection.ops.max_name_length())
  File "/root/.pyenv/versions/slc365/lib/python3.6/site-packages/django/utils/connection.py", line 15, in __getattr__
    return getattr(self._connections[self._alias], item)
  File "/root/.pyenv/versions/slc365/lib/python3.6/site-packages/django/utils/connection.py", line 62, in __getitem__
    conn = self.create_connection(alias)
  File "/root/.pyenv/versions/slc365/lib/python3.6/site-packages/django/db/utils.py", line 204, in create_connection
    backend = load_backend(db['ENGINE'])
  File "/root/.pyenv/versions/slc365/lib/python3.6/site-packages/django/db/utils.py", line 111, in load_backend
    return import_module('%s.base' % backend_name)
  File "/root/.pyenv/versions/3.6.5/lib/python3.6/importlib/__init__.py", line 126, in import_module
    return _bootstrap._gcd_import(name[level:], package, level)
  File "/root/.pyenv/versions/slc365/lib/python3.6/site-packages/django/db/backends/sqlite3/base.py", line 73, in <module>
    check_sqlite_version()
  File "/root/.pyenv/versions/slc365/lib/python3.6/site-packages/django/db/backends/sqlite3/base.py", line 69, in check_sqlite_version
    'SQLite 3.9.0 or later is required (found %s).' % Database.sqlite_version
django.core.exceptions.ImproperlyConfigured: SQLite 3.9.0 or later is required (found 3.7.17).

```

注释setting.py的sqllite, 再pycharm中操作注释, 选中中间3行，按着ctrl键，再按 /键，就可以快捷注释。

![image-20210527110947014](http://myapp.img.mykernel.cn/image-20210527110947014.png)

> demo 是项目名
>
> demo 是默认主模块
>
> demo/setting.py就是项目的配置文件，下文说修改配置，就是修改这个文件。
>
> demo/urls.py 项目的主路由
>
> manage.py 项目的管理脚本

再次尝试启动

```bash
(slc365) [root@2e2144919378 demo]# python manage.py runserver 0.0.0.0:8000
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).
May 27, 2021 - 08:57:09
Django version 3.2.3, using settings 'demo.settings'
Starting development server at http://0.0.0.0:8000/
Quit the server with CONTROL-C.
```

页面访问

![image-20210527165925699](http://myapp.img.mykernel.cn/image-20210527165925699.png)

### pycharm基础环境准备

#### 准备python虚拟环境

![image-20210527164659186](http://myapp.img.mykernel.cn/image-20210527164659186.png)

> file -> settings 

#### 准备django

```bash
# 32位系统
pip install https://download.lfd.uci.edu/pythonlibs/q4trcu4l/cp36/mysqlclient-1.4.6-cp36-cp36m-win32.whl
# 64位系统
pip install https://download.lfd.uci.edu/pythonlibs/q4trcu4l/cp36/mysqlclient-1.4.6-cp36-cp36m-win_amd64.whl




pip install django django_redis pymysql Pillow djangorestframework  django-filter coreapi ipython

# django 框架
# django_redis 连接redis, 保存用户会话
# django连接mysql: pymysql, mysqlclient
# Pillow 图片
# djangorestframework DRF框架
# django-filter 查询列时，支持基于哪些字段过滤
# coreapi       基于视图集生成网页API文档，暴露给前端使用。
# ipython       python manage.py shell会使用，有自动补全功能
```

## 子模块

项目可以有多个子模块：用户模块、新闻管理模块、...，每个模块功能独立，而且模块功能可以复用。

```bash
(slc365) [root@2e2144919378 demo]# python manage.py startapp users
(slc365) [root@2e2144919378 demo]# 
```

```bash
(slc356) [root@08cf1c438c29 demo]# tree users/
users/
|-- __init__.py
|-- admin.py           # 后台运营管理网站配置地方
|-- apps.py            # 当前子应用的描述配置信息
|-- migrations         # 数据操作历史
|   `-- __init__.py 
|-- models.py          # 子应用数据库
|-- tests.py           # 子应用的单元测试
`-- views.py           # 子应用的业务逻辑
```

django放了子应用，但是项目不知道有这个应用，需要安装注册, 在项目配置文件中的`INSTALLED_APPS`导入子模块`users`中的`apps.UsersConfig`

![image-20210527175604930](http://myapp.img.mykernel.cn/image-20210527175604930.png)

```diff
# settings.py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

+    'users.apps.UsersConfig', # 导入users模块
]
```



## 视图定义

完成hello world

flask中, request, flask 全局 from flask import request

django, 必须首个参数为 HttpRequest对象，命名request

![image-20210527190259507](http://myapp.img.mykernel.cn/image-20210527190259507.png)

```diff
(slc365) [root@2e2144919378 demo]# cat users/views.py 
+from django.http import HttpResponse
from django.shortcuts import render


# Create your views here.


+def index(request):      # django 首个参数
+    return HttpResponse("hello world")
```



## 路由定义

### 定义方法

- demo/urls.py定义路由：主路由中定义路由
  - 直接引用函数，则不需要子路由
  - include()子模块，例如 /users/list/, /users/retrieve/ 2个url, 突然增加一个模块 /booktest/list/, /booktest/retrieve/ 如果list, retrieve定义在模块中，可以直接复制出新模块，直接在主路由上加一个booktest。如果定义在主路由中，每个函数要写多次。
- users/urls.py定义路由：主路由使用include()时，模块中定义路由
  - 直接引用函数

### 主路由直接引用函数

![image-20210527192146694](http://myapp.img.mykernel.cn/image-20210527192146694.png)

```diff
(slc365) [root@2e2144919378 demo]# cat demo/urls.py 
"""demo URL Configuration

The `urlpatterns` list routes URLs to views. For more information please see:
    https://docs.djangoproject.com/en/3.2/topics/http/urls/
Examples:
Function views
    1. Add an import:  from my_app import views
    2. Add a URL to urlpatterns:  path('', views.home, name='home')
Class-based views
    1. Add an import:  from other_app.views import Home
    2. Add a URL to urlpatterns:  path('', Home.as_view(), name='home')
Including another URLconf
    1. Import the include() function: from django.urls import include, path
    2. Add a URL to urlpatterns:  path('blog/', include('blog.urls'))
"""
from django.contrib import admin
from django.urls import path

+from users import views

urlpatterns = [
    path('admin/', admin.site.urls),
    #     路由 ,      函数
+    path('users/', views.index)
]
```

> 注意：在xshell上始终还运行着 `(slc365) [root@2e2144919378 demo]# python manage.py runserver 0.0.0.0:8000`

现在网页上请求主页，不再是默认页面，当有子模型导入后，默认页面将不会显示。

![image-20210527192321123](http://myapp.img.mykernel.cn/image-20210527192321123.png)

网页上请求index, 将返回hello world.

![image-20210527192405802](http://myapp.img.mykernel.cn/image-20210527192405802.png)

### 主路由引用include

首先需要在子模块中定义urls.py, 其中有一个`urlpatterns`命名的列表

首先认真观察，现在users模块中并没有urls.py文件名。

![image-20210527192555832](http://myapp.img.mykernel.cn/image-20210527192555832.png)

现在新建一个urls.py文件, 直接在demo/urls.py进行拷贝，并在users目录上粘贴。粘贴后，打开2个urls.py

![image-20210531115158400](http://myapp.img.mykernel.cn/image-20210531115158400.png)

![image-20210531115204115](http://myapp.img.mykernel.cn/image-20210531115204115.png)

现在配置主路由，用户访问 /users/ 路由就Incloude到users模块的urls.py

![image-20210531115325587](http://myapp.img.mykernel.cn/image-20210531115325587.png)

![image-20210531115336023](http://myapp.img.mykernel.cn/image-20210531115336023.png)

主urls.py

```diff
from django.contrib import admin
+from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
+    path('users/', include('users.urls'))
]
```

> 注意：include('users.urls') 中间使用字符，django的url解释时， 会自动加载users模块中的urls文件中的urlpatterns, 并自动附加到users/目录后

users/urls.py文件，由于不需要admin了,  期望/users/index 访问hello world

```diff

from django.contrib import admin
from django.urls import path

from users import views

urlpatterns = [
    # path('admin/', admin.site.urls),
+    path('index/', views.index)
]
```

现在观察控制台

```bash
(slc365) [root@2e2144919378 demo]# python manage.py runserver 0.0.0.0:8000
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).
May 31, 2021 - 03:57:35
Django version 3.2.3, using settings 'demo.settings'
Starting development server at http://0.0.0.0:8000/
Quit the server with CONTROL-C.
```

现在网页访问`http://127.0.0.1:32764/users/index`, 可以观察到网页输出hello world.



## django配置文件

django配置文件指的是 django在执行`python manage.py startproject 项目名`之后，生成的项目名之下的 相同项目名目录下的setting.py文件的配置。

![image-20210531120049587](http://myapp.img.mykernel.cn/image-20210531120049587.png)

### 基本目录

```python
from pathlib import Path

# Build paths inside the project like this: BASE_DIR / 'subdir'.
BASE_DIR = Path(__file__).resolve().parent.parent
```

### DEBUG

开发时，打开DEBUG，所有修改会立即重载服务，并将报错详细打印在网页中。

生产中，一定要关闭DEBUG。

```python
# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = True
```

由于默认是DEUBG=True，所以直接修改users模块的视图函数，在不重新启动程序的情况下，再次访问网页中会出现详细的报错

```diff
from django.http import HttpResponse
from django.shortcuts import render


# Create your views here.


def index(request):
+    1 / 0
    return HttpResponse("hello world")
```

![image-20210531122615821](http://myapp.img.mykernel.cn/image-20210531122615821.png)

> Traceback, 会显示哪行代码有问题，而且会显示local 变量。

删除1/0时，再次访问，又恢复正常

### 访问控制

哪些主机可以访问django.

```python
ALLOWED_HOSTS = ['*'] # *表示 所有
```



### 数据库

django默认使用本地的sqllite, 此处我们直接使用mysql.

```python
# 启动mysql
docker run -v /var/lib/mysql:/data/mysql/data  -p 32763:3396 --name mysql -d slcnx/mariadb:5.6.49
            
# 配置密码
docker exec -it mysql bash
/mysql-5.6.49-linux-glibc2.12-x86_64//bin/mysqladmin -u root password 'new-password'

```

django配置mysql

```python
DATABASES = {
    # 'default': {
    #     'ENGINE': 'django.db.backends.sqlite3',
    #     'NAME': BASE_DIR / 'db.sqlite3',
    # }
    'default': {
        # Django MySQL database engine driver class.
        'ENGINE': 'django.db.backends.mysql',
        # MySQL database host ip.
        'HOST': '192.168.1.222',
        # port number.
        'PORT': '32761',
        # database name.
        'NAME': 'devops',
        # user name.
        'USER': 'root',
        # password
        'PASSWORD': '123python',
    }
}
```



### 本地化

现在访问admin页面, 显示英文

![image-20210531144913118](http://myapp.img.mykernel.cn/image-20210531144913118.png)

以下配置中文和时区

```python
LANGUAGE_CODE = 'zh-hans'

TIME_ZONE = 'Asia/Shanghai'
```

刷新网页, 变成中文



### 静态资源

仅在DEBUG=True时，才支持。django认为在线上运行，本身是动态的框架，提供静态文件的事件会有压力。
生产时，DEBUG=False, 用collectstatic命令收集静态文件，并交给静态文件服务器来提供。

```python
STATIC_URL = '/static/'
STATICFILES_DIRS = [BASE_DIR / 'static_files']
```

用户放在 根目录的static_files目录中的文件，可以在`debug=true`模式下通过/static/目录访问资源

![image-20210531145454984](http://myapp.img.mykernel.cn/image-20210531145454984.png)

![image-20210531145500407](http://myapp.img.mykernel.cn/image-20210531145500407.png)

### 路由命名

不包含include()的路由，命名。包含Include() 命名为名称空间。

#### 名称空间

![image-20210531150543871](http://myapp.img.mykernel.cn/image-20210531150543871.png)

访问`http://127.0.0.1:32764/users/index/`, 返回

```bash
/users/index/
[31/May/2021 15:03:47] "GET /users/index/ HTTP/1.1" 200 11
```

#### 非名称空间

![image-20210531150639572](http://myapp.img.mykernel.cn/image-20210531150639572.png)

> `demo/urls.py`
>
> `users/views.py`

访问`http://127.0.0.1:32764/say/`, 返回

```bash
/say/
[31/May/2021 15:04:41] "GET /say/ HTTP/1.1" 200 11
```

### url将/结束识别为目录

![image-20210531151236425](http://myapp.img.mykernel.cn/image-20210531151236425.png)

访问`http://127.0.0.1:32764/static/index.html`, 日志`[31/May/2021 15:11:07] "GET /static/1.png HTTP/1.1" 200 248884`

访问`http://127.0.0.1:32764/static/index.html/`, 日志`[31/May/2021 15:11:45] "GET /static/index.html/1.png HTTP/1.1" 404 1808`

```diff
urlpatterns = [
    path('admin/', admin.site.urls),
    path('users/', include(('users.urls','users'),namespace="users")),
+    path('say/', views.say, name='say')
]
```

django中，访问`http://127.0.0.1:32764/say` , 会自动跳转`http://127.0.0.1:32764/say/`, 2个URL都可以访问。

```diff
urlpatterns = [
    path('admin/', admin.site.urls),
    path('users/', include(('users.urls','users'),namespace="users")),
+    path('say', views.say, name='say')
]
```

django中，访问`http://127.0.0.1:32764/say/`, 404。仅在`http://127.0.0.1:32764/say`可以访问

# 获取请求数据

在views.py中定义的函数，接收的首个参数一定是reqeust, 他是 `HttpRequest`对象

定义函数，添加注释可以补全

```diff
+def index(request:HttpRequest):
+    request.补全
    return HttpResponse("hello world")
```



## http事务

### http请求

```http
GET /index.html HTTP/1.1
Header: Value
```

> 请求起始行：请求方法  请求URI HTTP版本
>
> 请求首部 https://blog.csdn.net/u011541946/article/details/96298259

## 获取请求携带的数据

### url携带

	非命名：url(r'^weather/([a-z]+)/(\d{4})/$', views.weather) 
	命名：url(r'^weather/(?P<city>[a-z]+)/(?P<year>\d{4})/$', views.weather) 

#### 非命名提取

将请求的目标数据放在url中，请求2018年的北京的天气,   /weather/beijing/2018

![image-20210531164631499](http://myapp.img.mykernel.cn/image-20210531164631499.png)

后端解析时，使用

```python
url(r'^weather/([a-z]+)/(\d{4})/$', views.weather)
```

访问`http://127.0.0.1:32764/weather/beijing/2021/`, 响应

```bash
beijing 2021
[31/May/2021 16:44:39] "GET /weather/beijing/2021/ HTTP/1.1" 200 11
```

一旦我们将函数修改参数对换，可以发现输出的值会对换

```diff
+def weather(reqeust,year, city):
    print(city, year)
    return HttpResponse("hello world")
```

```bash
2021 beijing
[31/May/2021 16:49:34] "GET /weather/beijing/2021/ HTTP/1.1" 200 11
```

> 这样city对应2021， year对应beijing, 不对了。

#### 命名提取

后端解析时，使用

```bash
url(r'^weather/(?P<city>[a-z]+)/(?P<year>\d{4})/$', views.weather)
```

无论将函数的参数怎么换位置，均正常

### 查询字符串

将请求的目标数据放在查询参数中，请求2018年的北京的天气,   /weather?city=beijing&year=2018

![image-20210531165645545](http://myapp.img.mykernel.cn/image-20210531165645545.png)

访问`http://127.0.0.1:32764/weather?city=beijing&year=2021`, 响应

```bash
request.GET <QueryDict: {'city': ['beijing'], 'year': ['2021']}>
request list ['beijing']
beijing 2021
[31/May/2021 16:55:30] "GET /weather?city=beijing&year=2021 HTTP/1.1" 200 11
```

> reqeust.GET -> QueryDict对象 返回字典 （不区别方法，仅是取查询参数）
>
> getlist同一个key返回列表。 表单的key可以相同

访问`http://127.0.0.1:32764/weather?city=beijing&year=2021&city=guangyuan`, 响应

```bash
request.GET <QueryDict: {'city': ['beijing', 'guangyuan'], 'year': ['2021']}>
request list ['beijing', 'guangyuan']
guangyuan 2021
[31/May/2021 16:57:21] "GET /weather?city=beijing&year=2021&city=guangyuan HTTP/1.1" 200 11
```

> 表单的key可以相同, get同一个key返回列表最后一个值。
>
> 不存在key, 返回空列表

### 请求体

```bash
request.POST, request.body 
body发送
格式不统一：
```

> 表单数据： request.POST 返回字典，form-data (多媒体表单)/x-www-form-urlencoded((普通表单) 均可以获取表单 

#### 关闭CSRF

非GET方法有CSRF防护。跨站伪造攻击

![image-20210531170414031](http://myapp.img.mykernel.cn/image-20210531170414031.png)

```diff
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
+    # 'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

#### 表单提交

![image-20210531170827976](http://myapp.img.mykernel.cn/image-20210531170827976.png)

使用postman提交表单

![image-20210531170736500](http://myapp.img.mykernel.cn/image-20210531170736500.png)

输出日志

```bash
<QueryDict: {}>
<QueryDict: {'a': ['1', '11'], 'b': ['2']}>
a 11
b 2
list a ['1', '11']
[31/May/2021 17:06:37] "POST /sayhello/ HTTP/1.1" 200 11
```

> get 获取列表最后一个值
>
> getlist 获取列表

##### 表单的原始数据

![image-20210531172055482](http://myapp.img.mykernel.cn/image-20210531172055482.png)

> x-www-form-urlencoded, 原始表单

![image-20210531172113549](http://myapp.img.mykernel.cn/image-20210531172113549.png)

请求时的日志

```bash
original data b'a=1&a=11&b=2'
<QueryDict: {}>
<QueryDict: {'a': ['1', '11'], 'b': ['2']}>
a 11
b 2
list a ['1', '11']
[31/May/2021 17:20:19] "POST /sayhello/ HTTP/1.1" 200 11

```

如果现在将content-type前面的勾去掉，会发现解析不了

![image-20210531172228798](http://myapp.img.mykernel.cn/image-20210531172228798.png)

```bash
original data b'a=1&a=11&b=2'
<QueryDict: {}>
<QueryDict: {}>
a None
b None
list a []
[31/May/2021 17:22:15] "POST /sayhello/ HTTP/1.1" 200 11

```

所以，后端识别表单依据类型，也可以换一种方式来提交表单

![image-20210531172329198](http://myapp.img.mykernel.cn/image-20210531172329198.png)

> 类型设定为表单

![image-20210531172351663](http://myapp.img.mykernel.cn/image-20210531172351663.png)

> 仅提交原始数据

后端输出日志，显示还是识别为表单

```bash
original data b'a=1&a=11&b=2'
<QueryDict: {}>
<QueryDict: {'a': ['1', '11'], 'b': ['2']}>
a 11
b 2
list a ['1', '11']
[31/May/2021 17:23:37] "POST /sayhello/ HTTP/1.1" 200 11

```

#### 前端传递JSON

![image-20210531171510110](http://myapp.img.mykernel.cn/image-20210531171510110.png)

```bash
request.body b'{\r\n    "a": 1,\r\n    "b": 2,\r\n    "a": 3\r\n}'
{'a': 3, 'b': 2}
[31/May/2021 17:13:24] "POST /sayhello/ HTTP/1.1" 200 11
```

```bash
request.body b'{\r\n    "a": 1,\r\n    "b": 2,\r\n    "ab": 3\r\n}'
{'a': 1, 'b': 2, 'ab': 3}
[31/May/2021 17:13:33] "POST /sayhello/ HTTP/1.1" 200 11
```

> json中相同的key，最终只会保留一个key, 值为最后一个key对应的值。

### 请求头

![image-20210531172625431](http://myapp.img.mykernel.cn/image-20210531172625431.png)

现在请求的响应结果 

```bash
original data b'a=1&a=11&b=2'
requst headers {'HOSTNAME': '2e2144919378', 'PYENV_ROOT': '/root/.pyenv', 'SHELL': '/bin/bash', 'TERM': 'xterm', 'HISTSIZE': '1000', 'SSH_CLIENT': '172.17.0.1 62568 22', 'SSH_TTY': '/dev/pts/0', 'PYENV_VERSION': 'slc365', 'USER': 'root', 'LS_COLORS': 'rs=0:di=01;34:ln=01;36:mh=00:pi=40;33:so=01;35:do=01;35:bd=40;33;01:cd=40;33;01:or=40;31;01:mi=01;05;37;41:su=37;41:sg=30;43:ca=30;41:tw=30;42:ow=34;42:st=37;44:ex=01;32:*.tar=01;31:*.tgz=01;31:*.arc=01;31:*.arj=01;31:*.taz=01;31:*.lha=01;31:*.lz4=01;31:*.lzh=01;31:*.lzma=01;31:*.tlz=01;31:*.txz=01;31:*.tzo=01;31:*.t7z=01;31:*.zip=01;31:*.z=01;31:*.Z=01;31:*.dz=01;31:*.gz=01;31:*.lrz=01;31:*.lz=01;31:*.lzo=01;31:*.xz=01;31:*.bz2=01;31:*.bz=01;31:*.tbz=01;31:*.tbz2=01;31:*.tz=01;31:*.deb=01;31:*.rpm=01;31:*.jar=01;31:*.war=01;31:*.ear=01;31:*.sar=01;31:*.rar=01;31:*.alz=01;31:*.ace=01;31:*.zoo=01;31:*.cpio=01;31:*.7z=01;31:*.rz=01;31:*.cab=01;31:*.jpg=01;35:*.jpeg=01;35:*.gif=01;35:*.bmp=01;35:*.pbm=01;35:*.pgm=01;35:*.ppm=01;35:*.tga=01;35:*.xbm=01;35:*.xpm=01;35:*.tif=01;35:*.tiff=01;35:*.png=01;35:*.svg=01;35:*.svgz=01;35:*.mng=01;35:*.pcx=01;35:*.mov=01;35:*.mpg=01;35:*.mpeg=01;35:*.m2v=01;35:*.mkv=01;35:*.webm=01;35:*.ogm=01;35:*.mp4=01;35:*.m4v=01;35:*.mp4v=01;35:*.vob=01;35:*.qt=01;35:*.nuv=01;35:*.wmv=01;35:*.asf=01;35:*.rm=01;35:*.rmvb=01;35:*.flc=01;35:*.avi=01;35:*.fli=01;35:*.flv=01;35:*.gl=01;35:*.dl=01;35:*.xcf=01;35:*.xwd=01;35:*.yuv=01;35:*.cgm=01;35:*.emf=01;35:*.axv=01;35:*.anx=01;35:*.ogv=01;35:*.ogx=01;35:*.aac=01;36:*.au=01;36:*.flac=01;36:*.mid=01;36:*.midi=01;36:*.mka=01;36:*.mp3=01;36:*.mpc=01;36:*.ogg=01;36:*.ra=01;36:*.wav=01;36:*.axa=01;36:*.oga=01;36:*.spx=01;36:*.xspf=01;36:', 'PYENV_DIR': '/projects/django/django/demo', 'PYENV_VIRTUALENV_INIT': '1', 'VIRTUAL_ENV': '/root/.pyenv/versions/3.6.5/envs/slc365', 'PYENV_VIRTUAL_ENV': '/root/.pyenv/versions/3.6.5/envs/slc365', 'PATH': '/root/.pyenv/versions/slc365/bin:/root/.pyenv/libexec:/root/.pyenv/plugins/python-build/bin:/root/.pyenv/plugins/pyenv-virtualenv/bin:/root/.pyenv/plugins/python-build/bin:/root/.pyenv/plugins/pyenv-virtualenv/bin:/root/.pyenv/plugins/pyenv-virtualenv/shims:/root/.pyenv/shims:/root/.pyenv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin', 'MAIL': '/var/spool/mail/root', 'PWD': '/projects/django/django/demo', 'PYENV_HOOK_PATH': '/root/.pyenv/pyenv.d:/usr/local/etc/pyenv.d:/etc/pyenv.d:/usr/lib/pyenv/hooks:/root/.pyenv/plugins/pyenv-virtualenv/etc/pyenv.d', '_OLD_VIRTUAL_PS1': '[\\u@\\h \\W]\\$ ', 'HISTCONTROL': 'ignoredups', 'HOME': '/root', 'SHLVL': '1', 'PYENV_SHELL': 'bash', 'LOGNAME': 'root', 'SSH_CONNECTION': '172.17.0.1 62568 172.17.0.2 22', 'LESSOPEN': '||/usr/bin/lesspipe.sh %s', 'DJANGO_SETTINGS_MODULE': 'demo.settings', 'TZ': 'Asia/Shanghai', 'RUN_MAIN': 'true', 'SERVER_NAME': '2e2144919378', 'GATEWAY_INTERFACE': 'CGI/1.1', 'SERVER_PORT': '8000', 'REMOTE_HOST': '', 'CONTENT_LENGTH': '12', 'SCRIPT_NAME': '', 'SERVER_PROTOCOL': 'HTTP/1.1', 'SERVER_SOFTWARE': 'WSGIServer/0.2', 'REQUEST_METHOD': 'POST', 'PATH_INFO': '/sayhello/', 'QUERY_STRING': '', 'REMOTE_ADDR': '172.17.0.1', 'CONTENT_TYPE': 'application/x-www-form-urlencoded', 'HTTP_USER_AGENT': 'PostmanRuntime/7.28.0', 'HTTP_ACCEPT': '*/*', 'HTTP_POSTMAN_TOKEN': '6cec539b-1589-4ba6-9de0-7ef6ede29f90', 'HTTP_HOST': '127.0.0.1:32764', 'HTTP_ACCEPT_ENCODING': 'gzip, deflate, br', 'HTTP_CONNECTION': 'keep-alive', 'wsgi.input': <django.core.handlers.wsgi.LimitedStream object at 0x7f972d8b3208>, 'wsgi.errors': <_io.TextIOWrapper name='<stderr>' mode='w' encoding='ANSI_X3.4-1968'>, 'wsgi.version': (1, 0), 'wsgi.run_once': False, 'wsgi.url_scheme': 'http', 'wsgi.multithread': True, 'wsgi.multiprocess': False, 'wsgi.file_wrapper': <class 'wsgiref.util.FileWrapper'>}
content type application/x-www-form-urlencoded
[31/May/2021 17:26:38] "POST /sayhello/ HTTP/1.1" 200 11
```

> 获取cookie...
> request.META('CONTENT_TYPE') 获取内容类型

### 请求方法

```python
def sayhello(request):
    print('original data', request.body)
    # print('requst headers', request.META)
    print('content type', request.META['CONTENT_TYPE'])
    print('request method: ',request.method) # 请求方法
    print('request user: ', request.user) # 登陆用户
    return HttpResponse("hello world")
```



# 获取响应数据

构建响应对象，构建的是响应报文

在views.py中定义的函数，return要求是`HttpResponse`对象

## http事务

### http响应

```http
HTTP/1.1 200 OK
Header: Value
body
```

> 响应起始行：请求版本 响应状态码 原因短语
>
> 响应首部 https://blog.csdn.net/qq_37964379/article/details/115282934



## 配置HttpResponse对象

![image-20210531174221540](http://myapp.img.mykernel.cn/image-20210531174221540.png)

请求`http://127.0.0.1:32764/resp`, 响应结果

```bash
Bad Request: /resp/
[31/May/2021 17:41:30] "GET /resp/ HTTP/1.1" 400 18
```

![image-20210531174316624](http://myapp.img.mykernel.cn/image-20210531174316624.png)

## 自定义响应头

![image-20210531174425369](http://myapp.img.mykernel.cn/image-20210531174425369.png) 

响应结果的首部有

![image-20210531174452649](http://myapp.img.mykernel.cn/image-20210531174452649.png)

## 自定义响应码

### status

如上在HttpResponse对象中传递status=400

### 使用不同的响应类

```diff
def demo_resp(reqeust):
    s = '{"name": "python"}'
+    resp = HttpResponseForbidden(s,content_type="application/json")
    resp['itcast'] = 'python'
    return resp
```

支持的响应类, 进入HttpResponseForbidden 类文件，生成图表

![image-20210531174901929](http://myapp.img.mykernel.cn/image-20210531174901929.png)

![responseclass](http://myapp.img.mykernel.cn/responseclass.png)

## json响应类

```diff
from django.http import HttpResponse, HttpRequest, HttpResponseForbidden, JsonResponse

def demo_resp(reqeust):
+    s = {"name": "python"}
+    resp = JsonResponse(s)
    resp['itcast'] = 'python'
    return resp
```

源码

```python
class JsonResponse(HttpResponse):
    """
    An HTTP response class that consumes data to be serialized to JSON.

    :param data: Data to be dumped into json. By default only ``dict`` objects
      are allowed to be passed due to a security flaw before EcmaScript 5. See
      the ``safe`` parameter for more information.
    :param encoder: Should be a json encoder class. Defaults to
      ``django.core.serializers.json.DjangoJSONEncoder``.
    :param safe: Controls if only ``dict`` objects may be serialized. Defaults
      to ``True``.
    :param json_dumps_params: A dictionary of kwargs passed to json.dumps().
    """

    def __init__(self, data, encoder=DjangoJSONEncoder, safe=True,
                 json_dumps_params=None, **kwargs):
        if safe and not isinstance(data, dict):
            raise TypeError(
                'In order to allow non-dict objects to be serialized set the '
                'safe parameter to False.'
            )
        if json_dumps_params is None:
            json_dumps_params = {}
        kwargs.setdefault('content_type', 'application/json')
        data = json.dumps(data, cls=encoder, **json_dumps_params)
        super().__init__(content=data, **kwargs)
```

> 只需要传递字典，传递而来的字典，会被json.dumps(dict), 并也把响应类型 'application/json' 也加入json中，最终传递给httpresponse.

## 重定向类

![image-20210531175829241](http://myapp.img.mykernel.cn/image-20210531175829241.png)

## session和cookie

### cookie

	浏览器中存储的数据
	
	设置cookie
		HttpResponse.set_cookie('cookie名', '值') # 默认max_age空，临时cookie
		HttpResponse.set_cookie('cookie名', '值', max_age=3600)


	读取cookie
		request.COOKIES

### session
​	原生支持session

	session，会话，多次频繁沟通
	
	广义：session机制 -> http身份认证，登陆状态
	狭义：session数据 -> 记录的状态数据，登陆之后记录的user_id.
	
	默认支持session.
		MIDDLEWARE = [
		    'django.middleware.security.SecurityMiddleware',
		    'django.contrib.sessions.middleware.SessionMiddleware', # 支持session.
		    'django.middleware.common.CommonMiddleware',
		    # 'django.middleware.csrf.CsrfViewMiddleware',
		    'django.contrib.auth.middleware.AuthenticationMiddleware',
		    'django.contrib.messages.middleware.MessageMiddleware',
		    'django.middleware.clickjacking.XFrameOptionsMiddleware',
		]

session存放位置, setting.py配置

```python
	INSTALLED_APPS = [
	    'django.contrib.sessions',
        ...
    ]
    
# SESSION默认存放位置，db
SESSION_ENGINE = 'django.contrib.sessions.backends.db'
        
# 缓存:1)重启：之前的session状态丢失。2) 多机调度session不一致。
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

# 混合存储
SESSION_ENGINE = 'django.contrib.sessions.backends.cached_db'

# redis
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://192.168.1.237:32003/7",
        "OPTIONS": {
            "PASSWORD": "123python",
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
        }
    }
}
SESSION_CACHE_ALIAS = "default" # 使用哪个名字的扩展
```

#### django 支持redis

安装django-redis

```bash
(slc365) [root@2e2144919378 django]# pip install django-redis
Requirement already satisfied: django-redis in /root/.pyenv/versions/3.6.5/envs/slc365/lib/python3.6/site-packages
Requirement already satisfied: Django>=2.2 in /root/.pyenv/versions/3.6.5/envs/slc365/lib/python3.6/site-packages (from django-redis)
Requirement already satisfied: redis>=3.0.0 in /root/.pyenv/versions/3.6.5/envs/slc365/lib/python3.6/site-packages (from django-redis)
Requirement already satisfied: asgiref<4,>=3.3.2 in /root/.pyenv/versions/3.6.5/envs/slc365/lib/python3.6/site-packages (from Django>=2.2->django-redis)
Requirement already satisfied: sqlparse>=0.2.2 in /root/.pyenv/versions/3.6.5/envs/slc365/lib/python3.6/site-packages (from Django>=2.2->django-redis)
Requirement already satisfied: pytz in /root/.pyenv/versions/3.6.5/envs/slc365/lib/python3.6/site-packages (from Django>=2.2->django-redis)
Requirement already satisfied: typing-extensions; python_version < "3.8" in /root/.pyenv/versions/3.6.5/envs/slc365/lib/python3.6/site-packages (from asgiref<4,>=3.3.2->Django>=2.2->django-redis)
You are using pip version 9.0.3, however version 21.1.2 is available.
You should consider upgrading via the 'pip install --upgrade pip' command.

```

setting.py配置django

```python
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://192.168.1.237:32003/7",
        "OPTIONS": {
            "PASSWORD": "123python",
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
        }
    }
}
SESSION_CACHE_ALIAS = "default" # 使用哪个名字的扩展
```

操作session

```bash
quest.session 当作字典操作session
			get
			set
			clear() 删除session的值
			flush() session记录
			del request.session['key']
			request.session.set_expiry(value)
				value 整数，指定s后过期
				0，浏览器关闭cookie清理后，session也会被清理
				value为None, 默认2周。 settings.py SESSION_COOKIE_AGE来设置全局默认值
```

连接redis

```bash
root@51a60b42fd47:/data/songliangcheng# redis-cli -p 32003 -a 123python
127.0.0.1:32003> SELECT 7
OK
127.0.0.1:32003[7]> KEYS *
(empty list or set)
```

添加session

![image-20210531181325694](http://myapp.img.mykernel.cn/image-20210531181325694.png)

现在有session出现

```bash
127.0.0.1:32003[7]> KEYS *
1) ":1:django.contrib.sessions.cachedgzpmk0eon3ryisyomv14tju8s9y9n9l"
```

输出信息

```bash
has no user_id
[31/May/2021 18:11:25] "GET /resp/ HTTP/1.1" 302 0
/say
[31/May/2021 18:11:25] "GET /say HTTP/1.1" 200 15
```

重复请求, 输出信息

```bash
[31/May/2021 18:12:21] "GET /resp HTTP/1.1" 301 0
has session 100 tom
[31/May/2021 18:12:21] "GET /resp/ HTTP/1.1" 302 0
/say
[31/May/2021 18:12:21] "GET /say HTTP/1.1" 200 15
```

### cookie和session区别

				              	位置
	session存放在后端， 			mysql、redis
	cookie存放在前端，  			浏览器存储

## 类视图和中间层

### 类视图

#### 函数视图面临的问题

函数视图，理解容易。如果一个视图区别不同的请求方式，可以使用if request.method == "GET", 代码不够复用。

views中

```python
def demo_resp(request):
    if request.method == "GET":
        if request.session.get('user_id'):
            uid = request.session['user_id']
            user_name = request.session['user_name']
            print('has session', uid, user_name)
        else:
            print('has no user_id')
            request.session['user_id'] = 100
            request.session['user_name'] = 'tom'
    else:
        pass
    return redirect(reverse('say'))
```

#### 类视图示例

准备一个新模块

```bash
(slc365) [root@2e2144919378 django]# cd demo/
(slc365) [root@2e2144919378 demo]# python manage.py startapp classview
```

加载模块、配置主路由、模块路由、定义视图

![image-20210531183036110](http://myapp.img.mykernel.cn/image-20210531183036110.png)

GET请求

![image-20210531182749239](http://myapp.img.mykernel.cn/image-20210531182749239.png)

POST请求

![image-20210531182810890](http://myapp.img.mykernel.cn/image-20210531182810890.png)

#### 类视图原理

通过此处的位置，进入类视图的源码

![image-20210601110030677](http://myapp.img.mykernel.cn/image-20210601110030677.png)

![image-20210601110041795](http://myapp.img.mykernel.cn/image-20210601110041795.png)

然后在base.py小窗，右键`move tab to new window`

![image-20210601110315033](http://myapp.img.mykernel.cn/image-20210601110315033.png)

dispatch方法，会将请求的方法转换成小写，在self实例中查询此方法

![image-20210601110406777](http://myapp.img.mykernel.cn/image-20210601110406777.png)

所以，你使用get方法，就会将请求传递给demoview类的get方法。请求post方法，就会将请求传递给demoview类的post方法。

#### 装饰器

实现在请求处理前完成一系列操作，可以对路径装饰（即这个路径无论什么方法，均装饰），可以对方法装饰（这个路径的GET方法添加装饰，POST方法不装饰）

##### 装饰器示例

实现一个装饰器，将函数传递过去，返回一个新的函数。此函数内部会代替你调用之前的函数

![image-20210601111514595](http://myapp.img.mykernel.cn/image-20210601111514595.png)

使用语法糖，就是将 `a = decorate(a)` 可以简写为放在函数上的`@decorate`

```python
def decorate(fn):
    def wrapper(*args, **kwargs):
        print('开始执行')
        ret = fn(*args, **kwargs)
        print('结束执行')
        return ret
    return wrapper

@decorate
def a(x,y):
    print(x+y)

# a = decorate(a)
a(3,2)
```

> 装饰什么对象？用户调用时，传递什么参数？

##### 装饰方法

```python
from django.http import HttpResponse
from django.views import View

def decorate(fn):
    def wrapper(*args, **kwargs):
        print('start decorate')
        ret = fn(*args, **kwargs)
        print('stop decorate')
        return ret
    return wrapper

class DemoView(View):
    @decorate
    def get(self,request):
        print('get')
        return HttpResponse('{} demoview get'.format(request.method))
    # get = decorate(get)

    @decorate
    def post(self,request):
        print('post')
        return HttpResponse('{} demoview post'.format(request.method))
```

postman发送get/post请求, 得到以下日志

```bash
start decorate
get
stop decorate
[01/Jun/2021 11:22:04] "GET /classview/demoview/ HTTP/1.1" 200 16

start decorate
post
stop decorate
[01/Jun/2021 11:21:57] "POST /classview/demoview/ HTTP/1.1" 200 18
```

##### 入口操作

###### 装饰入口

这个是入口函数，只需要装饰这个函数，即可。如果直接放在urls.py ，影响阅读，因为逻辑全部在views中。

![image-20210601112403314](http://myapp.img.mykernel.cn/image-20210601112403314.png)



在view中继承as_view，装饰此继承的view

![image-20210601113102509](http://myapp.img.mykernel.cn/image-20210601113102509.png)

###### 不装饰时，入口添加操作

直接覆盖父类方法

![image-20210601113519943](http://myapp.img.mykernel.cn/image-20210601113519943.png)

###### mix添加新特性

mix混合时，每一次的加入特性均会执行一次

```python
def decorate1(fn):
    def wrapper(*args, **kwargs):
        print('start decorate1')
        ret = fn(*args, **kwargs)
        print('stop decorate1')
        return ret
    return wrapper

def decorate2(fn):
    def wrapper(*args, **kwargs):
        print('start decorate2')
        ret = fn(*args, **kwargs)
        print('stop decorate2')
        return ret
    return wrapper

```

![image-20210601114003770](http://myapp.img.mykernel.cn/image-20210601114003770.png)

现在请求get, post

```bash
start decorate1
start decorate2
dispatcher
post
stop decorate2
stop decorate1
[01/Jun/2021 11:38:31] "POST /classview/demoview/ HTTP/1.1" 200 18
start decorate1
start decorate2
dispatcher
post
stop decorate2
stop decorate1
[01/Jun/2021 11:38:34] "POST /classview/demoview/ HTTP/1.1" 200 18
```

###### mix加载类前执行

```diff
class DecorateView1(View):
    @classmethod
    def as_view(cls, **initkwargs):
+        print('mix1')
        return decorate1(super().as_view(**initkwargs))


class DecorateView2(View):
    @classmethod
    def as_view(cls, **initkwargs):
+        print('mix1')
        return decorate2(super().as_view(**initkwargs))

class DemoView(DecorateView1,DecorateView2):
```

未请求前，配置重载 输出

```bash
mix1
mix1
System check identified no issues (0 silenced).
June 01, 2021 - 11:43:58
Django version 3.2.3, using settings 'demo.settings'
Starting development server at http://0.0.0.0:8000/
Quit the server with CONTROL-C.
```



### 中间件

类似于装饰器对入口的装饰, 以上是对单个视图装饰，中间件是对所有视图生效

![image-20210601115111912](http://myapp.img.mykernel.cn/image-20210601115111912.png)

请求GET, POST输出

![image-20210601114931611](http://myapp.img.mykernel.cn/image-20210601114931611.png)

## 框架提供的其他功能组件的使用

### 数据库

#### 安装

```bash
pip install pymysql
#https://www.jianshu.com/p/1d6824a77f1a
yum install python-devel mysql-devel
pip install mysqlclient
```

#### 使用

![image-20210601125535562](http://myapp.img.mykernel.cn/image-20210601125535562.png)

```python
from django.db import models

# Create your models here.

class BookInfo(models.Model):
    btitle = models.CharField(max_length=20, verbose_name='名称')
    bpub_date = models.DateField(verbose_name='发布日期')
    bread = models.IntegerField(default=0, verbose_name='阅读量')
    bcomment = models.IntegerField(default=0, verbose_name='评论量')
    is_delete = models.BooleanField(default=False, verbose_name='逻辑删除')

    # 默认表名： 小写应用名_小写类名
    # 通常模型类名和真实表名区分开，不希望数据库依赖程序里面的名字
    class Meta:
        db_table = "tb_bookinfo"  # 表名
        verbose_name = "图书"  # 表的人性化的名字
        verbose_name_plural = verbose_name  # django的复数形式的人性化名字


class HeroInfo(models.Model):
    GENDER_CHOICES = (
        (0, 'male'),
        (1, 'female')
    )
    hname = models.CharField(max_length=20, verbose_name='名称')
    hgender = models.SmallIntegerField(choices=GENDER_CHOICES, default=0, verbose_name='性别')
    hcomment = models.CharField(max_length=200, null=True, verbose_name='描述信息')
    # hbook 外键，自动关联bookinfo表的主键
    hbook = models.ForeignKey(BookInfo, on_delete=models.CASCADE, verbose_name='图书')
    is_delete = models.BooleanField(default=False, verbose_name='逻辑删除')

    class Meta:
        db_table = "tb_heroinfo"  # 表名
        verbose_name = "英雄"  # 表的人性化的名字
        verbose_name_plural = verbose_name  # django的复数形式的人性化名字
```

> flask需要自定义主键，外键。每个字段使用column定义。
>
> django，不定义主键时，默认添加id自增主键。如果django在某个字段添加primary key, 将不会自动添加主键。
>
> ```python
> choices, ((0,'male'),(1,'female')), 数据库是0和1，而这里的英文是标识，方便看
> hbook = ForeignKey(BookInfo,...) 不用考虑哪个表，自动会寻找另一个类（表）的主键关键，并成为主键. hbook 返回BookInfo对象本身
> 				级联删除。
> 					书名 -> 人1，人2.
> 
> 					on_delete如果删除书，是否自动删除人？
> 						PROTECT 默认：不允许删除书名, 只有先清理人，后才可以删除书名。 django 1.9之前不需要申明。
> 						CASCADE 级联，删除书名，自动另一个表的删除人。        django 1.9之后，需要手工申明on_delete CASCADE, PROTECT, SET_NULL, SET_DEFAULT, SET()
> 							SET_NULL 删除书名，将人字段置空，外键字段需要允许空
> 							SET_DEFAUTL, 引用空为默认值， 外键字段需要默认值
> 							SET()，指明一个函数，函数生成值。	
> ```

#### 生成表

制作migrations

```bash
(slc365) [root@2e2144919378 demo]# python manage.py makemigrations classview --empty
mix1
mix2
Migrations for 'classview':
  classview/migrations/0001_initial.py


(slc365) [root@2e2144919378 demo]# python manage.py makemigrations
mix1
mix1
Migrations for 'classview':
  classview/migrations/0001_initial.py
    - Create model BookInfo
    - Create model HeroInfo
```

查看指定模块的migration

```bash
(slc365) [root@2e2144919378 demo]# python manage.py showmigrations classview
mix1
mix1
classview
 [ ] 0001_initial
```

迁移

```bash
(slc365) [root@2e2144919378 demo]# python manage.py migrate
mix1
mix1
Operations to perform:
  Apply all migrations: admin, auth, classview, contenttypes, sessions
Running migrations:
  Applying classview.0001_initial... OK
```

![image-20210601130541986](http://myapp.img.mykernel.cn/image-20210601130541986.png)

> hero中记录着对应的book的id

#### CURD增删查改操作

##### 准备数据

```mysql
INSERT INTO tb_bookinfo ( btitle, bpub_date, bread, bcomment, is_delete )
VALUES
	( '射雕英雄传', '1980-5-1', 12, 34, 0 ),
	( '天龙八部', '1986-7-24', 36, 40, 0 ),
	( '笑傲江湖', '1995-12-24', 20, 80, 0 ),
	( '雪山飞狐', '1987-11-11', 58, 24, 0 );
INSERT INTO tb_heroinfo ( hname, hgender, hbook_id, hcomment, is_delete )
VALUES
	( '郭靖', 1, 1, '降龙十八掌', 0 ),
	( '黄蓉', 0, 1, '打狗棍法', 0 ),
	( '黄药师', 1, 1, '弹指神通', 0 ),
	( '欧阳锋', 1, 1, '蛤蟆功', 0 ),
	( '梅超风', 0, 1, '九阴白骨爪', 0 ),
	( '乔峰', 1, 2, '降龙十八掌', 0 ),
	( '段誉', 1, 2, '六脉神剑', 0 ),
	( '虚竹', 1, 2, '天山六阳掌', 0 ),
	( '王语嫣', 0, 2, '神仙姐姐', 0 ),
	( '令狐冲', 1, 3, '独孤九剑', 0 ),
	( '任盈盈', 0, 3, '弹琴', 0 ),
	( '岳不群', 1, 3, '华山剑法', 0 ),
	( '东方不败', 0, 3, '葵花宝典', 0 ),
	( '胡斐', 1, 4, '胡家刀法', 0 ),
	( '苗若兰', 0, 4, '黄衣', 0 ),
	( '程灵素', 0, 4, '医术', 0 ),
	( '袁紫衣', 0, 4, '六合拳', 0 );
```

![image-20210601170152940](http://myapp.img.mykernel.cn/image-20210601170152940.png)

> 通过navicat可以看到关系

##### shell和mysql日志

执行 python manage.py shell时，会把django进行初始化配置，会有数据库配置，可以直接进行数据库的操作。

```bash
(slc365) [root@2e2144919378 demo]# python manage.py shell
```

由于以上方式启动的mysql, 默认会打开genral.log日志，现在直接将日志打开

```bash
root@6eb0bfd67e24:/# tail -f /data/mysql/data/genral.log
```

##### 添加数据

###### save

- 添加不带主键的字段

    ```bash
    from classview.models import BookInfo, HeroInfo
    from datetime import date
    book = BookInfo(btitle="西游记",bpub_date=date(1988,7,1))
    book.save()
    ```

    ![image-20210601172259190](http://myapp.img.mykernel.cn/image-20210601172259190.png)

- 添加带主键的字段
    ```bash
    >>> from classview.models import BookInfo, HeroInfo
    >>> book = BookInfo.objects.get(btitle="西游记")
    >>> book
    <BookInfo: BookInfo object (5)>
    >>> hero = HeroInfo(hname="孙悟空", hgender=0, hbook=book)
    >>> hero.save()
    ```

    ```bash
    210602  0:48:38     5 Query     INSERT INTO `tb_heroinfo` (`hname`, `hgender`, `hcomment`, `hbook_id`, `is_delete`) VALUES ('孙悟空', 0, NULL, 5, 0)
    ```

###### objects

- 使用表字段添加

  ```python
  >>> from classview.models import BookInfo, HeroInfo
  >>> book = BookInfo.objects.get(btitle="西游记")
  >>> book
  <BookInfo: BookInfo object (5)>
  >>> HeroInfo.objects.create(hname="猪八戒", hgender=0,hbook_id=book.id)
  <HeroInfo: HeroInfo object (18)>
  ```

  ```bash
  210602  0:45:45     5 Query     INSERT INTO `tb_heroinfo` (`hname`, `hgender`, `hcomment`, `hbook_id`, `is_delete`) VALUES ('猪八戒', 0, NULL, 5, 0)
  ```

- 使用类模型添加

  ```bash
  >>> from classview.models import BookInfo, HeroInfo
  >>> book = BookInfo.objects.get(btitle="西游记")
  >>> book
  <BookInfo: BookInfo object (5)>
  >>> HeroInfo.objects.create(hname='沙僧',hbook=book,hgender=0)
  <HeroInfo: HeroInfo object (20)>
  ```

  ```bash
  210602  0:51:08     5 Query     INSERT INTO `tb_heroinfo` (`hname`, `hgender`, `hcomment`, `hbook_id`, `is_delete`) VALUES ('沙僧', 0, NULL, 5, 0)
  ```

##### 查询

###### 查询概述

语法 `ClassName.objects.<方法>`

```bash
基本查询
    get 单个        返回单个对象，不存在导演
    all 多个        返回对象列表
    count 统计      

过滤查询
    filter 过滤多个结果           		返回列表
    	单个条件 
            ==      -> exact 相等
            Like    -> contains 相当于 %xx%, startswith 相当于 xx%, endswith 相当于 %xx
            IS NULL -> isnull
            IN      -> in
            >       -> gt
            >=      -> gte
            <       -> lt
            <=      -> lte
		多个条件
			and     -> Q(id=5) & Q(btitle__contains='you')
			or      -> Q(id=5) | Q(btitle__contains='you')
			not     -> ~Q(id=5)
		
    get 过滤单个结果              		返回单个记录
    exclude 排除掉符合条件，剩下的结果        
	排序 
		ORDER BY id       => order_by('id')
		ORDER BY id  DESC => order_by('-id')
		
聚合函数
	sum             -> Sum()
	avg, max, min
	
关联查询
	1对多，一本书多个英雄。
        书左联英雄，书中找人物为 沙僧 对应的书

        对应的Python

	多对1，多个英雄一本书。
		英雄左联书，得到英雄
		
		对应的python
		
	inner join 只过滤on 条件满足
	left join  左表全部有，右表不足的显示空	
```

###### 基本查询

- all -> QuerySet, 相当于列表

  ```python
  >>> from classview.models import BookInfo, HeroInfo
  # 获取hero所有信息的列表
  >>> HeroInfo.objects.all()
  <QuerySet [<HeroInfo: HeroInfo object (1)>, <HeroInfo: HeroInfo object (2)>, <HeroInfo: HeroInfo object (3)>, <HeroInfo: HeroInfo object (4)>, <HeroInfo: HeroInfo object (5)>, <HeroInfo: HeroInfo object (6)>, <HeroInfo: HeroInfo object (7)>, <HeroInfo: HeroInfo object (8)>, <HeroInfo: HeroInfo object (9)>, <HeroInfo: HeroInfo object (10)>, <HeroInfo: HeroInfo object (11)>, <HeroInfo: HeroInfo object (12)>, <HeroInfo: HeroInfo object (13)>, <HeroInfo: HeroInfo object (14)>, <HeroInfo: HeroInfo object (15)>, <HeroInfo: HeroInfo object (16)>, <HeroInfo: HeroInfo object (17)>, <HeroInfo: HeroInfo object (18)>, <HeroInfo: HeroInfo object (19)>, <HeroInfo: HeroInfo object (20)>]>
  >>> 
  # 取出列表首个元素，首行
  >>> book = BookInfo.objects.all()[0]     
  
  # 首行的title
  >>> book.btitle   # 标题
  '射雕英雄传'        
  # 首行的id
  >>> book.id        # id
  1
  # 首行的主键
  >>> book.pk        # primary key
  1
  ```

- get

  ```bash
  >>> from classview.models import BookInfo, HeroInfo
  # 获取id为1的book信息
  >>> BookInfo.objects.get(id=1)
  <BookInfo: BookInfo object (1)>
  >>> 
  # 获取title为西游记的book信息
  >>> BookInfo.objects.get(btitle="西游记")
  <BookInfo: BookInfo object (5)>
  
  # 获取id为100的book信息, 如果不存在返回  BookInfo.DoesNotExist 异常
  >>> BookInfo.objects.get(id=100)
  Traceback (most recent call last):
    File "<console>", line 1, in <module>
    File "/root/.pyenv/versions/slc365/lib/python3.6/site-packages/django/db/models/manager.py", line 85, in manager_method
      return getattr(self.get_queryset(), name)(*args, **kwargs)
    File "/root/.pyenv/versions/slc365/lib/python3.6/site-packages/django/db/models/query.py", line 437, in get
      self.model._meta.object_name
  classview.models.BookInfo.DoesNotExist: BookInfo matching query does not exist.
  
  # 捕获异常
  >>> try:
  ...   BookInfo.objects.get(id=100)
  ... except BookInfo.DoesNotExist as e:
  ...   print(e)
  ... 
  BookInfo matching query does not exist.
  ```

- count

  ```python
  >>> from classview.models import BookInfo, HeroInfo
  
  # 20个英雄
  >>> HeroInfo.objects.count()
  20
  
  # 5本书
  >>> BookInfo.objects.count()
  5
  ```

  ![image-20210602090847613](http://myapp.img.mykernel.cn/image-20210602090847613.png)

###### 过滤查询

- filter

  > ==, exact

  ```python
  >>> from classview.models import BookInfo, HeroInfo
  >>> BookInfo.objects.filter(id=1)
  <QuerySet [<BookInfo: BookInfo object (1)>]>
  >>>  
  >>> BookInfo.objects.filter(id__exact=1)
  <QuerySet [<BookInfo: BookInfo object (1)>]>
  ```

  > LIKE

  ```python
  # 包含
  >>> from classview.models import BookInfo, HeroInfo
  >>> BookInfo.objects.filter(btitle__contains='游')
  # 起始
  >>> from classview.models import BookInfo, HeroInfo
  >>> BookInfo.objects.filter(btitle__startswith="西")
  <QuerySet [<BookInfo: BookInfo object (5)>]>
  # 结束
  >>> from classview.models import BookInfo, HeroInfo
  >>> BookInfo.objects.filter(btitle__endswith="记")
  <QuerySet [<BookInfo: BookInfo object (5)>]>
  ```

  > IS NULL, isnull=True

  ```python
  >>> BookInfo.objects.filter(btitle__isnull=True)                      
  <QuerySet []>
  >>> BookInfo.objects.filter(btitle__isnull=False)
  <QuerySet [<BookInfo: BookInfo object (1)>, <BookInfo: BookInfo object (2)>, <BookInfo: BookInfo object (3)>, <BookInfo: BookInfo object (4)>, <BookInfo: BookInfo object (5)>]>
  ```

  > IN

  ```python
  >>> BookInfo.objects.filter(id__in=[1,2,3])  
  <QuerySet [<BookInfo: BookInfo object (1)>, <BookInfo: BookInfo object (2)>, <BookInfo: BookInfo object (3)>]>
  ```

  > `>`,gt;     `>=`, gte;    `<`, lt;  `<=`, lte;

  ```python
  # id大于3
  >>> BookInfo.objects.filter(id__gt=3)
  <QuerySet [<BookInfo: BookInfo object (4)>, <BookInfo: BookInfo object (5)>]>
  
  # id小于等于2
  >>> BookInfo.objects.filter(id__lte=2)
  <QuerySet [<BookInfo: BookInfo object (1)>, <BookInfo: BookInfo object (2)>]>
  ```

  > 组合, and, or, not

  ```python
  # and
  >>> from django.db.models import Q        
  >>> BookInfo.objects.filter(id=5,btitle__contains="游")
  <QuerySet [<BookInfo: BookInfo object (5)>]>
  >>> BookInfo.objects.filter(Q(id=5) & Q(btitle__contains="游"))
  <QuerySet [<BookInfo: BookInfo object (5)>]>
  
  
  # or
  >>> BookInfo.objects.filter(Q(id=5) | Q(btitle__contains='you'))                
  <QuerySet [<BookInfo: BookInfo object (5)>]>
  
  # not
  >>> BookInfo.objects.filter(~Q(id=5))   
  <QuerySet [<BookInfo: BookInfo object (1)>, <BookInfo: BookInfo object (2)>, <BookInfo: BookInfo object (3)>, <BookInfo: BookInfo object (4)>]>
  
  ```

  > 排除

  ```python
  >>> BookInfo.objects.filter(~Q(id=5)).exclude(id=1)
  <QuerySet [<BookInfo: BookInfo object (2)>, <BookInfo: BookInfo object (3)>, <BookInfo: BookInfo object (4)>]>
  >>> 
  ```

  ```bash
  210602  1:52:16    15 Query     SELECT `tb_bookinfo`.`id`, `tb_bookinfo`.`btitle`, `tb_bookinfo`.`bpub_date`, `tb_bookinfo`.`bread`, `tb_bookinfo`.`bcomment`, `tb_bookinfo`.`is_delete` FROM `tb_bookinfo` WHERE (NOT (`tb_bookinfo`.`id` = 5) AND NOT (`tb_bookinfo`.`id` = 1)) LIMIT 21
  ```

  > 排序

  ```python
  # 正序
  >>> BookInfo.objects.filter(~Q(id=5)).exclude(id=1).order_by('id')
  <QuerySet [<BookInfo: BookInfo object (2)>, <BookInfo: BookInfo object (3)>, <BookInfo: BookInfo object (4)>]>
  # 逆序
  >>> BookInfo.objects.filter(~Q(id=5)).exclude(id=1).order_by('-id')
  <QuerySet [<BookInfo: BookInfo object (4)>, <BookInfo: BookInfo object (3)>, <BookInfo: BookInfo object (2)>]>
  ```

  ```bash
  # 正序
  210602  1:55:05    15 Query     SELECT `tb_bookinfo`.`id`, `tb_bookinfo`.`btitle`, `tb_bookinfo`.`bpub_date`, `tb_bookinfo`.`bread`, `tb_bookinfo`.`bcomment`, `tb_bookinfo`.`is_delete` FROM `tb_bookinfo` WHERE (NOT (`tb_bookinfo`.`id` = 5) AND NOT (`tb_bookinfo`.`id` = 1)) ORDER BY `tb_bookinfo`.`id` ASC LIMIT 21
  # 逆序
  210602  1:55:14    15 Query     SELECT `tb_bookinfo`.`id`, `tb_bookinfo`.`btitle`, `tb_bookinfo`.`bpub_date`, `tb_bookinfo`.`bread`, `tb_bookinfo`.`bcomment`, `tb_bookinfo`.`is_delete` FROM `tb_bookinfo` WHERE (NOT (`tb_bookinfo`.`id` = 5) AND NOT (`tb_bookinfo`.`id` = 1)) ORDER BY `tb_bookinfo`.`id` DESC LIMIT 21
  ```

  

  ###### 聚合查询

  > sum, max, min, avg

  ```python
  >>> from django.db.models import Sum, Avg, Min, Max
  >>> BookInfo.objects.aggregate(Sum('id'))
  {'id__sum': Decimal('15')}
  ```

  ```bash
  210602  1:53:59    15 Query     SELECT SUM(`tb_bookinfo`.`id`) AS `id__sum` FROM `tb_bookinfo`
  ```

##### 关联查询

###### django 内联

内置语法，另一个模型类小写__set, 就可以引用另一个表的外键相关的数据 

- 1对多, book对hero. 通过book对象查询hero与book相关的数据。

	  ```bash
    >>> from classview.models import BookInfo, HeroInfo
    >>> book = BookInfo.objects.get(id=1)        
    >>> book.heroinfo_set.all()
    <QuerySet [<HeroInfo: HeroInfo object (1)>, <HeroInfo: HeroInfo object (2)>, <HeroInfo: HeroInfo object (3)>, <HeroInfo: HeroInfo object (4)>, <HeroInfo: HeroInfo object (5)>]>
    
    ```

    ```bash
    210602  5:11:45    16 Query     SELECT `tb_heroinfo`.`id`, `tb_heroinfo`.`hname`, `tb_heroinfo`.`hgender`, `tb_heroinfo`.`hcomment`, `tb_heroinfo`.`hbook_id`, `tb_heroinfo`.`is_delete` FROM `tb_heroinfo` WHERE `tb_heroinfo`.`hbook_id` = 1 LIMIT 21
    ```
  
- 多对1, hero对book. 通过hero对象查询book与hero相关的数据。

  ```python
  >>> from classview.models import BookInfo, HeroInfo
  >>> hero = HeroInfo.objects.get(id=2)
  >>> hero.hbook
  <BookInfo: BookInfo object (1)>
  ```

  ```bash
  210602  5:18:23    17 Query     SELECT `tb_bookinfo`.`id`, `tb_bookinfo`.`btitle`, `tb_bookinfo`.`bpub_date`, `tb_bookinfo`.`bread`, `tb_bookinfo`.`bcomment`, `tb_bookinfo`.`is_delete` FROM `tb_bookinfo` WHERE `tb_bookinfo`.`id` = 1 LIMIT 21
  ```


###### 内联

只会过滤符合条件的字段

```bash
BookInfo.objects.filter(heroinfo__hname='沙僧')
# 查询结果显示信息 BookInfo 相关
# filter(heroinfo 后面的模型类小写，表示BookInfo与HeroInfo模型类进行“内联操作”
# heroinfo__hname 表示内联操作过过滤的是英雄的名字
```

- 获取指定英雄的书的信息

```bash
>>> BookInfo.objects.filter(heroinfo__hname='沙僧')
<QuerySet []>
>>> BookInfo.objects.filter(heroinfo__hname__contains="郭")
<QuerySet [<BookInfo: BookInfo object (1)>]>
```

```bash
 SELECT `tb_bookinfo`.`id`, `tb_bookinfo`.`btitle`, `tb_bookinfo`.`bpub_date`, `tb_bookinfo`.`bread`, `tb_bookinfo`.`bcomment`, `tb_bookinfo`.`is_delete` FROM `tb_bookinfo` INNER JOIN `tb_heroinfo` ON (`tb_bookinfo`.`id` = `tb_heroinfo`.`hbook_id`) WHERE `tb_heroinfo`.`hname` LIKE BINARY '%郭%'
```

- 获取指定书的英雄信息

  ```bash
  HeroInfo.objects.filter(hbook__btitle='天龙八部')
  ```

  ```bash
  FROM `tb_heroinfo` INNER JOIN `tb_bookinfo` ON (`tb_heroinfo`.`hbook_id` = `tb_bookinfo`.`id`) WHERE `tb_bookinfo`.`btitle` = '天龙八部'
  ```

  

##### 修改操作

###### 先查询再更新 save()

天龙八部修改为天龙九部

```bash
# 我们晓得天龙八部里面有乔峰，所以将乔峰对应的书修改。
>>> book = BookInfo.objects.filter(heroinfo__hname="乔峰")[0]
>>> book.btitle
'天龙八部'

>>> hero = HeroInfo.objects.get(hname="乔峰")
>>> book = hero.hbook
>>> book.btitle
'天龙八部'

# 修改
>>> book.btitle="天龙九部"
>>> book.save()

```

```bash
 UPDATE `tb_bookinfo` SET `btitle` = '天龙九部', `bpub_date` = '1986-07-24', `bread` = 36, `bcomment` = 40, `is_delete` = 0 WHERE `tb_bookinfo`.`id` = 2
```

```bash
# 验证
>>> BookInfo.objects.get(heroinfo__hname="乔峰").btitle
'天龙九部'
```



###### 查询时更新 filter().update()

- 关联查询更新
    ```bash
    # `tb_bookinfo` INNER JOIN `tb_heroinfo`之后，过滤乔峰相关的书，返回查询集
    # 更新书名
    BookInfo.objects.filter(heroinfo__hname="乔峰").update(btitle="天龙十部")
    ```

    ```bash
    SELECT `tb_bookinfo`.`id` FROM `tb_bookinfo` INNER JOIN `tb_heroinfo` ON (`tb_bookinfo`.`id` = `tb_heroinfo`.`hbook_id`) WHERE `tb_heroinfo`.`hname` = '乔峰'
                       18 Query     UPDATE `tb_bookinfo` SET `btitle` = '天龙十部' WHERE `tb_bookinfo`.`id` IN (2)
    ```

- 普通查询更新 

  ```python
  >>> BookInfo.objects.filter(btitle="天龙十部").update(btitle="天龙十一部")
  1
  
  ```
  
  ```bash
  UPDATE `tb_bookinfo` SET `btitle` = '天龙十一部' WHERE `tb_bookinfo`.`btitle` = '天龙十部'
  ```

##### 删除操作

```python
HeroInfo.objects.get(hname="乔峰").delete()


HeroInfo.objects.filter(hname="令狐冲").delete()
```

#### 模型类方法 `__str__`

```bash
# 默认对象显示为id号
>>> book = BookInfo.objects.get(btitle="天龙十一部")
>>> book.pk
2
>>> book.id
2
```

![image-20210602135823473](http://myapp.img.mykernel.cn/image-20210602135823473.png)

```bash
# 重新进入shell
>>> from classview.models import BookInfo, HeroInfo
>>> book = BookInfo.objects.get(btitle="天龙十一部")
>>> book
<BookInfo: 天龙十一部>

# 查询书对应的名字
>>> book = BookInfo.objects.get(btitle="天龙十一部")
>>> book.heroinfo_set.all()
<QuerySet [<HeroInfo: 段誉>, <HeroInfo: 虚竹>, <HeroInfo: 王语嫣>]>

>>> HeroInfo.objects.filter(hbook__btitle="天龙十一部")
<QuerySet [<HeroInfo: 段誉>, <HeroInfo: 虚竹>, <HeroInfo: 王语嫣>]>
```

#### 查询结果集

从数据库拿回来的结果集合，使用起来像列表。

特性：惰性执行: 真正需要数据时，才访问数据库； 缓存查询结果

all, filter, exlude, order_by

#### 管理器

##### 原理

objects是管理器，提供操作接口

django会默认生成管理器objects, 例如可以添加以下代码。如果不添加默认django会替你添加。一旦我们指定 objs = models.Manager() django不会自动生成。

![image-20210602140921934](http://myapp.img.mykernel.cn/image-20210602140921934.png)

##### 自定义管理器

在我们操作以上操作方法时，可能期望有一些**自定义的方法**。或者**原始方法新增操作**时，就需要自定义管理器。

示例：原始方法仅过滤未删除字段，is_delete=False

```python
# 默认情况下
>>> from classview.models import BookInfo, HeroInfo

# 所有均未删除
>>> book = BookInfo.objects.all()
>>> book
<QuerySet [<BookInfo: 射雕英雄传 False>, <BookInfo: 天龙十一部 False>, <BookInfo: 笑傲江湖 False>, <BookInfo: 雪山飞狐 False>, <BookInfo: 西游记 False>]>

# 将段誉对应的书修改成删除
>>> BookInfo.objects.filter(heroinfo__hname="段誉").update(btitle="天龙十一部",is_delete=True)
1
# 获取
>>> book
<QuerySet [<BookInfo: 射雕英雄传 False>, <BookInfo: 天龙十一部 True>, <BookInfo: 笑傲江湖 False>, <BookInfo: 雪山飞狐 False>, <BookInfo: 西游记 False>]>
```

当我们添加一个自定义管理类之后，原来的objects已经失效，并且并的管理器，会自动排除标记删除为True的字段。

> 注意：修改类之后，需要重启shell

![image-20210602142153165](http://myapp.img.mykernel.cn/image-20210602142153165.png)

### 模板

前后端分离中不会使用模板，所以大概了解即可

![image-20210602143724711](http://myapp.img.mykernel.cn/image-20210602143724711.png)

现在GET访问是模板

```bash
(slc365) [root@2e2144919378 demo]# curl http://127.0.0.1:8000/classview/demoview/
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <h1>北京</h1>
</body>
</html>
```



### admin站点

#### 管理员

```bash
(slc365) [root@2e2144919378 demo]# python manage.py createsuperuser 
mix1
mix2
用户名 (leave blank to use 'root'): admin
电子邮件地址: admin@mykernel.cn
Password: 
Password (again): 
密码跟 用户名 太相似了。
这个密码太常见了。
Bypass password validation and create user anyway? [y/N]: y
Superuser created successfully.
```

![image-20210602144037629](http://myapp.img.mykernel.cn/image-20210602144037629.png)

#### 站点定义和添加待管理的数据库

通过在admin.py引入模型类并注册即可在网页上管理数据模型，数据模型的app配置就是出现的表格第一个字段，模型类的配置就是每个表名，站点信息由admin.site统一管理

![image-20210602144753757](http://myapp.img.mykernel.cn/image-20210602144753757.png)

#### 自定义管理

当我们点进一个表，默认显示如下,  就是`__str__`方法的结果

![image-20210602144906663](http://myapp.img.mykernel.cn/image-20210602144906663.png)

现在首先需要展示所有字段，添加分页，过滤，搜索

编辑图书时，编辑对应的英雄

##### 准备配置类

注册方式一

```python
class BookInfoAdmin(admin.ModelAdmin):
    pass

admin.site.register(BookInfo, BookInfoAdmin)
```

注册方式二

```python
# admin.site.register(HeroInfo)
@admin.register(HeroInfo)
class HeroInfoAdmin(admin.ModelAdmin):
    pass
```

##### 显示页面控制

页面大小、动作位置、过滤、搜索、字段展示

![image-20210602152526264](http://myapp.img.mykernel.cn/image-20210602152526264.png)

##### 显示自定义结果

在model中定义一个方法，在管理类中显示这个方法即可

方法的short_description 属性控制网页显示的字段名

方法的admin_order_field 属性控制网页的排序

![image-20210602153232649](http://myapp.img.mykernel.cn/image-20210602153232649.png)

##### 显示关联对象的属性

![image-20210602153932431](http://myapp.img.mykernel.cn/image-20210602153932431.png)

##### 编辑图书时，编辑关联的英雄

2种TabularInline, StackedInline

fields展示书信息

fieldsets 分段展示书的信息

![image-20210602154332271](http://myapp.img.mykernel.cn/image-20210602154332271.png)

#

