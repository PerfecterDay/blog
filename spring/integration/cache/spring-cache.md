#  Spring Cache的集成
{docsify-updated}
> https://docs.spring.io/spring-framework/reference/integration/cache/annotations.html

对于缓存声明，Spring 的 `spring-cache` 提供了一组 Java 注解：
+ `@Cacheable` ：触发缓存操作。
+ `@CacheEvict` ：触发缓存删除。
+ `@CachePut` ：在不干扰方法执行的情况下更新缓存。
+ `@Caching` ：将多个缓存操作重新分组，应用于某个方法。
+ `@CacheConfig` ：在类级别共享一些常见的缓存相关设置。


## @Cacheable 
顾名思义，可以使用 `@Cacheable` 标注可缓存的方法——即其结果会被存储在缓存中的方法，这样在后续调用（使用相同参数）时，系统会直接返回缓存中的值，而无需实际调用该方法。在最简单的形式下，该注解的声明需要指定与被注解方法关联的缓存名称，如下例所示：
```
@Cacheable("books")
public Book findBook(ISBN isbn) {...}
```
在上面的代码片段中， `findBook` 方法与名为 `books` 的缓存相关联。每次调用该方法时，都会检查缓存以确认该调用是否已执行过，从而避免重复执行。虽然在大多数情况下只声明一个缓存，但该注解允许指定多个名称，以便使用多个缓存。在这种情况下，在调用方法之前会检查每个缓存——如果至少有一个缓存命中，则返回关联的值。
```
@Cacheable({"books", "isbns"})
public Book findBook(ISBN isbn) {...}
```
注意：在多个缓存的情况下，如果只有某一个缓存命中，所有其他不包含该值的缓存也会被更新，即使缓存的方法实际上并未被调用。

### Default Key Generation
由于缓存本质上是键值对存储，因此每次调用缓存方法时，都需要将其转换为适合缓存访问的键。 `spring-cache` 使用了一个基于以下算法的简单 `KeyGenerator` ：
+ 如果未提供参数，则返回 `SimpleKey.EMPTY` 。
+ 如果只给定一个参数，则返回该实例。
+ 如果给定多个参数，则返回一个包含所有参数的 `SimpleKey` 对象。

只要参数具有自然键，并且实现了有效的 `hashCode()` 和 `equals()` 方法，这种方法对大多数用例都效果良好。如果不符合这些条件，则需要更改策略。

若要提供一个不同的默认键生成器，需要实现 `org.springframework.cache.interceptor.KeyGenerator` 接口。

### Custom Key Generation Declaration
由于缓存是通用的，目标方法很可能具有多种不同的签名，这些签名无法直接映射到缓存结构上。当目标方法有多个参数，其中只有部分参数适合缓存（其余参数仅用于方法逻辑）时，这种情况往往尤为明显。请看以下示例：
```
@Cacheable("books")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)
```
乍一看，虽然这两个布尔参数会影响查找书籍的方式，但它们对缓存毫无用处。此外，如果其中一个参数很重要，而另一个并不重要，又该怎么办？

对于此类情况， `@Cacheable` 注解允许通过其 `key` 属性指定键的生成方式。可以使用 `SpEL` 筛选所需的参数（或其嵌套属性）、执行操作，甚至调用任意方法，而无需编写任何代码或实现任何接口。与默认生成器相比，这是更推荐的做法，因为随着代码库的增长，方法的签名往往会大不相同。虽然默认策略可能对某些方法有效，但很少能适用于所有方法。
```
@Cacheable(cacheNames="books", key="#isbn")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)

@Cacheable(cacheNames="books", key="#isbn.rawNumber")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)

@Cacheable(cacheNames="books", key="T(someType).hash(#isbn)")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)
```

如果负责生成键值的算法过于特定，或者需要在多处共享键生成算法，则可以在操作中定义一个自定义的 `keyGenerator` 。要实现这一点，只需指定要使用的 `KeyGenerator` Bean 的名称，如下例所示：
```
@Cacheable(cacheNames="books", keyGenerator="myKeyGenerator")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)
```

注意： `key` 和 `keyGenerator` 参数互斥，若同时指定这两个参数，将引发异常。

### 默认缓存解析
 `spring-cache` 使用一个简单的 `CacheResolver` ，该组件通过配置好的 `CacheManager` 来检索在操作级别定义的缓存。

若要提供一个不同的默认缓存解析器，需要实现 `org.springframework.cache.interceptor.CacheResolver` 接口。

### 自定义缓存解析
默认的缓存解析方式非常适合那些仅使用单个 `CacheManager` 且没有复杂缓存解析需求的应用程序。对于需要与多个 `CacheManager` 配合使用的应用程序，可以为每项操作设置要使用的缓存管理器，如下例所示：
```
@Cacheable(cacheNames="books", cacheManager="anotherCacheManager")
public Book findBook(ISBN isbn) {...}
```

还可以采用与替换 `KeyGenerator` 类似的方式，完全替换 `CacheResolver` 。每次缓存操作都会进行动态实时的缓存解析，从而让实现能够根据运行时参数来实际确定应使用的缓存。以下示例展示了如何指定 `CacheResolver` ：
```
@Cacheable(cacheResolver="runtimeCacheResolver")
public Book findBook(ISBN isbn) {...}
```

与 `key` 和 `keyGenerator` 类似， `cacheManager` 和 `cacheResolver` 参数也是互斥的，如果同时指定这两个参数，将会引发异常，因为自定义的 `CacheManager` 会被 `CacheResolver` 的实现忽略。这可能并非所期望的结果。

### 缓存同步
在多线程环境中，某些操作可能会针对同一参数被并发调用（通常发生在启动时）。默认情况下， `spring-cache` 层不会进行任何锁定，因此同一值可能会被计算多次，从而违背了缓存的初衷。

对于此类特殊情况，可以使用 `sync` 属性，指示底层缓存实现在计算值期间锁定缓存条目。这样一来，只有一个线程在计算值，而其他线程则会被阻塞，直到缓存中的条目更新为止。以下示例演示了如何使用 `sync` 属性：
```
@Cacheable(cacheNames="foos", sync=true)
public Foo executeExpensiveOperation(String id) {...}
```
注意： 这是一项可选功能，常用的缓存库可能不支持此功能。核心框架提供的所有 `CacheManager` 实现均支持此功能。更多详细信息，请参阅缓存提供商的文档。

### Caching with CompletableFuture and Reactive Return Types

### 条件缓存 
有时，某个方法可能并不总是适合进行缓存（例如，它可能取决于给定的参数）。缓存注解通过 `condition` 参数支持此类用例，该参数接受一个 `SpEL` 表达式，其求值结果为 `true` 或 `false` 。如果结果为 `true` ，则该方法会被缓存；否则，其行为就如同未被缓存一样（即无论缓存中存有何值或使用了何种参数，该方法都会被每次调用）。例如，只有当参数 `name` 的长度小于 `32` 时，以下方法才会被缓存：
```
@Cacheable(cacheNames="book", condition="#name.length() < 32")
public Book findBook(String name)
```

除了 `condition` 参数外，你还可以使用 `unless` 参数来阻止将某个值添加到缓存中。与 `condition` 不同， `unless` 表达式是在方法调用之后才进行求值的。以之前的示例为例，我们可能只想缓存平装书（ `paperback` ），如下例所示：
```
@Cacheable(cacheNames="book", condition="#name.length() < 32", unless="#result.hardback")
public Book findBook(String name)
```
在这种情况下，即使 `#name.length() < 32` 为 `true` ， 如果返回对象的 `#result.hardback` 为 `true` ，也不会缓存。

 `spring-cache` 支持 `java.util.Optional` 返回类型。如果存在 `Optional` 值，则将其存储在关联的缓存中；如果不存在 `Optional` 值，则将 `null` 存储在关联的缓存中。`#result` 始终指代业务实体，绝不指代受支持的包装器，因此前面的示例可以重写为：
```
@Cacheable(cacheNames="book", condition="#name.length() < 32", unless="#result?.hardback")
public Optional<Book> findBook(String name)
```
请注意， `#result` 仍然指向 `Book` ，而不是 `Optional<Book>` 。由于它可能为空，因此我们使用了 SpEL 的安全导航运算符 `?.` 。

### Available Caching SpEL Evaluation Context
每个 SpEL 表达式都会在专用的上下文中进行求值。除了内置参数外，该框架还提供了与缓存相关的专用元数据，例如参数名称。下表列出了上下文中可用的项目，以便您将其用于键值和条件计算：
<center><img src="pics/cache-spel.png" width="80%"></center>

## @CachePut  
当需要在不干扰方法执行的情况下更新缓存时，可以使用 `@CachePut` 注解。也就是说，该方法会始终被调用，其结果将根据 `@CachePut` 的配置选项存入缓存。它支持与 `@Cacheable` 相同的配置选项，应用于缓存填充，而非方法执行流的优化。以下示例使用了 `@CachePut` 注解：
```
@CachePut(cacheNames="book", key="#isbn")
public Book updateBook(ISBN isbn, BookDescriptor descriptor)
```

注意⚠️：通常强烈建议不要在同一个方法上同时使用 `@CachePut` 和 `@Cacheable` 注解，因为它们的行为不同。后者会利用缓存跳过方法调用，而前者则会强制执行调用以更新缓存。这会导致意外的行为，因此除了一些特定的特殊情况（例如注解的条件相互排斥）外，应避免此类声明。另请注意，此类条件不应依赖于结果对象（即 `#result` 变量），因为这些条件会在前期进行验证以确认排除关系。

从 6.1 版本开始， `@CachePut` 支持 `CompletableFuture` 和 `reactive` 返回类型，并在生成的对象可用时执行存储操作。

## @CacheEvict
 `spring-cache` 不仅支持缓存存储的填充，还支持缓存的清除。此过程有助于从缓存中移除过期或未使用的数据。与 `@Cacheable` 不同， `@CacheEvict` 用于标记执行缓存清除的方法（即标记一个方法作为触发器用来从缓存中移除数据）。与它的“兄弟”注解类似， `@CacheEvict` 要求指定一个或多个受此操作影响的缓存，允许指定自定义的缓存和键解析规则或条件，并提供了一个额外参数（ `allEntries` ），用于指示是否需要执行全缓存驱逐（而非仅基于键的条目驱逐）。以下示例将从 `books` 缓存中驱逐所有条目：
```
@CacheEvict(cacheNames="books", allEntries=true)
public void loadBooks(InputStream batch)
```
当需要清空整个缓存区域时，此选项非常实用。与逐个驱逐条目（由于效率低下，这将耗费大量时间）相比，如前例所示，所有条目都可通过一次操作移除。请注意，在此情况下，框架会忽略任何指定的键，因为该操作不适用（整个缓存都会被清空，而不仅仅是一个条目）。

您还可以通过使用 `beforeInvocation` 属性来指定驱逐操作应在方法调用之后（默认）还是之前进行。前者与其他注解的语义相同：一旦方法成功完成，就会对缓存执行一项操作（在此情况下为驱逐）。如果方法未执行（因为它可能已被缓存）或抛出了异常，则不会发生驱逐。后者（ `beforeInvocation=true` ）会导致驱逐操作总是在方法调用之前执行。当驱逐操作无需与方法结果挂钩时，此设置非常有用。

请注意， `void` 方法也可以与 `@CacheEvict` 一起使用——因为这些方法只是充当触发器，其返回值会被忽略（因为它们不会与缓存交互）。而 `@Cacheable` 则不同，它会向缓存中添加数据或更新缓存中的数据，因此需要返回结果。

从 6.1 版本开始， `@CachePut` 支持 `CompletableFuture` 和 `reactive` 返回类型，并在处理完成后执行调用后的驱逐操作。

## @Caching

## @CacheConfig

## Using Custom Annotations
