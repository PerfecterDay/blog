
# Jackson序列化注解
{docsify-updated}

> https://www.baeldung.com/jackson-annotations

- [Jackson序列化注解](#jackson序列化注解)
      - [用指定的方法来序列化某个属性：`@JsonGetter`](#用指定的方法来序列化某个属性jsongetter)
      - [平铺输出：`@JsonAnyGetter`](#平铺输出jsonanygetter)
      - [指定 key 的顺序：`@JsonPropertyOrder`](#指定-key-的顺序jsonpropertyorder)
      - [指示 Jackson 按原样序列化属性：`@JsonRawValue`](#指示-jackson-按原样序列化属性jsonrawvalue)
      - [指定序列化整个实例的方法: `@JsonValue`](#指定序列化整个实例的方法-jsonvalue)
      - [使用指定 key 包装 json: `@JsonRootName`](#使用指定-key-包装-json-jsonrootname)
      - [使用自定义序列化器：`@JsonSerialize`](#使用自定义序列化器jsonserialize)
      - [序列化时忽略某些属性](#序列化时忽略某些属性)
      - [@JsonAnyGetter](#jsonanygetter)

#### 用指定的方法来序列化某个属性：`@JsonGetter`
Jackson序列化时默认调用标准的 getter 方法获取对象值，如果我们想要使用特定方法作为Jackson获取对象值的方法，就可以用 `@JsonGetter` 标注该方法：
```
public class MyBean {
    public int id;
    private String name;

    @JsonGetter("name")
    public String getTheName() {
        return name;
    }
}
```

#### 平铺输出：`@JsonAnyGetter`
@JsonAnyGetter 注解允许灵活地将Map字段的key平铺为外层对象的标准属性。
```
public class ExtendableBean {
    public String name;
    private Map<String, String> properties;

    @JsonAnyGetter
    public Map<String, String> getProperties() {
        return properties;
    }
}

{
    "name":"My bean",
    "attr2":"val2",
    "attr1":"val1"
}
```

#### 指定 key 的顺序：`@JsonPropertyOrder` 
我们可以使用 @JsonPropertyOrder 注解来指定序列化时属性的顺序。
```
@JsonPropertyOrder({ "name", "id" })
public class MyBean {
    public int id;
    public String name;
}
{
    "name":"My bean",
    "id":1
}
```
还可以使用 `@JsonPropertyOrder(alphabetic=true)` 按字母顺序排列属性。在这种情况下，序列化的输出结果将是
```
{
    "id":1,
    "name":"My bean"
}
```

#### 指示 Jackson 按原样序列化属性：`@JsonRawValue`
```
public class RawBean {
    public String name;

    @JsonRawValue
    public String json;
}
{
    "name":"My bean",
    "json":{
        "attr":false
    }
}
```

#### 指定序列化整个实例的方法: `@JsonValue`


#### 使用指定 key 包装 json: `@JsonRootName`
```
@JsonRootName(value = "user")
public class UserWithRoot {
    public int id;
    public String name;
}

{
    "user":{
        "id":1,
        "name":"John"
    }
}

@JsonRootName(value = "user", namespace="users")
public class UserWithRootNamespace {
    public int id;
    public String name;
}

<user xmlns="users">
    <id xmlns="">1</id>
    <name xmlns="">John</name>
    <items xmlns=""/>
</user>
```

#### 使用自定义序列化器：`@JsonSerialize`
```
public class EventWithSerializer {
    public String name;

    @JsonSerialize(using = CustomDateSerializer.class)
    public Date eventDate;
}

public class CustomDateSerializer extends StdSerializer<Date> {

    private static SimpleDateFormat formatter 
      = new SimpleDateFormat("dd-MM-yyyy hh:mm:ss");

    public CustomDateSerializer() { 
        this(null); 
    } 

    public CustomDateSerializer(Class<Date> t) {
        super(t); 
    }

    @Override
    public void serialize(
      Date value, JsonGenerator gen, SerializerProvider arg2) 
      throws IOException, JsonProcessingException {
        gen.writeString(formatter.format(value));
    }
}
```


#### 序列化时忽略某些属性
如果我们在序列化时需要过滤掉某些属性，可以使用下面注解：
1. 忽略某个属性，类级别注解：`@JsonIgnoreProperties`
	```
	@JsonIgnoreProperties(value = { "intValue" })
	public class MyDto {

		private String stringValue;
		private int intValue;
		private boolean booleanValue;

		public MyDto() {
			super();
		}

		// standard setters and getters are not shown
	}
	```
	这样不管是序列化还是反序列化，intValue 的值都是不会额外设置的。由构造器设置值。
2. **忽略所有未知属性，类级别注解**：`@JsonIgnoreProperties(ignoreUnknown = true)`
3. 属性级别的注解：`JsonIgnore`
	```
	public class MyDto {

		private String stringValue;
		@JsonIgnore
		private int intValue;
		private boolean booleanValue;

		public MyDto() {
			super();
		}

		// standard setters and getters are not shown
	}
	```
3. 忽略特定类型的所有属性，类级别注解：`@JsonIgnoreType`。当一个被 @JsonIgnoreType 注解的类作为其他类的属性时，在序列化和反序列化期间将忽略该属性。**注意只有在该类作为其他类的属性时才会被忽略，直接序列化、反序列化该类本身是正常的。**
4. 使用过滤器过滤属性
	```
	@JsonFilter("myFilter")
	public class MyDtoWithFilter { ... }

    ObjectMapper mapper = new ObjectMapper();
    SimpleBeanPropertyFilter theFilter = SimpleBeanPropertyFilter
      .serializeAllExcept("intValue");
    FilterProvider filters = new SimpleFilterProvider()
      .addFilter("myFilter", theFilter);

    MyDtoWithFilter dtoObject = new MyDtoWithFilter();
    String dtoAsString = mapper.writer(filters).writeValueAsString(dtoObject);
	```

#### @JsonAnyGetter
`@JsonAnyGetter` 可以允许将Map类型的属性中的key-value作为类型本身的属性输出。比如：
```
public class ExtendableBean {
    public String name;
    private Map<String, String> properties;

    @JsonAnyGetter
    public Map<String, String> getProperties() {
        return properties;
    }
}

序列化输出时将会得到：
{
    "name":"My bean",
    "attr2":"val2",
    "attr1":"val1"
}
```