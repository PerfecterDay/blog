# 接口与Lambda表达式
{docsify-updated}

## 接口
1. 接口不是类，尤其不能使用new运算符实例化一个接口。
2. 接口中不能包含实例域，但却可以包含常量。
3. 与接口中的方法都自动地被设置为 public 一样，接口中的域将被自动设为 `public static final`。
4. 在Java SE 8中，允许在接口中增加**静态方法**。理论上讲，没有任何理由认为这是不合法的。只是这有违于将接口作为抽象规范的初衷。
5. 可以为接口方法提供一个默认实现。必须用default修饰符标记这样一个方法。

如果在一个接口中将一个方法定义为默认方法，然后某各类实现了该接口的同时又继承了另一个超类，超类中定义了同样的方法，会发生什么情况？
1. 超类优先。如果超类提供了一个具体方法，同名而且有相同参数类型的接口默认方法会被忽略。

如果类同时实现了另一个接口，这个接口和之前的接口也有个同样的方法呢？  
2. 接口冲突。如果一个接口提供了一个默认方法，另一个接口提供了一个同名而且参数类型（不论是否是默认参数）相同的方法，则实现类必须覆盖这个方法来解决冲突。

## Lambda 表达式简介
我们知道类和对象是 Java 中的一等公民，也就是说Java中几乎所有的语言都是要先定义一个类或接口，然后创建对象，然后调用对象的方法。不像C那样可以直接定义一个函数然后直接调用这个函数而无需创建对象。  
Java中的方法参数都是对象类型的，如果我想要传入一个函数作为参数，就比较麻烦，通常是使用一个匿名内部类的方式，语法显得很臃肿。  
Lambda 表达式正是为了解决这一个问题的，它是一种匿名函数(对 Java 而言这并不完全正确，但现在姑且这么认为)，简单地说，它是没有声明的方法，也即没有访问修饰符、返回值声明和名字。

你可以将其想做一种速记，在你需要使用某个方法的地方写上它。当某个方法只使用一次，而且定义很简短，使用这种速记替代之尤其有效，这样，你就不必在类中费力写声明与方法了。

Java 中的 Lambda 表达式形式: 参数，箭头（->）以及一个表达式。如果代码要完成的计算无法放在一个表达式中，就可以像写方法一样，把这些代码放在{}中，并包含显式的return语句。
```
(arg1, arg2...) -> { body }
(type1 arg1, type2 arg2...) -> { body }
```
以下是一些 Lambda 表达式的例子：
```
(int a, int b) -> {  return a + b; } ;
() -> System.out.println("Hello World"); 
(String s) -> { System.out.println(s); } ;
() -> 42 
() -> { return 3.1415 }; 
```
### Lambda 表达式的结构
让我们了解一下 Lambda 表达式的结构。
1. 一个 Lambda 表达式可以有零个或多个参数
2. 参数的类型既可以明确声明，也可以根据上下文来推断，如果可以推导出一个lambda表达式的参数类型，则可以忽略其类型。例如：`(int a)` 与 `(a)` 效果相同
3. 所有参数需包含在圆括号内，参数之间用逗号相隔。例如：`(a, b)` 或 `(int a, int b)` 或 `(String a, int b, float c)`
4. 空圆括号代表参数集为空，即使没有参数也不能省略括号。例如：`() -> 42`
5. 如果方法只有一个参数，而且这个参数的类型可以推导得出，那么圆括号 `()` 甚至可省略。例如：`a -> return a*a`
6. Lambda 表达式的主体可包含零条或多条语句
7. 如果 Lambda 表达式的主体只有一条语句，花括号 `{}` 可省略。匿名函数的返回类型与该主体表达式一致
8. 如果 Lambda 表达式的主体包含一条以上语句，则表达式必须包含在花括号 `{}` 中（形成代码块）。匿名函数的返回类型与代码块的返回类型一致，若没有返回则为空

### 函数式接口
在 Java 中，Marker（标记）类型的接口是一种没有方法或属性声明的接口，简单地说，marker 接口是空接口。相似地，函数式接口是只包含一个抽象方法声明的接口，可以包含多个默认方法、类方法，**但只能有一个抽象方法**。  
`java.lang.Runnable` 就是一种函数式接口，在 `Runnable` 接口中只声明了一个方法 `void run()` ，相似地， `ActionListener` 接口也是一种函数式接口，我们使用匿名内部类来实例化函数式接口的对象，有了 Lambda 表达式，这一方式可以得到简化。

每个 Lambda 表达式都能隐式地赋值给函数式接口，例如，我们可以通过 Lambda 表达式创建 `Runnable` 接口的引用。
```
Runnable r = () -> System.out.println("hello world");
```
当不指明函数式接口时，编译器会自动解释这种转化：
```
new Thread(
   () -> System.out.println("hello world")
).start();
```
因此，在上面的代码中，编译器会自动推断：根据线程类的构造函数签名 `public Thread(Runnable r) { }`，将该 Lambda 表达式赋给 `Runnable` 接口。

以下是一些 Lambda 表达式及其函数式接口：
```
Consumer<Integer>  c = (int x) -> { System.out.println(x) };
BiConsumer<Integer, String> b = (Integer x, String y) -> System.out.println(x + " : " + y);
Predicate<String> p = (String s) -> { s == null };
```
`@FunctionalInterface` 是 Java 8 新加入的一种注解，用于指明该接口类型声明是根据 Java 语言规范定义的函数式接口。Java 8 还声明了一些 Lambda 表达式可以使用的函数式接口，当你注释的接口不是有效的函数式接口时，可以使用 `@FunctionalInterface` 解决编译层面的错误。

以下是一种自定义的函数式接口：
```
@FunctionalInterface
public interface WorkerInterface {
	void doSomeWork();
}
```
根据定义，函数式接口只能有一个抽象方法，如果你尝试添加第二个抽象方法，将抛出编译时错误。例如：
```
@FunctionalInterface
public interface WorkerInterface {
	void doSomeWork();
	void doSomeMoreWork();
}
```
错误：
```
Unexpected @FunctionalInterface annotation 
    @FunctionalInterface ^ WorkerInterface is not a functional interface multiple 
    non-overriding abstract methods found in interface WorkerInterface 1 error
```
函数式接口定义好后，我们可以在 API 中使用它，同时利用 Lambda 表达式。例如：
```
 //定义一个函数式接口
@FunctionalInterface
public interface WorkerInterface {
	void doSomeWork();
}
```
```
public class WorkerInterfaceTest {
    public static void execute(WorkerInterface worker) {
        worker.doSomeWork();
    }

    public static void main(String [] args) {
        //invoke doSomeWork using Annonymous class
        execute(new WorkerInterface() {
            @Override
            public void doSomeWork() {
                System.out.println("Worker invoked using Anonymous class");
            }
        });
        //invoke doSomeWork using Lambda expression 
        execute( () -> System.out.println("Worker invoked using Lambda expression") );
    }
}
```
输出：
```
Worker invoked using Anonymous class 
Worker invoked using Lambda expression
```
这上面的例子里，我们创建了自定义的函数式接口并与 Lambda 表达式一起使用。 `execute()` 方法现在可以将 Lambda 表达式作为参数。

### Lambda 表达式与匿名类的区别
使用匿名类与 Lambda 表达式的一大区别在于 `this` 关键词的使用。**对于匿名类，关键词 this 解读为匿名类对象，而对于 Lambda 表达式，关键词 this 解读为写就 Lambda 的外部类对象**。

```
public class Rabbit extends Animal{
    public void quack(){
        Runnable r1 = ()->{
            System.out.println(this);
        };

        Runnable r2 = new Runnable() {
            @Override
            public void run() {
                System.out.println(this);
            }
        };

        r1.run();
        r2.run();
    }

    public static void main(String[] args) {
        Rabbit r = new Rabbit();
        r.quack();
    }
}
```

上述程序的输出结果为：
```
cn.com.demo.Rabbit@f6f4d33
cn.com.demo.Rabbit$1@23fc625e
```

Lambda 表达式与匿名类的另一不同在于两者的编译方法。Java 编译器编译 Lambda 表达式并将他们转化为类里面的私有函数，它使用 Java 7 中新加的 `invokedynamic` 指令动态绑定该方法，关于 Java 如何将 Lambda 表达式编译为字节码，Tal Weiss 写了一篇很好的文章。

## 方法引用
有时，可能已经有现成的方法可以完成你想要传递到其他代码的某个动作。这时可以直接使用方法引用语法来引用这段代码。  
方法引用要用`::`操作符分隔方法名与对象或类名。主要有以下几种种情况：
1. `object::instanceMethod`
2. `Class::staticMethod`  等价于提供方法参数的lambda表达式。
3. `Class::instanceMethod` 第1个参数会成为方法的目标，例如，`String::compareToIgnoreCase`等同于`（x，y）->x.compareToIgnoreCase（y）`。
4. 构造器引用与方法引用很类似，只不过方法名为new。 如： `Person :: new `

## lambda 的变量作用域
通常，你可能希望能够在lambda表达式中访问外围方法或类中的变量。例如：
```
public static void main(String[] args) {
    repeatMsg("hello",10);
    System.out.println("aaaa");
}

public static void repeatMsg(String text,int delay){
    Runnable  task = () -> {
        System.out.println(text);
    };
    ScheduledExecutorService scheduledExecutorService = Executors.newSingleThreadScheduledExecutor();
    scheduledExecutorService.schedule(task,10, TimeUnit.SECONDS);
}
```

输出：
```
aaaa
hello
```

lambda 表达式的代码会在 `repeatMessage` 调用返回很久(10s)以后才运行，而那时这个参数变量已经不存在了，实际上，使用lambda表达式的重点就是**延迟执行**（deferred execution）。如何保留 `text` 变量呢？要了解到底会发生什么，下面来巩固我们对lambda表达式的理解。lambda表达式有3个部分：
1. 一个代码块；
2. 参数；
3. 自由变量的值，这是指非参数而且不在代码块中定义的变量。

在上边的例子中，这个lambda表达式有1个自由变量 `text` 。表示lambda表达式的数据结构必须存储自由变量的值，在这里就是字符串"Hello"。我们说它被lambda表达式捕获（captured）。（例如，可以把一个lambda表达式转换为包含一个方法的对象，这样自由变量的值就会复制到这个对象的实例变量中。）

lambda表达式可以捕获外围作用域中变量的值。在Java中，要确保所捕获的值是明确定义的，这里有一个重要的限制。在lambda表达式中，**只能引用值不会改变的变量**，lambda表达式中捕获的变量必须实际上是最终变量（effectively final）。实际上的最终变量是指，这个变量初始化之后就不会再为它赋新值。