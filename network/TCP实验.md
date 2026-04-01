# TCP实验
{docsify-updated}


## 通过实验深入了解 TCP 连接的建立和关闭
> https://mp.weixin.qq.com/s/OpOCIVxKF1xK-HI-E-8uRg

## TCP的服务器设计
初始 Listen 状态：
<center><img src="pics/tcp-listen.jpg" width="60%"></center>
接收到若干请求后：
<center><img src="pics/tcp_listen_est.jpg" width="60%"></center>

TCP使用由本地地址和远端地址组成的4元组:**目的IP地址、目的端口号、源IP地址和源端口号来处理传入的多个请求。TCP仅通过目的端口号无法确定哪个进程应该处理一个TCP请求。**另外，在三个使用端口23的进程中，只有处于LISTEN的进程能够接收新的连接请求（SYN）。处于ESTABLISHED的进程将不能接收SYN报文段，而处于LISTEN的进程将不能接收数据报文段。

### 连接队列 （https://www.alibabacloud.com/blog/tcp-syn-queue-and-accept-queue-overflow-explained_599203）
当服务器正处于忙时，TCP是如何处理这些呼入的连接请求?
<center><img src="pics/tcp-queue.png" width="50%"></center>

+ 半连接队列，也称 SYN 队列: /proc/sys/net/ipv4/tcp_max_syn_backlog(linux)
+ 全连接队列，也称 accepet 队列:  /proc/sys/net/core/somaxconn(linux)

1. 正等待连接请求的一端有一个固定长度的**全连接队列**，该队列中的连接已被TCP接受(即三次握手已经完成)，但还没有被应用层所接受。注意区分TCP接受一个连接是将其放入这个队列，而应用层接受连接是将其从该队列中移出。
2. 应用层将指明该队列的最大长度，这个值通常称为 **积压值(backlog)** 。它的取值范围是0~5之间的整数，包括0和5(大多数的应用程序都将这个值说明为5)
3. 当一个连接请求(即SYN)到达时，TCP使用一个算法，根据当前连接队列中的连接数来确定是否接收这个连接。我们期望应用层说明的积压值为这一端点所能允许接受连接的最大数目，但情况不是那么简单。注意，**积压值说明的是TCP监听的端点已被TCP接受而等待应用层接受的最大连接数。这个积压值对系统所允许的最大连接数，或者并发服务器所能并发处理的客户数，并无影响。**
4. 如果对于新的连接请求，该TCP监听的端点的连接队列中还有空间，TCP模块将对SYN进行确认并完成连接的建立。但应用层只有在三次握手中的第三个报文段收到后才会知道这个新连接。另外，当客户进程的主动打开成功但服务器的应用层还不知道这个新的连接时，它可能会认为服务器进程已经准备好接收数据了(如果发生这种情况，服务器的TCP仅将接收的数据放入缓冲队列)。
5. 如果对于新的连接请求，连接队列中已没有空间，**TCP将不理会收到的SYN。也不发回任何报文段**(即不发回RST)。如果应用层不能及时接受已被TCP接受的连接，这些连接可能占满整个连接队列，客户的主动打开最终将超时。

### 实验验证全连接队列
实验代码：
```
public class NetDemo {
    public static void main(String[] args) throws IOException, InterruptedException {
        ServerSocket serverSocket = new ServerSocket(10000, 2);
        while (true) {
            Socket clientSocket = serverSocket.accept();
            if (clientSocket.isConnected()) {
                System.out.println(clientSocket.getInetAddress().getHostAddress() + ":" + clientSocket.getPort());
            }
            clientSocket.getOutputStream().write("hello".getBytes());
            Thread.sleep(1000000);
        }
    }
}
```

<center><img src="pics/backlog_exp.png" alt=""></center>

使用 lsof 观察状态的话会看到第四个连接有个短暂的 SYN_SENT 状态：
```
lsof -i :10000
COMMAND   PID USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
java    29099 demo    7u  IPv6 0xb801639153ee02d4      0t0  TCP *:ndmp (LISTEN)
java    29099 demo    8u  IPv6 0x75e6fa6faa349ae9      0t0  TCP localhost:ndmp->localhost:65274 (ESTABLISHED)
netcat  29207 demo    3u  IPv4 0x6e8271651ac6f5f9      0t0  TCP localhost:65274->localhost:ndmp (ESTABLISHED)
netcat  29292 demo    3u  IPv4 0x779ff09e5c83e011      0t0  TCP localhost:65292->localhost:ndmp (ESTABLISHED)
netcat  29917 demo    3u  IPv4 0x4a810dec1fddc647      0t0  TCP localhost:65414->localhost:ndmp (ESTABLISHED)
netcat  13274 demo    3u  IPv4 0x5d048ad334188048      0t0  TCP localhost:65473->localhost:ndmp (SYN_SENT)
```

超时失败后，套接字关闭：
```
lsof -i :10000
COMMAND   PID USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
java    29099 demo    7u  IPv6 0xb801639153ee02d4      0t0  TCP *:ndmp (LISTEN)
java    29099 demo    8u  IPv6 0x75e6fa6faa349ae9      0t0  TCP localhost:ndmp->localhost:65274 (ESTABLISHED)
netcat  29207 demo    3u  IPv4 0x6e8271651ac6f5f9      0t0  TCP localhost:65274->localhost:ndmp (ESTABLISHED)
netcat  29292 demo    3u  IPv4 0x779ff09e5c83e011      0t0  TCP localhost:65292->localhost:ndmp (ESTABLISHED)
netcat  29917 demo    3u  IPv4 0x4a810dec1fddc647      0t0  TCP localhost:65414->localhost:ndmp (ESTABLISHED)
```