# python_slots
{docsify-updated}


# __slots__
为了达到限制的目的，Python允许在定义class的时候，定义一个特殊的__slots__变量，来限制该class实例能添加的属性：
    
    class Student(object):
         __slots__ = ('name', 'age') # 用tuple定义允许绑定的属性名称
这样的话Student类只能绑定name和age属性，不管是在__init__()方法内绑定还是为实例绑定动态属性，都只能是name或age，否则会报错。