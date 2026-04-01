# JavaEE基础之Servlet
{docsify-updated}

- [JavaEE基础之Servlet](#javaee基础之servlet)
	- [init 方法](#init-方法)
	- [destory 方法](#destory-方法)
	- [配置可部署的 Servlet](#配置可部署的-servlet)
		- [部署描述符](#部署描述符)
		- [@WebServlet注解](#webservlet注解)
		- [编程式配置](#编程式配置)
	- [使用 HttpServletRequest](#使用-httpservletrequest)
		- [获取请求参数](#获取请求参数)
		- [获取与请求内容相关的信息](#获取与请求内容相关的信息)
		- [获取请求内容（请求体中的内容，如上传的附件）](#获取请求内容请求体中的内容如上传的附件)
		- [获取请求特有的数据，如URL、URI和请求头](#获取请求特有的数据如urluri和请求头)
		- [会话和Cookise](#会话和cookise)
	- [使用 HttpServletResponse](#使用-httpservletresponse)
		- [编写响应正文](#编写响应正文)
		- [设置响应头和其它属性](#设置响应头和其它属性)
	- [使用初始化参数配置应用程序](#使用初始化参数配置应用程序)
		- [上下文初始化参数](#上下文初始化参数)
		- [Servlet初始化参数](#servlet初始化参数)


Servlet是所有Java Web应用程序的核心类，它是唯一的既可以直接处理和相应用户请求，也可以将处理工作委托给应用中其它部分的类。除非某些过滤器提前终止了客户端的请求，否则所有的请求都将被发送到某些Servlet中。

Java EE 容器会有一个或多个内建的 Servlet，用于处理 JSP、显示目录列表和访问静态资源。

`javax.servlet.Servlet` 是个接口，只包含了初始化并销毁 Servlet 和处理请求的方法。不过，无论什么类型的请求，甚至是非HTTP请求（假设容器支持这样的请求），也将会调用 service 方法。目前 Java EE 7支持的唯一 Servlet 协议就是 HTTP。

大多数情况下，Servlet都会继承自 `javax.servlet.http.HttpServlet` ，它提供了相应每种 HTTP 方法类型的方法空实现：
<center><img src="pics/httpservlet方法.png" alt="" width="60%"></center>

`HttpServlet` 各个方法接收的是 `javax.servlet.http.HttpServletRequest` 和 `javax.servlet.http.HttpServletResponse` 参数，而不是 `javax.servlet.http.ServletRequest` 和 `javax.servlet.http.HttpServletResponse` ，这样就可以轻松访问 Servlet 服务所处理的请求中的 HTTP 特定的特性。

## init 方法
`init()` 方法在 Servlet 构造完成之后调用，但在相应第一个请求之前。与构造器不同，调用 `init()` 方法时， Servlet 中所有的属性都已经设置完成，并提供了对 `javax.servlet.ServletConfig` 和 `javax.servlet.ServletContext` 对象的访问。所以，可以使用该方法*读取属性文件*，或者使用 *JDBC连接数据库*。

`init()` 方法将在 Servlet 启动时调用。 Servlet 会在第一次映射到的请求访问它的时候启动，如果配置了在Web应用程序部署和启动时自动启动，那么 `init()` 方法也会被调用。

## destory 方法
`destory()` 方法在 Servlet 不再接受请求之后立即调用，这通常发生在 Web 应用程序被停止或卸载，或者 Web 容器关闭时。因为它将在卸载或关闭时立即调用，所以不需要等待垃圾收集器启动垃圾回收就可以清理资源。

这对于应用程序被卸载了但是服务器仍然运行的环境来说非常重要（服务器还要支持其它应用运行），因为垃圾收集器可能在几分钟或数小时之后运行。如果在垃圾收集时清理资源而不是在 `destory()` 方法清理资源，则会导致应用程序占用的资源无法释放。

因此，应当总是使用 `destory()` 方法清理 Servlet 持有的资源。

## 配置可部署的 Servlet
在编写创建好 Servlet 之后，需要告诉容器如何部署应用程序中的 Servlet。主要有以下三种方式：

### 部署描述符
在部署描述符中进行正确的配置：
1. 在 web.xml 中添加 Servlet
```
<servlet>
	<servlet-name>helloServlet</servlet-name>
	<servlet-class>com.servlet.HelloServlet</servlet-class>
	<load-on-startup>1</load-on-startup>
</servlet>
```
如果没有 load-on-startup 标签， Servlet 会在第一个请求到达的时候被初始化及调用init方法。可能需要很长时间。加上 load-on-startup 标签后，表示在部署时就创建相应的 Servlet。 load-on-startup 标签值的大小表示启动顺序，越小越先启动。相同则按在描述符文件中出现的顺序启动。

2. 将 Servlet 映射到 URL
告诉容器如何启动 Servlet 之后，还需要告诉容器不同的 Servlet 分别处理对应哪些URL的请求。
``` 
<servlet-mapping>
    <servlet-name>helloServlet</servlet-name>
    <url-pattern>/hello</url-pattern>
    <url-pattern>/hello2</url-pattern>
</servlet-mapping>
```
使用了上述标签后，所有访问应用程序相对URL：/hello和/hello2 的请求都将由上面配置的helloServlet来处理。此处的 servlet-name要和上面的一样。

### @WebServlet注解
```
@WebServlet(
    name = "helloServlet",
    urlPatterns = {"/hello1","/hello2"},
    loadOnStartup = 1
)
public class HelloServlet extends HttpServlet{...}
```

### 编程式配置
除了上述两种方法，还可以使用编程的方式配置 Servlet.
`ServletContext` 接口提供了动态添加 Servlet 的方法：
```
1. Dynamic addServlet(String var1, String var2); 
2. Dynamic addServlet(String var1, Servlet var2); 
3. Dynamic addServlet(String var1, Class<? extends Servlet> var2);
```
与编程式添加 Listener 和 Filter 一样，这必须要在在 `ServletContext` 配置完成之前完成，因为容器会根据 `ServletContext` 配置决定加载哪些 Listener/Filter/Servlet. 所以要在 `ServletContextListener` 的 `contextInitialized()` 方法或者 `ServletContainerInitializer` 中的 `onStartup()` 中注册 Servlet 。

## 使用 HttpServletRequest
`HttpServletRequest` 是对 `ServletRequest` 的扩展，可以提供关于收到请求的额外的与 HTTP 协议相关的信息。通过它可以获取 HTTP 请求的详细信息。

### 获取请求参数
 `HttpServletRequest` 最重要的功能就是从客户端发送的请求中获取查询参数。请求参数有两种不同的形式：
 1. 查询参数，?key=value...
 2. 以 `application/x-www-form-yrlencoded` / `multipart/form-data`  编码的请求正文。

常用方法：
1. `String getParameter(String var1)`
2. `String[] getParameterValues(String var1)`
3. `Map<String, String[]> getParameterMap()`
4. `Enumeration<String> getParameterNames()`

### 获取与请求内容相关的信息
与请求内容相关的信息包括：HTTP 请求内容的类型、长度和编码等。
1. `String getContentType()`
2. `int getContentLength()`
3. `long getContentLengthLong()`
4. `String getCharacterEncoding()`

### 获取请求内容（请求体中的内容，如上传的附件）
1. `ServletInputStream getInputStream() throws IOException: 二进制`
2. `BufferedReader getReader() throws IOException：文本`

>注意：请求流只能被读取一次，多次读取会导致 IllegalStateException 异常，如果请求中含有 POST 的请求参数，且使用了获取请求参数的方法获取了POST变量，则不能再次调用上述方法，否则也会导致 IllegalStateException 异常。不要在含有 POST 变量的请求上使用上述方法。


### 获取请求特有的数据，如URL、URI和请求头
1. `StringBuffer getRequestURL()`
2. `String getRequestURI()`
3. `String getServletPath()`
4. `String getHeader(String var1)`
5. `Enumeration<String> getHeaders(String var1)`
6. `Enumeration<String> getHeaderNames()`
7. `long getDateHeader(String var1)`
8. `int getIntHeader(String var1)`

### 会话和Cookise
1. `Cookie[] getCookies()`
2. `HttpSession getSession()/HttpSession getSession(boolean var1)`

## 使用 HttpServletResponse
`HttpServletResponse` 提供了对响应中与HTTP协议相关属性的访问。可以使用 `HttpServletResponse` 完成响应头设置、编写响应正文、重定向请求、设置 HTTP 状态码、设置 Cookies等任务。

### 编写响应正文
`HttpServletResponse` 最重要的功能就是向客户端返回数据内容，可以是在浏览器中显示的HTML、浏览器希望获取的图像或客户端下载的文件内容等。
1. `ServletOutputStream getOutputStream() throws IOException`
2. `PrintWriter getWriter() throws IOException`

同样，不要对同一个响应对象同时调用上述方法，也不能调用两次其中一个方法。

### 设置响应头和其它属性
1. `void addHeader(String var1, String var2)`
2. `void setHeader(String var1, int var2)`
3. `void setStatus(int var1)`
4. `void sendError(int var1) throws IOException`
5. `void sendRedirect(String var1) throws IOException`
6. `void setDateHeader(String var1, long var2)`
7. `void addDateHeader(String var1, long var2)`

## 使用初始化参数配置应用程序
编写 Java Web 应用程序时，不可避免地会需要提供一些配置应用程序和Servlet的方式。通过上下文初始化参数和Servlet初始化参数可对它们进行配置。可以定义数据库连接信息、提供发送订单警告的邮件地址等。这些配置在程序启动时被读取，修改后只有重启应用才会生效。

### 上下文初始化参数
在部署描述符中使用 <context-param> 标签声明上下文初始化参数。
```
<context-param>
    <param-name>settingOne</param-name>
    <param-value>foo</param-value>
</context-param>
<context-param>
    <param-name>settingTwo</param-name>
    <param-value>bar</param-value>
</context-param>
```
上述参数会被加载到 `ServletContext` 中，使用 `ServletContext` 的     
`String getInitParameter(String var1)` 方法可以获取到初始化参数。 `ServletContext` 是所有 Servlet 共享的，在任意 Servlet 中都可以获取它的引用，因此，上下文初始化参数是所有 Servlet 共享的。

`GenericServlet` 中有一个 `ServletConfig` 类型的成员，使用 `ServletConfig` 的 `getServletContext()` 可以获取到 `ServletContext` 。实际上，`GenericServlet` 提供了 `getServletContext()` 方法：
```
public ServletContext getServletContext() {
    return this.getServletConfig().getServletContext();
}
```
### Servlet初始化参数
1. 在部署描述符中使用下属代码可以为特定 Servlet 配置初始化参数：  
	```
	<servlet>
			<servlet-name>helloServlet</servlet-name>
			<servlet-class>com.servlet.HelloServlet</servlet-class>
			<init-param>
				<param-name>database</param-name>
				<param-value>db</param-value>
			</init-param>
				<init-param>
				<param-name>server</param-name>
				<param-value>10.1.21.3</param-value>
			</init-param>
			<load-on-startup>1</load-on-startup>
		</servlet>
	```
	Servlet初始化参数会被加载到各个Servlet的 `ServletConfig` 中，因此，它们是Servlet专有的。通过 `ServletConfig` 的 `String getInitParameter(String var1)` 方法获取参数值。

2. 使用注解：  
	```
	@WebServlet(
		name = "helloServlet",
		urlPatterns = {"/hello1","/hello2"},
		loadOnStartup = 1,
		initParams = {
			@WebInitParam(name = "database",value = "db"),
			@WebInitParam(name = "server",value = "10.1.21.3")
		}
	)
	public class HelloServlet extends HttpServlet{...}
	```
	使用注解的缺点是，修改配置后，需要重新编译程序；而使用部署描述符的方式，修改完后只要重新启动程序即可，无需重新编译。
