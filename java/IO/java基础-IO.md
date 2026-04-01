# java基础-IO
{docsify-updated}

- [java基础-IO](#java基础-io)
	- [IO 流](#io-流)
		- [输入输出流体系](#输入输出流体系)
			- [InputStream 和 Reader](#inputstream-和-reader)
			- [OutputStream 和 Writer](#outputstream-和-writer)
			- [文本输出](#文本输出)
			- [读写二进制数据(Datalnput和DataOutput和)](#读写二进制数据datalnput和dataoutput和)
			- [标准输入输出](#标准输入输出)
			- [JVM读取其它进程的数据](#jvm读取其它进程的数据)
			- [RandomAccessFile](#randomaccessfile)
	- [操作文件](#操作文件)
		- [老得File类及常见方法](#老得file类及常见方法)
		- [新的 Path 和 Files 类](#新的-path-和-files-类)
			- [Path](#path)
			- [Files](#files)
			- [内存映射文件](#内存映射文件)
			- [文件加锁](#文件加锁)


在 Java API 中 ，可以从其中读入一个字节序列的对象称做输入流，而可以向其中写人一个字节序列的对象称做输出流。这些字节序列的来源地和目的地可以是文件，而且通常都是文件，但是也可以是网络连接，甚至是内存块。 抽象类 `InputStream` 和 `OutputStream` 构成了输入/输出( I/O)类层次结构的基础 。

因为面向字节的流不便于处理以 Unicode形式存储的信息(回忆一下， Unicode中每个 字符都使用了多个字节来表示)，所以从抽象类 `Reader` 和 `Writer` 中继承出来了一个专门用于处理 Unicode 字符的单独的类层次结构 。 这些类拥有的读人和写出操作都是基于两字节的 Char值的(即， Unicode码元)，而不是基于 byte值的。

## IO 流
流按照不同的方式可以分为如下几类：
+ 输入流：从外设/网络等流入程序内存的流，只能从其中读数据，不能写
+ 输出流：从程序内存输出到外设/网络等的流，只能向其中写数据，不能读
+ 节点流：可以直接从/向一个特定的IO设备（磁盘、网络等）读写数据的流，称为节点流，也称为低级流
+ 处理流：对于一个已经存在的流进行连接或封装，通过封装后流来实现数据读写的功能，也称为高级流；关闭最上层的处理流，系统会自动关闭该处理流包装的节点流

<center><img src="pics/stream-2.png" width=20% /></center> 

字节流/字符流：两种流操作的数据单元不同，字节流按字节处理数据，而字符流按16位字符处理数据
<center><img src="pics/stream-1.png" width=60%/></center> 

### 输入输出流体系
<center><img src="pics/stream-framework.png" width=60%></center> 
粗体标出的类代表节点流，必须直接与指定的物理节点关联；斜体标出的类代表抽象基类，无法直接创建实例。  
访问数组和字符串的流看似没什么用，因为数组和字符串本来就可以在内存中直接访问，为啥还要使用流来读写呢？当你的API接口只接受IO流参数，但是你又想用字符串或数组去调用这个API时，就可以使用这种流来调用。

#### InputStream 和 Reader
InputStream/Reader 里包含以下三个读方法：  
+ `int read()`: 从输入流中读取一个字节，返回读取到的字节数据
+ `int read(byte[]/char[] b)`: 从输入流中读取数据到字节/字符数组b中，最多读取b.length个字节，返回实际读取的字节/字符数
+ `int read(byte[] b, int off, int len)`: 从输入流中最多读取len个字节/字符存入字节/字符数组b中，从off位置开始存放，返回实际读取的字节/字符数
除此之外，它们还支持如下几个方法来移动记录指针：
+ `boolean markSupported()`: 判断流是否支持记录标记
+ `synchronized void mark(int readlimit)`: 在记录指针（读取位置的记录，读取时会从这个位置开始读）当前位置记录一个标记
+ `synchronized void reset()`: 将记录指针重新定位到上一个记录标记的位置
+ `long skip(long n)`: 将记录指针向前移动 n 个字节/字符

#### OutputStream 和 Writer
OutputStream/Writer 也提供如下三个写方法：
+ `void write(int b)`: 将字节/字符 b写入到流中
+ `void write(byte[]/char[] buf)`: 将字节/字符数组buf中的数据写入输出流中
+ `void write(byte[]/char[] buf, int off, int len)`: 将字节/字符数组buf中从off位置开始长度为len的数据写入输出流中
因为字符流直接以字符为操作单位，因此 Writer 也支持直接写字符串：
+ `void write(String str)`: 将字符串 str 中的字符写入输出流中
+ `void write(String str,int off, int len)`: 将字符串 str 中从 off 位置开始，长度为len的字符写入输出流中

`read` 和 `write` 方法在执行时都将阻塞，直至字节确实被读入或写出 。 这就意味着如果流不能被立即访问(通常是因为 网络连接忙)，那么当前的线程将被阻塞 。 这使得在这两个方 法等待指定的流变为可用的这段时间里，其他的线程就有机会去执行有用的工作。

当你完成对输入/输出流的读写时，应该通过调用 close 方法来关闭它，这个调用会释 放掉十分有限的操作系统资源 。 如果一个应用程序打开了过多的输入/输出流而没有关闭， 那么系统资源将被耗尽 。 关闭一个输出流的同时还会冲刷用于该输出流的缓冲区 : 所有被临 时置于缓冲区中，以便用更大的包的形式传递的字节在关闭输出流时都将被送出。 特别是， 如果不关闭文件，那么写出字节 的最后一个包可能将永远也得不到传递 。 当然，我们还可以 用 flush方法来人为地冲刷这些输出。

#### 文本输出
对于文本输出，可以使用 `PrintWriter` 。 这个类拥有以文本格式打印字符串和数字的方法，它还有一个将 `PrintWriter` 链接到 `FileWriter` 的便捷方法，下面的语句 : 
```PrintWriter out = new PrintWriter(”employee.txt”，”UTF司8”);```
等同于 :
```
PrintWriter out = new PrintWriter(
new FileOutputStream (”employee.txt”),"UTF-8");
```
为了输出到打印写出器，需要使用与使用 `System.out` 时相同的 `print`、 `println` 和 `printf` 方法。 你可以用这些方法来打印数字(`int`、 `short`、 `long`、 `float` 、 `double`)、字符、 boolean值、字符串和对象。例如，考虑下面的代码:
```
String name = ”Harry Hacker”; 
double salary = 75000;
out.print (name) ;
out.print(’ ’); 
out.println(salary) ;
```

#### 读写二进制数据(Datalnput和DataOutput和)
为了读回数据，可以使用在 DataInput 接口中定义的下列方法 :
+ readint 
+ readShort 
+ readlong 
+ readFloat 
+ readDouble 
+ readChar 
+ readBoolean 
+ readUTF

DataOutput 接口定义了下面用于以二进制格式写数组、字符、boolean 值和字符串的方法:
+ writeChars
+ writeByte
+ writeInt
+ writeShort
+ writeLong
+ writeFloat
+ writeDouble
+ writeChar
+ writeBoolean
+ writeUTF

`DataInputStream` 类实现了 `DataInput` 接口，为了从文件中读入二进制数据，可以将 `DataInputStream` 与某个字节源相组合，例如 `FileInputStream`:
```DataInputStream in = new DataInputStream(new FileInputStream("employee.dat"));```
与此类似，要想写出二进制数据，你可以使用实现了 `DataOutput` 接口的 `DataOutput­Stream` 类 :
```DataOutputStrearn out =new DataOutputStream(new FileOutputStream("employee.dat"));```

#### 标准输入输出
Java的标准输入/输出分别通过 `System.in/System.out` 来代表，默认情况下分别代表键盘和显示器，当程序从 System.in 读数据时，实际上是从键盘读取输入；当程序通过 System.out 来输出数据时，数据会显示在屏幕上。
System类还提供了三个重定向标准输入/输出的方法：
+ `static void setIn(InputStream in)`:  重定向标准输入流
+ `static void setOut(PrintStream out)`: 重定向标准输出流
+ `static void setErr(PrintStream err)`: 重定向标准错误输出流

获取用户输入的几种方法：
1. 使用 BufferedReader 包装标准输入,一行一行的读取：`String line = (new BufferedReader(new InputStreamReader(System.in))).readLine();`
2. 使用 Scanner(System.in)：`Integer input = new Scanner(System.in).nextInt();`

#### JVM读取其它进程的数据
我们知道使用 `Runtime` 类的 `Process exec(String command)` 方法可以启动一个进程，并返沪一个 Process 对象， 该 Process 对象代表由 Java 启动的子进程，Process 提供了三个方法用于程序和其子进程进行通信：
+ `OutputStream getOutputStream()`: 获取子进程的输出流，实际对应子进程的输入流，对本进程来说是输出流，用于向子进程写数据
+ `InputStream getInputStream()`: 获取子进程的输入流，实际上对应子进程的输出流，对本进程来说是输入流，用于读取子进程的输出数据
+ `InputStream getErrorStream()`: 获取子进程的错误输入流，实际上对应子进程的错误输出流，对本进程来说是输入流，用于读取子进程的错误输出数据

#### RandomAccessFile
`RandomAccessFile` 功能非常强大，能够支持随机的读写数据，就是能够随机的从指定位置开始读/写数据，也可以跳过某些数据的读写。`RandomAccessFile`提供两种构造方法来生成对象：
1. RandomAccessFile(File file, String mode)
2. RandomAccessFile(String name, String mode)

mode 参数指定 RandomAccessFile 的访问模式，该参数有如下四个值：
   + "r": 以只读方式打开指定文件
   + "rw": 以读写方式打开指定文件，如果文件不存在，则尝试创建文件
   + "rws": 以读写方式打开指定文件，相对于 "rw"模式，还要求对文件内容或元数据的每个更新都同步写入到底层存储设备。
   + "rwd": 以读写方式打开指定文件，相对于 "rw"模式，还要求对文件内容的每个更新都同步写入到底层存储设备。

打开一个 `RandomAccessFile` 对象后，可以使用下述两个方法来操纵文件记录指针：
   + `long getFilePointer()`: 获取文件记录指针的位置
   + `void seek(long pos)`: 将文件记录指针定位到指定的 pos 位置。

RandomAccessFile 还包含类似于了 InputStream/OutputStream 中的三种 read()/write() 方法，用法是完全一样的，另外还包含了系列方便的 readXXX 和 writeXXX 方法。


## 操作文件

### 老得File类及常见方法
`File` 类看上去是指代文件，其实它既能代表一个特定文件，也能代表一个目录。

+ `String[] list()`: 返回所有文件名的字符串数组
+ `String[] list(FileNameFilter filter)`: 返回 `FileNameFilter` 过滤后的字符串数组
+ `File[] listFiles()`: 返回所有的文件数组
+ `File[] listFiles(FilenameFilter filter)`: 返回 `FilenameFilter` 过滤后的文件数组
+ `String getAbsolutePath()` :获取绝对路径
+ `String getName()` :获取名字
+ `File getParent()` :获取父目录
+ `long length()` :获取目录/文件大小
+ `long lastModified()` :获取最后修改时间
+ `boolean canExecute()` :是否可执行
+ `boolean canRead()` :是否可读
+ `boolean canWrite()` :是否可写
+ `boolean createNewFile()` :创建新文件当 `File` 代表一个文件时
+ `boolean mkdir()` :创建新目录当 `File` 代表一个目录时

假如 `File` 指代的是一个目录，那么就可以使用 `list()` 方法获取目录下的文件列表，如果想获取所有文件列表，使用不带参数的 `list()` 的方法即可；如果想获得一个受限列表，那么就要使用“目录过滤器”了。 `FilenameFilter` 接口的 `boolean accept(File dir,String name)` 返回 `true` 的才会返回到数组中。 `listFiles` 同理。

```
public class FileList {
    public static void main(String[] args) {****
        File f = new File("/usr/local/etc");
        File[] allFiles = f.listFiles();
        File[] filterFiles = f.listFiles((dir,name)->{
            return name.contains("a"); 
        });
        for (File allFile : allFiles) {
            System.out.print(allFile.getName()+", ");
        }
        System.out.println();
        for (File filterFile : filterFiles) {
            System.out.print(filterFile.getName()+", ");
        }
        System.out.println();
        String[] allNames = f.list();
        String[] filterNames = f.list((dir,name)-> {
                return name.contains(".");
        });
        for (String file : allNames) {
            System.out.print(file+", ");
        }
        System.out.println();
        for (String file : filterNames) {
            System.out.print(file+", ");
        }
    }
}
```

### 新的 Path 和 Files 类
`Path` 和 `Files` 类封装了在用户机器上处理文件系统所需的所有功能。

#### Path
Path 表示的是一个目录名序列，其后还可以跟着一个文件名 。 路径中的第一个部件可以是根部件，例如`/`或 `C:\`，而允许访问的根部件取决于文件系统。 以根部件开始的路径是绝对路径;否则，就是相对路径。

静态的 `Paths.get` 方法接受一个或多个字符串，并将它们用默认文件系统的路径分隔符(类 Unix 文件系统是 `/`, Windows 是`\`)连接起来 。 然后它解析连接起来的结果，如果其表示的不是给定文件系统中的合法路径，那么就抛出 InvalidPathException 异常 。 这个连接起来的结果就是一个 Path 对象。

组合或解析路径是司空见惯的操作,调用 `p.resolve(q)` 将按照下列规则返回一个路径:
+ 如果q是绝对路径， 则结果就是q。
+ 否则，根据文件系统的规则，将“p 后面跟着 q”作为结果 。

Files 类可以使得普通文件操作变得快捷，在创建文件或目录时，可以指定属性，例如文件的拥有者和权限。 但是，指定属性的细节取决于文件系统：
+ static Path createFile(Path path, FileAttribute<?> . . . attrs)
+ static Path createDirectory( Path path, Fil eAttri bute<?> ... attrs)
+ static Path createDirectories(Path path, FileAttribute<?> ... attrs) 还会创建路径中所有的中间目录 。

在适合临时文件的位置，或者在给定的父目录中，创建一个临时文件或目录 。 返回所创建的文件或目录的路径:
+ static Path createTempFile(String prefix, String suffix,FileAttribute<?> ... attrs)
+ static Path createTempFile(Path parentDir, String prefix, Stringsuffix, FileAttribute<?> ... attrs)
+ static Path createTempDirectory(String prefix, FileAttribute<?>... attrs) 
+ + static Path createTempDirectory( Path parentDi r, String prefix ,FileAttribute<?> ... attrs)

#### Files
创建新目录可以调用：
+ Files.createDirectory(path)
其中，路径中除最后一个部件外，其他部分都必须是己存在的 。要创建路径中的中间目录，

创建文件：
Files.createFile(path)
如果文件已经存在了，那么这个调用就会抛出异常。 检查文件是否存在和创建文件是**原子性**的，如果文件不存在，该文件就会被创建，并且其他程序在此过程中是无法执行文件创建操作的 。

复制、移动和删除文件:
+ Files.copy(fromPath,toPath)
+ Files.move(fromPath,toPath)

如果目标路径已经存在，那么复制或移动将失败。 如果想要覆盖已有的目标路径， 可以使用 REPLACE_EXISTING 选项 。如果想要复制所有的文件属性，可以使用 COPY_ ATTRIBUTES 选项：
```
Fi1es.copy(fromPath, toPath, StandardCopyOption.REPLACE_EXISTING, StandardCopyOption.COPY_ATTRIBUTES);
```

还可以将一个输入流复制到 Path 中，这表示你想要将该输入流存储到硬盘上。 类似地，你可以将一个 Path 复制到输出流中:
+ Files.copy(inputSream,toPath)
+ Files.copy(fromPath,outputStream)

删除文件： 
+ Fi1es.de1ete(path) 如果要删除的文件不存在，这个方法就会抛出异常。 
因此，可转而使用下面的方法:
+ boo1ean de1eted = Files.de1etelfExists(path) 该删除方法还可以用来移除空目录。

访问目录中的项:  
静态的 `Files.list` 方法会返回一个可以读取目录中各个项的 Stream<Path>对象 。 目 录是被惰性读取的，这使得处理具有大量项的目录可以变得更高效 。因为读取目录涉及需要关闭的系统资源，所以应该使用 try 块:
```
try (Stream<Path> entries= Files.list(pathToDirectory)){
	...
}
```

#### 内存映射文件
1. 首先，从文件中获得一个通道( channel)，通道是用于磁盘文件的一种抽象，它使我们可 以访问诸如内存映射、文件加锁机制以及文件间快速数据传递等操作系统特性。
```FileChannel channel = FileChannel.open(path,options);```
2. 然后，通过调用 `FileChannel` 类的 `map` 方法从这个通道中获得一个 `ByteBuffer`。 你可以指定想要映射的文件区域与映射模式，支持的模式有三种 :
	+ `FileChannel.MapMode.READ_ONLY`: 所产生的缓冲区是只读的，任何对该缓冲区写入的尝试都会导致 `ReadOnlyBufferException` 异常。
	+ `FileChannel.MapMode.READ_WRITE` :所产生的缓冲区是可写的，任何修改都会在某个时刻写回到文件中 。 注意，其他映射同一个文件的程序可能不能立即看到这些修改，多个程序同时进行文件映射的确切行为是依赖于操作系统的 。
	+ `FileChannel.MapMode.PRIVATE` :所产生的缓冲区是可写的，但是任何修改对这个缓冲区来说都是私有的，不会传播到文件中 。
3. 一旦有了缓冲区，就可以使用 `ByteBuffer` 类和 `Buffer` 超类的方法读写数据了。

#### 文件加锁
```
FileChannel =FileChannel.open(path); 
try(Filelock lock= channel .lock()){
	....
}

```