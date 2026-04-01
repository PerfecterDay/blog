# Spring 精细化配置注解
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/beans/annotation-config/autowired-primary.html  
> https://docs.spring.io/spring-framework/reference/core/beans/annotation-config/autowired-qualifiers.html

## @Primary 和 @Fallback
由于按类型自动装配时，可能有多个 bean 属于同一个类型，此时需要告诉 Spring 如何筛选出对象。实现这一目标的一种方式是使用 Spring 的 `@Primary` 注解。当多个 Bean 作为候选对象被自动装配到单值依赖时， `@Primary` 注解表明应优先选择该特定 Bean。若候选对象中恰好存在一个 `@Primary` 标注的 Bean，则该 Bean 将成为自动装配的值。
```
@Configuration
public class MovieConfiguration {

	@Bean
	@Primary
	public MovieCatalog firstMovieCatalog() { ... }

	@Bean
	public MovieCatalog secondMovieCatalog() { ... }

	// ...
}
```

从6.2版本开始，引入了 `@Fallback` 注解用于标记除常规注入对象之外的其他Bean。若仅剩一个常规Bean，该Bean也将自动成为主注入对象：
```
@Configuration
public class MovieConfiguration {

	@Bean
	public MovieCatalog firstMovieCatalog() { ... }

	@Bean
	@Fallback
	public MovieCatalog secondMovieCatalog() { ... }

	// ...
}
```

上例中， `firstMovieCatalog` 会作为对象注入到其它 bean 中。

## @Qualifier
解决多个依赖注入问题时，除了给被依赖项的某项加上 `@Primary` 注解外，还可以使用 `@Qualifier/ @Named` 注解在依赖注入的地方以指示 Spring 进行筛选。有时候不能更改被依赖项的源码的时候，可以用这种方式。

甚至可以使用 `CustomAutowireConfigurer` 来实现自定义的 qualifier 逻辑：
```
<bean id="customAutowireConfigurer"
		class="org.springframework.beans.factory.annotation.CustomAutowireConfigurer">
	<property name="customQualifierTypes">
		<set>
			<value>example.CustomQualifier</value>
		</set>
	</property>
</bean>
```

限定符同样适用于类型化集合，如前所述—— `Set<MovieCatalog>` 。此时，所有符合声明限定符的bean将作为集合注入。这意味着限定符无需唯一，而是构成过滤条件。例如，可以定义多个具有相同限定符值“action”的 `MovieCatalog Bean` ，它们都将被注入到带有 `@Qualifier("action")` 注解的 `Set<MovieCatalog>` 中。

作为按名称注入的替代方案，可考虑使用JSR-250的 `@Resource` 注解。该注解在语义上被定义为通过唯一名称识别特定目标组件，声明的类型在匹配过程中无关紧要。而@Autowired注解的语义则截然不同：在按类型筛选候选bean后，仅在类型筛选出的候选对象范围内考虑指定的String限定符值（例如，将account限定符与标记相同限定符标签的bean进行匹配）。

对于集合、Map或数组类型的Bean， `@Resource` 是一个不错的解决方案，它通过唯一名称引用特定的集合或数组Bean。不过，只要元素类型信息在 `@Bean` 返回类型签名或集合继承层次结构中得到保留，也可以通过Spring的 `@Autowired` 类型匹配算法来匹配集合、Map和数组类型，通过限定符值在同类型集合中进行选择。

`@Autowired` 适用于字段、构造函数和多参数方法，并允许通过参数级别的 `@Qualifier` 注解进行限定。相比之下， `@Resource` 仅支持字段和单参数的 Bean `setter` 方法。因此，若注入目标是构造函数或多参数方法，则只能使用限定符 `@Qualifier` 。

## 范型注入
```
@Configuration
public class MyConfiguration {

	@Bean
	public StringStore stringStore() {
		return new StringStore();
	}

	@Bean
	public IntegerStore integerStore() {
		return new IntegerStore();
	}
}
```
假设上面的Bean实现了泛型接口（即 `Store<String>` 和 `Store<Integer>` ），则可以使用 `@Autowire` 注解注入 `Store` 接口，并将泛型作为限定符，如下例所示：
```
@Autowired
private Store<String> s1; // <String> qualifier, injects the stringStore bean

@Autowired
private Store<Integer> s2; // <Integer> qualifier, injects the integerStore bean


// Inject all Store beans as long as they have an <Integer> generic
// Store<String> beans will not appear in this list
@Autowired
private List<Store<Integer>> s;
```

## CustomAutowireConfigurer
`CustomAutowireConfigurer` 是一个 `BeanFactoryPostProcessor` ，它允许注册自己的自定义 `qualifier` 注解类型，即使这些类型未被 Spring 的 `@Qualifier` 注解标记。以下示例展示了如何使用 `CustomAutowireConfigurer` ：
```
<bean id="customAutowireConfigurer"
		class="org.springframework.beans.factory.annotation.CustomAutowireConfigurer">
	<property name="customQualifierTypes">
		<set>
			<value>example.CustomQualifier</value>
		</set>
	</property>
</bean>
```

`AutowireCandidateResolver` 通过以下方式确定被注入的对象：
1. `BeanDefinition` 的 `autowire-candidate` 属性
2. `<beans/>` 元素配置的 `default-autowire-candidates` 匹配模式
3. `@Qualifier` 注解以及任何通过 `CustomAutowireConfigurer` 注册的注解

当多个Bean符合依赖注入条件时，Bean的确定规则如下：若候选Bean中恰有一个Bean定义的 `primary` 属性设置为 `true` ，则选定该Bean。否则， Spring 会报错。