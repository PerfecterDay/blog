#  MybatisPlus 多数据源
{docsify-updated}

> https://baomidou.com


```
@MapperScan("com.demo.gjyw.repo.mapper") // 指定扫描 mapper 的包

```

## 多数据源配置
1. 添加依赖
   ```
	<dependency>
		<groupId>com.baomidou</groupId>
		<artifactId>dynamic-datasource-spring-boot-starter</artifactId>
		<version>3.5.2</version>
	</dependency>
   ```
2. 配置数据源
   ```
   spring:
	datasource:
	  dynamic:
		primary: user #设置默认的数据源或者数据源组,默认值即为master
		strict: false
		datasource:
			user:
				url: jdbc:mysql://rm-3nsz59cw245j0v25w.mysql.rds.aliyuncs.com:3306/user?useSSL=false&useUnicode=true&characterEncoding=UTF-8
				username: mobileapp_svc
				password: p0BWm71pakWIZ3pH
				driver-class-name: com.mysql.cj.jdbc.Driver
			mmf:
				url: jdbc:mysql://rm-3nsz59cw245j0v25w.mysql.rds.aliyuncs.com:3306/mmf?useSSL=false&useUnicode=true&characterEncoding=UTF-8
				username: mobileapp_svc
				password: p0BWm71pakWIZ3pH
				driver-class-name: com.mysql.cj.jdbc.Driver
   ```
3. 使用 `@DS("user")` 注解
	```
	@Mapper
	@DS("user")
	public interface CouponMapper extends BaseMapper<CouponEntity> {
	}
	```

## Join 查询
```
<dependency>
    <groupId>com.github.yulichang</groupId>
    <artifactId>mybatis-plus-join-boot-starter</artifactId>
    <version>1.5.4</version>
</dependency>
```

示例：
```
@Data
public class User {
    private Long id;
    private String name;
    private Integer age;
    private String email;
}

@Data
public class Address {
    private Long id;
    private Long userId;
    private String city;
    private String address;
}

/**
 * 自定义resultType
 */
@Data
@ToString
public class UserDTO {
    private Long id;
    private String name;
    private Integer age;
    private String email;

    private String city;
    private String address;
}


public void testSelect() {
	MPJLambdaWrapper<User> wrapper = new MPJLambdaWrapper<User>()
			.selectAll(User.class)//查询user表全部字段
			.select(Address::getCity, Address::getAddress)
			.leftJoin(Address.class, Address::getUserId, User::getId);

	List<UserDTO> userList = userMapper.selectJoinList(UserDTO.class, wrapper);

	userList.forEach(System.out::println);
}
```