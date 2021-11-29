---
title: 搭建linux平台的python开发环境
date: 2020-12-31 08:04:51
tags:
- python
---





# 1. 操作系统准备

```bash
docker pull centos:7

docker run -p 22:22 -p 8888:8888 -it --restart=always --name python-learn  -d centos:7 /bin/bash
```

# 2. pyenv

python environment

当我们在生产服务器上需要部署新项目python 3.0，而已经存在老项目python 2.0, 这个时候直接部署上去会存在模块版本问题。

1. linux: pyenv, windows: pycharm
2. docker
3. 虚拟机

pyenv官网：https://github.com/pyenv/pyenv



pyenv是python多版本管理工具，会将不同的版本放在`~/.pyenv/versions`目录中。

pyenv的virtualenv插件是python虚拟环境，相当于基于以上版本的定制版本。会将不同的版本在`~/.pyenv/versions/3.5.3/envs` 目录中，也会在`~/.pyenv/versions`目录创建链接文件。



pyenv三种模式

1. global: python执行global, 当前用户的所有会话python环境修改。

2. shell：当前shell环境修改。

3. local：当前目录与环境绑定。

   

## 2.1 准备用户和ssh连接

```bash
docker exec -it python-learn bash
 
# 准备ssh
yum -y install openssh-server
 
ssh-keygen -t rsa -b 2048  -P '' -f /etc/ssh/ssh_host_rsa_key
ssh-keygen -t dsa  -P '' -f /etc/ssh/ssh_host_dsa_key
ssh-keygen -t ecdsa  -P '' -f /etc/ssh/ssh_host_ecdsa_key
ssh-keygen -t ecdsa  -P '' -f /etc/ssh/ssh_host_ed25519_key

/sbin/sshd


# 准备python普通用户
echo magedu | passwd --stdin root
useradd python
echo 'magedu' | passwd --stdin python

# 直接连接ssh
# 因为docker已经暴露端口在22
[c:\~]$ ssh localhost
[python@19d125e0548a ~]$ 

```

![image-20201231162421909](http://myapp.img.mykernel.cn/image-20201231162421909.png)

![image-20201231162429660](http://myapp.img.mykernel.cn/image-20201231162429660.png)



以相同方式，添加root的连接

## 2.2 安装Pyenv

root用户完成基础环境配置

```bash
# 安装git, pyenv会连接github
yum -y install git

# 安装基础包，pyenv会编译安装python
yum install -y vim wget tree  lrzsz gcc gcc-c++ automake pcre pcre-devel zlib zlib-devel openssl openssl-devel iproute net-tools iotop ntpdate lsof bzip2 sqlite-devel
```

python用户完成以下pyenv安装操作

https://github.com/pyenv/pyenv-installer

```bash
# 执行安装脚本
[python@19d125e0548a ~]$ curl -L https://github.com/pyenv/pyenv-installer/raw/master/bin/pyenv-installer | bash
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   148  100   148    0     0     56      0  0:00:02  0:00:02 --:--:--    56
100  2454  100  2454    0     0    590      0  0:00:04  0:00:04 --:--:--  4429
Cloning into '/home/python/.pyenv'...
remote: Enumerating objects: 717, done.
remote: Counting objects: 100% (717/717), done.
remote: Compressing objects: 100% (480/480), done.
remote: Total 717 (delta 373), reused 327 (delta 144), pack-reused 0
Receiving objects: 100% (717/717), 399.06 KiB | 295.00 KiB/s, done.
Resolving deltas: 100% (373/373), done.
Cloning into '/home/python/.pyenv/plugins/pyenv-doctor'...
...
remote: Total 10 (delta 1), reused 6 (delta 0), pack-reused 0
Unpacking objects: 100% (10/10), done.

WARNING: seems you still have not added 'pyenv' to the load path.

# Load pyenv automatically by adding
# the following to ~/.bashrc:

export PATH="/home/python/.pyenv/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"


# 添加环境变量
# ~/.bashrc
export PATH="/home/python/.pyenv/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"


# 读环境变量, 使pyenv命令生效
source ~/.bashrc

# 当前python版本
[python@19d125e0548a ~]$ python -V
Python 2.7.5
```



## 2.3 安装python

```bash
# pyenv帮助
[python@19d125e0548a ~]$ pyenv 
pyenv 1.2.21
Usage: pyenv <command> [<args>]

Some useful pyenv commands are:
   --version   Display the version of pyenv
   activate    Activate virtual environment
   commands    List all available pyenv commands
   deactivate   Deactivate virtual environment
   doctor      Verify pyenv installation and development tools to build pythons.
   exec        Run an executable with the selected Python version
   global      # 全局环境
   help        Display help for a command
   hooks       List hook scripts for a given pyenv command
   init        Configure the shell environment for pyenv
   install     # 安装python
   local       # 当前目录
   prefix      Display prefix for a Python version
   rehash      Rehash pyenv shims (run this after installing executables)
   root        Display the root directory where versions and shims are kept
   shell       # 当前shell
   shims       List existing pyenv shims
   uninstall   Uninstall a specific Python version
   version     # 当前python版本相当于 python -V
   version-file   Detect the file that sets the current pyenv version
   version-name   Show the current Python version
   version-origin   Explain how the current Python version is set
   versions    # pyenv管理的所有python版本
   virtualenv   Create a Python virtualenv using the pyenv-virtualenv plugin
   virtualenv-delete   Uninstall a specific Python virtualenv
   virtualenv-init   Configure the shell environment for pyenv-virtualenv
   virtualenv-prefix   Display real_prefix for a Python virtualenv version
   virtualenvs   List all Python virtualenvs found in `$PYENV_ROOT/versions/*'.
   whence      List all Python versions that contain the given executable
   which       Display the full path to an executable

See `pyenv help <command>' for information on a specific command. # 获取命令帮助的方式
For full documentation, see: https://github.com/pyenv/pyenv#readme


# pyenv install帮助
[python@19d125e0548a ~]$ pyenv help install
Usage: pyenv install [-f] [-kvp] <version>
       pyenv install [-f] [-kvp] <definition-file>
       pyenv install -l|--list
       pyenv install --version

  -l/--list          # 列出所有可以安装版本
  -f/--force         Install even if the version appears to be installed already
  -s/--skip-existing Skip if the version appears to be installed already

  python-build options:

  -k/--keep          Keep source tree in $PYENV_BUILD_ROOT after installation
                     (defaults to $PYENV_ROOT/sources)
  -p/--patch         Apply a patch from stdin before building
  -v/--verbose       Verbose mode: print compilation status to stdout
  --version          Show version of python-build
  -g/--debug         Build a debug version

For detailed information on installing Python versions with
python-build, including a list of environment variables for adjusting
compilation, see: https://github.com/pyenv/pyenv#readme

[python@19d125e0548a ~]$ 


# 列出所有可以安装的python版本
pyenv  install -l
```

  3.5.3
  3.5.4                  # 官方Cpython不带前缀
  ... 
  3.10-dev
  activepython*
  anaconda*
  graalpython*
  ironpython*            # python代码编译成.Net的字节码，在.Net虚拟机运行
  jython*                    # python代码编译成java的字节码，在JVM虚拟机运行
  micropython* 
  miniconda* 
  pypy*                        # **效率比cpython高1.5倍**，python语言写的python解释器，使用了just in time 实时编译技术。立马编译，运行期发现问题，动态优化。
  pyston*
  stackless*



ipython                     # 增强的Cpython, 通过pip安装



**安装**

```bash
[python@19d125e0548a ~]$ pyenv install 3.5.3
Downloading Python-3.5.3.tar.xz...
-> https://www.python.org/ftp/python/3.5.3/Python-3.5.3.tar.xz
Installing Python-3.5.3...
WARNING: The Python bz2 extension was not compiled. Missing the bzip2 lib?
WARNING: The Python readline extension was not compiled. Missing the GNU readline lib?
WARNING: The Python sqlite3 extension was not compiled. Missing the SQLite3 lib?
Installed Python-3.5.3 to /home/python/.pyenv/versions/3.5.3 # 安装的python位置

```

> 如果下载python慢, 将提前下载的包放在以下目录中
>
> ```bash
> install -dv ~/.pyenv/cache
> 
> ls ~/.pyenv/cache
> Python-3.5.3.tar.xz Python-3.5.3.tar.gz Python-3.5.3.tgz
> ```



## 2.4 pyenv使用

```bash
# 当前版本
[python@19d125e0548a ~]$ pyenv version
system (set by /home/python/.pyenv/version)
[python@19d125e0548a ~]$ python -V
Python 2.7.5

# 所有版本
[python@19d125e0548a ~]$ pyenv versions
* system (set by /home/python/.pyenv/version) # 带*为当前版本
  3.5.3
```



### 2.4.1 global

![image-20201231165618541](http://myapp.img.mykernel.cn/image-20201231165618541.png)

注意：新建会话的python环境也被修改



恢复环境

![image-20201231170104012](http://myapp.img.mykernel.cn/image-20201231170104012.png)



所以，**不会使用global**。同一个用户登陆，另一个用户执行global，会导致其他登陆用户环境被修改。

### 2.4.2 shell

![image-20201231170504392](http://myapp.img.mykernel.cn/image-20201231170504392.png)

如果我们配置了当前shell环境使用python指定版本，另起一个shell，版本又恢复了。所以也**不使用shell**

关闭左侧窗口，即可恢复环境

### 2.4.3 local

local绑定文件夹，这个最好

1. 不同目录不同环境
2. 子目录继承父目录环境

![image-20201231171221783](http://myapp.img.mykernel.cn/image-20201231171221783.png)

恢复cmdb目录的版本, 因为通常不会这样使用python环境

```bash
[python@19d125e0548a ~]$ cd magedu/projects/cmdb/
[python@19d125e0548a cmdb]$ pyenv versions
  system
* 3.5.3 (set by /home/python/magedu/projects/cmdb/.python-version)
[python@19d125e0548a cmdb]$ pyenv local system
[python@19d125e0548a cmdb]$ python -V
Python 2.7.5

```



## 2.5 python多环境部署

当只有当前用户使用当前主机时，可以仅使用多环境部署

但是**如果一个主机有多个项目使用相同版本python，所以最好使用虚拟环境**



以上通过`pyenv install 3.5.3`已经安装python 3.5的环境

```bash
[python@19d125e0548a projects]$ cd 
[python@19d125e0548a ~]$ pyenv versions # pyenv管理的版本们
* system (set by /home/python/.pyenv/version) # 管理版本的目录
  3.5.3        # 这个就是python安装3.5环境

```

以下在安装一个pypy环境，这个版本效率高，由python写的python解释器

```bash
# 列出版本
pyenv install -l

# 安装pypy
[python@19d125e0548a ~]$ pyenv install  pypy3.6-7.3.1 -v
/tmp/python-build.20201231093825.22500 ~
Installing pypy3.6-v7.3.1-linux64...
/tmp/python-build.20201231093825.22500/pypy3.6-v7.3.1-linux64 /tmp/python-build.20201231093825.22500 ~
Installed pypy3.6-v7.3.1-linux64 to /home/python/.pyenv/versions/pypy3.6-7.3.1 # 安装位置

```



## 2.6 python虚拟环境

以上python多环境在/home/python/.pyenv/version目录下管理



为何使用虚拟环境？

- 如果相同python用户不同人使用时，相同的python3.5.3版本就会环境冲突，同一个模块就会有不同的版本，所以需要python虚拟环境

- 如果相同用户同一个人使用相同版本python3.5.3, 但是你使用不同的项目，所以也建议使用虚拟环境。



pyenv 只是python多版本管理工具，其插件virtualenv才是某个版本多个虚拟环境管理工具



### 2.6.1 创建定制版本

虚拟环境是基于公有环境创建，每个虚拟环境可以有不同的包或版本，所以也称为**定制版本**

```bash
# 虚拟环境
[python@19d125e0548a cmdb]$ pyenv virtualenv 3.5.3 magedu353
Requirement already satisfied: setuptools in /home/python/.pyenv/versions/3.5.3/envs/magedu353/lib/python3.5/site-packages
Requirement already satisfied: pip in /home/python/.pyenv/versions/3.5.3/envs/magedu353/lib/python3.5/site-packages


[python@19d125e0548a cmdb]$ tree ~/.pyenv/versions/ -L 2
/home/python/.pyenv/versions/
|-- 3.5.3 # 公有环境
|   |-- bin
|   |-- envs
     	|-- magedu353 # 定制版本
            |-- bin
            |-- include
            |-- lib
                `-- python3.5
                    `-- site-packages # 虚拟环境的包
                        |-- __pycache__
                        |-- easy_install.py
                        |-- pip
                        |-- pip-9.0.1.dist-info
                        |-- pkg_resources
                        |-- setuptools
                        `-- setuptools-28.8.0.dist-info
            |-- lib64 -> lib
            `-- pyvenv.cfg

|   |-- include
|   |-- lib
        `-- python3.5
            `-- site-packages # 公有环境的包
                |-- __pycache__
                |-- easy_install.py
                |-- pip
                |-- pip-9.0.1.dist-info
                |-- pkg_resources
                |-- setuptools
                `-- setuptools-28.8.0.dist-info
|   `-- share
`-- magedu353 -> /home/python/.pyenv/versions/3.5.3/envs/magedu353 # 链接定制版本

```



### 2.6.2 绑定虚拟环境

```bash
# 项目目录结构
[python@19d125e0548a cmdb]$ tree ~/magedu/
/home/python/magedu/
`-- projects # 所有项目
    `-- cmdb # cmdb项目
        `-- service1

# 新建项目web开发
 install -dv ~/magedu/projects/web

# 项目目录结构
[python@19d125e0548a cmdb]$ tree ~/magedu/
/home/python/magedu/
`-- projects
    |-- cmdb # magedu353虚拟环境
    |   `-- service1
    `-- web  # pypy虚拟环境
    
    

```

绑定cmdb虚拟环境magedu353

```bash
# 注意之前已经创建过虚拟环境
# pyenv virtualenv 3.5.3 magedu353
```



![image-20201231173630539](http://myapp.img.mykernel.cn/image-20201231173630539.png)

> 在local绑定之后，目录前面多出(magedu353), 这表示当前目录绑定了虚拟环境



绑定web至虚拟环境magedu-pypy-36-731

```bash
# 创建虚拟环境
[python@19d125e0548a ~]$ pyenv virtualenv pypy3.6-7.3.1 magedu-pypy-36-731
Looking in links: /tmp/tmpj_u2el4u
Requirement already satisfied: setuptools in /home/python/.pyenv/versions/pypy3.6-7.3.1/envs/magedu-pypy-36-731/site-packages (44.0.0)
Requirement already satisfied: pip in /home/python/.pyenv/versions/pypy3.6-7.3.1/envs/magedu-pypy-36-731/site-packages (20.0.2)

# 绑定
[python@19d125e0548a ~]$ cd ~/magedu/projects/web/
[python@19d125e0548a web]$ pyenv local magedu-pypy-36-731 
(magedu-pypy-36-731) [python@19d125e0548a web]$  # 前面出现括号表示虚拟环境成功
```



# 3. pip使用

## 3.1 pip配置镜像源

当我们直接安装包时，会非常卡, 如下

```bash
(magedu353) [python@19d125e0548a cmdb]$ pip install redis
Collecting redis
  Cache entry deserialization failed, entry ignored # 失败
  Cache entry deserialization failed, entry ignored
  Downloading https://files.pythonhosted.org/packages/a7/7c/24fb0511df653cf1a5d938d8f5d19802a88cef255706fdda242ff97e91b7/redis-3.5.3-py2.py3-none-any.whl (72kB)

```

所以我们需要给python的pip源配置镜像

https://mirrors.tuna.tsinghua.edu.cn/help/pypi/

```bash
# linux
install -dv ~/.pip


cat > ~/.pip/pip.conf <<EOF
[global]
index-url = https://mirrors.aliyun.com/pypi/simple/
[install]
trusted-host=mirrors.aliyun.com
EOF


# windows
# 打开%appdata% 目录
# 新建 pip目录
# 目录中配置pip.ini, 内容如下
[global]
index-url = https://mirrors.aliyun.com/pypi/simple/
[install]
trusted-host=mirrors.aliyun.com
```



## 3.2 安装ipython

增强的cpython

```bash
pip install ipython
```



## 3.3 安装使用jupyter

基于web的python运行界面，安装jupyter，也会自动安装依赖的ipython

```bash
# 升级pip
pip install pip -U 

# 安装
(magedu353) [python@19d125e0548a cmdb]$ pip install jupyter

# 验证安装包在虚拟环境非公有环境
# 1. 查看公有环境
(magedu-pypy-36-731) [python@19d125e0548a web]$ ls ~/.pyenv/versions/3.5.3/lib/python3.5/site-packages/
README  __pycache__  easy_install.py  pip  pip-9.0.1.dist-info  pkg_resources  setuptools  setuptools-28.8.0.dist-info

# 2. 查看虚拟环境，可以看到输出大量的包，包含ipython, 所以在虚拟环境中。
(magedu-pypy-36-731) [python@19d125e0548a web]$ ls ~/.pyenv/versions/3.5.3/envs/magedu353/lib/python3.5/site-packages/
IPython                    ipython_genutils                  pickleshare.py                   setuptools
Pygments-2.7.3.dist-info   ipython_genutils-0.2.0.dist-info  pip                              setuptools-28.8.0.dist-info
__pycache__                jedi                              pip-9.0.1.dist-info              six-1.15.0.dist-info
backcall                   jedi-0.17.2.dist-info             pkg_resources                    six.py
backcall-0.2.0.dist-info   parso                             prompt_toolkit                   traitlets
decorator-4.4.2.dist-info  parso-0.7.1.dist-info             prompt_toolkit-2.0.10.dist-info  traitlets-4.3.3.dist-info
decorator.py               pexpect                           ptyprocess                       wcwidth
easy_install.py            pexpect-4.8.0.dist-info           ptyprocess-0.7.0.dist-info       wcwidth-0.2.5.dist-info
ipython-7.9.0.dist-info    pickleshare-0.7.5.dist-info       pygments


# 安装失败
# 在pypy下安装
(magedu-pypy-36-731) [python@19d125e0548a web]$ pip install jupyter

# 配置密码
(magedu-pypy-36-731) [python@19d125e0548a web]$ jupyter notebook password
Enter password: 
Verify password: 
[NotebookPasswordApp] Wrote hashed password to /home/python/.jupyter/jupyter_notebook_config.json

# 运行
(magedu-pypy-36-731) [python@19d125e0548a web]$ nohup jupyter notebook  --no-browser --ip=0.0.0.0   &
LISTEN      0      128                                           *:8888                                                      *:*   
```

![image-20201231180409134](http://myapp.img.mykernel.cn/image-20201231180409134.png)



>  运行目录即工作目录



准备一个新文件

```bash
(magedu353) [python@19d125e0548a cmdb]$ cd ~/magedu/projects/web/
(magedu-pypy-36-731) [python@19d125e0548a web]$ touch test.py
```

![image-20201231180513581](http://myapp.img.mykernel.cn/image-20201231180513581.png)

也可以在网页右键新建文件

![image-20201231180603402](http://myapp.img.mykernel.cn/image-20201231180603402.png)



# 4. jupyter应用

## ctrl + enter

执行python指令

![image-20201231180729592](http://myapp.img.mykernel.cn/image-20201231180729592.png)

## shift + enter

![image-20201231180836721](http://myapp.img.mykernel.cn/image-20201231180836721.png)

## `M`arkdown书写

![image-20201231181023087](http://myapp.img.mykernel.cn/image-20201231181023087.png)



# 5. 导出包

虚拟环境的好处就在于其他项目隔离，每一个独立环境都可以使用pip导出已经安装的包，在另一个环境安装包

```bash
# 在cmdb环境导出
(magedu353) [python@19d125e0548a cmdb]$ pip freeze > cmdb.requirement
(magedu353) [python@19d125e0548a cmdb]$ ll cmdb.requirement 
-rw-rw-r-- 1 python python 887 Dec 31 10:12 cmdb.requirement

# 在web环境导入
(magedu-pypy-36-731) [python@19d125e0548a web]$ pip install -r ../cmdb/cmdb.requirement
```

# 6. 卸载虚拟环境

```bash
(magedu-pypy-36-731) [python@19d125e0548a web]$ pyenv uninstall magedu353
pyenv-virtualenv: remove /home/python/.pyenv/versions/3.5.3/envs/magedu353? y

(magedu-pypy-36-731) [python@19d125e0548a web]$ pyenv uninstall magedu-pypy-36-731 
pyenv-virtualenv: remove /home/python/.pyenv/versions/pypy3.6-7.3.1/envs/magedu-pypy-36-731? y
[python@19d125e0548a web]$ pyenv versions
pyenv: version `magedu-pypy-36-731' is not installed (set by /home/python/magedu/projects/web/.python-version)
  system
  3.5.3
  pypy3.6-7.3.1
```

# 7. windows平台

www.jetbrains.com/pycharm

建立项目、编写代码、运行