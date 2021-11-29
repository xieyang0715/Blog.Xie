---
title: pyinstaller打包exe
tags:
  - python常用模块
photos:
  - http://myapp.img.mykernel.cn/pyinstaller-draft1c-header-trans.png
date: 2021-11-14 16:37:42
---

# 前言

今天新安装的pycharm上开发完成python文件之后，打包pyinstaller之后，竟然不能加载模块。

原因是我使用的python版本(3.10)太高了, 安装python3.6解决

- pyinstaller仅支持`python: 3.5 - 3.9`,  PyInstaller’s main advantages over similar tools 
- pyinstaller类似工具:  `py2exe`   `autopy2exe` `cx_freeze`
  - py2exe is the same like pyinstaller, 
  - autopy2exe has UI for converting the script to executable,
  -  cx_freeze is platform independent exe creation method.


> 引用自: http://www.pyinstaller.org/

<!--more-->



# pyinstaller

可以将操作系统支持的动态链接库打包到成一个二进制文件(exe, ELF), 完全可以跨平台运行。

pyinstaller可以 将[支持的python包](https://github.com/pyinstaller/pyinstaller/wiki/Supported-Packages) 打包成一个二进制文件

## 使用

Install PyInstaller from PyPI:

```
pip install pyinstaller
```

Go to your program’s directory and run:

```
pyinstaller yourprogram.py
```

This will generate the bundle in a subdirectory called `dist`.

For a more detailed walkthrough, see the [manual](https://pyinstaller.readthedocs.io/).

## 选项

- -F 打包成单一的文件
- -D 打包成一个目录

