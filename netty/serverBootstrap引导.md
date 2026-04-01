# ServerBootstrap 服务端引导
{docsify-updated}

> https://www.cnblogs.com/bytesfly/p/the-truth-of-netty.html  
> https://www.yuque.com/bytesfly/backend/ig71cd

<center>
<img src="pics/netty-bootstrap.png" width="50%">
</center>

相对于将具体的引导类分别看作用于服务器和客户端的引导来说，记住它们的本意是用来支撑不同的应用程序的功能的将有所裨益。也就是说，服务器致力于使用一个父 Channel 来接受来自客户端的连接，并创建子 Channel 以用于它们之间的通信;而客户端将最可能只需要一个单独的、没有父 Channel 的 Channel 来用于所有的网络交互。


## ServerBootstrap

<center><img src="pics/server-bootstrap.jpg" width="50%"></center>

`ServerBootstrap` 类负责引导启动整个网络程序，其中有几个重要方法：
+ `ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup)`: 绑定 EventLoopGroup
+ `B channel(Class<? extends C> channelClass)`： 设置 Channel 的类型
+ `ServerBootstrap childHandler(ChannelHandler childHandler)` : 设置初始化客户端连接的初始化处理器，这是重要的方法，一般会在初始化器中通过 pipeline 添加删除自定义的 `ChannelHandler` 来处理消息。
 
`childHandler()`、`childAttr()` 和 `childOption()` 。这些调用支持特别用于服务器应用程序的操作。具体来说，`ServerChannel` 的实现负责创建子 `Channel` ，这些子 `Channel` 代表了已被接受的连接。因此，负责引导 `ServerChannel` 的 `ServerBootstrap` 提供了这些方法，以简化将设置应用到已被接受的子 `Channel` 的 `ChannelConfig` 的任务。

<center><img src="pics/serverbootstrap-channel.jpg" width="50%"></center>


<center>
<img src="pics/netty-server.jpg" width="60%">
</center>


## 示例

```java
public class App {
    public static void main(String[] args) {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap serverBootstrap = new ServerBootstrap();
            serverBootstrap.group(bossGroup,workerGroup)
                    .channel(NioServerSocketChannel.class) //指定Channel 的类型
                    .childHandler(new MyChannelInitializer()); //定义建立与客户端连接时初始化Channel 的对象，可以在初始化时通过pipeline指定客户端连接的处理逻辑

            ChannelFuture future = serverBootstrap.bind(9999).sync(); //绑定监听的端口
            future.channel().closeFuture().sync();
        }catch (InterruptedException e){
            e.printStackTrace();
        }finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}


public class MyChannelInitializer extends ChannelInitializer<SocketChannel> {

    @Override
    protected void initChannel(SocketChannel socketChannel) throws Exception {
        ChannelPipeline pipeline = socketChannel.pipeline();
        //处理http消息的编解码
//        pipeline.addLast("httpServerCodec", new HttpServerCodec());
        //添加自定义的ChannelHandler
        pipeline.addLast("myHandler", new MyHandler());
    }
}

public class MyHandler extends ChannelHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf buf = (ByteBuf) msg;
        System.out.println("Server received:" + buf.toString(CharsetUtil.UTF_8));
        ctx.writeAndFlush(buf);
    }
}
```