---

title: SpringBoot集成Swagger2(整合token)
date: 2022-04-18 17:00:00
categories: 
   - [Spring Boot] 
toc: true
---



<!--more-->

##### pom文件添加swagger2依赖

```
<!-- https://mvnrepository.com/artifact/io.springfox/springfox-swagger2 -->
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger2</artifactId>
            <version>2.9.2</version>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
            <version>2.9.2</version>
        </dependency>
```

##### 添加配置启动类

```
@Configuration //声明该类为配置类
@EnableSwagger2 //声明启动Swagger2
public class Swagger2Config {
    @Bean
    public Docket docket(){
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(new ApiInfoBuilder()
                        .title("Hello")
                        .version("V1.0.1")
                        .contact(new Contact("马化腾","www.baidu.com","99999@百度.com"))
                        .build())
                .groupName("hello-controller")
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.example.spring_demo.web"))
                .paths(PathSelectors.any())
                .build()
                //添加登陆认证
                .securitySchemes(securitySchemes())
                .securityContexts(securityContexts());
    }
    
    private List<ApiKey> securitySchemes() {
        //设置请求头信息
        List<ApiKey> result = new ArrayList<>();
        ApiKey apiKey = new ApiKey("Authorization", "Authorization", "header");
        result.add(apiKey);
        return result;
    }

    private List<SecurityContext> securityContexts() {
        //设置需要登录认证的路径
        List<SecurityContext> result = new ArrayList<>();
        result.add(getContextByPath("^(?!/auth).*$"));// /*/.*
        return result;
    }

    private SecurityContext getContextByPath(String pathRegex){
        return SecurityContext.builder()
                .securityReferences(defaultAuth())
                .forPaths(PathSelectors.regex(pathRegex))
                .build();
    }

    //设置为全局调用，在测试界面输入一个Authorization值，后面测试的接口都将添加Authorization
    private List<SecurityReference> defaultAuth() {
        List<SecurityReference> result = new ArrayList<>();
        AuthorizationScope authorizationScope = new AuthorizationScope("global", "accessEverything");
        AuthorizationScope[] authorizationScopes = new AuthorizationScope[1];
        authorizationScopes[0] = authorizationScope;
        result.add(new SecurityReference("Authorization", authorizationScopes));
        return result;
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
    
     @GetMapping("/testToken")
    @ApiOperation(value = "swagger中token测试")
    public String token(HttpServletRequest httpServletRequest){
        //返回请求头中Authorization的值
        if(httpServletRequest.getHeader("Authorization") == null){
            return "Authorization is null";
        }
        return httpServletRequest.getHeader("Authorization");
    }
}

```



Swagger3 基本使用：

- @Api 描述类/接口的主要用途
- @ApiOperation 描述方法用途
- @ApiImplicitParam 描述方法的参数
- @ApiImplicitParams 描述方法的参数(Multi-Params)
- @ApiIgnore 忽略某类/方法/参数的文档



访问地址：http://localhost:port/swagger-ui.html#/

springfox:http://springfox.github.io/springfox/docs/current/#springfox-samples