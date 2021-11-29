---
title: "jenkins 迁移"
date: 2021-04-14 17:03:19
tags:
- "jenkins"
---



今天项目需要将一台jenkins从本地迁移到云端，先再本地测试环境部署jenkins测试迁移。

完成结果：迁移前后对业务无影响。

# 迁移方案

先把jenkins所有数据迁移到另一个主机启动，再验证从新主机的jenkins逐一迁移指定业务至新的jenkins主机。验证无误后再进行旧jenkins迁移到云

1. 全量备份，导入一个jenkins
2. 基于恢复的jenkins来测试增量恢复

## 整数据目录迁移

 https://stackoverflow.com/questions/8724939/how-to-move-jenkins-from-one-pc-to-another

Install a fresh Jenkins instance on the new server
Be sure the old and the new Jenkins instances are stopped
Archive all the content of the JENKINS_HOME of the old Jenkins instance
Extract the archive into the new JENKINS_HOME directory
Launch the new Jenkins instance
Do not forget to change documentation/links to your new instance of Jenkins :)
Do not forget to change the owner of the new Jenkins files : chown -R jenkins:jenkins $JENKINS_HOME

## 逐一迁移jenkins

利用python-jenkins, 逐一迁移，[Jenkins官方说明](https://www.jenkins.io/doc/book/using/remote-access-api/) [python-jenkins官方文档](https://python-jenkins.readthedocs.io/)

- [x] plugins
- [x] jobs
- [x] views
- [x] credentials

- [x] nodes



# 测试部署jenkins

## 启动jenkins

```bash
docker run --rm -it -p 9090:8080 -e JENKINS_HOME=/opt/data -v g:/opt/jenkins_data:/opt/data jenkins/jenkins:lts
```

## 配置

配置credential, job, 安装常用插件（build with parameters, git parameter, role-based authorization strategy)

配置job

![image-20210414171539776](http://myapp.img.mykernel.cn/image-20210414171539776.png)

![image-20210414171626599](http://myapp.img.mykernel.cn/image-20210414171626599.png)

![image-20210414171636736](http://myapp.img.mykernel.cn/image-20210414171636736.png)

## 测试构建

![image-20210414171654674](http://myapp.img.mykernel.cn/image-20210414171654674.png)

# 全量迁移

## 停止原容器

```bash
#略
```

## 打包原数据，迁移至新位置

```bash
把数据拷贝到新目录 g:/opt/new_jenkins_data
```

## 基于新位置启动新容器

jenkins的版本需要注意，低了启动后Job读不到中文

```bash
docker run --rm -it -p 9091:8080 -e JENKINS_HOME=/opt/data -v G:/opt/jenkins_backup_2021_04_15:/opt/data jenkins/jenkins:lts
```



## 配置新位置

系统配置

Jenkins Location

Jenkins URL

## 修改jenkins属主和属组

```bash
chown -R jenkins.jenkins /opt/data/
```



# 指定数据迁移

- [x] 升级
- [x] 迁移job, view,安装插件

## 获取api token

![image-20210415133726548](http://myapp.img.mykernel.cn/image-20210415133726548.png)

![image-20210415133733650](http://myapp.img.mykernel.cn/image-20210415133733650.png)

```bash
11e1095b094fc955b3fd87b2cb63c55703
```

> 避免密码泄漏，所以使用此密码串



## 安装插件

获取2个jenkins有哪些插件差距

```python
# encoding: utf-8
"""
@author: songliangcheng
@qq:     2192383945@qq.com
@file:   test.py
@time:   2021-04-16 13:19:41
@version: v1.0.1
@description: 获取新版jenkins少的插件
"""
import jenkins

jenkins_migration_server = jenkins.Jenkins('http://localhost:9091', username='jenkins',
                                           password='11d1111a827a2b5874b046e49c1fae4e44')
docker_server_jenkins = jenkins.Jenkins('http://localhost:9092', username='root',
                                        password='11b85ef56d687fe0f53c94df9a559c35fb')


old_jenkins_plugins_version = { k[0]:"{}".format(v['shortName'],v['version']) for k, v in jenkins_migration_server.get_plugins().items()}
new_jenkins_plugins_version = { k[0]:"{}:{}".format(v['shortName'],v['version']) for k, v in docker_server_jenkins.get_plugins().items()}
plugins_names = [ ov for ok,ov in old_jenkins_plugins_version.items() if ok not in new_jenkins_plugins_version.keys() ]
print(" ".join(plugins_names))
```

shell自动安装

```bash
https://github.com/jenkinsci/plugin-installation-manager-tool
wget https://github.com/jenkinsci/plugin-installation-manager-tool/releases/download/2.9.0/jenkins-plugin-manager-2.9.0.jar

#编辑hudson.model.UpdateCenter.xml，替换镜像：https://updates.jenkins-zh.cn/update-center.json
#重启
export JENKINS_UC=https://updates.jenkins-zh.cn
export JENKINS_UC_DOWNLOAD=https://mirrors.tuna.tsinghua.edu.cn/jenkins

for plugin in additional-identities-plugin aliyun-oss-uploader amazon-ecr authentication-tokens aws-credentials aws-java-sdk dingding-json-pusher dingding-notifications docker-commons external-monitor-job favorite generic-webhook-trigger git-changelog javadoc jenkins-design-language job-import-plugin jquery-detached kubernetes-client-api kubernetes-credentials kubernetes locale mapdb-api maven-plugin mercurial metrics postbuild-task publish-over-ssh publish-over pubsub-light sonar sse-gateway thinBackup variant windows-slaves; do  echo "开始安装$plugin"; java -jar ./jenkins-plugin-manager-2.9.0.jar --plugins "$plugin" --war /usr/lib/jenkins/jenkins.war -d /var/lib/jenkins/plugins; [ $? -eq 0 ]  || echo "$plugin 安装失败" >> /tmp/p.log; echo;done

# --war指定war位置
# -d 指定插件目录


# 读取/tmp/p.log安装失败的插件
[root@izbp1iegoe89wvmbuy7ky1z mnt]# cat /tmp/p.log 
amazon-ecr 安装失败
aws-credentials 安装失败
aws-java-sdk 安装失败
kubernetes-client-api 安装失败
kubernetes-credentials 安装失败
kubernetes 安装失败

wget -P /var/lib/jenkins/plugins/ https://updates.jenkins.io/download/plugins/amazon-ecr/1.6/amazon-ecr.hpi
syst

#重启
systemctl restart jenkins
```

>```bash
>Additional Identities Plugin      用户额外认证  https://plugins.jenkins.io/additional-identities-plugin/
>Aliyun OSS Uploader               作用   -> build -> 上传oss -> ssh插件连接生产，下载。
>	https://gitee.com/libfintech/aliyun-oss-plugin
>Amazon ECR plugin             推送aws docker镜像  
>	https://medium.com/paul-zhao-projects/continuous-delivery-pipeline-for-amazon-ecs-using-jenkins-github-and-amazon-ecr-5714df354d4e
>CloudBees AWS Credentials Plugin
>Amazon Web Services SDK
>Dingding JSON Pusher Plugin
>DingTalk                     钉钉	
>	https://plugins.jenkins.io/dingding-notifications/
>Docker Commons Plugin
>External Monitor Job Type Plugin
>	https://blog.csdn.net/mmh19891113/article/details/108174256
>Favorite 企业添加收藏
>Generic Webhook Trigger Plugin
>
>Git Changelog job配置-构建后支持添加changlog
>
>Javadoc Plugin 该插件允许选择“Publish Javadoc”作为构建后操作，指定要收集Javadoc的目录以及是否希望每次成功构建都保留。
>Kubernetes Client API Plugin
>Kubernetes Credentials Plugin
>Kubernetes plugin
>Locale plugin                                     jenkins默认语言
>MapDB API Plugin
>Maven Integration plugin                            直接添加maven的job  (没有使用)
>Jenkins Mercurial plugin                           与mercurial存储库集成 (没有使用)
>Metrics Plugin                                    指标暴露 (没有使用)
>Post build task                                    匹配不同java日志，执行不同的命令
>Publish Over SSH                                   构建后
>Infrastructure plugin for Publish Over X           
>Jenkins Pub-Sub "light" Bus
>SonarQube Scanner for Jenkins                       集成代码扫描
>Server Sent Events (SSE) Gateway Plugin             Jenkins的网关插件。使用pubsub-light-plugin jenkins-module接收轻量级事件并将其通过SSE转发到浏览器域。
>ThinBackup                                          备份jenkins
>Variant Plugin                                      
>WMI Windows Agents Plugin                           windows slave时，需要使用此插件
>```

## 升级jenkins

```bash
# 停止 
systemctl stop jenkins

#下载小版本最新的war, https://updates.jenkins-ci.org/download/war/
#查看正在使用的jenkins.war文件位置: 系统管理 - 系统信息 - 系统属性 -> executeable-war /usr/share/jenkins/jenkins.war
#替换
mv /usr/lib/jenkins/jenkins.war /usr/lib/jenkins/jenkins.war.20210419
#下载
wget https://updates.jenkins-ci.org/download/war/2.222.4/jenkins.war
#启动
systemctl start jenkins
```



 ## 迁移job, view

先把jobs上传到/opt/jenkins_jobs目录 

```bash
for newdir in $(ls /opt/jenkins_jobs/); do 
        if ls /var/lib/jenkins/jobs | grep "${newdir}$"; then 
                echo "$newdir exists" 
        else 
                cp -a /opt/jenkins_jobs/$newdir /var/lib/jenkins/jobs/
                [ $? -eq 0 ] && echo "cp $newdir to /var/lib/jenkins/jobs/$newdir" && echo "cp $newdir successful" || echo "cp $newdir failure"
                echo
        fi 
done
```

迁移jenkins的views和plugins对比

```python
#!/usr/bin/env python
# Author: songliangcheng
# QQ: 2192383945
# URL: https://python-jenkins.readthedocs.io/en/latest/examples.html
# DESC: 迁移jenkins 9091 -> 9092

import jenkins
import logging


class Config:
    # 迁移至的jenkins
    NEW_JENKINS_HOST = "http://localhost:9092"
    NEW_JENKINS_USER = 'root'
    NEW_JENKINS_PASS = '11b85ef56d687fe0f53c94df9a559c35fb'
    # 被迁移jenkins
    OLD_JENKINS_HOST = "http://localhost:9091"
    OLD_JENKINS_USER = 'jenkins'
    OLD_JENKINS_PASS = '11d1111a827a2b5874b046e49c1fae4e44'


class Jenkins:
    """
    迁移jenkins, view, plugins.
    """

    def __init__(self):
        """
        jenkins_migration_server：被迁移的服务器
        docker_server_jenkins:   新服务器
        """
        self.jenkins_migration_server = jenkins.Jenkins(Config.OLD_JENKINS_HOST, username=Config.OLD_JENKINS_USER,
                                                        password=Config.OLD_JENKINS_PASS)
        self.docker_server_jenkins = jenkins.Jenkins(Config.NEW_JENKINS_HOST, username=Config.NEW_JENKINS_USER,
                                                     password=Config.NEW_JENKINS_PASS)
        self.logger = logging.getLogger(__name__)
        self.logger.setLevel(level=logging.INFO)

        self.console = logging.StreamHandler()
        self.console.setLevel(logging.INFO)
        self.formatter = logging.Formatter(
            '%(asctime)s %(levelname)s  %(name)s %(filename)s %(funcName)s  %(lineno)d: %(message)s')
        self.console.setFormatter(self.formatter)
        self.logger.addHandler(self.console)

    def add_jobs(self):
        """
        添加所有job
        :return: None
        """
        job_names = [i['name'] for i in self.jenkins_migration_server.get_all_jobs()]
        for job in job_names:
            if self.docker_server_jenkins.job_exists(job):
                self.logger.error("{} 已经存在".format(job))
                continue
            try:
                self.docker_server_jenkins.create_job(name=job,
                                                      config_xml=self.jenkins_migration_server.get_job_config(job))
            except Exception as e:
                self.logger.info(e)
                continue
            self.logger.info("{} 已经添加".format(job))

    def add_views(self):
        """
        添加所有视图
        :return: None
        """
        view_names = [i['name'] for i in self.jenkins_migration_server.get_views()]
        for view in view_names:
            if self.docker_server_jenkins.view_exists(view):
                self.logger.error("{} view已经存在".format(view))
                continue
            self.docker_server_jenkins.create_view(view, self.jenkins_migration_server.get_view_config(view))
            self.logger.info("{} view添加ok".format(view))

    def get_plugins(self):
        """
        列表插件
        :return: None
        """
        old_jenkins_plugins_version = {k[0]: "{}".format(v['shortName'], v['version']) for k, v in
                                       self.jenkins_migration_server.get_plugins().items()}
        new_jenkins_plugins_version = {k[0]: "{}:{}".format(v['shortName'], v['version']) for k, v in
                                       self.docker_server_jenkins.get_plugins().items()}
        plugins_names = [ov for ok, ov in old_jenkins_plugins_version.items() if
                         ok not in new_jenkins_plugins_version.keys()]
        self.logger.info(
            """ for plugin in {}; do  echo "开始安装$plugin"; jenkins-plugin-cli --plugins "$plugin"; [ $? -eq 0 ] && echo "安装$plugin完成" || echo "$plugin 安装失败";done""".format(
                " ".join(plugins_names)))


if __name__ == '__main__':
    jk = Jenkins()
    #jk.add_jobs()
    jk.add_views()
    jk.get_plugins()
```

## jenkins测试脚本

```python
# encoding: utf-8
# Author: songliangcheng
# QQ: 2192383945
# DESC: jenkins测试脚本
import re
import jenkins
import logging


class Config:
    # 备份的jenkins
    NEW_JENKINS_HOST = "http://localhost:8081/"
    NEW_JENKINS_USER = 'xxx'
    NEW_JENKINS_PASS = 'xxxxxxx'

class Jenkins:
    """
    迁移jenkins, view, plugins.
    """

    def __init__(self):
        """
        docker_server_jenkins:   备份的jenkins服务器
        """
        self.docker_server_jenkins = jenkins.Jenkins(Config.NEW_JENKINS_HOST, username=Config.NEW_JENKINS_USER,
                                                     password=Config.NEW_JENKINS_PASS)
        self.logger = logging.getLogger(__name__)
        self.logger.setLevel(level=logging.INFO)

        self.console = logging.StreamHandler()
        self.console.setLevel(logging.INFO)
        self.formatter = logging.Formatter(
            '%(asctime)s %(levelname)s  %(name)s %(filename)s %(funcName)s  %(lineno)d: %(message)s')
        self.console.setFormatter(self.formatter)
        self.logger.addHandler(self.console)

    def test(self):
        """
        备份job
        :return: None
        """
        job_names = [i['name'] for i in self.docker_server_jenkins.get_all_jobs()]
        update_job = [ job for job in job_names if re.search("<assignedNode>windows</assignedNode>",self.docker_server_jenkins.get_job_config(job)) ]
        for job in update_job:
            self.logger.info("{} 开始处理".format(job))
            self.logger.info(self.docker_server_jenkins.get_job_config(job))
            a = re.sub(pattern="<assignedNode>windows</assignedNode>",repl="<assignedNode>windows_2016</assignedNode>",string=self.docker_server_jenkins.get_job_config(job))
            self.docker_server_jenkins.reconfig_job(name=job,config_xml=a)
            self.logger.info("{} 处理完成\n".format(job))

if __name__ == '__main__':
    jk = Jenkins()
    jk.test()
```

> 匹配job，替换文本，重新导入

