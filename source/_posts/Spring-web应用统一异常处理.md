---

title: Spring web应用统一异常处理
date: 2022-04-15 15:47:00
categories: 
   - [Spring] 
   - [Java] 
   - [异常处理]
toc: true
---



web应用自定义异常处理

<!--more-->

步骤：

###### 方式一、以页面返回自定义异常信息：

1.创建全局异常处理类：GlobalExceptionHandler

@ControllerAdvice 定义统⼀的异常处理类

@ExceptionHandler ⽤来定义函数针对的异常类型

ModelAndView将异常信息映射到error页面

```
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(value    =  Exception.class)
    public ModelAndView defaultErrorHandler(HttpServletRequest req, Exception e) throws  Exception  {
        ModelAndView mav = new ModelAndView();
        //将Exception对象和请求URL映射到 error.html 中
        mav.addObject("exception",e);
        mav.addObject("url",   req.getRequestURL());
        mav.setViewName("error");
        return mav;
    }
}
```

2.实现 error.html ⻚⾯展示：在 templates ⽬录下创建 error.html ，将请求的对象（URL和

Exception对象的message输出）。

```
<!DOCTYPE   html>
<html> <head lang="en">
    <meta charset="UTF-8"  />
    <title>统⼀异常处理</title>
</head> <body>
<h1>Error  Handler</h1>
<div th:text="${url}"></div>
<div th:text="${exception.message}"></div>
</body>
</html>
```

3.定义异常处理器接口测试

```
@GetMapping("/err")
public void err() throws Exception {
    throw new Exception("Im error");
}
```



###### 方式二、返回json格式

1.定义异常json返回模型

```
@Data
public class ErrorInfo<T> {
    @Getter
    @Setter
    private    Integer    code = 400;
    @Getter
    @Setter
    private    String message;
    @Getter
    @Setter
    private    String url;
    @Getter
    @Setter
    private    T data;
}
```

2.定义全局错误处理类

@ResponseBody 处理函数return的内容转换为JSON格式

```
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(value    =  Exception.class)
    @ResponseBody
    public ErrorInfo<String> jsonErrorHandler(HttpServletRequest req, Exception    e) throws  Exception  {
        ErrorInfo<String> errorInfo = new ErrorInfo<>();
        errorInfo.setMessage(e.getMessage());
        errorInfo.setData("这里是错误信息");
        return errorInfo;
    }
}
```

3.定义定义异常处理器接口测试

```
@GetMapping("/err")
public void err() throws Exception {
    throw new Exception("Im error");
}
```