# Java基础-写文件
{docsify-updated}

- [Java基础-写文件](#java基础-写文件)
    - [使用`BufferedWriter`](#使用bufferedwriter)
    - [使用 `PrintWriter`](#使用-printwriter)
    - [使用 `FileOutputStream`](#使用-fileoutputstream)
    - [使用 `DataOutputStream`](#使用-dataoutputstream)
    - [使用 `RandomAccessFile`](#使用-randomaccessfile)
    - [使用 `FileChannel`](#使用-filechannel)
    - [使用 `Files`](#使用-files)
    - [临时文件](#临时文件)
    - [文件锁](#文件锁)
    - [总结](#总结)

###  使用`BufferedWriter`  
如果我们能获取到文件的 `InputStream` 流，就能使用下述方法读取文件内容：
```
String str = "Hello";
BufferedWriter writer = new BufferedWriter(new FileWriter(fileName)); //覆盖写模式
writer.write(str);

BufferedWriter writer = new BufferedWriter(new FileWriter(fileName, true)); //追加写模式
writer.append(' ');
writer.append(str);

writer.close();
```
    
### 使用 `PrintWriter`
```
FileWriter fileWriter = new FileWriter(fileName);
PrintWriter printWriter = new PrintWriter(fileWriter);
printWriter.print("Some String");
printWriter.printf("Product name is %s and its price is %d $", "iPhone", 1000);
printWriter.close();
```
我们可以使用 `FileWriter` 、 `BufferedWriter` 甚至 `System.out` 创建写入器。


### 使用 `FileOutputStream`
使用 FileOutputStream 将二进制数据写入文件。
```
String str = "Hello";
FileOutputStream outputStream = new FileOutputStream(fileName);
byte[] strToBytes = str.getBytes();
outputStream.write(strToBytes);

outputStream.close();
```

### 使用 `DataOutputStream`
```
String value = "Hello";
FileOutputStream fos = new FileOutputStream(fileName);
DataOutputStream outStream = new DataOutputStream(new BufferedOutputStream(fos));
outStream.writeUTF(value);
outStream.close();
```

### 使用 `RandomAccessFile`
`FileOutputStream` 只能从头到尾写内容到文件中，要在文件某个位置进行读写操作需要使用 `RandomAccessFile`。

```
RandomAccessFile rw = new RandomAccessFile(filename, "rw");
rw.seek(position);
rw.writeInt(data);

rw.seek(position);
result = rw.readInt();

rw.close();

try (RandomAccessFile raf = new RandomAccessFile(file, "rw")) {
  raf.seek(0);
  raf.writeBytes("# ");
  System.out.println(path.getFileName() + "文件内容修改成功");
}
```

### 使用 `FileChannel`
```
RandomAccessFile stream = new RandomAccessFile(fileName, "rw");
FileChannel channel = stream.getChannel();
String value = "Hello";
byte[] strBytes = value.getBytes();
ByteBuffer buffer = ByteBuffer.allocate(strBytes.length);
buffer.put(strBytes);
buffer.flip();
channel.write(buffer);
stream.close();
channel.close();
```

### 使用 `Files`
```
String str = "Hello";
Path path = Paths.get(fileName);
byte[] strToBytes = str.getBytes();

Files.write(path, strToBytes);
```

### 临时文件
```
String toWrite = "Hello";
File tmpFile = File.createTempFile("test", ".tmp");
FileWriter writer = new FileWriter(tmpFile);
writer.write(toWrite);
writer.close();
```

### 文件锁
```
RandomAccessFile stream = new RandomAccessFile(fileName, "rw");
FileChannel channel = stream.getChannel();

FileLock lock = null;
try {
    lock = channel.tryLock();
} catch (final OverlappingFileLockException e) {
    stream.close();
    channel.close();
}
stream.writeChars("test lock");
lock.release();

stream.close();
channel.close();
```

### 总结
在探索了这么多写入文件的方法之后，让我们来讨论一些重要的注意事项：

+ 如果我们尝试从一个不存在的文件中读取数据，将抛出 FileNotFoundException 异常。
+ 如果我们尝试向不存在的文件写入内容，文件将首先被创建，不会抛出异常。
+ 使用流后，关闭流是非常重要的，因为它不会隐式关闭，以释放与之相关的任何资源。
+ 在输出流中，close() 方法会在释放资源前调用 flush()，强制将缓冲字节写入流。

纵观常见的使用方法，我们可以看到，例如，PrintWriter 用于写入格式化文本，FileOutputStream 用于写入二进制数据，DataOutputStream 用于写入原始数据类型，RandomAccessFile 用于写入特定位置，FileChannel 用于更快地写入较大的文件。