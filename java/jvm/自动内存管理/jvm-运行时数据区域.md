# java基础-JVM数据区域及参数设置
{docsify-updated}
> https://mp.weixin.qq.com/s/fg2Dy0Dbhcrn5QaNydp1WQ


<center>
<img src="pics/jvm运行时数据区.png" alt="JVM运行时数据区" width="40%" >
<img src="pics/java-memory-model.png" alt="JVM运行时数据区" width="55%" >

<img src="pics/jvm-memory.webp" alt="JVM运行时数据区" width="100%" >
</center>

JVM主要可以分为两大部分：
1. 各个线程独享的内存区域
   1. PC程序计数器
   2. JVM栈
   3. 本地方法栈
   
2. 线程间共享的内存
   1. 堆
   2. 方法区（运行时常量池是方法区的一部分）

### PC程序计数器
PC是一块较小的内存区域，用来指示当前线程所执行的字节码的行号指示器。在虚拟机的概念模型里（仅是概念模型，各种虚拟机可能会通过一些高效的方式去实现），字节码解释器工作时就是改变这个计数器的值来选取下一条需要执行的字节码指令，需要靠计数器来实现分支、跳转、循环、异常处理、线程恢复等基础功能。如果线程正在执行的是一个Java方法，这个计数器记录的是正在执行的虚拟机字节码指令的地址；如果正在执行的是本地（Native）方法，这个计数器值则应为空（Undefined）。

每个线程都有独立的程序计数器，各个线程之间的PC互不影响，是一块线程的私有内存。PC计数器是 JVM 规范中唯一没有规定任何 OOM 异常的内存区域。

### JVM 栈
栈也是线程私有的数据区，它的生命周期与线程相同。JVM 栈描述的是 Java 方法执行的内存模型：每个方法在执行的同时都会创建一个虚拟机栈用于存储**局部变量表、操作数栈、动态链接、方法出口**等信息。每一个方法调用直至执行完成的过程，对应这一个栈帧在虚拟机栈中从入栈到出栈的过程。

此内存区域存在两种内存异常： **StackOverFlow 异常和 OOM 异常**。栈深度大于 JVM 允许的最大深度时，抛出 StackOverFlow 异常。 如果 JVM 支持动态扩展，如果扩展时无法申请到足够内存，将抛出 OOM 异常。

`-Xss<size>` :设置栈内存大小， `-Xss1M`


### 本地方法栈
与 JVM 栈一样，只是本地方法栈为 Native 方法服务。本地方法栈也会抛出 **StackOverFlow 异常和 OOM 异常**。

### Java 堆
对于大多数应用来说， Java 堆是 JVM 管理的内存中最大的一块，是被所有线程共享的一块内存区域，在 JVM 启动时创建。此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例都在这里分配内存。 Java 堆物理上可以是不连续的，只要逻辑上连续即可。 

从回收内存的角度看，由于现代垃圾收集器大部分都是基于分代收集理论设计的，所以Java堆中经常会出现“新生代”“老年代”“永久代”“Eden空间”“From Survivor空间”“To Survivor空间”等名词，这些概念在本书后续章节中还会反复登场亮相，在这里笔者想先说明的是这些区域划分仅仅是一部分垃圾收集器的共同特性或者说设计风格而已，而非某个Java虚拟机具体实现的固有内存布局，更不是《Java虚拟机规范》里对Java堆的进一步细致划分。

如果从分配内存的角度看，所有线程共享的Java堆中**可以划分出多个线程私有的分配缓冲区（Thread Local Allocation Buffer，TLAB），以提升对象分配时的效率**。不过无论从什么角度，无论如何划分，都不会改变Java堆中存储内容的共性:**无论是哪个区域，存储的都只能是对象的实例，将Java堆细分的目的只是为了更好地回收内存，或者更快地分配内存。**

Java 堆可以是固定大小的，也可以是可扩展的。当前主流虚拟机都是按照可扩展来实现的，如果堆中没有可用内存来完成对象分配，且堆也无法扩展时，将会抛出 **OOM 异常**。

+ `-Xms<size>` :设置初始 Java 堆大小，`-Xms512m`
+ `-Xmx<size>` :设置堆内存最大值，`-Xmx1g`
+ `-XX:+HeapDumpOnOutOfMemoryError` :异常时 Dump 出当前内存堆转储快照。
+ `jhat -port 7401 -J-Xmx4G dump.hprof`: 使用 jhat 分析 Dump 出来的转储快照。

### 方法区(元数据区)
方法区与 Java 堆一样，是各个线程共享的内存区域，用于存储已被虚拟机加载的**类信息(方法代码)、常量、静态变量、 JIT 编译后代码等数据**。 JVM 规范对方法区的限制非常宽松，除了和 Java 堆一样不需要连续的物理内存外和可以选择固定大小或可扩展外，还可以不实现垃圾收集。此区域的的内存回收目标主要是针对常量池的回收和类的卸载。

HotSpot 虚拟机中，之前将此区域实现为永久代，到了JDK8，已经移除了永久代的概念，采用**元数据区**来实现。

方法区无法分配内存时，也会抛出 OOM 异常。

JDK1.6 及之前的版本：
+ `-XX:PermSize` :设置永久代最小值，`-XX:PermSize=128m`
+ `-XX:MaxPermSize` :设置永久代最大值，`-XX:MaxPermSize=512m`

JDk1.8之后：
+ `-XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=512m`
+ `-XX:CompressedClassSpaceSize`

#### 运行时常量池
运行时常量池是方法区的一部分。 Java 的 Class 文件中除了有**类的版本、字段、方法、接口**等信息外，还有一项就是常量池，**用于存放编译期生成的各种字面常量和符号引用**，这部分内容在类加载后进入方法区的运行时常量池存放。

JVM 规范对 Class 文件的每一部分（也包括常量池）的格式都有严格规定，每一个字节用于存储哪种数据都必须符合规范上的要求才会被虚拟机认可、装载和执行，但对于运行时常量池，JVM规范没有做任何细节的要求，不同的提供商实现的 JVM 可按照自己的需要来实现这个内存区域。一般来说，除了保存 Class 文件中描述的**符号引用**之外，还会把翻译出来的**直接引用**也存储在运行时常量池中。

运行时常量池相对于 Class 文件常量池的另外一个重要特征是具备动态性， Java 语言不要求常量一定只有编译期才能产生，也就是**并非只有预置入 Class 文件中的常量池内容才能进入运行时常量池，运行期间也能将新的常量放入池中，使用较多的就是 String 的 `intern()` 方法**。

运行时常量池作为方法区的一部分，也受到方法区内存的限制，当无法申请到内存时会抛出 OOM 异常。

### 直接内存
直接内存并不是虚拟机运行时数据区的一部分，也不是 JVM 规范中定义的内存区域。但是这部分内存也频繁地被使用，而且也可能导致 OOM 。

JDK 1.4 中引入了 NIO 类，引入了一种基于通道与缓冲区的 I/O 方式，它可以使用 Native 函数库直接在**堆外分配内存，然后通过存储在 Java 堆中的 `DirectByteBuffer` 对象作为这块内存的引用进行操作，避免了 Java 堆和 Native 堆中来回复制数据的开销**。

显然，直接内存不会受到 Java 堆大小的限制，但是，既然是内存，肯定还是会受到本机总内存大小及处理器寻址空间的限制。配置虚拟机参数时，不能忽略直接内存，此区域也可能会导致 OOM 异常。


### 代码缓存区
Java Code Cache 是啥： 如果 Java 每次都需要即时编译成机器码，再执行，效率太慢了。那么是不是对于某些热点代码，编译后的机器码，缓存起来，这样下次就不用重新即时编译了，多快乐。Java Code Cache 就是用来干这个的。但是编译后的机器码太大了，Java Code Cache 的空间是有限的，也不能将所有的代码都编译成机器码缓存起来。所以就需要一些优化与清理策略。

C1，C2 编译器与分层编译（Tiered Compilation）以及编译级别（Compilation Level）： C1，C2 按照老概念来看，分别是客户端编译器和服务端编译器，随着 -server 的 Java 参数的消失，代表着 Java 程序本质上不再区分客户端服务端，所以目前所有的 Java 进程都是 C1，C2混合使用。

C1 编译器对代码做一些简单的优化并加入一些采样代码， C2 针对 C1 加入的代码的采样结果，做更多的分析（语法解析，逃逸分析，高级优化器等等），优化成为更好的代码。C1是一个简单快速的编译器，主要关注点在于局部优化，而放弃许多耗时较长的全局优化手段。C2 则是关注于耗时较长的全局优化手段。同时由于 Java Code Cache 是有限的，有些代码可能优化后被淘汰，需要重新编译，没必要对于每种代码都进行 C1 C2 的优化，那些执行次数少的代码，直接编译执行即可。

Java Code Cache 分块（Segmented Code Cache）： 从 Java 9 开始引入的 Code Cache 分块，主要解决之前把所有种类代码放一起，导致扫描的时候效率低下。例如之前说的有些代码经过 C1 优化，之后 C2 优化，这些代码最好分开存储。（C1 优化过得代码，C2 优化完之后，C1的要被清理掉）。每个代码堆包含特定类型的编译代码，这样的设计可以将具有不同属性的代码分开。这样做的主要目的是提高性能，并实现未来的扩展。

目前 Java Code Cache 编译代码有三种不同的类型：
+ JVM internal (non-method) code :JVM 内部（非方法）代码，这些是 JIT 编译器要用到的内存区域，例如编译器要用的缓存等等。他们会永远存在于 Code Cache 内。
+ Profiled-code :带采样优化的代码
+ Non-profiled code :不带采样未优化的代码

相应的代码堆有:
+ 非方法代码堆（codeheap non-nmethods）：包含非方法代码，如编译器缓冲区和字节码解释器。这种代码类型将永远保留在代码缓存中。
+ 剖析代码堆(codeheap profiled nmethods)：包含经过轻度优化的剖析方法，生命周期较短。
+ 非剖析代码堆(codeheap non-profiled nmethods)：包含经过全面优化的非剖析方法，生命周期可能较长。

使用jvm选项设置内存：
+ `-XX:NonNMethodCodeHeapSize`: 非方法代码堆大小.
+ `-XX:ProfiledCodeHeapSize`: 带采样的代码堆大小.
+ `-XX:NonProfiledCodeHeapSize`: 不带采样的代码堆大小.
+ `-XX:ReservedCodeCacheSize` 以上三个加起来需要等于这个

### 垃圾收集器
垃圾回收器的结构和算法需要额外的内存来进行堆管理。这些结构包括 Mark Bitmap、Mark Stack（用于遍历对象图）、Remembered Sets（用于记录区域间引用）等。其中一些结构是可以直接调整的，例如 `-XX:MarkStackSizeMax`，其他结构则取决于堆布局，例如 G1 区域越大（-XX:G1HeapRegionSize），记忆集就越小。

不同 GC 算法的 GC 内存开销也不同。`-XX:+UseSerialGC` 和 `-XX:+UseShenandoahGC` 的内存开销最小。G1 或 CMS 可能会轻松占用堆总大小的 10% 左右。

### JIT 编译器
JIT 编译器本身也需要内存来完成工作。通过关闭分层编译或减少编译器线程数，可以再次减少内存占用：`-XX:CICompilerCount`.

## 实例
阿里云上服务 `jinfo 8` 显示的 JVM Flags:
```
-XX:CICompilerCount=4 -XX:ConcGCThreads=2 -XX:G1ConcRefinementThreads=8 -XX:G1HeapRegionSize=1048576 -XX:GCDrainStackTargetSize=64 -XX:InitialHeapSize=257949696 -XX:MarkStackSize=4194304 -XX:MaxHeapSize=4127195136 -XX:MaxNewSize=2475687936 -XX:MinHeapDeltaBytes=1048576 -XX:NonNMethodCodeHeapSize=5835340 -XX:NonProfiledCodeHeapSize=122911450 -XX:ProfiledCodeHeapSize=122911450 -XX:ReservedCodeCacheSize=251658240 -XX:+SegmentedCodeCache -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseG1GC
```

G1 算法有多个阶段，其中一些是 "stop the world" 阶段，即在垃圾收集期间停止应用程序，还有一些阶段是在应用程序运行时并发的（候选标记等），因此要考虑到这些信息： `ParallelGCThreads` 选项会影响在应用程序线程停止时阶段的G1线程数，而 `ConcGCThreads` 标志会影响用于G1与应用程序并发阶段的线程数。