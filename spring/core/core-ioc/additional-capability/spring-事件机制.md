#  spring 事件机制
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/beans/context-introduction.html

Spring 的 ApplicationContext 能够发布事件并且允许注册相应的事件监听器，因此，它拥有一套完善的事件发布和监听机制。在 Java 中， `java.util.EventObject` 类和 `java.util.EventListener` 接口描述了事件和监听器。在事件体系中，除了事件和监听器外，还有另外三个重要概念。

+ 事件源：事件的生产者，任何一个 `EventObject` 都必须有一个事件源。
+ 事件监听器注册表：组件或框架的事件监听器不可漂浮在空中，而必须有所依存。也就是说组件或框架必须提供一个地方保存事件监听器，这便是事件监听器注册表。当组件或框架中的事件源产生事件时，就会通知位于事件监听器注册表中的监听器。
+ 事件广播器：它是事件和事件监听器沟通的桥梁，负责把事件通知给事件监听器。

事件源、事件监听器注册表和事件广播器这3个角色有时可以由同一个对象承担，其实就是观察者模式中的 `Observable` ，监听器就类似于 `Observer` 。事件体系其实是观察者模式的一种具体实现方式。

## Java 观察者模式（事件机制）
<center><img src="pics/java-event.png" width=45%></center>

## spring 事件体系
<center><img src="pics/spring-event.png" width=55%></center>

可以根据需要扩展 `ApplicationEvent` 定义自己的事件。

## Spring事件体系的具体实现
Spring 在 `ApplicationContext` 接口的抽象实现类 `AbstratApplicationContext`中完成事件体系的搭建。`AbstratApplicationContext`中拥有一个 `ApplicationEventMulticaster` 的成员，该成员提供了容器监听器的注册表。`AbstratApplicationContext`在 `refresh()` 这个容器启动方法中通过以下3个步骤搭建了事件的基础设施：
```
..
initApplicationEventMulticaster();
...
registerListeners();
...
finishRefresh();
```
首先初始化事件广播器，用户可以在Spring配置中定义一个事件广播器，只要实现 `ApplicationEventMulticater` 即可， spring 会通过反射机制将其注册成容器的事件广播器，如果没有找到配置的外部事件广播器， Spring 自动使用 `SimpleApplicationEventMulticaster` 作为事件广播器。

然后， Spring 根据反射机制，从注册的 bean 中，找出所有实现了 `ApplicationListener` 的 bean ，并将它们注册为容器的事件监听器，实际操作就是将其添加到事件广播器所提供的事件监听器注册表中。

最后，调用容器的事件发布接口 **`publishEvent()`** 向容器中所有的监听器发布事件。所谓的发布事件本质上其实就是构造一个事件，然后循环调用 `ApplicationListener`的 `onApplicationEvent`方法：
```
@Override
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
	ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
	Executor executor = getTaskExecutor();
	for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
		if (executor != null) {
			executor.execute(() -> invokeListener(listener, event));
		}
		else {
			invokeListener(listener, event);
		}
	}
}
```

## Spring自带的发布事件

+ `ContextRefreshedEvent` — is published when the ApplicationContext is either initialized or refreshed. This can also be raised using the refresh() method on the ConfigurableApplicationContext interface.
+ `ContextStartedEvent` — is published when the ApplicationContext is started using the start() method on the ConfigurableApplicationContext interface. At this stage you can poll your database or you can restart any stopped application after receiving this event.
+ `ContextStoppedEvent` — is published when the ApplicationContext is stopped using the stop() method on the ConfigurableApplicationContext interface. You can do required housekeep work after receiving this event.
+ `ContextClosedEvent` — is published when the ApplicationContext is closed using the close() method on the ConfigurableApplicationContext interface. A closed context reaches its end of life; it cannot be refreshed or restarted.

<center><img src="pics/spring-events.webp" width="90%"></center>

## Springboot的内置事件
+ `ApplicationStartingEvent` — is sent at the start of a run but before any processing, except for the registration of listeners and initializers.
+ `ApplicationEnvironmentPreparedEvent` — is sent when the Environment to be used in the context is known but before the context is created.
+ `ApplicationContextInitializedEvent` — is sent when the ApplicationContext is prepared and ApplicationContextInitializers have been called but before any bean + definitions are loaded.
+ `ApplicationPreparedEvent` — is sent just before the refresh is started but after bean definitions have been loaded.
+ `ApplicationStartedEvent` — is sent after the context has been refreshed but before any application and command-line runners have been called.
+ `AvailabilityChangeEvent` — is sent right after with LivenessState.CORRECT to indicate that the application is considered as live and after with ReadinessState.ACCEPTING_TRAFFIC to indicate that the application is ready to service requests.
+ `ApplicationReadyEvent` — is sent after any application and command-line runners have been called.
+ `ApplicationFailedEvent` — is sent if there is an exception on startup.

The above list only includes SpringApplicationEvents that are tied to a SpringApplication. In addition to these, the following events are also published after ApplicationPreparedEvent and before ApplicationStartedEvent:
+ `WebServerInitializedEvent` is sent after the WebServer is ready. `ServletWebServerInitializedEvent` and `ReactiveWebServerInitializedEvent` are the servlet and reactive variants respectively.
+ `ContextRefreshedEvent` is sent when an ApplicationContext is refreshed.

<center><img src="pics/springboot-events.webp" width="90%"></center>

## 手动实现自己的业务事件发布与监听
1. 自定义一个业务事件 Event，以前需要继承自 `ApplicationEvent` ，现在只要是一个 POJO 即可。
2. 发布者（需要发布自定义业务事件的业务Bean）实现 `ApplicationEventPublisherAware`,`ApplicationContextAware` 接口，利用 `ApplicationContext` 对象就可以发布自定义的事件
3. 可以使用 `@EventListener` 注解：
   ```
	@Component
	public class AnnotationDrivenEventListener {
		@EventListener
		public void handleContextStart(MyEvent cse) {
			System.out.println("Handling self defined event.");
		}
	}
   ```
   使用这种方式，如果想监听 spring/springboot 的自带事件，只能监听到 `ContextRefreshedEvent`/`ApplicationStartedEvent`/`ApplicationReadyEvent`/`AvailabilityChangeEvent`/`ContextClosedEvent` 这些事件，之前的事件无法获取到。
4. 上述方法只能在监听器被注册到容器后才能生效，如果想监听 sprin 启动时的一些事件，需要实现 `ApplicationListener` 接口并手动注册到容器中。
   ```
     public static void main(String[] args) {
        SpringApplication app = new SpringApplication(Application.class);
        app.addListeners(new MyOwnListener());
        app.run();
    }
   ```
   这样才能监听到 `ApplicationStartingEvent`/`ApplicationEnvironmentPreparedEvent`/`ApplicationContextInitializedEvent`/`ApplicationPreparedEvent`
