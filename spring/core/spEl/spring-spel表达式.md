#  SpEL
{docsify-updated}

## 编程式 API
文本表达式支持字符串、日期、数字（正数、实数及八进制、十六进制等）、布尔类型、null。其中字符串必须使用单引号或者`\"`(反斜杠+双引号)括起来。
```
ExpressionParser expressionParser = new SpelExpressionParser();
String helloWorld = expressionParser.parseExpression("\"Hello World\"").getExpressionString();
String helloWorld2 = expressionParser.parseExpression("'Hello World'").getExpressionString();
Double doubleValue = expressionParser.parseExpression("6.098").getValue(Double.class);
Integer intVal = expressionParser.parseExpression("34").getValue(Integer.class);
Boolean boolVal = expressionParser.parseExpression("true").getValue(Boolean.class);


User user = new User();
user.setUserName("tom");
user.setCredits(100);
ExpressionParser parser = new SpelExpressionParser();
EvaluationContext context = new StandardEvaluationContext(user);
String username =(String)parser.parseExpression("userName").getValue(context);
```

在创建 `StandardEvaluationContext` 实例时，指定一个根对象作为求值目标对象，这样在求值表达式中就可以引用根对象属性。在求值内部可以使用反射机制从注册对象中获取相应的属性值。

在单独使用SpEL时，需要创建一个 `ExpressionParser` 解析器，并提供一个 `EvaluationContext` 求值上下文。在普通的基于Spring的应用开发中，这些API一般很少会涉及，仅需编写SpEL表达式字符串即可。例如，在Bean定义的时候使用SpEL表达式，只需写好相应的表达式字符串即可，至于解析器、求值上下文、根对象和其他预定义变量等基础设施，Spring都会创建。

## 基础
Spring EL 的基础语法： `#{<expression string>}`

- 字符串：`'hello'`
- 数字：`123`, `3.14`
- 布尔值：`true`, `false`
- 空值：`null`

## 运算符
- 算术：`1 + 2`, `5 % 2`
- 关系：`3 > 2`, `2 == 2`
- 逻辑：`true and false`, `a or b`, `!a`
- 三元：`a > b ? a : b`
- Elvis：`name ?: 'default'`

## 对象访问
- 属性访问：`person.name` → 相当于 `getName()`
- 方法调用：`person.getName()`
- 安全导航：`person?.address` → 避免 `NullPointerException`
- 数组/集合：`list[0]`, `map['key']`


## 集合操作
- 过滤：`list.?[age > 18]` → 返回所有符合条件的元素
- 投影：`list.![name]` → 提取字段，结果是新集合
- 第一个：`list.^[age > 18]`
- 最后一个：`list.$[age > 18]`


## 类与静态方法
- 类引用：`T(java.lang.Math)`
- 静态方法：`T(java.lang.Math).random()`
- 静态字段：`T(java.lang.Math).PI`


## Bean 引用
- Bean 引用：`@myService`
- Bean 方法：`@myService.doSomething()`
- 环境属性：`environment['JAVA_HOME']`


## 字符串模板
- SpEL 表达式：`#{...}` -->  计算表达式
- 属性占位符：`${...}` -->  配置占位符

示例：
```java
@Value("#{2 * T(java.lang.Math).PI}") // SpEL，计算表达式
private double piTimes2;

@Value("${server.port}") // 占位符，读取配置
private int serverPort;

#{systemProperties['os.name']} //获取系统变量名
#{mybean.name} //获取某个bean 的name 属性
```