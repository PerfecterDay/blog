# 依赖注入
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html

依赖注入（DI）是一种过程，其中对象通过:
+ 构造函数参数
+ 工厂方法参数
+ 设置的属性

来定义其依赖项（即与其协同工作的其他对象）。

容器在创建Bean时注入这些依赖项。该过程本质上避免了 Bean 自身控制依赖项实例化或定位行为，容器会自动注入依赖对象。采用依赖注入原则能使代码更简洁，当对象通过依赖项获取所需资源时，解耦效果更为显著。对象无需主动查找依赖项，也不必知晓依赖项的位置或具体类。由此，类变得更易于测试——尤其当依赖项基于接口或抽象基类时，单元测试中便可灵活运用存根或模拟实现。

DI主要存在两种变体：
+ 基于构造函数的依赖注入
+ 基于Setter的依赖注入


## 构造函数依赖注入
基于构造函数的依赖注入是通过容器调用构造函数实现的，该构造函数接受多个参数，每个参数代表一个依赖对象。当使用特定参数调用静态工厂方法来构造Bean时，也是如此，静态工厂方法的参数代表依赖对象。

构造函数参数解析匹配是通过参数类型实现的。如果在 Bean 定义中构造函数参数不存在潜在歧义，则 Bean 定义中构造函数参数的定义顺序即为 Bean 实例化时向相应构造函数传递参数的顺序。请考虑以下类：
```
public class ThingOne {

	public ThingOne(ThingTwo thingTwo, ThingThree thingThree) {
		// ...
	}
}

<beans>
	<bean id="beanOne" class="x.y.ThingOne">
		<constructor-arg ref="beanTwo"/>
		<constructor-arg ref="beanThree"/>
	</bean>

	<bean id="beanTwo" class="x.y.ThingTwo"/>
	<bean id="beanThree" class="x.y.ThingThree"/>
</beans>

```

但是，有时候 Spring 需要我们提供一些额外信息来帮助 Spring 匹配正确的参数：
```
public class ExampleBean {

	// Number of years to calculate the Ultimate Answer
	private final int years;

	// The Answer to Life, the Universe, and Everything
	private final String ultimateAnswer;

	public ExampleBean(int years, String ultimateAnswer) {
		this.years = years;
		this.ultimateAnswer = ultimateAnswer;
	}
}
```

在 XML 中，我们需要明确指定类型， Spring 才知道将参数传递给哪个
```
<bean id="exampleBean" class="examples.ExampleBean">
	<constructor-arg type="int" value="7500000"/>
	<constructor-arg type="java.lang.String" value="42"/>
</bean>
```

也可以使用 `index` 来指明：
```
<bean id="exampleBean" class="examples.ExampleBean">
	<constructor-arg index="0" value="7500000"/>
	<constructor-arg index="1" value="42"/>
</bean>
```

或者直接指定参数名：
```
<bean id="exampleBean" class="examples.ExampleBean">
	<constructor-arg name="years" value="7500000"/>
	<constructor-arg name="ultimateAnswer" value="42"/>
</bean>
```

请注意，使用参数名方式时，代码必须启用 `-parameters` 标志进行编译，以便 Spring 能从构造函数中查找参数名称。若无法或不愿使用 `-parameters` 标志编译代码，可通过 `@ConstructorProperties` JDK 注解显式命名构造函数参数。此时示例类应如下所示：
```
public class ExampleBean {
	@ConstructorProperties({"years", "ultimateAnswer"})
	public ExampleBean(int years, String ultimateAnswer) {
		this.years = years;
		this.ultimateAnswer = ultimateAnswer;
	}
}
```

## 基于 Setter 的依赖注入
如果使用基于 `Setter` 的依赖注入（DI），容器在调用无参构造函数或无参静态工厂方法实例化你的Bean后，会调用该Bean的 `Setter` 方法来注入依赖。
比如下边这个Bean：
```
public class SimpleMovieLister {
	private MovieFinder movieFinder;
	private int integerProperty;

	public void setMovieFinder(MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}
}


<bean id="exampleBean" class="examples.ExampleBean">
	<!-- setter injection using the neater ref attribute -->
	<property name="movieFinder" ref="movieFinder"/>
	<property name="integerProperty" value="1"/>
</bean>

<bean id="movieFinder" class="examples.MovieFinder"/>
```

## 基于构造函数的注入还是基于Setter 的注入
由于可以混合使用基于构造函数和基于设置器的依赖注入，因此一个经验法则是：**将构造函数用于必需依赖项，而将设置器方法或配置方法用于可选依赖项。**请注意，在设置器方法上使用 `@Autowired` 注解可使该属性成为必需依赖项；但更推荐采用构造函数注入并配合参数的程序化验证。

Spring团队通常倡导**构造函数注入**，因为它允许将应用程序组件实现为不可变对象，并确保所需依赖项不会为空。此外，通过构造函数注入的组件始终以完全初始化的状态返回给客户端（调用方）代码。值得注意的是，过多的构造函数参数是一种不良代码气味，暗示该类可能承担了过多职责，应进行重构以更好地实现关注点分离。

为特定类选择最合适的DI风格。有时在处理第三方类时（尤其当你无法获取其源代码时），选择权往往不在你手中。例如，若第三方类未暴露任何设置器方法，构造函数注入可能成为唯一可用的DI形式。

## 依赖解析的过程
Spring 容器按以下方式执行 Bean 依赖解析：
1. 使用描述所有Bean的配置元数据创建并初始化`ApplicationContext`，配置元数据可通过XML、Java代码或注解进行指定。
2. 对于每个Bean，其依赖关系以属性、构造函数参数或静态工厂方法参数的形式表达（若使用静态工厂方法替代常规构造函数）。这些依赖项在Bean实际创建时，由Spring容器提供给该Bean。
3. 每个属性或构造函数参数要么是一个具体确定的值，要么是容器中另一个 Bean 的引用。
4. 如果要为一个Bean的属性或者构造函数参数提供一个具体的值，Spring 可以根据参数的实际类型将提供的字符串转换为参数的具体类型值。默认情况下，Spring 支持所有基本类型的转换：将字符串转换为 `int`, `long`, `String`, `boolean` 等。

Spring容器在创建过程中会验证每个Bean的配置。然而，Bean属性本身直到Bean实际创建时才会被设置。具有单例作用域且被设置为预实例化（默认预实例化）的Bean会在容器创建时被创建。否则，bean 仅在被请求时才创建。bean 的创建可能引发 bean 依赖图的生成，因为该 bean 的依赖项及其依赖项的依赖项（依此类推）都会被创建并分配。需注意，这些依赖项间的解析冲突可能延迟显现——即在受影响的 bean 首次创建时才暴露。

通常可以相信Spring会正确处理事务。它能在容器加载时检测配置问题，例如对不存在的Bean的引用和循环依赖。Spring会在尽可能晚的阶段设置属性并解析依赖关系——即在bean实际创建时。这意味着当Spring容器正确加载后，后续请求对象时仍可能抛出异常（例如因属性缺失/无效导致bean抛出异常）。正是这种潜在的配置问题延迟暴露特性，使得 `ApplicationContext` 默认预先实例化单例bean。虽然需要提前消耗时间和内存来创建这些尚未被使用的单例 bean，但这种方式能确保配置问题在 `ApplicationContext` 创建时就被发现，而非延迟暴露。可以覆盖此默认行为，使单例 bean 采用延迟初始化而非预先实例化的方式。

若不存在循环依赖，当一个或多个协作 Bean 被注入到依赖 Bean 时，每个协作 Bean 都会在注入到依赖 Bean 之前完成全部配置。这意味着，若bean A依赖于bean B，Spring IoC容器会在调用bean A的setter方法前完成对bean B的完整配置。具体而言，系统会实例化该bean B（若其非预实例化的单例），设置其依赖项，并调用相关生命周期方法（如配置的 `init` 方法或 `InitializingBean` 回调方法）。

### 循环依赖
假设： 类 A 通过构造函数注入需要类 B 的实例，而类 B 同样通过构造函数注入需要类 A 的实例。若将 A 和 B 配置为相互注入，Spring IoC 容器将在运行时检测到此循环引用，并抛出 `BeanCurrentlyInCreationException` 异常。

一种可能的解决方案是修改某些类的源代码，使其通过 `Setter` 而非构造函数进行注入。或者，避免使用构造函数注入，仅采用 `Setter` 注入。换言之，尽管不推荐这样做，但你可以通过设置器注入来配置循环依赖关系。

当bean A与bean B之间存在循环依赖时，其中一个bean必须在自身完全初始化之前就被注入到另一个bean中（这正是典型的先有鸡还是先有蛋的困境）。