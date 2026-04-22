# 自动注册的类型
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet/special-bean-types.html

`DispatcherServlet` 会将请求处理和响应渲染的任务委托给特殊的 `Bean` 。这里所说的“特殊 Bean”是指实现了框架约定的接口或拓展了约定的类型、由 Spring 管理的对象实例。这些 Bean 通常自带内置契约，但是我们也可以自定义其属性，并对其进行扩展或替换。

+ `HandlerMapping`
+ `HandlerAdapter`
+ `HandlerExceptionResolver`
+ `ViewResolver`
+ `LocaleResolver, LocaleContextResolver`
+ `MultipartResolver`
+ `FlashMapManager`

应用程序可以声明“特殊 Bean 类型”中列出的、用于处理请求的基础设施 Bean。 `DispatcherServlet` 会检查 `WebApplicationContext` 中是否存在一个特殊 Bean。如果存在就是用这个 Bean ，如果没有匹配的 Bean 类型，则会根据 `DispatcherServlet.properties` 中的配置的默认类型实例化一个。

