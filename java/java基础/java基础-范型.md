# 范型
{docsify-updated}
> https://docs.oracle.com/javase/tutorial/java/generics/index.html  
> https://docs.oracle.com/javase/tutorial/extra/generics/index.html

## 范型类
Java 集合有个缺点：当我们把一个对象放入集合中时，集合就会忘记这个对象的数据类型，当再次取出对象时，对象的编译类型就变成了Object类型，必须进行强制类型转换才能转换成需要的编译类型。 

所谓范型：**就是允许在定义类、接口时指定类型参数(在类名后边加上一对尖括号，类型形参放在尖括号内，多个形参用逗号分隔)，这些类型形参将在声明变量、创建对象时确定（即传入实际的类型参数），类型形参在在整个类内部可用，几乎所有可以使用普通类型的地方都可以使用这种类型形参。**JDK1.5之后改写了所有集合类支持范型。

使用范型类时，可以传入不同的类型实参创建对象，如 `List<Object>`、`List<String>`、`List<Integer>`...，效果上，相当于创建了集合类的一个子类，子类只能存放特定的类型，但是系统并没有为这些声明生成新的 class 文件，更不会把他们当成新类来处理，它们是同样的 List 类。List 类的静态变量和方法会在所有实例之间共享，所以在**静态方法、静态初始化块或者静态成员变量的声明和初始化块中不允许使用类型形参**。自然，这些效果上的子类之间更不会存在继承关系，也就是说 `List<String>`、`List<Integer>` 不是 `List<Object>` 的子类。所以下述代码会出现编译错误：
```
public void test(List<Object> list){...}//严格要求传入 List<Object> 范型
List<String> sList = new ArrayList();
test(sList);
```
当创建了带范型声明的接口、父类之后，可以创建接口的实现类，或者从父类来派生子类，如果**实现类或者派生子类不是范型类（判断是不是范型类，只要看类名后边是否有尖括号包裹的类型形参即可），那么当使用这些接口、父类时，不能再包含类型形参，必须指定具体的类型实参；如果实现类或者派生子类是范型类，则可以为指定与父类相同的类型参数**。如下面的代码是错误的：
```
class Creatrue<T>{}
class Man extends Creatrue<T>{} //编译错误
class Man extends Creatrue<String>{} //可以正确编译
class Man<T> extends Creatrue<T>{} //可以正确编译，此时形参名必须相同为T
class Man<T> extends Creatrue<String>{} //可以正确编译
class Man<U,R> extends Creatrue<String>{} //可以正确编译
```
## 类型变量的限定与范型通配符
声明类型参数时使用 extends 关键字可以限定类型参数的范围：
```
<T extends BindingType>
<T extends Person & Comparable & Serializable>
```
表示T应该是绑定类型的子类型（subtype）。T和绑定类型可以是类，也可以是接口。一个类型变量或通配符可以有多个限定。限定类型用“&”分隔，而逗号用来分隔类型量。
在Java的继承中，可以根据需要拥有多个接口超类型，但限定中至多有一个类。如果用一个类作为限定，它必须是限定列表中的第一个。
`List<? extends Shape>` 表示匹配所有Shape及其子类类型实参的范型。
`List<? super Shape>` 表示匹配所有Shape及其父类类型实参的范型。

将 ? 作为类型实参传递范型类，表示匹配任何类型实参类型。上面的代码改成下面这样可以编译通过：
```
public void test(List<?> list){...}//可以
List<String> sList = new ArrayList();
test(sList);
```

## 范型方法
即使在定义类、接口时没有使用范型，但是在定义方法时想定义类型形参，这也是可以的。
```
修饰符 <T,S> 返回值类型 方法名（形参列表）{
    //方法体
}
```
类型参数 T,S只能在该方法中使用。
```
public class Util {
    public static <K, V> boolean compare(Pair<K, V> p1, Pair<K, V> p2) {
        return p1.getKey().equals(p2.getKey()) &&
               p1.getValue().equals(p2.getValue());
    }
}
Pair<Integer, String> p1 = new Pair<>(1, "apple");
Pair<Integer, String> p2 = new Pair<>(2, "pear");

boolean same = Util.<Integer, String>compare(p1, p2); //显示传递类型参数
boolean same = Util.compare(p1, p2);  //编译器会自动推断类型
```
范型方法允许类型参数被用来表示方法的多个参数之间的类型依赖关系，或者方法返回值和参数之间的类型依赖关系。如果没有这样的依赖关系，不应该使用范型方法。


## 泛型擦除
泛型被引入Java语言，旨在提供更严格的编译时类型检查并支持泛型编程。为实现泛型，Java编译器通过类型擦除机制：

+ 虚拟机中没有泛型，只有普通的类和方法。
+ 将泛型类型中的所有类型参数替换为其边界类型，若类型参数未限定则替换为Object类型。因此生成的字节码仅包含普通类、接口和方法。
+ 在必要时插入类型转换以保持类型安全。
+ 生成桥接方法以在扩展泛型类型中保持多态性。

类型擦除确保参数化类型**不会创建新类**，因此泛型不会产生运行时开销。


## 范型的约束与局限性
1. 不能用基本类型作为类型参数。因此，没有`Pair<double>`，只有`Pair<Double>`。当然， 其原因是类型擦除。擦除之后，Pair 类含有Object 类型的域，而Object不能存储double值。
2. 不能在静态域或方法中引用类型变量。禁止使用带有类型变量的静态域和方法。
   ```
   public class Singleton<T> {
	private static T singleInstance; // EROR 
	public static T getSingleInstance{ // EROR
		if (singleInstance = nul) construct new instance of T 
		return singleInstance;
	} 
   ```
3. 不能抛出或捕获泛型类的实例，既不能抛出也不能捕获泛型类对象。实际上，甚至泛型类扩展 `Throwable` 都是不合法的 。

## 泛型与继承
假设我们定义了一个泛型类`Pair`，`Manager`是`Employee`的子类，那么它们的继承关系如下:
<center><img src="pics/generics-1.png" alt=""></center>

**总结一下就是无论S与T有什么关系，通常，`Pair<S>`与`Pair<T>`都没有任何关系。**

还需要说明的是，泛型类可以扩展或实现其他的泛型类。就这一点而言，它们与普通的类没有什么区别。  
ArrayList类实现了List接口。这意味着，一个`ArrayList<Manager>`可以转换为一个`List<Manager>`。但是，如前面所见，`ArrayList<Manager>`不是一个`ArrayList<Employee>`或`List<Employee>`。

<center><img src="pics/generics-2.png" alt=""></center>


## 桥方法
```
public class UserCenterApp<T> {
    public static void main(String[] args) {
        A a = new A();
        Method[] declaredMethods = A.class.getDeclaredMethods();
        for (int i = 0; i < declaredMethods.length; i++) {
            System.out.println(declaredMethods[i].toGenericString()+":"+declaredMethods[i].isBridge());
        }
    }

    public void draw(T  shape) {
        System.out.println("Drawing shape: " + shape);
    }
}

class A extends UserCenterApp<String>{
    @Override
    public void draw(String shape) {
        System.out.println("Drawing A: " + shape);
    }
}
```

输出为：
```
public void com.demo.demo.A.draw(java.lang.String):false
public void com.demo.demo.A.draw(java.lang.Object):true
```

如果改一下代码：
```
class A extends UserCenterApp<Object>{
    @Override
    public void draw(Object shape) {
        System.out.println("Drawing A: " + shape);
    }
}
```
输出为：
```
public void com.demo.demo.A.draw(java.lang.Object):false
```

这时候，不需要生成桥方法了。