# DispatcherServlet 的注册配置与上下文层级
{docsify-updated}
> https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet/container-config.html
> https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet.html

## Servlet 注册与配置
与其他 `Servlet` 一样， `DispatcherServlet` 需要根据 `Servlet` 规范，通过 Java 配置或 `web.xml` 进行声明和映射。随后， `DispatcherServlet` 会利用 Spring 配置来发现其所需的代理组件，以实现请求映射、视图解析、异常处理等功能。

`WebApplicationInitializer` 是 Spring MVC 提供的一个接口，它能确保您的实现被检测到，并自动用于初始化任何 Servlet 3 容器。名为 `AbstractDispatcherServletInitializer` 的 `WebApplicationInitializer` 抽象基类实现，通过重写方法来指定 Servlet 映射和 `DispatcherServlet` 配置的位置，从而使 `DispatcherServlet` 的注册变得更加简单。
```
public class MyWebApplicationInitializer implements WebApplicationInitializer {

	@Override
	public void onStartup(ServletContext servletContext) {

		// Load Spring web application configuration
		AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
		context.register(AppConfig.class);

		// Use XML to config ApplicationContext
		XmlWebApplicationContext appContext = new XmlWebApplicationContext();
		appContext.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");

		// Create and register the DispatcherServlet
		DispatcherServlet servlet = new DispatcherServlet(context);
		ServletRegistration.Dynamic registration = servletContext.addServlet("app", servlet);
		registration.setLoadOnStartup(1);
		registration.addMapping("/app/*");
	}
}
```
除了直接使用 `ServletContext` API 之外，您还可以继承 `AbstractAnnotationConfigDispatcherServletInitializer` 并重写特定方法。

## 上下文的层级关系及DispatcherServlet的配置
`DispatcherServlet` 需要一个 `WebApplicationContext` （它是普通 `ApplicationContext` 的扩展）来处理自身的配置。 `WebApplicationContext` 与 `ServletContext` 及其关联的 `Servlet` 之间都是通过引用关联在一起的。它还与 `ServletContext` 绑定，因此应用程序若需访问 `WebApplicationContext` ，可通过 `RequestContextUtils` 的静态方法进行查找。

对于许多应用程序而言，使用单个 `WebApplicationContext` 既简单且足够了。但是某些特定场景下，也可以构建上下文层次结构，其中一个根 `WebApplicationContext` 被多个 `DispatcherServlet` （或其他 `Servlet` ）实例共享，每个实例又都有自己独有的子 `WebApplicationContext` 配置。

根 `WebApplicationContext` 通常包含基础架构 Bean，例如需要在多个 `Servlet` 实例之间共享的数据存储库和业务服务。这些 `Bean` 实际上会被继承，并且可以在特定于 `Servlet` 的子 `WebApplicationContext` 中被重写（即重新声明），该子 `WebApplicationContext` 通常包含特定于给定 `Servlet` 的本地 Bean。下图展示了这种关系：

<center><img src="pics/context-hierarchy.png" width="30%"></center>

可以通过下述配置来配置上下文层级：
```
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

	@Override
	protected Class<?>[] getRootConfigClasses() {
		return new Class<?>[] { RootConfig.class }; //配置 Root 上下文
	}

	@Override
	protected Class<?>[] getServletConfigClasses() {
		return new Class<?>[] { App1Config.class }; //配置 Servlet 上下文
	}

	@Override
	protected String[] getServletMappings() {
		return new String[] { "/app1/*" };
	}

	@Override
	protected Filter[] getServletFilters() {
		return new Filter[] {
			new HiddenHttpMethodFilter(), new CharacterEncodingFilter() };
	}
}
```
