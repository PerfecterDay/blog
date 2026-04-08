# DispatcherServlet 处理请求的流程
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet/sequence.html

## DispatcherServlet 的工作流程
`DispatcherServlet` 处理请求的过程如下：
+ `WebApplicationContext` 会被搜索并绑定到 `HttpSerletRequest` 请求对象中的 `attributes`（`ConcurrentHashMap`类型） 属性中，作为后续处理流程中 `controller` 和其组件可以使用的属性。它默认绑定到 `DispatcherServlet.WEB_APPLICATION_CONTEXT_ATTRIBUTE` key 上。
+ `LocaleResolver` 绑定到请求中，以便在后续流程中处理请求（渲染视图、准备数据等）时解析要使用的本地 locale 。如果不需要 locale 解析，就不需要 `LocaleResolver`
+ theme resolver 绑定到请求中，以便让视图等组件决定使用。如果不使用 themes ，可以忽略它。
+ 如果指定了 multipart file resolver ，则会检查请求中是否是 multiparts 类型。如果是 multiparts 类型的请求，则会将请求封装为 `MultipartHttpServletRequest` ，以便后续处理。
+ 搜索匹配的 `handler` 。如果找到了 `handler` ，就会运行与 `handler` 相关的执行链（ `preprocessors` , `postprocessors` , and `controllers` ），为渲染准备模型。另外，对于注解 `controller` ，可以直接渲染响应，而不是返回视图。
+ 如果返回了 `model` ，就会渲染视图。如果没有返回模型（可能是由于预处理器或后处理器拦截了请求，也可能是出于安全原因），则不会渲染视图。

在 `WebApplicationContext` 中声明的 `HandlerExceptionResolver` Bean 用于处理请求处理过程中抛出的异常。这些异常处理程序允许自定义处理异常的逻辑。

若要支持 HTTP 缓存，处理程序可以使用 `WebRequest` 的 `checkNotModified` 方法，并结合《控制器 HTTP 缓存》中所述的带注解控制器的其他选项。

下图展示了 `DispatcherServlet` 绑定一些组件到 `HttpSerletRequest` 上：
<center><img src="pics/request-attribute.png" width="50%"></center>