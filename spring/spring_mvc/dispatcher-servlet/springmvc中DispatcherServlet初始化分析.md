# Spring MVC 中 DispatcherServlet 
{docsify-updated}

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

于是可以看出， `DispatcherServlet` 初始化了 `ApplicationContext` 。

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

找一个 `LocaleResolver` 的例子看一下：
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


