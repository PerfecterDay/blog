# Java基础-字符串类
{docsify-updated}

## 定义多行字符串
Java 15 开始，我们可以用`""""`（三个双引号）来声明字符串，从而使用文本块。(三个双引号）来声明字符串，就可以使用文本块：
```
public String textBlocks() {
    return """
        Get busy living
        or
        get busy dying.
        --Stephen King""";
}
```

## 获取系统行分隔符
每个操作系统都有自己定义和识别新行的方法。
在 Java 中，获取操作系统的行分隔符非常容易：
```
String newLine = System.getProperty("line.separator");
```

## String.join
```
public String stringJoin() {
    return String.join(newLine,
                       "Get busy living",
                       "or",
                       "get busy dying.",
                       "--Stephen King");
}
```
## StringWriter
`StringWriter` 是另一种我们可以用来创建多行字符串的方法。在这里我们不需要 newLine，因为我们使用的是 `PrintWriter` :
```
public String stringWriter() {
    StringWriter stringWriter = new StringWriter();
    PrintWriter printWriter = new PrintWriter(stringWriter);
    printWriter.println("Get busy living");
    printWriter.println("or");
    printWriter.println("get busy dying.");
    printWriter.println("--Stephen King");
    return stringWriter.toString();
}
```

## 去除字符串最后一个字符
```
StringUtils.chop(TEST_STRING); //  Apache Commons Lang3

public static String removeLastCharOptional(String s) {
	return Optional.ofNullable(s)
		.filter(str -> str.length() != 0)
		.map(str -> str.substring(0, str.length() - 1))
		.orElse(s);
}
```

## 统计字符串里字符的数量
```
int count = StringUtils.countMatches("elephant", "e"); //  Apache Commons Lang3
long count = someString.chars().filter(ch -> ch == 'e').count();
```

## split
```
String[] splitted = StringUtils.split("car jeep scooter"); //  Apache Commons Lang3
String[] splitted = "peter,james,thomas".split(",");
```

## 字符串压缩
在JDK早期版本中，字符串在内部由包含字符串字符的 `char[]` 表示。每个字符由 2 个字节组成，因为 Java 内部使用 UTF-16。

例如，如果字符串包含一个英语单词，那么每个字符的前 8 位都将为 0，因为 ASCII 字符可以用一个字节来表示。

许多字符需要 16 位来表示，但根据统计，大多数字符只需要 8 位--LATIN-1 字符表示法。因此，内存消耗和性能都有提高的空间。

同样重要的是，字符串通常占据了 JVM 堆空间的很大一部分。而且，由于 JVM 的存储方式，在大多数情况下，字符串实例可能会占用实际需要的双倍空间。

JDK6 中引入的压缩字符串选项和 JDK9 中引入的新的紧凑字符串。这两个选项都是为了优化 JMV 上字符串的内存消耗而设计的。

### Java6 的 Compressed String
```
-XX:+UseCompressedStrings
```
启用该选项后，字符串将存储为 `byte[]`，而不是 `char[]`，从而节省大量内存。不过，该选项最终在 JDK 7 中被删除，主要是因为它带来了一些意想不到的性能后果。

### Java 9 的 Compact String
Java 9 恢复了紧凑字符串的概念。

这意味着，当我们创建一个字符串时，如果该字符串的所有字符都可以用字节 - LATIN-1 表示，那么内部将使用一个字节数组，这样一个字节表示一个字符。

在其他情况下，如果任何字符需要超过 8 位才能表示，那么所有字符都将使用两个字节存储，每个字节使用 UTF-16 表示法。

因此，基本上只要有可能，每个字符都会使用一个字节。

现在的问题是，所有字符串操作将如何工作？它将如何区分 LATIN-1 和 UTF-16 表示法？

为了解决这个问题，我们对字符串的内部实现进行了另一项修改。字符串有一个有一个最终字段`coder`，可以保留这些信息。
```
static final byte LATIN1 = 0;
static final byte UTF16 = 1;
```

如果要关闭 compactString,，可以加上启动参数: `+XX:-CompactStrings`

## 字符串比较
```
String string1 = "using equals method";
String string2 = "using equals method";
        
String string3 = "using EQUALS method";
String string4 = new String("using equals method");

assertThat(string1.equals(string2)).isTrue();
assertThat(string1.equals(string4)).isTrue();

assertThat(string1.equals(null)).isFalse();
assertThat(string1.equals(string3)).isFalse();

String string1 = "using equals ignore case";
String string2 = "USING EQUALS IGNORE CASE";

assertThat(string1.equalsIgnoreCase(string2)).isTrue();
```

## 换行符
`\r` 和 `\n` 是 ASCII 值分别为 13 (CR) 和 10 (LF) 的字符。它们都表示两行之间的分隔符，但操作系统使用它们的方式不同。

在 Windows 系统中，两个字符的序列用于开始新行，CR 紧接着 LF。相反，在类 Unix 系统中，只使用 LF。

在编写 Java 应用程序时，我们必须注意所使用的换行符，因为不同操作系统上运行的应用程序会有不同的表现。

最安全、最兼容的方法是使用 `System.lineSeparator()` 或者 `System.getProperty("line.separator")`。这样，我们就不必考虑操作系统。

## 字符串池
由于字符串在 Java 中具有不变性，JVM 可以通过在池中只存储每个字面字符串的一个副本来优化为字符串分配的内存量。这个过程被称为 "插值"（interning）。

当我们创建一个字符串变量并为其赋字面值时，JVM 会在池中搜索等值的字符串。如果找到，Java 编译器将直接返回其内存地址的引用，而不会分配额外的内存。如果未找到，则会将其添加到池中（内部化），并返回其引用。

当我们使用 new() 操作符创建字符串对象时，它总是在堆内存中创建一个新对象。另一方面，如果我们使用字符串字面值语法（如 "Baeldung"）创建对象，如果该对象已经存在，它可能会从字符串池中返回一个现有对象。否则，它将创建一个新的 String 对象，并将其放入字符串池供将来重复使用。

从高层来看，两者都是字符串对象，但主要区别在于 new() 操作符总是创建一个新的字符串对象。此外，当我们使用 literal 创建字符串时，它是被内部化的。

在 Java 7 之前，JVM 将 Java 字符串池放在 PermGen 空间中，该空间有固定大小，不能在运行时扩展，也不能进行垃圾回收。在 PermGen（而不是 Heap）中置入字符串的风险在于，如果置入的字符串过多，JVM 就会出现 OutOfMemory 错误。

从 Java 7 开始，Java 字符串池存储在堆空间，由 JVM 进行垃圾回收。这种方法的优点是降低了发生 OutOfMemory 错误的风险，因为未引用的字符串将从池中删除，从而释放了内存。

## String 的API
+ String.String()
+ String.codePointCount()
+ String.codePointAt()
+ String.concat()
+ String.contains()
+ String.copyValueOf()
+ String.endsWith()
+ String.format()
+ String.getBytes()
+ String.indexOf()
+ String.intern()
+ String.isEmpty()
+ String.lastIndexOf()
+ String.regionMatches()
+ String.replace()
+ String.replaceAll()
+ String.split()
+ String.startsWith()
+ String.subSequence()
+ String.substring()
+ String.toLowerCase()
+ String.toUpperCase()
+ String.trim()
+ String.valueOf()