# Spring 中的资源处理
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/core/resources.html

## Resource 接口
```
public interface InputStreamSource {
	InputStream getInputStream() throws IOException;
}

public interface Resource extends InputStreamSource {
	boolean exists();
	boolean isReadable();
	boolean isOpen();
	boolean isFile();
	URL getURL() throws IOException;
	URI getURI() throws IOException;
	File getFile() throws IOException;
	ReadableByteChannel readableChannel() throws IOException;
	long contentLength() throws IOException;
	long lastModified() throws IOException
	Resource createRelative(String relativePath) throws IOException;
	String getFilename();
	String getDescription();
}
```

+ `getInputStream()` ：定位并打开资源，返回用于从资源读取的 `InputStream` 。每次调用都应返回新的 `InputStream` 。调用方负责关闭该流。
+ `exists()` ：返回布尔值，指示该资源是否实际存在于物理介质中。
+ `isOpen()` ：返回布尔值，指示该资源是否代表具有打开流的句柄。若返回 `true` ，则该 `InputStream` 不可重复读取，必须仅读取一次后立即关闭以避免资源泄漏。除 `InputStreamResource` 外，所有常规资源实现均返回 false。
+ `getDescription()` ：返回该资源的描述信息，用于处理资源操作时的错误输出。通常为资源的完全限定文件名或实际 URL。

### 常见的 Resource 实现
+ `UrlResource`
+ `ClassPathResource`
+ `FileSystemResource`
+ `PathResource`
+ `ServletContextResource`
+ `InputStreamResource`
+ `ByteArrayResource`


## ResourceLoader 接口
```
public interface ResourceLoader {
	Resource getResource(String location);
	ClassLoader getClassLoader();
}
```