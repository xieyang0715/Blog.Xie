---
title: 'Error response from daemon: error parsing HTTP 408 response'
date: 2021-02-23 06:01:24
tags: 
- 生产小经验
---

今天突然遇到docker拉不到镜像的情况，看加速也配置了, 就不能拉镜像。

因为默认去docker官方的镜像仓库下载镜像，最好用阿里云 清华大学 163这些仓库

# 阿里云

https://cr.console.aliyun.com

![image-20210223140922828](http://myapp.img.mykernel.cn/image-20210223140922828.png)

# 清华大学

百度无docker registry



# 163

https://yeasy.gitbook.io/docker_practice/install/mirror

```bash
{
  "registry-mirrors": [
    "https://hub-mirror.c.163.com",
    "https://mirror.baidubce.com"
  ]
}
```



# 中科大

http://mirrors.ustc.edu.cn/help/dockerhub.html?highlight=docker

```bash
{
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn/"]
}
```



# 合并

/etc/docker/daemon.json

```bash
{
  "registry-mirrors": [
    "https://hub-mirror.c.163.com",
    "https://mirror.baidubce.com",
    "https://docker.mirrors.ustc.edu.cn"
  ]
}
```

```bash
# systemctl daemon-reload
# systemctl restart docker
```

