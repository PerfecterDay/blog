# Java应用程序的打包
{docsify-updated}

- [Java应用程序的打包](#java应用程序的打包)
	- [jar 命令详解](#jar-命令详解)
	- [清单文件](#清单文件)
	- [可执行jar](#可执行jar)
	- [资源文件](#资源文件)
	- [Multi-Release JAR](#multi-release-jar)

在将应用程序进行打包时，使用者一定希望仅提供给其一个单独的文件，而不是一个含有大量类文件的目录，Java归档（JAR）文件就是为此目的而设计的。一个JAR文件既可以包含类文件，也可以包含诸如图像和声音这些其他类型的文件。此外，JAR文件是压缩的，它使用了大家熟悉的ZIP压缩格式。

## jar 命令详解
jar 是 JDK 自带的工具，依赖于 JDK 的 tools.jar 包。
+ -c  创建新档案
+ -t  列出档案目录
+ -x  从档案中提取指定的 (或所有) 文件
+ -u  更新现有档案
+ -v  在标准输出中生成详细输出
+ -f  指定档案文件名
+ -m  包含指定清单文件中的清单信息
+ -n  创建新档案后执行 Pack200 规范化
+ -e  为捆绑到可执行 jar 文件的独立应用程序指定应用程序入口点
+ -0  仅存储; 不使用任何 ZIP 压缩
+ -P  保留文件名中的前导 '/' (绝对路径) 和 ".." (父目录) 组件
+ -M  不创建条目的清单文件
+ -i  为指定的 jar 文件生成索引信息
+ -C  更改为指定的目录并包含以下文件
示例：
1. jar -cf test.jar test: 将test路径下的全部内容打包成 test.jar
2. jar -cvf test.jar test: 同上，但是会显示打包过程
3. jar -cvfM test.jar test: 同上，不生成清单文件
4. jar -cvfm test.jar manifest.mf test: 同上，使用指定的清单文件
5. jar tvf test.jar: 查看指定 jar 的文件列表的详细信息
6. jar -xvf test.jar: 解压缩指定 jar 文件

## 清单文件
除了类文件、图像和其他资源外，每个JAR文件还包含一个用于描述归档特征的清单文件（manifest）。
清单文件被命名为MANIFEST.MF，它位于JAR文件的一个特殊META-INF子目录中。最小的符合标准的清单文件是很简单的：
    Manifest-Version: 1.0

复杂的清单文件可能包含更多条目。这些清单条目被分成多个节。第一节被称为**主节（main section）**。它作用于整个JAR文件。随后的条目用来指定已命名条目的属性，这些已命名的条目可以是某个文件、包或者URL。它们都必须起始于名为Name的条目。节与节之间用空行分开。例如：
```
Manifest-Version: 1.0
Built-By: BaIcy
Created-By: Apache Maven 3.8.1
Build-Jdk: 1.8.0_292
其它描述这个归档文件的行

Name:Wowo.class
描述这个文件的行
Name:com/mycompany/mypkg
描述这个包的行

Main-Class:com.mycompany.mypkg.MainClass
```

要想编辑清单文件，需要将希望添加到清单文件中的行放到文本文件中，然后运行：
`jar cfm JarFileName ManifestFileName`

要想更新一个已有的JAR文件的清单，则需要将增加的部分放置到一个文本文件中，然后执行下列命令：
`jar ufm MyArchive.jar manifest-addition.mf`

## 可执行jar
可以使用jar命令中的e选项指定程序的入口点，即通常需要在调用java程序加载器时指定的类：
`java cvfe MyArchive.jar com.mycompany.mypkg.MainClass files to add`
或者，可以在清单文件中指定应用程序的主类，包括以下形式的语句：
`Main-Class:com.mycompany.mypkg.MainClass`

不论哪一种方法，用户可以简单地通过下面命令来启动应用程序： `java -jar MyArchive.jar`。
否则，如果打包时没有指定主类，运行上述命令会报错：`java_in_action-1.0-SNAPSHOT.jar中没有主清单属性`

## 资源文件
类加载器知道如何搜索类文件，直到在类路径、存档文件或web服务器上找到为止。利用资源机制，对于非类文件也可以同样方便地进行操作。下面是必要的步骤：
1. 获得具有资源的Class对象，例如，AboutPanel.class。
2. 如果资源是一个图像或声音文件，那么就需要调用 getResource（filename）获得作为URL的资源位置，然后利用getImage或getAudioClip方法进行读取。
3. 与图像或声音文件不同，其他资源可以使用getResourceAsStream方法读取文件中的数据。  
重点在于类加载器可以记住如何定位类，然后在同一位置查找关联的资源。例如，要想利用about.gif图像文件制作图标，可以使用下列代码：
```
URL url = ResourcsTest.class.getResource("about.gif");
Image img = new ImageIcon(url).getImage();
```
这段代码的含义是“在找到ResourceTest类的地方查找about.gif文件”。  
除了可以将资源文件与类文件放在同一个目录中外，还可以将它放在子目录中。可以使用下面所示的层级资源名：
+ 相对路径资源名，它会被解释为相对于加载这个资源的类所在的包： `data/text/about.txt`
+ 绝对路径资源名，它的定位方式与类在包中的定位方式一样，在 classpath 下的路径名下查找： `/java/title.txt`


## Multi-Release JAR
使用场景：你写了一个工具 jar 包，其中有一个 `Hello` 类，你希望在不同的 JDK 下使用不同的 JDK API 来实现某个功能。  
核心问题只有一个：
在不同 JDK 版本下，用“同一个 class 名”，提供“不同实现”。 JVM 自动选择, 对使用者完全透明。
