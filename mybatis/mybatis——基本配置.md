# mybatis 基本配置
{docsify-updated}


mybatis 的配置文件的顶层结构如下：
```
+ configuration 配置
    + [properties 属性](#properties)
    + [settings 设置](#settings)
    + [typeAliases 类型别名](#typeAliases)
    + [typeHandlers 类型处理器](#typeHandlers)
    + [objectFactory 对象工厂](#objectFactory)
    + [plugins 插件](#plugins)
    + [environments 环境](#environments)
        * [environment 环境变量](#environment)
            - [transactionManager 事务管理器](#transactionManager)
            - [dataSource 数据源](#dataSource)
    + [databaseIdProvider 数据库厂商标识](#databaseIdProvider)
    + [mappers 映射器](#mappers)
```

## <span id="properties">properties</span>
这些属性都是可外部配置且可动态替换的，既可以在典型的 Java 属性文件中配置，亦可通过 properties 元素的子元素来传递。例如：

    <properties resource="org/mybatis/example/config.properties">
      <property name="username" value="dev_user"/>
      <property name="password" value="F2Fa3!33TYyg"/>
    </properties>
这个例子中的 username 和 password 将会由 properties 元素中设置的相应值来替换。 driver 和 url 属性将会由 config.properties 文件中对应的值来替换。这样就为配置提供了诸多灵活选择。

属性也可以被传递到 SqlSessionFactoryBuilder.build()方法中。例如：

    SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(reader, props);
    // ... or ...
    SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(reader, environment, props);
如果属性在不只一个地方进行了配置，那么 MyBatis 将按照下面的顺序来加载：

+ 在 properties 元素体内指定的属性首先被读取。
+ 然后根据 properties 元素中的 resource 属性读取类路径下属性文件或根据 url 属性指定的路径读取属性文件，并覆盖已读取的同名属性。
+ 最后读取作为方法参数传递的属性，并覆盖已读取的同名属性。

因此，通过方法参数传递的属性具有最高优先级，resource/url 属性中指定的配置文件次之，最低优先级的是 properties 属性中指定的属性。

## <span id="settings">settings</span>
## <span id="typeAliases">typeAliases</span>
类型别名是为 Java 类型设置一个短的名字。它只和 XML 配置有关，存在的意义仅在于用来减少类完全限定名的冗余。例如:

    <typeAliases>
      <typeAlias alias="Author" type="domain.blog.Author"/>
      <typeAlias alias="Blog" type="domain.blog.Blog"/>
      <typeAlias alias="Comment" type="domain.blog.Comment"/>
      <typeAlias alias="Post" type="domain.blog.Post"/>
      <typeAlias alias="Section" type="domain.blog.Section"/>
      <typeAlias alias="Tag" type="domain.blog.Tag"/>
    </typeAliases>
    当这样配置时，Blog可以用在任何使用domain.blog.Blog的地方。

也可以指定一个包名，MyBatis 会在包名下面搜索需要的 Java Bean，比如:

    <typeAliases>
      <package name="domain.blog"/>
    </typeAliases>
每一个在包 domain.blog 中的 Java Bean，在没有注解的情况下，会使用 Bean 的首字母小写的非限定类名来作为它的别名。 比如 domain.blog.Author 的别名为 author；若有注解，则别名为其注解值。看下面的例子：

    @Alias("author")
    public class Author {
        ...
    }
## <span id="typeHandlers">typeHandlers</span>
无论是 MyBatis 在预处理语句（PreparedStatement）中设置一个参数时，还是从结果集中取出一个值时，都会用类型处理器将获取的值以合适的方式转换成 Java 类型。

你可以重写类型处理器或创建你自己的类型处理器来处理不支持的或非标准的类型。 具体做法为：实现 `org.apache.ibatis.type.TypeHandler` 接口， 或继承一个很便利的类 `org.apache.ibatis.type.BaseTypeHandler`， 然后可以选择性地将它映射到一个 JDBC 类型。比如：

    // ExampleTypeHandler.java
    @MappedJdbcTypes(JdbcType.VARCHAR)
    public class ExampleTypeHandler extends BaseTypeHandler<String> {

      @Override
      public void setNonNullParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType) throws SQLException {
        ps.setString(i, parameter);
      }
      @Override
      public String getNullableResult(ResultSet rs, String columnName) throws SQLException {
        return rs.getString(columnName);
      }
      @Override
      public String getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        return rs.getString(columnIndex);
      }
      @Override
      public String getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        return cs.getString(columnIndex);
      }
    }

    <!-- mybatis-config.xml -->
    <typeHandlers>
      <typeHandler handler="org.mybatis.example.ExampleTypeHandler"/>
    </typeHandlers>

