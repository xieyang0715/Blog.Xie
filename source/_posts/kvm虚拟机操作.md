---
title: kvm虚拟机操作
date: 2021-01-13 02:23:31
tags: 
- 生产小经验
- kvm
toc: true
---



# 安装kvm程序包

打开虚拟化支持，这样创建的虚拟机可以二次运行虚拟机

```python
# 安装kvm
安装组件 脚本：http://myapp.img.mykernel.cn/install_kvm.sh
开启虚拟化支持
[root@centos7-iaas ~]# cat /sys/module/kvm_intel/parameters/nested
N

# /etc/modprobe.d/kvm-nested.conf
options kvm-intel nested=1
options kvm-intel enable_shadow_vmcs=1
options kvm-intel enable_apicv=1
options kvm-intel ept=1     

# modprobe -r kvm_intel
# modprobe -a kvm_intel
# cat /sys/module/kvm_intel/parameters/nested
Y

```

# 基于iso镜像安装虚拟机模板

以后直接基于模板快速启动虚拟机

```bash
# 基于iso镜像安装虚拟机模板
# ls /usr/local/src/CentOS-7-x86_64-Minimal-1804.iso

# qemu-img create -f qcow2 /var/lib/libvirt/images/centos7-1804-template-1C1G.qcow2 10G 


# 启动模板
# virt-install --virt-type kvm --name ubuntu-1804-template-2C-2G --ram 2048 --vcpus 2 --cdrom=/ISOs/ubuntu-18.04.3-server-amd64.iso --os-variant centos7.0 --disk path=/VMs/template/centos7-1804-template-2C2G.qcow2,bus=virtio,cache=none,io=native --network bridge=br0,model=virtio --graphics vnc,listen=0.0.0.0 --noautoconsole #--autostart

# 赶紧连接，或vnc连接
# virt-manager
```

> https://itnext.io/etcd-performance-consideration-43d98a1525a3
>
> https://www.cnblogs.com/wclwcw/p/8535001.html
>
> etcd运行kvm之上的优化
>
> ```bash
> --os-variant centos7.0
> osinfo-query os | less
> ```

```bash
# 优化主机
基础包
打开串行终端
edit the /etc/default/grub file and configure the GRUB_CMDLINE_LINUX option. Delete the rhgb quiet and add console=tty0 console=ttyS0,115200n8 to the option.
# centos
grub2-mkconfig -o /boot/grub2/grub.cfg
# ubuntu
update-grub

# 重启主机
reboot
```



# 宿主机直连虚拟机

```bash
# 查看虚拟机
virsh list --all

# 连接控制台    domain
virsh console 1
```

# 添加快照

```bash
# 虚拟机关机
# virsh snapshot-create-as --name NewOS --domain centos7-template
```



# 其他命令

```bash
# # virsh --help     #获取帮助
# virsh list --all #列出所有虚拟机
# virsh shutdown CentOS-7-x86_64 #正常关机
# virsh start CentOS-7-x86_64 #正常开机
# virsh destroy centos7 #强制停止/关机
# virsh undefine Win_2008_r2-x86_64 #强制删除
# virsh autostart centos7 #设置当前虚拟机开机自启动
# virsh autostart --disable centos7 #取消开机自动启动
# 快照相关命令
# virsh snapshot-list haproxy-12
# 恢复快照
# virsh snapshot-revert --snapshotname NewOS haproxy-12 
# virsh snapshot-delete centos7-bridge --snapshotname NewOS
```



# 基于模板创建虚拟机

在宿主机上给虚拟机准备一个虚拟桥, 编辑/etc/rc.local

```bash
# vmbr0 provider
# 添加桥接接口
brctl addbr vmbr0
# 配置桥接接口地址，此即为provider网络的网关
ifconfig vmbr0 172.16.0.1/16 up
# 需要内部通信完成SNAT转换源地址
iptables -t nat -A POSTROUTING -s 172.16.0.0/16 -j MASQUERADE
```

批量创建虚拟机

```bash
# 批量创建
 for i in {100..110}; do bash safe_clone_kvm.sh -i /VMs/template/ubuntu-1804-1C1G.qcow2 -d /VMs/kubernetes/172.16.0.$i  -t kvm  -n 172.16.0.$i -r 2048  -v 4 -b vmbr0; done

# 逐个连接配置
#virsh console <domain>
#配置ip
#配置主机名
#poweroff

# 批量添加快照
for i in {100..110}; do virsh snapshot-create-as --name NewOS --domain 172.16.0.$i; done
```



# 修改kvm特性

```bash
virsh destroy 192.168.0.11 && virsh undefine 192.168.0.11

 for i in 11; do virt-install --virt-type kvm --name 192.168.0.$i --os-variant centos7.0 --ram 4096 --vcpus 4 --disk path=/VMs/kubernetes/192.168.0.$i,bus=virtio,cache=none,io=native --graphics vnc,listen=0.0.0.0 --noautoconsole --autostart --network bridge=br0,model=virtio --import --cpu host; done
```

