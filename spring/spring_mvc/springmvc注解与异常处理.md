# Spring MVC 的常见注解与异常处理
{docsify-updated}

- [Spring MVC 的常见注解与异常处理](#spring-mvc-的常见注解与异常处理)
  - [异常处理](#异常处理)
    - [统一异常处理](#统一异常处理)
      - [使用 @ExceptionHandler 注解](#使用-exceptionhandler-注解)
      - [实现 HandlerExceptionResolver 接口并注册到 bean 容器](#实现-handlerexceptionresolver-接口并注册到-bean-容器)
      - [使用 @ControllerAdvice+ @ExceptionHandler 注解](#使用-controlleradvice-exceptionhandler-注解)

## 异常处理
如果在请求处理过程中抛出异常， `DispatcherServlet` 会委托一连串 `HandlerExceptionResolver` Bean 来解决异常并提供替代处理方法，通常是错误响应。
下面列出了可用的 `HandlerExceptionResolver` 实现：
1. `SimpleMappingExceptionResolver` : 异常类名称与错误视图名称之间的映射。可用于在浏览器应用程序中呈现错误页面。
2. `DefaultHandlerExceptionResolver` : 解决 Spring MVC 引发的异常，并将它们映射到 HTTP 状态代码。
3. `ResponseStatusExceptionResolver` : 使用 `@ResponseStatus` 注解解决异常，并根据注解中的值将异常映射为 HTTP 状态代码。
4. `ExceptionHandlerExceptionResolver` : 通过调用 `@Controller` 或 `@ControllerAdvice` 类中的 `@ExceptionHandler` 方法来解决异常

可以在 Spring 配置中声明多个 HandlerExceptionResolver Bean，并根据需要设置它们的 order 属性，从而形成一个异常解析器链。顺序属性越高，异常解析器的位置就越靠后。

Spring MVC的 `DispatcherServlet.properties` 会自动为一些默认异常、`@ResponseStatus` 注释的异常以及 `@ExceptionHandler` 注释的方法配置内置解析器。我们可以自定义或替换该列表。

```property
org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver,\
	org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
	org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver
```

### 统一异常处理
Spring 统一异常处理有 3 种方式，分别为：

1. 使用 @ ExceptionHandler 注解
2. 实现 HandlerExceptionResolver 接口
3. 使用 @controlleradvice 注解

#### 使用 @ExceptionHandler 注解
使用该注解有一个不好的地方就是：进行异常处理的方法必须与出错的方法在同一个Controller里面。使用如下：
```
@Controller      
public class GlobalController {               
 
   /**    
     * 用于处理异常的    
     * @return    
     */      
    @ExceptionHandler({MyException.class})       
    public String exception(MyException e) {       
        System.out.println(e.getMessage());       
        e.printStackTrace();       
        return "exception";       
    }       
 
    @RequestMapping("test")       
    public void test() {       
        throw new MyException("出错了！");       
    }                    
```
这种方式最大的缺陷就是不能全局控制异常。每个类都要写一遍。但是这种方法对普通的 Controller 和 RestController 都可以使用。

#### 实现 HandlerExceptionResolver 接口并注册到 bean 容器
这种方式可以进行全局的异常控制，但是只对普通的 Controller 有效，对于 RestController 无效。
```
@Component
public class UnifiedExceptionResolver implements HandlerExceptionResolver {
    @Override
    public ModelAndView resolveException(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, Exception e) {

        e.printStackTrace();
        ModelAndView modelAndView = new ModelAndView();
        modelAndView.addObject("exception",e);
        return modelAndView;
    }
}
```
同时使用 HandlerExceptionResolver 和 @ExceptionHandler 注解时，@ExceptionHandler 会覆盖 HandlerExceptionResolver 。

#### 使用 @ControllerAdvice+ @ExceptionHandler 注解
上文说到 @ ExceptionHandler 需要进行异常处理的方法必须与出错的方法在同一个Controller里面。那么当代码加入了 @ControllerAdvice，则不需要必须在同一个 controller 中了。这也是 Spring 3.2 带来的新特性。从名字上可以看出大体意思是控制器增强。 也就是说，@controlleradvice + @ ExceptionHandler 也可以实现全局的异常捕捉，请确保此WebExceptionHandle 类能被扫描到并装载进 Spring 容器中。
```
@ControllerAdvice
@ResponseBody
public class WebExceptionHandler {
    private static Logger logger = LoggerFactory.getLogger(WebExceptionHandle.class);
    /**
     * 400 - Bad Request
     */
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ExceptionHandler(HttpMessageNotReadableException.class)
    public ServiceResponse handleHttpMessageNotReadableException(HttpMessageNotReadableException e) {
        logger.error("参数解析失败", e);
        return ServiceResponseHandle.failed("could_not_read_json");
    }
    
     /**
     * 405 - Method Not Allowed
     */
    @ResponseStatus(HttpStatus.METHOD_NOT_ALLOWED)
    @ExceptionHandler()
    public ServiceResponse handleHttpRequestMethodNotSupportedException(HttpRequestMethodNotSupportedException e) {
        logger.error("不支持当前请求方法", e);
        return ServiceResponseHandle.failed("request_method_not_supported");
    }


    /**
     * 500 - Internal Server Error
     */
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    @ExceptionHandler(Exception.class)
    public ServiceResponse handleException(Exception e) {
        if (e instanceof BusinessException){
            return ServiceResponseHandle.failed("BUSINESS_ERROR", e.getMessage());
        }
        
        logger.error("服务运行异常", e);
        e.printStackTrace();
        return ServiceResponseHandle.failed("server_error");
    }
}
```
如果 `@ExceptionHandler` 注解中未声明要处理的异常类型，则默认为参数列表中的异常类型。参见上面的 405 异常处理。
