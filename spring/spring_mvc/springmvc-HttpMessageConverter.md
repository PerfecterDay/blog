# HttpMessageConverter
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/web/webmvc/message-converters.html#message-converters

```
public interface HttpMessageConverter<T> {
	boolean canRead(Class<?> clazz, @Nullable MediaType mediaType);

	boolean canWrite(Class<?> clazz, @Nullable MediaType mediaType);

	List<MediaType> getSupportedMediaTypes();

	default List<MediaType> getSupportedMediaTypes(Class<?> clazz) {
		return (canRead(clazz, null) || canWrite(clazz, null) ?
				getSupportedMediaTypes() : Collections.emptyList());
	}

	T read(Class<? extends T> clazz, HttpInputMessage inputMessage)
			throws IOException, HttpMessageNotReadableException;

	void write(T t, @Nullable MediaType contentType, HttpOutputMessage outputMessage)
			throws IOException, HttpMessageNotWritableException;
}
```

Spring Web 模块包含 `HttpMessageConverter` 接口，用于通过 `InputStream` 和 `OutputStream` **读写HTTP请求和响应的正文** 。 `HttpMessageConverter` 实例既可在客户端（例如在 `RestClient` 中）使用，也可在服务器端（例如在 `Spring MVC REST` 控制器中）使用。

Spring 提供了主要 `MIME` 类型的具体实现，这些实现默认已在客户端的 `RestClient` 和 `RestTemplate` 以及服务器端的 `RequestMappingHandlerAdapter` 中注册。

下面介绍几种 `HttpMessageConverter` 的实现。完整的列表请参阅 `HttpMessageConverter` 的 Javadoc。所有转换器均使用默认 `MIME` 类型，但可以通过设置 `supportedMediaTypes` 属性来覆盖该默认值。
+ `StringHttpMessageConverter`： 读取 `text/*` 的请求，写回 `Content-Type: text/plain` 
+ `FormHttpMessageConverter`
+ `ByteArrayHttpMessageConverter`
+ `MarshallingHttpMessageConverter`
+ `JacksonJsonHttpMessageConverter`
+ `JacksonXmlHttpMessageConverter`
+ `KotlinSerializationJsonHttpMessageConverter`
+ `JacksonCborHttpMessageConverter`
+ `SourceHttpMessageConverter`
+ `GsonHttpMessageConverter`
+ `JsonbHttpMessageConverter`
+ `ProtobufHttpMessageConverter` : `application/x-protobuf`, 需要 `com.google.protobuf:protobuf-java` 依赖
+ `ProtobufJsonFormatHttpMessageConverter`
