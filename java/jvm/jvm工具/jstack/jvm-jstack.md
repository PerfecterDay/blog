# jstack
{docsify-updated}

`jstack` 用于生成java虚拟机当前时刻的线程快照。线程快照是当前java虚拟机内每一条线程正在执行的方法堆栈的集合，生成线程快照的主要目的是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等。 线程出现停顿的时候通过jstack来查看各个线程的调用堆栈，就可以知道没有响应的线程到底在后台做什么事情，或者等待什么资源。 如果java程序崩溃生成core文件，jstack工具可以用来获得core文件的java stack和native stack的信息，从而可以轻松地知道java程序是如何崩溃和在程序何处发生问题。另外，jstack工具还可以附属到正在运行的java程序中，看到当时运行的java程序的java stack和native stack的信息, 如果现在运行的java程序呈现hung的状态，jstack是非常有用的。

**命令格式：**
`jmap [option] LVMID`

**常用参数：**
+ `-F` : 当正常输出请求不被响应时，强制输出线程堆栈
+ `-l` : 除堆栈外，显示关于锁的附加信息
+ `-m` : 如果调用到本地方法的话，可以显示C/C++的堆栈

## 实战分析
线上服务处于假死状态：使用 `jps` 能看到 jvm 进程，但是接口请求都是超时：
<center><img src="pics/zombie.png" alt=""></center>

+ `jmap -dump:format=b,file=heapdump.hprof 7` dump 了堆
+ 使用 `jstack -l 7` dump 了[线程快照](jstack_l.md)

> https://www.joyk.com/dig/detail/1651125444231517#gsc.tab=0
