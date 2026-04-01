# JSON 与 YAML 简介
{docsify-updated}


## JSON
在 JSON 中，最外层不仅可以是对象 `{}` 或数组 `[]`，也可以是单个合法的 **JSON 值**（value），数据类型是可以在编程语言中存储和操作的不同类型的值。每种数据类型都有自己的一套属性和行为。JSON 支持多种数据类型，包括：
+ **对象** ：JSON 对象数据类型是 {}（花括号）之间插入的一系列名称或值对。键必须是字符串，以逗号分隔，并且应当是唯一的。  `{"name":"tom"}`
+ **数组** ：数组数据类型是值的有序集合。在 JSON 中，数组值必须是字符串、数字、对象、数组、布尔值或 null。 `[1,2,3]`
+ **字符串** ：在 JSON 中，字符串用双引号括起来，可以包含任何 Unicode 字符，通常用于存储和传输基于文本的数据，如名称、地址或说明。`"abcd"，"1234"` 都是合法 json。
+ **布尔值** ：布尔值可指定为 true 或 false。布尔值不加引号，视为字符串值。 `true/false`
+ **Null** ：Null 代表有意留空的值。当没有为某个键分配值时，可以将其视为 `null` 值。 `null`
+ **数字** ：数字用于存储计算、比较或数据分析等各种用途的数值。JSON 支持正数、负数和小数点。JSON 数字遵循 JavaScript 的双精度浮点格式。 

## YAML
YAML 全称 YAML Ain't Markup Language， 它是一种以数据为中心的标记语言，比 XML 和 JSON 更适合作为配置文件。

由于YAML是一个JSON的超集，任何合法的JSON文档 都应该 是合法的YAML。

想要使用 YAML 作为属性配置文件（以 .yml 或 .yaml 结尾），需要将 `SnakeYAML` 库添加到 classpath 下，Spring Boot 中的spring-boot-starter-web 或 spring-boot-starter 都对 SnakeYAML 库做了集成， 只要项目中引用了这两个 Starter 中的任何一个，Spring Boot 会自动添加 SnakeYAML 库到 classpath 下。

### YAML 语法
YAML 的语法如下：
+ 使用缩进表示层级关系。
+ 缩进时不允许使用 Tab 键，只允许使用空格。
+ 缩进的空格数不重要，但同级元素必须左侧对齐。
+ 大小写敏感。

YAML 支持以下三种数据结构：
1. 对象：键值对的集合
2. 数组：一组按次序排列的值
3. 字面量：单个的、不可拆分的值

#### YAML 字面量写法
字面量是指单个的，不可拆分的值，例如：数字、字符串、布尔值、以及日期等。  
空字符串是 null。  
在 YAML 中，使用 `key:[空格]value` 的形式表示一对键值对（空格不能省略），如 `url: www.biancheng.net` 。  
字面量直接写在键值对的 `value` 中即可，且默认情况下字符串是不需要使用单引号或双引号的。

若字符串使用单引号，则不会转义特殊字符:  
`name: 'zhangsan \n lisi'`  输出为 `zhangsan \n lisi`

若字符串使用双引号，则会转义特殊字符，特殊字符会输出为其本身想表达的含义:
`name: "zhangsan \n lisi"` 输出为 
```
zhangsan 
lisi
```

+ 裸字没有引号，也没有转义，因此，必须小心使用字符。
+ 双引号字符串可以使用\转义指定字符。比如，"\"Hello\", she said"。可以使用\n转义换行。
+ 单引号字符串是字面意义的字符串，不用\转义，只有单引号''需要转义，转成单个'。

#### YAML 对象写法
在 YAML 中，对象可能包含多个属性，每一个属性都是一对键值对。 YAML 为对象提供了 2 种写法：
1. 普通写法，使用缩进表示对象与属性的层级关系。

```
website: 
  name: bianchengbang
  url: www.biancheng.net
```

1. 行内写法：`website: {name: bianchengbang,url: www.biancheng.net}`

#### YAML 数组写法
YAML 使用“-”表示数组中的元素，同样有两种写法。
1. 普通写法如下：

```
pets:
  -dog
  -cat
  -pig
```
2. 行内写法: `pets: [dog,cat,pig]`

### 多文档 YAML
一个 YAML 文件可以由一个或多个文档组成，文档之间使用“---”作为分隔符，且个文档相互独立，互不干扰。如果 YAML 文件只包含一个文档，则“---”分隔符可以省略。

```
---
website:
  name: bianchengbang
  url: www.biancheng.net
---
website: {name: bianchengbang,url: www.biancheng.net}
pets:
  -dog
  -cat
  -pig
index:
  rotation:
    version: 1
    hk:
      - picUrl: http://1.jpg
        jumUrl: https://openaccounthkuat.demo.com/activity/#/?AECode=&channel=CH002&plan=HKUS00
      - picUrl: http://2.jpg
        jumUrl: https://openaccounthkuat.demo.com/activity/#/finance?AECode=&channel=CH002&plan=HKUS00
      - picUrl: http://3.jpg
        jumUrl: https://openaccounthkuat.demo.com/iopen/
---
pets: [dog,cat,pig]
name: "zhangsan \n lisi"
---
name: 'zhangsan \n lisi'
```

