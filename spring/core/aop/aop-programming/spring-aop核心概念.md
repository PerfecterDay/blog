# AOP 核心概念及目标
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/aop/introduction-defn.html  
> https://docs.spring.io/spring-framework/reference/core/aop/introduction-spring-defn.html  
> https://docs.spring.io/spring-framework/reference/core/aop/introduction-proxies.html


## 核心概念
1. 切面- `Aspect`：对横跨多个类的关注点进行模块化处理。事务管理是企业Java应用中横切关注点的典型示例。在Spring AOP中，切面可通过常规类（基于模式的方法）或带有@Aspect注解的常规类（ `@AspectJ` 风格）实现。
2. 连接点-`Join point`：程序执行过程中的某个点，例如方法执行或异常处理。在Spring AOP中，连接点只能是方法执行点。
3. 增强-`Advice`：在特定连接点上由某个切面执行的操作。 `Advice` 类型包括"around", "before", and "after" 类型。包括Spring在内的许多AOP框架都将 `Advice` 建模为 `interceptor` ，并在连接点周围维护一个 `interceptor` 链。
4. 切点-`Pointcut`: 用于匹配连接点的**谓词表达式**。 `Advice` 与 `Pointcut` 表达式相关联，并在 `Pointcut` 匹配到的任何连接点（例如执行特定名称的方法）处运行。 `Pointcut` 表达式匹配到的连接点概念是AOP的核心，Spring默认使用 ` `AspectJ`  Pointcut` 表达式语言。
5. `Introduction` ：代表类型声明额外的方法或字段。Spring AOP允许为任何被切入对象引入新接口（及对应实现）。例如，可通过引入机制使Bean实现 `IsModified` 接口，从而简化缓存操作。（此引入机制在 `AspectJ` 社区中称为跨类型声明。）
6. 目标对象- `Target object` ：由一个或多个切面进行增强的对象。也称为 `advised object` 。由于Spring AOP通过运行时代理实现，该对象始终是**代理对象**。
7. AOP代理- `AOP proxy` ：由AOP框架创建的对象，用于实现切面契约（如 `Advice` 方法执行等）。在Spring框架中，AOP代理可以是JDK动态代理或CGLIB代理。
8. 织入- `Weaving` ：将切面与其他应用类型或对象关联，从而创建一个被切入对象。此操作可在编译时（例如使用 `AspectJ` 编译器）、加载时或运行时完成。Spring AOP与其他纯Java AOP框架类似，在运行时执行织入操作。

Spring AOP包含以下类型的 `Advice` ：
+ `Before advice` : 在连接点之前执行，但无法阻止执行流继续前进至连接点（除非抛出异常）。
+ `After returning advice` : 在连接点正常完成后执行（例如，当方法返回且未抛出异常时）。
+ `After throwing advice` : 如果方法通过抛出异常退出，则该方法将被运行。
+ `After (finally) advice` : 无论连接点以何种方式退出（正常返回或异常返回），该 `Advice` 都应执行。
+ `Around advice` : 围绕方法调用等连接点的 `advice` 。这是最强大的 `Advice` 类型。围绕 `Advice` 可在方法调用前后执行自定义行为，同时负责决定是否继续执行连接点操作，或通过返回自身返回值或抛出异常来跳过连接点方法的执行。

`Around advice` 是最通用的 `Advice` 类型。由于Spring AOP与 `AspectJ` 一样提供了完整的 `Advice` 类型体系，我们应该使用能够实现所需行为的最低权限 `Advice` 类型。例如，若仅需将方法返回值更新至缓存，采用 `after returning advice ` 比 `Around advice` 更优——尽管 `Around advice` 也能实现相同效果。采用最精确的 `Advice` 类型能简化编程模型并降低错误风险。例如，如果不使用 ``Around advice`， 就无需调用 `JoinPoint` 的 `proceed()` 方法，因此不会出现调用失败的情况。

切入点的匹配机制是AOP的核心概念，它使AOP区别于仅提供拦截功能的传统技术。切入点使 `Advice` 能够独立于面向对象的层次结构进行定位。例如，你可以将提供声明式事务管理的 `Around Advice` 应用于跨越多个对象的方法集（如服务层中的所有业务操作）。

## Spring AOP 的能力与目标
`Spring AOP` 采用纯Java实现，无需特殊编译过程。它无需控制类加载器层次结构，因此适用于Servlet容器或应用服务器环境。

`Spring AOP` 目前仅支持方法执行连接点（对Spring bean方法的执行进行 `advice` ）。字段拦截功能尚未实现，但如果要添加支持字段拦截的功能，是可以添加的，而且不用破坏Spring AOP核心API。若需对字段访问和更新进行拦截，请考虑使用 `AspectJ` 等语言。

 `Spring AOP`  的面向切面方法与大多数其他 AOP 框架不同。其目标并非提供最完整的 AOP 实现（尽管  `Spring AOP`  功能相当强大），而是致力于实现 AOP 实现与 Spring IoC 的紧密集成，从而帮助解决企业应用中的常见问题。

因此，Spring框架的AOP功能通常与Spring IoC容器配合使用。切面配置采用常规bean定义语法（尽管这提供了强大的"自动代理"能力）。这与其他AOP实现存在关键差异。使用 `Spring AOP` 无法轻松高效地完成某些操作，例如对精细粒度的对象（通常是领域对象）进行切面。在这种情况下， `AspectJ` 是最佳选择。但根据我们的经验，对于企业Java应用中适合AOP处理的大多数问题，Spring AOP都能提供出色的解决方案。

Spring AOP从未试图与 `AspectJ` 竞争以提供全面的AOP解决方案。无论是Spring AOP这类基于代理的框架，还是 `AspectJ` 这类完整的框架，都具有价值且互补而非竞争关系。Spring将 `Spring AOP` 与IoC无缝集成至 `AspectJ` ，使所有AOP应用都能在统一的Spring架构中运行。此集成不影响 `Spring AOP API` 或 `AOP Alliance API` ，Spring AOP保持向后兼容性。

Spring框架的核心理念之一是无侵入性。这意味着您不应被迫在业务模型或领域模型中引入框架专属的类和接口。然而在某些场景下，Spring框架确实提供了将框架专属依赖引入代码库的选择。提供这些选项的初衷在于：特定场景下，采用此类方式实现某些功能可能更易于阅读或编写代码。但Spring框架（几乎）始终赋予你选择权：你可以自由地根据具体用例或场景，做出明智的选择。

## AOP Proxies
Spring AOP默认使用标准JDK动态代理作为AOP代理。这使得任何接口（或接口集合）都能被代理。

Spring AOP 亦可使用 CGLIB 代理。当需要代理类而非接口时，此方案不可或缺。默认情况下，若业务对象未实现接口，系统将自动启用 CGLIB。鉴于面向接口而非面向类的编程实践更具优势，业务类通常会实现一个或多个业务接口。在某些场景下，可强制使用CGLIB代理：例如需要对未声明在接口上的方法添加 `advice` ，或需将代理对象作为具体类型传递给方法时。