# python函数
{docsify-updated}



# 函数
------------
## 函数定义
在Python中，定义一个函数要使用def语句，依次写出函数名、括号、括号中的参数和冒号:，然后，在缩进块中编写函数体，函数的返回值用return语句返回。

    def myFunction(arg1,arg2):
        a = arg1+arg2
        a++
        return a

如果没有return语句，函数执行完毕后也会返回结果，只是结果为None。return None可以简写为return。

## 空函数
如果想定义一个什么事也不做的空函数，可以用pass语句：
    
    def nop():
        pass

pass语句什么都不做，那有什么用？实际上pass可以用来作为占位符，比如现在还没想好怎么写函数的代码，就可以先放一个pass，让代码能运行起来。

    if age>18:
        pass
缺少pass，代码会报错。

## 参数检查
调用函数时，如果参数个数不对，Python解释器会自动检查出来，并抛出TypeError：

    myFunction(1)
    Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    TypeError: TypeError: myFunc() missing 1 required positional argument: 'arg2'
但是如果参数类型不对，Python解释器就无法帮我们检查。必须手动检查参数类型是否正确。数据类型检查可以用内置函数isinstance()实现：

    def myFunction(arg1,arg2):
        if not isinstance(arg1, (int, float)):
            raise TypeError('bad operand type')
        if not isinstance(arg2, (int, float)):
            raise TypeError('bad operand type')
        a = arg1+arg2
        a++
        return a

## 返回元组（多个值）

    import math
    def move(x, y, step, angle=0):
        nx = x + step * math.cos(angle)
        ny = y - step * math.sin(angle)
        return nx, ny

    x,y=move(100,100,60,math.pi/6)
    print(x,y) //151.96152422706632 70.0
    r = move(100, 100, 60, math.pi / 6)
    print(r) //(151.96152422706632, 70.0)元组
原来返回值是一个tuple！但是，在语法上，返回一个tuple可以省略括号，而多个变量可以同时接收一个tuple，按位置赋给对应的值，所以，Python的函数返回多值其实就是返回一个tuple，但写起来更方便。

## 函数参数
Python的函数定义非常简单，但灵活度却非常大。除了正常定义的必选参数外，还可以使用默认参数、可变参数和关键字参数，使得函数定义出来的接口，不但能处理复杂的参数，还可以简化调用者的代码。

### 位置参数
位置参数就是参数的位置与函数调用时对应位置上的参数一一对应。

### 默认参数
定义函数时，为参数赋上一个默认值，此参数即成为默认参数。当调用函数时，不传此位置的参数时，即使用默认参数而不是报错。但是默认参数只能放在必选必选参数后面，可以有多个默认参数，但是这些默认参数必须都在必选参数的后面。

### 可变参数
在Python函数中，还可以定义可变参数。顾名思义，可变参数就是传入的参数个数是可变的，可以是1个、2个到任意个，还可以是0个。

仅仅在参数前面加了一个*号。在函数内部，参数接收到的是一个tuple，因此，函数代码完全不变。但是，调用该函数时，可以传入任意个参数，包括0个参数：

    def calc(*numbers):
    sum = 0
    for n in numbers:
        sum = sum + n * n
    return sum

如果已经有一个list或者tuple，要调用一个可变参数怎么办？可以这样做：

     nums = [1, 2, 3]
     calc(nums[0], nums[1], nums[2])
这种写法当然是可行的，问题是太繁琐，所以Python允许你在list或tuple前面加一个*号，把list或tuple的元素变成可变参数传进去：

    nums = [1, 2, 3]
    calc(*nums)
*nums表示把nums这个list的所有元素作为可变参数传进去。这种写法相当有用，而且很常见。

## 关键字参数
可变参数允许你传入0个或任意个参数，这些可变参数在函数调用时自动组装为一个tuple。而关键字参数允许你传入0个或任意个含参数名的参数，这些关键字参数在函数内部自动组装为一个dict。请看示例：

    def person(name, age, **kw):
        print('name:', name, 'age:', age, 'other:', kw)
可以这样调用

    person('Bob', 35, city='Beijing') //name: Bob age: 35 other: {'city': 'Beijing'}
也可以这样调用

    person('Adam', 45, gender='M', job='Engineer') //name: Adam age: 45 other: {'gender': 'M', 'job': 'Engineer'}

关键字参数有什么用？它可以扩展函数的功能。比如，在person函数里，我们保证能接收到name和age这两个参数，但是，如果调用者愿意提供更多的参数，我们也能收到。试想你正在做一个用户注册的功能，除了用户名和年龄是必填项外，其他都是可选项，利用关键字参数来定义这个函数就能满足注册的需求。

和可变参数类似，也可以先组装出一个dict，然后，把该dict转换为关键字参数传进去:

    extra = {'city': 'Beijing', 'job': 'Engineer'}
    person('Jack', 24, city=extra['city'], job=extra['job'])//name: Jack age: 24 other: {'city': 'Beijing', 'job': 'Engineer'}
也可以使用简化的写法：
    
    person('Jack', 24, **extra)

\*\*extra表示把extra这个dict的所有key-value用关键字参数传入到函数的\*\*kw参数，kw将获得一个dict，注意kw获得的dict是extra的一份拷贝，对kw的改动不会影响到函数外的extra

## 命名关键字参数
关键字参数对调用时传入的参数的名字不作限制，可以传入任意的不受限制的关键字参数。如果要限制关键字参数的名字，就可以用命名关键字参数，例如，只接收city和job作为关键字参数。这种方式定义的函数如下：

    def person(name, age, *, city, job):
        print(name, age, city, job)
和关键字参数\*\*kw不同，命名关键字参数需要一个特殊分隔符\*，\*后面的参数被视为命名关键字参数。使用如下方方式调用:

    person('Jack', 24, city='Beijing', job='Engineer')
如果函数定义中已经有了一个可变参数，后面跟着的命名关键字参数就不再需要一个特殊分隔符*了：

    def person(name, age, *args, city, job):
        print(name, age, args, city, job)
使用命名关键字参数时，要特别注意，如果没有可变参数，就必须加一个\*作为特殊分隔符。如果缺少\*，Python解释器将无法识别位置参数和命名关键字参数。

## 参数组合
参数组合
在Python中定义函数，可以用必选参数、默认参数、可变参数、关键字参数和命名关键字参数，这5种参数都可以组合使用。但是请注意，参数定义的顺序必须是：必选参数、默认参数、可变参数、命名关键字参数和关键字参数。