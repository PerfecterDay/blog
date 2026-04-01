# python list
{docsify-updated}



# list

Python内置的一种数据类型是列表：list。list是一种有序的集合，可以随时添加和删除其中的元素。

## list的定义
直接用[]定义一个list：

    classmates = ['Michael', 'Bob', 'Tracy']

## list元素个数
用len()函数可以获得list元素的个数：

    len(classmates) //3

## list中元素的访问
直接用list变量名+\[位置\]访问对应位置的元素,正数从0开始递增编码，倒数从-1开始递减编码

    0,1,2
    -3,-2,-1

    classmates[0] //'Michael'
    classmates[1] //'Bob'
    classmates[2] //'Tracy'
    classmates[3] //'IndexError'
    classmates[-1] //'Tracy'
    classmates[-2] //'Bob'
    classmates[-3] //'Michael'
    classmates[-4] //IndexError

当索引超出了范围时，Python会报一个IndexError错误，所以，要确保索引不要越界，记得最后一个元素的索引是len(classmates) - 1。

## list添加元素
list是一个可变的有序表，所以，可以往list中追加元素。

可以使用append()方法往list末尾追加元素：

    classmates.append('Adam') //['Michael', 'Bob', 'Tracy','Adam']

也可以用insert()方法把元素插入到指定的位置，比如索引号为1的位置

    classmates.insert(1,'Jack') //['Michael', 'Jack', 'Bob', 'Tracy', 'Adam']

## list删除元素
使用pop()方法可以删除list末尾的元素：
    
    classmates.pop() //['Michael', 'Jack', 'Bob', 'Tracy']

要删除指定位置的元素，使用pop(i)方法，其中i是索引位置：
    
    classmates.pop(1) //['Michael', 'Bob', 'Tracy']

## list元素替换
要把某个元素替换成别的元素，可以直接赋值给对应的索引位置：

    classmates[1] = 'Sarah' //['Michael', 'Sarah', 'Tracy']

## 多维list

    p = ['asp', 'php']
    s = ['python', 'java', p, 'scheme']
    s[2][0] //'asp'
    s[2][1] //'php'




