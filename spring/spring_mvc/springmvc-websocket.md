# springmvc 对 websocket 的支持
{docsify-updated}

## Websocket API
创建 WebSocket 服务器只要实现 `WebSocketHandler` 或扩展 `TextWebSocketHandler` 或 `BinaryWebSocketHandler` 一样简单。下面的示例使用了 `TextWebSocketHandler` ：
```
public class MyHandler extends TextWebSocketHandler {
	@Override
	public void handleTextMessage(WebSocketSession session, TextMessage message) {
		// ...
	}
}
```

有专门的 WebSocket Java 配置和 XML 命名空间支持将前面的 `WebSocketHandler` 处理程序映射到特定 URL：
```
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
	@Override
	public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
		registry.addHandler(myHandler(), "/myHandler");
	}

	@Bean
	public WebSocketHandler myHandler() {
		return new MyHandler();
	}
}
```
