# Spring MVC 中 RequestMappingHandlerAdapter 分析
{docsify-updated}


## handleInternal 方法
在 handleInternal 方法中最主要的是调用 invokeHandlerMethod 方法。

## invokeHandlerMethod 方法
此方法是整个类的核心方法，大部分处理流程在这个方法中完成。

```
WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
if (this.argumentResolvers != null) {
	invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
}
if (this.returnValueHandlers != null) {
	invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
}
invocableMethod.setDataBinderFactory(binderFactory);
invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);
ModelAndViewContainer mavContainer = new ModelAndViewContainer();
......
invocableMethod.invokeAndHandle(webRequest, mavContainer);
```

有几个重要的对象 `WebDataBinderFactory` , `ModelFactory` , `ModelAndViewContainer` ,`ServletInvocableHandlerMethod` 。

## ServletInvocableHandlerMethod
<center><img src="pics/handlermappingAdapter1.png" width="50%"></center>

上图中红圈的地方是核心的两个方法：
1. 第一个调用 `invokeForRequest()`，该方法会一步步处理请求参数，直到调用完 controller 的方法得到返回值
2. 第二个是使用 `HandlerMethodReturnValueHandlerComposite` 对象处理返回值。

### 请求参数处理
```
ServletInvocableHandlerMethod.invokeAndHandle -> InvocableHandlerMethod.invokeForRequest ->
    Object[] args = InvocableHandlerMethod.getMethodArgumentValues -> 	
	args[i] = HandlerMethodArgumentResolverComposite.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
    ->ModelAttributeMethodProcessor.resolveArgument()
        			WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name);
                    bindRequestParameters(binder, webRequest);
				    validateIfApplicable(binder, parameter);-> validator.validate();
                    return args;

    return doInvoke(args);        
```

更详细的处理过程请参照[Spring-MVC中 参数绑定与校验](./springmvc-处理方法的参数解析转换与校验.md)

### 响应参数处理
为什么我们在 controller 方法中返回一个自定义的对象，客户端会收到 json 响应呢？ 为什么返回一个 `SseEmitter` 对象就能实现 SSE 呢？

这都是因为 spring 在获取 controller 处理方法的返回值后，会根据返回值调用不同的 `HandlerMethodReturnValueHandler` 对象做进一步处理。

<center><img src="pics/handlermappingAdapter2.png" width="50%"></center>

当请求处理方法或者 controller 用 `@ResponseBody` 注解修饰时， 返回值就会用 `RequestResponseBodyMethodProcessor` 处理。
当返回值是 `ResponseBodyEmitter` 类型时（`SseEmitter`是其子类），就会用 `ResponseBodyEmitterReturnValueHandler` 处理。

上图中还有许多其他的 `HandlerMethodReturnValueHandler` 类，处理不同的返回值，注意，返回值可以被多个 `HandlerMethodReturnValueHandler` 对象处理，只要 `HandlerMethodReturnValueHandler` 能处理返回值。

`@ControllerAdvice` 和 `@RestControllerAdvice` 都是在这个环节被调用的。