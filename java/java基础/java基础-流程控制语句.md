# 流程控制语句
{docsify-updated}

无论哪种编程语言，都会提供两种基本的流程控制结构：
+ **分支结构**
+ **循环结构**

## 分支结构
Java 提供了两种常见的分支控制结构： **if 语句** 和 **switch 语句** 。

###  if 语句
if 语句使用布尔表达式或布尔值作为分支条件来进行分支控制，if 语句有如下三种形式：

```
if(condition){
    statement
}
```

```
if(condition){
    statement1
}else{
    statement2
}
```

```
if(contio1){
    statement1
}else if(condition2){
    statement2
}
....//中间可以有多个 else if
else{
    statementn;
}
```
等价于
```
if(contio1){
    statement1
}else{
    if(condition2){
        statement2
    }else{
        if()
    }
}
```
**注意**：第三种形式的 if 语句有个容易踩的坑，看下面代码:
```
int age = 45;
if(age > 20){
    print("青年人")
}else if(age > 40){
    print("中年人")
}else if(age > 60){
    print("老年人")
}
```
很有可能以为会打印 "青年人" 和 "中年人"，实际上只会打印 "青年人"。因为 else 本来就包含了对之前一个 condition 的取反。**也就是说 `else if` 语句实际上是在之前的 `condition` 都不满足的情况下且满足当前条件才会执行。**

### switch 语句
`switch` 语句后面的表达式类型只支持 `byte` 、 `short` `、char` 、 `int` 四种整数类型（不支持 `long` 类型）， `String` (Java 7开始支持)和枚举类型。
基本语法：
```
switch(expression){
    case(condition1):
        statements2;
        break;
    case(condition2):
        statements2;
        break;
    case(condition3):
        statements3;
        break;
        ...
    default:
        statements
}
```
switch 语句会先计算出表达式的值，然后拿着个表达式的值和 case 标签后的值比较，一旦遇到相等的值，程序就开始执行这个 case 标签后的代码，不再判断后边的 case 、 default 标签的条件是否匹配，除非遇到 break ，如果所有的 case 标签都不匹配，则执行 default 标签后的语句。**和 if 语句中的 else 类似， default 看似没有条件，其实是有条件的，条件就是 expression 不能与 case 标签后的值相等。**

## 循环结构
循环结构用于当满足条件的时候重复执行一段语句。一般循环语句包含四个部分：
1. 初始化语句：用于一些初始化操作，在循环开始之前执行
2. 循环条件：一个 `boolean` 表达式，这个表达式的结果决定循环是否执行
3. 循环体：当条件满足时，重复执行的语句
4. 迭代语句：一般在循环体结束后执行（有可能包含在循环体中），在对循环条件求值之前执行，用于控制循环条件中的变量，使得循环在合适的时候结束，避免死循环

循环结构需要注意的是**一定要让循环条件能够变成 `false` ,避免死循环。**

1. while 循环
```
[init_statement]
while(test_expression){
    statements;
    [iteration_statment]
}
```
在这种结构中，对 `test_expression` 的测试求值一定比循环体**多执行一次**，当最后一次执行 `test_expression` 为 `false` 时，循环体不会被执行。

2. do while 循环
```
[init_statement]
do{
    statement;
    [iteration_statment]
}while(test_expression);
```
最后的分号必须有，do while循环至少会执行一次，即使循环条件一开始不满足。这种结构中， `test_expression` 的测试求值和循环体**执行次数一样**，当最后一次执行 `test_expression` 为 `false` 时，循环体不会被执行，但是最开始多执行了一次。

3. for 循环
```
for([init_statement]; [test_expression];[iteration_statment]){
    statements
}
```
每次执行 for 循环时，先执行初始化语句，初始化语句只在循环开始之前执行一次；每次执行循环体之前，先判断一下循环条件，满足条件才去执行循环体，循环体结束后，执行迭代语句。因此对于for 循环而言，**循环条件判断永远比循环体多执行一次**，最后一次为 `false` 时，不执行循环体。
而且，因为迭代语句和循环体是分开的，因此即使循环体中有 `continue` 语句跳过本次循环，**迭代语句也会执行**。

### 循环控制语句
1. `continue`: 跳过本次循环剩下未执行的语句，继续下一次循环。
2. `break`: 默认直接结束循环
3. `break 标号`：可以在嵌套循环的内部结束外部循环，标号必须标在外部循环上。
    ```
    System.out.println("MainClass1 Start...");
        outer:
        for (int i = 0; i < 10; i++) {
            for (int j = 0; j < 10; j++) {
                System.out.println("i=" + i + ",j=" + j);
                if (j == 5)
                    break outer;
            }
        }
        
        System.out.println("MainClass1 End.");
    ```
