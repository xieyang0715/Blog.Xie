---
title: hexo更新及next主题更新
date: 2021-09-24 14:21:14
tags:
- hexo
toc: true
---

# 前言

今天由于gitalk挂了，好像不可以评论了。故更新主题





<!--more-->
# 更新npm

```bash
npm install -g npm
```

# 更新hexo

```bash
npm install --save hexo
```



# 更新next

## 主题配置与仓库独立 

https://theme-next.js.org/docs/getting-started/configuration

1. Please ensure you are using Hexo 5.0 (or later).

2. Create a config file in site's root directory, e.g. `_config.next.yml`.

3. Copy needed NexT theme options from theme config file into this config file. If it is the first time to install NexT, then copy the whole theme config file by the following command:

   ```
   # Installed through npm
   cp node_modules/hexo-theme-next/_config.yml _config.next.yml
   # Installed through Git
   cp themes/next/_config.yml _config.next.yml
   ```

## 备份next主题

```bash
mv thems/next thems/next-$(date +%F)
```

## 添加新的next主题

```bash
# git clone --recurse-submodules https://github.com/next-theme/hexo-theme-next themes/next
```

新的主题文件和旧的主题文件的配置文件对比

```bash
vimdiff _config.next.yml themes/next/_config.yml 
来处理不同，
]c  后一处不同
[c  前一处不同
do obtain对方的不同
dp push自己的不同到对方
```

可以使用此工具：https://github.com/slcnx/vimdiffprov

# 备份镜像

```bash
# docker commit -p 41b10e9098e9 registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:20210924
# docker login registry.cn-hangzhou.aliyuncs.com
# docker push registry.cn-hangzhou.aliyuncs.com/slcnx/hexo-slc-aliyun-auto-pro:20210924
```

