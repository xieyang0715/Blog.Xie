---
title: vim
date: 2021-03-15 08:43:11
tags:
- vim
- 生产小经验
---



# vim

将tab转换为空白

```bash
:%retab!
或
:%ret!
```

读文件

```bash
:read <path/to/file>
```



# 加载文件至剪切板

```bash
xclip <path/to/file>
```



# vim配置

```bash
set nu 
set cul
set tabstop=2
set expandtab
set shiftwidth=2
set ai
set softtabstop=2
```

> 行号
>
> 光标线
>
> 单行缩进
>
> tab -> space. 
>
> 多行缩进
>
> 自动缩进
>
> 回退格数