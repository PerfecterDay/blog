# 反射
{docsify-updated}

JAVA反射机制是在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意一个方法和属性；这种动态获取类的信息以及动态调用对象的方法的功能称为java语言的反射机制。

要想解剖一个类,必须先要获取到该类的字节码文件对象，进而使用的Class类中的方法解剖类。所以先要获取到一个字节码文件对应的Class类型的对象。

## Class对象
反射把一个java类字节码加载到内存并构造成一个Class类型的对象，并将类中的属性、方法等各种成分映射成相应的Java对象。

一个类有：成员变量、方法、构造方法、包等等信息，利用反射技术可以对一个类进行解剖，把各个组成部分映射成一个个对象。（其实：一个类中这些成员方法、构造方法等在Java中都有一个专门的类来描述）。如图是类的正常加载过程：反射的原理在于class对象。
熟悉一下加载：Class对象的由来是将class字节码文件读入内存，并为之创建一个Class对象。

<center><img src="pics/ClassObject.png" alt="" width=70%></center>

**上图中说的每个类对应的Class对象绝对只有一个在被同一个加载器加载的时候才是正确的。**

## java.lang.Class类
Class类是对java字节码文件的抽象，每个字节码文件加载到jvm后，jvm都会为其生成一个Class类的实例来描述该字节码文件，且每个类或接口的字节码文件只会生成一个Class类的实例。Class类的定义如下：
```
public final class Class<T> implements java.io.Serializable,
                            GenericDeclaration,
                            Type,
                            AnnotatedElement{...}
```
Class没有公共构造方法。Class对象是在加载类时由Java虚拟机以及通过调用类加载器中的defineClass方法自动构造的。也就是说不需要我们自己去处理创建，JVM已经帮我们创建好了。

## 获取 Class 对象的三种方式
+ 调用某个对象的 `getClass()` 方法会返回该对象所属类的Class对象实例， `getClass()` 是Object类的方法。
+ 任何数据类型（即类，包括基本数据类型）都有一个“静态”的 `class` 属性。
+ Class类的静态方法：`forName(String className)`。

## Class 类的方法

### 获取类的方法

#### 构造方法
1. 获取公有访问级别构造方法的方法
   + `getgetConstructors()`: 可以获取到该类的所有公有构造方法构造方法组成的数组：`Constructor[]` 。
   + `getConstructor(Class<?>... parameterTypes)`: 可以获取到参数类型为 `parameterTypes` 指定的公有构造方法的 `Constructor` 实例对象。

2. 获取所有访问级别的构造方法
   + `getDeclaredConstructors()`: 可以获取到所有的构造方法(包括：私有、受保护、默认、公有)组成的数组：`Constructor[]`。
   + `getDeclaredConstructor(Class<?>... parameterTypes)`: 可以获取到参数类型为 `parameterTypes` 指定的所有构造方法(包括：私有、受保护、默认、公有)的 `Constructor` 实例对象。注意 `parameterTypes` 是可变参数，可以不传，当不传时获取的时默认构造方法。

3. 调用构造方法
通过上述方法获取 `Constructor`对象后，可以 调用 `Constructor` 对象的 `newInstance(Object... initargs)` 方法来创建类的新实例，并用指定的初始化参数（可以不传）初始化该实例。

#### 普通方法

+ `public Method[] getMethods() throws SecurityException` : `getMethods` 方法返回一个数组，其中包含类和超类的所有**公共方法**。
+ `public Method[] getDeclaredMethods() throws SecurityException` : 返回声明的所有方法，包括公共、保护、私有的方法。

获取到 Method 对象后，可以调用它的 `public Object invoke(Object obj, Object... args)` 对方法进行调用。

### 获取成员变量并使用
1. 获取公有访问级别的属性
   + `getFields()`: 方法获取类的所有公有访问级别的属性，返回一个属性数组:`Field[]`。
   + `getField(String name)`: 方法获取名字为name的公有属性，返回属性对象：`Field`。

2. 获取所有访问级别的属性
   + `getDeclaredFields()`: 方法获取所有的属性(包括：私有、受保护、默认、公有)，返回一个属性数组: `Field[]`。
   + `getDeclaredField(String name)`: 方法获取名字为name的所有的属性(包括：私有、受保护、默认、公有)，返回属性对象：`Field`。

#### 属性赋值
通过上述方法获取 `Field`对象后，可以调用 `Field` 类的 `set(Object object,Object value)` 方法，可以为object对象的对应属性设置为value值。不过在调用之前必须先调用`void setAccessible(true)` 设置允许访问。

## 性能
https://docs.oracle.com/javase/tutorial/reflect/index.html

```
Because reflection involves types that are dynamically resolved, certain Java virtual machine optimizations can not be performed. 
Consequently, reflective operations have slower performance than their non-reflective counterparts, 
and should be avoided in sections of code which are called frequently in performance-sensitive applications.
```