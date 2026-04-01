# ApplicationContext 的其它功能
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/beans/context-introduction.html


## Application Startup Tracking
`ApplicationContext` 管理Spring应用的生命周期，并围绕组件提供丰富的编程模型。因此，复杂的应用程序可能具有同样复杂的组件图和启动阶段。  
通过特定指标追踪应用程序启动步骤，不仅有助于了解启动阶段的时间消耗情况，还能作为全面理解上下文生命周期的有效途径。

`AbstractApplicationContext`（及其子类）通过 `ApplicationStartup` 进行指标观测，该启动器收集有关不同启动阶段的 `StartupStep` 数据：
+ 应用上下文生命周期 （base packages scanning, config classes management)
+ Bean生命周期 (instantiation, smart initialization, post processing)
+ 应用事件处理

默认的 `ApplicationStartup` 实现 `DefaultApplicationStartup` 是不可操作的，以实现最小开销。这意味着应用程序启动期间默认不会收集任何指标。Spring框架自带了使用 Java Flight Recorder 追踪启动步骤的实现： `FlightRecorderApplicationStartup` 。要使用此变体，必须在 `ApplicationContext` 创建后立即将其实例配置到 `ApplicationContext` 中。

Spring 还提供了一个 `BufferingApplicationStartup` 的实现：
```
public static void main(String[] args) {
    AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
    BufferingApplicationStartup bufferingApplicationStartup = new BufferingApplicationStartup(1024);
    context.setApplicationStartup(bufferingApplicationStartup);
    context.scan(DemoConfig.class.getPackage().getName());
    context.refresh();

    bufferingApplicationStartup.getBufferedTimeline().getEvents()
            .forEach( e -> {
                System.out.print(e.getStartupStep().getName()+":");
                System.out.println(e.getDuration());
            });
}
```

在 Springboot 中可以通过配置启动:
```
spring:
  application:
    startup: buffer
```

要开始收集自定义的 `StartupStep` ，组件可以直接从 `ApplicationCOntext` 中获取 `ApplicationStartup` 实例: 
+ 让组件实现 ApplicationStartupAware 接口
+ 在任何注入点请求 `ApplicationStartup` 类型， 使用 `@Autowired/@Inject` 等注解注入即可


目前，Spring 定义的一些处理阶段的名字可以在这个链接查看：
https://docs.spring.io/spring-framework/reference/core/appendix/application-startup-steps.html