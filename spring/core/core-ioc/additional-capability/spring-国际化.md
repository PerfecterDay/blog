# Spring 国际化
{docsify-updated}

获取当前 Request 的 locale: `LocaleContextHolder.getLocale()`

## MessageSource
`ApplicationContext` 接口扩展了一个名为 `MessageSource` 的接口，该接口提供了国际化（"i18n"）功能。Spring 还提供了 `HierarchicalMessageSource` 接口，该接口可以分层解析消息。这些接口共同构成了 Spring 实现消息解析的基础。这些接口定义的方法包括：
+ `String getMessage(String code, Object[] args, String default, Locale loc)`: 用于从 `MessageSource` 获取消息的基本方法。如果在指定的本地没有找到消息，则使用默认消息 `default` 。通过标准库提供的 `MessageFormat` 功能，传入的任何参数 `args` 都会成为替换值。
+ `String getMessage(String code, Object[] args, Locale loc)`: 与前一种方法基本相同，但有一点不同：不能指定默认信息。如果找不到信息，就会抛出 `NoSuchMessageException` 异常。
+ `String getMessage(MessageSourceResolvable resolvable, Locale locale)`: 前面方法中使用的所有属性也都封装在一个名为 `MessageSourceResolvable` 的类中，你可以使用该方法。

加载 `ApplicationContext` 时，它会自动搜索上下文中定义的 `MessageSource` Bean。该 Bean 的名称必须是 `messageSource` 。如果找到了这样一个 Bean，对前面方法的所有调用都会委托给该 bean。如果没有找到消息源， `ApplicationContext` 会尝试从父容器查找包含同名的 Bean 。如果找到了，它就会使用该 bean 作为消息源。如果 `ApplicationContext` 无法找到任何消息源，则会实例化一个空的 `DelegatingMessageSource` ，以便能够接受对上述方法的调用。

Spring 提供了三种 `MessageSource` 实现：
+ `ResourceBundleMessageSource`： 基础的从文件加载的 `MessageSource`
+ `ReloadableResourceBundleMessageSource` : 可以热重载的 `MessageSource`
+ `StaticMessageSource` 

它们都实现了 `HierarchicalMessageSource` ，以便进行嵌套消息传递。 `StaticMessageSource` 很少使用，但它提供了向消息源添加消息的编程方法 。

由于 Spring 的 `MessageSource` 基于 Java 的 `ResourceBundle` ，因此**它不会合并具有相同基名的捆绑包，而只会使用找到的第一个捆绑包。具有相同基名的后续消息捆绑包将被忽略。**


### `MessageSource` 的一般使用步骤
1. 定义消息资源文件（*.properties）
```
src/main/resources/messages.properties            # 默认（英文）
src/main/resources/messages_zh_CN.properties      # 简体中文
src/main/resources/messages_zh_TW.properties      # 繁体中文
```
文件内容以key=val 的属性方式呈现：
```
en_US
greeting=Hello, {0}!
farewell=Goodbye!

zh_CN
greeting=你好，{0}！
farewell=再见！
```

2. 配置 MessageSource Bean
方式1:
```
spring:
  messages:
    basename: messages        # 默认值就是 messages
    encoding: UTF-8
```

方式2：
```
@Bean
public MessageSource messageSource() {
    ReloadableResourceBundleMessageSource messageSource = new ReloadableResourceBundleMessageSource();
    messageSource.setBasename("classpath:messages");
    messageSource.setDefaultEncoding("UTF-8");
    return messageSource;
}
```
注意：如果资源文件是放在classpath 下边的，一定要加上 `classpath:xxx` 前缀。

3. 使用 MessageSource 获取消息
```
return messageSource.getMessage("greeting", new Object[]{name}, locale);
```

## ResourceBundles
https://www.baeldung.com/java-resourcebundle


## MessageSource 与 ResourceBundles 的对比
1. `MessageSource` 支持带参数的消息占位符（如 {0}, {1}），可以自动格式化参数； `ResourceBundle` 只能获取字符串，格式化需要手动用 `MessageFormat` 再处理。
2. `MessageSource` 支持按 `Locale` 自动解析不同的语言文件; `ResourceBundle` 虽然也支持国际化，但加载方式和路径更受限，不能很好地与 Spring 应用整合。
3. `ReloadableResourceBundleMessageSource` 实现类支持配置缓存时间，从而在开发过程中实现消息资源的热更新（不重启也能生效）。 `ResourceBundle` 是完全基于内存的，一旦加载就不能更新，需要重启程序才能生效。
4. `MessageSource` 是 Spring 官方推荐的方式，可以方便地在控制器、服务中注入使用，也可以通过 SpEL、@Value、@RequestMapping 等注解中间接调用，非常灵活。
5. `MessageSource` 如果找不到 key，可以返回你指定的默认文本; `ResourceBundle` 抛异常（ `MissingResourceException` ），不够友好。
6. `MessageSource` 本质上是一个接口，你可以自定义实现来支持读取 XML/YAML、数据库、Redis 等多种数据源。


## Springmvc 中的国际化
`DispatcherServlet` 在初始化时会查找 `public static final String LOCALE_RESOLVER_BEAN_NAME = "localeResolver";` 名字的 bean，如果没有，就会使用默认的 `AcceptHeaderLocaleResolver`:
```
private void initLocaleResolver(ApplicationContext context) {
    try {
        this.localeResolver = context.getBean(LOCALE_RESOLVER_BEAN_NAME, LocaleResolver.class);
        if (logger.isTraceEnabled()) {
            logger.trace("Detected " + this.localeResolver);
        }
        else if (logger.isDebugEnabled()) {
            logger.debug("Detected " + this.localeResolver.getClass().getSimpleName());
        }
    }
    catch (NoSuchBeanDefinitionException ex) {
        // We need to use the default.
        this.localeResolver = getDefaultStrategy(context, LocaleResolver.class);
        if (logger.isTraceEnabled()) {
            logger.trace("No LocaleResolver '" + LOCALE_RESOLVER_BEAN_NAME +
                    "': using default [" + this.localeResolver.getClass().getSimpleName() + "]");
        }
    }
}
```

<center><img src="pics/default-strategies.png" alt=""></center>

### 自定义 LocaleResolver
```
public class CustomLocaleResolver extends AcceptHeaderLocaleResolver {

    @Override
    public Locale resolveLocale(HttpServletRequest request) {
        // 1. lang header
        String acceptLanguage = request.getHeader("lang");
        Locale locale = new Locale("zh" , "CN");
        try {
            if (!StrUtil.isBlank(acceptLanguage)) {
                String[] s = acceptLanguage.split("-");
                locale = new Locale(s[0], s[1]);
                return locale;
            }
        } catch (Exception e) {
            return locale;
        }

        // 2. Accept-Language Header
        return super.resolveLocale(request);
    }
}


@Bean
public LocaleResolver localeResolver() {
    return new CustomLocaleResolver();
}
```

**注意使用 `@Bean` 显示注册 bean ，通过标注 `@Component` 方式扫描注册时，因为注册顺序的问题可能会导致不生效。**

## 与校验的结合使用
```
public class PersonValidator implements Validator {

    public boolean supports(Class clazz) {
        return Person.class.equals(clazz);
    }

    public void validate(Object obj, Errors e) {
        ValidationUtils.rejectIfEmpty(e, "name", "name.empty");
        Person p = (Person) obj;
        if (p.getAge() < 0) {
            e.rejectValue("age", "negativevalue");
        } else if (p.getAge() > 110) {
            e.rejectValue("age", "too.darn.old");
        }
    }

    public static void main(String[] args) {
        PersonValidator pe = new PersonValidator();
        Person p = new Person();
        p.setAge(120);
        MapBindingResult xcc = new MapBindingResult(new HashMap<>(), "xcc");
        pe.validate(p,xcc);
        System.out.println("axs");
    }
}

@Data
class Person{
    private String name;
    private int age;
}
```
<center><img src="pics/validate-errcode.png" width="50%"></center>

我们介绍了数据库绑定和验证。本节将介绍与验证错误相对应的信息输出。在上一节的示例中，我们剔除了姓名和年龄字段。如果我们想使用 `MessageSource` 输出错误信息，可以使用拒绝字段时提供的错误代码（本例中为 "姓名 "和 "年龄"）。在调用（直接调用或间接调用 `ValidationUtils` 类等） `rejectionValue` 或 `Errors` `接口中的其他剔除方法时，底层实现不仅会注册您传入的代码，还会注册一些额外的错误代码。MessageCodesResolver` 决定 `Errors` 接口注册哪些错误代码。默认情况下，使用的是 `DefaultMessageCodesResolver` ，它（例如）不仅会注册包含您提供的代码的消息，还会注册包含您传递给剔除方法的字段名的消息。因此，如果使用 `rejectValue("age", "too.darn.old")` 剔除一个字段，除了 `too.darn.old` 代码外，Spring 还会注册 `too.darn.old.age` 和 `too.darn.old.age.int` 消息（前者包括字段名称，后者包括字段类型）。这样做是为了方便开发人员定位错误信息。

