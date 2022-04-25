---

title: Spring Boot读取配置文件的三种方式
date: 2022-04-18 15:17:00
categories: 
   - [Spring Boot] 
   - [Java]
toc: true
---



读取application.properties、application.yml、application*.yaml等文件中的配置信息

<!--more-->

application*.properties样例配置文件内容

```
server.port=8000
spring.profiles.active=test

com.test.name=java development
com.test.age=18
```

###### 使用 @Value 注解

@Value是读取单个信息最简便的方式

```
    
public class HelloController {
	@Value("${com.test}")
    private String name;
    
    @Value("${com.test}")
    private String age;
}
```

###### 使用 `Environment`

使用@Autowired注解注入Environment配置

使用environment.getProperty方法获取配置数据

```
@RestController
@ApiOperation("Hello")
public class HelloController {

    @Autowired
    private Environment environment;

    @GetMapping("/hello22")
    public String hello3(@RequestParam(value = "name", defaultValue = "World") String name) {
        environment.getProperty("com.test.name");
        return String.format("Hello %s!", name);
    }
}
```

###### 使用@ConfigurationProperties 注解

@ConfigurationProperties 注解获取配置一般用于获取对象值

1.编写配置模型

@Component  让Springboot识别bean

@ConfigurationProperties 用于识别配置文件

prefix = "com.test" 对象前缀

```
@Component
@Data
@ConfigurationProperties(prefix = "com.test")
public class TestProperties {

    @Getter
    @Setter
    private String name;

    @Getter
    @Setter
    private int age;
}
```

2.使用@Autowired自动注入该对象

```
@RestController
@ApiOperation("Hello")
public class HelloController {

    @Autowired
    private TestProperties testProperties;
}
```

