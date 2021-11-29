---
title: find多文件归档
date: 2021-02-26 11:44:27
tags:
- 生产小经验
---


备份生产写的k8s yaml文件时，jar文件太大需要排除了然后归档，没有找到合适的选项，find结果归档如下
```bash
find weizhixiu/ ! -name "*.jar" -type f -print0 | xargs -0 tar cvf weizhixiu-2021-02-26-dev.tar 
```

