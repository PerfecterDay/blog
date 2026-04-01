# Spring 数据绑定
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/validation/data-binding.html

数据绑定可用于将用户输入绑定到目标对象，其中用户输入是一个以 **属性路径(如 `account.name`)** 为 key 的 map。 `DataBinder` 是支持该功能的主类，它提供了两种绑定用户输入的方法：
+ 构造函数绑定: 将用户输入绑定到公共构造函数，将用户输入的参数绑定到构造函数的对应参数中。（只有高版本才支持）
+ 属性绑定：将用户输入与 `Setter` 绑定，将用户输入的键与目标对象结构的对象属性相匹配。

可同时应用构造函数绑定与属性绑定，也可仅采用其中一种方式。

## 构造函数绑定
1. 创建一个以 `null` 为目标对象的 `DataBinder` 
2. 设置 `targetType` 为目标类
3. 调用 `construct`

目标类应具有单个公共构造函数或单个带参数的非公共构造函数。若存在多个构造函数，则使用默认构造函数（若存在）。

默认情况下，参数值通过构造函数参数名称进行查找。Spring MVC 和 WebFlux 支持通过构造函数参数或字段（若存在）上的 `@BindParam` 注解实现自定义名称映射。如有必要，还可以在 `DataBinder` 上配置 `NameResolver` 来自定义使用的参数名称。

类型转换会根据需要应用于用户输入的转换。若构造函数参数为对象，则通过嵌套属性路径以相同方式递归构造。这意味着构造函数绑定会同时创建目标对象及其包含的所有对象。

构造函数绑定支持 `List` , `Map` 和数组参数，这些参数可通过单个字符串转换（例如逗号分隔的列表），也可基于索引键（如 `accounts[2].name` 或 `account[KEY].name` ）。

绑定和转换错误会反映在 `DataBinder` 的 `BindingResult` 中。如果目标对象创建成功，则在调用 `construct` 方法后， `target` 会被设置为创建的实例。

```
public class Contact {
    private String street;
    private int number;
// setters & getters
};
public class Person {

    @AllArgsConstructor
    @ToString
    static class User {
        private final String name;
        private final int age;
        private final String email;

    }
    
    public static void main(String[] args) {
        System.out.println("=== Demo 4: 绑定错误处理 ===");

        Map<String, String> params = Map.of(
                "name", "Dave",
                "age", "12",   // ← 会导致 String→int 转换失败
                "email", "dave@example.com"
        );

        DataBinder binder = new DataBinder(null);
        binder.setTargetType(ResolvableType.forClass(User.class));

        binder.construct(new DataBinder.ValueResolver() {
            @Override
            public Object resolveValue(String name, Class<?> type) {
                return params.get(name);
            }

            @Override
            public Set<String> getNames() {
                return params.keySet();
            }
        });

        if (binder.getBindingResult().hasErrors()) {
            System.out.println("绑定存在错误:");
            binder.getBindingResult().getAllErrors().forEach(err ->
                    System.out.println("  - " + err.getDefaultMessage())
            );
        }
        System.out.println("target: " + binder.getTarget());
    }
}
```

## 使用 BeanWrapper 进行属性绑定
`BeanWrapper` 接口及其相应的实现（ `BeanWrapperImpl` ）是 Bean 包中一个相当重要的类。 `BeanWrapper` 提供了设置和获取属性值（单独或批量）、获取属性描述符以及查询属性以确定其是否可读或可写的功能。此外， `BeanWrapper` 还支持嵌套属性，可在无限深度的子属性上设置属性。 `BeanWrapper` 还支持添加标准 `JavaBeans PropertyChangeListeners` 和 `VetoableChangeListeners` ，而无需在目标类中添加支持代码。最后 `BeanWrapper` 支持设置索引属性。应用程序代码通常不会直接使用 `BeanWrapper` ，但 `DataBinder` 和 `BeanFactory` 会使用它。

`BeanWrapper` 的工作原理从其名称中可窥见一斑：**它通过封装 Bean 来对该 Bean 执行操作，例如设置和获取属性。**

### 设置和获取基本属性与嵌套属性
```
@Data
public class Company {
	private String name;
	private Employee managingDirector;
}

@Data
public class Employee {
	private String name;
	private float salary;
}

BeanWrapper company = new BeanWrapperImpl(new Company());
// setting the company name..
company.setPropertyValue("name", "Some Company Inc.");
// ... can also be done like this:
PropertyValue value = new PropertyValue("name", "Some Company Inc.");
company.setPropertyValue(value);

// ok, let's create the director and tie it to the company:
BeanWrapper jim = new BeanWrapperImpl(new Employee());
jim.setPropertyValue("name", "Jim Stravinsky");
company.setPropertyValue("managingDirector", jim.getWrappedInstance());

// retrieving the salary of the managingDirector through the company
Float salary = (Float) company.getPropertyValue("managingDirector.salary");
```

## PropertyEditors
Spring使用PropertyEditor的概念来实现**对象**和**字符串**之间的转换。可以通过注册 `java.beans.PropertyEditor` 类型的自定义编辑器来实现。在 `BeanWrapper` 上注册自定义编辑器，或在特定 IoC 容器中注册自定义编辑器，可使其了解如何将属性转换为所需类型。  
在Spring中使用属性编辑的几个例子：
+  在Bean上设置属性是通过使用 `PropertyEditor` 的实现来完成的。当您使用 String 作为在 XML 文件中声明的某个 Bean 的属性值时，Spring（如果相应属性的设置器具有 Class 参数）会使用 `ClassEditor` 尝试将该参数解析为 `Class` 对象。
+  在Spring的MVC框架中，解析HTTP请求参数是通过使用各种 `PropertyEditor` 实现来完成的，你可以在 `CommandController` 的所有子类中手动绑`PropertyEditor`。

Spring有许多内置的 `PropertyEditor` 实现来简化生活。它们都位于 `org.springframework.beans.propertyeditors` 包中。大多数（但不是全部，如下表所示）默认由 `BeanWrapperImpl` 注册。可以注册你自己的变量来覆盖默认的编辑器。下表描述了Spring提供的各种 `PropertyEditor` 实现：
+ `ByteArrayPropertyEditor`
+ `ClassEditor`
+ `CustomBooleanEditor`
+ `CustomCollectionEditor`
+ `CustomDateEditor`
+ `CustomNumberEditor`
+ `FileEditor`
+ `InputStreamEditor`
+ `LocaleEditor`
+ `PatternEditor`
+ `PropertiesEditor`
+ `StringTrimmerEditor`
+ `URLEditor`

`BeanWrapperImpl`继承自 `PropertyEditorRegistrySupport` ，在 `PropertyEditorRegistrySupport` 的 `createDefaultEditors()` 方法中注册了若干的默认 `PropertyEditor` 。

Spring使用 `java.beans.PropertyEditorManager` 来设置可能需要的属性编辑器的搜索路径。搜索路径还包括 `sun.bean.editors` ，其中包括 `Font` 、 `Color` 和大多数基元类型的 `PropertyEditor` 实现。还请注意，如果 `PropertyEditor` 类与它们所处理的类位于同一个包中，并且名称与该类相同，并附加了 Editor，那么标准 JavaBeans 基础架构会自动发现 `PropertyEditor` 类（无需显式注册）。例如，我们可以使用下面的类和包结构，这样就可以识别 `SomethingEditor` 类，并将其用作 `Something` 类型属性的 `PropertyEditor` ，也可以在这里使用标准的 `BeanInfo` JavaBeans 机制（在这里进行了一定程度的描述）。一般通过继承 `SimpleBeanInfo` 来是实现自己的 `BeanInfo` 。
```
com
  chank
    pop
      Something1
      Something2
      Something1Editor // the PropertyEditor for the Something class
      Something2BeanInfo // the BeanInfo for the Something class
```

### 使用自定义 PropertyEditor
如果需要注册其他自定义 `PropertyEditor` ，有几种机制可供选择。
1. 如果能获取到 `BeanFactory` 的引用，可以使用 `ConfigurableBeanFactory` 接口的 `registerCustomEditor()`
2. 使用 `CustomEditorConfigurer` 的特殊 `BeanFactoryPostProcessor`，直接在 `ApplicationContext` 中注册一个该类型的 bean，并通过`setCustomEditors(Map<Class<?>, Class<? extends PropertyEditor>> customEditors)`方法将自定义一些 `PropertyEditor` 添加到 `private Map<Class<?>, Class<? extends PropertyEditor>> customEditors` 的属性中即可


### CustomEditorConfigurer 与 PropertyEditorRegistrar
向Spring容器注册属性编辑器的另一种机制是创建并使用 `PropertyEditorRegistrar` 。当您需要在多种不同情况下使用同一组属性编辑器时，该接口尤其有用。可以编写一个相应的注册器，并在每种情况下重复使用它。`PropertyEditorRegistrar`实例与名为 `PropertyEditorRegistry` 的接口协同工作，该接口由Spring `BeanWrapper` （和 `DataBinder`）实现。`PropertyEditorRegistrar`实例与 `CustomEditorConfigurer` 结合使用时特别方便， `CustomEditorConfigurer` 提供了一个名为 `setPropertyEditorRegistrars(...)` 的属性。以这种方式添加到 `CustomEditorConfigurer` 的`PropertyEditorRegistrar`实例可以轻松地与 `DataBinder` 和Spring MVC控制器共享。此外，它还避免了对自定义编辑器同步的需求：`PropertyEditorRegistrar` 在每次Bean创建尝试创建新的 `PropertyEditor` 实例。