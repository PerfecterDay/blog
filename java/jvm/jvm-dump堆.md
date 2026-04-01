# Dump Heap 堆与分析
{docsify-updated}
- [Dump Heap 堆与分析](#dump-heap-堆与分析)
	- [Dump 堆](#dump-堆)
		- [JDK 工具](#jdk-工具)
			- [jmap](#jmap)
			- [jcmd](#jcmd)
		- [自动捕获堆转储](#自动捕获堆转储)
	- [Jhat分析Dump堆文件](#jhat分析dump堆文件)
	- [Eclipse Memory Analyzer 分析 Dump 堆](#eclipse-memory-analyzer-分析-dump-堆)
		- [Dominator Tree](#dominator-tree)
		- [Thread Overview线程视图](#thread-overview线程视图)


## Dump 堆
堆转储是 JVM 中某一时刻内存中所有对象的快照。它们对于排除内存泄漏问题和优化Java应用程序中的内存使用非常有用。

堆转储通常存储在二进制格式的 `hprof` 文件中。我们可以使用 `jhat` 或 `JVisualVM` 等工具打开并分析这些文件。

### JDK 工具
JDK 自带了多种工具，可通过不同方式捕获堆转储。所有这些工具都位于 JDK 主目录下的 bin 文件夹中。因此，只要系统路径中包含该目录，我们就可以通过命令行启动它们。

#### jmap
jmap 是一个打印运行中的 JVM 内存统计信息的工具。我们可以将其用于本地或远程进程。  
`jmap -dump:[live],format=b,file=<file-path> <pid>`  
我们还应该指定几个参数：
+ `live`：如果设置了，则只打印有活动引用的对象，并丢弃准备被垃圾回收的对象。该参数为可选参数。
+ `format=b`：指定转储文件为二进制格式。如果未设置，则默认二进制。
+ `file`：将转储写入的文件
+ `pid`：Java 进程的 id

示例：
```
jmap -dump:live,format=b,file=/tmp/dump.hprof 12587

```
另外，请记住 jmap 是 JDK 中作为实验工具引入的，不受支持。因此，在某些情况下，最好使用其他工具来代替它

#### jcmd
jcmd 是一个非常完整的工具，它通过向 JVM 发送命令请求来工作。我们必须在运行 Java 进程的同一台机器上使用它。与 jmap 一样，生成的数据转储为二进制格式。

`GC.heap_dump` 就是它的众多命令之一。只需指定进程的 `pid` 和输出文件路径，我们就能用它获取堆转储：
```
jcmd <pid> GC.heap_dump <file-path>
```
示例：`jcmd 12587 GC.heap_dump /tmp/dump.hprof`

### 自动捕获堆转储
我们在前面章节中展示的所有工具都是为了在特定时间手动捕获堆转储。在某些情况下，我们希望在发生 `java.lang.OutOfMemoryError` 时获取堆转储，以帮助我们调查错误。  
针对这些情况，Java 提供了 `HeapDumpOnOutOfMemoryError` 命令行选项，它可以在抛出 `java.lang.OutOfMemoryError` 时生成堆转储：
```
java -XX:+HeapDumpOnOutOfMemoryError
```

默认情况下，它会将转储存储在运行应用程序的目录下的 `java_pid<pid>.hprof`(java_pid12587.hprof)文件中。如果我们想指定其他文件或目录，可以在 `HeapDumpPath` 选项中进行设置：
```
java -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=<file-or-dir-path>
```
该选项非常有用，使用该选项运行应用程序不会产生任何开销。因此，**强烈建议始终使用该选项，尤其是在生产过程中。**


## Jhat分析Dump堆文件
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


## Eclipse Memory Analyzer 分析 Dump 堆
[安装问题解决](https://bbs.huaweicloud.com/blogs/356789)
```
<string>-vm</string>
<string>/opt/homebrew/opt/sdkman-cli/libexec/candidates/java/current/bin/java</string>
```

<canter><img src="pics/mat.png" width="80%"></canter>

核心概念：
1. `shallow size` : 浅堆，代表了对象本身的内存占用，以及“为了引用”其他对象所占用的内存(引用指针的大小)，引用对象本身的大小不算。非数组类型的对象的 $shallow-heap=对象头+各成员变量大小之和+对齐填充$。其中，各成员变量大小之和就是实例数据，如果存在继承的情况，需要包括父类成员变量。数组类型的对象的$shallow-heap=对象头+类型变量大小*数组长度+对齐填充$，如果是引用类型，则是四字节或者八字节（64 位系统），如果是 boolean 类型，则是一个字节。这里类型变量大小*数组长度就是数组实例数据，强调是变量不实际是对象本身。
2. `Retained heap`：深堆，一个统计结果，会循环计算引用的具体对象所占用的内存。但是深堆和“对象大小”有一点不同，深堆指的是一个对象被垃圾回收后，能够释放的内存大小，这些被释放的对象集合，叫做保留集（Retained Set）。
3. `outgoing reference`: 对外部其他对象的引用
4. `incoming reference` : 引用当前对象的引用


### Dominator Tree
Dominator Tree-支配树： 列出Heap Dump中处于活跃状态中的最大的几个对象，默认按 `retained size` 进行排序，因此很容易找到占用内存最多的对象。排在第一的最大的对象就是占用内存最多的对象，它在树中的子节点都是被该对象直接或间接引用的对象。


### Thread Overview线程视图
可以生成Heap Dump文件的时候线程视图Thread Overview，查看线程的运行情况，抛出的异常的分析。