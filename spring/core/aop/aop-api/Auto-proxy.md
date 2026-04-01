# 自动代理
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/aop-api/autoproxy.html

到目前为止，我们主要探讨了如何通过 `ProxyFactoryBean` 或类似的工厂 bean 显式创建 AOP 代理。

Spring 还允许我们使用“自动代理”Bean 定义，它能够自动对选定的 Bean 定义进行代理。该功能基于 Spring 的 `bean post processor` 架构，该架构允许在容器加载时修改任何 Bean 定义。

在此模型中，您需要在 XML Bean 定义文件中设置一些特殊的 Bean 定义来配置自动代理基础设施。这使您能够声明哪些目标符合自动代理的条件。无需使用  `ProxyFactoryBean`。

有两种方法能实现这个功能：
1. 通过使用一个自动代理生成器，该生成器会引用当前上下文中的特定 Bean。
2. 自动代理创建的一个特例，值得单独探讨：由源代码级元数据属性驱动的自动代理创建。

## Auto-proxy Bean Definitions
本节介绍了由 `org.springframework.aop.framework.autoproxy` 包提供的自动代理生成器。

### BeanNameAutoProxyCreator
`BeanNameAutoProxyCreator` 类是一个 `BeanPostProcessor` ，它会自动为名称与字面值或通配符匹配的 Bean 创建 AOP 代理。以下示例演示了如何创建一个 `BeanNameAutoProxyCreator` Bean：
```
<bean class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator">
	<property name="beanNames" value="jdk*,onlyJdk"/>
	<property name="interceptorNames">
		<list>
			<value>myInterceptor</value>
		</list>
	</property>
</bean>
```

与 `ProxyFactoryBean` 类似，这里使用的是 `interceptorNames` 属性，而非 `interceptors` 列表，以确保原型 `Advisor` 能正常工作。命名的“拦截器”可以是 `Advisor` ，也可以是任何类型的 `Advice` 。

与一般的自动代理机制类似，使用 `BeanNameAutoProxyCreator` 的主要目的是以最少的配置量，将相同的配置一致地应用于多个对象。这是将声明式事务应用于多个对象时的一种常见选择。

名称匹配的 Bean 定义（例如前例中的 `jdkMyBean` 和 `onlyJdk` ）是与目标类对应的普通 Bean 定义。 `BeanNameAutoProxyCreator` 会自动创建一个 AOP 代理。相同的 `Advice` 将应用于所有匹配的 Bean。请注意，如果使用 `Advisor` （而非前例中的拦截器），切入点对不同 Bean 的应用方式可能会有所不同。

### DefaultAdvisorAutoProxyCreator
一个更通用且功能极其强大的自动代理生成器是 `DefaultAdvisorAutoProxyCreator` 。它能自动在当前上下文中应用符合条件的 `Advisor` ，而无需在自动代理 `Advisor` 的 Bean 定义中包含具体的 Bean 名称。它与 BeanNameAutoProxyCreator 一样，具有配置一致且避免重复的优势。

1. 声明一个 `DefaultAdvisorAutoProxyCreator`  bean 定义。
2. 在相同或相关的上下文中指定任意数量的 `advisors` 。请注意，这些必须是 `advisors` ，而非 `interceptors` 或其他 `Advice` 。这是必要的，因为必须有一个切入点来进行评估，以检查每个 `Advice` 是否适用于候选 Bean 定义。

`DefaultAdvisorAutoProxyCreator` 会自动评估每个 `Advisor` 中包含的切入点，以确定应向每个业务对象（例如示例中的 `businessObject1` 和 `businessObject2` ）应用哪些（如有） `Advice` 。

这意味着可以为每个业务对象自动应用任意数量的 `Advisor` 。如果所有 `Advisor` 中的切入点均未与业务对象中的任何方法匹配，则该对象不会被代理。当为新的业务对象添加 Bean 定义时，系统会在必要时自动对其进行代理。

总体而言，自动代理的优势在于能够确保调用方或依赖项无法获取未经过 AOP 处理的对象。在此 `ApplicationContext` 上调用 `getBean("businessObject1")` 时，返回的是一个 AOP 代理，而非目标业务对象。（前面提到的“内部 Bean”模式也具备这一优势。）

以下示例创建了一个 `DefaultAdvisorAutoProxyCreator` Bean 以及本节中讨论的其他元素：
```
<bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"/>

<bean class="org.springframework.transaction.interceptor.TransactionAttributeSourceAdvisor">
	<property name="transactionInterceptor" ref="transactionInterceptor"/>
</bean>

<bean id="customAdvisor" class="com.mycompany.MyAdvisor"/>

<bean id="businessObject1" class="com.mycompany.BusinessObject1">
	<!-- Properties omitted -->
</bean>

<bean id="businessObject2" class="com.mycompany.BusinessObject2"/>
```

如果需要将相同的 `Advice` 一致地应用于多个业务对象， `DefaultAdvisorAutoProxyCreator` 非常有用。一旦基础设施定义就位，就可以添加新的业务对象，而无需包含具体的代理配置。此外，还可以轻松地添加其他切面（例如，跟踪或性能监控切面），而只需对配置进行微小的更改。

`DefaultAdvisorAutoProxyCreator` 支持过滤（通过命名约定仅评估特定的 `Advisor` ，从而允许在同一个工厂中使用多个配置不同的 `AdvisorAutoProxyCreator` ）和排序。如果排序是关键问题， `Advisor` 可以实现 `org.springframework.core.Ordered` 接口以确保正确的排序顺序。前例中使用的 `TransactionAttributeSourceAdvisor` 具有可配置的排序值。默认设置为无序。

