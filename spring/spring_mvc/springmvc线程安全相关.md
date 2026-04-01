# Springmvc 中的线程安全

每个Http请求到达服务时，都会绑定到一个线程处理，当我们的服务中涉及到多线程时，需要注意线程安全。

## HttpServletRequest注入
当我们注入一个 `HttpServletRequest` 到一个单例 bean 时，Spring 通过 `ServletRequestAttributes` 类型来封装 `HttpServletRequest` 对象（以及当前 `HttpSession` 对象）。该封装对象绑定到 `ThreadLocal` ，可通过调用静态方法 `RequestContextHolder.currentRequestAttributes()` 获得。

`ServletRequestAttributes` 提供了 `getRequest()` 方法来获取当前 `HttpServletRequest` 对象，`getSession()` 方法来获取当前 `HttpSession` 对象 ，以及其他方法来获取存储在两个作用域中的属性。下面的代码虽然有点难看，但可以在应用程序适当的地方获取当前请求对象：
```
HttpServletRequest curRequest =
   ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest()；
```
请注意，`RequestContextHolder.currentRequestAttributes()` 方法返回的是一个接口，需要类型转换到实现该接口的 `ServletRequestAttributes`。

值得注意的是，假如上述代码运行在另一个非当前请求处理的线程中（通常是在代码中启动了另一个线程）可能会报错：
```java
@RestController
public class Application {
	@Autowire
	Box box;

	@GetMapping("/test")
    public String test(HttpServletRequest request){
        CompletableFuture.supplyAsync(()->{
            System.out.println(this);
            System.out.println(box.request.getMethod());
            return "";
        });
        System.out.println(box.request.getMethod());
        return "Hello";
    }
}

@Component
public class Box {
    @Autowired
    HttpServletRequest request;
}
```

当 Box 中注入 `HttpServletRequest` 对象时，Spring 会自动对其进行代理增强：
<center><img src="pics/springmvc-box-1.png" alt=""></center>

如图所示，注入的 request 实际上是一个 `Proxy` 代理对象，并且使用 `AutowireUtils$ObjectFactoryDelegatingInvocationHandler` 进行增强。方法中的 request 是一个 `RequestFacade` 对象。运行上述代码并访问这个 test 路径，会报以下错误：  
`No thread-bound request found: Are you referring to request attributes outside of an actual web request, or processing a request outside of the originally receiving thread? If you are actually operating within a web request and still receive this message, your code is probably running outside of DispatcherServlet: In this case, use RequestContextListener or RequestContextFilter to expose the current request.`

原因就是在调用 `box.request.getMethod()` 方法时，代理回去调用 `ObjectFactoryDelegatingInvocationHandler` 的 `invoke` 方法：
<center><img src="pics/springmvc-box-2.png" width="50%"></center>

`invoke` 调用反射尝试在真正的 request 对象上调用对应方法，获取真正 request 对象的方法是 `this.objectFactory.getObject()` ，`objectFactory` 实际上是 `WebApplicationContextUtils$RequestObjectFactory`类型：
```java
private static class RequestObjectFactory implements ObjectFactory<ServletRequest>, Serializable {
        
	public ServletRequest getObject() {
		return WebApplicationContextUtils.currentRequestAttributes().getRequest();
	}
}

private static ServletRequestAttributes currentRequestAttributes() {
	RequestAttributes requestAttr = RequestContextHolder.currentRequestAttributes();
	if (!(requestAttr instanceof ServletRequestAttributes)) {
		throw new IllegalStateException("Current request is not a servlet request");
	} else {
		return (ServletRequestAttributes)requestAttr;
	}
}
```

与前文呼应了，因为此时处理的线程并不是处理请求的线程，所以获取不到请求对象，抛出异常。