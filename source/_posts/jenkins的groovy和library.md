---
title: jenkins的groovy和library
date: 2021-05-17 13:57:05
tags:
- jenkins
- groovy
---



# 前言

jenkinsfile有大量冗余代码可以使用library



<!--more-->
# jenkins加载library?

## 准备git仓库

```bash
install -dv src/org/devops vars

# vim src/org/devops/tools.groovy
package org.devops

// 钉印
def printmes(content) {
    println(content)
}



# vim vars/hello.groovy
def call() {
    println("hello")
}

# vim vars/devops.groovy
def hello(content) {
	println("devops hello")
}
```

http克隆地址：http://git.mykernel.cn/songliangcheng/jenkinslib.git

## jenkins加入仓库

manage jenkins -> configure jenkins

![image-20210517140035895](http://myapp.img.mykernel.cn/image-20210517140035895.png)

记录库名 `jenkinslib`



## jenkinsfile调用 

准备VSCODE 编辑器，插件如下

![image-20210517140132143](http://myapp.img.mykernel.cn/image-20210517140132143.png)

准备一个文件 `desk.groovy` 这样就可以识别groovy的语法

```groovy
#!groovy

@Library('jenkinslib') _        //引入jenkins定义的共享库

def tools = new org.devops.tools() // 实例化 src/org/devops/tool.groovy 文件，引用方法：tools.printmes()

hello()                            // 直接引用vars/hello.groovy 脚本, hello() 直接调用内部的call()方法
pipeline {
    agent {
        kubernetes {
            yaml '''\
            apiVersion: v1
            kind: Pod
            metadata:
              namespace: jenkins
              labels:
                some-label: some-label-value
            spec:
              nodeSelector:
                kubernetes.io/os: linux
              serviceAccountName: jenkins-build-sa
              containers:
              - name: docker
                image: docker:19.03.12
                command:
                - cat
                tty: true
                env:
                - name: LANG
                  value: zh_CN.UTF-8
                - name: TZ
                  value: Asia/Shanghai
                securityContext:
                  privileged: true
                volumeMounts:
                  - name: dind-storage
                    mountPath: /var/run/docker.sock
              volumes:
              - name: dind-storage
                hostPath:
                  path: /var/run/docker.sock

            '''.stripIndent()
        }
    }

    stages {
        stage('test') {
            steps {
                script {
                    println('1')        // groovy内置函数
                    tools.printmes('2') // pipeline同级生成的tools,
                    devops.hello('3')  // 在vars目录中定义的模块名中的函数，可以在jenkinsfile中直接 调用，不需要以上方法引用。
                }
            }
        }
    }
}
```



# groovy语法

## 安装groovy解释器

### 图形

安装jdk: https://www.oracle.com/hk/java/technologies/javase/javase-jdk8-downloads.html

安装groovy: https://groovy.apache.org/download.html

![image-20210517144258275](http://myapp.img.mykernel.cn/image-20210517144258275.png)

### 命令

```bash
docker run --rm --name groovy -v g:/data/groovydata  -dit groovy bash

docker exec -it groovy bash

groovy@4345ad2e1130:~$ groovysh
Groovy Shell (3.0.8, JVM: 1.8.0_282)
Type ':help' or ':h' for help.
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
groovy:000> 
groovy:000> 

```



## 字符类型

字符串表示： 单引号、双引号、三引号

```groovy
groovy:000> "hello world"
===> hello world
groovy:000> 'hello world'
===> hello world
groovy:000> '''hello world'''
===> hello world
groovy:000> """hello world"""
===> hello world

# 变量：" ", """ """
# 固定的值： ''
groovy:000> name = "abc"
===> abc
groovy:000> "my name is ${name}"
===> my name is abc
groovy:000> 'my name is ${name}'
===> my name is ${name}

```



contains() 是否包含特定内容, 返回bool

```groovy
groovy:000> "devopsstestops".contains("ops")
===> true
groovy:000> "devopsstestops".contains("user")
===> false
groovy:000> 

```

size() length() 数量大小长度

```groovy
groovy:000> "devopsstestops".size()
===> 14
groovy:000> "devopsstestops".length()
===> 14

```

toString() 转换成string类型

indexOf() 元素的索引

endsWith() 指定字符结尾

```groovy
groovy:000> "devopsstestops".endsWith("tops")
===> true
groovy:000> "devopsstestops".endsWith("top")
===> false
groovy:000> 
```

minus() plus() 去掉、增加字符串

```bash
groovy:000> "dev" + "ops"
===> devops
groovy:000> "devops" - "ops"
===> dev
groovy:000> 
groovy:000> "dev".plus("ops")
===> devops
groovy:000> "devops".minus("ops")
===> dev
```



reverse() 反向排序

substring(1,2) 字符串从指定索引开始的子字符串

toUpperCase() toLowerCase() 字符串大小写转换

```groovy
groovy:000> "devops".toUpperCase()
===> DEVOPS
groovy:000> "devops".toUpperCase().toLowerCase()
===> devops
groovy:000> 
```

split() 字符串分割 默认空格分割 返回列表

```groovy
groovy:000> "host01,host02,host03".split()
===> [host01,host02,host03]
```

## 列表类型

符号: []

方法： + - += -= 元素增加或减少

add()  << 添加元素

```groovy
groovy:000> []
===> []
groovy:000> [1,2,3,4] + 4
===> [1, 2, 3, 4, 4]
groovy:000> [1,2,3,4] << 14
===> [1, 2, 3, 4, 14]
groovy:000> result = [1,2,3,4]
===> [1, 2, 3, 4]
groovy:000> result.add(14)
===> true
groovy:000> result
===> [1, 2, 3, 4, 14]
```



isEmpty() 判断是否为空

intersect([2,3] disjoint([1]))  取交集、判断是否有交集

flatten() 合并嵌套的列表

unique() 去重

```groovy
groovy:000> [2,2,1,2].unique()
===> [2, 1]
```

reverse() sort() 反转 升序

count() 元素个数

join() 将元素按照参数链接

```groovy
groovy:000> [2,2,1,2].join("-")
===> 2-2-1-2

```



sum() min() max() 求和 最小值 最大值

contains() 包含特定元素

remove(2) removeAll()

each{} 遍历

```groovy
groovy:000> [2,2,1,2].each{
groovy:001> println it }
2
2
1
2
===> [2, 2, 1, 2]

```

## 字典

字符: [:]

```groovy
groovy:000> ["key":1]
===> [key:1]
groovy:000> ["key":1]["key"]
===> 1
```

size() map大小

```groovy
groovy:000> ["key":1].size()
===> 1

```

['key']        get() 获取value

keySet() 所有key变成列表

```groovy
groovy:000> ["key":2,'3':4,"5":6].keySet()
===> [key, 3, 5]

```

values() value 变成列表

```groovy
groovy:000> ["key":2,'3':4,"5":6].values()
===> [2, 4, 6]
groovy:000> 
```

加元素

```groo
groovy:000> ["key":2,'3':4,"5":6] + [7:8]
===> [key:2, 3:4, 5:6, 7:8]

```

remove('a') 删除元素 (k-v)

each{} 遍历map

containValue() containKey() 是否包含value, 是否包含key

isEmpty() 是否为空

## 语句

### if

```groovy
if (condition) {
    
} else if (condition) {
    
} else {
    
}
```

### switch

```groovy
buildtype="maven"
switch("${buildtype}") {
    case "maven":
    	println("maven")
    	;;
    	break
    case "gradle":
    	println("gradle")
    	;;
    	break
    default:
        println("default")
    	;;
    	break
}
```

### for

```groo
lang = ['java','python','groovy']
for ( l in lang) {
    println("lang is ${l}")
}
```

### while

```groo
while(1==1) {
	
}
```

## 函数

### 无参

```groovy
def funcname() {
    println("hello")
}

funcname()
```

### 有参

```groovy
def funcname(info) {
    println("hello ${info}")
    return 1
}

a = funcname("ops")
println(a)
```

## 正则 表达式



# jenkins方法

```groovy
withCredentials() 凭据

checkout 检出源代码

publishHTML 单元测试、自动化测试生成HTML报告，展示workspace中的报告

input 选择

wrap builduser, 获取构建项目的用户ID, EMAIL

httpRequest, 访问接口

readjson


junit 收集报告 
```

