---
title: windows一键安装开源软件
tags:
  - windows
photos:
  - http://myapp.img.mykernel.cn/logo-square.svg
date: 2021-11-17 09:39:45
---

# 前言

windows上管理软件的一生非常麻烦，从图形化安装, 需要各种路径配置，写注册表，卸载时也需要到指定位置指定各种选项，非常消耗时间。

以下推荐几种管理软件的工具: 

- [scoop](https://github.com/ScoopInstaller/Scoop) 的[搜索软件的目录](https://github.com/rasa/scoop-directory) 
- [chocolatey](https://chocolatey.org)的[搜索软件的目录](https://community.chocolatey.org/packages)



以下介绍如何使用powershell上安装scoop和chocolatey

<!--more-->

# 如何进入管理员界面

http://www.howtogeek.com/194041/how-to-open-the-command-prompt-as-administrator-in-windows-8.1/

允许powershell执行本地脚本

```powershell
set-executionpolicy remotesigned -scope currentuser
```

# 安装scoop

> https://github.com/ScoopInstaller/Scoop/wiki/Quick-Start

```powershell
$psversiontable.psversion.major # should be >= 5.0
```

一键安装

```powershell
Invoke-Expression (New-Object System.Net.WebClient).DownloadString('https://get.scoop.sh')
# or
iwr -useb get.scoop.sh | iex
```

安装scoop在自定义目录

```powershell
$env:SCOOP='C:\scoop'
[environment]::setEnvironmentVariable('SCOOP',$env:SCOOP,'User')
iwr -useb get.scoop.sh | iex

```

安装app在自定义目录

```powershell
$env:SCOOP_GLOBAL='c:\apps'
[environment]::setEnvironmentVariable('SCOOP_GLOBAL',$env:SCOOP_GLOBAL,'Machine')
scoop install -g <app>

```

升级

```powershell
scoop update scoop
```

卸载

```powershell
scoop uninstall scoop
```



# 安装chocolatey

>  https://docs.chocolatey.org/en-us/choco/setup

```powershell
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
```

升级

```powershell
choco upgrade chocolatey
```

卸载

Most of Chocolatey is contained in `C:\ProgramData\chocolatey` or whatever `$env:ChocolateyInstall` evaluates to. You can simply delete that folder.

