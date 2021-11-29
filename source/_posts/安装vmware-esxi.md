---
title: 安装vmware esxi
date: 2020-11-06 07:27:22
tags: plan
toc: true
---





Vmware服务器虚拟化第一个产品叫ESX，后来Vmware在4版本的时候推出了ESXI，ESXI和ESX的版本最大的技术区别是内核的变化，从4版本开始VMware把ESX及ESXi产品统称为vSphere，但是VMware从5版本开始以后取消了原来的ESX版本，所以现在来讲的话vSphere就是ESXI，只是两种叫法而已。一般官方文档中以称呼vSphere为主。


<!--more-->

# 1. 下载vsphere镜像

下载页面: https://my.vmware.com/en/group/vmware/home

![image-20201106174704355](http://myapp.img.mykernel.cn/image-20201106174704355.png)

![image-20201106174724850](http://myapp.img.mykernel.cn/image-20201106174724850.png)

![image-20201106174750397](http://myapp.img.mykernel.cn/image-20201106174750397.png)

![image-20201106175331901](http://myapp.img.mykernel.cn/image-20201106175331901.png)

![image-20201106175440509](http://myapp.img.mykernel.cn/image-20201106175440509.png)



![image-20201106175519791](http://myapp.img.mykernel.cn/image-20201106175519791.png)

![image-20201106175532353](http://myapp.img.mykernel.cn/image-20201106175532353.png)

填写信息, 点击免费试用

![image-20201106175633860](http://myapp.img.mykernel.cn/image-20201106175633860.png)

![image-20201106175717177](http://myapp.img.mykernel.cn/image-20201106175717177.png)

点击下载镜像

![image-20201106175738930](http://myapp.img.mykernel.cn/image-20201106175738930.png)



# 2. 虚拟机加载镜像

![image-20201106175908504](http://myapp.img.mykernel.cn/image-20201106175908504.png)

![image-20201106175937371](http://myapp.img.mykernel.cn/image-20201106175937371.png)



选择vsphere

![image-20201106175955568](http://myapp.img.mykernel.cn/image-20201106175955568.png)

机械磁盘，C盘在外磁道更快。固态没有区别

![image-20201106180053618](http://myapp.img.mykernel.cn/image-20201106180053618.png)

## 2.1 分配磁盘*

vsphere上要跑很多虚拟机, 磁盘给大一些. 一般1个虚拟机2C,2G. 40-60G硬盘。例如当前8G内存跑4个主机，一个40G，就需要160G存储。本身还需要存储。所以分配200G

![image-20201106180234706](http://myapp.img.mykernel.cn/image-20201106180234706.png)

![image-20201106180242065](http://myapp.img.mykernel.cn/image-20201106180242065.png)

## 2.2 开启vt-x/EPT

vmware exsi是全虚拟化，必须开启此功能

![image-20201106180328837](http://myapp.img.mykernel.cn/image-20201106180328837.png)

## 2.3 dvd加载刚刚下载的镜像 *

注意使用**桥接**

![image-20201106180428483](http://myapp.img.mykernel.cn/image-20201106180428483.png)

# 3. 配置vsphere

ESXI 7.0分配了200G存储，但实际安装完却只有70G多可用的，原因是：

虚拟内存本身会占用120g，所以就变小了，exsi7.0以上会出现这个问题，建议使用exsi6.7。

当然7.0也有解决方法：
1、再安装一次exsi7.0，在启动后的第一个画面那一刻，赶紧按下shift+O
2、然后出现cdromBoot runweasel 后，我们在这一行后面加上autoPartitionOSDataSize=8192 也就是设置成8G，本身的120g太大了。

![image-20201106181106707](http://myapp.img.mykernel.cn/image-20201106181106707.png)

![image-20201106181152840](http://myapp.img.mykernel.cn/image-20201106181152840.png)

## 3.1 欢迎页 enter

![image-20201106181254912](http://myapp.img.mykernel.cn/image-20201106181254912.png)

## 3.2 接受license F11

![image-20201106181317883](http://myapp.img.mykernel.cn/image-20201106181317883.png)

## 3.3 选择磁盘 Enter

由于只有1个，直接回车

![image-20201106181342912](http://myapp.img.mykernel.cn/image-20201106181342912.png)

## 3.4 键盘布局 Enter

![image-20201106181419161](http://myapp.img.mykernel.cn/image-20201106181419161.png)

## 3.5 root密码 * Enter

数字、字母、大小写、特殊字符

![image-20201106181446521](http://myapp.img.mykernel.cn/image-20201106181446521.png)

## 3.6 确认安装 F11

![image-20201106181533721](http://myapp.img.mykernel.cn/image-20201106181533721.png)

![image-20201106181601814](http://myapp.img.mykernel.cn/image-20201106181601814.png)

## 3.7 安装完成，移除安装介质，并enter

![image-20201106181658908](http://myapp.img.mykernel.cn/image-20201106181658908.png)

移除安装介质

![image-20201106181727551](http://myapp.img.mykernel.cn/image-20201106181727551.png)

在enter

![image-20201106181741972](http://myapp.img.mykernel.cn/image-20201106181741972.png)

## 3.8 重启进入vsphere

F2 配置界面 

上面是当前主机的IP

![image-20201106181945268](http://myapp.img.mykernel.cn/image-20201106181945268.png)

F2进入配置界面

![image-20201106182030599](http://myapp.img.mykernel.cn/image-20201106182030599.png)

## 3.9 配置vsphere的静态ip地址 *

方便后期管理

![image-20201106182110641](http://myapp.img.mykernel.cn/image-20201106182110641.png)

enter

![image-20201106182125315](http://myapp.img.mykernel.cn/image-20201106182125315.png)

enter

![image-20201106182140565](http://myapp.img.mykernel.cn/image-20201106182140565.png)

空格

![image-20201106182234853](http://myapp.img.mykernel.cn/image-20201106182234853.png)

配置IP: 192.168.0.216, NETMASK: 255.255.255.0, 默认网关：192.168.0.1

enter

## 3.10 配置dns

![image-20201106182322339](http://myapp.img.mykernel.cn/image-20201106182322339.png)



![image-20201106182347711](http://myapp.img.mykernel.cn/image-20201106182347711.png)

enter



## 3.11 esc保存

![image-20201106182427254](http://myapp.img.mykernel.cn/image-20201106182427254.png)

## 3.12 打开shell和ssh

ssh用于中ssh, 方便虚拟机迁移

![image-20201106182504584](http://myapp.img.mykernel.cn/image-20201106182504584.png)

![image-20201106182520495](http://myapp.img.mykernel.cn/image-20201106182520495.png)

打开ssh

![image-20201106182536290](http://myapp.img.mykernel.cn/image-20201106182536290.png)

## 3.13 切换桌面

ctrl + alt + f1

![image-20201106182737062](http://myapp.img.mykernel.cn/image-20201106182737062.png)

## 3.14 允许ssh密码登陆 *

```bash
vi /etc/ssh/sshd_config
```

![image-20201106182841418](http://myapp.img.mykernel.cn/image-20201106182841418.png)

重启服务

```bash
service.sh restart
```



## 3.15 xshell连接 *

```shell
[c:\~]$ ssh  root@192.168.0.126
```

![image-20201106182944924](http://myapp.img.mykernel.cn/image-20201106182944924.png)

![image-20201106182958139](http://myapp.img.mykernel.cn/image-20201106182958139.png)

![image-20201106183007456](http://myapp.img.mykernel.cn/image-20201106183007456.png)

# 4. 网页访问vsphere

`http://192.168.0.126` 网页访问这个地址, ip是上面配置的静态ip

登陆用户和密码是root账户和密码

![image-20201106183115892](http://myapp.img.mykernel.cn/image-20201106183115892.png)

![image-20201106183207070](http://myapp.img.mykernel.cn/image-20201106183207070.png)

# 5. 许可证配置

如果后期许可过期，虚拟机不能启动

![image-20201106183242657](http://myapp.img.mykernel.cn/image-20201106183242657.png)

# 6. 配置存储

## 6.1 配置nfs存储

在另一台主机安装nfs

![image-20201106183321783](http://myapp.img.mykernel.cn/image-20201106183321783.png)

![image-20201106183344016](http://myapp.img.mykernel.cn/image-20201106183344016.png)

![image-20201106183358808](http://myapp.img.mykernel.cn/image-20201106183358808.png)

## 6.2 配置ios

![image-20201106183437668](http://myapp.img.mykernel.cn/image-20201106183437668.png)

![image-20201106183452875](http://myapp.img.mykernel.cn/image-20201106183452875.png)

上传

![image-20201106183505799](http://myapp.img.mykernel.cn/image-20201106183505799.png)

上传进度

![image-20201106183530467](http://myapp.img.mykernel.cn/image-20201106183530467.png)

![image-20201111135702318](http://myapp.img.mykernel.cn/image-20201111135702318.png)

# 7. 安装虚拟机

![image-20201111135741745](http://myapp.img.mykernel.cn/image-20201111135741745.png)

![image-20201111135809440](http://myapp.img.mykernel.cn/image-20201111135809440.png)

## 7.1 配置nfs存储 *

操作系统放nfs方便迁移, 但是在很多虚拟机时，需要nfs主机万兆网卡, 否则虚拟机会变慢

![image-20201111140229124](http://myapp.img.mykernel.cn/image-20201111140229124.png)

## 7.2 配置cpu, mem, disk, 安装介质

![image-20201111140757064](http://myapp.img.mykernel.cn/image-20201111140757064.png)

![image-20201111140845561](http://myapp.img.mykernel.cn/image-20201111140845561.png)

![image-20201111140859933](http://myapp.img.mykernel.cn/image-20201111140859933.png)

# 8. 启动虚拟机

![image-20201111140925590](http://myapp.img.mykernel.cn/image-20201111140925590.png)



# 9. 安装配置centos7虚拟机

以下为简略安装步骤，如果需要更详细解释请参考[Ubuntu安装及优化](http://liangcheng.mykernel.cn/2020/09/29/Ubuntu-Server-18-04%E7%9A%84%E5%AE%89%E8%A3%85-%E4%BC%98%E5%8C%96%E7%B3%BB%E7%BB%9F/#2-Ubuntu-server%E5%AE%89%E8%A3%85)

## 9.1 安装

加载光驱进入，后tab

![image-20201111143700197](http://myapp.img.mykernel.cn/image-20201111143700197.png)

传递内核参数`net.ifnames=0 biosdevname=0`

回车启动

![image-20201111143716461](http://myapp.img.mykernel.cn/image-20201111143716461.png)

![image-20201111142932843](http://myapp.img.mykernel.cn/image-20201111142932843.png)

![image-20201111142948842](http://myapp.img.mykernel.cn/image-20201111142948842.png)

![image-20201111142959340](http://myapp.img.mykernel.cn/image-20201111142959340.png)![image-20201111143012249](http://myapp.img.mykernel.cn/image-20201111143012249.png)![image-20201111143023519](http://myapp.img.mykernel.cn/image-20201111143023519.png)![image-20201111143032544](http://myapp.img.mykernel.cn/image-20201111143032544.png)![image-20201111143038818](http://myapp.img.mykernel.cn/image-20201111143038818.png)![image-20201111143044216](http://myapp.img.mykernel.cn/image-20201111143044216.png)![image-20201111143050927](http://myapp.img.mykernel.cn/image-20201111143050927.png)![image-20201111143103449](http://myapp.img.mykernel.cn/image-20201111143103449.png)![image-20201111143111135](http://myapp.img.mykernel.cn/image-20201111143111135.png)

![image-20201111142849198](http://myapp.img.mykernel.cn/image-20201111142849198.png)

重启

![image-20201111144405635](http://myapp.img.mykernel.cn/image-20201111144405635.png)

![image-20201111144504587](http://myapp.img.mykernel.cn/image-20201111144504587.png)

## 9.2 初始化

```bash
[root@localhost ~]# curl -O http://myapp.img.mykernel.cn/linux_template_install.sh

# 初始化
[root@localhost ~]# bash linux_template_install.sh --port=22 --allow-root-login=yes --allow-pass-login=yes --root-password=123456 --hostname=chengdu-huayang-centos-template.magedu.local --ipaddr=192.168.0.233 --netmask=255.255.255.0 --gateway=192.168.0.1 --dns=223.6.6.6 --author=songliangcheng --qq=2192383945 --desc="A test toy"

# 关机快照
[root@localhost ~]# poweroff

```

# 10. exsi快照

![image-20201111144816805](http://myapp.img.mykernel.cn/image-20201111144816805.png)

![image-20201111144857630](http://myapp.img.mykernel.cn/image-20201111144857630.png)