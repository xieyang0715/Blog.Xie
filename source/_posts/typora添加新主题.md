---
title: typora添加新主题
date: 2021-10-09 09:14:33
tags:
- 学习方法
- typora
---



# 前言

今天早上看微信公众号，看到此文章说明学习方法中，引用到typora，并给了一个新的主题

> 原文链接: https://mp.weixin.qq.com/s/G2mRLg4nD-pZn8PeRcUykA



<!--more-->
# 学习新技能的整体思路

作为一个过来人，我把整个学习过程分成三个阶段：

1. 前期：花点时间选一门口碑上佳的入门教程，坚持下去，不要轻易换教程
2. 中期：跟着教程，先学会抄代码，从抄代码中学习代码思维，举一反三，在这个阶段要做好学习笔记，从零开始构建自己的知识库，方便后面速查。
3. 后期：确定方向（运维 或者 爬虫 或者 web ）后，去慕课网上找 Python 的项目实战课堂，一门小几百，质量很高，到这个阶段，要面向工作学编程，从项目中去巩固前面的基础，不断查缺补漏，学习项目开发完整流程。

# 怎样构建自己的知识库
关于这个，我只给出我自认为最完美的方案，仅你参考。

1. 写作技能：去学习下 Markdown 写作语法，真的能让写文章做笔记变成一种享受。
2. 写作工具：在本地写文章的话，一定要下载个 Typora，Windows 和 Mac 都可以用。个人CSS排版：https://github.com/iswbm/typora-theme
3. 在线托管：使用 Sphinx + Github + Readthedoc 搭建个人知识库。搭建教程：https://iswbm.com/134.html



# 如何使用typora排版

1. 访问https://github.com/iswbm/typora-theme

2. 进入主题文件夹

   文件 - **偏好设置** 

   ![image-20211009091819929](http://myapp.img.mykernel.cn/image-20211009091819929.png)

3. 克隆项目

   ```bash
   rwx@DESKTOP-HHT6P69 MINGW64 ~/AppData/Roaming/Typora/themes
   $ git clone https://github.com/iswbm/typora-theme.git
   ```

4. 拷贝文件, 将蓝色方框中的红色方框的文件拷贝到当前位置

   ![image-20211009091927689](http://myapp.img.mykernel.cn/image-20211009091927689.png)

5. 在typora中切换主题

   ![image-20211009092111777](http://myapp.img.mykernel.cn/image-20211009092111777.png)

> 乱花渐欲迷人眼，浅草才能没马蹄。

还是`github`简单

通过`shift+f12`, 可以吸收一些好的样式，应用到github上， 比如引用代码块，可以统一使用

`typora, hexo`

```css
blockquote {
  margin: 1.5625em 0;
  padding: 0.6rem;
  overflow: hidden;
  font-size: 1rem;
  page-break-inside: avoid;
  border-left: 0.3rem solid #42b983;
  border-radius: 0.3rem;
  box-shadow: 0 0.1rem 0.4rem rgba(0,0,0,0.05), 0 0 0.05rem rgba(0,0,0,0.1);
  background-color: #fafafa;
  border-color: #ff9100;  
}
```

# 如何在线托管

Sphinx + Github + Readthedoc 搭建个人知识库。搭建教程：https://iswbm.com/134.html
