---
title: docker容器添加选项，容器获取启动命令
date: 2021-06-28 10:13:11
tags:
- 生产小经验
- docker
---

# docker 添加选项

```bash
docker container update --restart always <容器id|容器名>
```

# 获取命令

```python
pip install runlike
runlike -p <容器id|容器名>
```



