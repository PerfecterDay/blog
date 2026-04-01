# Spring 线程池与任务调度
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/integration/scheduling.html

## TaskExecutor
`TaskExecutor` 最初的设计目的是为其他Spring组件提供线程池抽象功能，使其在需要时能够复用线程池。诸如 `ApplicationEventMulticaster` 、JMS的 `AbstractMessageListenerContainer` 以及 `Quartz` 集成等组件，均通过 `TaskExecutor` 抽象实现线程池管理。如果 `Bean` 需要线程池行为，同样可以利用此抽象机制满足自身需求。

Spring 包含多个预构建的 `TaskExecutor` 实现。在绝大多数情况下，我们无需自行实现。Spring 提供的实现如下：

+ `SyncTaskExecutor` ：此实现不会异步执行调用。相反，**每次调用都在调用的线程中进行**。它主要用于不需要多线程的情况，例如简单的测试用例。
+ `SimpleAsyncTaskExecutor` ：此实现不复用任何线程，而是**为每次调用启动新线程**。但它支持并发限制机制，当调用数超过限制时会阻塞调用，直至释放出线程槽位。当启用 `virtualThreads` 选项时，该实现将使用JDK 21的虚拟线程。此实现还支持通过Spring的生命周期管理实现优雅关闭。
+ `ConcurrentTaskExecutor` ：此实现是 `java.util.concurrent.Executor` 实例的适配器。其将 `Executor` 配置参数作为bean属性暴露。通常无需直接使用 `ConcurrentTaskExecutor` ，但若 `ThreadPoolTaskExecutor` 无法满足需求，可将其作为备选方案。
+ `ThreadPoolTaskExecutor` ：此实现最为常用。它通过暴露bean属性配置 `java.util.concurrent.ThreadPoolExecutor` ，并将其封装为 `TaskExecutor` 。若需适配其他类型的 `java.util.concurrent.Executor` ，建议改用 `ConcurrentTaskExecutor` 。该实现还通过 Spring 的生命周期管理提供暂停/恢复功能及优雅关闭机制.
+ `DefaultManagedTaskExecutor` ：此实现采用 JSR-236 兼容运行环境（如 Jakarta EE 应用服务器）中通过 JNDI 获取的 `ManagedExecutorService` ，替代 CommonJ WorkManager 实现该功能。

### TaskExecutor 与 TaskDecorator 的使用与配置
```
public class LoggingTaskDecorator implements TaskDecorator {

	private static final Log logger = LogFactory.getLog(LoggingTaskDecorator.class);

	@Override
	public Runnable decorate(Runnable runnable) {
		return () -> {
			logger.debug("Before execution of " + runnable);
			runnable.run();
			logger.debug("After execution of " + runnable);
		};
	}
}

@Bean
ThreadPoolTaskExecutor taskExecutor() {
	ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
	taskExecutor.setCorePoolSize(5);
	taskExecutor.setMaxPoolSize(10);
	taskExecutor.setQueueCapacity(25);
	taskExecutor.setTaskDecorator(new LoggingTaskDecorator());
	return taskExecutor;
}
```

如果需使用多个装饰器，可借助 `org.springframework.core.task.support.CompositeTaskDecorator` 实现多个装饰器的顺序执行。

如果不在代码中自定义 bean 的方式配置 `ThreadPoolTaskExecutor` ，同时我们启用了 `@EnableAsync` 时， Spring 会自动注入一个 `ThreadPoolTaskExecutor` 的 bean，我们可以通过以下属性配置它：
```
spring:
  task:
    execution:
      pool:
        core-size: 10        # 核心线程数
        max-size: 50         # 最大线程数
        queue-capacity: 200  # 队列容量
        keep-alive: 60s
      thread-name-prefix: biz-exec-
      shutdown:
        await-termination: true
        await-termination-period: 30s
```

#### 使用Spring Async
使用自定义线程池有两种方式，
   1. 局部注入：定义一个线程池的bean 后，在 `@Async` 注解后指定要使用的线程池的名字
   2. 全局注入：在注入线程池bean的配置类上实现`AsyncConfigurer`	接口，这样注入的线程池就是默认的线程池了（见上面的代码）

如果启用了 `@EnableAsync` 注解，Spring 就会开启异步任务处理，就可以在一些方法上使用 `@Async` 注解来告诉 Spring 使用异步任务来执行某些方法。
```
@Async
Future<String> returnSomething(int i) {
	// this will be run asynchronously
}

@Async("otherExecutor") //使用指定的线程池执行任务
void doSomething(String s) {
	// this will be run asynchronously by "otherExecutor"
}
```

注意：在需要异步执行的方法上加上 `@Async` 注解，注意不能在定义这个异步方法类的内部其他方法中调用这个方法，否则会失效。（AOP增强失效的原因）
```
@Async
@Cacheable(value = "smsCode")
public void saveSms(String mobile, String code) {
    log.info(Thread.currentThread().getName());
    SmsEntity sms = new SmsEntity(mobile,code,"login");
    save(sms);
}
```
原因：
As I said, the two annotations do not work together on the same method. If you think about it, it is obvious. Marking with @Async is also @Cacheable means to delegate the whole cache management to different asynchronous threads. If the computation of the value of the CompletableFuture will take a long time to complete, the value in the cache will be placed after that time by Spring Proxy.

#### @Async 的异常处理
当 `@Async` 方法的返回类型为 `Future` 时，方法执行过程中抛出的异常易于管理，因为该异常会在调用 `Future` 结果的 `get` 方法时被抛出。然而，当返回类型为 `void` 时，异常将无法被捕获且无法传递。此时可提供 `AsyncUncaughtExceptionHandler` 来处理此类异常。以下示例展示了具体实现方式：
```
public class MyAsyncUncaughtExceptionHandler implements AsyncUncaughtExceptionHandler {

	@Override
	public void handleUncaughtException(Throwable ex, Method method, Object... params) {
		log.error("异步任务异常，方法名：{}，异常信息：{}", method.getName(), ex.getMessage(), ex);
	}
}

public class AsyncThreadPoolConfig implements AsyncConfigurer {
    private Integer coreSize;
    private Integer maxSize;
    private Integer queueSize;
    private Integer keepAlive;

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setThreadNamePrefix("user-service-pool");
        executor.setTaskDecorator(new MdcTaskDecorator());
        executor.setCorePoolSize(coreSize);
        executor.setMaxPoolSize(maxSize);
        executor.setQueueCapacity(queueSize);
        executor.setKeepAliveSeconds(keepAlive);
        executor.initialize();
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        return executor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new MyAsyncUncaughtExceptionHandler();
    }

}
```


## 任务调度
使用 `@EnableScheduling` 注解启用 Spring 的任务调度功能， `@Scheduled` 注解标注的方法会被 Spring 调度执行。

```
@Scheduled(cron="*/5 * * * * MON-FRI")
public void doSomething() {
	// something that should run on weekdays only
}

@Scheduled(initialDelay = 1000, fixedRate = 5000)
public void doSomething() {
	// something that should run periodically
}
```

### Cron 表达式
```
 ┌───────────── second (0-59)
 │ ┌───────────── minute (0 - 59)
 │ │ ┌───────────── hour (0 - 23)
 │ │ │ ┌───────────── day of the month (1 - 31)
 │ │ │ │ ┌───────────── month (1 - 12) (or JAN-DEC)
 │ │ │ │ │ ┌───────────── day of the week (0 - 7)
 │ │ │ │ │ │          (0 or 7 is Sunday, or MON-SUN)
 │ │ │ │ │ │
 * * * * * *
 ```