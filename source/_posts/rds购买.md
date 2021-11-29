---
title: rds购买
date: 2020-10-21 03:23:52
tags:
toc: true
---



根据上篇文章完成ecs购买: http://liangcheng.mykernel.cn/2020/10/20/ECS%E8%B4%AD%E4%B9%B0/#more

前台登陆时，后端需要使用数据库

<!--more-->

# 1. 进入控制台

https://homenew.console.aliyun.com

右侧 *号可以将此服务添加至左侧的经常使用界面

![image-20201021112659112](http://myapp.img.mykernel.cn/image-20201021112659112.png)

# 2. 创建实例

![image-20201021112810163](http://myapp.img.mykernel.cn/image-20201021112810163.png)

# 3. 计费方式

实验按量

生产 预付费包年

![image-20201021112840969](http://myapp.img.mykernel.cn/image-20201021112840969.png)

# 4. 地域

跟机器在同一个地域的同一个可用区

![image-20201021113028821](http://myapp.img.mykernel.cn/image-20201021113028821.png)

![image-20201021112939867](http://myapp.img.mykernel.cn/image-20201021112939867.png)

# 5. 类型

mysql 及版本

![image-20201021113004323](http://myapp.img.mykernel.cn/image-20201021113004323.png)

# 6. 系列

基础：单节点

生产：高可用版或PolarDB

高可用：主+备

高性能：polardb主+多备 

![image-20201021113226476](http://myapp.img.mykernel.cn/image-20201021113226476.png)

# 7. 存储类型

![image-20201021113257592](http://myapp.img.mykernel.cn/image-20201021113257592.png)

# 8. 主节点可用区

和ecs同区

![image-20201021113322745](http://myapp.img.mykernel.cn/image-20201021113322745.png)



# 9. 实例规格

实验：最便宜

生产：4C,8G常用

![image-20201021113416949](http://myapp.img.mykernel.cn/image-20201021113416949.png)

# 10. 存储空间

实验：最小

![image-20201021113444017](http://myapp.img.mykernel.cn/image-20201021113444017.png)

# 11. 下一步

![image-20201021114055245](http://myapp.img.mykernel.cn/image-20201021114055245.png)

 ![image-20201021114124262](http://myapp.img.mykernel.cn/image-20201021114124262.png)

# 12. 确认订单

![image-20201021114137867](http://myapp.img.mykernel.cn/image-20201021114137867.png)

![image-20201021114152890](http://myapp.img.mykernel.cn/image-20201021114152890.png)

![image-20201021114204725](http://myapp.img.mykernel.cn/image-20201021114204725.png)

# 13. 控制台

等待变成运行状态

![image-20201021114547110](http://myapp.img.mykernel.cn/image-20201021114547110.png)

![image-20201022133825758](http://myapp.img.mykernel.cn/image-20201022133825758.png)



# 14. 实例

申请外网地址：最好不申请外网地址	

内网地址: rm-8vby7u25zfa13jv1858870.mysql.zhangbei.rds.aliyuncs.com (需要添加ECS内网白名单, 而后使用)

![image-20201022133902438](http://myapp.img.mykernel.cn/image-20201022133902438.png)



右上角配置白名单，加载内网IP



# 15. 添加白名单 

![image-20201022134131392](http://myapp.img.mykernel.cn/image-20201022134131392.png)	![image-20201022134208506](http://myapp.img.mykernel.cn/image-20201022134208506.png)



# 16. 添加root账号

![image-20201022134235913](http://myapp.img.mykernel.cn/image-20201022134235913.png)

![image-20201022134329591](http://myapp.img.mykernel.cn/image-20201022134329591.png)

# 17. 添加库, 并授权账号有此库权限

![image-20201022134355438](http://myapp.img.mykernel.cn/image-20201022134355438.png)![image-20201022134410506](http://myapp.img.mykernel.cn/image-20201022134410506.png)

![image-20201022134434851](http://myapp.img.mykernel.cn/image-20201022134434851.png)

![image-20201022134515250](http://myapp.img.mykernel.cn/image-20201022134515250.png)



# 18. 登陆数据库DMS界面

![image-20201022134537178](http://myapp.img.mykernel.cn/image-20201022134537178.png)

![image-20201022134630384](http://myapp.img.mykernel.cn/image-20201022134630384.png)

![image-20201022134650951](http://myapp.img.mykernel.cn/image-20201022134650951.png)

# 19. 报警添加

cpu，慢日志

sdk