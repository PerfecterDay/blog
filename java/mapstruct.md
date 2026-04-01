# Mapstruct
{docsify-updated}

> https://mapstruct.org/documentation/stable/reference/html/  
> https://www.baeldung.com/mapstruct  
> https://www.baeldung.com/java-mapstruct-mapping-collections


MapStruct 的核心理念是生成尽可能接近手动编写的代码。具体而言，不同对象值的拷贝传递是通过简单的 `getter/setter` 调用从源对象复制到目标对象，而非借助反射或其他类似机制实现。

## 依赖与插件
```
<properties>
    <org.mapstruct.version>1.6.3</org.mapstruct.version>
</properties>
...
<dependencies>
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>${org.mapstruct.version}</version>
    </dependency>
</dependencies>
...
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.8.1</version>
            <configuration>
                <annotationProcessorPaths>
                    <path>
                        <groupId>org.mapstruct</groupId>
                        <artifactId>mapstruct-processor</artifactId>
                        <version>${org.mapstruct.version}</version>
                    </path>
                </annotationProcessorPaths>
            </configuration>
        </plugin>
    </plugins>
</build>
```

## IDEA 插件
[MapStruct Support](https://plugins.jetbrains.com/plugin/10036-mapstruct-support)

## 定义一个 Mapper

### 基础映射
要创建映射器，只需定义一个包含所需映射方法的Java接口，并为其添加 `org.mapstruct.Mapper` 注解，MapStruct 代码生成器在构建时会扫描 `@Mapper` 注解并为接口创建实现类：
```
@Mapper
public interface CarMapper {
    @Mapping(target = "manufacturer", source = "make")
    @Mapping(target = "seatCount", source = "numberOfSeats")
    CarDto carToCarDto(Car car);

    @Mapping(target = "fullName", source = "name")
    PersonDto personToPersonDto(Person person);
}

//生成的代码
public class CarMapperImpl implements CarMapper {

    @Override
    public CarDto carToCarDto(Car car) {
        if ( car == null ) {
            return null;
        }

        CarDto carDto = new CarDto();

        if ( car.getFeatures() != null ) {
            carDto.setFeatures( new ArrayList<String>( car.getFeatures() ) );
        }
        carDto.setManufacturer( car.getMake() );
        carDto.setSeatCount( car.getNumberOfSeats() );
        carDto.setDriver( personToPersonDto( car.getDriver() ) );
        carDto.setPrice( String.valueOf( car.getPrice() ) );
        if ( car.getCategory() != null ) {
            carDto.setCategory( car.getCategory().toString() );
        }
        carDto.setEngine( engineToEngineDto( car.getEngine() ) );

        return carDto;
    }

    @Override
    public PersonDto personToPersonDto(Person person) {
        //...
    }

    private EngineDto engineToEngineDto(Engine engine) {
        if ( engine == null ) {
            return null;
        }

        EngineDto engineDto = new EngineDto();

        engineDto.setHorsePower(engine.getHorsePower());
        engineDto.setFuel(engine.getFuel());

        return engineDto;
    }
}
```
在生成的方法实现中，源类型（例如 Car）的所有可读属性都将被复制到目标类型（例如 CarDto）的对应属性中：
+ 当属性与其目标实体对应项名称相同时，将隐式映射该属性。
+ 当属性在目标实体中具有不同名称时，可通过 `@Mapping` 注解指定名称的对应关系。

如示例所示，生成的代码会考虑通过 `@Mapping` 注解指定的任何名称映射。若源实体与目标实体中映射属性的**类型不同**，MapStruct将执行**自动转换**（例如对price属性，参见隐式类型转换），或可选地调用/创建另一个映射方法（例如对`driver/engine` 属性，参见对象引用映射）。MapStruct 仅在满足以下条件时创建新映射方法：源属性与目标属性均为 Bean 属性，且它们本身是Bean或简单属性（即非集合或映射类型属性）。

具有相同元素类型的集合类型属性将通过创建目标集合类型的全新实例来实现复制，该实例包含源属性的所有元素。对于具有不同元素类型的集合类型属性，每个元素将单独映射并添加到目标集合中，**调用集合元素的映射器来转换添加**。

MapStruct 会考虑源类型和目标类型中的所有公共属性，**包括在超类型上声明的公共属性**。

### 增加自定义转换逻辑
```
@Mapper
public interface CarMapper {

    @Mapping(...)
    ...
    CarDto carToCarDto(Car car);

    default PersonDto personToPersonDto(Person person) {
        //hand-written mapping logic
    }
}
```
这样在将 `carToCarDto` 中会自动调用 `personToPersonDto` 方法将 `Car.Person` 转换为 `PersonDto` 对象并设置到 `CarDto 对应属性中`.

映射器也可以采用抽象类的形式定义，而非接口，并在映射器类中直接实现自定义方法。此时 MapStruct 将生成该抽象类的扩展类，并实现所有抽象方法。相较于声明默认方法，此方法的优势在于可在映射器类中声明额外字段。
```
@Mapper
public abstract class CarMapper {
    @Mapping(...)
    ...
    public abstract CarDto carToCarDto(Car car);

    public PersonDto personToPersonDto(Person person) {
        //hand-written mapping logic
    }
}
```

### 固定值
在某些时候，目标对象中有一个属性是源目标中没有的，我们想在每次转换时赋值一个固定的值，可以使用 `constant` ，这样做：
```
public interface UserMapper {
    @Mapping(target = "status", constant = "ACTIVE")
    UserDTO toDto(UserEntity entity);
}
```

也可以使用 `@AfterMapping` :
```
@Mapper(componentModel = "spring")
public interface ProductMapper {

    ProductDTO toDto(ProductEntity entity);

    @AfterMapping
    default void fillStatus(@MappingTarget ProductDTO dto) {
        dto.setStatus("DEFAULT");
    }
}
```



## 类型转换
当类型需要转换时，比如时间转字符串：
```
@Mapping(target="employeeId", source = "entity.id")
@Mapping(target="employeeName", source = "entity.name")
@Mapping(target="employeeStartDt", source = "entity.startDt",
           dateFormat = "dd-MM-yyyy HH:mm:ss")
EmployeeDTO employeeToEmployeeDTO(Employee entity);

@Mapping(target="id", source="dto.employeeId")
@Mapping(target="name", source="dto.employeeName")
@Mapping(target="startDt", source="dto.employeeStartDt",
           dateFormat="dd-MM-yyyy HH:mm:ss")
Employee employeeDTOtoEmployee(EmployeeDTO dto);
```

## 执行java语句
当某个字段需要特殊处理时：
```
// List<String> 转 String
@Mapper
public interface PublishVersionRequst2AppVersionEntityMapper {

    @Mapping(target = "memo",expression = "java(cn.hutool.core.util.StrUtil.toString(request.getMemo()))")
    AppVersionEntity map(PublishVersionRequest request);
}
```

## 集合转换
转换 List 集合类型时，必须声明元素类型的转换方法：
```
@Mapper
public abstract class TradeNews2MessageMapper {

    public abstract List<UserMessageVo> toUserMessageVoList(List<TradeNews> tradeNews);

    @BeforeMapping
    protected void beforeMapping(TradeNews tradeNews, @MappingTarget UserMessageVo messageVo) {
        messageVo.setReadState(StrUtil.equals(tradeNews.getFstate(), "New") ? 0 : 1);
    }

    @Mapping(source = "ftitle", target = "title")
    @Mapping(source = "fdate", target = "time")
    @Mapping(source = "fstate", target = "readState")
    @Mapping(source = "fno", target = "msgId")
    protected abstract UserMessageVo toUserMessageVo(TradeNews tradeNews);

}
```

## 命名方法
当有多个转换方法可调用时，需要使用 `@Named` 区别各个方法，并在转 List 的方法上使用 `@IterableMapping` 注解指明要使用的方法：
```
    @Mapping(target = "memo", expression = "java(cn.hutool.json.JSONUtil.toList(version.getMemo(),String.class))")
    @Named("toHistoryVersionResponse")
    HistoryVersionResponse toHistoryVersionResponse(AppVersionEntity version);

    @Mapping(target = "memo", expression = "java(cn.hutool.json.JSONUtil.toList(version.getMemoHk(),String.class))")
    @Named("toHistoryVersionResponseHk")
    HistoryVersionResponse toHistoryVersionResponseHk(AppVersionEntity version);

    @IterableMapping(qualifiedByName = "toHistoryVersionResponse")
    List<HistoryVersionResponse> toHistoryVersionResponseList(List<AppVersionEntity> versions);

    @IterableMapping(qualifiedByName = "toHistoryVersionResponseHk")
    List<HistoryVersionResponse> toHistoryVersionResponseListHk(List<AppVersionEntity> versions);
```

## 问题
### 插件冲突
与 Lombok 配合出现：
No property named "sms" exists in source parameter/No property named "sms" exists in result parameter
需要先用Lombok处理后才能用mapstruct，加入下列配置：
```
<build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>3.8.1</version>
                    <configuration>
                        <source>${java.version}</source>
                        <target>${java.version}</target>
                        <encoding>${project.build.sourceEncoding}</encoding>
                        <annotationProcessorPaths>
                            <path>
                                <groupId>org.projectlombok</groupId> <!-- IMPORTANT - LOMBOK BEFORE MAPSTRUCT -->
                                <artifactId>lombok</artifactId>
                                <version>${lombok.version}</version>
                            </path>
                            <path>
                                <groupId>org.mapstruct</groupId>
                                <artifactId>mapstruct-processor</artifactId>
                                <version>${org.mapstruct.version}</version>
                            </path>
                        </annotationProcessorPaths>
                    </configuration>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
```
### IDEA
使用IDEA build 和 maven compile ，生成的代码不一样。maven compile 生成的代码正确

## 复杂转换-多个输入组成一个对象
I have Car:
+ id
+ brand
+ model
+ owner

And CarDTO:
+ id
+ brand
+ model

In my service class I'm passing additional parameter "owner" and I need to convert the list.  
Is it possible to add "owner" to Mapper?  
If yes then I suppose it should be something similar to this (not working).
```
@Mapper
public interface CarMapper {

@Mapping(target = "owner", source = "owner")
List<Car> mapCars(List<CarDTO> cars, String owner);
}
```

Firstly, add a single object mapping method:
```
@Maping(target = "owner", source = "owner")
Car mapCar(CarDTO car, String owner);
```
Then define a method for mapping a list of objects with @Context:

```List<Car> mapCars(List<CarDTO> cars, @Context String owner);```

Since @Context parameters are not meant to be used as source parameters, a proxy method should be added to point MapStruct to the right single object mapping method to make it work.

In the end, add the proxy method:
```
default Car mapContext(CarDTO car, @Context String owner) {
    return mapCar(car, owner);
}
```

```
@Mapper
public interface CustomerScrResultMapper {

    @Mapping(target = "tradeAccount", source = "tradeAccount")
    @Mapping(target = "scrCheckSeq", source = "scrSeq")
    @Mapping(target = "targetItemId", source = "result.targetItemName")
    CustomerScrResult transFromMatchResult(MatchItemResult result, String tradeAccount, String scrSeq);

    //    @Mapping(source = "targetItemName",target = "target_item_name")
//    @Mapping(source = "fundValue",target = "fund_value")
//    @Mapping(source = "customerValue",target = "customer_value")
//    @Mapping(source = "matchResult", target = "target_item_name")
    default CustomerScrResult mapContext(MatchItemResult matchItemResult, @Context ScrContext context) {
        return transFromMatchResult(matchItemResult, context.getCustomer().getTradeAccount(), context.getScrSeq());
    }

    List<CustomerScrResult> transFromMatchResults(List<MatchItemResult> resultList,
                                                  @Context ScrContext context);

}
```


## 使用 Mapper
1. 没有使用 DI 依赖注入，使用 `org.mapstruct.factory.Mappers` 的 `getMapper` 方法：
```
CarMapper mapper = Mappers.getMapper( CarMapper.class );
```
按惯例，映射器接口应定义名为 `INSTANCE` 的成员，该成员持有映射器类型的单个实例：
```
@Mapper
public interface CarMapper {

    CarMapper INSTANCE = Mappers.getMapper( CarMapper.class );

    CarDto carToCarDto(Car car);
}
```

2. 当使用了依赖注入框架时
```
@Mapper(componentModel = MappingConstants.ComponentModel.CDI)
public interface CarMapper {

    CarDto carToCarDto(Car car);
}

@Inject
private CarMapper mapper;
```

