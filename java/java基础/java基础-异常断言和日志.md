# 异常、断言和日志
{docsify-updated}

## 异常
Java 的异常结构：

<img style="display: inline;" src="pics/exception.png" alt="" width=500px>

```
              ---> Throwable <--- 
              |                  |
              |                  |
              |                  |
      ---> Exception           Error
      |                     (unchecked)
      |
RuntimeException
  (unchecked)
```

Java语言规范:
+ 将派生于 `Error` 类或 `RuntimeException` 类的所有异常称为**非受查（unchecked）异常**。
+ 所有其他的异常称为**受查（checked）异常**。 
 
**编译器将核查是否为所有的受查异常提供了异常处理器。**

派生于 `RuntimeException` 的异通常是由程序错误导致的，包含下面几种常见情况：
+ 错误的类型转换。 -`ClassCastException`
+ 数组访问越界。- `IndexOutOfBoundsException`
+ 访问null指针 - `NullPointerException`
+ `IllegalArgumentException`
+ `SecurityException`

`Errors`: 代表严重且通常无法恢复的情况，如库不兼容、无限递归或内存泄漏。
+ `StackOverflowError`
+ `OutOfMemoryError`
+ `UnsupportedClassVersionError`: calss 文件不兼容

而程序本身没有问题，但由于像I/O错误这类问题导致的异常不是派生于 `RuntimeException` ，常见的有：
+ 试图在文件尾部后面读取数据。
+ 试图打开一个不存在的文件。 `FileNotFoundException`
+ 运行时“主动加载”类失败，如 `Class.forname("xxx)`;试图根据给定的字符串查找Class对象，而这个字符串表示的类并不存在。`ClassNotFoundException`
+ `NoClassDefFoundError` : 编译时存在、运行时找不到 class 文件

### 声明受查异常
如果遇到了无法处理的情况，那么Java的方法可以抛出一个异常。这个道理很简单：一个方法不仅需要告诉编译器将要返回什么值，还要告诉编译器有可能发生什么错误。 
 
一个方法必须声明所有可能抛出的受查异常，而非受查异常要么不可控制（Error），要么就应该避免发生（RuntimeException）。如果方法没有声明所有可能发生的受查异常，编译器就会发出一个错误消息。也就是说如果方法中本身含有 `throw` 异常语句，编译器严格地执行 `throws` 说明符。如果调用了一个抛出受查异常的方法，就必须对它进行处理，或者继续传递（throw）。

如果编写一个覆盖超类的方法，而这个方法又没有抛出异常（如JComponent中的paintComponent），那么这个方法就必须捕获方法代码中出现的每一个受查异常。**不允许在子类的throws说明符中出现超过超类方法所列出的异常类范围。**

### finally 语句

<center><img src="pics/finally.png" alt="" width=30%></center>

在上面这段代码中，有下列3种情况会执行finally子句：
1. 代码没有抛出异常。在这种情况下，程序首先执行try语句块中的全部代码，然后执行finally子句中的代码。随后，继续执行try语句块之后的第一条语句。也就是说，执行标注的1、2、5、6处。
2. 抛出一个在catch子句中捕获的异常。在上面的示例中就是IOException异常。在这种情况下，程序将执行try语句块中的部分代码，执行到发生异常的代码处为止。此时，将跳过try语句块中的剩余代码，转去执行与该异常匹配的catch子句中的代码，最后执行finally子句中的代码。

    + 如果catch子句没有抛出异常，程序将执行try语句块之后的第一条语句。在这里，执行标注1、3、4、5、6处的语句。
    + 如果catch子句抛出了一个异常，异常将被抛回这个方法的调用者。在这里，执行标注1、3、5处的语句。

3. 代码抛出了一个异常，但这个异常不是由catch子句捕获的。在这种情况下，程序将执行try语句块中的部分代码，执行到发生异常的代码处为止。此时，将跳过try语句块中的剩余代码，然后执行finally子句中的语句，并将异常抛给这个方法的调用者。在这里，执行标注1、5处的语句。因为发生了未捕获的异常，所以 try 语句块后边的代码也跳过执行了。

try-finally 总是配对使用，也就是说一个 try 只能跟一个 finally 语句，不能同时使用多个 finally 语句。


#### finally 与 return
1. try-catch 中有 return 语句时（finally 没有return 语句），函数返回值会根据执行路径返回 return 语句的返回值（可以想象成返回了一个 final 类型的对象/值）。但是 finally 中的语句还是会执行，此时如果返回的是基本类型，那么即使 finally 中操作了返回值，也不会对函数的返回值产生影响。但是，如果返回的是对象，finally 中的语句虽然不能返回另一个对象，但是可以改变返回对象的状态。
2. finally中有return时，会直接在finally中退出，导致try、catch中的return失效。

```
public static void main(String argv[]) {
    System.out.println(test1().x);  //输出1
    System.out.println(test2().x);  //输出 2
    System.out.println(test3()); //输出1
    System.out.println(test4()); //输出2
}

static A test1() {
    A a = new A();
    try {
        a.x = 1;
        return a;
    } finally {
        a = new A(2);
    }
}

static A test2() {
    A a = new A();
    try {
        a.x = 1;
        return a;
    } finally {
        a.x = 2;
    }
}

static int test3() {
    int a = 0;
    try {
        a = 1;
        return a;
    } finally {
        a = 2;
    }
}

static int test4() {
    try {
        return 1;
    } finally {
        return 2;
    }
}
```



### 带资源的 try 语句
假设资源属于一个实现了AutoCloseable接口的类,则带资源的try语句（try-with-resources）的最简形式为：
```
try(Resource res1 = ....; Resource res2 =...){
    // work with res
}
结束时，会自动依次调用 res2.close()、res1.close() 关闭资源
```
这个块正常退出时，或者存在一个异常时，都会调用 `res.close()`方法，就好像使用了finally块一样。  
还可以指定多个资源。当指定多个资源时，会按照声明顺序的逆序依次调用各个资源的 `close()` 方法以释放资源。

## 断言
断言机制允许在测试期间向代码中插入一些检查语句。当代码发布时，这些插入的检测语句将会被自动地移走。Java语言引入了关键字assert。这个关键字有两种形式：
1. `assert 条件;`
2. `assert 条件:表达式;`

这两种形式都会对条件进行检测，如果结果为false，则抛出一个AssertionError异常。在第二种形式中，表达式将被传入AssertionError的构造器，并转换成一个消息字符串。

在默认情况下，断言被禁用。可以在运行程序时用`-enableassertions`或`-ea`选项启用。需要注意的是，在启用或禁用断言时不必重新编译程序。启用或禁用断言是类加载器（class loader）的功能。当断言被禁用时，类加载器将跳过断言代码，因此，不会降低程序运行的速度。

## 日志