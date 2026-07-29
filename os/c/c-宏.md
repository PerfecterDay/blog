# 宏
{docsify-updated}

宏定义又称为宏替换、宏代换，简称“宏”，是C提供的三种预处理功能①的其中一种。其主要目的是为程序员在编程时提供一定的方便，并能在一定程度上提高程序的运行效率。

## 定义宏
`#define` 命令是C语言中的一个宏定义命令，它用来讲一个标识符定义为一个字符串，该标识符被称为宏名，被定义的字符串称为替换文本。该命令有两种格式：
+ 一种是简单的宏定义（不带参数的宏定义）
+ 另一种是带参数的宏定义

### 简单宏定义
```c
#define <宏名/标识符> <字符串>
```
比如：
```
#define PI 3.1415926
```

### 带参数的宏定义
```
#define <宏名>(<参数表>) <字符串>
```

比如
```
#define S(a,b) a*b
```

### 条件编译
```
#ifndef _STDARG_H
#define _STDARG_H

#define va_start(v,l) __builtin_va_start(v,l)

#endif
```
只有 `#ifndef` 判断为真时， `#ifndef` 和 `#endif` 之间的代码才会被编译处理。

假设
```
#include <stdio.h>
#include <stdarg.h>
#include "myheader.h"
```

而 myheader.h 又 include 了 stdarg.h : 
```
main.c
 |
 +-- stdio.h
 |
 +-- stdarg.h
 |
 +-- myheader.h
       |
       +-- stdarg.h
```
如果没有 `#ifndef _STDARG_H`，那么在预处理 myheader.h 时，可能导致 `error: redefinition of 'va_list'`