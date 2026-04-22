# java基础-常用jvm工具
{docsify-updated}

## jps
JVM Process Status Tool,显示指定系统内所有的HotSpot虚拟机进程。

**常用参数：**
+ `-l` : 输出主类全名或jar路径
+ `-q` : 只输出LVMID
+ `-m` : 输出JVM启动时传递给main()的参数
+ `-v` : 输出JVM启动时显示指定的JVM参数

## jstat
jstat(JVM statistics Monitoring)是用于监视虚拟机运行时状态信息的命令，它可以显示出虚拟机进程中的类装载、内存、垃圾收集、JIT编译等运行数据。

**命令格式：**
`jstat [option] LVMID [interval] [count]`
+ `[option]`:操作参数
+ `LVMID`:本地虚拟机进程ID
+ `[interval]`：连续输出的时间间隔
+ `[count]` : 连续输出的次数

**常用参数：**
+ `-class`: class loader的行为统计
+ `-compiler`: HotSpt JIT编译器行为统计
+ `-gc`: 垃圾回收的行为统计
+ `-gccapacity`: 各个垃圾回收代容量(young,old,perm)和他们相应的空间统计
+ `-gccause`: 垃圾收集统计概述（同-gcutil），附加最近两次垃圾回收事件的原因
+ `-gcmetacapacity`: 元空间垃圾收集行为统计
+ `-gcnew`: 新生代垃圾收集行为统计
+ `-gcnewcapacity`: 新生代与其相应的内存空间的统计
+ `-gcold`: 年老代行为统计
+ `-gcoldcapacity`:  年老代与其相应的内存空间的统计
+ `-gcutil`: 垃圾回收统计概述
+ `-printcompilation`: HotSpot编译方法统计

实例： `jstat -gc 17016 2s 3` 每隔2s打印一次垃圾收集行为，总共打印3次
<center>
<img src="pics/jstat-gc.png">
</center>

+ `S0C` : survivor0区的总容量
+ `S1C` : survivor1区的总容量
+ `S0U` : survivor0区已使用的容量
+ `S1U` : survivor1区已使用的容量
+ `EC` : Eden区的总容量
+ `EU` : Eden区已使用的容量
+ `OC` : Old区的总容量
+ `OU` : Old区已使用的容量
+ `MC` ： 当前 MetaSpace 的容量 (KB)
+ `MU` : MetaSpace 的使用 (KB)
+ `YGC` : 新生代垃圾回收次数
+ `YGCT` : 新生代垃圾回收时间
+ `FGC` : 老年代垃圾回收次数
+ `FGCT` : 老年代垃圾回收时间
+ `CGC` :  Concurrent s Collection
+ `CGCT` : Concurrent Total Garbage Collection
+ `CCSC` : 压缩类空间大小
+ `CCSU` : 压缩类空间使用大小
+ `GCT` : 垃圾回收总消耗时间

## jmap
jmap(JVM Memory Map)命令用于生成 heap dump 文件，如果不使用这个命令，还阔以使用 `-XX:+HeapDumpOnOutOfMemoryError` 参数来让虚拟机出现OOM的时候自动生成dump文件。 jmap不仅能生成dump文件，还可以查询finalize执行队列、Java堆和永久代的详细信息，如当前使用率、当前使用的是哪种收集器等。

**命令格式：**
`jmap [option] LVMID`
`jmap -dump:format=b,file=heapdump.hprof pid`

**常用参数：**
+ `-dump` : 生成堆转储快照
+ `-finalizerinfo` : 显示在F-Queue队列等待Finalizer线程执行finalizer方法的对象
+ `-heap` : 显示Java堆详细信息
+ `-histo` : 显示堆中对象的统计信息
+ `-permstat` : to print permanent generation statistics
+ `-F` : 当-dump没有响应时，强制生成dump快照

关于如何dump heap，参考：
> https://www.baeldung.com/java-heap-dump-capture

## jhat
jhat(JVM Heap Analysis Tool)命令是与jmap搭配使用，用来分析jmap生成的dump，jhat内置了一个微型的HTTP/HTML服务器，默认端口7000，生成dump的分析结果后，可以在浏览器中查看。在此要注意，一般不会直接在服务器上进行分析，因为jhat是一个耗时并且耗费硬件资源的过程，一般把服务器生成的dump文件复制到本地或其他机器上进行分析。

**命令格式：**
`jhat [dumpfile]`

**常用参数：**
+ `-stack false|true`: 关闭对象分配调用栈跟踪(tracking object allocation call stack)。 如果分配位置信息在堆转储中不可用. 则必须将此标志设置为 false. 默认值为 true.>
+ `-refs false|true`: 关闭对象引用跟踪(tracking of references to objects)。 默认值为 true. 默认情况下, 返回的指针是指向其他特定对象的对象,如反向链接或输入引用(referrers or incoming references), 会统计/计算堆中的所有对象。
+ `-port port-number` 设置 jhat HTTP server 的端口号. 默认值 7000
+ `-exclude exclude-file` 指定对象查询时需要排除的数据成员列表文件(a file that lists data members that should be excluded from the reachable objects query)。 例如, 如果文件列列出了 java.lang.String.value , 那么当从某个特定对象 Object o 计算可达的对象列表时, 引用路径涉及 java.lang.String.value 的都会被排除。>
+ `-baseline exclude-file` 指定一个基准堆转储(baseline heap dump)。 在两个 heap dumps 中有相同 object ID 的对象会被标记为不是新的(marked as not being new). 其他对象被标记为新的(new). 在比较两个不同的堆转储时很有用.>
+ `-debug int` 设置 debug 级别. 0 表示不输出调试信息。 值越大则表示输出更详细的 debug 信息.>
+ `-version` 启动后只显示版本信息就退出>
+ `-J flag` 因为 jhat 命令实际上会启动一个JVM来执行, 通过 -J 可以在启动JVM时传入一些启动参数. 例如, -J-Xmx512m 则指定运行 jhat 的Java虚拟机使用的最大堆内存为 512 MB. 如果需要使用多个JVM启动参数,则传入多个 -Jxxxxxx.

## jstack
jstack用于生成java虚拟机当前时刻的线程快照。线程快照是当前java虚拟机内每一条线程正在执行的方法堆栈的集合，生成线程快照的主要目的是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等。 线程出现停顿的时候通过jstack来查看各个线程的调用堆栈，就可以知道没有响应的线程到底在后台做什么事情，或者等待什么资源。 如果java程序崩溃生成core文件，jstack工具可以用来获得core文件的java stack和native stack的信息，从而可以轻松地知道java程序是如何崩溃和在程序何处发生问题。另外，jstack工具还可以附属到正在运行的java程序中，看到当时运行的java程序的java stack和native stack的信息, 如果现在运行的java程序呈现hung的状态，jstack是非常有用的。

**命令格式：**
`jmap [option] LVMID`

**常用参数：**
+ `-F` : 当正常输出请求不被响应时，强制输出线程堆栈
+ `-l` : 除堆栈外，显示关于锁的附加信息
+ `-m` : 如果调用到本地方法的话，可以显示C/C++的堆栈

## jinfo
jinfo(JVM Configuration info)这个命令作用是实时查看和调整虚拟机运行参数。 之前的`jps -v`口令只能查看到显示指定的参数，如果想要查看未被显示指定的参数的值就要使用jinfo口令

**命令格式：**
`jinfo [option] [args] LVMID`

**常用参数：**
+ `-flag` : 输出指定args参数的值
+ `-flags` : 不需要args参数，输出所有JVM参数的值
+ `-sysprops` : 输出系统属性，等同于System.getProperties()
+ `jinfo 8 | grep GC`: 查看jvm使用的是什么垃圾收集器
+ `jinfo 8`: 查看jvm的信息
+ ```jinfo -flag  +PrintGC `pgrep -f tomcat` ```: 动态开启VM参数
+ `jinfo -flag SoftMaxHeapSize=2048 {PID} (sets it to 2M)` : 动态设置VM参数值

并不是所有JVM参数都可以动态设置，只有 `java -XX:+PrintFlagsInitial -vesion` 中输出的  `manageable` 类型的 flags 才能动态修改。

## jcmd
| 传统命令      | jcmd 等价                          |
|--------------|-----------------------------------|
| jstack       | jcmd <pid> Thread.print           |
| jmap -heap   | jcmd <pid> GC.heap_info           |
| jmap -histo  | jcmd <pid> GC.class_histogram     |
| jstat        | jcmd <pid> VM.uptime 等           |
| jinfo        | jcmd <pid> VM.flags               |