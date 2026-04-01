#  Mysql 常用函数
{docsify-updated}

## 字符串函数
+ `ASCII(str)` : 返回值为字符串str 的**最左字符的数值**。假如str为空字符串，则返回值为 0 。假如str 为NULL，则返回值为NULL。 ASCII()用于带有从 0到255的数值的字符。
+ `BIN(N)` ：返回值为N的二进制值的字符串表示，其中 N 为一个longlong (BIGINT) 数字。这等同于 `CONV(N,10,2)`。假如N 为NULL，则返回值为 NULL。
+ `BIT_LENGTH(str)` :　返回值为二进制的字符串str 长度。
+ `CHAR(N,... [USING charset])` ： CHAR()将每个参数N理解为一个整数，其返回值为一个包含这些整数的代码值所给出的字符的字符串。NULL值被省略。
	`SELECT CHAR(77,121,83,81,'76');` -> 'MySQL'
	`SELECT CHAR(77,77.3,'77.3');` -> 'MMM'
+ `CHAR_LENGTH(str)`: 返回值为字符串str 的长度，长度的单位为**字符**。一个多字节字符算作一个单字符。对于一个包含5个2字节字符集, `LENGTH()`返回值为 10, 而`CHAR_LENGTH()`的返回值为5。
+ `CHARACTER_LENGTH(str)` : 等同于 `CHAR_LENGTH(str)`
+ `CONCAT(str1,str2,...) ` :连结字符串，返回结果为连接参数产生的字符串。如有任何一个参数为NULL ，则返回值为 NULL。或许有一个或多个参数。 如果所有参数均为非二进制字符串，则结果为非二进制字符串。 如果变量中含有任一二进制字符串，则结果为一个二进制字符串。一个数字参数被转化为与之相等的二进制字符串格式；若要避免这种情况，可使用显式类型 cast, 例如： `SELECT CONCAT(CAST(int_col AS CHAR), char_col)`
+ `CONCAT_WS(separator,str1,str2,...)` :　`CONCAT_WS()` 代表 `CONCAT With Separator` ，是`CONCAT()`的特殊形式。 第一个参数是其它参数的分隔符。分隔符的位置放在要连接的两个字符串之间。分隔符可以是一个字符串，也可以是其它参数。如果分隔符为 NULL，则结果为 NULL。函数会忽略任何分隔符参数后的 NULL 值。
+ `LEFT(str,len)` : 返回从字符串str 最左开始的len 个字符。
+ `LOWER(str)` :　转小写
+ `LENGTH(str)` : 返回值为字符串str 的长度，单位为字节。一个多字节字符算作多字节。这意味着 对于一个包含5个2字节字符的字符串， LENGTH() 的返回值为 10, 而 CHAR_LENGTH()的返回值则为5。
+ `LTRIM(str)` :　返回字符串 str ，其左侧开始的引导空字符被删除。
+ `REVERSE(str)` : 返回字符串 str ，顺序和字符顺序相反。
+ `RIGHT(str,len)` : 从字符串str结尾开始，返回最右len 个字符。
+ `RPAD(str,len,padstr)` : 返回填充后的字符串str, 其右边被字符串 padstr填补至len 字符长度。假如字符串str 的长度大于 len,则返回值被缩短到与 len 字符相同长度。
+ `RTRIM(str)` : 返回字符串 str ，结尾空格字符被删去。
+ `SPACE(N)` :　返回一个由N 空隔符号组成的字符串。
+ `SUBSTRING(str,pos) , SUBSTRING(str FROM pos) SUBSTRING(str,pos,len) , SUBSTRING(str FROM pos FOR len)`: 不带有len 参数的格式从字符串str返回一个**子字符串**，起始于位置 pos。带有len参数的格式从字符串str返回一个长度同len字符相同的子字符串，起始于位置 pos。`SUBSTR()` 是 `SUBSTRING()`的同义词。
+ `UPPER(str)` ：转大写
+ `REPLACE(field,src,to)`: 字符串替换
	```
	update info_content2 set `explain_content`=REPLACE(explain_content,"~nn",'') WHERE `explain_content` LIKE '%~nn%';
	update info_content2 set `cont`=REPLACE(cont,"~nn",'') WHERE `cont` LIKE '%~nn%';
	```

## 日期和时间函数
+ `ADDDATE(date,INTERVAL expr type) ADDDATE(expr,days)`: 返回日期相加后的日期，当被第二个参数的INTERVAL格式激活后， ADDDATE()就是DATE_ADD()的同义词。相关函数SUBDATE() 则是DATE_SUB()的同义词。
+ `ADDTIME(expr,expr2)` ：ADDTIME()将 expr2添加至expr 然后返回结果。 expr 是一个时间或时间日期表达式，而expr2 是一个时间表达式。
+ `CURDATE()` :　将**当前日期**按照'YYYY-MM-DD' 或YYYYMMDD 格式的值返回，具体格式根据函数用在字符串或是数字语境中而定。
+ `CURTIME()` :　将当前时间以'HH:MM:SS'或 HHMMSS 的格式返回， 具体格式根据函数用在字符串或是数字语境中而定。`CURRENT_TIME` 和`CURRENT_TIME()` 是`CURTIME()`的同义词。
+ `CURRENT_TIMESTAMP` : 返回当前时间，和`CURRENT_TIMESTAMP()` 、`NOW()`是同义词。
+ `DATE(expr)/YEAR(expr)/TIME(expr)` : 提取日期或时间日期表达式expr中的日期/年份/时间部分。 `SELECT DATE('2003-12-31 01:02:03');->  '2003-12-31'`
+ `DATEDIFF(expr,expr2)` : `DATEDIFF()` 返回起始时间 expr和结束时间expr2之间的天数。Expr和expr2 为日期或 date-and-time 表达式。计算中只用到这些值的日期部分。
+ `DATE_ADD(date,INTERVAL expr type) DATE_SUB(date,INTERVAL expr type)`: 这些函数执行日期运算。 date 是一个 DATETIME 或DATE值，用来指定起始时间。 expr 是一个表达式，用来指定从起始日期添加或减去的时间间隔值。  Expr是一个字符串;对于负值的时间间隔，它可以以一个 ‘-’开头。type 为关键词，它指示了表达式被解释的方式。 
+ `FROM_UNIXTIME(unix_timestamp) , FROM_UNIXTIME(unix_timestamp,format)`: 返回'YYYY-MM-DD HH:MM:SS'或YYYYMMDDHHMMSS 格式值的unix_timestamp参数表示，具体格式取决于该函数是否用在字符串中或是数字语境中。
+ `LAST_DAY(date)`: 获取一个日期或日期时间值，返回该月最后一天对应的值。若参数无效，则返回NULL。
+ `NOW()`: 返回当前日期和时间值，其格式为 'YYYY-MM-DD HH:MM:SS' 或YYYYMMDDHHMMSS ， 具体格式取决于该函数是否用在字符串中或数字语境中。
+ `QUARTER(date)`: 返回date 对应的一年中的季度值，范围是从 1到 4。
+ `STR_TO_DATE(str,format)`: 这是DATE_FORMAT() 函数的倒转。它获取一个字符串 str 和一个格式字符串format。若格式字符串包含日期和时间部分，则 STR_TO_DATE()返回一个 DATETIME 值， 若该字符串只包含日期部分或时间部分，则返回一个 DATE 或TIME值。`ELECT STR_TO_DATE('04/31/2004', '%m/%d/%Y'); -> '2004-04-31'`
+ `UNIX_TIMESTAMP(), UNIX_TIMESTAMP(date)` :若无参数调用，则返回一个Unix timestamp ('1970-01-01 00:00:00' GMT 之后的秒数) 作为无符号整数。若用date 来调用UNIX_TIMESTAMP()，它会将参数值以'1970-01-01 00:00:00' GMT后的秒数的形式返回。date可以是一个DATE 字符串、一个 DATETIME字符串、一个 TIMESTAMP或一个当地时间的YYMMDD或YYYMMDD格式的数字。

## 数值处理函数
+ `Abs()` : 返回绝对值
+ `Cos()` : 返回余弦值
+ `Exp()` : 返回指数
+ `Mod()` : 返回余数
+ `Pi()` : 返回圆周率
+ `Rand()` : 返回一个随机数
+ `Sin()` : 返回正弦值
+ `Sqrt()` : 返回平方根
+ `Tan()` : 返回正切值

## JSON 处理函数
```
select su.id,su.username,su.email,su.status,su.login_date,su.create_time,GROUP_CONCAT(sur.role_id) as roles from system_users su left join system_user_role sur on su.id=sur.user_id left join system_role sr on sur.role_id=sr.id group by su.id,su.username,su.email,su.status,su.login_date,su.create_time\G;

//查询用户的角色ID和名字
select su.id,su.username,su.email,su.status,su.login_date,su.create_time,JSON_ARRAYAGG(JSON_OBJECT('id', sr.id, 'name', sr.name)) AS roles from system_users su left join system_user_role sur on su.id=sur.user_id left join system_role sr on sur.role_id=sr.id group by su.id,su.username,su.email,su.status,su.login_date,su.create_time\G;
```