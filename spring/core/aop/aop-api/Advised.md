# 操作 Advised 对象
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/aop-api/advised.html

无论如何创建的 AOP 代理，都可以通过使用 `org.springframework.aop.framework.Advised` 接口来对其进行操作。任何 AOP 代理都可以被强制转换为该接口，无论它还实现了哪些其他接口。该接口包含以下方法：
```
Advisor[] getAdvisors();
void addAdvice(Advice advice) throws AopConfigException;
void addAdvice(int pos, Advice advice) throws AopConfigException;
void addAdvisor(Advisor advisor) throws AopConfigException;
void addAdvisor(int pos, Advisor advisor) throws AopConfigException;
int indexOf(Advisor advisor);
boolean removeAdvisor(Advisor advisor) throws AopConfigException;
void removeAdvisor(int index) throws AopConfigException;
boolean replaceAdvisor(Advisor a, Advisor b) throws AopConfigException;
boolean isFrozen();
```

`getAdvisors()` 方法会为工厂中添加的每个 `Advisor` 、 `interceptor` 或其他 `Advice` 类型返回一个 `Advisor` 。如果你添加了一个 `Advisor` ，则该索引返回的 `advisor` 即是你添加的那个对象。如果添加的是 `interceptor` 或其他 `advice` 类型，Spring 会将其封装在一个 `advisor` 中，该 `advisor` 的切入点始终返回 `true` 。因此，如果您添加了一个 `MethodInterceptor` ，则该索引返回的 `advisor` 是一个  `DefaultPointcutAdvisor` ，它返回您的 `MethodInterceptor` 以及一个匹配所有类和方法的切入点。

`addAdvisor()` 方法可用于添加任何 `Advisor` 。通常，持有切入点和 `Advice` 的 `Advisor` 是通用的 `DefaultPointcutAdvisor` ，可以将其与任何 `Advice` 或切入点配合使用（但不适用于 `introductions` ）。

默认情况下，即使代理已创建，仍可添加或移除 `Advisor` 或 `interceptor` 。唯一的限制是无法添加或移除 `introduction advisor` ，因为工厂生成的现有代理不会反映接口的变更。（您可以通过工厂获取新的代理来避免此问题。）

以下示例演示了如何将 AOP 代理强制转换为 `Advised` 接口，并检查和操作其 `Advice` ：
```
Advised advised = (Advised) myObject;
Advisor[] advisors = advised.getAdvisors();
int oldAdvisorCount = advisors.length;
System.out.println(oldAdvisorCount + " advisors");

// Add an advice like an interceptor without a pointcut
// Will match all proxied methods
// Can use for interceptors, before, after returning or throws advice
advised.addAdvice(new DebugInterceptor());

// Add selective advice using a pointcut
advised.addAdvisor(new DefaultPointcutAdvisor(mySpecialPointcut, myAdvice));

assertEquals("Added two advisors", oldAdvisorCount + 2, advised.getAdvisors().length);
```

在生产环境中修改业务对象的 `Advice` 是否明智（此处无意双关），这一点值得商榷，尽管毫无疑问存在正当的使用场景。然而，在开发过程中（例如在测试中），这可能非常有用。我们有时发现，能够以 `interceptor` 或其他 `Advice` 的形式添加测试代码，从而深入到我们要测试的方法调用中，这非常有用。（例如， `Advice` 可以进入为该方法创建的事务中，也许是为了在将事务标记为回滚之前，运行 SQL 来检查数据库是否已正确更新。）

根据创建代理的方式，通常可以设置一个冻结标志。在这种情况下， `Advised` 的 `isFrozen()` 方法将返回 `true` ，任何试图通过添加或移除来修改 `Advice` 的操作都会引发 `AopConfigException` 。在某些情况下，冻结 `Advised` 对象的状态非常有用（例如，防止调用代码移除安全限制的拦截器）。