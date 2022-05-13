---

title: SpringBoot集成Swagger3
date: 2022-04-18 15:17:00
categories: 
   - [Spring Boot] 
toc: true
---



<!--more-->

##### pom文件添加swagger3依赖

```
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-boot-starter</artifactId>
    <version>3.0.0</version>
</dependency>
```

##### 添加配置启动类

```
@Configuration
@EnableOpenApi
@EnableWebMvc
public class Swagger3Config {
    @Bean
    public Docket docket(){
        return new Docket(DocumentationType.OAS_30)
                .apiInfo(apiInfo())
                // 是否开启swagger
                .enable(true)
                .select()
                // apis： 添加swagger接口提取范围
                .apis(RequestHandlerSelectors.basePackage("com.example.spring_demo.web"))
                // 指定路径处理，PathSelectors.any()代表不过滤任何路径
                .paths(PathSelectors.any())
                .build();
    }


    @SuppressWarnings("all")
    private ApiInfo apiInfo(){ // 项目相关信息
        return new ApiInfoBuilder()
                .title("Swagger3测试文档")
                .description("文档描述信息")
                .contact(new Contact("deserts", "#", "1111111@qq.com"))
                .version("1.0")
                .build();
    }
}
```

##### 添加modal

```
@Component
@Data
@ApiModel(description = "modal描述")
@ConfigurationProperties(prefix = "com.test")
public class TestProperties {

    @Getter
    @Setter
    @ApiModelProperty("name描述")
    private String name;

    @Getter
    @Setter
    private int age;
}
```

##### 编写接口文档----即开启spring整合swagger配置

```
@RestController
@Component
@RequestMapping("/api")
@Api(tags = "配置")
public class HelloController  implements InitializingBean {

    @GetMapping("/hello22")
    @ApiOperation(value = "ApiOperation  value", tags = "ApiOperation tags")
    public void hello() throws Exception {
        
    }
     @PostMapping("/hello")
    public void hellod(@RequestBody TestProperties testProperties) throws Exception {

    }
}

```



Swagger3 基本使用：

- @Api 描述类/接口的主要用途
- @ApiOperation 描述方法用途
- @ApiImplicitParam 描述方法的参数
- @ApiImplicitParams 描述方法的参数(Multi-Params)
- @ApiIgnore 忽略某类/方法/参数的文档



访问地址：http://localhost:port/swagger-ui/index.html#/

springfox:http://springfox.github.io/springfox/docs/current/#springfox-samples

