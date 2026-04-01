#  spring AOP
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/aop.html

面向切面编程（AOP）通过提供另一种思考程序结构的方式，对面向对象编程（OOP）进行了补充。在OOP中，模块化的关键单元是类，而在AOP中，模块化的单元则是切面。切面使跨多个类型和对象的关注点（如事务管理）得以模块化。（此类关注点在AOP文献中常被称为"横切关注点"。）


## 实现机制
1. 动态代理  
    引入动态代理后，我们可以将横切关注点逻辑（增强）封装到动态代理的 InvocationHandler 接口中，然后在运行期间，根据横切关注点的需要织入的模块位置，将横切逻辑织入到相应的代理类中。这种方式的缺点是所有要织入横切逻辑的模块类都要实现相应的接口，因为动态代理只对接口有效。
2. 动态字节码增强  
    使用ASM或者CGLIB等Java工具库，在程序运行期间，动态生成字节码文件，为需要织入横切逻辑的模块类动态生成相应的子类，将横切逻辑加到这些子类中。这样就可以克服动态代理需要实现接口的限制，不过字节码增强无法扩展final类和final方法。


## Spring AOP 的织入
AspectJ使用 ajc 编译器作为它的织入器，Jboss AOP 自定义的ClassLoader作为织入器， Spring AOP 中使用 `ProxyFactory` 作为织入器。使用 `ProxyFactory` 很简单，只需要以下3步：
1. 第一就是告诉它你要织入的目标对象：可以在构造函数中直接传入，也可以用setter方法设置
2. 第二就是告诉它要应用到目标对象的 `Aspect(Advisor)` 或者直接告诉它 `Advice` ：addAdvisor()/addAdvice()
3. 最后直接调用 `ProxyFactory` 的 `getProxy()` 方法就可以获得增强的代理对象

```
ProxyFactory proxyFactory = new ProxyFactory(targetObj);
//或者 proxyFactory.setTarget(targetObj);
proxyFactory.addAdvisor(advisor);
//或者 proxyFactory.addAdvice(advice);
Object proxy = proxyFactory.getProxy();
```
默认情况下，如果目标对象实现了接口， ProxyFactory 会使用 JDK 的动态代理生成代理对象，下述三种情况会使用基于类的代理：
1. 目标类没有实现任何接口，忽略 proxyTargetClass 的值
2. 如果 ProxyFactory 的 proxyTargetClass 属性为 true
3. 如果 ProxyFactory 的 optimize 属性为 true

上述方式主要是对于编程式AOP的支持。另外，为了支持容器，Spring提供了 `ProxyFactoryBean` 来支持配置式的AOP。

### Spring AOP 的自动织入
Spring 提供了自动代理机制，让容器自动生成代理，把开发人员从繁琐的配置工作中解放出来。在内部，Spring 使用 BeanPostProcessor 自动完成这项工作。只要提供一个 BeanPostProcessor，这个 BeanPostProcessor 在对象实例化时，为符合条件的 bean 生成代理对象并返回，而不是返回实例化的目标 bean 本身，可以用下述伪代码表示大致过程：
```
for(bean in beanfactory){
    if(如果要为该bean 生成代理对象){
        Object proxy = createProxy(bean);
        return proxy;
    }else{
        Object instance = createInstance(bean);
        return instance;
    }
}
```
<center><img src="pics/ProxyCreator.png" alt="" height=60% width=60%></center>

## @AspectJ 支持
`@AspectJ` 是一种使用 Java 注解来实现 AOP 的编码风格.
`@AspectJ` 风格的 AOP 是 AspectJ Project 在 AspectJ 5 中引入的, 并且 Spring 也支持 `@AspectJ` 的 AOP 风格.

### 使能 @AspectJ 支持
`@AspectJ` 可以以 XML 的方式或以注解的方式来使能, 并且不论以哪种方式使能 `@ASpectJ`, 我们都必须保证 aspectjweaver.jar 在 classpath 中.

1. 使用 Java Configuration 方式使能@AspectJ
```
@Configuration
@EnableAspectJAutoProxy
public class AppConfig {
}
```

2. 使用 XML 方式使能@AspectJ
```
<aop:aspectj-autoproxy/>
```

### 定义aspect(切面)
当使用注解 `@Aspect` 标注一个 Bean 后, 那么 Spring 框架会自动收集这些 Bean, 并添加到 Spring AOP 中, 例如:
```
@Component
@Aspect
public class MyTest {
}
```
注意, 仅仅使用 `@Aspect` 注解, 并不能将一个 Java 对象转换为 Bean, 因此我们还需要使用类似 `@Component` 之类的注解.
注意, 如果一个类被 `@Aspect` 标注, 则这个类就不能是其他 aspect 的 **advised object** 了, 因为使用 `@Aspect` 后, 这个类就会被排除在 `auto-proxying` 机制之外.


### 声明 pointcut
一个 pointcut 的声明由两部分组成:

* 一个方法签名, 包括方法名和相关参数
* 一个 pointcut 表达式, 用来指定哪些方法执行是我们感兴趣的(即因此可以织入 advice).

在@AspectJ风格的AOP中, 我们使用一个方法来描述 pointcut, 即:

    @Pointcut("execution(* com.xys.service.UserService.*(..))") // 切点表达式
    private void dataAccessOperation() {} // 切点前面
这个方法必须无返回值.
这个方法本身就是 pointcut signature, pointcut 表达式使用@Pointcut 注解指定.
上面我们简单地定义了一个 pointcut, 这个 pointcut 所描述的是: 匹配所有在包 com.xys.service.UserService 下的所有方法的执行.






### 声明 advice
advice 是和一个 pointcut 表达式关联在一起的, 并且会在匹配的 join point 的方法执行的前/后/周围 运行. pointcut 表达式可以是简单的一个 pointcut 名字的引用, 或者是完整的 pointcut 表达式.
下面我们以几个简单的 advice 为例子, 来看一下一个 advice 是如何声明的.

#### Before advice
    /**
     * @author xiongyongshun
     * @version 1.0
     * @created 16/9/9 13:13
     */
    @Component
    @Aspect
    public class BeforeAspectTest {
        // 定义一个 Pointcut, 使用 切点表达式函数 来描述对哪些 Join point 使用 advise.
        @Pointcut("execution(* com.xys.service.UserService.*(..))")
        public void dataAccessOperation() {
        }
    }
    
    @Component
    @Aspect
    public class AdviseDefine {
        // 定义 advise
        @Before("com.xys.aspect.PointcutDefine.dataAccessOperation()")
        public void doBeforeAccessCheck(JoinPoint joinPoint) {
            System.out.println("*****Before advise, method: " + joinPoint.getSignature().toShortString() + " *****");
        }
    }
这里, @Before 引用了一个 pointcut, 即 "com.xys.aspect.PointcutDefine.dataAccessOperation()" 是一个 pointcut 的名字.
如果我们在 advice 在内置 pointcut, 则可以:

    @Component
    @Aspect
    public class AdviseDefine {
        // 将 pointcut 和 advice 同时定义
        @Before("within(com.xys.service..*)")
        public void doAccessCheck(JoinPoint joinPoint) {
            System.out.println("*****doAccessCheck, Before advise, method: " + joinPoint.getSignature().toShortString() + " *****");
        }
    }

#### around advice

around advice 比较特别, 它可以在一个方法的之前之前和之后添加不同的操作, 并且甚至可以决定何时, 如何, 是否调用匹配到的方法.

    @Component
    @Aspect
    public class AdviseDefine {
        // 定义 advise
        @Around("com.xys.aspect.PointcutDefine.dataAccessOperation()")
        public Object doAroundAccessCheck(ProceedingJoinPoint pjp) throws Throwable {
            StopWatch stopWatch = new StopWatch();
            stopWatch.start();
            // 开始
            Object retVal = pjp.proceed(); 
            stopWatch.stop();
            // 结束
            System.out.println("invoke method: " + pjp.getSignature().getName() + ", elapsed time: " + stopWatch.getTotalTimeMillis());
            return retVal;
        }
    }
around advice 和前面的 before advice 差不多, 只是我们把注解 @Before 改为了 @Around 了.
