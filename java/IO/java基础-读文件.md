# Java基础-读文件
{docsify-updated}

- [Java基础-读文件](#java基础-读文件)
  - [使用`InputStream`](#使用inputstream)
  - [使用 `BufferedReader`](#使用-bufferedreader)
  - [使用NIO的 `Files` 方法: `readAllLines`,`readAllBytes`,`newBufferedReader`,`lines`](#使用nio的-files-方法-readalllinesreadallbytesnewbufferedreaderlines)
  - [使用 `Scanner`](#使用-scanner)
  - [使用 `StreamTokenizer`](#使用-streamtokenizer)
  - [使用 `DataInputStream`](#使用-datainputstream)
  - [使用 `FileChannel`](#使用-filechannel)


###  使用`InputStream`  
如果我们能获取到文件的 `InputStream` 流，就能使用下述方法读取文件内容：
```
private String readFromInputStream(InputStream inputStream)
throws IOException {
    StringBuilder resultStringBuilder = new StringBuilder();
    try (BufferedReader br
    = new BufferedReader(new InputStreamReader(inputStream))) {
        String line;
        while ((line = br.readLine()) != null) {
            resultStringBuilder.append(line).append("\n");
        }
    }
return resultStringBuilder.toString();
}
```
所以，关键的问题是如何获取到文件的 `InputStream` 流：
使用 `Class/Classloader` 的 `getResourceAsStream(String resourceName)`
```
Class clazz = FileOperationsTest.class;
InputStream inputStream = clazz.getResourceAsStream("/fileTest.txt");
String data = readFromInputStream(inputStream);

ClassLoader classLoader = this.getClass().getClassLoader();
InputStream inputStream = classLoader.getResourceAsStream("fileTest.txt");
String data = readFromInputStream(inputStream);
```
    
### 使用 `BufferedReader`
```
String expected_value = "Hello, world!";
String file ="src/test/resources/fileTest.txt";

BufferedReader reader = new BufferedReader(new FileReader(file));
char[] buf = new char[10];
while (bufferedReader.read(buf) > 0){
    System.out.print(buf);
}
String currentLine = reader.readLine(); // 按行读取
reader.close();
```

### 使用NIO的 `Files` 方法: `readAllLines`,`readAllBytes`,`newBufferedReader`,`lines`
```
String expected_value = "Hello, world!";
Path path = Paths.get("src/test/resources/fileTest.txt");
List<String> read = Files.readAllLines(path);  // 读取所有行，每行一个元素放入 list 
byte[] bytes = Files.readAllBytes(path); //二进制数据
assertEquals(expected_value, read);

BufferedReader reader = Files.newBufferedReader(path); //获取到 BufferedReader，然后读取
String line = reader.readLine();

Stream<String> lines = Files.lines(path);
String data = lines.collect(Collectors.joining("\n"));
lines.close();

```

### 使用 `Scanner`
```
@Test
public void whenReadWithScanner_thenCorrect()
  throws IOException {
    String file = "src/test/resources/fileTest.txt";
    Scanner scanner = new Scanner(file);
    scanner.useDelimiter("\n");
    while (scanner.hasNext()){
        String next = scanner.next();
        System.out.println(next);
    }
    scanner.close();
}
```
请注意，默认分隔符是空格，但`Scanner`可以使用多个分隔符。  
`Scanner` 类适用于从控制台读取内容，或当内容包含已知分隔符的原始值时（例如：用空格分隔的整数列表）。

### 使用 `StreamTokenizer`
```
StreamTokenizer tokenizer = new StreamTokenizer(Files.newBufferedReader(path));
while (tokenizer.nextToken() !=  StreamTokenizer.TT_EOF){
    if (tokenizer.ttype==StreamTokenizer.TT_WORD){
        System.out.printf(tokenizer.sval + " ");
    }else if (tokenizer.ttype==StreamTokenizer.TT_NUMBER){
        System.out.printf(String.valueOf(tokenizer.nval+" "));
    } else if (tokenizer.ttype == StreamTokenizer.TT_EOL) {
        System.out.println();
    }
}
```

### 使用 `DataInputStream`
我们可以使用 DataInputStream 从文件中读取二进制或原始数据类型。
```
@Test
public void whenReadWithDataInputStream_thenCorrect() throws IOException {
    String expectedValue = "Hello, world!";
    String file ="src/test/resources/fileTest.txt";
    String result = null;

    DataInputStream reader = new DataInputStream(new FileInputStream(file));
    int nBytesToRead = reader.available();
    if(nBytesToRead > 0) {
        byte[] bytes = new byte[nBytesToRead];
        reader.read(bytes);
        result = new String(bytes);
    }

    assertEquals(expectedValue, result);
}
```

### 使用 `FileChannel`
如果我们读取的是大文件，FileChannel 可能比标准 IO 更快。
```
@Test
public void whenReadWithFileChannel_thenCorrect()
  throws IOException {
    String expected_value = "Hello, world!";
    String file = "src/test/resources/fileTest.txt";
    RandomAccessFile reader = new RandomAccessFile(file, "r");
    FileChannel channel = reader.getChannel();

    int bufferSize = 1024;
    if (bufferSize > channel.size()) {
        bufferSize = (int) channel.size();
    }
    ByteBuffer buff = ByteBuffer.allocate(bufferSize);
    channel.read(buff);
    buff.flip();
    
    assertEquals(expected_value, new String(buff.array()));
    channel.close();
    reader.close();
}
```