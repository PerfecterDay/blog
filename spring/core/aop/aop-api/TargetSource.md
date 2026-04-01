# TargetSource
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/aop-api/targetsource.html

Spring 提供了 `TargetSource` 概念，该概念通过 `org.springframework.aop.TargetSource` 接口进行定义。该接口负责返回实现连接点的“目标对象”。每当 AOP 代理处理方法调用时，都请求一个 `TargetSource` 的实现来获取一个目标对象的实例。

使用 Spring AOP 的开发人员通常无需直接处理 `TargetSource` 的实现，但这为支持池化、热插拔及其他复杂的目标提供了强有力的手段。例如，一个池化的 `TargetSource` 可以通过使用池来管理实例，从而在每次调用时返回不同的目标实例。

如果未指定 `TargetSource` ，系统将使用默认实现来封装本地对象。每次调用都会返回相同的目标（正如您所预期的那样）。

使用自定义 `TargetSource` 时，目标对象通常应定义为原型（ `prototype` ），而非单例（ `singleton` ）Bean。这样，Spring 就能在需要时创建新的目标实例。

```
public interface TargetSource extends TargetClassAware {
	@Override
	@Nullable Class<?> getTargetClass();

	default boolean isStatic() {
		return false;
	}

	@Nullable Object getTarget() throws Exception;

	default void releaseTarget(Object target) throws Exception {
	}

}
```

## Hot-swappable Target Sources
`org.springframework.aop.target.HotSwappableTargetSource` 的存在，是为了允许在保持调用方对其引用不变的同时，切换 AOP 代理的目标。更改 `TargetSource` 的目标会立即生效。 `HotSwappableTargetSource` 支持多线程安全。

可以使用 `HotSwappableTargetSource` 上的 `swap()` 方法来更改目标，如下例所示：
```
<bean id="initialTarget" class="mycompany.OldTarget"/>

<bean id="swapper" class="org.springframework.aop.target.HotSwappableTargetSource">
	<constructor-arg ref="initialTarget"/>
</bean>

<bean id="swappable" class="org.springframework.aop.framework.ProxyFactoryBean">
	<property name="targetSource" ref="swapper"/>
</bean>


HotSwappableTargetSource swapper = (HotSwappableTargetSource) beanFactory.getBean("swapper");
Object oldTarget = swapper.swap(newTarget);
```
前面的 `swap()` 调用更改了可交换 Bean 的目标实例。持有该 Bean 引用客户端不会察觉到这一变化，但新的AOP 方法调用会转移到新的目标实例。

尽管此示例未添加任何 `Advice` （使用 `TargetSource` 时无需添加 `Advice` ），但任何 `TargetSource` 均可与任意 `Advice` 结合使用。

## Pooling Target Sources
使用池化目标源提供的编程模型与无状态会话 EJB 类似，其中会维护一个由相同实例组成的池，方法调用将指向池中可用的对象。

Spring 池化与 SLSB 池化之间一个关键的区别在于，Spring 池化可应用于任何 POJO。与 Spring 的其他特性一样，该服务可以以非侵入式的方式进行应用。

Spring 支持 Commons Pool 2，该库提供了一种相当高效的池化实现。要使用此功能，需要在应用程序的类路径中添加 `commons-pool` JAR 文件。还可以继承 `org.springframework.aop.target.AbstractPoolingTargetSource` 类，以支持任何其他池化 API。
```
<bean id="businessObjectTarget" class="com.mycompany.MyBusinessObject"
		scope="prototype">
	... properties omitted
</bean>

<bean id="poolTargetSource" class="org.springframework.aop.target.CommonsPool2TargetSource">
	<property name="targetBeanName" value="businessObjectTarget"/>
	<property name="maxSize" value="25"/>
</bean>

<bean id="businessObject" class="org.springframework.aop.framework.ProxyFactoryBean">
	<property name="targetSource" ref="poolTargetSource"/>
	<property name="interceptorNames" value="myInterceptor"/>
</bean>
```

请注意，目标对象（如前例中的 `businessObjectTarget` ）必须是 `prototype` 。这使得 `PoolingTargetSource` 的实现能够根据需要创建目标的新实例，从而扩展池。有关其属性的详细信息，请参阅 `AbstractPoolingTargetSource` 及其具体子类的 Javadoc。 `maxSize` 是最基本的属性，且始终存在。

在此示例中， `myInterceptor` 是一个拦截器的名称，该拦截器需要在同一个 IoC 上下文中进行定义。不过，使用池化功能时无需指定拦截器。若仅需池化功能而无需其他 `Advice` ，则完全不必设置 `interceptorNames` 属性。

可以配置 Spring，使其能够将任何池化对象强制转换为 `org.springframework.aop.target.PoolingConfig` 接口，该接口通过 `introduction` 机制公开了有关配置和当前池大小的信息。需要定义一个类似于以下示例的 `advisor` ：
```
<bean id="poolConfigAdvisor" class="org.springframework.beans.factory.config.MethodInvokingFactoryBean">
	<property name="targetObject" ref="poolTargetSource"/>
	<property name="targetMethod" value="getPoolingConfigMixin"/>
</bean>
```
该 `advisor` 是通过调用 `AbstractPoolingTargetSource` 类上的一个便捷方法获得的，因此使用了 `MethodInvokingFactoryBean` 。该 `Advisor` 的名称（此处为 `poolConfigAdvisor` ）必须出现在暴露该池化对象的 `ProxyFactoryBean` 的拦截器名称列表中。

```
PoolingConfig conf = (PoolingConfig) beanFactory.getBean("businessObject");
System.out.println("Max pool size is " + conf.getMaxSize());
```
通常没有必要对无状态服务对象进行池化。这不应作为默认选项，因为大多数无状态对象本身就是线程安全的，而且如果资源被缓存，实例池化会带来问题。

通过使用自动代理，可以实现更简便的池化。可以设置任何自动代理创建器所使用的 `TargetSource` 实现。

## Prototype Target Sources
配置 `prototype` 类型的 `TargetSource` 与配置池化 `TargetSource` 的原理相似。在这种情况下，每次调用方法时都会创建目标的新实例。尽管在现代 JVM 中创建新对象的开销并不高，但连接新对象（满足其 IoC 依赖关系）的开销可能更高。因此，除非有充分的理由，否则不应采用这种方法。
```
<bean id="prototypeTargetSource" class="org.springframework.aop.target.PrototypeTargetSource">
	<property name="targetBeanName" ref="businessObjectTarget"/>
</bean>
```
唯一的属性是目标 Bean 的名称。 `TargetSource` 的实现中使用了继承机制，以确保命名的一致性。与池化目标源一样，目标 Bean 必须是 `prototype` 类型。

## ThreadLocal Target Sources
如果需要为每个传入的请求（即每个线程）创建一个对象， `ThreadLocalTargetSource` 会非常有用。 `ThreadLocal` 概念提供了一种 JDK 范围内的机制，可透明地将资源与线程绑定在一起。设置 `ThreadLocalTargetSource` 的方法与其他类型的 `TargetSource` 基本相同，如下例所示：

```
<bean id="threadlocalTargetSource" class="org.springframework.aop.target.ThreadLocalTargetSource">
	<property name="targetBeanName" value="businessObjectTarget"/>
</bean>
```

如果在多线程和多类加载器环境中错误地使用 `ThreadLocal` 实例，可能会引发严重问题（可能导致内存泄漏）。应始终考虑将 `ThreadLocal` 封装在其他类中，切勿直接使用 `ThreadLocal` 本身（封装类中的情况除外）。此外，应始终记得正确设置和释放（其中释放操作需调用 `ThreadLocal.remove()` ）线程本地资源。无论何种情况都应进行释放操作，因为若不释放可能会导致异常行为。Spring 的 `ThreadLocal` 支持功能会为您自动处理这些操作，因此相较于直接使用 `ThreadLocal` 实例却未进行其他适当处理的情况，应优先考虑使用 Spring 的 `ThreadLocal` 支持。