---
title: '自动化发布.net程序到IIS'
tags:
- CI
- windows
- jenkins
- gitlab
photos:
- http://myapp.img.mykernel.cn/GHVfby4xDTutXfd-t2erYA.png
date: 2021-11-16 15:05:15
---

# 前言

我们公司还有部分业务在使用.NET开发语言，这个开发出来的程序需要在windows IIS(windows上可以托管任何服务)上跑业务。

起初开发人员每天都需要开发.NET程序，通过本地的`visual studio`进行编译之后，然后上传到Windows IIS之上的`FTP`服务，之后再上windows 之上，将FTP的业务拷贝到windows IIS应用的工作目录中。

以上这种每天都需要重复进行发布程序的操作，可以通过以下自动化的方式完成。

1. visual studio配置自动发布到IIS/FTP.

   > 参考: https://jhrs.com/2019/32202.html

2. jenkins获取代码, 构建，使用msdeploy发布到IIS。

   > 参考:  https://jhrs.com/2019/32202.html

3. `jenkins`获取代码, 构建,   使用`powershell`上传目标FTP, 在使用`powershell`远程执行命令完成，目标主机发布到`IIS`站点。 

   > 下面我将介绍此种方法的使用



此外，有一些学习powershell的方法:

- Microsoft Virtual Academy: [Getting Started with PowerShell](https://channel9.msdn.com/Series/GetStartedPowerShell3)
- [Why Learn PowerShell](https://blogs.technet.microsoft.com/heyscriptingguy/2014/10/18/weekend-scripter-why-learn-powershell/) by Ed Wilson
- PowerShell Web Docs: [Basic cookbooks](https://docs.microsoft.com/powershell/scripting/samples/sample-scripts-for-administration)
- [The Guide to Learning PowerShell](https://www.idera.com/resourcecentral/whitepapers/powershell-ebook) by Tobias Weltner
- [PowerShell-related Videos](https://channel9.msdn.com/Search?term=powershell#ch9Search) on Channel 9
- [PowerShell Quick Reference Guides](https://www.powershellmagazine.com/2014/04/24/windows-powershell-4-0-and-other-quick-reference-guides/) by PowerShellMagazine.com
- [Learn PowerShell Video Library](https://community.idera.com/database-tools/powershell/video_library/) from Idera
- [PowerShell 5 How-To Videos](https://blogs.technet.microsoft.com/tommypatterson/2015/09/04/ed-wilsons-powershell5-videos-now-on-channel9-2/) by Ed Wilson
- [PowerShell Documentation](https://docs.microsoft.com/powershell)
- [Interactive learning with PSKoans](https://aka.ms/pskoans)

> 引用自:  https://github.com/PowerShell/PowerShell/tree/master/docs/learning-powershell

<!--more-->



# 使用jenkins自动发布.NET到IIS

给开发一个账号，可以操作`jenkins`的job, 然后点击`jenkins`的job选择发布哪个分支，就可以一键发布。

1. 先把开发那里获取到的代码，放在IIS上得可以访问

   > https://jhrs.com/2018/8804.html

2. 安装配置jenkins,` jenkins在linux不能发布.net需要在windows slave上完成，每个.net任务就使用节点标签选择windows slave.`
   - [Jenkins environment installation](https://znlive.com/jenkins-continuous-integration-asp-net-website#Jenkins_environment_installation)
   - [Jenkins plugin installation](https://znlive.com/jenkins-continuous-integration-asp-net-website#Jenkins_plugin_installation)
   - [Configure MSbuild](https://znlive.com/jenkins-continuous-integration-asp-net-website#Configure_MSbuild)
   - [New Item](https://znlive.com/jenkins-continuous-integration-asp-net-website#New_Item)
   - [Configure build task parameters](https://znlive.com/jenkins-continuous-integration-asp-net-website#Configure_build_task_parameters)

3. 重点配置jenkins
   - [Build (quite important settings)](https://znlive.com/jenkins-continuous-integration-asp-net-website#Build_quite_important_settings)

> 引用自: https://znlive.com/jenkins-continuous-integration-asp-net-website

## powershell发布测试环境

接下来讲说明如何通过`powershell`命令来发布站点, 以下示例：jenkins slave服务和目标主机内网(192.168.1.123)是通的, 现在通过内网发布构建的结果到目标服务器的IIS。

1. 安装powershell插件

   > https://www.cnblogs.com/sparkdev/p/7224171.html

2. 编写powershell, 从以上构建输出的结果中，获取包，之后把包归档

   ```powershell
   # powershell以管理员工作
   $currentWi = [Security.Principal.WindowsIdentity]::GetCurrent()
   $currentWp = [Security.Principal.WindowsPrincipal]$currentWi
    
   if( -not $currentWp.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator))
   {
     $boundPara = ($MyInvocation.BoundParameters.Keys | foreach{
        '-{0} {1}' -f  $_ ,$MyInvocation.BoundParameters[$_]} ) -join ' '
     $currentFile = (Resolve-Path  $MyInvocation.InvocationName).Path
    
    $fullPara = $boundPara + ' ' + $args -join ' '
    Start-Process "$psHome\powershell.exe"   -ArgumentList "$currentFile $fullPara"   -verb runas
    return
   }
   $remote_file="service1.zip"
   # powershell编写函数，归档输出的目录
   function CompressFile($FileUrl,$Destination){
   
       if(Test-Path $Destination){
          Remove-Item $Destination
          Write-Host "Delete $Destination Success" -ForegroundColor Green
       }
       # 本地是开发人员的web.config, 由于只是拷贝文件，就直接不使用开发的web.config即可
       if(Test-Path D:\project\service1\Web.config){
           Remove-Item D:\project\service1\Web.config
           Write-Host "Delete Web.config Success" -ForegroundColor Green
       }
       
       Write-Host "Start Compress $FileUrl to $Destination"
       try{
           Compress-Archive -Path $FileUrl -DestinationPath $Destination
       }catch{
           Write-Host "Compress Error" -ForegroundColor Red
       }
       Write-Host "Compress Finished" -ForegroundColor Green
   }
   
   CompressFile "D:\project\service1\*" "d:\"+"$remote_file"
   ```

   1. `D:\project\service1` 构建结果的目录
   2. `d:\service1.zip`打包后的目录

3. 编写powershell, 将打包的zip文件上传到目标的FTP服务器

   > 配置FTP服务器:  https://neoserver.site/help/setting-ftp-server-windows-server-2016

   ```powershell
   $remote_uri = "ftp://192.168.1.123/Publish/"+"$remote_file"
   
   function UploadFileToFTP($User,$Password){
       $webclient = New-Object System.Net.WebClient
       $webclient.Credentials = New-Object System.Net.NetworkCredential($user,$password)  
       $uri = New-Object System.Uri($remote_uri) 
       Write-Host "remote uri: $uri"
       $webclient.UploadFile($uri,$remote_file)
       if(echo $?){
           Write-Host "Uploading $item succeed" -ForegroundColor Green
       }else{
           Write-Host "Uploading $item failed" -ForegroundColor Red
       }
   }
   $user = "administrator"
   $password = "123456"
   UploadFileToFTP $user $password
   ```

4. 在目标服务器展开压缩包

   > 在目标服务器完成一些预配置操作
   >
   > 1. 目标主机打开winrm服务
   >
   >    ```powershell
   >    net start winrm
   >    ```
   >
   > 2. 目标主机信任源主机
   >
   >    ```powershell
   >    Set-Item WSMan:\localhost\Client\TrustedHosts -Force -Value '发起powershell命令的主机IP'
   >    ```
   >
   > 3. 执行命令的源主机
   >
   >    ```powershell
   >    Set-Item WSMan:\localhost\Client\TrustedHosts -Force -Value '*'
   >    ```
   >    
   >    
   >    
   > 3. 目标主机
   >
   >    ```powershell
   >    Enable-PSRemoting -Force;
   >    ```

   ```powershell
   $servers="192.168.1.123"
   $pwd=ConvertTo-SecureString $password -AsPlainText -Force; 
   $cred=New-Object System.Management.Automation.PSCredential($user,$pwd); 
   Invoke-Command -ComputerName $servers -credential $cred -ErrorAction Stop -ScriptBlock {
       Invoke-Expression -Command 'C:\Windows\System32\inetsrv\appcmd.exe stop site "demo" ' 
       Invoke-Expression -Command 'Expand-Archive D:\ftproot\Publish\service1.zip D:\demo -f' 
       Invoke-Expression -Command 'C:\Windows\System32\inetsrv\appcmd.exe start site "demo" ' 
   }
   ```

   1. `D:\ftproot\Publish\agenttest.zip` 此目录就是ftp的根目录
   2. `stop site` 和`start site`就是管理站点的命令
   2. `appcmd`使用: https://docs.microsoft.com/en-us/iis/get-started/getting-started-with-iis/getting-started-with-appcmdexe

5. 退出命令

   ```powershell
   exit
   ```

   

## powershell发布灰度环境

jenkins调用脚本，powershell调用python完成, 将测试环境的包拷贝到灰度的应用程序目录，只是将web.config文件通过正则替换即可

```powershell
$currentWi = [Security.Principal.WindowsIdentity]::GetCurrent()
$currentWp = [Security.Principal.WindowsPrincipal]$currentWi

$servers="192.168.1.123"
$user = ""
$password = ''

$PARAM = ""
foreach ($n in $args){
    $PARAM += "$n" + " "
}
echo $PARAM

$pwd=ConvertTo-SecureString $password -AsPlainText -Force; 
$cred=New-Object System.Management.Automation.PSCredential($user,$pwd); 
Invoke-Command -ComputerName $servers -credential $cred -ErrorAction Stop -ScriptBlock {
    param($PARAM)
    Invoke-Expression -Command "C:\Python\python.exe $PARAM" 
} -ArgumentList $PARAM
exit
```

