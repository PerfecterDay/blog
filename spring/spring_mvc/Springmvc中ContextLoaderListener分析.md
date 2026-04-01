# Spring MVC 中ContextLoaderListener分析
{docsify-updated}


典型的Spring mvc项目的 web.xml配置：

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>
            classpath:spring-common-config.xml,
            classpath:spring-budget-config.xml
        </param-value>
    </context-param>
    <listener>  
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>
contextConfigLocation 参数配置了 Spring 上下文的 bean 配置，我们来看看，这个配置是如何生效的。

# ContextLoaderListener
首先，看 `ContextLoaderListener` 的定义：

    public class ContextLoaderListener extends ContextLoader implements ServletContextListener{}
`ServletContextListener` 的 `contextInitialized` 方法的注释：
  
     Notification that the web application initialization process is starting.
     All ServletContextListeners are notified of context initialization before
     any filter or servlet in the web application is initialized.
就是说 `contextInitialized` 方法会在所有的其他 `filter` 或者 `servlet` 被初始化之前执行。

`ContextLoaderListener` 实现了 `ServletContextListener` 接口：

    public void contextInitialized(ServletContextEvent event) {
        //调用父类 ContextLoader 的 initWebApplicationContext 方法
		initWebApplicationContext(event.getServletContext());
	}

`ContextLoader` 的 `initWebApplicationContext` 方法：

    public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
            if (this.context == null) {
                //生成 ConfigurableWebApplicationContext 对象
				this.context = createWebApplicationContext(servletContext);
			}
			if (this.context instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
				if (!cwac.isActive()) {
				    if (cwac.getParent() == null) {
					    ApplicationContext parent = loadParentContext(servletContext);
						cwac.setParent(parent);
					}
                    //根据 web.xml 中的配置，配置 ConfigurableWebApplicationContext
					configureAndRefreshWebApplicationContext(cwac, servletContext);
				}
			}
            //将 ConfigurableWebApplicationContext 对象设置到 servletContext 的 ‘WebApplicationContext.root’属性上
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
			return this.context;
		}
    }

再看 `configureAndRefreshWebApplicationContext` 方法：

    protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
		wac.setServletContext(sc);
        //获取 ServletContext 中的 contextConfigLocation 参数配置
		String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
		if (configLocationParam != null) {
			wac.setConfigLocation(configLocationParam);
		}

		ConfigurableEnvironment env = wac.getEnvironment();
		if (env instanceof ConfigurableWebEnvironment) {
			((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
		}

		customizeContext(sc, wac);
		wac.refresh();
	}
可以看出在 web.xml 中配置的 `contextConfigLocation` 参数，最终被设置到上下文中。

详细的可以看 `XmlWebApplicationContext` 的 `loadBeanDefinitions` 方法：

    protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws IOException {
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			for (String configLocation : configLocations) {
				reader.loadBeanDefinitions(configLocation);
			}
		}
	}