# Netty官方demo-HexDumpProxy 分析
{docsify-updated}


<center><img src="pics/HexDumpProxy.png" alt=""></center>

`HexDumpProxyFrontendHandler` 的 `channelRead` 方法中收到客户端请求，就调用 outboundChannel 的 write 方法将数据转发
给 upstream:
```
@Override
public void channelRead(final ChannelHandlerContext ctx, Object msg) {
    if (outboundChannel.isActive()) {
        outboundChannel.writeAndFlush(msg).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) {
                if (future.isSuccess()) {
                    // was able to flush out data, start to read the next chunk
                    ctx.channel().read();
                } else {
                    future.channel().close();
                }
            }
        });
    }
}
```

与此类似， `HexDumpProxyBackendHandler` 的 `channelRead` 方法中收到 upstream 的响应后，就调用 inboundChannel 的 write 方法将数据转发给客户端：
```
@Override
public void channelRead(final ChannelHandlerContext ctx, Object msg) {
    inboundChannel.writeAndFlush(msg).addListener(new ChannelFutureListener() {
        @Override
        public void operationComplete(ChannelFuture future) {
            if (future.isSuccess()) {
                ctx.channel().read();
            } else {
                future.channel().close();
            }
        }
    });
}
```

另外有一点需要注意的是，代理端在创建 upstream 的 socket 时，使用了下述代码：
```
Bootstrap b = new Bootstrap();
b.group(inboundChannel.eventLoop())
    .channel(ctx.channel().getClass())
    .handler(new HexDumpProxyBackendHandler(inboundChannel))
    .option(ChannelOption.AUTO_READ, false);
```
重点是 `b.group(inboundChannel.eventLoop())` ，这样就能保证这些 socket 是在同一个线程中处理，避免的多线程处理问题。
