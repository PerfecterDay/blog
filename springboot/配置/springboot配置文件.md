#  Springboot 配置文件
{docsify-updated}

> https://docs.spring.io/spring-boot/reference/features/external-config.html#features.external-config.files

Spring Boot 允许您将配置外部化，这样您就可以在不同的环境中使用相同的应用程序代码。您可以使用各种外部配置源，包括 Java 属性文件、YAML 文件、环境变量和命令行参数。  
属性值可以通过 `@Value` 注解直接注入到您的 Bean 中，也可以通过 Spring 的环境抽象访问，还可以通过 `@ConfigurationProperties` 绑定属性到结构化对象。  

Spring Boot 使用一种非常特殊的 `PropertySource` 顺序，旨在允许对值进行合理的覆盖。后面的属性源可以覆盖前面属性源中定义的值。属性源按以下顺序考虑：
1. 默认属性（通过设置 `SpringApplication.setDefaultProperties(Map)` 指定）。
2. 在 `@Configuration` 类上添加 `@PropertySource` 注解。请注意，在应用程序上下文刷新之前，此类属性源不会添加到环境中。这对于配置某些属性（如 `logging.*` 和 `spring.main.*`）来说为时已晚，因为这些属性是在刷新开始前读取的。
3. 配置数据文件（如 `application.properties` 文件）。
4. `RandomValuePropertySource` 只包含 `random.*` 中的属性。
5. 操作系统环境变量。
6. Java 系统属性（`System.getProperties()`）。
7. 来自 `java:comp/env` 的 JNDI 属性。
8. `ServletContext` 初始参数
9. `ServletConfig` 初始参数
10. 来自 `SPRING_APPLICATION_JSON` 的属性（嵌入到环境变量或系统属性中的内联 JSON）。
11. 命令行参数。
12. 测试属性。在 `@SpringBootTest` 和测试注解中可用，用于测试应用程序的特定片段。
13. 在测试中使用 `@DynamicPropertySource` 注解。
14. 测试中的 `@TestPropertySource` 注解。
15. Devtools 激活时，`$HOME/.config/spring-boot` 目录中的 Devtools 全局设置属性。

配置数据文件的审议顺序如下：
1. 打包在 jar 中的应用程序属性（`application.properties` 和 YAML 变体）。
2. 打包在 jar 中的特定于配置文件的应用程序属性（`application-{profile}.properties` 和 YAML 变体）。
3. 打包在 jar 之外的应用程序属性（`application.properties` 和 YAML 变体）。
4. 打包的 jar 之外的特定配置文件应用程序属性（`application-{profile}.properties` 和 YAML 变体）。

优先级从上到下依次递增。

**主工程的配置文件优先级更高，通常会“覆盖”依赖 jar 包中的配置。**

## 访问命令行属性
默认情况下，SpringApplication 会将任何命令行选项参数（即以 `--` 开头的参数，如 `--server.port=9000`）转换为属性，并将其添加到 Spring `Environment` 中。如前所述，**命令行属性总是优先于基于文件的属性源。**

如果不想将命令行属性添加到环境中，可以使用 `SpringApplication.setAddCommandLineProperties(false)` 将其禁用。

## JSON 应用程序属性
环境变量和系统属性通常有一些限制，意味着某些属性名称不能使用。为了帮助解决这个问题，Spring Boot 允许您将一组属性编码为单一的 JSON 结构。当应用程序启动时，`spring.application.json` 或 `SPRING_APPLICATION_JSON` 属性将被解析并添加到 `Environment` 中。

例如， `SPRING_APPLICATION_JSON` 属性可以作为环境变量或者系统属性在 UN*X shell 的命令行中提供：
```
$ SPRING_APPLICATION_JSON='{"my":{"name":"test"}}' java -jar myapp.jar //环境变量
$ java -Dspring.application.json='{"my":{"name":"test"}}' -jar myapp.jar //系统属性
$ java -jar myapp.jar --spring.application.json='{"my":{"name":"test"}}' //命令行参数
```

## 外部应用程序属性
Spring Boot 会在应用程序启动时自动从以下位置查找并加载 `application.properties` 和 `application.yaml` 文件：
1. 从 classpath
   1. classpath 根目录
   2. classpath /config 包

2. 从当前目录
   1. 当前目录
   2. 当前目录中的 config/ 子目录
   3. config/ 子目录的即时子目录

列表按优先级排序，从上到下优先级依次递增。加载文件中的文档将作为 `PropertySource` 实例添加到 Spring `Environment` 中。如果不喜欢使用 `application` 作为配置文件名，可以通过指定 `spring.config.name` 环境属性切换到其他文件名。例如，要查找 `myproject.properties` 和 `myproject.yaml` 文件，可以按如下方式运行应用程序：
```
$ java -jar myproject.jar --spring.config.name=myproject
```

还可以使用 `spring.config.location` 环境属性来引用指定路径位置的文件。该属性接受一个以逗号分隔的列表，其中包含一个或多个要检查的位置：
```
$ java -jar myproject.jar --spring.config.location=\
	optional:classpath:/default.properties,\
	optional:classpath:/override.properties
```
如果 `spring.config.location` 指定的是一个目录（而非文件），则应以 `/` 结尾，这时配置文件会由目录和 `spring.config.name` 指定的文件名同时确定。如果 `spring.config.location` 指定的是文件， 文件将被直接导入。  
在大多数情况下，你添加的每个 `spring.config.location` 项都将引用一个文件或目录。配置文件按照定义的顺序进行处理，后面的位置可以覆盖前面位置的值。

如果指定的配置文件不存在时允许使用默认配置，要使用前缀 `optional:` 。 

> 注意：`spring.config.name`、`spring.config.location` 和 `spring.config.additional-location` 用于确定需要加载的文件。它们必须定义为环境属性（通常是操作系统环境变量、系统属性或命令行参数）。也就是说你在一个配置文件中配置这些属性将不会生效。


### 可选配置
默认情况下，当指定的配置文件位置不存在时，Spring Boot 会抛出 `ConfigDataLocationNotFoundException` 异常，应用程序将无法启动。

如果您想指定一个位置，但又不介意它并不总是存在，可以使用可选的前缀 `optional:`。可以在 `spring.config.location` 和 `spring.config.additional-location` 属性以及 `spring.config.import` 声明中使用该前缀。  
例如，`spring.config.import` 值为 `optional:file:./myconfig.properties` 时，即使缺少 `myconfig.properties` 文件，应用程序也能启动。

如果想忽略所有 `ConfigDataLocationNotFoundException` 错误并始终继续启动应用程序，可以使用 `spring.config.on-not-found` 属性。使用 `SpringApplication.setDefaultProperties(...)` 或系统/环境变量设置为 `ignore`。

### 通配符位置
如果配置文件位置的最后一个路径段包含 `*` 字符，则视为通配符位置。通配符会在加载配置时展开，直接子目录也会被检查。在 Kubernetes 等环境中，当配置属性有多个来源时，通配符位置尤其有用。

例如，如果您有一些 Redis 配置和一些 MySQL 配置，您可能希望将这两项配置分开，同时要求这两项配置都存在于一个 `application.properties` 文件中。这可能会导致两个独立的 `application.properties` 文件被挂载到不同的位置，如 `/config/redis/application.properties` 和 `/config/mysql/application.properties` 。在这种情况下，使用 `config/*/` 通配符位置将导致两个文件都被处理。

默认情况下，Spring Boot 在默认搜索位置中包含 `config/*/` 。这意味着将搜索 jar 外 `/config` 目录的所有子目录。

通配符位置必须只包含一个 `*` ，搜索**目录位置**时以 `*/` 结尾，搜索**文件位置**时以 `*/<filename>` 结尾。带有通配符的位置会根据文件名的绝对路径按字母顺序排序来决定优先级。

### Profile专用文件
除了应用程序属性文件外，Spring Boot 还会尝试使用命名约定 `application-{profile}` 加载特定于profile的配置文件。例如，如果您的应用程序激活了名为 `prod` 的配置文件并使用 YAML 文件，那么 `application.yaml` 和 `application-prod.yaml` 都将被考虑。

profile配置文件的加载位置与标准 `application.properties` 一致，profile配置文件总是优先于非特定文件。如果激活了多个profile，则采用后胜策略。例如，如果通过 `spring.profiles.active` 属性激活了 `prod` 、 `live` ，则 `application-prod.properties` 中的值就会被 `application-live.properties` 中的值覆盖。

`Environment` 有一组默认配置文件（默认为 [default]），如果没有设置激活的配置文件，就会使用这些配置文件。换句话说，如果没有明确激活任何配置文件，则会考虑 `application-default` 文件中的属性。

后胜策略适用于位置组级别。`spring.config.location` 中的 `classpath:/cfg/`,`classpath:/ext/`与`classpath:/cfg/;classpath:/ext/`的覆盖规则不同。  
例如，继续上面的 prod,live 示例，我们可能会有以下文件：
```
/cfg
 application-live.properties
/ext
 application-live.properties
 application-prod.properties
```
当我们的 `spring.config.location` 为 `classpath:/cfg/,classpath:/ext/` 时，我们会在处理所有 `/ext` 文件之前处理所有 `/cfg` 文件：
1. /cfg/application-live.properties
2. /ext/application-prod.properties
3. /ext/application-live.properties

当我们使用 `classpath:/cfg/;classpath:/ext/` 时（使用 ; 分隔符），我们会在同一级别处理 `/cfg` 和 `/ext` 文件：
1. /ext/application-prod.properties
2. /cfg/application-live.properties
3. /ext/application-live.properties

### 导入其他数据
### 导入无扩展文件

### 属性占位符
springboot 在处理 `application.properties` 和 `application.yaml` 中的值时，会使用 `Environment` 中已有的属性值进行处理 ，因此您可以引用以前定义的值（例如，从系统属性或环境变量中）。标准的 `${name}` 属性占位符语法可用于值的任何位置。属性占位符还可以指定默认值，使用 `:` 分隔属性名称和默认值，例如 `${name:default}` 。

实例：
```
app.name=MyApp
app.description=${app.name} is a Spring Boot application written by ${username:Unknown}
```

应该始终使用其规范连字符式（仅使用小写字母的 kebab-case）来引用占位符中的属性名称。保持与Spring Boot `@ConfigurationProperties` 相同的逻辑。  
例如，`${demo.item-price}` 将从 `application.properties` 文件中获取 `demo.item-price` 和 `demo.itemPrice` 属性值，并从系统环境中获取 `DEMO_ITEMPRICE` 。如果使用 `${demo.itemPrice}` ，则不会考虑 `demo.item-price` 和 `DEMO_ITEMPRICE`。

## 类型安全配置属性