# Spring 使用JDBC访问数据库
{docsify-updated}

对照学习，请参考 Java 中 [JDBC](/java/java基础/java基础-JDBC访问数据库.md) 访问数据库的一般步骤。

## 使用 JdbcTemplate
`JdbcTemplate` 是 JDBC 核心包中的核心类。它处理资源的创建和释放，处理JDBC连接的打开与关闭。它执行核心 JDBC 工作流的基本任务（如语句创建和执行），让应用代码只关心如何提供 SQL 查询和根据结果集映射成 Java 对象。 `JdbcTemplate` 类可以：
+ 运行 SQL 查询
+ 更新语句和存储过程调用
+ 对 ResultSet 实例执行迭代，并提取返回的参数值。
+ 捕获 JDBC 异常，并将其转换为 org.springframework.dao 包中定义的通用、信息量更大的异常层次结构。

一旦配置完成， `JdbcTemplate` 实例就是线程安全的，都很少需要在每次运行 SQL 时创建一个新的 JdbcTemplate 类实例。如果应用程序需要访问多个数据库，则可能需要多个 `JdbcTemplate` 实例，这就需要多个数据源以及多个不同配置的 `JdbcTemplate` 实例。

### JdbcTemplate查询
```java
int countOfActorsNamedJoe = this.jdbcTemplate.queryForObject(
		"select count(*) from t_actor where first_name = ?", Integer.class, firstName);

// 如果需要将结果集映射成Java对象使用 RowMapper
private final RowMapper<Actor> actorRowMapper = (resultSet, rowNum) -> {
	Actor actor = new Actor();
	actor.setFirstName(resultSet.getString("first_name"));
	actor.setLastName(resultSet.getString("last_name"));
	return actor;
};

// 如果需要直接处理结果集可以使用 ResultSetExtractor
private final ResultSetExtractor<String> stringExtractor = (resultSet, rowNum) -> {
	System.out.println(resultSet.getString("first_name"));
	return resultSet.getString("last_name");
};

public List<Actor> findAllActors() {
	return this.jdbcTemplate.query("select first_name, last_name from t_actor", actorRowMapper);
}

Actor actor = jdbcTemplate.queryForObject(
		"select first_name, last_name from t_actor where id = ?", actorRowMapper, 123L);

Actor actor = jdbcTemplate.queryForObject(
		"select first_name, last_name from t_actor where id = ?", stringExtractor, 123L);
```

### JdbcTemplate执行增删改等更新
```java
this.jdbcTemplate.update(
		"insert into t_actor (first_name, last_name) values (?, ?)",
		"Leonor", "Watling");

this.jdbcTemplate.update(
		"update t_actor set last_name = ? where id = ?",
		"Banjo", 5276L);

this.jdbcTemplate.update(
		"delete from t_actor where id = ?",
		Long.valueOf(actorId));
```

### 批量插入

```
public class JdbcActorDao implements ActorDao {

	private JdbcTemplate jdbcTemplate;

	public void setDataSource(DataSource dataSource) {
		this.jdbcTemplate = new JdbcTemplate(dataSource);
	}

	public int[] batchUpdate(final List<Actor> actors) {
		return this.jdbcTemplate.batchUpdate(
				"update t_actor set first_name = ?, last_name = ? where id = ?",
				new BatchPreparedStatementSetter() {
					public void setValues(PreparedStatement ps, int i) throws SQLException {
						Actor actor = actors.get(i);
						ps.setString(1, actor.getFirstName());
						ps.setString(2, actor.getLastName());
						ps.setLong(3, actor.getId().longValue());
					}
					public int getBatchSize() {
						return actors.size();
					}
				});
	}
}

public class JdbcActorDao implements ActorDao {

	private JdbcTemplate jdbcTemplate;

	public void setDataSource(DataSource dataSource) {
		this.jdbcTemplate = new JdbcTemplate(dataSource);
	}

	public int[] batchUpdate(final List<Actor> actors) {
		List<Object[]> batch = new ArrayList<>();
		for (Actor actor : actors) {
			Object[] values = new Object[] {
					actor.getFirstName(), actor.getLastName(), actor.getId()};
			batch.add(values);
		}
		return this.jdbcTemplate.batchUpdate(
				"update t_actor set first_name = ?, last_name = ? where id = ?",
				batch);
	}

	// ... additional methods
}
```
上述示例中的所有的 List 都会作为一批数据一次性插入数据库中，如果要插入的批数据特别大，可能会对数据库造成很大的压力。因此，我们可能需要分多批次的批处理插入数据。

#### 多批次批处理插入
```
public class JdbcActorDao implements ActorDao {

	private JdbcTemplate jdbcTemplate;

	public void setDataSource(DataSource dataSource) {
		this.jdbcTemplate = new JdbcTemplate(dataSource);
	}

	public int[][] batchUpdate(final Collection<Actor> actors) {
		int[][] updateCounts = jdbcTemplate.batchUpdate(
				"update t_actor set first_name = ?, last_name = ? where id = ?",
				actors,
				100,
				(PreparedStatement ps, Actor actor) -> {
					ps.setString(1, actor.getFirstName());
					ps.setString(2, actor.getLastName());
					ps.setLong(3, actor.getId().longValue());
				});
		return updateCounts;
	}
}
```
此调用的批次更新方法返回一个 int[][] 二维数组，其中包含每个批次的数组条目，以及每个更新的受影响行数数组。顶层数组的长度表示运行的批次数，第二层数组的长度表示该批次中的更新数。根据提供的更新对象总数，每个批次中的更新次数应是所有批次（最后一个批次除外，可能会少一些）的批次大小。每个更新语句的更新次数是 JDBC 驱动程序报告的次数。如果没有更新计数，JDBC 驱动程序会返回一个 -2 的值。

### JdbcTemplate建表和执行存储过程
您可以使用 execute(..) 方法运行任意 SQL。因此，该方法经常用于 DDL 语句。该方法有很多重载变体，包括回调接口、绑定变量数组等。下面的示例创建了一个表：
```java
this.jdbcTemplate.execute("create table mytable (id integer, name varchar(100))");
```

下边的示例调用了一个存储过程：
```java
this.jdbcTemplate.update(
		"call SUPPORT.REFRESH_ACTORS_SUMMARY(?)",
		Long.valueOf(unionId));
```

## 统一 JDBC 查询/更新操作: JdbcClient
从 6.1 开始， `NamedParameterJdbcTemplate` 的命名参数语句和普通 `JdbcTemplate` 的位置参数语句可通过具有流畅交互模型的统一客户端 API- `JdbcClient` 使用。
```java
private JdbcClient jdbcClient = JdbcClient.create(dataSource);

public int countOfActorsByFirstName(String firstName) {
	return this.jdbcClient.sql("select count(*) from t_actor where first_name = ?")
			.param(firstName)
			.query(Integer.class).single();
}

public int countOfActorsByFirstName(String firstName) {
	return this.jdbcClient.sql("select count(*) from t_actor where first_name = :firstName")
			.param("firstName", firstName)
			.query(Integer.class).single();
}

List<Actor> actors = this.jdbcClient.sql("select first_name, last_name from t_actor")
		.query((rs, rowNum) -> new Actor(rs.getString("first_name"), rs.getString("last_name")))
		.list();
```

JdbcClient 是用于 JDBC 查询/更新语句的灵活而简化的界面。批量插入和存储过程调用等高级功能通常需要额外的定制。可以考虑使用 Spring 的 `SimpleJdbcInsert` 和 `SimpleJdbcCall` 类或直接使用 `JdbcTemplate` 来实现 `JdbcClient` 中不具备的功能。

## 获取自动生成的主键
`update()` 方法支持检索数据库生成的主键。该支持是 JDBC 3.0 标准的一部分。该方法的第一个参数是 `PreparedStatementCreator` ，这是指定所需插入语句的方式。另一个参数是 `KeyHolder` ，其中包含更新成功返回时生成的主键。下面的示例适用于 Oracle，但可能不适用于其他平台：
```java
final String INSERT_SQL = "insert into my_test (name) values(?)";
final String name = "Rob";

KeyHolder keyHolder = new GeneratedKeyHolder();
jdbcTemplate.update(connection -> {
	PreparedStatement ps = connection.prepareStatement(INSERT_SQL, new String[] { "id" });
	ps.setString(1, name);
	return ps;
}, keyHolder);
```
