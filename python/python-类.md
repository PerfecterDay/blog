# python类
{docsify-updated}



# 类
----------------
## 定义类
在python中，定义类是通过class关键字实现的：

    class Student(object):
        pass
class后面紧接着是类名，即Student，类名通常是大写开头的单词，紧接着是(object)，表示该类是从哪个类继承下来的，继承的概念我们后面再讲，通常，如果没有合适的继承类，就使用object类，这是所有类最终都会继承的类。

## 定义属性
### 实例属性
定义好了Student类，就可以根据Student类创建出Student的实例，创建实例是通过类名+()实现的：
    
    studentA = Student()
可以自由地给一个实例变量绑定属性，即使在定义类时并没有为该类定义属性。比如，给实例studentA绑定一个name属性：
    
    studentA.name = 'A'
这样只是给该实例绑定了一个属性，Student类的其它实例并没有绑定name属性。由于类可以起到模板的作用，如果要给Student类的所有实例添加属性，可以在定义类，创建实例的时候，把一些我们认为必须绑定的属性强制填写进去。通过定义一个特殊的__init__方法，在创建实例的时候，就把name，score等属性绑上去：

    class Student(object):

        def __init__(self,name,score):
            self.name = name
            self.score = score
注意到__init__方法的第一个参数永远是self，表示创建的实例本身，因此，在__init__方法内部，就可以把各种属性绑定到self，因为self就指向创建的实例本身。

有了__init__方法，在创建实例的时候，就不能传入空的参数了，必须传入与__init__方法匹配的参数，但self不需要传，Python解释器自己会把实例变量传进去：

    studentB = Student('B',90)
和普通的函数相比，在类中定义的函数只有一点不同，就是第一个参数永远是实例变量self，并且，调用时，不用传递该参数。除此之外，类的方法和普通函数没有什么区别，所以，你仍然可以用默认参数、可变参数、关键字参数和命名关键字参数。
### 类属性
但是，如果Student类本身需要绑定一个属性呢？可以直接在class中定义属性，这种属性是类属性，归Student类所有：

    class Student(object):
        name = 'Student'
当我们定义了一个类属性后，这个属性虽然归类所有，但类的所有实例都可以访问到。


## 定义方法
要定义一个方法，除了第一个参数是self外，其他和普通函数一样：
    
    class Student(object):

        def __init__(self, name, score):
            self.name = name
            self.score = score

        def print_score(self):
            print('%s: %s' % (self.name, self.score))

要调用一个方法，只需要在实例变量上直接调用，除了self不用传递，其他参数正常传入：
    
    studentB.print_score()




