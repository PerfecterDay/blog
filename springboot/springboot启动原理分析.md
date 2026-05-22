# Springboot 启动原理-run方法及扩展点
{docsify-updated}

- [Springboot 启动原理-run方法及扩展点](#springboot-启动原理-run方法及扩展点)
  - [扩展点](#扩展点)
    - [SpringApplicationRunListener](#springapplicationrunlistener)
    - [ApplicationContextInitializer](#applicationcontextinitializer)
    - [ApplicationRunner 和 CommandLineRunner](#applicationrunner-和-commandlinerunner)
      - [ApplicationRunner](#applicationrunner)
      - [CommandLineRunner](#commandlinerunner)
      - [自定义添加 Runner](#自定义添加-runner)
    - [测试代码](#测试代码)
  - [通过 SpringApplication 的 setXXX() 方法自定义 SpringApplication](#通过-springapplication-的-setxxx-方法自定义-springapplication)
  - [Headless 模式](#headless-模式)

实例化了 `SpringApplication` 对象之后，紧接着就是调用了 `run()` 方法：
<center><img src="pics/springapplication-run.png" width="70%"></center>

运行到 `refresh(context)` 将会调用 `applicationContext` 的核心 `refresh()` 方法，就是 [spring 的核心启动过程](../core-ioc/spring-ioc-核心原理.md)

## 扩展点

### SpringApplicationRunListener
`SpringApplicationRunListener` 是 `SpringApplication run()` 方法的侦听器。 **与 spring events(`ApplicationListener`) 机制不同，它是专门用来监控 `run()` 方法的，只在 `run()` 方法作用域内有效， 而 sring events 中的 listeners 是长期存在与容器中，可以在各个业务逻辑中随时使用监听特定事件发生的。**创建并使用自定义 `SpringApplicationRunListener` 一般要两个步骤：
1. `SpringApplicationRunListener` 是通过 `SpringFactoriesLoader` 加载的，所以必须在 `META-INF/spring.factories`中声明：
   ```
	org.springframework.boot.SpringApplicationRunListener=\
	com.demo.demo.MyRunnerListener
   ```
   因为 `SpringApplication` 没有提供 `addSpringApplicationRunListener` 之类的方法，而是采用 SPI 机制，所以只能用上述方法。  
2. 自定义一个类`SpringApplicationRunListener`接口，实现类应该定义一个接受 `SpringApplication` 实例和`String[]`参数的公共构造函数。`SpringApplication` 每次运行都会创建一个新的 `SpringApplicationRunListener` 实例。

`SpringApplicationRunListener` 的方法会在 `SpringApplication run()` 方法的不同阶段被调用。
```
public class MyRunnerListener implements SpringApplicationRunListener {
    public MyRunnerListener(SpringApplication application, String[] args) {
    }

    @Override
    public void starting(ConfigurableBootstrapContext bootstrapContext) {
        System.out.println(">>>>>>>>> starting");
    }

    @Override
    public void environmentPrepared(ConfigurableBootstrapContext bootstrapContext, ConfigurableEnvironment environment) {
        System.out.println(">>>>>>>>> environmentPrepared");
    }

    @Override
    public void contextPrepared(ConfigurableApplicationContext context) {
        System.out.println(">>>>>>>>> contextPrepared");
    }

    @Override
    public void contextLoaded(ConfigurableApplicationContext context) {
        System.out.println(">>>>>>>>> contextLoaded");
    }

    @Override
    public void started(ConfigurableApplicationContext context, Duration timeTaken) {
        System.out.println(">>>>>>>>> started >>>>>>");
    }

    @Override
    public void ready(ConfigurableApplicationContext context, Duration timeTaken) {
        System.out.println(">>>>>>>>> ready");
    }
    
    @Override
    public void failed(ConfigurableApplicationContext context, Throwable exception) {
        System.out.println(">>>>>>>>> failed");
    }
}
```

输出:
```
>>>>>>>>> starting
>>>>>>>>> environmentPrepared

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.7.2)

>>>>>>>>> contextPrepared
2025-04-12 19:19:48.541  INFO 89853 --- [           main] com.alibaba.demo.Application             : Starting Application using Java 17.0.7 on coder-wangdeMacBook-Pro.local with PID 89853 (/Users/coder_wang/Workspace/demo-service/start/target/classes started by coder_wang in /Users/coder_wang/Workspace/demo-service)
2025-04-12 19:19:48.542 DEBUG 89853 --- [           main] com.alibaba.demo.Application             : Running with Spring Boot v2.7.2, Spring v5.3.22
2025-04-12 19:19:48.542  INFO 89853 --- [           main] com.alibaba.demo.Application             : No active profile set, falling back to 1 default profile: "default"
>>>>>>>>> contextLoaded
2025-04-12 19:19:49.111  INFO 89853 --- [           main] com.alibaba.demo.Application             : Started Application in 0.773 seconds (JVM running for 1.306)
>>>>>>>>> started >>>>>>
>>>>>>>>> ready
```

### ApplicationContextInitializer
回调接口，**用于在刷新 Spring ConfigurableApplicationContext 之前对其进行初始化**。
通常用于需要对应用上下文进行编程初始化的网络应用程序中。例如，针对上下文环境注册属性源或激活配置文件。请参阅 `ContextLoader` 和 `FrameworkServlet` 支持，分别用于声明 `contextInitializerClasses` 上下文参数和初始参数。
我们鼓励 `ApplicationContextInitializer` 处理程序检测 Spring 的 `Ordered` 接口是否已实现或 `@Order` 注解是否存在，并在调用前对实例进行相应排序。

`ApplicationContextInitializer` 是在springboot启动过程(refresh方法前)中调用,主要是在 `ApplicationContextInitializer` 中 `initialize` 方法中拉起了 `ConfigurationClassPostProcessor` 这个类(我在springboot启动流程中有描述)，通过这个 processor 实现了 beandefinition 的扫描与加载。言归正传， 自定义 `ApplicationContextInitializer` 主要有3种方式：
1. **使用spring.factories方式**

	首先我们自定义个类实现了 `ApplicationContextInitializer` ,然后在resource下面新建 `META-INF/spring.factories` 文件。然后在文件中加入：
	`org.springframework.context.ApplicationContextInitializer=com.baicy.springbootexplore.initializer.KeyInitializer`

2. **application.properties添加配置方式**

	对于这种方式是通过 `DelegatingApplicationContextInitializer` 这个初始化类中的 `initialize` 方法获取到application.properties中`context.initializer.classes`对应的类并执行对应的initialize方法。只需要将实现了ApplicationContextInitializer的类添加到application.properties即可。
	`context.initializer.classes=com.baicy.springbootexplore.initializer.KeyInitializer`
	下面是 `DelegatingApplicationContextInitializer` 加载配置文件中 initializer 的代码：
	```
	private static final String PROPERTY_NAME = "context.initializer.classes";
	private List<Class<?>> getInitializerClasses(ConfigurableEnvironment env) {
			String classNames = env.getProperty(PROPERTY_NAME);
			List<Class<?>> classes = new ArrayList<Class<?>>();
			if (StringUtils.hasLength(classNames)) {
				for (String className : StringUtils.tokenizeToStringArray(classNames, ",")) {
					classes.add(getInitializerClass(className));
				}
			}
			return classes;
	}
	```

3. **直接通过 `SpringApplication` 的 `addInitializers()` 方法注册**

	```
	public static void main(String[] args) {
		SpringApplication application = new SpringApplication(SpringbootExploreApplication.class);
		application.addInitializers(new KeyInitializer());
		application.run(args);
	}
	```

### ApplicationRunner 和 CommandLineRunner
在开发过程中会有这样的场景：需要在容器启动的时候执行一些内容，比如：读取配置文件信息，数据库连接，删除临时文件，清除缓存信息，在Spring框架下是通过 `ApplicationListener` 监听器来实现的。在Spring Boot中给我们提供了两个接口 `CommandLineRunner` 和 `ApplicationRunner` ，来帮助我们实现这样的需求。

在没有指定加载顺序 @Order 时或 @Order 值一致时, 先执行 `ApplicationRunner` 。
如果指定了加载顺序 @Order,则按照 @Order 的顺序进行执行。
说明：数字越小，优先级越高，也就是@Order(1)注解的类会在@Order(2)注解的类之前执行。

Springboot 在程序启动之后（ApplicationContext创建、refresh 之后）调用这些接口方法。
<!-- <center><img src="pics/springboot-runner.png" width="50%"></center> -->

#### ApplicationRunner
```
public interface ApplicationRunner {

	/**
	 * Callback used to run the bean.
	 * @param args incoming application arguments
	 * @throws Exception on error
	 */
	void run(ApplicationArguments args) throws Exception;
}
```

#### CommandLineRunner
```
@FunctionalInterface
public interface CommandLineRunner {

	/**
	 * Callback used to run the bean.
	 * @param args incoming main method arguments
	 * @throws Exception on error
	 */
	void run(String... args) throws Exception;

}
```

#### 自定义添加 Runner
不同于 `ApplicationContextInitializer` , `ApplicationRunner` 和 `CommandLineRunner` 是在容器初始化完成、程序启动成功后调用的，Springboot 能够根据类型自动识别并调用，所以要实现自定义的runner 只需要声明一个 bean 并且实现对应接口即可。

### 测试代码
```
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        SpringApplication application = new SpringApplication(App.class);
        application.setBanner((Environment environment, Class<?> sourceClass, PrintStream out)->{out.println("My own banner");});
        application.addInitializers(applicationContext -> {
            System.out.println("ApplicationContextInitializer>>>");
        });
        application.run();
    }

    @Bean
    public ApplicationRunner applicationRunner(){
        return (ApplicationArguments agrs) ->{
            System.out.println("ApplicationRunner>>>"+agrs.getSourceArgs());
        };
    }

    @Bean
    public CommandLineRunner commandLineRunner(){
        return (String[] agrs) ->{
            System.out.println("CommandLineRunner>>>"+Arrays.asList(agrs));
        };
    }
}
```
除此之外还有上面自定义的 `SpringApplicationRunListener`，运行截图：
<center><img src="pics/springboot-run.png" alt=""></center>


## 通过 SpringApplication 的 setXXX() 方法自定义 SpringApplication
1. `setBeanNameGenerator(BeanNameGenerator beanNameGenerator)` ：设置自定义的 beanName 生成器
2. `setBannerMode(Banner.Mode bannerMode)`：设置启动 Banner 的模式：OFF、CONSOLE、LOG
3. `setBanner(Banner banner)`: 设置自定义 banner

## Headless 模式
如果您的 Java 应用程序不与用户直接交互，则可以使用无头模式。 这意味着您的 Java 应用程序不显示窗口或对话框，不接受键盘或鼠标输入，也不使用任何重量级 AWT 组件。 在 Java 调用中指定 Java 属性 `java.awt.headless=true` 即可选择该模式。 通过使用无头模式，可以避免使用 VNC/X 服务器。