#  Springboot 中MVC 的启动与配置原理
{docsify-updated}

SpringMVC 的启动与配置根本上包括两个部分：
1. 将 `DispatcherServlet` 注册、配置到 Servlet 容器中
2. 配置 MVC 本身（ `HanMadlerpping、HandlerAdapter、HandlerExceptionResolver、ViewResolver`等）

## 注册 DispatcherServlet
Sprinboot 使用代码编程的方式启动内置的 Servlet 容器，通过 `TomcatServletWebServerFactory/JettyServletWebServerFactory/UndertowServletWebServerFactory` 等类实现。Springboot 启动内置 Servlet 的具体过程如下：
1. 创建一个继承自 `ServletWebServerApplicationContext` 的 `AnnotationConfigServletWebServerApplicationContext` 
2. 在 `ServletWebServerApplicationContext#createWebServer()` 方法中创建内置的 servlet 容器，使用 `ServletWebServerFactory#getWebServer(ServletContextInitializer... initializers)` 工厂接口方法来创建，可以传入 `ServletContextInitializer` 来配置 servlet。
    <img src="pics/springboot-embed.png" alt="" />

	`ServletWebServerApplicationContext` 负责创建并配置好 `WebServer` 实例，比如创建一个 Tomcat 服务器并且注册 `DispactherServlet` 到容器中。  
	`WebServerStartStopLifecycle` 负责启动 `WebServer` （tomcat/jetty/undertow)。
	
	```java
	public WebServer getWebServer(ServletContextInitializer... initializers) {
		if (this.disableMBeanRegistry) {
			Registry.disableRegistry();
		}
		Tomcat tomcat = new Tomcat();
		File baseDir = (this.baseDirectory != null) ? this.baseDirectory : createTempDir("tomcat");
		tomcat.setBaseDir(baseDir.getAbsolutePath());
		for (LifecycleListener listener : this.serverLifecycleListeners) {
			tomcat.getServer().addLifecycleListener(listener);
		}
		Connector connector = new Connector(this.protocol);
		connector.setThrowOnFailure(true);
		tomcat.getService().addConnector(connector);
		customizeConnector(connector);
		tomcat.setConnector(connector);
		tomcat.getHost().setAutoDeploy(false);
		configureEngine(tomcat.getEngine());
		for (Connector additionalConnector : this.additionalTomcatConnectors) {
			tomcat.getService().addConnector(additionalConnector);
		}
		prepareContext(tomcat.getHost(), initializers); //在这里执行 ServletContextInitializer 的方法
		return getTomcatWebServer(tomcat);
	}

	private org.springframework.boot.web.servlet.ServletContextInitializer getSelfInitializer() {
		return this::selfInitialize; //注意这里只是放回的方法引用，并没有真正执行方法，执行是在Tomcat启动之后执行的
	}

	private void selfInitialize(ServletContext servletContext) throws ServletException {
		prepareWebApplicationContext(servletContext);
		registerApplicationScope(servletContext);
		WebApplicationContextUtils.registerEnvironmentBeans(getBeanFactory(), servletContext);
		for (ServletContextInitializer beans : getServletContextInitializerBeans()) {
			beans.onStartup(servletContext);
		}
	}
	```
	在 `selfInitialize` 方法中，会去遍历调用 `ServletContextInitializer#onStartup(ServletContext servletContext)` 方法。


3. 在 `ServletContextInitializer` <- `RegistrationBean`<-`DynamicRegistrationBean`<-`ServletRegistrationBean`<-`DispatcherServletRegistrationBean` 的继承体系下，只要我们声明 `DispatcherServletRegistrationBean` 或者 其他的 `RegistrationBean` 类型，springboot 就会帮我们注册到 servlet 容器。
4. springboot 的 `DispatcherServletAutoConfiguration#DispatcherServletRegistrationConfiguration#dispatcherServletRegistration(DispatcherServlet dispatcherServlet,WebMvcProperties webMvcProperties, ObjectProvider<MultipartConfigElement> multipartConfig)`方法就声明了`DispatcherServletRegistrationBean`，springboot 正是用这种方法实现了 `DispatcherServlet` 在容器中的注册的。

通过以上分析，如果我们想注册除了 `DispatcherServlet` 以外的自定义 servlet ，只要声明一个 `ServletRegistrationBean` 的 bean 即可。类似的，`FilterRegistrationBean` 可以注册自定义的 `Filter` 。 如果自己实现 `Filetr` 接口又想使用 Spring 容器功能，springboot 提供了方便的 `DelegatingFilterProxyRegistrationBean` 类型，我们只要自定义一个 `DelegatingFilterProxyRegistrationBean` 类型的 bean 即可。

### 注册多个 DispatcherServlet
1. `spring.mvc.servlet.register=false` 禁用默认的 `DispatcherServlet` 自动注册
2. 自定义注册 `DispatcherServlet` bean:
```
@Configuration
public class DispatcherConfig {

    @Bean
    public DispatcherServlet apiDispatcherServlet() {
        DispatcherServlet ds = new DispatcherServlet();
    	ds.setApplicationContext(apiContext());
    	return ds;
    }

	AnnotationConfigWebApplicationContext apiContext() {
		AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
		context.scan("com.example.api");
		return context;
	}

    @Bean
    public ServletRegistrationBean apiServletBean() {
        ServletRegistrationBean registration = new ServletRegistrationBean(apiDispatcherServlet(), "/api/*");
        registration.setName("apiDispatcherServlet");
        registration.setLoadOnStartup(1);
        return registration;
    }

    @Bean
    public DispatcherServlet adminDispatcherServlet() {
        DispatcherServlet ds = new DispatcherServlet();
    	ds.setApplicationContext(adminContext());
    	return ds;
    }

	AnnotationConfigWebApplicationContext adminContext() {
		AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
		context.scan("com.example.admin");
		return context;
	}

    @Bean
    public ServletRegistrationBean adminServletBean() {
        ServletRegistrationBean registration = new ServletRegistrationBean(adminDispatcherServlet(), "/admin/*");
        registration.setName("adminDispatcherServlet");
        registration.setLoadOnStartup(1);
        return registration;
    }
}
```
3. 上边的例子将 DispatcherServlet 的 context 也进行了区分，如果不区分就会共用一个 context 。

## 配置 MVC
在第二步中，使用Spring MVC，应用程序可以声明处理请求所需的特殊 Bean 类型（`HanMadlerpping、HandlerAdapter、HandlerExceptionResolver、ViewResolver`等）。 `DispatcherServlet` 在初始化时（Servlet 的 init()方法调用时），会在 `WebApplicationContext` 中检查这些特殊的 Bean。如果有就会将其配置到相应的属性中，如果没有匹配的 Bean 类型，它就会使用classpath下的 `DispatcherServlet.properties` 中列出的默认类型配置。

如果只使用 SpringMVC + 独立Servlet容器部署的方式，启动原理[见此](spring的启动.md)，这种模式需要将代码部署到容器中才能工作，并且需要编写代码启用 `DispatcherServlet` 。

### WebMvcConfigurer
在Java配置中，可以使用 `@EnableWebMvc` 注解并实现 `WebMvcConfigurer` 接口来启用MVC配置，，如下例所示：

```
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
	// Implement configuration methods...
}


@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(DelegatingWebMvcConfiguration.class)
public @interface EnableWebMvc {
}
```

`@EnableWebMvc` 注解的功能大多是通过 `DelegatingWebMvcConfiguration` 配置类来实现的，它继承自 `WebMvcConfigurationSupport`，默认会注入所有的 `WebMvcConfigurer` 。
```
@Configuration(proxyBeanMethods = false)
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {

	private final WebMvcConfigurerComposite configurers = new WebMvcConfigurerComposite();


	@Autowired(required = false)
	public void setConfigurers(List<WebMvcConfigurer> configurers) {
		if (!CollectionUtils.isEmpty(configurers)) {
			this.configurers.addWebMvcConfigurers(configurers);
		}
	}
....
}
```
在父类 `WebMvcConfigurationSupport` 中有很多声明了 MVC 相关的 bean 的 `@Bean` 方法 ，这些 bean 会注入到 `DispatcherServlet` 以配置 MVC 相关的功能。这些声明Bean的方法中都留了扩展点方法，子类重写这些方法就可以自定义地配置相关 bean 的属性和行为。 `DelegatingWebMvcConfiguration` 就是重写了这些 `configureXXX`/`addXXX` 的扩展方法，这些方法最终会去调用注入的 `WebMvcConfigurer` 相关的方法。这就是为什么我们只要写一个实现 `WebMvcConfigurer` 接口的类并注入到容器中，就能起到配置作用的原因。

`WebMvcConfigurer` 接口中的一些方法：

+ `default void configurePathMatch(PathMatchConfigurer configurer) {} ` ：自定义Path匹配逻辑
+ `default void configureContentNegotiation(ContentNegotiationConfigurer configurer) {}` ：自定义内容协商
+ `default void configureAsyncSupport(AsyncSupportConfigurer configurer) {}` ：自定义异步支持配置
+ `default void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {}`: 
+ `default void addFormatters(FormatterRegistry registry) {}` ：添加自定义格式化器
+ `default void addInterceptors(InterceptorRegistry registry) {}` ：添加 Interceptor
+ `default void addResourceHandlers(ResourceHandlerRegistry registry) {}` ：配置静态资源处理器
+ `default void addCorsMappings(CorsRegistry registry) {}` ：添加 CORS 跨域配置
+ `default void addViewControllers(ViewControllerRegistry registry) {}` ：配置视图控制器
+ `default void configureViewResolvers(ViewResolverRegistry registry) {}` ：配置视图解析器 ViewResolver 
+ `default void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {}` ：配置HandlerMethodArgumentResolver
+ `default void addReturnValueHandlers(List<HandlerMethodReturnValueHandler> handlers) {}` ：配置返回值处理器
+ `default void configureMessageConverters(List<HttpMessageConverter<?>> converters) {}` ：配置 HttpMessageConverter
+ `default void extendMessageConverters(List<HttpMessageConverter<?>> converters) {}` ：配置扩展 HttpMessageConverter
+ `default void configureHandlerExceptionResolvers(List<HandlerExceptionResolver> resolvers) {}` ：配置 HandlerExceptionResolver
+ `default void extendHandlerExceptionResolvers(List<HandlerExceptionResolver> resolvers) {}` ：配置扩展异常解析器
+ `Validator getValidator()`: 配置验证器

## Springboot 的配置
Springboot 中使用 `ServletWebServerFactoryAutoConfiguration/DispatcherServletAutoConfiguration/WebMvcAutoConfiguration` 等共同实现了全自动化的配置。

1. `ServletWebServerFactoryAutoConfiguration`  
   引入了创建Servlet容器的工厂bean。

2. `DispatcherServletAutoConfiguration`  
   实现了 `DispatcherServlet` 的在Servlet容器中的注册与简单配置。

3. `WebMvcAutoConfiguration`  
   	实现了 MVC 的各个组件的配置。并且也是导入上述两个配置的导入入口类。  
	在 `WebMvcAutoConfiguration` 中定义了 `EnableWebMvcConfiguration` 内部类，这个类实现了和 `@EnableWebMvc` 相同的功能。
	```
	@AutoConfiguration(after = { DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class,
			ValidationAutoConfiguration.class })
	@ConditionalOnWebApplication(type = Type.SERVLET)
	@ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })
	@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
	@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
	public class WebMvcAutoConfiguration{
		....

		// Configuration equivalent to @EnableWebMvc.
		@Configuration(proxyBeanMethods = false)
		@EnableConfigurationProperties(WebProperties.class)
		public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration implements ResourceLoaderAware{...}
	}
	```
	这就是为什么在 Springboot 中没有使用 `@EnableWebMvc` 注解，依然能启用 mvc 功能的原因。

### tomcat 容器的配置
```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
	<exclusions>
		<!-- Exclude the Tomcat dependency -->
		<exclusion>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-tomcat</artifactId>
		</exclusion>
	</exclusions>
</dependency>
<!-- Use Jetty instead -->
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```
1. tomcat: 2024-04-29 13:41:25.078 [main] INFO  com.demo.gjyw.UserCenterApp  - Started UserCenterApp in 50.119 seconds (JVM running for 51.346)
2. un4dertow: 2024-04-29 17:42:40.493 [main] INFO  com.demo.gjyw.UserCenterApp  - Started UserCenterApp in 32.094 seconds (JVM running for 33.4)
3. jetty: 2024-04-29 17:43:50.493 [main] INFO  com.demo.gjyw.UserCenterApp  - Started UserCenterApp in 32.223 seconds (JVM running for 33.654) 

### Springboot 内置容器配置
> https://docs.spring.io/spring-boot/docs/2.0.9.RELEASE/reference/html/howto-embedded-web-servers.html

默认配置如下（`org.springframework.boot.autoconfigure.web.ServerProperties`）：
```
maxConnections - 最大连接数 -----> 8192
maxThreads-最大工作线程数（最大并发请求数） ---> 200
minSpareThreads-预创建的空闲线程数 -----> 10
acceptCount-等待队列长度（连接请求排队数）-------> 100

```

可以通过以下配置项修改：
```
server:
  tomcat:
    maxConnections: 100   # 同时允许100个TCP连接
    acceptCount: 5        # 超出maxConnections时，最多再排队5个
    threads:
      max: 5              # 同时能处理的请求线程只有 5 个
      minSpare: 3
  http2: enabled          # 启用 http2
  ssl:
	keyStore: classpath:spring.keystore
	keyStorePassword: 123456
```
按照上述配置假设同时有500个并发请求过来：
+ 前 100 个请求 会被 Tomcat 接受建立连接（TCP accepted）
+ 第 101 ~ 105 个请求 会进入 acceptCount = 5 的队列等待
+ 从第 106 个请求开始，Tomcat 直接拒绝连接（客户端可能看到：Connection refused、Read timeout 或 RST）

所以只有前 105 个请求成功建立连接，剩下 395 个会立刻失败


如果希望使用编程式的方式对Web服务器进行配置，Spring Boot则 提供了如下两种方式:  
+ 定义一个实现 `WebServerFactoryCustomizer` 接口的Bean实例。
+ 直接在容器中配置一个自定义的 `ConfigurableServletWebServerFactory` ，它负责创建Web服务器。