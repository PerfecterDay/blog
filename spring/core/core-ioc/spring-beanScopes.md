# Bean scopes
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html

创建一个 `BeanDefinition` 时，你实际上是在创建一个如何创建 Bean 的配方。将 `BeanDefinition` 视为配方这一概念至关重要，因为这意味着如同Java类一样，我们能创建一个类的多个实例对象，同样， Spring 也可以通过 `BeanDefinition` 配方生成多个对象实例 Bean。

您不仅能够控制从特定Bean定义创建的对象中需要注入的各种依赖项和配置值，还能控制从特定Bean定义创建的对象的 `Scope` 。这种方法既强大又灵活，因为您可以通过配置选择创建对象的 `Scope` ，而无需在Java类级别硬编码对象的 `Scope` 。Bean可以被定义为部署在多种 `Scope` 之一中。Spring框架支持六种 `Scope` ，其中四种仅在使用Web感类型的 `ApplicationContext` 时可用。

+ `singleton`: 标记为拥有 `singleton scope` 的 `BeanDefinition`，在Spring的IoC容器中只存在一个实例，所有对该对象的引用将共享这个实例。该实例从容器启动，并因为第一次被请求而初始化之后，将一直存活到容器退出，也就是说，它与IoC容器“几乎”拥有相同的生命周期。
+ `prototype` ：容器在接到该类型对象的请求的时候，会每次都重新生成一个新的对象实例给请求方。虽然这种类型的对象的实例化以及属性设置等工作都是由容器负责的，但是只要准备完毕，并且对象实例返回给请求方之后，容器就不再拥有当前返回对象的引用，请求方需要自己负责当前返回对象的后继生命周期的管理工作，包括该对象的销毁。也就是说，容器每次返回给请求方一个新的对象实例之后，就任由这个对象实例“自生自灭”了。
+ `request`: Spring容器会为每个HTTP请求创建一个全新的bean对象供当前请求使用，当请求结束后，该对象实例的生命周期即告结束。
+ `session` ：Spring容器会为每个独立的 `session` 创建属于它们自己的全新的bean对象实例。与 `request` 相比，除了可能更长的存活时间，其他方没什么差别。
+ `application`: 将单个 Bean 定义的 `Scope` 扩展到 `ServletContext` 的生命周期。仅在Web 类型的 `Spring ApplicationContext` 的上下文中有效。
+ `websocket`: 将单个 Bean 定义的 `Scope` 扩展到 `WebSocket` 的生命周期。仅在Web 类型的 `Spring ApplicationContext` 的上下文中有效。


## Singleton
<center><img src="pics/singleton.png" alt=""></center>

仅管理单例 bean 的一个共享实例，所有请求 ID 与该 bean 定义匹配的 bean 请求，都会由 Spring 容器返回该特定 bean 实例。换言之，当你定义一个 `singleton scope` 的Bean时，Spring IoC容器会为该Bean定义创建且仅创建一个对象实例。这个单一实例会被存储在单例Bean的缓存中，此后所有对该命名Bean的请求和引用都会返回缓存中的对象。

若在单个Spring容器中为特定类定义一个Bean，则该容器仅会创建该Bean定义所对应类的唯一实例。单例范围是Spring的默认 `Scope` 。Spring 默认会预先创建好所有的单例 Bean.

## Prototype
<center><img src="pics/prototype.png" alt=""></center>

注意，Spring 默认只会预先创建单例 bean， `prototype` 类型的 bean 不会被预先创建， 只有当 `prototype` 的 bean被注入到另一个单例bean中，或通过容器的 `getBean()` 方法调用请求bean时，才会触发实例创建。通常应遵循以下原则：**所有有状态bean应采用 `prototype scope` ，无状态bean则应采用 `singleton scope` 。**

与其他 `Scope` 不同，Spring 不会管理原型 Bean 的完整生命周期。容器会实例化、配置并组装原型对象，然后将其传递给应用程序后，此后就不会对该原型实例进行后续管理。因此，尽管初始化生命周期回调方法（`@PostConstruct`）都会被Spring调用（无论 `Scope` 如何），但在 `prototype` 情况下，**配置的销毁生命周期回调（`@PreDestroy`）不会被调用。客户端代码必须自行清理 `prototype scope` 对象，并释放原型 bean 占用的高成本资源**。若需让 Spring 容器释放 `prototype scope`  bean 占用的资源，可尝试使用自定义 `bean post-processor` ，该处理器会持有待清理 bean 的引用。

```
<bean id="accountService" class="com.something.DefaultAccountService" scope="prototype"/>

@Scope(value = BeanDefinition.SCOPE_PROTOTYPE)
@Component
public class DefaultAccountService {
	// ...
}


@Configuration
class AccountConfiguration {

    // 声明原型bean
    @Scope(BeanDefinition.SCOPE_PROTOTYPE)
    @Bean
    public DefaultAccountService accountService(){
        ...
    }
}

```

从某些方面来看，Spring容器对原型 `Scope` Bean的作用相当于Java new运算符的替代方案。此后所有生命周期管理都必须由客户端自行处理。

### signleton 依赖 prototype 
当一个 `singleton` 类型的 bean 依赖一个 `prototype` 类型的 bean 时，请注意这个 `prototype` 类型的bean 只会在 `singletone` bean 实例化的时候完成构造注入。系统会先实例化新的 `prototype` bean，再将其注入 `singleton` bean。此后，该原型实例将成为唯一提供给 `singletone` bean 的实例。

然而，假设你希望 `singletone` bean在运行时反复获取 `prototype` bean的新实例。不能通过将 `prototype` bean依赖注入到 `singletone` bean中的方式实现，因为这种注入仅发生一次，就是当Spring容器实例化 `singletone` bean并解析注入其依赖项时。若需在运行时多次获取 `prototype` bean的新实例，请参阅方法注入。

## Request, Session, Application, and WebSocket Scopes
`request` , `session` , `application` 和 `websocket` scopes 仅在使用支持 Web 的Spring `ApplicationContext` 实现（如 `XmlWebApplicationContext` ）时可用。若在常规Spring IoC容器（如 `ClassPathXmlApplicationContext`）中使用这些 `Scope` ，将抛出 `IllegalStateException` 异常，提示存在未知bean `Scope` 。

### 初始化 Web 配置
为支持在 `request` , `session` , `application` 和 `websocket` 级别（即Web `Scope` Bean）定义Bean的 `Scope` ，您需要在定义Bean之前进行一些简单的初始配置。（单例和原型则无需进行。）

+ 在Spring Web MVC中访问 `Scope` bean时，实际上是在Spring `DispatcherServlet` 处理的请求内进行操作，因此无需特殊配置。 `DispatcherServlet` 已自动暴露所有相关状态。
+ 若使用Servlet Web容器且请求在Spring的DispatcherServlet之外处理（例如使用JSF时），则需注册org.springframework.web.context.request.RequestContextListener ServletRequestListener。可通过WebApplicationInitializer接口以编程方式实现，或在Web应用程序的web.xml文件中添加以下声明：
  ```
  <web-app>
      ...
      <listener>
          <listener-class>
              org.springframework.web.context.request.RequestContextListener
          </listener-class>
      </listener>
      ...
  </web-app>
  ```
+ 或者，如果监听器配置存在问题，请考虑使用 Spring 的 `RequestContextFilter` 。该过滤器的映射取决于周围的 Web 应用程序配置，因此您需要根据实际情况进行调整。以下代码片段展示了 Web 应用程序中的过滤器部分：
  ```
  <web-app>
  	...
  	<filter>
  		<filter-name>requestContextFilter</filter-name>
  		<filter-class>org.springframework.web.filter.RequestContextFilter</filter-class>
  	</filter>
  	<filter-mapping>
  		<filter-name>requestContextFilter</filter-name>
  		<url-pattern>/*</url-pattern>
  	</filter-mapping>
  	...
  </web-app>
  ```

`DispatcherServlet` 、 `RequestContextListener` 和 `RequestContextFilter` 的功能完全相同，即把 HTTP 请求对象绑定到处理该请求的线程。这使得请求 `Scope` 和 `session scope` 的 Bean 在调用链的后续环节中可用。


### Request scope
```
<bean id="loginAction" class="com.something.LoginAction" scope="request"/>

@RequestScope
@Component
public class LoginAction {
	// ...
}
```
Spring容器会为每个HTTP请求使用loginAction Bean定义创建一个新的LoginAction Bean实例。也就是说，loginAction Bean的 `Scope` 限定在HTTP请求级别。您可以随意修改创建实例的内部状态，因为其他从同一loginAction bean定义创建的实例不会看到这些状态变化。这些变化仅针对单个请求。当请求处理完成时， `Scope` 限定于该请求的bean会被丢弃。

### Session Scope
```
<bean id="userPreferences" class="com.something.UserPreferences" scope="session"/>

@SessionScope
@Component
public class UserPreferences {
	// ...
}
```
Spring容器通过 userPreferences Bean定义创建UserPreferences Bean的新实例，其生命周期仅限于单次HTTP会话。换言之，userPreferences Bean的有效 `Scope` 限定在HTTP会话级别。与请求 `Scope` 的 Bean 类似，您可以随意修改所创建实例的内部状态，但需注意：其他使用相同 userPreferences Bean 定义创建的实例不会看到这些状态变更，因为这些变更仅属于特定的 HTTP 会话。当 HTTP 会话最终被丢弃时，属于该特定 HTTP  `session scope` 的 Bean 也将被丢弃。

### Application Scope
```
<bean id="appPreferences" class="com.something.AppPreferences" scope="application"/>

@ApplicationScope
@Component
public class AppPreferences {
	// ...
}
```

Spring容器通过在整个Web应用程序中仅使用一次appPreferences Bean定义来创建AppPreferences Bean的新实例。也就是说，appPreferences Bean的 `Scope` 限定在 `ServletContext` 级别，并作为常规的 `ServletContext` 属性存储。这与Spring单例bean有相似之处，但存在两点重要差异：
+ 其一，它是按 `ServletContext` 实例化的单例（而非按Spring `ApplicationContext` 实例化——单个Web应用中可能存在多个 `ApplicationContext`）；
+ 其二，它实际暴露为 `ServletContext` 属性，因此可见。

### WebSocket Scope
> https://docs.spring.io/spring-framework/reference/web/websocket/stomp/scope.html


### 如何注入 request/session 类型的 bean
Spring IoC容器不仅管理对象（Bean）的实例化，还负责协作对象（或依赖项）的连接。若需将（例如）HTTP请求 `Scope` 的bean注入到 `Scope` 更长的另一个bean中，可选择注入AOP代理替代原有 `Scope` bean。具体而言，需注入一个代理对象：该对象既暴露与被注入对象相同的公共接口，又能从相关 `Scope` （如HTTP请求）中获取真实目标对象，并将方法调用委托给真实对象执行。

为何在常见场景中， request 、 session  `Scope` 和自定义 `Scope` 级别的Bean定义需要使用 `<aop:scoped-proxy/>` 元素？请考虑以下单例Bean定义:
```
<bean id="userPreferences" class="com.something.UserPreferences" scope="session"/>

<bean id="userManager" class="com.something.UserManager">
	<property name="userPreferences" ref="userPreferences"/>
</bean>
```
在上例中，单例 bean（ `userManager` ）被注入了对 HTTP  `session scope`  bean（ `userPreferences` ）的引用。关键点在于 `userManager` bean是单例模式：每个容器仅实例化一次，其依赖项（此处仅有 `userPreferences` bean）也仅在实例化的时候被注入一次。这意味着 `userManager` bean始终操作着完全相同的 `userPreferences` 对象（即最初注入时关联的那个对象）。但是，实际上我们需要的是和 `session` 关联的对象。

当将短生命周期 `Scope` 的 Bean 注入到长生命周期 `Scope` 的 Bean 中时（例如将 HTTP  `session scope` 的协作 Bean 作为依赖注入到单例 Bean 中），这种行为并非我们所期望的。相反，你需要一个唯一的 `userManager` 对象，并且在 HTTP 会话的生命周期内，需要一个专属于该会话的 `userPreferences` 对象。因此容器会创建一个暴露与 `UserPreferences` 类完全相同公共接口的对象（理想情况下应为 `UserPreferences` 实例），该对象可从 `Scope` 机制（HTTP请求、会话等）中获取真实的 `UserPreferences` 对象。容器将此代理对象注入 `userManager` bean，而该bean并不知晓此 `UserPreferences` 引用实为代理。在此示例中，当 `UserManager` 实例调用依赖注入的 `UserPreferences` 对象的方法时，实际上是在调用代理对象的方法。代理随后从（本例中）HTTP会话中获取真实的 `UserPreferences` 对象，并将方法调用委托给该真实对象执行。

正确的配置应该如下：
```
<bean id="userPreferences" class="com.something.UserPreferences" scope="session">
	<aop:scoped-proxy/>
</bean>

<bean id="userManager" class="com.something.UserManager">
	<property name="userPreferences" ref="userPreferences"/>
</bean>
```

当使用java 编程配置时，可以使用 `@Scope` 等注解达到相同的效果:
```
// an HTTP Session-scoped bean exposed as a proxy
@Bean
@SessionScope
public UserPreferences userPreferences() {
	return new UserPreferences();
}

@Bean
public Service userService() {
	UserService service = new SimpleUserService();
	// a reference to the proxied userPreferences bean
	service.setUserPreferences(userPreferences());
	return service;
}

@Bean
@Scope(proxyMode=ScopedProxyMode.TARGET_CLASS)
public TestBean testBean(){
    reurn new TestBean();
}
```
在Java中使用 `@Scope` 注解配置Bean时，其提供的支持与 `proxyMode` 属性等效。默认值为 `ScopedProxyMode.DEFAULT` ，通常表示除非在组件扫描指令级别配置了其他默认值，否则不应创建作用域代理。可以指定 `ScopedProxyMode.TARGET_CLASS` 、 `ScopedProxyMode.INTERFACES` 或 `ScopedProxyMode.NO` 。

#### 选择动态代理的类型
默认情况下，当Spring容器为标记有 `<aop:scoped-proxy/>` 元素的bean创建代理时，会生成基于 `CGLIB` 的类代理。

也可以通过将 `<aop:scoped-proxy/>` 元素的 `proxy-target-class` 属性值设为 `false` ，配置Spring容器为这类 `Scope` bean创建基于标准JDK接口的代理。使用基于JDK接口的代理意味着应用程序类路径中无需额外库即可实现此类代理功能。但这也意味着： `Scope`  Bean 的类必须至少实现一个接口，且所有注入该 Bean 的协作对象都必须通过其接口之一来引用该 Bean。以下示例展示了基于接口的代理实现：
```
<!-- DefaultUserPreferences implements the UserPreferences interface -->
<bean id="userPreferences" class="com.stuff.DefaultUserPreferences" scope="session">
	<aop:scoped-proxy proxy-target-class="false"/>
</bean>

<bean id="userManager" class="com.stuff.UserManager">
	<property name="userPreferences" ref="userPreferences"/>
</bean>
```

#### ObjectFactory 和 Provider
此外，`scoped-proxy` 并非在生命周期安全模式下访问较短 `Scope` 中Bean的唯一途径。还可将注入点（即构造函数、setter方法参数或自动装配字段）声明为 `ObjectFactory<MyTargetBean>` 类型，这样每次需要时都能通过调用 `getObject()` 按需获取当前实例——无需持有实例或单独存储。

JSR-330的此变体称为 `Provider` ，需配合 `Provider<MyTargetBean>` 声明使用，每次检索操作都需调用对应的 `get()` 


## 自定义 Scopes
Bean Scopes 机制具有可扩展性。可以定义自己的 `Scope` ，甚至重定义现有 `Scope` ——尽管后者被视为不良实践，且无法覆盖内置的 `singleton` 和 `prototype` 。

### 实现 Scope 接口
如果需要自定义 Scopes ，需要实现 `org.springframework.beans.factory.config.Scope` 接口。 `Scope` 接口提供了四种方法：获取 `Scope` 中的对象、从 `Scope` 中移除对象，以及允许对象被销毁。

 `session scope` 的实现，例如，会返回 `session scope` 的 Bean（如果该 Bean 不存在，方法会返回 Bean 的新实例，并绑定到会话后供后续引用）。以下方法从底层 `Scope` 返回对象：
```
Object get(String name, ObjectFactory<?> objectFactory)
```

`session scope` 会从底层会话中移除 `session scope` 的 Bean。应返回该对象，但若未找到指定名称的对象，则可返回 null。以下方法将对象从底层 `scope` 中移除：
```
Object remove(String name)
```

以下方法用于注册回调函数，该函数将在 `scope` 被销毁时或 `scope` 中指定对象被销毁时被调用：
```
void registerDestructionCallback(String name, Runnable destructionCallback)
```

以下方法获取底层作用域的对话标识符：
```
String getConversationId()
```
该标识符在每个作用域中各不相同。对于会话作用域的实现，该标识符可以是会话标识符。

### 注册自定义的 Scope
在编写并测试一个或多个自定义作用域实现后，我们需要让Spring容器识别这些新作用域。以下方法是向Spring容器注册新作用域的核心方法：
```
void registerScope(String scopeName, Scope scope);
```
这是 `ConfigurableBeanFactory` 接口的方法，绝大多数的 `ApplicationContext` 的 `BeanFactory` 属性都是这个接口类型。  
`registerScope(..)` 方法的第一个参数是与作用域关联的唯一名称。Spring 容器中常见的此类名称包括 `singleton` 和 `prototype` 。该方法的第二个参数则是自定义的 `Scope` 实例。
```
Scope threadScope = new SimpleThreadScope();
beanFactory.registerScope("thread", threadScope);
```
然后就可以使用这个 `Scope` 了： `<bean id="..." class="..." scope="thread">`

除了上述编程方式的注册外，还能使用 `CustomScopeConfigurer` 来注册：
```
<bean class="org.springframework.beans.factory.config.CustomScopeConfigurer">
    <property name="scopes">
        <map>
            <entry key="thread">
                <bean class="org.springframework.context.support.SimpleThreadScope"/>
            </entry>
        </map>
    </property>
</bean>

<bean id="thing2" class="x.y.Thing2" scope="thread">
    <property name="name" value="Rick"/>
    <aop:scoped-proxy/>
</bean>

<bean id="thing1" class="x.y.Thing1">
    <property name="thing2" ref="thing2"/>
</bean>
```