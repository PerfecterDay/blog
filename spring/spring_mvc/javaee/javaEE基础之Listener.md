# JavaEE基础之Listener
{docsify-updated}

- [JavaEE基础之Listener](#javaee基础之listener)
	- [Java EE 中的 Listener](#java-ee-中的-listener)
	- [配置 Listener](#配置-listener)



## Java EE 中的 Listener 
与许多其他不同类型的应用程序一样， web 应用程序也有自己的生命周期。 既有启动进程也有关闭进程， 在这些阶段中可以执行许多不同的任务。Java EE web 应用程序支持各种不同类型的监听器，这些监听器可以通知代码多种事件，例如应用程序启动、应用程序关闭、 HTTP 会话创建和会话销毁。

下面是 Java EE 中的 8 个 `Listener` 接口，可分为三类：
1. `ServletContext` 相关接口

	+ `ServletContextListener` ：用于监听 `ServletContext` 的启动和销毁。
	+ `ServletContextAttributeListener` ：用于监听 application 范围的属性变化。

2. `HttpSession` 相关接口
   
	+ `HttpSessionListener` ：用于监听 session 的创建和销毁。
	+ `HttpSessionIdListener`  ：用于监听 session 的 id 是否被更改。
	+ `HttpSessionAttributeListener` ：用于监听 session 范围的属性变化。
	+ `HttpSessionActivationListener` ：用于监听绑定在 HttpSession 对象中的 JavaBean 状态。
	+ `HttpSessionBindingListener` ：用于监听对象与 session 的绑定和解绑。

3. `ServletRequest` 相关接口

	+ `ServletRequestListener` ：用于监听 ServletRequest 对象的初始化和销毁。
	+ `ServletRequestAttributeListener` ：用于监听 ServletRequest 对象的属性变化。

4. 异步接口


	+ `AsyncListener` : 用于监听异步请求。

## 配置 Listener
可以使用3种方式配置：
1. 使用部署描述符

	在部署描述符中添加
	```
	<listener>
		<listener-class>
		com.journaldev.listener.AppContextListener
		</listener-class>
	</listener>
	```

2. 使用 `@WebListener` 注解

	在实现了一个或多个上述Listener接口的类上标注 `@WebListener` 注解，即可将这个类声明为 Listener。
	```
	@WebListener
	public class AppContextListener implements ServletContextListener {
	}
	```

3. 编程式配置

	`ServletContext` 接口提供了动态添加 Listener 的方法：
	```
	1. void addListener(String var1);
	2. <T extends EventListener> void addListener(T var1);
	3. void addListener(Class<? extends EventListener> var1);
	```
	与编程式添加 Servlet 和 Filter 一样，这必须要在在 `ServletContext` 配置完成之前完成，因为容器会根据 `ServletContext` 配置决定加载哪些 Listener/Filter/Servlet. 所以要在 `ServletContextListener` 的 `contextInitialized()` 方法或者 `ServletContainerInitializer` 中的 `onStartup()` 中注册 Listener。