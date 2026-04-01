# awk
{docsify-updated}

> https://www.bookstack.cn/books/junmajinlong-awk.   
> https://mp.weixin.qq.com/s/sxL6m0DsLtGrKt1Jg_f9Og

## 基本流程
AWK的工作流程大概是这样：
1. 读取一行数据
2. 检查是否匹配 `pattern`
3. 如果匹配，执行对应的 `action`
4. 继续下一行，直到文件结束

AWK的基本语法结构是这样的：
```
awk 'pattern { action }' filename
```
其中 `pattern` 是匹配模式， `action` 是要执行的动作。如果省略 `pattern` ，就对所有行执行 `action` ；如果省略 `action` ，就打印匹配的行。

## 内置变量
+ `NR（Number of Records）`：当前处理的行号 ---->  `awk '{print NR, $0}' users.txt`
+ `NF（Number of Fields）`：当前行的字段数量 -----> `awk '{print NF, $0}' users.txt`
+ `FS（Field Separator）`：字段分隔符，默认是空格 ------> `awk 'BEGIN{FS=","} {print $1}' data.csv`
+ `RS（Record Separator）`：记录分隔符，默认是换行符 ------> ``
+ `OFS（Output Field Separator）`：输出字段分隔符 -------> `awk 'BEGIN{OFS="|"} {print $1, $2}' users.txt`

## 模式匹配
1. 正则表达式匹配：`awk '/工程师/ {print $1}' users.txt`
2. 条件匹配：`awk '$2 > 27 {print $1, $2}' users.txt`
3. 范围匹配：`awk '/start/,/end/ {print}' logfile`
4. `BEGIN` 和 `END` ：`awk 'BEGIN{print "姓名 年龄"} {print $1, $2} END{print "总共", NR, "条记录"}' users.txt`


## 函数
1. `length()` ：返回字符串长度 ---> `awk '{if(length($1) > 5) print $1}' users.txt`
2. `substr()`：截取子字符串 ----> `awk '{print substr($1, 1, 2)}' users.txt  # 打印姓名的前两个字符`
3. `gsub()`：全局替换 ----> `awk '{gsub(/工程师/, "Engineer"); print}' users.txt`
4. `split()` ：分割字符串 -----> `awk '{split($4, arr, "/"); print arr[1]}' logfile`