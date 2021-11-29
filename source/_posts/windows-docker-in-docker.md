---
title: "windows docker-in-docker"
date: 2021-04-26 16:20:33
tags:
- "生产小经验"
---

```bash
docker run -d -v //var/run/docker.sock:/var/run/docker.sock --name image-make -p 31103:22 ubuntu:18.04
```

安装docker

https://docs.docker.com/engine/install/

