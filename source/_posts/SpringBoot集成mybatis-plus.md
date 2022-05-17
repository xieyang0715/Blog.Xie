---

title: SpringBoot集成mybatis-plus
date: 2022-05-17 16:08:00
categories: 
   - [Spring Boot] 
toc: true
---

[MyBatis-Plus (opens new window)](https://github.com/baomidou/mybatis-plus)（简称 MP）是一个 [MyBatis (opens new window)](https://www.mybatis.org/mybatis-3/)的增强工具，在 MyBatis 的基础上只做增强不做改变，为简化开发、提高效率而生。

<!--more-->

##### 引入依赖包

mybatis-plus

```
<!--引入 mybatis-spring-boot-starter 的依赖-->
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>3.5.1</version>
        </dependency>
```

如果没有添加过数据库及springboot web包，还需要引入

```
		<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!-- https://mvnrepository.com/artifact/mysql/mysql-connector-java -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.22</version>
        </dependency>
```

##### 添加配置文件

```
spring:
  datasource:
    url: jdbc:mysql://${DB_HOST:****}:${DB_PORT:3306}/${DB_NAME:test}?serverTimezone=Asia/Shanghai&characterEncoding=utf8&useSSL=false&allowPublicKeyRetrieval=true
    password: ****
    hikari:
      driver-class-name: com.mysql.cj.jdbc.Driver
    username: ****
mybatis-plus:
  configuration:
    #输出打印SQL
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
```



##### 编写实体类

```
@Data
@Accessors(chain = true)
@TableName("account")
public class Account {

    @TableId(value = "id", type = IdType.AUTO)
    private Integer Id;
    private String Name;
    private String Tel;
    private String Pwd;
    @TableField("IsDelete")
    private Integer IsDelete;
    @TableField("CreateTime")
    private Date CreateTime;
    @TableField("CreateAccountId")
    private Integer CreateAccountId;
}
```

注解：

- @Accessors		

  ​		loombok注解，chain中文含义是链式的，设置为true，则setter方法返回当前对象

  ​    	luent的中文含义是流畅的，设置为true，则getter和setter方法的方法名都是基础属性名，且setter方法返回当前对象

  ​		prefix的中文含义是前缀，用于生成getter和setter方法的字段名会忽视指定前缀（遵守驼峰命名）

- @TableName	表名注解，标识实体类对应的表

- @TableId		   主键注解，type = IdType.AUTO表示主键自增；type = IdType.ASSIGN_UUID表示分配 UUID,主键类型为 String

- @TableField	  字段注解

[更多注解]: https://baomidou.com/pages/223848/

##### 编写mapper包接口

```
package com.example.spring_demo.service.accountServices.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.example.spring_demo.domain.Dto.Account;
import org.apache.ibatis.annotations.Mapper;
import org.springframework.stereotype.Repository;

/**
 * @author: xy
 * @Description:
 * @Date 2022/5/16 15:44
 */
@Repository
public interface AccountServiceMapper extends BaseMapper<Account> {
    Account getAccountById(Integer id);
}
```

BaseMapper ：提供CRUD的 接口

##### 添加sql配置

- 在resources下编写对应xml

  ```
  <?xml version="1.0" encoding="UTF-8"?>
  <!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
  <mapper namespace="com.example.spring_demo.service.accountServices.mapper.AccountServiceMapper">
      <select id="getAccountById" resultType="com.example.spring_demo.domain.Dto.Account" parameterType="integer">
          select * from account where id = #{id}
      </select>
  </mapper>
  ```

- 在mapper包接口以注解的方式直接写sql

```
@Repository
public interface AccountServiceMapper extends BaseMapper<Account> {
	@Select("select * from account where id = #{id}")
    Account getAccountById(Integer id);
}
```

##### 添加 `@MapperScan` 注解，扫描 Mapper 文件夹

```
@SpringBootApplication
@RestController
//@EnableOpenApi //Enable open api 3.0.3 spec
@MapperScan("com.example.spring_demo.service.accountServices.mapper")
public class SpringDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringDemoApplication.class, args);
    }

    @GetMapping("/hello")
    public String hello(@RequestParam(value = "name", defaultValue = "World") String name) {
        return String.format("Hello %s!", name);
    }

}
```

##### 使用

```
@Service
@RequiredArgsConstructor
public class AccountServiceImpl implements AccountService {

    private final AccountServiceMapper accountServiceMapper;
    @Override
    public Account getAccountById(Integer id) {
        Account account = accountServiceMapper.getAccountById(id);
        return account;
    }
    @Override
    public Account getById(Integer id) {
        Account account = accountServiceMapper.selectById(id);
        return account;
    }
    @Override
    public void createAccount(Account account) {
        accountServiceMapper.insert(account);
    }
}
```

@Service：自动注册到spring容器
@RequiredArgsConstructor：loombok的注解，生成带有必须参数的构造函数。这里用来注入AccountServiceMapper，和@Autowired功能一致。



[loombok注解]: https://projectlombok.org/features/all
[mybatis-plus文档]: https://baomidou.com/

