# JMX
{docsify-updated}

- [JMX](#jmx)
	- [简介](#简介)
	- [MBean](#mbean)
	- [注册一个MBean的一般步骤](#注册一个mbean的一般步骤)


### 简介
Java 管理扩展（JMX）框架是在 Java 1.5 中引入的，自问世以来已得到 Java 开发人员的广泛认可。它为本地或远程管理 Java 应用程序提供了一个易于配置、可扩展、可靠和友好的基础架构。该框架引入了 MBeans 概念，用于实时管理应用程序。

JMX 核心组件：
+ MBeans ：向 JMX 代理注册的 MBeans，通过它们管理资源
+ JMX agent layer ：核心组件（MbeanServer），负责维护受管 MBeans 的注册表并提供访问它们的接口
+ Remote management layer ：通常是客户端工具，如 JConsole/VisualVm

使用 JMX 技术，一个或多个 Java 对象（称为托管 Bean 或 MBeans）可对给定资源进行检测。这些 MBean 注册在一个核心管理对象服务器（称为 MBean 服务器）中。MBean 服务器充当管理代理，可在大多数已启用 Java 编程语言的设备上运行。

JMX 规范定义了 JMX 代理，您可以用它来管理任何已正确配置为管理的资源。JMX 代理由 MBean 服务器和一组用于处理 MBean 的服务组成，其中 MBean 服务器用于注册 MBean。这样，JMX 代理就能直接控制资源，并将其提供给远程管理应用程序。

JMX 技术定义了标准 connector（称为 JMX connector），使您能够从远程管理应用程序访问 JMX 代理。使用不同协议的 JMX 连接器提供相同的管理界面。因此，无论使用何种通信协议，管理应用程序都能透明地管理资源。

### MBean
JMX 规范定义了五种 MBean 类型：
+ Standard MBeans
+ Dynamic MBeans
+ Open MBeans
+ Model MBeans
+ MXBeans

### 注册一个MBean的一般步骤
1. 定义MBean  

	在创建 MBean 时，我们必须遵守一种特殊的设计模式。模型 MBean 类必须实现一个接口，其名称如下："模型类名称+MBean"。
	```
		public interface HelloMBean {
			public void sayHello();
			public int add(int x, int y);
			public String getName();
			public int getCacheSize();
			public void setCacheSize(int size);
		}

		public class Hello implements HelloMBean {
			private final String name = "Reginald";
			private int cacheSize = DEFAULT_CACHE_SIZE;
			private static final int
			DEFAULT_CACHE_SIZE = 200;
			public void sayHello() {
				System.out.println("hello, world");
			}

			public int add(int x, int y) {
				return x + y;
			}

			public String getName() {
				return this.name;
			}

			public int getCacheSize() {
				return this.cacheSize;
			}

			public synchronized void setCacheSize(int size) {
				this.cacheSize = size;
				System.out.println("Cache size now " + this.cacheSize);
			}
		}
	```

2. 定义MBean的名字  
	使用 `ObjectName` - 向 `PlatformMbeanServer` 注册 Mbean 类实例；这是一个由两部分组成的字符串：
	+ domain：可以是任意字符串，但根据 MBean 命名约定，它应包含 Java 包名（避免命名冲突）
	+ key：用逗号分隔的 "key=value "对列表
	在本例中，我们将使用"com.baledung.tutorial:type=basic,name=game"。

3. 创建 MBeanServer 并注册MBean

```
try {
	ObjectName objectName = new ObjectName("com.baeldung.tutorial:type=basic,name=game");
	MBeanServer server = ManagementFactory.getPlatformMBeanServer();
	server.registerMBean(new Game(), objectName);
} catch (MalformedObjectNameException | InstanceAlreadyExistsException |
		MBeanRegistrationException | NotCompliantMBeanException e) {
	// handle exceptions
}
```







