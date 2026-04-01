# Autowiring
{docsify-updated}

Spring容器能够自动连接协作Bean之间的关系。通过检查 `ApplicationContext` 的内容， Spring自动为Bean解析协作/依赖对象（其他Bean）。自动连接具有以下优势：
+ 自动连接能显著减少指定属性或构造函数参数的需求。
+ 自动装配能在对象演进过程中自动更新配置。例如，当需要为某个类添加依赖时，该依赖可自动满足，无需手动修改配置。因此自动装配在开发阶段尤为实用，同时不排除在代码库趋于稳定后切换为显式装配的选项。

使用基于XML的配置元数据时，可通过 `<bean/>` 元素的 `autowire` 属性为bean定义指定自动注入模式。自动注入功能包含四种模式。您可针对每个bean单独指定自动注入模式，从而选择需要自动注入的对象。下表描述了四种自动注入模式：

**Table 1. Autowiring modes**
| Mode        | Explanation          |
| ------------| ------------------------ |
| no          | 不使用autowiring， bean 之间的引用使用 ref 元素来显式注入  |
| byName      | 通过属性名称进行自动连接。Spring会寻找与需要自动连接的**属性名称**相同的Bean。例如，如果某个Bean定义设置为按名称自动连接，且包含一个 `master` 属性（即具有 `setMaster(..)` 方法），Spring就会查找名为 `master` 的Bean定义，并用其来设置该属性。             |
| byType      |当容器中恰好存在一个该属性的**类型**对应的Bean时，允许该属性被自动注入。若存在多个匹配的Bean，则抛出致命异常，这表明不能对该Bean使用 `byType` 自动注入。若不存在匹配的Bean，则不执行任何操作（该属性不会被设置）。    |
| constructor | 类似于byType，但适用于构造函数参数。如果容器中没有恰好一个构造函数参数类型的Bean，则会引发致命错误。  |


在使用 `byType` 或 `constructor` 自动装配模式时，可以装配数组和集合。此时，容器中所有符合预期类型的自动装配候选项都会被提供以满足依赖关系。若预期键类型为 String，则可自动装配强类型的 Map 实例。自动装配的 Map 实例的值包含所有符合预期类型的 Bean 实例，而 Map 实例的键则包含对应的 Bean 名称。


## Autowiring 的缺点
Autowiring 在整个项目中都保持一致使用时效果最佳。若项目整体未使用 Autowiring ，不适合仅为一两个Bean定义使用 Autowiring 。

Autowiring 的局限性和缺点：
1. `property` 与 `constructor-arg` 的显式依赖关系始终覆盖 Autowiring 。无法对基本类型、字符串、 `Classe`（及其数组）等简单属性进行自动装配。
2. Autowiring 的精确度低于显式装配。尽管如前文表格所述，Spring 会谨慎避免在可能产生意外结果的模糊场景中进行猜测。Spring 管理对象之间的关联关系不再被显式记录。
3. 使用 Autowiring 时， 容器生成文档的工具可能无法获取装配信息生成文档。
4. 容器中可能存在多个与自动装配的设置器方法或构造函数参数类型匹配的 Bean 定义。对于数组、集合或 Map 实例，这通常不成问题。但对于期望单一值的依赖项，此类歧义不会被随意解决。若无法找到唯一的 Bean 定义，系统将抛出异常。

在第4种情况下，你有以下几种选择：
+ 放弃 Autowiring ，转而采用显式装配。
+ 通过将 bean 定义的 `autowire-candidate` 属性设置为 `false` 来避免 Autowiring。
+ 通过将 `<bean/>` 元素的 `primary` 属性设置为 `true` ，指定单个 bean 作为主要候选对象。
+ 使用基于注解配置提供的更精细控制。

## 排除 Autowiring
在单个 Bean 层面上，可以将某个 Bean 排除在自动装配之外。在 Spring 的 XML 格式中，将 `<bean/>` 元素的 `autowire-candidate` 属性设置为 `false` ；使用 `@Bean` 注解时，该属性名为 `autowireCandidate` 。容器会将该特定 Bean 定义从自动装配基础设施中移除，包括基于注解的注入点（如 `@Autowired` ）。

`autowire-candidate` 属性仅影响 `byType` 的 Autowiring ，不会影响显式 `byName` 的引用。 即使指定的Bean 的 `autowire-candidate=false` ，byName Autoring 在名称匹配时仍会注入Bean。


还可以通过与 Bean 名称进行模式匹配来限制自动注入候选对象。 `<beans/>` 元素在其 `default-autowire-candidates` 属性中接受一个或多个模式。例如，若要将自动注入候选资格限制为名称以 `Repository` 结尾的 Bean， 将模式设置为 `*Repository` 。若需指定多个模式，以逗号分隔的形式列出。若在 Bean 定义中显式设置 `autowire-candidate` 属性为 `true` 或 `false` ，将会覆盖模式匹配的设置。

以上设置只会告诉 Spring 不要将当前 Bean 自动注入到其它 Bean 中， 但是该 Bean 本身的依赖 Bean 是可以被 Spring Autowiring 的。

从6.2版本起， `@Bean` 支持两种自动注入候选标志： `autowireCandidate` 和 `defaultCandidate` 。
+ 使用修饰符时，标记为 `defaultCandidate=false` 的Bean仅在使用了 `@qualifier` 注解饮用时才会使用。  
+ 相反， `autowireCandidate=false` 的行为完全符合上述 `autowire-candidate` 属性的说明：此类Bean将完全无法通过类型注入。