# ProxyFactory
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/aop-api/prog.html


使用 Spring 通过编程方式创建 AOP 代理非常简单。这使您能够在不依赖 Spring IoC 的情况下使用 Spring AOP。

目标对象实现的接口会自动被代理。以下代码片段展示了如何为目标对象创建一个代理，其中包含一个 `interceptor` 和一个 `advisor`：
```
ProxyFactory factory = new ProxyFactory(myBusinessInterfaceImpl);
factory.addAdvice(myMethodInterceptor);
factory.addAdvisor(myAdvisor);
MyBusinessInterface tb = (MyBusinessInterface) factory.getProxy();
```

第一步是创建一个 `org.springframework.aop.framework.ProxyFactory` 类型的对象。可以像前面的示例那样使用目标对象来创建它，或者在另一个构造函数中指定要代理的接口。

可以在 `ProxyFactory` 的生命周期内添加 `advice` （其中拦截器是 `advice` 的一种特殊类型）、 `advisor` ，或两者兼有，并对它们进行操作。如果添加一个 `IntroductionInterceptionAroundAdvisor` ，就可以让代理实现额外的接口。

`ProxyFactory` （继承自 `AdvisedSupport` ）还提供了一些便捷方法，允许添加其他类型的 `advice` ，例如 `before advice` 和 `throws advice` 。 `AdvisedSupport` 是 `ProxyFactory` 和 `ProxyFactoryBean` 的共同父类。

在大多数应用程序中，将 AOP 代理的创建与 IoC 框架集成是最佳实践。 `Advice` 将配置从 Java 代码中分离出来，并使用 AOP 进行管理，这通常也是应该采取的做法。



```
            三种入口
   --------------------------------
   |              |              |
@Aspect      AspectJProxyFactory   ProxyFactoryBean
   |              |              |
   |              |              |
   -------> Advisor <------------
               ↓
         ProxyFactory
               ↓
      DefaultAopProxyFactory
               ↓
     -----------------------
     |                     |
JDK Dynamic Proxy     CGLIB Proxy
     |                     |
     --------> Proxy Object
                     ↓
            MethodInterceptor 链
                     ↓
                 目标方法
```
3个核心类:
```
AnnotationAwareAspectJAutoProxyCreator  ⭐⭐⭐⭐⭐
ProxyFactory                            ⭐⭐⭐⭐
DefaultAopProxyFactory                  ⭐⭐⭐⭐
```