---

title: Spring Boot导入Spring配置
date: 2022-04-18 15:17:00
categories: 
   - [Spring Boot] 
   - Java 
toc: true
---


Spring Boot 中是不包含任何的 Spring 配置文件的，即使我们手动添加 Spring 配置文件到项目中，也不会被识别。

<!--more-->

Spring Boot 为了我们提供了以下 2 种方式来导入 Spring 配置：

1. 使用 @ImportResource 注解加载 Spring 配置文件
2. 使用全注解方式加载 Spring 配置



##### @ImportResource 导入 Spring 配置文件

在主启动类上使用 @ImportResource 注解可以导入一个或多个 Spring 配置文件，并使其中的内容生效。

###### 编写PersonService接口

```
public interface PersonService {
    public Person getPersonInfo();
}
```

###### 编写PersonService接口实现类PersonServiceImpl 

```
public class PersonServiceImpl implements PersonService {
    @Autowired
    private Person person;
    @Override
    public Person getPersonInfo() {
        return person;
    }
}
```

###### 在该项目的 resources 下添加一个名为 beans.xml 的 Spring 配置文件

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="personService" class="net.biancheng.www.service.impl.PersonServiceImpl"></bean>
</beans>
```

###### 在主启动程序类上使用 @ImportResource 注解，将 Spring 配置文件 beans.xml 加载到项目中

```
//将 beans.xml 加载到项目中
@ImportResource(locations = {"classpath:/beans.xml"})
@SpringBootApplication
public class HelloworldApplication {
    public static void main(String[] args) {
        SpringApplication.run(HelloworldApplication.class, args);
    }
}
```



##### 全注解方式加载 Spring 配置

###### 使用 @Configuration 注解定义配置类，替换 Spring 的配置文件

###### 添加@Bean注解的方法

配置类内部可以包含有一个或多个被 @Bean 注解的方法，这些方法会被 AnnotationConfigApplicationContext 或 AnnotationConfigWebApplicationContext 类扫描

方法的返回值会以组件的形式添加到容器中，组件的 id 就是方法名。

 @Bean 注解的方法，该方法相当于 Spring 配置文件中的 **<bean>** 标签定义的组件

```
@Configuration
public class MyAppConfig {
    /**
     * 与 <bean id="personService" class="PersonServiceImpl"></bean> 等价
     * 该方法返回值以组件的形式添加到容器中
     * 方法名是组件 id（相当于 <bean> 标签的属性 id)
     */
    @Bean
    public PersonService personService() {
        System.out.println("在容器中添加了一个组件:peronService");
        return new PersonServiceImpl();
    }
}
```

