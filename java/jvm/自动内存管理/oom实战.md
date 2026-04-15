# OOM 实战
{docsify-updated}

## Java堆溢出
将堆的最小值 `-Xms` 参数与最大值 `-Xmx` 参数设置为一样即可避免堆自动扩展， 通过参数 `-XX:+HeapDumpOnOutOfMemoryError` 可以让虚拟机在出现内存溢出异常的时候Dump出当前的内存堆转储快照以便进行事后分析。

```
/**
 * VM Args:-Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
 * @author zzm
 */
public class HeapOom {
    static class OOMObject {
    }
    public static void main(String[] args) {
        List<OOMObject> list = new ArrayList<OOMObject>();
        while (true) {
            list.add(new OOMObject());
        }
    }
}

java.lang.OutOfMemoryError: Java heap space
Dumping heap to java_pid20109.hprof ...
Heap dump file created [33034372 bytes in 0.038 secs]
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.base/java.util.Arrays.copyOf(Arrays.java:3509)
	at java.base/java.util.Arrays.copyOf(Arrays.java:3478)
	at java.base/java.util.ArrayList.grow(ArrayList.java:238)
	at java.base/java.util.ArrayList.grow(ArrayList.java:245)
	at java.base/java.util.ArrayList.add(ArrayList.java:484)
	at java.base/java.util.ArrayList.add(ArrayList.java:497)
	at com.gtja.gjyw.HeapOom.main(HeapOom.java:22)
```

要解决这个内存区域的异常，常规的处理方法是首先通过内存映像分析工具（如 `Eclipse Memory Analyzer` ）对Dump出来的堆转储快照进行分析。第一步首先应确认内存中导致OOM的对象是否是必要的，也就是要先分清楚到底是出现了**内存泄漏**（Memory Leak）还是**内存溢出**（Memory Overflow）。

如果是内存泄漏，可进一步通过工具查看泄漏对象到GC Roots的引用链，找到泄漏对象是通过怎样的引用路径、与哪些GC Roots相关联，才导致垃圾收集器无法回收它们，根据泄漏对象的类型信息以及它到GC Roots引用链的信息，一般可以比较准确地定位到这些对象创建的位置，进而找出产生内存泄漏的代码的具体位置。

如果不是内存泄漏，换句话说就是内存中的对象确实都是必须存活的，那就应当检查Java虚拟机的堆参数（ `-Xmx` 与 `-Xms` ）设置，与机器的内存对比，看看是否还有向上调整的空间。再从代码上检查是否存在某些对象生命周期过长、持有状态时间过长、存储结构设计不合理等情况，尽量减少程序运行期的内存消耗。

## 方法栈溢出
栈容量只能由 `-Xss` 参数来设定。 关于虚拟机栈和本地方法栈，在《Java虚拟机规范》中描述了两种异常：
+ 如果线程请求的栈深度大于虚拟机所允许的最大深度，将抛出 `StackOverflowError` 异常。
+ 如果虚拟机的栈内存允许动态扩展，当扩展栈容量无法申请到足够的内存时，将抛出 `OutOfMemoryError` 异常。

《Java虚拟机规范》明确允许Java虚拟机实现自行选择是否支持栈的动态扩展，**而HotSpot虚拟机的选择是不支持扩展**，所以除非在创建线程申请内存时就因无法获得足够内存而出现 `OutOfMemoryError` 异常，否则在线程运行时是不会因为扩展而导致内存溢出的，只会因为栈容量无法容纳新的栈帧而导致 `StackOverflowError` 异常。

## 本机直接内存溢出
直接内存（Direct Memory）的容量大小可通过 `-XX:MaxDirectMemorySize` 参数来指定，如果不去指定，则默认与Java堆最大值（由 `-Xmx` 指定）一致，代码清单2-10越过了 `DirectByteBuffer` 类直接通过反射获取 `Unsafe` 实例进行内存分配（ `Unsafe` 类的 `getUnsafe()` 方法指定只有引导类加载器才会返回实例，体现了设计者希望只有虚拟机标准类库里面的类才能使用 `Unsafe` 的功能，在JDK 10时才将 `Unsafe` 的部分功能通过 `VarHandle` 开放给外部使用），因为虽然使用 `DirectByteBuffer` 分配内存也会抛出内存溢出异常，但它抛出异常时并没有真正向操作系统申请分配内存，而是通过计算得知内存无法分配就会在代码里手动抛出溢出异常，真正申请分配内存的方法是 `Unsafe::allocateMemory()` 。

Java 25 中的示例报错代码:
```
/**
 * VM Args:-Xmx20M -XX:MaxDirectMemorySize=10M
 */
public class DirectMemoryOOM {
    private static final int _1MB = 1024 * 1024;
    public static void main(String[] args) throws Exception {
        List<ByteBuffer> list = new ArrayList<>();

        while (true) {
            // 分配直接内存
            ByteBuffer buffer = ByteBuffer.allocateDirect(_1MB);
            list.add(buffer); // 防止被 GC 回收
        }
    }
}

Exception in thread "main" java.lang.OutOfMemoryError: Cannot reserve 1048576 bytes of direct buffer memory (allocated: 10485760, limit: 10485760)
	at java.base/java.nio.Bits.reserveMemory(Bits.java:178)
	at java.base/java.nio.DirectByteBuffer.<init>(DirectByteBuffer.java:108)
	at java.base/java.nio.ByteBuffer.allocateDirect(ByteBuffer.java:367)
	at com.gtja.gjyw.DirectMemoryOOM.main(App.java:35)
```

由直接内存导致的内存溢出，一个明显的特征是在Heap Dump文件中不会看见有什么明显的异常情况，如果读者发现内存溢出之后产生的Dump文件很小，而程序中又直接或间接使用了 `DirectMemory`（典型的间接使用就是NIO），那就可以考虑重点检查一下直接内存方面的原因了。