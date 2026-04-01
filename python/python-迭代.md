# python迭代
{docsify-updated}




# 迭代
---------------
如果给定一个list、tuple、dict、set、str或者其它可迭代对象，我们可以通过for循环来遍历这个list或tuple，这种遍历我们称为迭代（Iteration）。

## dict迭代
    d = {'a': 1, 'b': 2, 'c': 3}
    for key in d:
        print(key)
    a
    c
    b
默认情况下，dict迭代的是key。如果要迭代value，可以用for value in d.values()，如果要同时迭代key和value，可以用for k, v in d.items()。

## 字符串迭代
    for ch in 'ABC':
        print(ch)
    A
    B
    C

## 判断是否可迭代
通过collections模块的Iterable类型可以判断是否可迭代。
    
     from collections import Iterable
     isinstance('abc', Iterable) //True

## 索引下标访问
Python内置的enumerate函数可以把一个list变成索引-元素对，这样就可以在for循环中同时迭代索引和元素本身：
    
    for i, value in enumerate(['A', 'B', 'C']):
        print(i, value)

    0 A
    1 B
    2 C