# Tomcat 线程模型
{docsify-updated}


<center><img src="pics/tomcat.png" alt=""></center>

```
public class TomcatModel {

    // 负责接收 TCP 连接
    private final Thread acceptor = new Thread(() -> {
        while (true) {
            SocketChannel channel = serverSocket.accept();

            // 将连接注册到 Poller
            poller.register(channel);
        }
    });

    // 负责监听 Socket I/O 事件
    private final Thread poller = new Thread(() -> {
        while (true) {
            // 阻塞等待多个 Socket 的 I/O 就绪事件
            selector.select();

            for (SelectionKey key : selector.selectedKeys()) {

                if (key.isReadable()) {
                    SocketChannel channel =
                            (SocketChannel) key.channel();

                    // 将请求处理任务提交给 Worker
                    workerPool.execute(() -> {
                        processHttpRequest(channel);
                    });
                }
            }
        }
    });

    // Worker 线程池：执行 HTTP 请求处理
    private final ExecutorService workerPool =
            Executors.newFixedThreadPool(200);

    private void processHttpRequest(SocketChannel channel) {
        // 1. 读取 HTTP 请求
        // 2. 解析 HTTP 协议
        // 3. 执行 Filter / Servlet / Controller
        // 4. 写回 HTTP Response
    }
}
```

对照 Netty： 
```
public class NettyModel {

    // Boss：负责接收 TCP 连接
    private final EventLoop boss = new EventLoop();

    // Worker：负责处理连接上的 I/O 事件
    private final EventLoop[] workers = {
            new EventLoop(),
            new EventLoop(),
            new EventLoop(),
            new EventLoop()
    };

    // Boss 接收连接
    public void acceptConnection() {
        SocketChannel channel = serverSocket.accept();

        // 将 Channel 分配给一个 Worker EventLoop
        EventLoop worker = nextWorker();

        worker.register(channel);
    }

    // 每个 EventLoop 都有自己的线程、Selector 和任务队列
    class EventLoop {

        private final Selector selector = openSelector();

        private final Queue<Runnable> tasks =
                new ConcurrentLinkedQueue<>();

        private final Thread thread = new Thread(() -> {
            while (true) {

                // 1. 监听 I/O 就绪事件
                selector.select();

                // 2. 处理 I/O 事件
                for (SelectionKey key : selectedKeys()) {
                    if (key.isReadable()) {
                        SocketChannel channel =
                                (SocketChannel) key.channel();

                        // 直接在当前 EventLoop 线程执行 Handler
                        channelRead(channel);
                    }
                }

                // 3. 执行提交到 EventLoop 的任务
                runAllTasks();
            }
        });

        void register(SocketChannel channel) {
            // 注册到当前 EventLoop 的 Selector
            channel.configureBlocking(false);
            channel.register(selector, SelectionKey.OP_READ);
        }

        void channelRead(SocketChannel channel) {
            // 读取数据
            // 解码 HTTP
            // 执行 Pipeline Handler
            // 写回响应
        }

        void execute(Runnable task) {
            tasks.offer(task);
            selector.wakeup();
        }
    }
}
```

Netty 也可以通过 EventExecutorGroup 把 Handler 转移到独立业务线程池中执行；Tomcat 也支持 Servlet 异步处理。因此，这里比较的是两者的默认调度模型，而不是它们全部的能力。