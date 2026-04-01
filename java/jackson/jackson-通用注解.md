# Jackson 通用注解
{docsify-updated}

- [Jackson 通用注解](#jackson-通用注解)
    - [忽略特定的属性/字段：`@JsonIgnoreProperties({ "id" })`](#忽略特定的属性字段jsonignoreproperties-id-)
    - [忽略特定属性：`@JsonIgnore`](#忽略特定属性jsonignore)
    - [忽略所有特定类型的属性：`@JsonIgnoreType`](#忽略所有特定类型的属性jsonignoretype)
    - [忽略掉null的属性：`@JsonInclude(Include.NON_NULL)`](#忽略掉null的属性jsonincludeincludenon_null)
    - [指定需要序列化的字段：`@JsonIncludeProperties`](#指定需要序列化的字段jsonincludeproperties)
    - [指定key名：`@JsonProperty("keyName")`](#指定key名jsonpropertykeyname)
    - [指定格式/忽略大小写：`@JsonFormat`](#指定格式忽略大小写jsonformat)
    - [`JsonUnwrapped`](#jsonunwrapped)
    - [`@JsonView`](#jsonview)
    - [循环引用问题：`@JsonIdentityInfo`](#循环引用问题jsonidentityinfo)


### 忽略特定的属性/字段：`@JsonIgnoreProperties({ "id" })`
类级别的注解
```
@JsonIgnoreProperties({ "id" })
public class BeanWithIgnore {
    public int id;
    public String name;
}
```
要无例外地忽略 JSON 输入中的任何未知属性，我们可以设置 `@JsonIgnoreProperties(ignoreUnknown=true)`。

### 忽略特定属性：`@JsonIgnore`
```
public class BeanWithIgnore {
    @JsonIgnore
    public int id;

    public String name;
}
```

### 忽略所有特定类型的属性：`@JsonIgnoreType`
```
public class User {
    public int id;
    public Name name;

    @JsonIgnoreType
    public static class Name {
        public String firstName;
        public String lastName;
    }
}
```

### 忽略掉null的属性：`@JsonInclude(Include.NON_NULL)`
```
@JsonInclude(Include.NON_NULL)
public class MyBean {
    public int id;
    public String name;
}
```

### 指定需要序列化的字段：`@JsonIncludeProperties`
@JsonIncludeProperties 是请求最多的 Jackson 功能之一。它在 Jackson 2.12 中引入，可用于标记一个属性或一个属性列表，Jackson 将在序列化和反序列化过程中包含这些属性。
```
@JsonIncludeProperties({ "name" })
public class BeanWithInclude {
    public int id;
    public String name;
}
```

### 指定key名：`@JsonProperty("keyName")`
```
public class MyBean {
    public int id;
    private String name;

    @JsonProperty("name")
    public void setTheName(String name) {
        this.name = name;
    }

    @JsonProperty("name")
    public String getTheName() {
        return name;
    }
}
```

### 指定格式/忽略大小写：`@JsonFormat`
@JsonFormat 注解指定了日期/时间值序列化时的格式。
```
public class EventWithFormat {
    public String name;
    @JsonFormat(
      shape = JsonFormat.Shape.STRING,
      pattern = "dd-MM-yyyy hh:mm:ss")
    public Date eventDate;
}

@Data
@JsonPropertyOrder(alphabetic = true)
@JsonFormat(with = JsonFormat.Feature.ACCEPT_CASE_INSENSITIVE_PROPERTIES)
public class OrderSyncResult {
    private String orderNo;
    private String failReason;
}
```

### `JsonUnwrapped`
```
public class UnwrappedUser {
    public int id;

    @JsonUnwrapped
    public Name name;

    public static class Name {
        public String firstName;
        public String lastName;
    }
}
```

### `@JsonView`
@JsonView 表示在序列化/解序列化时将包含该属性的视图。
```
public class Views {
    public static class Public {}
    public static class Internal extends Public {}
}

public class Item {
    @JsonView(Views.Public.class)
    public int id;

    @JsonView(Views.Public.class)
    public String itemName;

    @JsonView(Views.Internal.class)
    public String ownerName;
}
```

### 循环引用问题：`@JsonIdentityInfo`
@JsonIdentityInfo 表示在序列化/反序列化值时，例如在处理无限递归类型的问题时，应使用对象标识。
```
@JsonIdentityInfo(
  generator = ObjectIdGenerators.PropertyGenerator.class,
  property = "id")
public class ItemWithIdentity {
    public int id;
    public String itemName;
    public UserWithIdentity owner;
}

@JsonIdentityInfo(
  generator = ObjectIdGenerators.PropertyGenerator.class,
  property = "id")
public class UserWithIdentity {
    public int id;
    public String name;
    public List<ItemWithIdentity> userItems;
}
```
