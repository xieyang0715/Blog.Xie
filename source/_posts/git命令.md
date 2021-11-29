---
title: "git命令"
date: 2021-05-07 10:48:30
tags:
- "git"
---



# 获取commit相关的信息

```bash
# 最近一次提交的message
git  --no-pager show -s --format="%s"  $(git rev-parse --short HEAD)

# 其他信息参考
https://blog.csdn.net/liurizhou/article/details/89234032
```

![image-20211002094408327](http://myapp.img.mykernel.cn/image-20211002094408327.png)