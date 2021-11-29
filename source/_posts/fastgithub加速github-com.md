---
title: fastgithub加速github.com
date: 2021-10-27 11:41:31
tags:
- github
---

# 前言

由于自己的博客中的主题需要拉github.com完成主题更新，但是非常慢。

网络上有一个github加速的项目:  [FastGithub](https://github.com/dotnetcore/FastGithub)

> 官方已经有linux, windows的一键启动二进制包，本次为了方便，制作镜像。

一般我们只需要拉github.com时临时才使用代理，所以没有必要一直使用这个代理。



<!--more-->
# 通过ssh克隆

一位群友的提供

```bash
root@mykernel:~# git clone git@github.com:kubernetes/kubernetes.git
Cloning into 'kubernetes'...
remote: Enumerating objects: 1277944, done.
remote: Counting objects: 100% (189/189), done.
remote: Compressing objects: 100% (127/127), done.
Receiving objects:  18% (230030/1277944), 159.25 MiB | 6.67 MiB/s   
```

> 实测的确快

# 下面准备了一个fastgithub制作过程

## 克隆官方仓库最新版本

> 官方: https://github.com/dotnetcore/FastGithub/releases/tag/2.0.4

```bash
wget https://github.com/dotnetcore/FastGithub/releases/download/2.0.4/fastgithub_linux-x64.zip
```

## 准备Dockerfile

在展开的目录中，编写`Dockerfile`

```bash
FROM ubuntu:18.04
LABEL MAINTAINER=slcnx

RUN apt update &&  apt install  libicu-dev libgssapi-krb5-2 libssl-dev -y --fix-missing && \
    apt install -y locales && rm -rf /var/lib/apt/lists/* && localedef -i zh_CN -c -f UTF-8 -A /usr/share/locale/locale.alias zh_CN.UTF-8
ENV LANG zh_CN.utf8

ADD . /fastgithub/
EXPOSE 38457
CMD /fastgithub/fastgithub
```

## 准备示例的docker-compose

```yaml
version: "3.7"
services:
  fastgithub:
    image: slcnx/fastgithub
    network_mode: host
    restart: always
    volumes:
    - cacert:/fastgithub/cacert/
  sample:
    depends_on:
    - fastgithub
    image: slcnx/ubuntu:18.04
    volumes:
    - cacert:/tmp/cacert
    working_dir: /hugo_data
    restart: on-failure
    tty: true
    entrypoint: sh -c 'cp /tmp/cacert/fastgithub.cer /usr/local/share/ca-certificates/fastgithub.crt && update-ca-certificates && cat'
    command: ""
    environment:
      https_proxy: http://127.0.0.1:38457
      http_proxy: http://127.0.0.1:38457
    network_mode: host
volumes:
  cacert: {}
```

> 1. `network_mode` 主机模式，为了可以多个容器可以通过`localhost`通信。 一般只需要拉github.com仓库的镜像才需要工作于host
> 2. `depends_on` 拉代码的主机肯定后于`fastgithub`运行
> 3. `volumes` 这个是将`fastgithub`生成的`cacert` 通过卷在多个容器间共享
> 4. `entrypoint` 这个是将`fastgithub`中的`cacert`证书，加载到自己的linux主机。`update-ca-certificates -d`会扫描`/usr/share/ca-certificates`下的`*.crt`文件。或者不加`-d` 会扫描`/usr/local/share/ca-certificates下`的`*.crt`文件。
> 5. ` restart: on-failure` 当脚本退出码非0，就重来，是0，正常结束，就OK

运行

```bash
docker-compose up --remove-orphans 
```

![image-20211027120752770](http://myapp.img.mykernel.cn/image-20211027120752770.png)

## 测试走代理

```bash
docker exec -it hugo_sample_1 bash
root@alyy:/hugo_data# git clone https://github.com/kakawait/hugo-tranquilpeak-theme
Cloning into 'hugo-tranquilpeak-theme'...
remote: Enumerating objects: 6394, done.
remote: Counting objects: 100% (1324/1324), done.
remote: Compressing objects: 100% (686/686), done.
remote: Total 6394 (delta 736), reused 1026 (delta 517), pack-reused 5070
Receiving objects: 100% (6394/6394), 17.63 MiB | 15.88 MiB/s, done.
Resolving deltas: 100% (3499/3499), done.
```

可以观察到速度非常快, 并且代理日志显示走的代理

![image-20211027120902021](http://myapp.img.mykernel.cn/image-20211027120902021.png)

![image-20211027120933154](http://myapp.img.mykernel.cn/image-20211027120933154.png)

# 示例使用此demo拉镜像

假如你的/build_data目录下，需要一个github.com上的kubernetes仓库，给一个应用使用

1. 准备一个跨容器的数据卷
2. 需要源码的容器就挂载这个目录即可

```diff
version: "3.7"
services:
  fastgithub:
    image: slcnx/fastgithub
    network_mode: host
    restart: always
    volumes:
    - cacert:/fastgithub/cacert/
  sample:
    depends_on:
    - fastgithub
    image: slcnx/ubuntu:18.04
    volumes:
    - cacert:/tmp/cacert
+    - build_data:/build_data
    working_dir: /build_data
    restart: on-failure
    tty: true
+    entrypoint: sh -c 'cp /tmp/cacert/fastgithub.cer /usr/local/share/ca-certificates/fastgithub.crt && update-ca-certificates && git clone https://github.com/kubernetes/kubernetes.git'
    command: ""
    environment:
      https_proxy: http://127.0.0.1:38457
      http_proxy: http://127.0.0.1:38457
    network_mode: host
  build:
    working_dir: /build_data
    depends_on:
+    - sample
    image: nginx
    volumes:
+    - build_data:/build_data
volumes:
  cacert: {}
+  build_data: {}
```

> build镜像后于sample拉源码
>
> sample拉源码，需要手工替换
>
> <strong style="color: red">注意: 其他的你要使用这个源码的镜像的docker-compose 选项就自已按照想要的方式来定义即可, 例如: restart: always|on-failure|...</strong>

```bash
docker-compose up --remove-orphans 
```

> 查看日志
>
> ```bash
> remote: Compressing objects: 100% (119/119), done.
> fastgithub_1  | 2021-10-27T04:33:34.2991830+00:00 [INF]
> fastgithub_1  | FastGithub.HttpServer.RequestLoggingMiddleware
> fastgithub_1  | POST https://github.com/kubernetes/kubernetes.git/git-upload-pack responded 200 in 73534.4064 ms
> fastgithub_1  | 
> remote: Total 1277733 (delta 80), reused 62 (delta 61), pack-reused 1277553
> Receiving objects: 100% (1277733/1277733), 785.83 MiB | 10.92 MiB/s, done..47 MiB/s   
> ```
>
> > 也达到了10.92MB/s, 速度非常感人

最终可以看到，克隆1.2G的内容，我才等了1分钟的时间

```bash
root@66312944cca2:/build_data# du -sh kubernetes/
1.2G    kubernetes/
```



# 给fastgithub提PR

先fork官方镜像，然后添加docker-compose.yaml

![image-20211027124130750](http://myapp.img.mykernel.cn/image-20211027124130750.png)

对README.md更新，docker启动方式

![image-20211027124534371](http://myapp.img.mykernel.cn/image-20211027124534371.png)

![image-20211027124551618](http://myapp.img.mykernel.cn/image-20211027124551618.png)

现在给作者提交PR, 即merge请求

![image-20211027124638712](http://myapp.img.mykernel.cn/image-20211027124638712.png)

![image-20211027124800505](http://myapp.img.mykernel.cn/image-20211027124800505.png)

现在此仓库的pull request, 将有自己的请求合并的请求。

![image-20211027124847353](http://myapp.img.mykernel.cn/image-20211027124847353.png)

