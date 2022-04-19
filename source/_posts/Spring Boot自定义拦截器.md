---

title: Spring Boot自定义拦截器
date: 2022-04-19 15:36:00
categories: 
   - [Spring Boot] 
   - [Java]
toc: true
---


拦截器主要应用于登陆校验、权限验证、乱码解决、性能监控和异常处理等功能上

<!--more-->

在 Spring Boot 项目中，使用拦截器功能通常需要以下 3 步：

1. 定义拦截器；
2. 注册拦截器；
3. 指定拦截规则（如果是拦截所有，静态资源也会被拦截）。
4. 验证拦截器

##### 定义拦截器

定义拦截器只需要创建一个拦截器类，并实现 HandlerInterceptor 接口即可

| 返回值类型 | 方法声明                                                     | 描述                                                         |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| boolean    | preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) | 该方法在控制器处理请求方法前执行，其返回值表示是否中断后续操作，返回 true 表示继续向下执行，返回 false 表示中断后续操作。 |
| void       | postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) | 该方法在控制器处理请求方法调用之后、解析视图之前执行，可以通过此方法对请求域中的模型和视图做进一步修改。 |
| void       | afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) | 该方法在视图渲染结束后执行，可以通过此方法实现资源清理、记录日志信息等工作。 |

```java
/**
 * 定义拦截器
 */
public class ApiRequestInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        //请求校验
        System.out.println("preHandle");
        return HandlerInterceptor.super.preHandle(request, response, handler);
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        //请求校验
        System.out.println("postHandle");
        HandlerInterceptor.super.postHandle(request, response, handler, modelAndView);
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        //请求校验
        System.out.println("afterCompletion");
        HandlerInterceptor.super.afterCompletion(request, response, handler, ex);
    }
}
```



##### 注册拦截器

创建一个实现了 WebMvcConfigurer 接口的配置类（使用了 [@Configuration 注解](http://blog.cdzq.ltd/2022/04/18/Spring%20Boot%E5%AF%BC%E5%85%A5Spring%E9%85%8D%E7%BD%AE/#more)的类），重写 addInterceptors() 方法，并在该方法中调用 registry.addInterceptor() 方法将自定义的拦截器注册到容器中

```
/**
 * 实现 WebMvcConfigurer 接口可以来扩展 SpringMVC 的功能
 * @EnableWebMvc注解  接管SpringMVC
 */
@Configuration
public class MyMvcConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        System.out.println("注册ApiRequestInterceptor拦截器");
        registry.addInterceptor(new ApiRequestInterceptor());
    }
}

```



##### 指定拦截规则

修改拦截器addInterceptors方法，添加拦截规则

```
/**
 * 实现 WebMvcConfigurer 接口可以来扩展 SpringMVC 的功能
 * @EnableWebMvc注解  接管SpringMVC
 */
@Configuration
public class MyMvcConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        System.out.println("注册ApiRequestInterceptor拦截器");
//        registry.addInterceptor(new ApiRequestInterceptor());

        registry.addInterceptor(new ApiRequestInterceptor()).addPathPatterns("/**") //拦截所有请求，包括静态资源文件
                .excludePathPatterns("/", "/login", "/index.html", "/user/login", "/css/**", "/images/**", "/js/**", "/fonts/**"); //放行登录页，登陆操作，静态资源;
    }
}
```

在指定拦截器拦截规则时，调用了两个方法，这两个方法的说明如下：

- addPathPatterns：该方法用于指定拦截路径，例如拦截路径为“/**”，表示拦截所有请求，包括对静态资源的请求。
- excludePathPatterns：该方法用于排除拦截路径，即指定不需要被拦截器拦截的请求。



##### 验证登陆及登陆拦截功能

调用接口查看拦截打印是否输出验证