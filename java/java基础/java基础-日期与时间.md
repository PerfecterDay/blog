# 日期与时间
{docsify-updated}

日期-时间API由主包`java.time`和四个子包组成：

+ `java.time`  
表示日期和时间的API的核心。它包括日期、时间、日期和时间的组合、时区、瞬时、持续时间和时钟等类。这些类是基于ISO-8601中定义的日历系统，并且是不可变的和线程安全的。
+ `java.time.Chrono`  
用于表示除默认的ISO-8601之外的日历系统的API。你也可以定义你自己的日历系统。本教程不涉及这个包的任何细节。
+ `java.time.format`  
用于格式化和解析日期和时间的类。
+ `java.time.temporal`  
扩展的API，主要针对框架和库的编写者，允许日期和时间类之间的互操作，查询，和调整。字段（TemporalField和ChronoField）和单位（TemporalUnit和ChronoUnit）都在这个包中定义。
+ `java.time.zone`  
支持时区、时区偏移和时区规则的类。如果要处理时区问题，大多数开发者只需要使用 `ZonedDateTime`，以及`ZoneId`或`ZoneOffset`。

### 设计原则
1. 不可变  
日期-时间API中的大多数类创建的对象都是不可变的，这意味着，在对象创建后，它不能被修改。要改变一个不可变的对象的值，必须构建一个新的对象作为原始对象的修改副本。这也意味着，根据定义，日期-时间API是线程安全的。这对API的影响在于，大多数用于创建日期或时间对象的方法都以of、from或with为前缀，而不是构造函数，并且没有设置方法。比如说：
```
LocalDate dateOfBirth = LocalDate.of(2012, Month.MAY, 14);
LocalDate firstBirthday = dateOfBirth.plusYears(1);
```
2. 流式调用  
日期-时间API提供了一个流式调用的接口，使代码易于阅读。因为大多数方法不允许带有空值的参数，也不返回空值，所以方法调用可以被串联起来，由此产生的代码可以被快速理解。比如说：
```
LocalDate today = LocalDate.now();
LocalDate payday = today.with(TemporalAdjusters.lastDayOfMonth()).minusDays(2);
```
3. 可扩展  
日期-时间API是可扩展的，只要有可能。例如，你可以定义你自己的时间调整器和查询，或建立你自己的日历系统。


### Instant-机器时间
Java 用 `Instant` 表示时间线上的某个点，**可以用来表示时间戳或者称为机器时间**。时间线的原点被设置为穿过伦敦格林威治天文台的本初子午线所处时区的1970年1月1日的午夜。
+ `Instant.MIN` 的值可向回追溯10亿年
+ `Instant.MAX` 是公1000000000年的12月31日
+ `Instant.now()`：返回当前的时刻

为了得到两个时刻之间的时间差，可以使用 `Duration Duration.between(Instant start,Instant end)` 方法。`Duration` 代表两个时刻之间的时间差。可以通过以下一些方法来获得 `Duration` 表示的按照常规单位度量的时间长度。
`toNonos`,`toMills`,`toSeconds`,`toMinutes`,`toHours`,`toDays`。

Duration 对象的内部存储所需的空间超过了一个 long 的值 ，因 此秒数存储在一个long 中，而纳秒数存储在一个额外的 int 中。

### 本地日期时间
机器时间对机器来说是很好识别的，但是对人类来说却不是很方便。Java中有两种人类时间：本地日期/时间和时区时间。

本地日期/时间包含日期和当天的时间，但是不包含任何时区信息。这种时间不能对应精确的机器时间。因为，同一个日期在不同时区对应的机器时间是不同的。时区时间能表示一个精确的机器时间。通俗的解释就是，本地时间可以用来代表一个时间字符串，如 "2023-06-15 00:00:00" ，但是这个字符串所代表的精确时间对不同时区的人来说是不一样的。

`LocalDate`代表带有年月日的日期。为了构建 `LocalDate` 对象可以是该类的静态方法 `now` 或 `of`：
```
LocalDate localDate = LocalDate.now();
LocalDate localDate2 = LocalDate.of(1990,1,19);
localDate = LocalDate.of(1990, Month.JANUARY,19);
Period period = localDate.until(localDate2);
long days = localDate.until(localDate2, ChronoUnit.DAYS);
```
类似与两个 `Instant` 之间的时长用 `Duration` 表示，两个本地时间之间的时长用 `Period` 表示，**代表了流逝的年、月或日的数量。** `DUration`转换成年月日可能需要其他的方法，而 `Period` 可以直接得到这些信息。

与`LocalDate`类似的还有`LocalTime`和 `LocalDateTime`，表示的本地时间。
```
LocalTime time1 = LocalTime.now();
LocalTime time2 = LocalTime.of(8,20,0);
LocalDateTime localDateTime = LocalDateTime.now();
localDateTime = LocalDateTime.of(localDate,time1);
```

#### 日期调整器
对于日程安排应用来说，经常需要计算诸如"每个月的第一个星期二"这种计算。 `TemporalAdjusters` 提供了大量的常见调整的静态方法。
<center><img src="pics/TemporalAdjusters.jpg" width="70%"></center>

也可以通过实现 `TemporalAdjuster` 接口自定义自己的调整器。

### 时区时间
Java 使用 `ZonedDateTime` 来代表时区时间。`ZoneId` 代表不同的时区。  
`ZonedDateTime apollolllaunch = ZonedDateTime.of(1969, 7, 16 , 9, 32, 0, 0, Zoneid. of("America/New_York")`

这是一个具体的时刻，调用 `apollolllaunch.tolnstant()` 可以获得对应的 `Instant` 对象。 反过来，如果你有一个时刻对象，调用 `instant.atZone(Zoneld.of("UTC"))` 可以获得格林威治皇家天文台的 `ZonedDateTime` 对象，或者使用其他的 `ZoneId` 获得这个时刻在地球上其他地方代表的时区时间 `ZonedDateTime` 。

### 日期/时间与字符串的互转

#### 日期时间转字符串

`DateTimeFormatter` 提供了三种不同的日期/时间格式器：
1. 预定义的标准格式器  
<center><img src="pics/predefined-formatter.jpg" width="70%"></center>

这些预定义的格式器作为 `DateTimeFormatter` 的静态成员，可以直接通过调用`format()`方法来格式化日期时间。
```
	String str = DateTimeFormatter.ISO_LOCAL_DATE_TIME.format(localDateTime);
```

2. Locale 相关的格式器  
标准格式器主要是为了机器刻度的时间而设置的，表示人类时间可以用Locale相关的格式化器。
	<center><img src="pics/locale-dateformatter.jpg" width="70%"></center>

	`DateTimeFormatter`的静态方法`ofLocalizedDate`,`ofLocalizedTime`,`ofLocalizedDateTime`可以创建这种格式器。
	```
	DateTimeFormatter formatter = DateTimeFormatter.ofLocalizedDateTime(FormatStyle.LONG);
	String str = formatter.withLocale(Locale.CHINA).format(localDateTime);
	```

3. 定制模式的格式器
```
	DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
	formatter.format(localDatetime);
```
格式定义如下，注意严格区分大小写：
<center><img src="pics/formater.jpg" width="70%"></center>

#### 字符串转时间
为了解析字符串中的日期／时间值，可以使用众多的静态 `parse` 方法之一:
```
LocalDateTime.parse("1990-01-01 1:00:00");
LocalDate.parse("1990-01-01");
ZonedDateTime zonedDateTime = ZonedDateTime.parse("1990-01-01 1:00:00",DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
```

### 与老的时间类的转换
<center><img src="pics/date-transform.png" width="90%"></center>

### Period 和 Duration
这两个类都可以用来表示时间量或确定两个日期之间的差值。这两个类的主要区别在于，Period 使用基于日期的值,用来表示多少年月日，而 Duration 使用基于时间的值，用来表示多少天时分秒。

`Period` 类使用年、月、日单位来表示一段时间。
通过使用 `between()` 方法，我们可以获得一个`Period`对象，作为两个日期之间的差值，并且可以使用 `getYears()`、`getMonths()` 和 `getDays()` 方法确定`Period`的日期单位：
```
LocalDate startDate = LocalDate.of(2015, 2, 20);
LocalDate endDate = LocalDate.of(2017, 1, 15);

Period period = Period.between(startDate, endDate);

LOG.info("Years:" + period.getYears() + 
  " months:" + period.getMonths() + 
  " days:"+period.getDays());
```

另一种创建`Period`对象的方法是使用专用方法根据天数、月数、周数或年数创建：
```
Period fromUnits = Period.of(3, 10, 10);
Period fromDays = Period.ofDays(50);
Period fromMonths = Period.ofMonths(5);
Period fromYears = Period.ofYears(10);
Period fromWeeks = Period.ofWeeks(40);

assertEquals(280, fromWeeks.getDays());
```

还可以通过解析文本序列来创建周期对象，文本序列的格式必须是 `PnYnMnD`：
```
Period fromCharYears = Period.parse("P2Y");
assertEquals(2, fromCharYears.getYears());

Period fromCharUnits = Period.parse("P2Y3M5D");
assertEquals(5, fromCharUnits.getDays());
```


`Duration` 类表示以秒或纳秒为单位的时间间隔，最适合在需要更高精度的情况下处理较短的时间量。
要根据文本序列创建 "持续时间 "对象，其形式必须是 `PnDTnHnMn.nS`：
```
Instant start = Instant.parse("2017-10-03T10:15:30.00Z");
Instant end = Instant.parse("2017-10-03T10:16:30.00Z");
        
Duration duration = Duration.between(start, end);

Duration fromChar1 = Duration.parse("P1DT1H10M10.5S");
Duration fromChar2 = Duration.parse("PT10M");
```