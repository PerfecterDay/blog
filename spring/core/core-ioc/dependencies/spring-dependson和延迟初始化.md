# Depends-on 与延迟初始化
{docsify-updated}

如果一个 Bean 是另一个 Bean 的依赖项，通过基于 XML 的元数据中的 **`<ref/>`** 元素或**自动装配**来实现此功能。

然而，有时 Bean 之间的依赖关系并不直接。例如当需要触发类中的静态初始化器时（如数据库驱动程序注册），便可通过 `depends-on` 属性或 `@DependsOn` 注解显式强制要求一个或多个 Bean 在使用该元素的 Bean 初始化之前完成初始化。下例使用 `depends-on` 属性表达对单个 Bean 的依赖：

```
<bean id="beanOne" class="ExampleBean" depends-on="manager,accountDao">
	<property name="manager" ref="manager" />
</bean>

<bean id="manager" class="ManagerBean" />
<bean id="accountDao" class="x.y.jdbc.JdbcAccountDao" />
```

## Lazy-initialized
默认情况下， `ApplicationContext` 实现会在初始化过程中立即创建并配置所有单例 bean。通常这种预实例化是可取的，因为配置或环境中的错误能立即被发现，而非数小时甚至数天后才暴露。当不想预先创建 bean 时，可通过将 bean 定义标记为**延迟初始化**来阻止单例 bean 的预实例化。懒加载的 Bean 指示 IoC 容器在首次请求时才创建 Bean 实例，而非在启动时创建。

可以使用过 `@Lazy` 注解控制，或在XML中通过 `<bean/>` 元素的 `lazy-init` 属性控制，如下例所示：
```
@Bean
@Lazy
ExpensiveToCreateBean lazy() {
	return new ExpensiveToCreateBean();
}

@Bean
AnotherBean notLazy() {
	return new AnotherBean();
}
```

然而，当一个延迟初始化的Bean是某个非延迟初始化单例Bean的依赖项时， `ApplicationContext` 依然会在启动时创建该延迟初始化Bean，因为它必须满足单例的依赖关系。该延迟初始化Bean会被注入到其他非延迟初始化的单例Bean中。

如果我们希望能同时定义一组延迟初始化的bean，可以通过在标注为 `@Configuration` 的类上使用 `@Lazy` 注解，或在XML中为 `<beans/>` 元素添加 `default-lazy-init` 属性，，如下例所示：
```
@Configuration
@Lazy
public class LazyConfiguration {
	// No bean will be pre-instantiated...
}
```
这样，这个配置类或者xml 配置文件中声明的 bean 都会延迟初始化。