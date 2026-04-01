
# Jackson反序列化注解
{docsify-updated}

> https://www.baeldung.com/jackson-annotations

## 使用 `@JsonCreator` 注解来调整反序列化中使用的构造器/工厂。
```
{
    "id":1,
    "theName":"My bean"
}

public class BeanWithCreator {
    public int id;
    public String name;

    @JsonCreator
    public BeanWithCreator(
      @JsonProperty("id") int id, 
      @JsonProperty("theName") String name) {
        this.id = id;
        this.name = name;
    }
}
```

## `@JacksonInject`
```
public class BeanWithInject {
    @JacksonInject
    public int id;
    
    public String name;
}

@Test
public void whenDeserializingUsingJsonInject_thenCorrect()
  throws IOException {
 
    String json = "{\"name\":\"My bean\"}";
    
    InjectableValues inject = new InjectableValues.Std()
      .addValue(int.class, 1);
    BeanWithInject bean = new ObjectMapper().reader(inject)
      .forType(BeanWithInject.class)
      .readValue(json);
    
    assertEquals("My bean", bean.name);
    assertEquals(1, bean.id);
}
```

## 平铺设置 Map 属性:`@JsonAnySetter`
```
public class ExtendableBean {
    public String name;
    private Map<String, String> properties;

    @JsonAnySetter
    public void add(String key, String value) {
        properties.put(key, value);
    }
}

{
    "name":"My bean",
    "attr2":"val2",
    "attr1":"val1"
}
```

## 使用指定方法设置属性：`@JsonSetter`
```
public class MyBean {
    public int id;
    private String name;

    @JsonSetter("name")
    public void setTheName(String name) {
        this.name = name;
    }
}
```

## `@JsonDeserialize`
```
public class EventWithSerializer {
    public String name;

    @JsonDeserialize(using = CustomDateDeserializer.class)
    public Date eventDate;
}

public class CustomDateDeserializer
  extends StdDeserializer<Date> {

    private static SimpleDateFormat formatter
      = new SimpleDateFormat("dd-MM-yyyy hh:mm:ss");

    public CustomDateDeserializer() { 
        this(null); 
    } 

    public CustomDateDeserializer(Class<?> vc) { 
        super(vc); 
    }

    @Override
    public Date deserialize(
      JsonParser jsonparser, DeserializationContext context) 
      throws IOException {
        
        String date = jsonparser.getText();
        try {
            return formatter.parse(date);
        } catch (ParseException e) {
            throw new RuntimeException(e);
        }
    }
}
```

## 为反序列化过程中的属性定义了一个或多个别名：`@JsonAlias`
```
public class AliasBean {
    @JsonAlias({ "fName", "f_name" })
    private String firstName;   
    private String lastName;
}

"{\"fName\": \"John\", \"lastName\": \"Green\"}";
```



## 反序列化范型
```
List<User> users =
        jsonMapper.readValue(json,
                new TypeReference<List<User>>() {});

ApiResponse<User> response =
        jsonMapper.readValue(json,
                new TypeReference<ApiResponse<User>>() {});
```