# JVM Ergonomics
{docsify-updated}

> https://docs.oracle.com/en/java/javase/23/gctuning/introduction-garbage-collection-tuning.html

在 JVM 和 GC（垃圾回收）相关的上下文中，**Ergonomics（自适应优化） 指的是 JVM 根据 硬件配置 和 应用程序行为 自动调整 GC 设置和参数，以优化性能，而无需用户手动指定所有 JVM 选项。JVM 在启动时，会根据 系统环境（如 CPU 核心数、可用内存大小）自动选择 GC 类型和默认参数，这就是 GC Ergonomics 的概念**

+ JVM 提供了与平台相关的**默认选择**，包括垃圾回收器、堆大小和运行时编译器。这些默认设置能够满足不同类型应用程序的需求，同时减少对命令行参数的调整需求。
+ 此外，基于行为的调优（behavior-based tuning）能够动态优化堆的大小，以满足应用程序的特定性能需求。

## 垃圾收集器、堆和运行时编译器默认选择
这些是重要的垃圾收集器、堆大小和运行时编译器默认选项：
+ 在 `server-class` 计算机上使用**Garbage-First (G1) 收集器**，否则使用 **Serial收集器**。（如果虚拟机检测到两个以上处理器，且堆大小大于或等于 1792 MB，虚拟机就会将机器视为 `server-class` 。）
+ GC 线程的最大数量受堆大小和可用 CPU 资源的限制 
+ 初始堆大小为物理内存的 1/64
+ 最大堆大小为物理内存的 1/4 
+ 分层编译器（JIT），同时使用 C1 和 C2

以下是一个JDK 11 JVM进程部署在阿里云K8S后，使用 jinfo 输出的示例，所有单位都是字节。机器配置为：内存-22.446Gi，deployment 中配置为2C 2G。
```
-XX:CICompilerCount=2 -XX:ConcGCThreads=1 -XX:G1ConcRefinementThreads=2 -XX:G1EagerReclaimRemSetThreshold=32 -XX:G1HeapRegionSize=4194304 
-XX:GCDrainStackTargetSize=64 -XX:InitialHeapSize=364904448 -XX:MarkStackSize=4194304 -XX:MaxHeapSize=5809111040 -XX:MaxNewSize=3485466624 
-XX:MinHeapDeltaBytes=4194304 -XX:MinHeapSize=8388608 -XX:NonNMethodCodeHeapSize=5826188 -XX:NonProfiledCodeHeapSize=122916026 
-XX:ProfiledCodeHeapSize=122916026 -XX:ReservedCodeCacheSize=251658240 -XX:+SegmentedCodeCache -XX:SoftMaxHeapSize=5809111040 
-XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseFastUnorderedTimeStamps -XX:+UseG1GC 
```



## Behavior-Based Tuning
Java HotSpot VM 垃圾收集器可配置为优先满足两个目标中的一个：
+ 最长暂停时间
+ 应用程序吞吐量。

如果首选目标已实现，垃圾收集器将尝试最大化另一个目标。 当然，这些目标并非总能实现。

### Maximum Pause-Time Goal
**暂停时间是垃圾回收器停止应用程序并回收内存空间的持续时间。 最大暂停时间目标的目的是限制最长的暂停时间**。

垃圾回收器会维护暂停的**平均时间**和**平均时间的方差**。 平均值从JVM开始执行时开始统计，但会进行加权处理，因此最近的暂停时间会被更多地计算在内。 如果平均值加上暂停时间的方差大于最大暂停时间目标，那么垃圾回收器就会认为目标没有达到。

最大暂停时间目标可通过命令行选项 `-XX:MaxGCPauseMillis=<nnn> milliseconds` 指定。 表示需要 `<nnn>` 毫秒或更少的暂停时间 。 垃圾收集器会调整 Java 堆大小和其他与垃圾收集相关的参数，以尽量使垃圾收集暂停时间短于 `<nnn>` 毫秒。 最大暂停时间目标的默认值因收集器而异。 这些调整可能会导致垃圾收集发生得更频繁，从而降低应用程序的总体吞吐量。 不过，在某些情况下，无法实现所需的暂停时间目标。

### Throughput Goal
吞吐量目标是根据收集垃圾所花费的时间来衡量的，而垃圾收集以外的时间就是应用时间。

该目标由命令行选项 `-XX:GCTimeRatio=nnn` 指定。 垃圾回收时间与应用程序时间的比率为 `1/(1+nnn)`。 例如，`-XX:GCTimeRatio=19` 设置的目标是垃圾收集时间的 1/20 或总时间的 5%。 

垃圾收集所用时间是所有垃圾收集引起的暂停的总时间。 如果没有达到吞吐量目标，那么垃圾收集器可能采取的措施之一就是增加堆的大小，从而延长两次收集暂停之间在应用程序中花费的时间。

**这两个目标很多时候是相悖的，要想缩短GC最大暂停时间，通常要缩小堆大小，堆内存小所以一次GC所需要的时间也会少。但是，堆太小又可能会导致比大堆更频繁的产生GC，这就增加了GC的频率和时间，有可能降低吞吐量。**

### Footprint(占用量)
如果达到了吞吐量和最长暂停时间目标，垃圾收集器就会减少堆的大小，直到其中一个目标（通常是吞吐量目标）无法实现为止。 可以使用 `-Xms=<nnn>` 和 `-Xmx=<mmm>` 分别设置垃圾收集器可以使用的最小堆大小和最大堆大小。

## 调优策略
不要选择堆的最大值，除非你知道你需要一个大于默认最大堆大小的堆。 选择一个足以满足应用需求的吞吐量目标。

如果堆增长到最大值，但吞吐量目标仍未实现，则说明最大堆大小对于吞吐量目标来说太小。 将最大堆大小设置为接近平台物理内存总量的值。 再次执行应用程序。 如果仍未达到吞吐量目标，则说明对于平台上的可用内存而言，应用时间目标过高。