---
title: landscape配置menu的Header的归档和自定义目录
date: 2020-10-14 07:23:15
tags: hexo
toc: true
---

# landscape配置menu

```bash
vim themes/landscape/_config.yml
```

```yaml
menu:
  主页: /
  目录: /archives/
  简历: /about/
```

<!--more-->

添加目录

```bash
hexo new page 'archives'
hexo new page 'about'
```

```bash
[root@c4ef680948cd apps]# cat source/archives/index.md 
---
title: archives
date: 2020-10-14 14:41:39
---
```

```bash
[root@c4ef680948cd apps]# cat source/about/index.md 
---
title: about
date: 2020-10-14 14:58:35
---
```

