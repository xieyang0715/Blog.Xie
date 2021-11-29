---
title: python虚拟环境
date: 2021-10-09 09:08:23
tags:
- python
---

# 前言

之前下载docker-compose, 可以有替代的安装方式 ，使用pip, 其中说明了python虚拟环境的制作方法

>  原文: https://docs.python-guide.org/dev/virtualenvs/

<!--more-->
# Pipenv & Virtual Environments

此向导合适python3, 也支持python 2.7

## Make sure you’ve got Python & pip[¶](https://docs.python-guide.org/dev/virtualenvs/#make-sure-you-ve-got-python-pip)

确保存在python， pip

```bash
$ python --version
Python 2.7.17

 pip --version
WARNING: pip is being invoked by an old script wrapper. This will fail in a future version of pip.
Please see https://github.com/pypa/pip/issues/5599 for advice on fixing the underlying issue.
To avoid this problem you can invoke Python with '-m pip' instead of running pip directly.
pip 20.3.4 from /root/.local/lib/python2.7/site-packages/pip (python 2.7)
```

使用源码安装的python, 会带有pip. 如果是linux 包管理器安装的python, 需要额外安装pip. 参考:  [install pip](https://pip.pypa.io/en/stable/installing/) 

## Installing Pipenv

pipenv是python的依赖管理器，pip可以安装python包

```bash
$ pip install --user pipenv
```

> 验证安装:
>
> ```bash
> $ pipenv -v
> pipenv: command not found
> ```
>
> > 如果不可以用，linux上，需要在~/.profile 文件中永久 配置`python -m site --user-base` 目录下的bin目录到PATH中
>
> 再次验证安装
>
> ```bash
> pipenv --version
> pipenv, version 2021.5.29
> ```

## Installing packages for your project

进入你的项目，只需要进入一个空目录

```bash
$ cd project_folder
$ pipenv install requests
```

> ```bash
>  pipenv install requests
> Creating a virtualenv for this project...
> Pipfile: /root/project_folder/Pipfile
> Using /usr/bin/python3.6m (3.6.9) to create virtualenv...
> ⠇ Creating virtual environment...created virtual environment CPython3.6.9.final.0-64 in 529ms
>   creator CPython3Posix(dest=/root/.local/share/virtualenvs/project_folder-FQc41dUk, clear=False, no_vcs_ignore=False, global=False)
>   seeder FromAppData(download=False, pip=bundle, wheel=bundle, setuptools=bundle, via=copy, app_data_dir=/root/.local/share/virtualenv)
>     added seed packages: pip==21.2.4, setuptools==58.1.0, wheel==0.37.0
>   activators NushellActivator,PythonActivator,FishActivator,CShellActivator,PowerShellActivator,BashActivator
> ✔ Successfully created virtual environment!  # 创建好了虚拟环境
> 
> # 安装包
> Virtualenv location: /root/.local/share/virtualenvs/project_folder-FQc41dUk
> Creating a Pipfile for this project...
> Installing requests...
> ```

现在pipenv将会安装极好的 [Requests](http://docs.python-requests.org/en/master/) 库和创建一个`Pipfile`文件于 `project_folder`此项目目录中。

`Pipfile`文件 会追踪项目的依赖，方便重装项目。例如：你共享项目给其他人使用时。

## Using installed packages

创建一个`main.py` 来使用这个Requests库

```python
import requests

response = requests.get('https://httpbin.org/ip')

print('Your IP is {0}'.format(response.json()['origin']))

```

通过`pipenv run`运行脚本

```bash
root@ip-10-0-0-245:~/project_folder#  pipenv run python main.py
Your IP is 52.83.161.189
```

也可以通过`pipenv shell`产生一个新的shell命令行，让所有命令都能访问到新的环境.

```bash
root@ip-10-0-0-245:~/project_folder# pipenv shell
Launching subshell in virtual environment...
root@ip-10-0-0-245:~/project_folder#  . /root/.local/share/virtualenvs/project_folder-FQc41dUk/bin/activate
(project_folder) root@ip-10-0-0-245:~/project_folder# python main.py 
Your IP is 52.83.161.189
```

# Lower level: virtualenv

[virtualenv](http://pypi.org/project/virtualenv)用来创建隔离的python环境，会创建一个目录，其中包含所有必要可执行文件，这些可执行文件用来使用python项目需要的包。

直接代替pipenv

```bash
$ pip install virtualenv
```

> 测试安装:
>
> ```bash
> root@ip-10-0-0-245:~/project_folder# virtualenv --version
> virtualenv 20.8.1 from /root/.local/lib/python2.7/site-packages/virtualenv/__init__.pyc
> ```

## 使用

### 创建虚拟环境

```bash
$ cd project_folder_virtualenv
$ virtualenv venv
created virtual environment CPython2.7.17.final.0-64 in 339ms
  creator CPython2Posix(dest=/root/project_folder_virtualenv/venv, clear=False, no_vcs_ignore=False, global=False)
  seeder FromAppData(download=False, pip=bundle, wheel=bundle, setuptools=bundle, via=copy, app_data_dir=/root/.local/share/virtualenv)
    added seed packages: pip==20.3.4, setuptools==44.1.1, wheel==0.37.0
  activators NushellActivator,PythonActivator,FishActivator,CShellActivator,PowerShellActivator,BashActivator
```

> 1. `virtualenv venv` 会在当前目录下创建一个venv目录，其中包含一些必要的二进制文件。并拷贝一些pip库用来安装其他包。python文件
> 2. 虚拟环境名可以是任何名，当前是venv
> 3. 省略名字时，将会以当前目录名替代
> 4. 通常使用venv, 因为很容易被.gitignore忽略

你也可以使用指定的python版本来创建

```bash
$ virtualenv -p /usr/bin/python2.7 venv
```

> 或者通过环境变量指定python版本
>
> ```bash
> $ export VIRTUALENVWRAPPER_PYTHON=/usr/bin/python2.7
> ```

### 使用虚拟环境

#### Linux

```bash
$ source venv/bin/activate
```

当前虚拟环境名，将会出现在命令提示符的左边。例如:`(venv) root@ip-10-0-0-245:~/project_folder_virtualenv# `

现在，所有安装的包将会存储在venv目录中，并且与全局的python安装隔离了。

#### Windows

假设你在项目目录下

```
C:\Users\SomeUser\project_folder> venv\Scripts\activate
```

### 释放虚拟环境

```bash
$ deactivate
```

这将使得当前目录回退到系统python环境，

删除虚拟环境，仅需要 `rm -rfvenv`

过一段时间后，你的系统会有大量的虚拟环境垃圾，但是你会忘记你的虚拟环境在哪儿了。

> Python has included venv module from version 3.3. For more details: [venv](https://docs.python.org/3/library/venv.html).

## 注意

如果`virtualenv --no-site-packages`   将会不包括全局包，在virtualenv 1.7+是默认行为

为了保持环境一致，可以`freeze`当前环境

```bash
$ pip freeze > requirements.txt
```

> 这将生成 requirements.txt 文件，它包含当前环境所有安装的包的列表及它们的版本。`pip list`也可以看到

之后的其他开发人员重建环境，确保一致性的环境。

```bash
$ pip install -r requirements.txt
```

最后记得将虚拟环境目录，从源码控制中排除，通过添加这个目录到ignore列表中实现。



# virtualenvwrapper

## 安装

[virtualenvwrapper](https://virtualenvwrapper.readthedocs.io/en/latest/index.html)提供一组命令使得在虚拟环境中工作非常愉快。这个也可以将所有虚拟环境放在一个地方。

确保virtualenv已经安装

```bash
pip install virtualenv
virtualenv --version
virtualenv 20.8.1 from /root/.local/lib/python2.7/site-packages/virtualenv/__init__.pyc
```

安装virtualenvwrapper

```bash
$ pip install virtualenvwrapper
$ export WORKON_HOME=~/Envs
$ source /usr/local/bin/virtualenvwrapper.sh
```

> ([Full virtualenvwrapper install instructions](https://virtualenvwrapper.readthedocs.io/en/latest/install.html).)
>
> For Windows, you can use the [virtualenvwrapper-win](https://github.com/davidmarble/virtualenvwrapper-win/).
>
> To install (make sure **virtualenv** is already installed):
>
> ```
> $ pip install virtualenvwrapper-win
> ```
>
> In Windows, the default path for WORKON_HOME is %USERPROFILE%\Envs

## 使用

1. 创建虚拟环境

   ```bash
   $ mkvirtualenv project_folder_virtualenvwrapper
   ```

   > This creates the `project_folder` folder inside `~/Envs`.

2. 进入虚拟环境

   ```bash
   workon project_folder_virtualenvwrapper
   ```

   或者，你可以直接创建一个项目，其中会自动创建虚拟环境, 并自动进入这个项目目录中

   ```bash
   export PROJECT_HOME=~/Envs
   mkproject project_folder_vew
   ```

   > **virtualenvwrapper** 支持tab补全环境名，它真正帮助你有大量环境管理或需要记得每一个环境名。

3. 释放环境

   `workon` 可以快速切换环境，同时也会释放之前的环境

   释放当前环境

   ```bash
   $ deactivate
   ```

   删除环境

   ```bash
   $ rmvirtualenv <环境名>
   ```

## 其它有用的命令

```
lsvirtualenv
```

List all of the environments.

```
cdvirtualenv
```

Navigate into the directory of the currently activated virtual environment, so you can browse its `site-packages`, for example.

```
cdsitepackages
```

Like the above, but directly into `site-packages` directory.

```
lssitepackages
```

Shows contents of `site-packages` directory.

[Full list of virtualenvwrapper commands](https://virtualenvwrapper.readthedocs.io/en/latest/command_ref.html).

基于模板创建项目

配置项目与环境关联

```bash
setvirtualenvproject django1 /opt/django1/
```

进入项目

```bash
cdprojectw
```

清理当前环境第3方包

```bash
wipeenv
```

## 常用

```bash
mkvirtualenv -a /opt/django3.6.9 -p python3.6.9 d369
```

> 1. `-a` 此项目与哪个目录关联
> 2. `-p` 此虚拟环境使用哪个版本
> 3. `d369`虚拟环境名

## 示例

准备写一个博客，采用python3.6.9版本的虚拟环境，将虚拟环境统一管理在/opt/Envs目录下，虚拟环境名为dj3

```bash
pip install  virtualenv virtualenvwrapper
```

```bash
# export WORKON_HOME=/opt/Envs
# source /usr/local/bin/virtualenvwrapper.sh
```

```bash
# mkvirtualenv -p python3.6.9 dj3
```

> 若不指定`-p` 将会使用系统默认的python环境，`/usr/bin/python` 来创建虚拟环境，一般是python2.7

```bash
# python -V
Python 3.6.9
```

关联项目目录，例如在/opt/blog

```bash
# install -dv /opt/blog
install: creating directory '/opt/blog'
# setvirtualenvproject dj3 /opt/blog
Setting project for dj3 to /opt/blog
```

现在只需要一进入这个环境，就自动进入关联的目录了

```bash
# deactivate 
# workon dj3
(dj3) root@ip-10-0-0-245:/opt/blog# 
```

查看所有环境

```bash
(dj3) root@ip-10-0-0-245:/opt/blog# lsvirtualenv 
dj3
===

```

删除环境

```bash
# rmvirtualenv dj3
```



# virtualenv-burrito

使用 [virtualenv-burrito](https://github.com/brainsik/virtualenv-burrito),   virtualenv + virtualenvwrapper  会包装成一个单一的命令

## direnv

cd进入一个拥有.env文件的目录，会自动activate。

Install it on Mac OS X using `brew`:

```
$ brew install direnv
```

On Linux follow the instructions at [direnv.net](https://direnv.net/)



