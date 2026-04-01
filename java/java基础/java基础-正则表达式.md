# 正则表达式
{docsify-updated}
> https://regex101.com/  
> https://regex-vis.com/

- [正则表达式](#正则表达式)
	- [基础语法](#基础语法)
	- [群组](#群组)


### 基础语法
+ 大部分字符都可以与它们 自身匹配，例如在前面示例中的 ava 字符。
+ `.`符号可以匹配任何字符（有可能不包括行终止符 ，这取决于标志的设置）。
+ 使用 `\`作为转义字符，例如，`\．`匹配句号而`\\`匹配反斜线 。
+ `^`和`$`分别匹配一行的开头和结尾。
+ 如果 X 和 Y 是正则表达式，那么 `XY` 表示“任何 X 的匹配后面跟随 Y 的匹配”， `X | Y`表示“任何 X 或 Y 的匹配” 。
+ 你可以将量词运用到表达式 X : `X+`( 1 个或多个）、 `X*`( 0 个或多个）与 `X?` ( 0 个或 1 个）。
+ 默认情况下，量词要匹配能够使整个匹配成功的最大可能的重复次数。 你可以修改这种行为，方法是使用后缀`?`（使用勉强或吝啬匹配 ，也就是匹配最小的重复次数）或使用后缀`＋`（使用占有或贪婪匹配， 也就是即使让整个匹配失败，也要匹配最大的重复次数）
	例如，字符串 cab 匹配`[a-z]*ab `，但是不匹配`[a-z]*+ab`。 在第一种情况中，表达式`[a-z]＊`只匹配字符 c ，使得字符 ab 匹配该模式的剩余部分；但是贪婪版本`[a-z]*+`将匹配字符 cab ，模式的剩余部分(ab)将无法匹配字符串中的内容。
+ 我们使用群组来定义子表达式，其中群组用括号`()`括起来。 例如，`([+-]?)([0-9])+` 。然后你可以询问模式匹配器，让其返回每个组的匹配，或者用`\n` 来引用某个群组，其中 n 是群组号（从\1开始） 。


正则表达式的最简单用法就是测试某个特定的字符串是否与它匹配。 首先用表示正则表达式的字符串构建一个 `Pattern` 对象。 然后从这个模式中获得一个 `Matcher` ，并调用它的 `matches` 方法：
```
Pattern pattern = Pattern. compile(patternString); 
Matcher matcher ＝ pattern.matcher(input);
if (matcher. matches()) ...
```
在编译这个模式时，你可以设置一个或多个标志，例如：
```
Pattern pattern = Pattern. compile(expression,Pattern.CASE_INSENSITIVE 
+ Pattern.UNICODE_CASE);
```
或者可以在模式中指定它们：
```
String regx = "?iU:expression";
```

+ `Pattern.CASE_INSENSITIVE` 或 `r` ：匹配字符时忽略字母的大小写，默认情况下，这个标志只考虑 US ASCii 字符。
+ `Pattern.UNICODE_CASE` 或 `u` ：当与 `CASE_INSENSITIVE` 组合使用时，用 Unicode 字母的大小写来匹配。
+ `Pattern.UNICODE_CHARACTER_CLASS` 或 `U` ：选择 Unicode 字符类代替 POSIX ，其中蕴含了 `UNICODE_CASE`。

### 群组
如果正则表达式包含群组，那么 `Matcher` 对象可以揭示群组的边界。 下面的方法:
```
int start(int groupIndex)
int end(int groupIndex)
```
将产生指定群组的开始索引和结束之后的索引。可以直接通过调用下面的方法抽取匹配的字符串：
```
String group(int groupIndex);
```
群组 0 是整个输入，而用于第一个实际群组的群组索引是 1 。 调用 `groupCount` 方法可以获得全部群组的数量。 对于具名的组，使用下面的方法:
```
int start(String groupName)
int end(String groupName)
String group(String groupName)
```

<center><img src="pics/regex-group.png" width="50%"></center>
