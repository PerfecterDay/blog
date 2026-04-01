# Binder
{docsify-updated}

## 实例
一个容器对象，将对象与来自一个或多个 `ConfigurationPropertySources` 的配置值绑定。

新建一个 app.yml 配置文件：
```
database:
  url: localhost
  port: 3306
  userName: root
  password: root
```

测试程序：
```
public class BinderTest {

    public static void main(String[] args) throws IOException {
        YamlPropertySourceLoader loader = new YamlPropertySourceLoader();
        ClassPathResource resource = new ClassPathResource("app.yml");
        List<PropertySource<?>> custome = loader.load("custome", resource);
        StandardEnvironment environment = new StandardEnvironment();
        environment.getPropertySources().addFirst(custome.get(0));
        Binder binder = Binder.get(environment);

        BindResult<DataBaseConf> database = binder.bind("database", Bindable.of(DataBaseConf.class));
        System.out.println(database.get());
    }

    @Data
    static class DataBaseConf{
        private String url;
        private String port;
        private String userName;
        private String password;
    }
}
```

输出结果：
```
BinderTest.DataBaseConf(url=localhost, port=3306, userName=root, password=root)
```


## 与 DataBinder 的区别与联系
| 特性         | Spring Framework `DataBinder`                                                           | Spring Boot `Binder`                                                                                         |
| :----------- | :-------------------------------------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------- |
| **框架归属** | Spring Framework Core（org.springframework.validation）  | Spring Boot（org.springframework.boot.context.properties.bind） |
| **主要用途** | 将 **请求参数** 或 **Map** 绑定到 Command Object。主要用于 **Web 层** 和 **表单提交**。 | 将 **Environment** (配置属性) 中的键值对绑定到 **Configuration Properties Bean**。主要用于 **启动/配置层**。 |
| **数据源**   | `PropertyValue` (如 HTTP 请求参数、Map Entries)。                                       | `ConfigurationPropertySource` (从 Environment 提取的属性)。                                                  |
| **命名规则** | **严格匹配**。字段名必须与源属性名精确匹配。       | **宽松绑定(Relaxed-Binding)** 。支持 Kebab-case, Snake-case, Dot notation 等多种命名风格。                   |
| **类型转换** | 依赖于 **ConversionService** 和 **PropertyEditor**。                                    | 依赖于 **ConversionService** (通常是 Spring Boot 默认配置的)；有更强大的集合和复杂类型支持。                 |
| **核心目标** | 方便地处理用户输入数据，实现 MVC 中的 Command 模式。                                    | 灵活、高性能地处理外部配置，支持层次结构和多种命名约定。                                                     |