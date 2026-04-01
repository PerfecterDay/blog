# python列表生成器
{docsify-updated}



# python列表生成器
-----------------------
如果要生成[1x1, 2x2, 3x3, ..., 10x10]，仅用一行代码即可：
    
    [x * x for x in range(1, 11)]

写列表生成式时，把要生成的元素x \* x放到前面，后面跟for循环，就可以把list创建出来，十分有用。

for循环后面还可以加上if判断，这样我们就可以筛选出仅偶数的平方：

    [x * x for x in range(1, 11) if x%2==0]

for循环其实可以同时使用两个甚至多个变量，比如dict的items()可以同时迭代key和value：

     d = {'x': 'A', 'y': 'B', 'z': 'C' }
     [k + '=' + v for k, v in d.items()]//['y=B', 'x=A', 'z=C']