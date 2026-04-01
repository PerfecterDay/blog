# JavaEE基础之Web程序基本结构
{docsify-updated}

- [JavaEE基础之Web程序基本结构](#javaee基础之web程序基本结构)
	- [目录结构和War文件](#目录结构和war文件)
	- [部署描述符](#部署描述符)
	- [类加载器架构](#类加载器架构)


## 目录结构和War文件
所有的 Java EE 服务器都支持War应用程序归档，大部分服务器还支持未归档的应用程序目录。如下图所示：
<center><img src="pics/java-web程序目录结构.png" alt="Java Web应用目录结构" width=20% height=20%></center>

无论是未归档的文件还是 War归档文件，它们的目录结构都是一样的。
+ /META-INF:包含应用程序清单文件，也可以存储Web容器或服务器需要使用的资源。
+ /WEB-INF:存放一些包含了信息或指令的文件。
+ /WEB-INF/classes : 存放编译好的class类文件和资源文件。
+ /WEB-INF/classes/META-INF : 存放编译好的class类文件和资源文件。
+ /WEB-INF/lib : 存放依赖的jar包，在类路径上可用。
+ /WEB-INF/i18n : 存放国际化（i18n）和本地化(L10n)文件。
+ /WEB-INF/tags : 存放JSP标签文件。
+ /WEB-INF/tld : 存放JSP标签库描述符。

根路径下的 */META-INF* 不在类路径下，但是 */WEB-INF/classes/META-INF*在路径下。根路径下的资源文件一般可以通过URL直接访问。但是 */WEB-INF* 和 */META-INF* 下的文件是受保护的，不能直接通过URL访问。

## 部署描述符
部署描述符是用于描述Web应用程序的元数据，并为Java EE Web 应用程序服务器部署和运行程序提供指令。一般部署描述符文件在 */WEB-INF/web.xml* 中。该文件通常包含Servlet、监听器和过滤器的定义，以及HTTP会话、JSP和应用程序的配置选项。  

Servlet 3.0及之后更高版本的环境将扫描 Web应用程序和Web片段中的 Java EE Web应用程序注解，用于配置 Servlet、监听器和过滤器等，也就是说提供了注解配置应用程序的能力。如果需要，可以在在根&lt;web-app&gt; 或&lt;web-fragment&gt;元素中添加特性 metadata-complete="true"，禁止扫描和注解配置。

## 类加载器架构
不同于 JAVA SE 中的双亲优先委托模式，Java EE Web应用程序并不完全适用这种模式。考虑以下两种情况：
1. Java EE Web容器适用了与应用程序相同的第三方库，可能存在版本冲突。
2. 不同的web应用程序之间也可能存在版本冲突。

Java EE中，每个Web程序都被分配了一个自由的相互隔离的类加载器，它们都继承自公共的Web容器的类加载器。通过隔离不同的应用程序，它们不能访问相互的类（解决了上述第二个问题），不仅消除了类冲突的风险，还是一种防止web应用程序之间互相干扰的方式。

另外，Web应用程序使用子女优先的类加载模式，即类加载器通常只会在自己无法加载某个类的时候，才请求父类加载器帮助加载（解决上述第二个问题），Java EE服务器也提供了修改类加载模式的方法，可以改为双亲优先加载模式。