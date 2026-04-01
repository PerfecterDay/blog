# Spring 的启动
{docsify-updated}

- [Spring 的启动](#spring-的启动)
	- [Spring Framework 的启动](#spring-framework-的启动)
		- [部署描述符启动](#部署描述符启动)
		- [初始化器中使用编程的方式启动 Spring](#初始化器中使用编程的方式启动-spring)


## Spring Framework 的启动
Spring Framework 是另一个容器，它可以运行在任何 Java SE和 Java EE  容器中，并作为应用程序的运行时环境。另外，如同计算机或者jvm的那个样例一样， Spring 必须被启动并且需要得到如何运行它所包含的应用程序的指令。
配置和启动 Spring Framework 是两个不同的任务，并且相互独立，都可以通过多种方式实现。配置告诉 Spring 如何运行它所包含的应用程序时，启动进程将启动 Spring Framework并将配置指令传递给它。

+ **在 Java SE 中，只有一种方式启动 Spring Framework：通过应用程序的 main 方法以编程的方式启动。**
+ **在 Java EE 应用程序中，有两种选择：**
  1. **可以使用 XML 创建部署描述符启动 Spring**
  2. **也可以在 `javax.servlet.ServletContainerInitializer` 中通过编程的方式启动。**

### 部署描述符启动
传统的 Spring Framework 应用程序总是使用 Java EE 的部署描述符启动。配置文件中至少包含一个 `DispatcherServlet` 的实例，然后以 `contextConfigLocation` 初始化参数的形式为它提供配置文件。也可以包含多个 `DispatcherServlet` 实例。另外，一般还会配置 `ContextLoaderListener` 实例加上 `contextConfigLocation`的上下文参数。其典型的配置如下：
```xml
<web-app>

	<listener>
		<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
	</listener>

	<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>/WEB-INF/app-context.xml</param-value>
	</context-param>

	<servlet>
		<servlet-name>app</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value></param-value>
		</init-param>
		<load-on-startup>1</load-on-startup>
	</servlet>

	<servlet-mapping>
		<servlet-name>app</servlet-name>
		<url-pattern>/app/*</url-pattern>
	</servlet-mapping>

</web-app>
```
`ContextLoaderListener` 将在 WEB 应用程序启动时被初始化（因为它实现了 `ServletContextListener`)，然后从 `contextConfigLocation` 上下文初始化参数指定的配置文件中加载根应用上下文，并启动根应用上下文。

> 注意： `contextConfigLocation` 上下文初始化参数不同于 `DispatcherServlet` 的 `contextConfigLocation` Servlet初始化参数。它们不冲突；前者作用于整个 Servlet 上下文，而后者只作用于它所指定的 Servlet 。 监听器创建的根应用上下文将被自动设置为所有通过 `DispatcherServlet` 创建的应用上下文的父亲上下文。

### 初始化器中使用编程的方式启动 Spring
`ServletContextListener` 可以以编程的方式配置应用程序的中 Servlet、 Listener 和 Filter 。使用该接口的缺点是：监听器的 `contextInitialized` 方法可能在其它监听器之后调用。 Java EE 6 中添加了一个新的接口 `ServletContainerInitializer` 。 实现了 `ServletContainerInitializer` 接口的类将在程序启动时，并在所有监听器启动之前调用它的 `onStartup` 方法。这是应用程序生命周期中最早可以使用的时间点。但是，不要再部署描述符中配置 `ServletContainerInitializer` ，相反，需要使用 Java 的服务提供接口（SPI：Service Provider Interface）声明实现了 `ServletContainerInitializer` 的一个或多个类，在文件 `/META-INF/services/javax.servlet.ServletContainerInitializer` 中列出它们，每行一个类。

这种方式不利的一面在于文件不能直接存在于应用程序的 WAR 文件或解压后的目录中——不能将文件放在 Web 应用程序的 `/META-INF/services` 目录中。它必须在 JAR 文件的 `/META-INF/services` 目录中，并且需要将该JAR文件包含在应用程序WAR的 `WEB-INF/lib` 目录中。     

Spring Framework 提供了一个桥接口，是这种方式更容易实现。 `org.springframework.web.SpringServletContainerInitializer` 实现了 `ServletContainerInitializer` 接口，而且包含该类的 JAR 中包含了一个服务提供文件，列出了 `SpringServletContainerInitializer` 类的名字，所以应用程序会在启动时调用它的 `onStartup` 方法。然后，该类将在此方法中扫描应用程序以寻找实现了 `org.springframework.web.WebApplicationInitializer` 接口的实现类，并调用所有找到类的 `onStartup` 方法。在 `WebApplicationInitializer` 实现类中可以配置 `Servlet` 、 `Filter` 和 `Listener` ，更重要的是，可以在该方法中启动 Spring 。
```java
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