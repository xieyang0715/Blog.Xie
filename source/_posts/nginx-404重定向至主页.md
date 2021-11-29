---
title: nginx 404重定向至主页
date: 2021-03-02 08:45:02
tags:
- 生产小经验 
- nginx
---

```bash
error_page   404    http://test.youwoyouqu.io;  # session会变
error_page   404    /index.html;                # session不变
```

