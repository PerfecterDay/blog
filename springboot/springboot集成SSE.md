# Springboot 使用 SSE(server-sent events)
{docsify-updated}

> https://www.ruanyifeng.com/blog/2017/05/server-sent_events.html


```
@GetMapping(path="/events", produces=MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter handle() {
	SseEmitter emitter = new SseEmitter();
	// Save the emitter somewhere..
	return emitter;
}

// In some other thread
emitter.send("Hello once");

// and again later on
emitter.send("Hello again");

// and done at some point
emitter.complete();
```


## 示例
1. 服务端代码：
```
@RequestMapping(path = "/stream", method = RequestMethod.GET)
    public SseEmitter stream() throws IOException {
        SseEmitter emitter = new SseEmitter();

        emitters.add(emitter);
        emitter.onCompletion(() -> emitters.remove(emitter));

        return emitter;
    }

    @ResponseBody
    @RequestMapping(path = "/chat", method = RequestMethod.POST)
    public Message sendMessage(@Valid Message message) {
        log.info("Got message" + message);

        emitters.forEach((SseEmitter emitter) -> {
            try {
                emitter.send(message, MediaType.APPLICATION_JSON);
            } catch (IOException e) {
                emitter.complete();
                emitters.remove(emitter);
                e.printStackTrace();
            }
        });
        return message;
    }
```

2. web 端代码
```
<!DOCTYPE html>
<html>
<body>
    <h1>SSE Demo</h1>
    <div id="messages"></div>
    <script>
        const eventSource = new EventSource('http://localhost:8050/user/handleSse');
        const messages = document.getElementById('messages');
        eventSource.onmessage = function(event) {
            const message = document.createElement('p');
            message.textContent = event.data;
            messages.appendChild(message);
        };
        eventSource.onerror = function(error) {
            console.error('EventSource failed:', error);
            eventSource.close();
        };
    </script>
</body>
</html>
```

## springboot的原理分析
根据 [Spring-MVC中 RequestMappingHandlerAdapter](/spring/spring_mvc/springmvc中RequestMappingHandlerAdapter分析.md#响应参数处理)分析可知， `ResponseBodyEmitterReturnValueHandler` 负责处理 `SseEmitter` 类型的返回值。