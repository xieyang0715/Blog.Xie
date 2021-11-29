---
title: 工作空间RGB配置
date: 2021-01-08 16:49:36
tags:
toc: true
---

ps配置颜色`编辑-颜色设置`

![image-20210108165203361](http://myapp.img.mykernel.cn/image-20210108165203361.png)

主要配置工作空间的RGB，CMYK

CMYK根据国家来使用。

RGB建议Adobe RGB(1998)来获取更多的颜色



SRGB: 默认RGB，小色彩空间，屏幕、移动设备、平板设备、电视基于SRGB色彩空间。 

Adobe RGB(1998), 较大的色彩空间，使用ps工作非常流行。

ProPhoto RGB, 最大的色彩空间，可包含显示器不能显示的颜色。**广色域**, 电视也支持广色域，即色彩空间支持ProPhoto RGB。硬件也相对昂贵。

CMYK，印刷行业，限制的色彩空间。只需要显示 。





区别

- 加色模式：即3通道的颜色相加
  - SRGB
  - Adobe RGB
  - ProPhoto RGB
- 减色模式：墨色



<!--more-->

# CMYK区别RGB模式

RGB比CMYK更亮，打印出来肯定较暗。

![](http://myapp.img.mykernel.cn/20190220005647-597970025_jpeg_360_360_15501.jpg)

## 加色模式

色板点开

![image-20210108170621851](http://myapp.img.mykernel.cn/image-20210108170621851.png)

![image-20210108170701043](http://myapp.img.mykernel.cn/image-20210108170701043.png)

把颜色拉出来

![image-20210108170810470](http://myapp.img.mykernel.cn/image-20210108170810470.png)

点击更多，选择RGB

![image-20210108170845095](http://myapp.img.mykernel.cn/image-20210108170845095.png)

当所有色为0，显示黑色

![image-20210108170935102](http://myapp.img.mykernel.cn/image-20210108170935102.png)

红色最大，红色。

绿色最大，红+绿=黄色。

![image-20210108171016384](http://myapp.img.mykernel.cn/image-20210108171016384.png)

绿+蓝=青

![image-20210108171112046](http://myapp.img.mykernel.cn/image-20210108171112046.png)

3个颜色混合，白色

![image-20210108171028296](http://myapp.img.mykernel.cn/image-20210108171028296.png)

## 减色模式

切换至CMYK

![image-20210108171219344](http://myapp.img.mykernel.cn/image-20210108171219344.png)

可以看到是百分比，是油墨的百分比。

如果将他们混合，所有的墨水泼到一起，就是黑色，所以是`减色模式`

![image-20210108171314068](http://myapp.img.mykernel.cn/image-20210108171314068.png)

# 颜色空间的色彩范围

## 视频中的标准

Adobe RGB 只是虚线部分

![img](http://myapp.img.mykernel.cn/8c191b2a705e4508ae7c82e74e315530.jpeg)

## 硬件屏幕

LCD非常少的颜色，不鲜艳

OLED 显示更多



硬件决定你显示的颜色

![](http://myapp.img.mykernel.cn/QQ图片20210108173232.png)

## RGB模式区域

sRGB不足

adobe rgb可以完全包含cmyk，所以打印后，会缺失多余的部分颜色

prophoto rgb是包含最多的颜色

![img](http://myapp.img.mykernel.cn/v2-75a2192764dfe3b6228e1a3f59e1756f_720w.jpg)

# 位深度和RGB模式应用

32位色使用ProPhoto RGB，经常摄影，可以使用这个。

8位可以使用sRGB, Adobe RGB

16位可以使用Adobe RGB



