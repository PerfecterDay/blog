# makefile 和 Cmake
{docsify-updated}
> https://www.ruanyifeng.com/blog/2015/02/make.html

## MakeFile
Makefile文件由一系列规则（rules）构成。每条规则的形式如下。

```
<target> : <prerequisites> 
[tab]  <commands>
```
+ 第一行冒号前面的部分，叫做**目标**（target），冒号后面的部分叫做 **前置条件**（prerequisites）
+ 第二行必须由一个 `tab` 键起首，后面跟着 **命令** （commands）。
  
**目标**是必需的，不可省略；**前置条件** 和**命令**都是可选的，但是两者之中必须至少存在一个。  
每条规则就明确两件事：**构建目标的前置条件是什么，以及如何构建**。下面就详细讲解，每条规则的这三个组成部分。


### 目标（target）
一个目标（target）就构成一条规则。目标通常是文件名，指明Make命令所要构建的对象，比如 mbr.bin 。目标可以是一个文件名，也可以是多个文件名，之间用空格分隔。  
除了文件名，目标还可以是某个操作的名字，这称为"**伪目标**"（phony target）。如：
```
clean:
      rm *.o
```
上面代码的目标是clean，它不是文件名，而是一个操作的名字，属于"伪目标"，作用是删除对象文件。可以执行 `make clean` 类执行目标构建。  
但是，如果当前目录中，正好有一个文件叫做clean，那么这个命令不会执行。因为Make发现clean文件已经存在，就认为没有必要重新构建了，就不会执行指定的rm命令。  
为了避免这种情况，可以明确声明clean是"伪目标"，写法如下:
```
.PHONY: clean
clean:
        rm *.o temp
```

声明clean是"伪目标"之后，make就不会去检查是否存在一个叫做clean的文件，而是每次运行都执行对应的命令。  
**如果Make命令运行时没有指定目标，默认会执行Makefile文件的第一个目标。**

### 前置条件（prerequisites）
前置条件通常是一组文件名，之间用空格分隔。它指定了"目标"是否重新构建的判断标准：**只要有一个前置文件不存在，或者有过更新（前置文件的last-modification时间戳比目标的时间戳新），"目标"就需要重新构建。**
如果前置条件有多个，那么这多个条件的构建顺序与makefile中书写的顺序是一致的，除非有另外的依赖定义。
默认情况下，执行顺序与先决条件列表中指定的相同，除非这些先决条件之间定义有任何依赖关系。
`abc: x y z` -> 顺序是x y z。

```
abc: x y z
y : z
```
顺序应该是x z y。

但理想情况下，你应该设计你的Makefiles，使它不依赖于先决条件的指定顺序。也就是说，如果y应该在z之后执行，就必须有一个y : z的依赖关系。

### 命令
命令（commands）表示如何更新目标文件，由一行或多行的Shell命令组成。它是构建"目标"的具体指令，它的运行结果通常就是生成目标文件。  
每行命令之前必须有一个tab键。如果想用其他键，可以用内置变量`.RECIPEPREFIX`声明。
需要注意的是，每行命令在一个单独的shell中执行。这些Shell之间没有继承关系。
```
var-lost:
    export foo=bar
    echo "foo=[$$foo]"
```
上面代码执行后（make var-lost），取不到foo的值。因为两行命令在两个不同的进程执行。一个解决办法是将两行命令写在一行，中间用分号分隔。
```
var-kept:
    export foo=bar; echo "foo=[$$foo]"
```

### 语法
+ `\#` 在Makefile中表示注释。
+ 在命令的前面加上 `@` ，就可以关闭回声。由于在构建过程中，需要了解当前在执行哪条命令，所以通常只在注释和纯显示的 `echo` 命令前面加上`@`。
+ `%` 通配符匹配任何非空子串，`%.o: %.c`
+ 变量定义与引用，变量需要用 `$(var)` 引用
  ```
  txt = Hello World
  test:
    @echo $(txt)
  ```
+ 内置变量
  1. `$(CC)` 指向当前使用的编译器
  2. `$(MAKE)`指向当前使用的Make工具

+ 自动变量
  1. `$@` 指代当前**目标**，就是Make命令当前构建的那个目标。
  2. `$<` 指代**第一个前置条件**。假设规则为 `t: p1 p2`，那么 $< 就指代p1
  3. `$?` 指代比**目标更新的所有前置条件**，之间以空格分隔。比如，规则为 t: p1 p2，其中 p2 的时间戳比 t 新，$?就指代p2。
  4. `$^` 指代所有**前置条件**，之间以空格分隔。比如，规则为 t: p1 p2，那么 $^ 就指代 p1 p2 
  5. `$*` 指代匹配符 `%` 匹配的部分， 比如% 匹配 f1.txt 中的f1 ，$* 就表示 f1。
  6. `$(@D) 和 $(@F)` 分别指向 `$@` 的**目录名和文件名**。比如，`$@` 是 src/input.c，那么 `$(@D)` 的值为 `src` ，`$(@F)` 的值为 `input.c`。
  7. `$(<D) 和 $(<F)` 分别指向 `$<` 的目录名和文件名。

### 函数
函数的语法：
```
$(function-name arguments...)
```
+ 函数名 和 参数 之间用 空格分隔；
+ 逗号（,) 用来分隔多个参数；
+ 可以嵌套使用（常见于变量替换和字符串处理）；
+ 所有函数都是字符串处理函数（Makefile 的本质是文本处理）。


常用 Makefile 函数一览：
| 函数名       | 用途                         | 示例说明                                                            |
|--------------|------------------------------|---------------------------------------------------------------------|
| `wildcard`   | 匹配符合模式的文件           | `$(wildcard *.go)` → `main.go test.go`                             |
| `patsubst`   | 模式替换                     | `$(patsubst %.go, %.so, main.go)` → `main.so`                      |
| `subst`      | 字符串替换（精确匹配）       | `$(subst go,so,main.go)` → `main.so`                               |
| `addprefix`  | 为每个元素加前缀             | `$(addprefix ./src/,a b.c)` → `./src/a ./src/b.c`                 |
| `addsuffix`  | 为每个元素加后缀             | `$(addsuffix .o,main test)` → `main.o test.o`                     |
| `notdir`     | 去掉路径只保留文件名         | `$(notdir src/main.go)` → `main.go`                               |
| `dir`        | 获取路径部分                 | `$(dir src/main.go)` → `src/`                                     |
| `foreach`    | 遍历一个列表                 | `$(foreach var,list,text)`                                        |
| `if`         | 条件判断                     | `$(if condition,then-part,else-part)`                             |
| `filter`     | 过滤符合模式的元素           | `$(filter %.go,main.go readme.md)` → `main.go`                    |
| `filter-out` | 删除符合模式的元素           | `$(filter-out %.md,main.go readme.md)` → `main.go`                |
| `shell`      | 调用 shell 命令              | `$(shell pwd)` → `/your/current/dir`                              |

### 获取命令行参数
`make image-sit version=1.0.1 --debug`  
然后在 Makefile 中使用 `$(version)` 引用指定的变量。

### make 执行流程
默认情况下，make 会从第一个目标开始（不包括名称以". "开头的目标，除非它们也包含一个或多个"/"）。这就是默认目标。（目标是 make 最终要更新的目标。您可以使用命令行（参见指定目标的参数）或 .DEFAULT_GOAL 特殊变量（参见其他特殊变量）来覆盖这一行为。

## Cmake
<center><img src="/pics/cmake.jpg" alt=""></center>