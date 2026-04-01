# 基础类库
{docsify-updated}

## 系统相关

### System 类

`System` 类代表当前 Java 程序的运行平台，程序不能创建 `System` 类的对象，但是 `System` 类提供了一些类变量和类方法供使用。

`System` 类提供了标准输入输出和错误输出的类变量（`System.in`/`System.out`/`System.err`)，并提供了一些静态方法用于访问静环境变量、系统属性，还提供了加载文件和动态链接库的方法。

+ `Map<String,String> getEnv()`:获取所有环境变量。
+ `String getEnv(String name)`:获取指定的环境变量。
+ `Properties getProperties()`:获取所有系统属性。
+ `Property getProperty(String name)`:获取指定名字的系统属性。
+ `long currentTimeMillis()`:返回当前时间与1970/1/1 00:00:00 的时间差，毫秒为单位。
+ `long nanoTime()`:同上，纳秒为单位。
+ `int identityHashCode(Object x)`:根据对象地址计算出对象的 hashCode 值。重写 `hashCode()` 不影响这个方法。

### Runtime类
`Runtime` 类代表 Java 程序运行时环境，每个 Java 程序都有一个与之关联的 `Runtime` 实例，应用程序通过该对象与其运行时环境相连。应用程序不能创建自己的 `Runtime` 实例，但可以通过类方法 `public static Runtime getRuntime()` 获取与之关联的 `Runtime` 对象。

Runtime 类代表 Java 运行时环境，可以访问 JVM 的相关信息，如处理器数量、内存信息等。
+ `public int availabeProcessors()` :处理器数量。
+ `public long freeMemory()` :空闲内数。
+ `public long totalMomery()` :总内存数。
+ `public long maxMomery()` :可用最大内存。
+ `public Process exec(String command)` :运行指定命令单独启动一个进程，返回启动进程的引用
+ `public void addShutdownHook(Thread hook)` : 注册一个新的 virtual-machine shutdown hook.

#### 优雅停机支持
JVM虚拟机会在以下两种情况下关闭：
1. 程序运行完毕，所有的非 daemon 线程都退出执行或者 `System.exit()` 方法执行
2. JVM接收到一些中断信号，比如 `^C`，或者一些系统事件，比如用户退出登录或者系统关机

shutdown hook 就是一个初始化好的但是没有启动的线程。当JVM虚拟机开始关闭时，它会启动所有的注册过的 shutdown hook 线程，它们会以不确定的顺序并发地执行。当所有的 hooks 执行完毕，JVM虚拟机就会关闭。请注意：在关闭期间，守护线程将继续运行，而如果关闭是通过调用 `exit()` 方法启动的，则非守护线程也会继续运行。

一旦关闭序列开始，只有通过调用 `halt()` 方法才能停止，该方法会强制终止虚拟机。此外，一旦关闭序列开始，就无法注册新的关闭钩子或取消注册已注册的钩子。尝试执行这些操作中的任意一个都会导致抛出 `IllegalStateException` 异常。

关闭钩子在虚拟机生命周期的一个关键时刻运行，因此应该以防御性方式进行编码。特别是，它们应该是线程安全的，并尽可能避免死锁。此外，关闭钩子不应盲目依赖可能已注册其自身关闭钩子并因此可能正在关闭的服务。例如，尝试使用基于线程的其他服务（如 AWT 事件分派线程）可能会导致死锁。

关闭钩子还应尽快完成其工作。当程序调用 `exit()` 时，期望虚拟机能迅速关闭并退出。当虚拟机因用户注销或系统关闭而终止时，底层操作系统可能仅允许固定时间来关闭和退出。因此，不建议在关闭钩子中尝试与用户交互或执行长时间运行的计算。

未捕获的异常在关闭钩子中与其他线程一样被处理，方法是调用线程的 `ThreadGroup` 对象的 `uncaughtException()` 方法。该方法的默认实现会将异常的堆栈跟踪打印到 `System.err`，然后终止线程；它不会导致虚拟机退出或停止。

在极少数情况下，虚拟机可能会中止运行，即在未正常关闭的情况下停止运行。这种情况发生在虚拟机被外部强制终止时，例如在 Unix 系统中收到 `SIGKILL` 信号或在 Microsoft Windows 中调用 `TerminateProcess` 。如果本地方法出现问题，例如破坏了内部数据结构或尝试访问不存在的内存，虚拟机也可能中止。如果虚拟机中止，则无法保证是否会运行任何关闭钩子。

### 获取机器的物理内存
Runtime 只能获取 JVM 中的内存信息，要获取物理机器的内存信息使用下面代码：
```
import java.lang.management.OperatingSystemMXBean;
import java.lang.management.ManagementFactory;

public class A{
    public static void main(String[] args){
        OperatingSystemMXBean osBean = ManagementFactory.getOperatingSystemMXBean();
        long totalMemory;
        if (osBean instanceof com.sun.management.OperatingSystemMXBean) {
            totalMemory = ((com.sun.management.OperatingSystemMXBean) osBean).getTotalPhysicalMemorySize();
            System.out.println("总物理内存大小：" + totalMemory / (1024 * 1024) + "MB");
        } else {
            System.out.println("该操作系统不支持获取物理内存大小");
        }
    }
}
```

## 常用类

### Object类
+ `boolean equals(Object obj)`:判断指定对象与调用对象的引用是否相等，直接用 == 号比较两个对象。
+ `void finalize(Object obj)`:清理对象时的方法。
+ `Class<?> getClass()`:返回该对象的运行时类，真实类型。
+ `int hashCode()`:返回该对象的 hashcode 值，默认情况下根据对象的地址来计算（与 `System.identityHashCode()`一致），但很多类重写了该方法。
+ `int toString()`:返回该对象的运行时类名+@+十六进制的 hashcode 值。
+ `Object clone()`:复制当前对象（浅复制），返回当前对象的一个副本。

<center><img src="pics/copy.png" alt="" width=40%></center>

### Objects类
`Objects` 类提供了一些方法来操作对象（Object类），且这些方法大多是“空指针”安全的。就是说，如果你不能确定一个对象是否是 `null` ，如果贸然调用它的实例方法，就会引发 `NullPointerexception` ，但是如果使用 `Objects` 类提供的方法就不会。
+ `String toString(Object o)`:如果 o 为 `null`，则返回 "null"。
+ `int hashCode(Object o)`:如果 o 为 `null`，则返回 0。
+ `T requireNoNull(T o)`:如果 o 为 `null`，抛出异常，否则返回参数本身。

### 日期时间相关 Date/Calendar
`Calendar` 是抽象类，不能实例化，需要通过它的类方法 `getInstance()` 获取实例。可以传入 TimeZone 、 Locale 类来获取指定的 Calendar ，如果不指定，则使用默认的 TimeZone 、 Locale 来创建 Calendar 。

两者的相互转换：
```
Calendar calendar = Calendar.getInstance();
Date date = calendar.getTime();

calendar.setTime(date);
```
Java 8 新增了 `java.time` 包，包含了许多与时间相关的类。

## 国际化（I18N）与本地化（L10N）
Java 程序的国际化主要通过如下三个类完成：
+ `java.util.ResourceBundle`:用于加载国家、语言资源包。
+ `java.util.Locale`:用于封装特定国家/区域、语言环境。
+ `java.text.MessageFormat`:用于格式化带占位符的字符串。

为了实现国际化，必须先提供资源文件。资源文件的内容是很多的 key-value 对， key会在程序中被引用，对应的 value 是显示的字符串。  
资源文件的命名是有规范的：
+ `baseName_language_country.propertites` 
+ `baseName_language.propertites`  
+ `baseName.propertites`

baseName是资源文件的基本名，可以随意指定；language 和 country 必须是 Java 支持的语言和国家，不能随意变化。
1. 新建 test.properties 文件，内容为 `hello=你好，{0}`;
2. 新建 test_en_US.properties文件，内容为 `hello=Welcome You,{0}`;
3. 使用JDK工具 `native2ascii` : `native2ascii test.properties test_zh_CN.properties`， 将非西欧文字转换成 Unicode 编码文件
4. `native2ascii -encoding UTF-8 test.properties test_2.properties`
5. 使用下边代码测试：
 ```
Locale locale = Locale.getDefault(Locale.Category.FORMAT);
ResourceBundle bundle = ResourceBundle.getBundle("test",locale);
String str = bundle.getString("hello);
MessageFormat.format(str,"wzz");
System.out.println();
 ```

## 属性文件加载
`Properties` 类可以方便的加载配置文件，加载文件后，相当于一个 key/value 都是 String 的 Map。
+ `String getProperty(String key)`:获取指定 key 的值。
+ `String getProperty(String key, String defaultValue)`:获取指定 key 的值， key 不存在时，返回默认值 defaultValue 。
+ `String setProperty(String key, String value)`:设置属性值。
+ `void load(InputStream in)`:从属性文件中追加 key-value 属性对到 Properties 对象中。
+ `void store(OutputStream out, String comments)`:将 Properties 对象中的 key-value 对输出到指定的属性文件中，并添加 comments 。不是追加，会覆盖整个文件内容。

### Math类
+ `double floor(double a)`: 向下取整
+ `double ceil(double a)`: 向上取整
+ `static double pow(double a, double b)`: 计算 a 的 b 次方
+ `int round(float a)`: 四舍五入值
+ `double random()`: （0，1）区间的随机值
+ `double abs(double a)`: 取绝对值
+ `int max(int a, int b)`: 取两者之间的最大值
+ `int min(int a, int b)`: 取两者之间的最小值