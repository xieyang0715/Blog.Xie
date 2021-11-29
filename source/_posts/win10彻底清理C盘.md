---
title: win10彻底清理C盘
date: 2021-10-28 05:43:06
tags:
- windows
---



# 前言

今天更新win10失败，网上说需要至少16G空间，目前很少的C盘。

> 网上有说明至少20G的磁盘空间，才可以更新
>
> http://baijiahao.xitongzhijia.net/article/212603.html

> 更新工具：https://support.microsoft.com/en-us/topic/windows-10-update-assistant-3550dfb2-a015-7765-12ea-fba2ac36fb3f
>
> 更新前的准备，避免失败：http://www.xitongcheng.com/jiaocheng/dnrj_article_63237.html
>
> http://m.dnpz.net/diannaozhishi/3656.html?ivk_sa=1025883j
>
> win10 设置 更新 疑难解答 - windows更新, 几乎可以解决更新问题

<!--more-->
# 引用

磁盘占用快速分析工具: https://diskanalyzer.com/download

修改虚拟内存位置: https://www.bilibili.com/read/cv8221641

关闭休眠: https://www.cnbeta.com/articles/tech/958489.htm

新的软件存放位置修改：https://www.eefocus.com/e/489859

预览版查看错误码：https://docs.microsoft.com/en-us/windows/deployment/update/windows-update-errors



# 用户数据更新位置

```powershell
setx USERPROFILE I:\users\liangcheng
setx APPDATA   I:\users\liangcheng\AppData\Roaming
setx ALLUSERSPROFILE I:\ProgramData
setx ProgramData I:\ProgramData
```
计算机\HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList
regedit.msc 更新路径
