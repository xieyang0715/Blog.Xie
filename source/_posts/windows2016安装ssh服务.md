---
title: windows 2016 安装配置openssh server
date: 2021-04-21 11:24:00
tags:
- windows server
---
# 前言

公司开发的需要使用windows的jenkins做CI,CD，他需要推代码，所以就需要ssh服务。

安装openssh:

https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_overview

https://hostadvice.com/how-to/how-to-install-an-openssh-server-client-on-a-windows-2016-server/

https://www.hostwinds.com/guide/how-to-install-and-configure-openssh-windows-server-2016/



powershell：

https://www.cnblogs.com/lsgxeva/p/9309576.html

https://jeremyxu2010.github.io/2018/02/powershell%E5%AD%A6%E4%B9%A0%E5%A4%87%E5%BF%98/

https://docs.microsoft.com/en-us/powershell/scripting/overview?view=powershell-7.1



<!--more-->
# 安装openssh

## 下载安装包

从https://github.com/PowerShell/Win32-OpenSSH/releases，下载最新的Windows 2016使用openssh包

![image-20210421104451171](http://myapp.img.mykernel.cn/image-20210421104451171.png)

## 配置openssh

解压缩OpenSSH-Win64.zip文件并将其保存在控制台上。

将解压包放到C盘的目录,记录位置 `C:\Program Files (x86)\OpenSSH-Win64`

![image-20210421104723240](http://myapp.img.mykernel.cn/image-20210421104723240.png)

修改配置文件（在ssh服务启动后，才可以修改，修改之后再重启服务即可）

![image-20210421112046434](http://myapp.img.mykernel.cn/image-20210421112046434.png)

![image-20210421112103165](http://myapp.img.mykernel.cn/image-20210421112103165.png)

配置端口

```bash
Port 22
```

## 配置环境变量

打开控制台的“*控制面板”*。在“*系统和安全性”*部分，打开“**系统”**。单击**高级系统设置**，然后在*系统属性*对话框中，单击**环境变量**。

![image-20210421104826356](http://myapp.img.mykernel.cn/image-20210421104826356.png)

![image-20210421104839261](http://myapp.img.mykernel.cn/image-20210421104839261.png)

在对话框下半部分的“*系统变量”*部分中，选择“**Path”**。点击**编辑...**。

![image-20210421104853109](http://myapp.img.mykernel.cn/image-20210421104853109.png)

点击**新建**。添加OpenSSH文件夹路径。添加文件夹后，现在可以关闭“*系统属性”>“*对话框”。

![image-20210421104910761](http://myapp.img.mykernel.cn/image-20210421104910761.png)

```bash
setx PATH "$env:path;C:\Program Files (x86)\OpenSSH-Win64" -m
```



## 安装openssh

以管理员身份运行Powershell。在适当的字段中输入OpenSSH文件夹路径。要安装OpenSSH，请运行“。\ install-sshd.ps1” 命令。如果出现“成功安装sshd和ssh-agent服务”行 ，则表明安装成功。

```powershell
PS C:\Windows\system32> CD "C:\Program Files (x86)\OpenSSH-Win64\"
PS C:\Program Files (x86)\OpenSSH-Win64> .\install-sshd.ps1
[SC] SetServiceObjectSecurity 成功
[SC] ChangeServiceConfig2 成功
[SC] ChangeServiceConfig2 成功
sshd and ssh-agent services successfully installed
PS C:\Program Files (x86)\OpenSSH-Win64>
```

## 生成主机密钥

```powershell
PS C:\Program Files (x86)\OpenSSH-Win64> .\ssh-keygen.exe -A
PS C:\Program Files (x86)\OpenSSH-Win64>
```

## sshd服务

重新打开*控制面板*，然后单击**管理服务/管理工具**。在出现的对话框中，点击**服务**。在列表中找到“ sshd”，并将启动类型更改为**Automatic**。然后，运行sshd。

![image-20210421105954181](http://myapp.img.mykernel.cn/image-20210421105954181.png)

```bash
# 服务自动启动
Set-Service sshd -StartupType Automatic; Set-Service ssh-agent -StartupType Automatic;
# 启动服务
Stop-Service sshd; Stop-Service ssh-agent
Start-Service sshd; Start-Service ssh-agent
```

最好使用管理员权限

![image-20210426154128296](http://myapp.img.mykernel.cn/image-20210426154128296.png)

## 防火墙 22

在*控制面板中，*单击**系统安全**，然后单击**Windows防火墙**。单击对话框左侧面板中的“**高级设置**”。

在“*具有高级安全性*的*Windows防火墙”*对话框中，从左侧菜单中选择“**入站规则”**。从右侧菜单中选择“**新建规则**”。

这将打开“*新建入站规则向导”*对话框。当“*新建入站规则向导”*对话框打开时，从对话框左侧的菜单中选择**“协议和端口**”。选择“**自定义”**选项，然后单击“**下一步”**。

在“*程序”*部分中，选择“**所有程序”**

最后，在“*协议和端口”*部分中，选择“ **TCP** ”，然后在“*特定本地端口”*框中输入“ **22** ” 。完成并退出“*新建入站规则向导”*

现在，您已成功确保SSH设置可以连接到Windows服务器。

```powershell
New-NetFirewallRule -DisplayName "OpenSSH-Server-In-TCP" -Direction Inbound -LocalPort 22     -Protocol TCP -Action Allow
```

> https://docs.microsoft.com/en-us/powershell/module/netsecurity/new-netfirewallrule?view=windowsserver2019-ps



## openssh 默认shell

```powershell
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell -Value "C:\Program Files\Git\usr\bin\bash.exe" -PropertyType String -Force
```



