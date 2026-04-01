# 容器与 Bean
{docsify-updated}

> 

## Container Overview

`org.springframework.beans` 和 `org.springframework.context` 包构成了 Spring 框架 IoC 容器的基础。BeanFactory 接口提供了一种高级配置机制，能够管理任何类型的对象。 `ApplicationContext` 是 `BeanFactory` 的子接口，它增加了：
+ 更轻松地集成 Spring 的 AOP 功能
+ Message resource 处理（用于国际化）
+ 事件发布于处理
+ 应用层特有的上下文，例如用于Web应用程序的 `WebApplicationContext` 。

简而言之， `BeanFactory` 提供了配置框架和基本功能，而 `ApplicationContext` 则增加了更多企业级特定功能。 `ApplicationContext` 是 `BeanFactory` 的完整超集。

在Spring中，构成应用程序核心并由Spring IoC容器管理的对象称为Bean。Bean是由Spring IoC容器实例化、组装和管理的对象。Bean及其间的依赖关系，都体现在容器使用的配置元数据中。

`org.springframework.context.ApplicationContext` 接口代表 Spring IoC 容器，负责实例化、配置和组装 Bean。容器通过读取配置元数据来获取关于实例化、配置和组装组件的指令。配置元数据可以是带注解的组件类、包含工厂方法的配置类，或是外部XML文件或Groovy脚本。无论采用何种格式，都可据此构建应用程序，并实现组件间的丰富交互依赖关系。

Spring 核心中包含了多个实现 `ApplicationContext` 接口的类。在独立应用程序中，通常会创建 `AnnotationConfigApplicationContext` 或 `ClassPathXmlApplicationContext` 的实例。

下图展示了Spring工作原理的高级视图。应用程序类与配置元数据相结合，因此在 `ApplicationContext` 创建并初始化后，将获得一个完全配置且可执行的系统或应用程序。

<center><img src="pics/container-magic.png" alt=""></center>

### Configuration Metadata
如前图所示，Spring IoC容器会消耗一种配置元数据。这种配置元数据代表了作为应用程序开发人员如何指示Spring容器对应用程序中的组件进行实例化、配置和组装。

Spring IoC容器本身与配置元数据的实际写入格式完全解耦。如今，许多开发者为其Spring应用程序选择基于Java的配置：
+ 注解式配置：在应用程序的组件类上使用注解式配置元数据来定义Bean。
+ 基于Java的配置：通过基于Java的配置类在应用程序类外部定义Bean。要使用这些功能，请参阅 `@Configuration` 、 `@Bean` 、 `@Import` 和 `@DependsOn` 注解。

These bean definitions correspond to the actual objects that make up your application. Typically, you define service layer objects, persistence layer objects such as repositories or data access objects (DAOs), presentation objects such as Web controllers, infrastructure objects such as a JPA EntityManagerFactory, JMS queues, and so forth. Typically, one does not configure fine-grained domain objects in the container, because it is usually the responsibility of repositories and business logic to create and load domain objects.


这些bean定义对应于构成应用程序的实际对象。通常，需要定义服务层对象、持久层对象（如存储库或数据访问对象DAO）、展示层对象（如Web控制器）、基础架构对象（如JPA实体管理器工厂、JMS队列等）。通常情况下，细粒度的领域对象无需在容器中进行配置，因为创建和加载领域对象通常是存储库和业务逻辑的职责。


### Using the Container
```
// create and configure beans
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");
ApplicationContext context = new GenericGroovyApplicationContext("services.groovy", "daos.groovy");

GenericApplicationContext context = new GenericApplicationContext();
new XmlBeanDefinitionReader(context).loadBeanDefinitions("services.xml", "daos.xml");
context.refresh();

// retrieve configured instance
PetStoreService service = context.getBean("petStore", PetStoreService.class);

// use configured instance
List<String> userList = service.getUsernameList();
```

随后可通过 `getBean` 方法获取 Bean 实例。 `ApplicationContext` 接口还提供其他获取 Bean 的方法，但理想情况下应用程序代码应完全避免使用这些方法。实际上，应用程序代码不应包含任何 `getBean()` 调用，从而彻底消除对 Spring API 的依赖。例如，Spring 与 Web 框架的集成可为控制器、JSF 管理 Bean 等组件提供依赖注入，允许通过元数据（如自动装配注解）声明对特定 Bean 的依赖关系。


## Bean Overview
Spring IoC容器管理一个或多个Bean。这些Bean是根据提供给容器的配置元数据创建的（例如，以XML `<bean/>` 定义的形式）。

在容器内部，这些 Bean 定义以 `BeanDefinition` 对象的形式呈现，其中包含（除其他信息外）以下元数据：
+ 包限定类名：通常指正在定义的Bean的实际实现类。
+ Bean行为配置元素，用于定义Bean在容器中的行为方式（作用域、生命周期回调等）。
+ 其他 bean 的引用，这些引用对于该 bean 完成其工作是必需的。这些引用也被称为协作对象或依赖项。
+ 在新创建的对象中需设置的其他配置参数——例如，连接池的大小限制，或用于管理连接池的Bean中使用的连接数量。

这些元数据对应于构成每个 `BeanDefinition` 的一组属性。下表描述了这些属性：

**Table 1. The bean definition**
| Property                 | Explained in...               |
|--------------------------|-------------------------------|
| Class                    | Instantiating Beans           |
| Name                     | Naming Beans                  |
| Scope                    | Bean Scopes                   |
| Constructor arguments    | Dependency Injection          |
| Properties               | Dependency Injection          |
| Autowiring mode          | Autowiring Collaborators      |
| Lazy initialization mode | Lazy-initialized Beans        |
| Initialization method    | Initialization Callbacks      |
| Destruction method       | Destruction Callbacks         |

除了包含如何创建特定 Bean 的信息的 Bean 定义外， `ApplicationContext` 的实现还允许注册在容器外部（由用户）创建的现有对象。这通过调用  `getAutowireCapableBeanFactory()` 方法访问 `ApplicationContext` 的 `BeanFactory` 来实现，该方法返回 `DefaultListableBeanFactory` 的实现。`DefaultListableBeanFactory` 通过 `registerSingleton(..)` 和 `registerBeanDefinition(..)` 方法支持此类注册。但典型应用仅处理通过常规Bean定义元数据定义的Bean。

### Overriding Beans
当注册 Bean 时使用相同的 "ID" ，就会发生 Bean 覆盖。虽然 Bean 覆盖是可行的，但会降低配置文件的可读性。 Overriding 将来会被淘汰。

要完全禁用 Bean 重写功能，可以在 `ApplicationContext` 刷新之前将其 `allowBeanDefinitionOverriding` 标志设置为 `false` 。在此配置下，若尝试使用 Bean 重写，系统将抛出异常。

默认情况下，容器会以 `INFO` 级别记录每次尝试覆盖Bean的行为，以便据此调整配置。虽然不建议这样做，但可以通过将 `allowBeanDefinitionOverriding` 标志设置为`true` 来禁用这些日志。

**若使用 Java 配置，只要 `@Bean` 方法的返回类型与某个 bean 类匹配，对应的 `@Bean` 方法就会始终静默覆盖具有相同组件名称的扫描 bean 类。这意味着容器会优先调用  `@Bean` 工厂方法，而非注解扫描的 bean 。**


### Bean 实例化
`BeanDefinition` 本质上是创建一个或多个对象的配方。容器在被请求时会根据配方查找 `BeanDefinition` ，并利用该 `BeanDefinition` 封装的配置元数据来创建（或获取）实际对象。

+ 通常，在容器通过反射调用构造函数直接创建 Bean 的情况下，需指定要构造的 Bean 类，这与 Java 代码中使用 new 操作符的效果相当。
+ 在容器通过调用某个类的静态工厂方法来创建Bean的较少见情况下，需指定实际包含该静态工厂方法的类。静态工厂方法调用返回的对象类型可能是同类，也可能是完全不同的另一类。

例如，若你在 `com.example` 包中有一个名为 `SomeThing` 的类，且该 `SomeThing` 类包含一个名为 `OtherThing` 的静态嵌套类，则可用美元符号 (`$`) 或点号 (`.`) 分隔它们。因此在 Bean 定义中，类属性的值应为 `com.example.SomeThing$OtherThing` 或 `com.example.SomeThing.OtherThing` 。

#### Instantiation with a Constructor
当通过构造函数方式创建Bean时，所有常规类均可被Spring使用且与之兼容。也就是说，被开发的类无需实现任何特定接口，也无需采用特定编码方式。仅需指定Bean类即可满足需求。但根据该Bean使用的具体IoC类型，可能需要提供默认（空）构造函数。

#### 静态工厂方法注册 bean
```
<bean id="clientService"
	class="examples.ClientService"
	factory-method="createInstance"/>

public class ClientService {
	private static ClientService clientService = new ClientService();
	private ClientService() {}

	public static ClientService createInstance() {
		return clientService;
	}
}
```

工厂方法重载的典型问题案例是Mockito及其众多重载的mock方法。请尽可能选择最具体的mock变体： 
```
<bean id="clientService" class="org.mockito.Mockito" factory-method="mock">
	<constructor-arg type="java.lang.Class" value="examples.ClientService"/>
	<constructor-arg type="java.lang.String" value="clientService"/>
</bean>
```

#### 实例工厂方法注册 bean
与通过静态工厂方法进行实例化类似，使用实例工厂方法进行实例化时，会调用容器中现有 Bean 的非静态方法来创建新 Bean。要使用此机制，请将 `class` 属性留空，并在 `factory-bean` 属性中指定当前（或父容器、祖先容器）中包含待调用实例方法的 **Bean 名称**，该方法将用于创建对象。通过 `factory-method` 属性设置工厂方法本身的名称。以下示例展示了如何配置此类bean：
```
<bean id="serviceLocator" class="examples.DefaultServiceLocator">
	<!-- inject any dependencies required by this locator bean -->
</bean>

<bean id="clientService"
	factory-bean="serviceLocator"
	factory-method="createClientServiceInstance"/>

<bean id="accountService"
	factory-bean="serviceLocator"
	factory-method="createAccountServiceInstance"/>


public class DefaultServiceLocator {
	private static ClientService clientService = new ClientServiceImpl();

	private static AccountService accountService = new AccountServiceImpl();

	public ClientService createClientServiceInstance() {
		return clientService;
	}

	public AccountService createAccountServiceInstance() {
		return accountService;
	}
}
```

#### 确定 bean 的运行时类型
要确定一个特定Bean的运行时类型是不容易的。在Bean元数据定义中指定的类只是一个初始的类引用，可能与已声明的工厂方法相结合，或者是一个FactoryBean类，这可能导致Bean的运行时类型不同，或者在实例级工厂方法的情况下根本没有被设置（而是通过指定的工厂-bean名称来解决）。此外，AOP代理可能会用基于接口的代理来包装Bean实例，对目标Bean的实际类型（只是其实现的接口）的暴露有限。

要了解某个特定Bean的实际运行时类型，建议使用 `BeanFactory.getType(String beanName)` 方法或者指定bean的类型。这将考虑到上述所有情况，并返回 `BeanFactory.getBean` 调用对同一Bean名称将返回的对象类型。

## 在非网络应用中优雅地关闭Spring IoC容器
如果你在非网络应用环境中使用Spring的IoC容器（例如，在富客户端桌面环境中），请向JVM注册一个关机钩。这样做可以确保优雅地关闭，并在你的单体Bean上调用相关的 `destroy` 方法，从而释放所有资源。你仍然必须正确配置和实现这些销毁回调。

要注册一个关机钩子，请调用`registerShutdownHook()`方法，该方法在`ConfigurableApplicationContext`接口上声明，如下面的例子所示:
```
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public final class Boot {

	public static void main(final String[] args) throws Exception {
		ConfigurableApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml");

		// add a shutdown hook for the above context...
		ctx.registerShutdownHook();

		// app runs here...

		// main method exits, hook is called prior to the app shutting down...
	}
}
```

