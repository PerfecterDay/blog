# 异步请求与长轮询
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-async.html#mvc-ann-async-vs-webflux

## 异步请求
Spring MVC 与 Servlet **异步请求处理**深度集成：
+ 控制器方法中返回 `DeferredResult` 、 `Callable` 和 `WebAsyncTask` 类型可支持单个异步返回值。
+ 控制器可流式传输多种值，包括 `SSE` 和原始数据。
+ 控制器可使用响应式客户端并返回响应式类型进行响应处理。

### DeferredResult
在 Servlet 容器中启用异步请求处理功能后，控制器方法可以将任何受支持的控制器方法的返回值包装为 `DeferredResult` ，如下例所示：

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
控制器可以异步地从另一个线程返回结果——例如，响应外部事件（JMS 消息）、定时任务或其他事件。

### Callable
如以下示例所示，控制器可以将任何受支持的返回值包装为 `java.util.concurrent.Callable`：

```
@PostMapping
public Callable<String> processUpload(final MultipartFile file) {
	return () -> "someView";
}
```
随后，可以通过将给定的任务交由已配置的 `AsyncTaskExecutor` 执行来获取返回值。

### WebAsyncTask
`WebAsyncTask` 与使用 `Callable` 类似，但允许自定义额外设置，例如请求超时值，并可指定 `AsyncTaskExecutor` 来执行 `java.util.concurrent.Callable`，而非使用 Spring MVC 全局默认的配置。以下是一个使用 `WebAsyncTask` 的示例：

```
@GetMapping("/callable")
WebAsyncTask<String> handle() {
	return new WebAsyncTask<String>(20000L,()->{
		Thread.sleep(10000); //simulate long-running task
		return "asynchronous request completed";
	});
}
```

### 异步请求的处理
`Servlet` 异步请求处理的简要概述：
+ 可以通过调用 `request.startAsync()` 方法将 `ServletRequest` 设置为异步模式。这样做的主要效果是， `Servlet` （以及任何过滤器）可以退出，但响应仍保持打开状态，以便稍后完成处理。
+ 调用 `request.startAsync()` 会返回一个 `AsyncContext` 对象，可以利用它进一步控制异步处理。例如，它提供了 `dispatch` 方法，该方法与 `Servlet API` 中的 `forward` 类似，不同之处在于它允许应用程序在 `Servlet` 容器线程上继续处理请求。
+ `ServletRequest` 提供了对当前 `DispatcherType` 的访问，可以利用它来区分初始请求的处理、异步分发、转发以及其他分发器类型。

`DeferredResult` 的处理机制如下：
+ 控制器返回一个 `DeferredResult` ，并将其保存在某个内存队列或列表中，以便后续访问。
+ Spring MVC 调用了 `request.startAsync()` 。
+ 与此同时， `DispatcherServlet` 以及所有已配置的过滤器都会退出请求处理线程，但响应仍保持打开状态。
+ 应用程序从某个线程设置了 `DeferredResult` ，而 Spring MVC 将请求发回给 Servlet 容器。
+ `DispatcherServlet` 再次被调用，处理流程将根据异步生成的返回值继续进行。

`Callable` 的工作原理如下：
+ 控制器返回一个 `Callable` 对象。
+ Spring MVC 会调用 `request.startAsync()` ，并将该 `Callable` 提交给 `AsyncTaskExecutor` ，以便在单独的线程中进行处理。
+ 与此同时， `DispatcherServlet` 以及所有过滤器都已退出 `Servlet` 容器线程，但响应仍保持打开状态。
+ 最终， `Callable` 会返回一个结果，Spring MVC 会将请求发回 `Servlet` 容器以完成处理。
+ `DispatcherServlet` 再次被调用，处理流程将根据 `Callable` 异步生成的返回值继续进行。


#### CAllable 详细分析
测试代码：
```
@RestController()
@RequestMapping(value = "/device", version = "v1.0")
@RequiredArgsConstructor
public class TestController {

    @GetMapping(value = "/test",version = "v1.0")
    public Callable<ResponseEntity> processUpload(final MultipartFile file) {
        return () -> {
            Thread.sleep(10000);
            return ResponseEntity.ok("someView");
        };
    }
}
```

1. 首先， `SpringMvc` 会按照跟处理其他请求一样的逻辑调用分到 `processUpload` 方法，并且返回一个 `Callable` 对象
2. `CallableMethodReturnValueHandler` 会借助 `WebAsyncManager.startCallableProcessing(..)` 开启 `request.startAsync()`，在这一步会调用 `Servlet` 的异步请求支持 API `AsyncContext asyncContext = request.startAsync(..)` 并获取到 `AsyncContext` 对象保存到 `AsyncWebRequest` （Spring的接口）实现中
3. 向线程池提交一个任务，这个任务会调用 `Callable` 对象执行，获取到 `Callable` 的执行结果后调用 `AsyncContext.dispatch()` 方法在次进入 `DispatcherServlet` 处理流程。注意这里， `Spring MVC` 中处理请求的线程，提交完这个任务，就继续往下运行了，不会等待任务的执行结果，这个线程可以去处理其他请求。 `Callable` -示例中的 `Thread.sleep(10000)` 会在线程池中运行。
4. Spring MVC 再次调度处理这个请求的时候，发现有 `AsyncContext.hasConcurrentResult()` 为 true ，就不会去调用 `Controller` 中的方法，具体看 `ServletInvocableHandlerMethod.wrapConcurrentResult()` 方法
5. `DeferredResultMethodReturnValueHandler`、`AsyncTaskMethodReturnValueHandler` 

```
CallableMethodReturnValueHandler.handleReturnValue(...) -> WebAsyncManager.startCallableProcessing(...) 
DeferredResultMethodReturnValueHandler.handleReturnValue(...) -> WebAsyncManager.startDeferredResultProcessing(...)
AsyncTaskMethodReturnValueHandler.handleReturnValue(...) -> WebAsyncManager.startCallableProcessing(...) 
```

`WebAsyncManager.startCallableProcessing` 代码跟踪：
```
public void startCallableProcessing(final WebAsyncTask<?> webAsyncTask, Object... processingContext)
			throws Exception {
        ...
		final Callable<?> callable = webAsyncTask.getCallable();
		...
		startAsyncProcessing(processingContext); // -> StandardServletAsyncWebRequest.startAsync()  
-> this.asyncContext = getRequest().startAsync(getRequest(), getResponse());

        // 提交任务到异步线程池中，开启callable 的调用并重新调度请求
		try {
			Future<?> future = this.taskExecutor.submit(() -> {
				Object result = null;
				try {
					interceptorChain.applyPreProcess(this.asyncWebRequest, callable);
					result = callable.call();
				}
				catch (Throwable ex) {
					result = ex;
				}
				finally {
					result = interceptorChain.applyPostProcess(this.asyncWebRequest, callable, result);
				}
				setConcurrentResultAndDispatch(result); //调用 AsyncContext.dispatch 方法重新走一遍 DispatcherServlet 的调度
			});
			interceptorChain.setTaskFuture(future);
		}
		catch (Throwable ex) {
			Object result = interceptorChain.applyPostProcess(this.asyncWebRequest, callable, ex);
			setConcurrentResultAndDispatch(result);
		}
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