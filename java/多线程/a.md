学习java多线程的工具 -> https://github.com/vgrazi/JavaConcurrentAnimatedReboot

线程主要通过共享对字段以及字段所引用的对象的访问权限来进行通信。这种通信方式极其高效，但可能导致两种错误：
+ 线程干扰
+ 内存一致性错误

防止这些错误所需的工具就是同步。


在编程中，原子操作是指实际上一次性完成的操作。原子操作不能中途停止：要么完全执行完毕，要么根本不执行。在操作完成之前，其任何副作用都不会显现。

我们已经看到，诸如 c++ 这样的递增表达式并不描述原子操作。即使是极其简单的表达式，也可能定义出可分解为其他操作的复杂操作。不过，有些操作是可以明确指定为原子的：

+ 对于引用变量和大多数基本类型变量（除 long 和 double 以外的所有类型），读取和写入操作都是原子性的。
+ 对于所有声明为 volatile 的变量（包括 long 和 double 变量），读取和写入操作都是原子性的。

原子操作不能被交错执行，因此在使用时无需担心线程干扰。但这并不意味着完全不需要对原子操作进行同步，因为内存一致性错误仍然可能发生。使用 volatile 变量可以降低内存一致性错误的风险，因为对 volatile 变量的任何写入都会与随后对该变量的读取建立“发生在……之前”的关系。这意味着对 volatile 变量的更改对其他线程始终可见。此外，这也意味着当一个线程读取一个易失性变量时，它不仅能看到该易失性变量的最新更改，还能看到导致该更改的代码所产生的副作用。

直接访问原子变量比通过同步代码访问这些变量更高效，但程序员需要更加谨慎，以避免出现内存一致性错误。这种额外努力是否值得，取决于应用程序的规模和复杂程度。


线程的问题：
+ 死锁
+ 饥饿
+ 活锁
  一个线程通常是针对另一个线程的操作而做出响应的。如果另一个线程的操作也是对另一个线程操作的响应，那么就可能导致活死锁。与死锁一样，陷入活死锁的线程无法继续执行。然而，这些线程并未被阻塞——它们只是忙于相互响应，而无法恢复工作。这可以比作两个人试图在走廊里错身而过：阿尔方斯向左让路让加斯顿通过，而加斯顿则向右让路让阿尔方斯通过。看到他们仍然互相阻挡，阿尔方斯便向右移动，而加斯顿则向左移动。他们仍然互相阻挡，所以……

## 不可变对象
如果一个对象在构造完成后其状态无法改变，则该对象被视为不可变对象。最大限度地依赖不可变对象，已被广泛认可为编写简单、可靠代码的有效策略。

不可变对象在并发应用程序中尤为有用。由于它们的状态无法改变，因此不会因线程干扰而遭到破坏，也不会被观察到处于不一致的状态。

程序员往往不愿使用不可变对象，因为他们担心创建新对象的成本，而更倾向于就地更新现有对象。对象创建的影响通常被高估了，而且这种影响可以被不可变对象带来的某些效率优势所抵消。这些优势包括因垃圾回收而减少的开销，以及无需编写保护可变对象免受破坏的代码。

```
public class SynchronizedRGB {

    // Values must be between 0 and 255.
    private int red;
    private int green;
    private int blue;
    private String name;

    private void check(int red,
                       int green,
                       int blue) {
        if (red < 0 || red > 255
            || green < 0 || green > 255
            || blue < 0 || blue > 255) {
            throw new IllegalArgumentException();
        }
    }

    public SynchronizedRGB(int red,
                           int green,
                           int blue,
                           String name) {
        check(red, green, blue);
        this.red = red;
        this.green = green;
        this.blue = blue;
        this.name = name;
    }

    public void set(int red,
                    int green,
                    int blue,
                    String name) {
        check(red, green, blue);
        synchronized (this) {
            this.red = red;
            this.green = green;
            this.blue = blue;
            this.name = name;
        }
    }

    public synchronized int getRGB() {
        return ((red << 16) | (green << 8) | blue);
    }

    public synchronized String getName() {
        return name;
    }

    public synchronized void invert() {
        red = 255 - red;
        green = 255 - green;
        blue = 255 - blue;
        name = "Inverse of " + name;
    }
}
```

使用 SynchronizedRGB 时必须谨慎，以免被观察到处于不一致状态。例如，假设一个线程执行以下代码：
```
SynchronizedRGB color =
    new SynchronizedRGB(0, 0, 0, "Pitch Black");
...
int myColorInt = color.getRGB();      //Statement 1
String myColorName = color.getName(); //Statement 2
```
如果另一个线程在语句 1 之后、语句 2 之前调用了 `color.set`，那么 `myColorInt` 的值将与 `myColorName` 的值不一致。为避免这种情况，必须将这两个语句绑定在一起：
```
synchronized (color) {
    int myColorInt = color.getRGB();
    String myColorName = color.getName();
} 
```

### 定义不可变对象的策略
以下规则定义了一种创建不可变对象的简单策略。并非所有被标注为“不可变”的类都遵循这些规则。这并不一定意味着这些类的创建者马虎——他们可能有充分的理由相信，其类的实例在构造完成后绝不会发生变化。然而，此类策略需要进行深入的分析，不适合初学者。
+ 不要提供 `setter` 方法——即修改字段或字段所引用的对象的方法。
+ 将所有字段设为 `final` 且 `private` 。
+ 不要允许子类重写方法。实现这一点最简单的方法是将类声明为 `final` 。更高级的做法是将构造函数设为 `private` ，并在工厂方法中创建实例。
+ 如果实例字段包含对可变对象的引用，请勿允许修改这些对象：
    + 不要提供用于修改可变对象的方法。
    + 不要共享对可变对象的引用。切勿存储传递给构造函数的外部可变对象的引用；如有必要，请创建副本，并存储对副本的引用。同样，在必要时请创建内部可变对象的副本，以避免在方法中返回原始对象。

将此策略应用于 SynchronizedRGB 时，具体步骤如下：

1. 该类中有两个设置器方法。第一个方法 `set` 会对对象进行任意转换，因此不适合用于该类的不可变版本。第二个方法 `invert` 则可以通过让其创建新对象而非修改现有对象的方式进行调整。
2. 所有字段均为私有；此外，它们还被声明为 `final` 。
3. 类本身也被声明为 `final` 。
4. 仅有一个字段引用了一个对象，而该对象本身也是不可变的。因此，无需采取任何防护措施来防止“包含”的可变对象的状态发生改变。

```
final public class ImmutableRGB {

    // Values must be between 0 and 255.
    final private int red;
    final private int green;
    final private int blue;
    final private String name;

    private void check(int red,
                       int green,
                       int blue) {
        if (red < 0 || red > 255
            || green < 0 || green > 255
            || blue < 0 || blue > 255) {
            throw new IllegalArgumentException();
        }
    }

    public ImmutableRGB(int red,
                        int green,
                        int blue,
                        String name) {
        check(red, green, blue);
        this.red = red;
        this.green = green;
        this.blue = blue;
        this.name = name;
    }


    public int getRGB() {
        return ((red << 16) | (green << 8) | blue);
    }

    public String getName() {
        return name;
    }

    public ImmutableRGB invert() {
        return new ImmutableRGB(255 - red,
                       255 - green,
                       255 - blue,
                       "Inverse of " + name);
    }
}
```


## 高级并发工具
+ `Lock objects`  支持各种锁定模式，可简化许多并发应用程序的实现。
+ `Executors` 定义了一个用于启动和管理线程的高级 API。java.util.concurrent 包提供的执行器实现提供了适用于大规模应用程序的线程池管理功能。
+ `Concurrent collections` 使管理大型数据集合变得更加容易，并能大大减少对同步的需求。
+ `Atomic variables` 具有能够最大限度减少同步操作并帮助避免内存一致性错误的特性。
+ `ThreadLocalRandom`（在 JDK 7 中）支持从多个线程高效生成伪随机数。


### Executors
在上述所有示例中，由 `Runnable` 对象定义的新线程所执行的任务与其由 `Thread` 对象定义的线程本身之间存在着密切的联系。这种方式对于小型应用程序来说效果良好，但在大型应用程序中，将线程管理和创建与应用程序的其他部分分离更为合理。封装这些功能的对象被称为执行器。
+ `Executor` 接口定义了三种执行器对象类型。
+ 线程池是最常见的执行器实现方式。
+ `Fork/Join` 是一个（在 JDK 7 中新增的）用于充分利用多处理器的框架。