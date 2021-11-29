---
title: python开发环境使用
date: 2021-01-04 08:29:25
tags:
toc: true
---





使用jupyter的优点

- [x] 可以保存之前所有练习

- [x] 修改，重新ctrl/shift + enter运行，反复使用

- [x] []中的数字表示当前第几次输出，数字从小到大，正在运行，是*号，表示运行未结束

- [x] cell可以认为是控制台，ctrl/shift + enter仅会执行当前cell中的所有代码

- [x] ipython _引用上一个cell结果

  ![image-20210104164302631](http://myapp.img.mykernel.cn/image-20210104164302631.png)

- [x] cell**不需要使用print**, 并且**输出变量的类型**。字符串使用引号引用，非字符不用引号引用。

  ![image-20210104164235168](http://myapp.img.mykernel.cn/image-20210104164235168.png)

- [x] 配置中文

- [x] markdown/js/python代码书写

- [x] 函数执行结果有Out，表示有返回值。 结果没有Out，表示返回值 是 None

<!--more-->

# 1. jupyter配置中文

仅需要系统环境配置中文

```bash
# centos

yum -y install kde-l10n-Chinese  glibc-common
localedef -c -f UTF-8 -i zh_CN zh_CN.utf8
echo '
export LANG=zh_CN.utf8
' > /etc/profile.d/chinese.sh
source /etc/profile.d/chinese.sh


# ubuntu
apt-get install language-pack-zh* -y
echo 'LANG="zh_CN.UTF-8"' > /etc/default/locale
dpkg-reconfigure --frontend=noninteractive locales
update-locale LANG=zh_CN.UTF-8
```



# 2. 多种代码

jupyter支持markdown/python/javascripts代码。

1. markdown

   先在左侧点Ln, 方框变成蓝色， 然后按m,  前面的Ln和[]消失，输入markdown语言，ctrl/shift + enter执行即可

   ```markdown
   ~~你好~~
   
   **你好**
   
   ```

2. python

   ![image-20210104163809598](http://myapp.img.mykernel.cn/image-20210104163809598.png)

3. javascripts

   ![image-20210104163914010](http://myapp.img.mykernel.cn/image-20210104163914010.png)

   