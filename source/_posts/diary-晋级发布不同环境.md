---
title: 晋级发布不同环境
date: 2021-11-03 11:34:44
tags:
- 个人日记
photos:
- https://docs.microsoft.com/zh-tw/azure/architecture/example-scenario/apps/media/architecture-devops-with-aks.png
---

# 原理
<!--more-->

镜像过程：`test:latest -> gray:latest -> prod:<time>`

构建,推test:latest;            拉测试推gray镜像;         拉灰度, 推生产镜像



# 实现devops

运维：系统统一、配置版本化、不同环境版本一致。 

 研发：统一研发环境，代码版本，流程制度化

> ![image-20211124171920087](http://myapp.img.mykernel.cn/image-20211124171920087.png)

改进：单应用试点，自动化构建3种环境，部署。容器化改造。

# gitlab

分2个仓库，一个放测试、灰度。另一个放生产环境



#  jenkins分为3个任务

将测试和灰度授权给测试、开发

将生产授权给运维



# 实现脚本

## kustomize操作脚本

```bash
#!/bin/bash
#
#********************************************************************
#Author:                SongLiangCheng
#QQ:                    1062670898
#Date:                  2021-07-19
#FileName：             config.sh
#URL:                   http://www.magedu.com
#Description：          update Kubernetes's application :app, namespace,  image, version
#Copyright (C):        2021 All rights reserved
#********************************************************************
cd $(readlink -f $(dirname ${0}))
PROG="${0##*/}"
APP_NAME=ishop-api
NAMESPACE=ishop

while getopts "a:n:i:v:" opt
do
   case $opt in
        a)
        APP_NAME=${OPTARG}
        ;;
        n)
        NAMESPACE=${OPTARG}
        ;;
        i)
        IMAGE=${OPTARG}
        ;;
        v)
        VERSION=${OPTARG}
        ;;
        ?)
				echo "${PROG} [-a 应用名] [-n 名称空间] -i 镜像地址  -v 镜像版本"
        exit 1
        ;;
   esac
done



echo "获取到变量..............."
echo APP_NAME=$APP_NAME
echo NAMESPACE=$NAMESPACE
echo IMAGE=$IMAGE
echo VERSION=$VERSION

echo "开始更新kustomize配置............."
/usr/local/bin/kustomize edit set namespace  $NAMESPACE
/usr/local/bin/kustomize edit set nameprefix ${APP_NAME}-
/usr/local/bin/kustomize edit set label app:${APP_NAME}
/usr/local/bin/kustomize edit set image ikubernetes/myapp=$IMAGE:$VERSION
echo "真正更新镜像................."
/usr/local/bin/kustomize build | kubectl apply -f - --record 
kubectl rollout status deploy -n ishop         ${APP_NAME}-deploy --timeout=600s
```



## 测试

git分支构建 

```bash
version=latest
TESTIMG=demoapp
# build and push
cd CloudAPI/iShopGlobal/
docker build -t $TESTIMG:${version} 
docker login 
docker push $TESTIMG:${version}

# publish
rm -fr configcenter
git clone http://github.com/xxx.git
cd configcenter/test/
APPNAME_LABELNAME=ishop-api
bash config.sh -a $APPNAME_LABELNAME -i $TESTIMG -v ${version}          # 将git配置文件转换成yaml, 并同步到集群

kubectl delete pod -n ishop -l app=$APPNAME_LABELNAME      # 避免latest版本不更新

git add . && git commit -m "${BUILD_USER} update TEST  version: ${version} on jenkins job: ${JOB_NAME} build id:${BUILD_ID}" && git push -u origin main 
cd -
rm -fr configcenter
```

> 1. 构建
> 2. 拉test/gray仓库，通过自带的脚本，将新的镜像和版本，更新到仓库。
> 3. 基于仓库生成yaml文件，然后通过kustomze命令更新k8s的资源对象
> 4. 我们测试和灰度是固定的label, 所以避免k8s没有检测到变量，就删除镜像，触发拉镜像。
> 5. 将本地修改的最新镜像信息同步到仓库，并添加commit.

## 灰度

参数化构建 ，避免用户误点

```bash
TESTIMG=demoapp
GRAYIMG=gray-demoapp # 灰度的镜像名不一样了

docker login 
docker pull $TESTIMG
docker tag  $TESTIMG $GRAYIMG
docker push $GRAYIMG

# 发布
rm -fr configcenter
git clone http://github.com/xxx.git
cd configcenter/gray/
APPNAME_LABELNAME=grap-ishop-api-deploy
bash config.sh -a $APPNAME_LABELNAME -i $GRAYIMG -v latest # 更新配置

kubectl delete pod -n ishop -l app=$APPNAME_LABELNAME      # 避免latest版本不更新
git add . && git commit -m "${BUILD_USER} update GRAY version: ${version} on jenkins job: ${JOB_NAME} build id:${BUILD_ID}" && git push -u origin main 
cd -
rm -fr configcenter
```

> 1. 拉测试镜像
> 2. 更名为灰度镜像，推送
> 3. 拉test/gray仓库，通过自带的脚本，将新的灰度镜像和版本，更新到仓库。
> 4. kustomize | kubectl 将灰度配置同步到k8s, 避免latest不检测，删除pod
> 5. 提交commit, 表示已经更新。 

## 生产

参数化构建 ，避免用户误点

```bash
version=$(date +"%Y%m%d%H%M")
GRAYIMG=gray-demoapp 
PRODIMG=prod-demoapp # 生产的镜像名
docker login 
docker pull $GRAYIMG
docker tag  $GRAYIMG $PRODIMG:$version      
docker push $PRODIMG:$version

# 发布
rm -fr prod-configcenter
git clone 
cd prod-configcenter/
APPNAME_LABELNAME=ishop-api
bash config.sh  -a $APPNAME_LABELNAME -i $PRODIMG -v $version # 生产是独立的k8s集群
git add . && git commit -m "${BUILD_USER} update PROD version: ${version} on jenkins job: ${JOB_NAME} build id:${BUILD_ID}" && git push -u origin main 
cd -
rm -fr prod-configcenter
```

> 1. 生产环境，拉灰度，打tag基于时间，而不是latest
> 2. 将配置同步到生产仓库，并滚动更新来升级镜像。
> 3. 将commit提交到git.
