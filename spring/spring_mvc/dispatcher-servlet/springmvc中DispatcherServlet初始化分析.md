# Spring MVC 中 DispatcherServlet 
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet/container-config.html
> https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet.html

## Servlet 注册与配置
与其他 `Servlet` 一样， `DispatcherServlet` 需要根据 `Servlet` 规范，通过 Java 配置或 `web.xml` 进行声明和映射。随后， `DispatcherServlet` 会利用 Spring 配置来发现其所需的代理组件，以实现请求映射、视图解析、异常处理等功能。

```
public class MyWebApplicationInitializer implements WebApplicationInitializer {

	@Override
	public void onStartup(ServletContext servletContext) {

		// Load Spring web application configuration
		AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
		context.register(AppConfig.class);

		// Create and register the DispatcherServlet
		DispatcherServlet servlet = new DispatcherServlet(context);
		ServletRegistration.Dynamic registration = servletContext.addServlet("app", servlet);
		registration.setLoadOnStartup(1);
		registration.addMapping("/app/*");
	}
}
```
除了直接使用 `ServletContext` API 之外，您还可以继承 `AbstractAnnotationConfigDispatcherServletInitializer` 并重写特定方法。

## 上下文的继承关系及DispatcherServlet的配置
`DispatcherServlet` 需要一个 `WebApplicationContext` （它是普通 `ApplicationContext` 的扩展）来处理自身的配置。 `WebApplicationContext` 与 `ServletContext` 及其关联的 `Servlet` 之间都是通过引用关联在一起的。它还与 `ServletContext` 绑定，因此应用程序若需访问 `WebApplicationContext` ，可通过 `RequestContextUtils` 的静态方法进行查找。

对于许多应用程序而言，使用单个 `WebApplicationContext` 既简单且足够了。但是某些特定场景下，也可以构建上下文层次结构，其中一个根 `WebApplicationContext` 被多个 `DispatcherServlet` （或其他 `Servlet` ）实例共享，每个实例又都有自己独有的子 `WebApplicationContext` 配置。

根 `WebApplicationContext` 通常包含基础架构 Bean，例如需要在多个 `Servlet` 实例之间共享的数据存储库和业务服务。这些 `Bean` 实际上会被继承，并且可以在特定于 `Servlet` 的子 `WebApplicationContext` 中被重写（即重新声明），该子 `WebApplicationContext` 通常包含特定于给定 `Servlet` 的本地 Bean。下图展示了这种关系：

<center><img src="pics/context-hierarchy.png" width="30%"></center>

可以通过下述配置来配置上下文层级：
```
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

	@Override
	protected Class<?>[] getRootConfigClasses() {
		return new Class<?>[] { RootConfig.class };
	}

	@Override
	protected Class<?>[] getServletConfigClasses() {
		return new Class<?>[] { App1Config.class };
	}

	@Override
	protected String[] getServletMappings() {
		return new String[] { "/app1/*" };
	}
}
```

## DispatcherServlet 的初始化
首先看 `DispatcherServlet` 继承关系：
`DispatcherServlet` -> `FrameworkServlet` --> `HttpServletBean` -> `HttpServlet`

找到 `HttpServletBean` 的 `init` 方法：

    @Override
	public final void init() throws ServletException {
		// Set bean properties from init parameters.
		PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
		if (!pvs.isEmpty()) {
            BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
            ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
            bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
            initBeanWrapper(bw);
            bw.setPropertyValues(pvs, true);
		}
        // Let subclasses do whatever initialization they like.
		initServletBean();
	}

跟到 `FrameworkServlet` 的 `initServletBean` 方法：

    @Override
	protected final void initServletBean() throws ServletException {
        this.webApplicationContext = initWebApplicationContext();
        initFrameworkServlet();	
	}
继续跟到 `initWebApplicationContext` 方法：

    protected WebApplicationContext initWebApplicationContext() {
		WebApplicationContext rootContext =
				WebApplicationContextUtils.getWebApplicationContext(getServletContext());
		WebApplicationContext wac = null;
		if (this.webApplicationContext != null) {
			wac = this.webApplicationContext;
			if (wac instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
				if (!cwac.isActive()) {
						if (cwac.getParent() == null) {
						cwac.setParent(rootContext);
					}
					configureAndRefreshWebApplicationContext(cwac);
				}
			}
		}
		if (wac == null) {
			wac = findWebApplicationContext();
		}
		if (wac == null) {
			wac = createWebApplicationContext(rootContext);
		}
		if (!this.refreshEventReceived) {
			onRefresh(wac);
		}
		return wac;
	}
一路跟到 `FrameworkServlet` 的 `createWebApplicationContext` 方法：

    protected WebApplicationContext createWebApplicationContext(@Nullable ApplicationContext parent) {
		ConfigurableWebApplicationContext wac =
				(ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
		wac.setEnvironment(getEnvironment());
		//设置 parent 上下文
		wac.setParent(parent);
		String configLocation = getContextConfigLocation();
		if (configLocation != null) {
			wac.setConfigLocation(configLocation);
		}
		configureAndRefreshWebApplicationContext(wac);
		return wac;
	}

跟着看 `configureAndRefreshWebApplicationContext` 方法：

    protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) {
		wac.setServletContext(getServletContext());
		wac.setServletConfig(getServletConfig());
		wac.setNamespace(getNamespace());
		wac.addApplicationListener(new SourceFilteringListener(wac, new ContextRefreshListener()));
		ConfigurableEnvironment env = wac.getEnvironment();
		if (env instanceof ConfigurableWebEnvironment) {
			((ConfigurableWebEnvironment) env).initPropertySources(getServletContext(), getServletConfig());
		}
		postProcessWebApplicationContext(wac);
		applyInitializers(wac);
		wac.refresh();
	}

于是可以看出， `DispatcherServlet` 初始化了 ApplicationContext。

再看 `DispatcherServlet` 的 `onRefresh` 方法：
```
protected void onRefresh(ApplicationContext context) {
	initStrategies(context);
}

/**
	* Initialize the strategy objects that this servlet uses.
	* <p>May be overridden in subclasses in order to initialize further strategy objects.
	*/
protected void initStrategies(ApplicationContext context) {
	initMultipartResolver(context);
	initLocaleResolver(context);
	initThemeResolver(context);
	initHandlerMappings(context);
	initHandlerAdapters(context);
	initHandlerExceptionResolvers(context);
	initRequestToViewNameTranslator(context);
	initViewResolvers(context);
	initFlashMapManager(context);
}
```

找一个 LocaleResolver 的例子看一下：
```
private void initLocaleResolver(ApplicationContext context) {
	try {
		this.localeResolver = context.getBean(LOCALE_RESOLVER_BEAN_NAME, LocaleResolver.class);
		if (logger.isTraceEnabled()) {
			logger.trace("Detected " + this.localeResolver);
		}
		else if (logger.isDebugEnabled()) {
			logger.debug("Detected " + this.localeResolver.getClass().getSimpleName());
		}
	}
	catch (NoSuchBeanDefinitionException ex) {
		// We need to use the default.
		this.localeResolver = getDefaultStrategy(context, LocaleResolver.class);
		if (logger.isTraceEnabled()) {
			logger.trace("No LocaleResolver '" + LOCALE_RESOLVER_BEAN_NAME +
					"': using default [" + this.localeResolver.getClass().getSimpleName() + "]");
		}
	}
}
```
初始化的时候，首先会查找相应类型的 bean ，如果找不到就会使用默认的配置。

默认的初始化会配置一些默认的 `LocaleResolver/ThemeResolver/HandlerMapping/HandlerAdapter` 等组件，由 `DEFAULT_STRATEGIES_PATH` 指定，默认在 `DispatcherServlet.properties` 文件中配置：
```
private static final String DEFAULT_STRATEGIES_PATH = "DispatcherServlet.properties";

org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver

org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver

org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping,\
	org.springframework.web.servlet.function.support.RouterFunctionMapping

org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
	org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter,\
	org.springframework.web.servlet.function.support.HandlerFunctionAdapter

org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver,\
	org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
	org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver

org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator

org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver

org.springframework.web.servlet.FlashMapManager=org.springframework.web.servlet.support.SessionFlashMapManager
```

## DispatcherServlet 的工作流程
`DispatcherServlet` 处理请求的过程如下：
+ `WebApplicationContext` 会被搜索并绑定到 `HttpSerletRequest` 请求对象中的 `attributes`（`ConcurrentHashMap`类型） 属性中，作为流程中 controller 和其组件可以使用的属性。它默认绑定到 `DispatcherServlet.WEB_APPLICATION_CONTEXT_ATTRIBUTE` key 上。
+ locale resolver 绑定到请求中，以便在处理请求（渲染视图、准备数据等）时解析要使用的本地 locale 。如果不需要 locale 解析，就不需要 locale resolver。
+ theme resolver 绑定到请求中，以便让视图等组件决定使用。如果不使用 themes ，可以忽略它。
+ 如果指定了 multipart file resolver ，则会检查请求中是否有多部分文件。如果发现多部分文件，则会将请求封装为 `MultipartHttpServletRequest` ，以便后续处理。
+ 搜索匹配的 handler 。如果找到了 handler ，就会运行与 handler 相关的执行链（前置处理程序、后置处理程序和 handler ），为渲染准备模型。另外，对于注解 controller ，可以直接渲染响应，而不是返回视图。
+ 如果返回了模型，就会渲染视图。如果没有返回模型（可能是由于预处理器或后处理器拦截了请求，也可能是出于安全原因），则不会渲染视图。

下图展示了 `DispatcherServlet` 绑定一些组件到 `HttpSerletRequest` 上：
<center><img src="pics/request-attribute.png" width="50%"></center>


