---
title: hexo添加文章加密
date: 2021-02-20 06:29:42
tags: 
- hexo
---

 

```bash
# 添加相同的文章名
addarticle diary-日记xxx
INFO  Created: /apps/source/_posts/diary-1.md 

# nginx侧对diary url进行访问限制
   location ~* "diary.*" {
     auth_basic "Administrator’s Area";
     auth_basic_user_file /apps/nginx/conf/.htpasswd;
     proxy_pass http://upstream_servers; # 此proxy_pass有高级功能
   }

# 生成用户
htpasswd -c /apps/nginx/conf/.htpasswd  aaa
/apps/nginx/sbin/nginx -s reload
```

参考：https://docs.nginx.com/nginx/admin-guide/security-controls/configuring-http-basic-authentication/