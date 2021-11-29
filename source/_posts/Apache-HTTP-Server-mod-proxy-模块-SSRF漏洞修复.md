---
title: "centos Apache HTTP Server mod_proxy 模块 SSRF漏洞修复"
tags:
  - null
photos:
  - http://myapp.img.mykernel.cn/image-20211123134303027.png
date: 2021-11-23 13:42:24
---

# 前言

阿里云检测到此漏洞, 制作rpm

<!--more-->

# 制作流程 （失败)

## 准备工作

1. 准备目录 `yum install gcc rpm-build rpm-devel rpmlint make python bash coreutils diffutils patch  rpmdevtools`

   ```bash
   # useradd test
   # su - test
   [test@1c7f6faa5aea ~]$ rpmdev-setuptree
   [test@1c7f6faa5aea ~]$ ls
   rpmbuild
   [test@1c7f6faa5aea ~]$ tree rpmbuild/
   rpmbuild/
   ├── BUILD
   ├── RPMS
   ├── SOURCES
   ├── SPECS
   └── SRPMS
   ```

   > 参考: https://rpm-packaging-guide.github.io/

## 制作apr

1. 下载源码 https://apr.apache.org ， 将bz2源码放在`SOURCES`

2. 拷贝其中的apr.spec, 到`SPECS`

   `yum install libtool doxygen -y`

   将其中的`check`部分注释

   ```diff
    46 %check
   +47 # Run non-interactive tests
   +48 #pushd test
   +49 #make %{?_smp_mflags} all CFLAGS=-fno-strict-aliasing
   +50 #make check || exit 1
   +51 #popd
    52
    53 %install
   ```

3. 进入`rpmbuild`执行`rpmbuild -bb SPECS/apr.spec`

## 制作apr-utils

1. 下载源码 https://apr.apache.org ， 将bz2源码放在`SOURCES`

2. 拷贝其中的apr.spec, 到`SPECS`

   `yum install db4-devel postgresql-devel mysql-devel sqlite-devel unixODBC-devel nss-devel -y `

   将其中的`check`部分注释

   ```diff
   122
   123 %check
   +124 ## Run non-interactive tests
   +125 #pushd test
   +126 #make %{?_smp_mflags} all CFLAGS=-fno-strict-aliasing
   +127 #make check || exit 1
   +128 #popd
   129
   130 %install
   ```

3. 进入`rpmbuild`执行`rpmbuild -bb SPECS/apr-util.spec`

## 制作httpd

1. 先安装上面的构建结果作为构建环境

   ```bash
   [root@1c7f6faa5aea x86_64]# pwd
   /home/test/rpmbuild/RPMS/x86_64
   [root@1c7f6faa5aea x86_64]# yum install *.rpm
   ```

2. 下载源码 https://httpd.apache.org/ ， 将bz2源码放在`SOURCES`

3. 拷贝其中的apr.spec, 到`SPECS`

   `yum install -y zlib-devel libselinux-devel libuuid-devel apr-devel apr-util-devel pcre-devel openldap-devel lua-devel libxml2-devel openssl-devel autoconf libcurl-devel xmlto`

   

   将其中的`check`部分注释

   ```diff
   %check
   +## Check the built modules are all PIC
   +#if readelf -d $RPM_BUILD_ROOT%{_libdir}/httpd/modules/*.so | grep TEXTREL; then
   +#   : modules contain non-relocatable code
   +#   exit 1
   +#fi
   ```

   第1行添加

   ```bash
   %define _unpackaged_files_terminate_build 0
   ```

   configure添加`--enable-systemd`

4. 进入`rpmbuild`执行`rpmbuild -bb SPECS/httpd.spec`

   > 如果构建过程中有问题，避免重复构建，`rpmbuild -bb --short-circuit httpd.spec`, 此方法制作的rpm不可用，当测试通过时，需要重复第4步，制作rpm才可以使用

## 制作过程中的问题

`yum install jansson-devel libsystemd-devel`

可能需要升级编译gcc:  https://wpcademy.com/how-to-install-gcc-on-centos-7/

> 内存至少4G，才能编译

`nghttp2`版本低了:  http://www.srcmini.com/41785.html#heading_6 



# 使用src.rpm

apr, apr-utils, httpd

1. 下载srpm： http://pub.mirrors.aliyun.com/centos-vault/7.9.2009/os/Source/SPackages/
2. 下载源码 https://httpd.apache.org/ ，https://apr.apache.org 将bz2源码放在`SOURCES`
3. 修改spec
   - 版本号
   - 去掉补丁
   - 添加 `%define debug_package %{nil}`
   - 添加 `%define _unpackaged_files_terminate_build 0`
4. httpd的spec`%configure`段之后添加` --enable-systemd`