---

title: Spring Boot Profile（多环境配置）
date: 2022-04-18 15:44:00
categories: 
   - [Spring Boot] 
   - [Java] 
toc: true
---


Profile 为在不同环境下使用不同的配置提供了支持，我们可以通过激活、指定参数等方式快速切换环境

<!--more-->

##### 多 Profile 文件方式

- 支持三种文件

1. .properties 文件
2. .yml 文件
3. .yaml 文件

- 格式

  {profile} 一般为各个环境的名称或简称，例如 dev、test 和 prod 等等

  application-{profile}.properties

```
#默认端口号
server.port=8080
#激活指定的profile
spring.profiles.active=prod
```

​	application-{profile}.yml/yaml

```
#默认配置
server:
  port: 8080
#切换配置
spring:
  profiles:
    active: dev #激活开发环境配置
```



- 加载顺序

1. Spring Boot 启动时会扫描以下 5 个位置的 application.properties 或 apllication.yml 文件，并将它们作为 Spring boot 的默认配置文件。

2. 如果配置文件名相同的文件分布在不容的目录下

   优先级从高到低如下：

   ```
   1. file:./config/
   2. file:./config/*/
   3. file:./
   4. classpath:/config/
   5. classpath:/
   ```

   注：file: 指当前项目根目录；classpath: 指当前项目的类路径，即 resources 目录

3. 如果配置文件在同一路径下

yml  >  yaml  >  properties

优先级由低到高，后加载的配置文件会覆盖之前加载的配置文件

- 激活配置

1. 配置文件中配置

   ```
   #激活指定的profile
   spring.profiles.active=prod
   ```

2. 命令行配置

   ```
    java	-jar	xxx.jar	--spring.profiles.active=test
   ```



##### 多 Profile 文档块模式

在 YAML 配置文件中，可以使用“---”把配置文件分割成了多个文档块，因此我们可以在不同的文档块中针对不同的环境进行不同的配置，并在第一个文档块内对配置进行切换

```
#默认配置
server:
  port: 8080
#切换配置
spring:
  profiles:
    active: test
---
#开发环境
server:
  port: 8081
spring:
  config:
    activate:
      on-profile: dev
```



备注：

[spring boot常见应用程序属性](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#appendix.application-properties)