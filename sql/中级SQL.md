# 中级 SQL
{docsify-updated}

<center><img src="pics/sql-join.jpg" width="50%"></center>

## 连接表达式
连接运算把两个关系的属性连接成一个关系，并按照一定规则筛选、连接两个关系中的元组以放入新的连接后的关系结果中。  
内连接：不保留未匹配元组的连接运算。  
外连接：保留某个关系或全部关系的全部元组的连接运算。  

### 自然连接

**自然连接** 运算作用于两个关系，并产生一个关系作为结果。
不同于笛卡尔积（笛卡尔积将第一个关系的每个元组与第二个关系的所有元组都依次进行连接）；自然连接只考虑那些在两个关系模式中的相同属性上取值相同的元组对。默认情况下，要求所有共同属性上的取值相同；可以使用 `using` 子句指定要求相同的列。两个关系没有公共属性，那么连接运算将变成两个关系的笛卡尔积运算。

A和B的自然连接：**来自A表的元组和来自B表的元组在共同属性（p1,p2...）上取值相同的元组进行连接。**

假设A、B含有共同属性（p1,p2,p3），自然连接可以用 *natural join* 表示：   
`A natural join B`（p1,p2,p3）上的取值都相同的元组进行连接  
`A join B using (p1)` p1 上值相同的元组进行连接。

### on 表达式
自然连接默认连接的是两个关系中 **相同属性上(表中列名相同的列)** 值相同的元组。不同属性不会被考虑在内。SQL的 on 表达式可以指定任意的连接条件。on 只能出现在连接表达式的末尾。
```
<select id="selectLoginGroupMsg" resultMap="grpMsgResultMap">
	SELECT m.id, m.title,m.subtitle,m.category,m.subcategory,m.url,mc.name as subcategoryName,
	m.priority,m.unix_create_time_stamp as unixCreateTimeStamp,
	case
		when ugm.state=1 then 1
		else 0
	end as readState
	FROM
	GROUP_MSG gp LEFT JOIN MESSAGE m ON gp.msg_id=m.id
	LEFT JOIN USER_GROUP_MSG ugm ON (gp.msg_id = ugm.msg_id and ugm.uid = #{uid})
	LEFT JOIN msg_category mc ON m.subcategory = mc.id
	<where>
		date(gp.CREATE_TIME) &gt;= date_sub(curdate(), interval 7 day)
		AND gp.state = 1
	</where>
	ORDER BY m.PRIORITY DESC
</select>
```

### 外连接

1. 左外连接（left outer join | left join）：只保留出现在左外连接运算之前（左边）的关系中的元组。
2. 右外连接（right outer join | right join）：只保留出现在右外连接运算之后（右边）的关系中的元组。
3. 全外连接（full outer join | full join）：保留出现在两个关系中的元组。

可以按照如下方式计算左外连接：首先，像计算内连接那样计算出内连接的结果；然后，对于内连接左侧关系中任意一个与右侧关系中任何元组都不匹配的元组t，向连接结果中加入一个元组r,r的构造如下：
+ 元组r从左侧关系得到的属性被赋予t中的值
+ r的其它属性（右侧关系）被赋值为nul
右外连接与左外连接是对称的。来自右侧关系中的不匹配左侧关系任何元组的元组被补上空值，并加入到右外连接的结果中。

全外连接是左外连接与右外连接的组合。先计算出内连接后，左侧关系中不匹配右侧关系任何元组的元组被添加上空值添加到结果中；类似的，右侧关系中不匹配左侧关系任何元组的元组也被添上空值加入到结果中。

### 内连接
为了把常规连接和外连接区分开来，SQL中把常规连接称作内连接。可以用 inner join 替换 outer join，inner 是可选的，当 join 子句中没有使用 outer 前缀，默认的连接类型是 inner join。

下图给出了所有连接类型的列表。任意的连接形式可以和任意的连接条件（自然连接、using条件连接或on条件连接）进行组合。
<center><img src="pics/join.jpg" alt=""></center>

### 连接更新
```
UPDATE LOGIN_RECORD L JOIN DEVICE D ON L.LOGIN_DEVICE=D.DEVICE_ID SET L.DEVICE_MODEL = D.DEVICE_MODEL;

UPDATE TABLE1 T1
    LEFT JOIN  TABLE2 T2
    ON T1.COLUMN1 = T2.COLUMN1
SET T1.COLUMN2 = T2.COLUMN2,
    T1.COLUMN3 = T2.COLUMN3
WHERE T1.COLUMN1 IN(21,31);
```

### 删除重复数据
```
UPDATE `allfunds_price` as a JOIN 
(SELECT max(`update_time_cpzx`) as update_time_cpzx, `isin`  , `pub_dt` 
FROM allfunds_price
GROUP BY isin, pub_dt) as b on a.`update_time_cpzx` = b.update_time_cpzx and a.`isin` = b.isin
and a.`pub_dt` =b.pub_dt set a.is_del = 1;
```

## 完整性约束
完整性约束保证授权用户对数据库所做的修改不会破坏数据的一致性，防止的是对数据库的以外破坏。

### 单个关系上的约束
0. primary key  
   主键约束

1. not null  
指定了 not null 的列，不允许插入 null 值。任何尝试插入 null 的操作都会失败

2. unique  
`unique(A1 ,A2, A3 ....)`
unique声明指出属性 A1 ,A2, A3 ....形成一个候选码；即在关系中没有两个元组能在所有列出的属性上取值相同。但是属性值可以为null。

3. check子句  
当应用于关系声明时（创建表时），check(P)子句指定一个谓词P，关系中的每个元组都必须满足谓词P。
    ```
    create table T(
        id varchar(10),
        season varchar(8),
        ....,
        primary key(id),
        check(season in('Spring','Summer','Autumn','Winter'))
    )
    ```

### 参照完整性（多个关系）

参照完整性用来保证一个关系中给定属性集上的取值也在另一个关系的特定属性集的取值中出现。

外键就是参照完整性约束。关系 r1 和 r2 的属性集分别为 R1 和 R2，主码分别为 K1 和 K2。如果要求对 r2 中任意元组 t2，均存在 r1 中的元组 t1 使得 t1.K1 = t2.\alpha ，我们称 R2 的子集 \alpha 为参照关系 r1 中 K1 的外键。
