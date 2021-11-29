---
title: rsync 排除文件
date: 2020-12-15 08:09:45
tags:
- 日常挖掘
- rsync
---



今天在使用rsync使用排除，误删除.git文件, 记录一下rsync

```bash
/usr/bin/rsync   -avzpP --delete --exclude=.git --exclude=*.swp    root@10.21.107.1:/opt/linux_scripts/ /tools/downloads/linux_scripts/

# avzpP 能用选项
# --delete 源目录没有，目标目录是否删除？
# --exclude=PATTERN以模式匹配文件, .git表示.git文件。 *.swp所有.swp结尾的文件。可以有多个选项
# 源目录：/path/to/somedir 表示复制目录。  
         #/path/to/somedir/表示复制目录下的文件
```

同步abc目录下的所有文件至目标主机，源目录的文件在目标主机不存在，则**不删除**

```bash
/usr/bin/rsync   -avzpP  --exclude=.git --exclude=*.swp    root@10.21.107.1:/opt/abc/ /tools/downloads/abc/
```

同步abc目录下的所有文件至目标主机，源目录的文件在目标主机不存在，则**删除**

```bash
/usr/bin/rsync   -avzpP  --exclude=.git --exclude=*.swp --delete   root@10.21.107.1:/opt/abc/ /tools/downloads/abc/
```

同步abc目录至目标主机, 没有 incline line.

```bash
/usr/bin/rsync   -avzpP  --exclude=.git --exclude=*.swp --delete   root@10.21.107.1:/opt/abc /tools/downloads/abc/
```

