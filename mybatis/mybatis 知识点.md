
# mybatis 细节总结
{docsify-updated}

### xml文件中的特殊字符
当我们需要通过xml格式处理sql语句时，经常会用到< ，<=，>，>=等符号，但是很容易引起xml格式的错误，这样会导致后台将xml字符串转换为xml文档时报错，从而导致程序错误。

这样的问题在 mybatis 中或者自定义的xml处理sql的程序中经常需要我们来处理。其实很简单，我们只需作如下替换即可避免上述的错误：

| 原符号| < | <= | > | >= | & | ' | " |
| :------| :------|:------|:------|:------|:------|:------|:------|
| 替换符号 | \&lt; | \&lt;= | \&gt; | \&gt;= | \&amp; | \&apos;| \&quot;|

或者我们也可以使用 \<![CDATA[   ]]> 将特殊字符包含起来。


### 自增ID
解决自增ID不是从1开始，而是随机数问题：
`@TableId(value = “id”, type = IdType.AUTO)`

首先，如果你的数据库支持自动生成主键的字段（比如 MySQL 和 SQL Server），那么你可以设置 useGeneratedKeys=”true”，然后再把 keyProperty 设置为目标属性就 OK 了。
```
 <insert id="save" parameterType="com.demo.demo.repo.dao.entity.SmsEntity"
	useGeneratedKeys="true" keyProperty="id,create_time,update_time">
	INSERT INTO sms(mobile,sms_code,purpose) VALUES ('${mobile}','${smsCode}','${purpose}')
</insert>
```
For insert statement, the method return type must be void or int.
If you use int, the number of inserted rows (always 1 in your case) will be returned.
The generated key will be set to the id field of the parameter instance.
Also, you should specify keyColumn.

### 日志打印
mybatis-plus 打印数据库语句：
mybatis-plus:
  mapper-locations: classpath*:mappers/**/*Mapper.xml
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl #开启sql日志

Querywrapper 的 columns 参数是指定 select 的列



https://stackoverflow.com/questions/42387576/how-to-get-a-field-value-which-is-default-to-current-timestamp-in-an-insert-st

mybatis-plus:
  mapper-locations: classpath*:mappers/**/*Mapper.xml

### 参数引用
如果在方法中传入对象，需要使用 `@Param` 注解将参数命名，然后在XML文件中使用 #{author.name} 形式应用对象属性，在test语句中可以直接使用`author.name`
```
<select id="findActiveBlogLike"
     resultType="Blog">
  SELECT * FROM BLOG WHERE state = ‘ACTIVE’
  <if test="title != null">
    AND title like #{title}
  </if>
  <if test="author != null and author.name != null">
    AND author_name like #{author.name}
  </if>
</select>
```
### $ 和 # 区别
默认情况下，使用 `#{}` 参数语法时，MyBatis 会创建 PreparedStatement 参数占位符，并通过占位符安全地设置参数（就像使用 ? 一样）。 这样做更安全，更迅速，通常也是首选做法，不过有时你就是想直接在 SQL 语句中直接插入一个不转义的字符串。 比如 ORDER BY 子句，这时候你可以：
```ORDER BY ${columnName}```
这样，MyBatis 就不会修改或转义该字符串了。用这种方式接受用户的输入，并用作语句参数是**不安全**的，会导致潜在的 SQL 注入攻击。因此，要么不允许用户输入这些字段，要么自行转义并检验这些参数。
