# JVM、JRE和JDK之前的区别
{docsify-updated}

## JVM
Java 虚拟机（JVM）是执行 Java 程序的虚拟机的一种实现。JVM 首先解释字节码。然后，它将类信息存储在内存区域。最后，它执行 java 编译器生成的字节码。它是一个抽象的计算机器，有自己的指令集，并在运行时操作各种内存区域。

JVM 的组件包括：
+ 类加载器
+ 运行时数据区
+ 执行引擎

```
JVM
├── ClassLoader
├── Runtime Data Areas
│   ├── Heap
│   ├── Stack
│   ├── Metaspace
│   └── Native Method Stack
├── Execution Engine
│   ├── Interpreter - 解释器
│   ├── JIT Compiler (C1 / C2) - JIT运行时编译器
│   └── GC
├── Thread & Safepoint
└── JNI / Native Interface
```

类加载器和运行时数据区在后文中会介绍，这里介绍下执行引擎。


### 执行引擎
执行引擎利用内存区域中的信息执行指令。它包括三个部分：

+ 解释器

	一旦类加载器加载并验证字节码，解释器就会逐行执行字节码。这种执行速度相当慢。解释器的缺点是，当一个方法被多次调用时，每次都需要进行新的解释。  
	不过，JVM 使用 JIT 编译器来减轻这一缺点。

+ 即时（JIT）编译器

	JIT 编译器在运行时将常调用方法的字节码编译成本地代码。因此，它负责优化 Java 程序。
	JVM 会自动监控正在执行的方法。一旦某个方法符合 JIT 编译条件，它就会被安排编译成机器代码。这种方法被称为热方法。编译成机器代码的过程在单独的 JVM 线程上进行。  
	因此，它不会中断当前程序的执行。编译成机器代码后，运行速度会更快。

+ 垃圾收集器

	Java 使用垃圾回收来管理内存。这是一个查看堆内存、识别哪些对象在使用、哪些不在使用并最终删除未使用对象的过程。
	GC 是一个守护线程。它可以使用 System.gc() 方法显式调用，但不会立即执行，而是由 JVM 决定何时调用 GC。

## JRE
Java 运行时环境 (JRE) 是用于运行 Java 应用程序的软件组件包。

JRE 的核心组件包括:
+ Java 虚拟机（JVM）的实现
+ 运行 Java 程序所需的类
+ 属性文件

我们在上一节讨论了 JVM。下面我们将重点讨论核心类和支持文件。

### 引导类
我们可以在 jre/lib/ 下找到引导类。该路径也称为引导路径。它包括
+ `rt.jar` 中的运行时类
+ `i18n.jar` 中的国际化类
+ `charsets.jar`中的字符转换类
+ 其他类

Bootstrap ClassLoader 会在 JVM 启动时加载这些类。

### 扩展类
我们可以在 jre/lib/extn/ 中找到扩展类，它是 Java 平台的扩展目录。该路径也称为扩展类路径。

它包含 `jfxrt.jar` 中的 JavaFX 运行时库，以及 `localedata.jar` 中的 `java.text` 和 `java.util` 包的本地数据。用户还可以在此目录中添加自定义 jar。

### 属性设置
Java 平台使用这些属性设置来维护其配置。根据使用情况，它们位于 /jre/lib/ 内的不同文件夹中。其中包括
+ `calendar.properties` 中的日历配置
+ `logging.properties` 中的日志配置
+ `net.properties` 中的网络配置
+ `/jre/lib/deploy/` 中的部署属性
+ `/jre/lib/management/` 中的管理属性

### 其他文件
除了上述文件和类之外，JRE 还包含用于其他事项的文件：
+ 安全管理，目录为 jre/lib/security
+ 用于放置小程序支持类的目录：jre/lib/applet
+ 与字体相关的文件，位于 jre/lib/fonts 及其他

## JDK
Java 开发工具包（JDK）提供了开发、编译、调试和执行 Java 程序的环境和工具。JDK 的核心组件包括:
+ JRE
+ 开发工具

我们在上一节讨论了 JRE。现在，我们将重点讨论各种开发工具。让我们根据用途对这些工具进行分类

### 基础工具
这些工具奠定了 JDK 的基础，用于创建和构建 Java 应用程序。在这些工具中，我们可以找到用于编译、调试、归档、生成 Javadocs 等的实用程序。它们包括：
+ `javac` - 读取类和接口定义并将其编译成类文件
+ `java` - 启动 Java 应用程序
+ `javadoc` - 从 Java 源文件生成 HTML 页面的 API 文档
+ `apt` - 根据指定源文件集中的注释查找并执行注释处理器
+ `appletviewer` - 使我们无需网络浏览器即可运行 Java 小程序
+ `jar` - 将 Java 小程序或应用程序打包成一个单独的归档文件
+ `jdb` - 命令行调试工具，用于查找和修复 Java 应用程序中的错误
+ `javah` - 从 Java 类生成 C 头文件和源文件
+ `javap` - 反汇编类文件并显示类文件中的字段、构造函数和方法信息
+ `extcheck` - 检测目标 Java 档案 (JAR) 文件与当前安装的扩展 JAR 文件之间的版本冲突

### 安全工具
这些工具包括用于操作 Java 密钥存储的密钥和证书管理工具。Java Keystore 是授权证书或公钥证书的容器。因此，基于 Java 的应用程序经常使用它来进行加密、身份验证和通过 HTTPS 提供服务。

此外，它们还有助于在我们的系统上设置安全策略，并创建可在生产环境中根据这些策略工作的应用程序。其中包括
+ `keytool` - 帮助管理密钥库条目，即加密密钥和证书
+ `jarsigner` - 使用密钥库信息生成数字签名的 JAR 文件
+ `policytool` - 使我们能够管理定义安装安全策略的外部策略配置文件

一些安全工具还有助于管理 Kerberos tickets。 Kerberos 是一种网络身份验证协议。它以票据为基础，允许通过非安全网络通信的节点以安全的方式相互证明身份：
+ `kinit` - 用于获取和缓存 Kerberos 票证授予票证
+ `ktab` - 管理密钥表中的原则名称和密钥对
+ `klist` - 显示本地凭证缓存和密钥表中的条目

### 国际化工具
国际化是指在设计应用程序时，使其能够在不进行工程改动的情况下适用于各种语言和地区。  
为此，JDK 提供了 `native2ascii` 。该工具可将包含 JRE 支持的字符的文件转换为以 ASCII 或 Unicode 转义码编码的文件。

### 远程方法调用 (RMI) 工具
RMI 工具可实现 Java 应用程序之间的远程通信，从而为开发分布式应用程序提供了空间。RMI 使运行在一个 JVM 中的对象能够调用运行在另一个 JVM 中的对象的方法。这些工具包括:
+ `rmic` - 使用 Java 远程方法协议 (JRMP) 或 Internet 交互协议 (IIOP) 为远程对象生成存根、骨架和绑定类
+ `rmiregistry` - 创建并启动远程对象注册表
+ `rmid` - 启动激活系统守护进程。这允许在 Java 虚拟机中注册和激活对象
+ `serialver` - 返回指定类的序列版本 UID

### Java IDL 和 RMI-IIOP 工具
Java 接口定义语言（IDL）为 Java 平台增加了基于通用对象的请求代理架构（CORBA）功能。这些工具使分布式 Java 网络应用程序能够使用行业标准对象管理集团（OMG）- IDL 在远程网络服务上调用操作。
同样，我们也可以使用 Internet InterORB 协议（IIOP）。RMI-IIOP，即 IIOP 上的 RMI，可以通过 RMI API 对 CORBA 服务器和应用程序进行编程。这样，两个用任何 CORBA 兼容语言编写的应用程序之间就可以通过 Internet InterORB 协议 (IIOP) 进行连接。这些工具包括:
+ `tnameserv` - 瞬时命名服务，为对象引用提供树形结构目录
+ `idlj` - IDL 到 Java 编译器，用于为指定的 IDL 文件生成 Java 绑定
+ `orbd`--使客户端能够在 CORBA 环境中透明地定位和调用服务器上的持久对象
+ `servertool` - 提供命令行界面，用于在 ORB 守护进程（orbd）中注册或取消注册持久化服务器，启动或关闭在 ORB 守护进程中注册的持久化服务器等。

### Java 部署工具
这些工具有助于在网络上部署 Java 应用程序和小程序。它们包括：
+ `pack200` - 使用 Java gzip 压缩器将 JAR 文件转换为 pack200 文件
+ `unpack200` - 将 pack200 文件转换为 JAR 文件

### Java 插件工具
JDK 为我们提供了 `htmlconverter` 。此外，它还与 Java Plug-in 结合使用。  
一方面，Java 插件在常用浏览器和 Java 平台之间建立了连接。通过这种连接，网站上的小程序可以在浏览器中运行。  
另一方面， `htmlconverter` 是一种将包含小程序的 HTML 页面转换为 Java Plug-in 格式的实用程序。

### Java 网络启动工具
JDK 带来了 `javaws` 。我们可以将其与 Java Web Start 结合使用。
通过该工具，我们只需在浏览器中点击一下，就能下载并启动 Java 应用程序。因此，无需运行任何安装程序。

### 监控和管理工具
我们可以使用这些出色的工具来监控 JVM 性能和资源消耗。以下是其中几种：
+ `jconsole` - 提供图形控制台，让您监控和管理 Java 应用程序
+ `jps` - 列出目标系统上的工具 JVM
+ `jstat` - 监控 JVM 统计数据
+ `jstatd` - 监控被检测 JVM 的创建和终止

### 故障排除工具
这些是我们可以用于故障排除任务的实验工具：
+ `jinfo` - 生成指定 Java 进程的配置信息
+ `jmap` - 打印指定进程的共享对象内存映射或堆内存详细信息
+ `jsadebugd` - 附加到 Java 进程并充当调试服务器
+ `jstack` - 打印指定 Java 进程的 Java 线程的 Java 堆栈跟踪

## 常用JDK
<center><img src="pics/jdk.png" width="70%"></center>


## java 版本的关系
当我们使用：`javap -verbose -classpath cap-api-1.0.0.jar com.demo.gjyw.ErrorCode | grep major` 可以查看一个 class 文件的版本信息，比如：
```
javap -verbose ErrorCode.class | grep version
minor version: 0
major version: 55
```

class 文件里的 major version 指的是： **该 class 文件遵循的 Java Class File Format 版本**，也就是 JVM 能否加载它的最低要求。
它不是 JDK 版本号
它不是 Java 语言版本号
它直接约束 JVM（HotSpot 等）的兼容性

```
Java 语言规范
      ↓
javac 编译器
      ↓
class 文件格式（major/minor）
      ↓
JVM（class loader 校验）
```
major version 属于 class 文件格式规范（JVMS）。

为什么大家会把它和 JDK 混在一起？ 因为 JDK 版本 与 class major version 是“一一对应发布”的：
| Major Version | JDK Version |
|---------------|--------------|
| 52            | Java 8       |
| 53            | Java 9       |
| 54            | Java 10      |
| 55            | Java 11      |
| 56            | Java 12      |
| 57            | Java 13      |
| 58            | Java 14      |
| 59            | Java 15      |
| 60            | Java 16      |
| 61            | Java 17      |
| 62            | Java 18      |
| 63            | Java 19      |
| 64            | Java 20      |
| 65            | Java 21      |

## macos 编译 JDK 
1. `gcl https://github.com/openjdk/jdk.git`
2. `bash configure` 根据报错提示安装以下工具
   1. 安装 xcode
   2. jdk 版本 26+
   3. `xcodebuild -downloadComponent MetalToolchain` 安装 metal
3. `make images`
4. `./build/*/images/jdk/bin/java -version` 查看编译好的 jvm 版本