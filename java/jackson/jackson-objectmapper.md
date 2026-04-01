# Jackson-ObjectMapper
{docsify-updated}

> https://www.baeldung.com/jackson

- [Jackson-ObjectMapper](#jackson-objectmapper)
		- [引入依赖](#引入依赖)
		- [ObjectMapper](#objectmapper)
			- [Jackson 与 Java 集合](#jackson-与-java-集合)
			- [configure 方法](#configure-方法)
			- [自定义序列化器和反序列化器](#自定义序列化器和反序列化器)
			- [处理时间（ java.util.Date ）](#处理时间-javautildate-)
	- [问题](#问题)

Jackson是常用的JSON处理工具。

### 引入依赖
```
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.13.3</version>
</dependency>
```
这依赖会间接引入：
+ jackson-annotations
+ jackson-core

### ObjectMapper
Jackson 最常用的类就是 `ObjectMapper` 类，它有两个最常用的方法系列：
+ `writeValue()/writeValueAsxxx()`：将 Java 对象序列化成 Json 字符串
+ `readValue()/readValueAsxxx()`：将Json字符串反序列化成 Java 对象
+ `readTree()`：将Json字符串反序列化成Jackson内置的 JsonNode

之所以说是两个系列，是因为它有诸如 `writeValueAsString()` 、 `writeValueAsBytes()` 等方法。

```
ObjectMapper objectMapper = new ObjectMapper();
Car car = new Car("yellow", "renault");
objectMapper.writeValue(new File("target/car.json"), car);
objectMapper.writeValueAsString(car);

String json = "{ \"color\" : \"Black\", \"type\" : \"BMW\" }";
Car car = objectMapper.readValue(json, Car.class);	
Car car = objectMapper.readValue(new File("src/test/resources/json_car.json"), Car.class);
Car car = 
  objectMapper.readValue(new URL("file:src/test/resources/json_car.json"), Car.class);

String json = "{ \"color\" : \"Black\", \"type\" : \"FIAT\" }";
JsonNode jsonNode = objectMapper.readTree(json);
String color = jsonNode.get("color").asText();
// Output: color -> Black

```
**需要注意的是， ObjectMapper 在序列化时只会序列化类中的 public 成员，或者是提供了标准 getter 方法的成员，它是调用 getter 方法获取对象的成员值的。类似的，在反序列化时， ObjectMapper 需要调用类的默认构造方法和相应的 setter 方法为对应的成员赋值，所以一定要为类编写默认构造器和相应的 getter/setter 方法。**

#### Jackson 与 Java 集合
如果要通过Jackson反序列化一个Json字符串为 Java 集合的时候，通常要用到 `TypeReference`。
```
String jsonCarArray = 
  "[{ \"color\" : \"Black\", \"type\" : \"BMW\" }, { \"color\" : \"Red\", \"type\" : \"FIAT\" }]";
List<Car> listCar = objectMapper.readValue(jsonCarArray, new TypeReference<List<Car>>(){});

String json = "{ \"color\" : \"Black\", \"type\" : \"BMW\" }";
Map<String, Object> map 
  = objectMapper.readValue(json, new TypeReference<Map<String,Object>>(){});
```

#### configure 方法
通过 ObjectMapper 的 `configure` 方法能自定义Jackson的许多行为。这个方法是针对 ObjectMapper 对象的，后边会介绍许多注解，这些注解定义的一些行为是指针对特定POJO。类似于全局配置和局部配置的意思。

1. 反序列化时忽略不存在的属性 (`DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES`)

	如果Json 字符串中存在一个或多个类中没有定义的属性（setter），那么反序列化时通常会导致 `UnrecognizedPropertyException` 异常。
	```
	String jsonString 
		= "{ \"color\" : \"Black\", \"type\" : \"Fiat\", \"year\" : \"1970\" }";

	objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
	Car car = objectMapper.readValue(jsonString, Car.class);

	JsonNode jsonNodeRoot = objectMapper.readTree(jsonString);
	JsonNode jsonNodeYear = jsonNodeRoot.get("year");
	String year = jsonNodeYear.asText();
	```
2. 是否允许 null 的反序列化(`DeserializationFeature.FAIL_ON_NULL_FOR_PRIMITIVES`)
	`objectMapper.configure(DeserializationFeature.FAIL_ON_NULL_FOR_PRIMITIVES, false);`
3. 是否允许枚举到数字的转换(`DeserializationFeature.FAIL_ON_NUMBERS_FOR_ENUMS`)
	`objectMapper.configure(DeserializationFeature.FAIL_ON_NULL_FOR_PRIMITIVES, false);`
4. 忽略大小写（`MapperFeature.ACCEPT_CASE_INSENSITIVE_PROPERTIES`）
	`objectMapper.configure(MapperFeature.ACCEPT_CASE_INSENSITIVE_PROPERTIES,true);`

还有许多其他的配置可以参考[官网](https://github.com/FasterXML/jackson-databind/wiki/Serialization-Features)

#### 自定义序列化器和反序列化器
当POJO与Json字符串的结构不同时，通常我么需要自定义序列化器和反序列化器来满足我们的需求。通常，当我们要操作的类中包含了另一个自定义类的时候，就需要自定义序列化器和反序化器了。

1. 自定义序列化器
	```
	public class CustomCarSerializer extends StdSerializer<Car> {
		
		public CustomCarSerializer() {
			this(null);
		}

		public CustomCarSerializer(Class<Car> t) {
			super(t);
		}

		@Override
		public void serialize(
		Car car, JsonGenerator jsonGenerator, SerializerProvider serializer) {
			jsonGenerator.writeStartObject();
			jsonGenerator.writeStringField("car_brand", car.getType());
			jsonGenerator.writeEndObject();
		}
	}

	ObjectMapper mapper = new ObjectMapper();
	SimpleModule module = 
	new SimpleModule("CustomCarSerializer", new Version(1, 0, 0, null, null, null));
	module.addSerializer(Car.class, new CustomCarSerializer());
	mapper.registerModule(module);
	Car car = new Car("yellow", "renault");
	String carJson = mapper.writeValueAsString(car);
	```
2. 自定义反序列化器
	```
	public class CustomCarDeserializer extends StdDeserializer<Car> {
    
		public CustomCarDeserializer() {
			this(null);
		}

		public CustomCarDeserializer(Class<?> vc) {
			super(vc);
		}

		@Override
		public Car deserialize(JsonParser parser, DeserializationContext deserializer) {
			Car car = new Car();
			ObjectCodec codec = parser.getCodec();
			JsonNode node = codec.readTree(parser);
			
			// try catch block
			JsonNode colorNode = node.get("color");
			String color = colorNode.asText();
			car.setColor(color);
			return car;
		}
	}

	String json = "{ \"color\" : \"Black\", \"type\" : \"BMW\" }";
	ObjectMapper mapper = new ObjectMapper();
	SimpleModule module =
	new SimpleModule("CustomCarDeserializer", new Version(1, 0, 0, null, null, null));
	module.addDeserializer(Car.class, new CustomCarDeserializer());
	mapper.registerModule(module);
	Car car = mapper.readValue(json, Car.class);
	```
#### 处理时间（ java.util.Date ）
```
ObjectMapper objectMapper = new ObjectMapper();
DateFormat df = new SimpleDateFormat("yyyy-MM-dd HH:mm a z");
objectMapper.setDateFormat(df);
String carAsString = objectMapper.writeValueAsString(request);
// output: {"car":{"color":"yellow","type":"renault"},"datePurchased":"2016-07-03 11:43 AM CEST"}
```
https://stackoverflow.com/questions/21384820/is-there-a-jackson-datatype-module-for-jdk8-java-time

## 问题
字符串反序列化时自动转换为科学技术法的问题