# 异步请求与长轮询
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-async.html#mvc-ann-async-vs-webflux

## 异步请求
Spring MVC 与 Servlet **异步请求处理**深度集成：
+ 控制器方法中返回 `DeferredResult` 、 `Callable` 和 `WebAsyncTask` 类型可支持单个异步返回值。
+ 控制器可流式传输多种值，包括 `SSE` 和原始数据。
+ 控制器可使用响应式客户端并返回响应式类型进行响应处理。

### DeferredResult
```
@GetMapping("/quotes")
@ResponseBody
public DeferredResult<String> quotes() {
	DeferredResult<String> deferredResult = new DeferredResult<>();
	// Save the deferredResult somewhere..
	return deferredResult;
}

// From some other thread...
deferredResult.setResult(result);
```

### Callable
```
@PostMapping
public Callable<String> processUpload(final MultipartFile file) {
	return () -> "someView";
}
```

### WebAsyncTask
```
@GetMapping("/callable")
WebAsyncTask<String> handle() {
	return new WebAsyncTask<String>(20000L,()->{
		Thread.sleep(10000); //simulate long-running task
		return "asynchronous request completed";
	});
}
```


## 长轮询
```
@SpringBootApplication
@RestController
public class DefferedResultDemo {
    public static void main(String[] args) throws IOException {
        SpringApplication springApplication = new SpringApplication(DefferedResultDemo.class);
        Properties properties = new Properties();
        properties.put("server.port", "9000");
        springApplication.setDefaultProperties(properties);
        ConfigurableApplicationContext run = springApplication.run(args);
    }


    Map<String,DeferredResult<String>> map = new HashMap<>();

    @GetMapping("/get/{id}")
    public DeferredResult<String> deferredResult(@PathVariable String id) {
        DeferredResult<String> result = new DeferredResult<>();
        map.put(id, result);
        return result;
    }

    @GetMapping("/put/{id}")
    public Boolean putResult(@PathVariable String id) {
        DeferredResult<String> stringDeferredResult = map.get(id);
        return stringDeferredResult.setResult("I am back");
    }

}
```