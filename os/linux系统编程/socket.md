# Socket
{docsify-updated}

本来套接字就只是一个概念而已，并不存在实体，如果一定要赋予它一个实体，我们可以说这些控制信息就是套接字的实体，或者说存放控制信息的内存空间就是套接字的实体。协议栈在执行操作时需要参阅这些控制信息。**协议栈是根据套接字中记录的控制信息来工作的。**

关键的 socket 系统调用包括以下几种:
+ socket()：系统调用创建一个新 socket。
+ bind()  ：系统调用将一个 socket 绑定到一个地址上。通常，服务器需要使用这个调用来将其 socket 绑定到一个众所周知的地址上使得客户端能够定位到该 socket 上。
+ listen()：系统调用允许一个流 socket 接受来自其他 socket 的接入连接。
+ accept()：系统调用在一个监听流 socket 上接受来自一个对等应用程序的连接，并可选地返回对等 socket 的地址。
+ connect()：系统调用建立与另一个 socket 之间的连接。

socket 的 I/O 操作可以使用传统的 read()和 write()系统调用或使用一组 socket 特有的系统调用(如 send()、recv()、sendto()以及 recvfrom())来完成。在默认情况下，这些系统调用在 I/O 操作无 法被立即完成时会阻塞。

## Socket创建
```
#include <sys/socket.h>
int socket(int domain, int type, int protocol);
```

### domain 参数
domain 参数指定了 socket 的通信 domain。socket 存在于一个通信 domain 中，它确定:
+ 识别出一个 socket 的方法(即 socket“地址”的格式);
+ 通信范围(即是在位于同一主机上的应用程序之间还是在位于使用一个网络连接起来的不同主机上的应用程序之间)。 
 
现代操作系统至少支持下列 domain:
+ UNIX (AF_UNIX) domain： 允许在同一主机上的应用程序之间进行通信。(POSIX.1g 使用名称 AF_LOCAL 作为 AF_UNIX 的同义词，但 SUSv3 并没有使用这个名称。)
+ IPv4 (AF_INET) domain： 允许在使用因特网协议第 4 版(IPv4)网络连接起来的主机上的应用程序之间进行通信。
+ IPv6 (AF_INET6) domain： 允许在使用因特网协议第 6 版(IPv6)网络连接起来的主机上的应用程序之间进行通信。尽管 IPv6 被设计成了 IPv4 接任者，但目前后一种协议仍然是使用最广的协议。
<center><img src="pics/socket-domain.png" width="60%"></center>

### type 参数
type 参数指定了 socket 类型，每个 socket 实现都至少提供了两种 socket：流和数据报。这个参数通常在创建 TCP socket（流） 时会被指定为 `SOCK_STREAM` ，而在创建 UDP socket（数据报）时会被指定为 `SOCK_DGRAM` 。
<center><img src="pics/socket-type.png" width="60%"></center>

流 socket(SOCK_STREAM)提供了一个可靠的双向的字节流通信信道。在这段描述中 的术语的含义如下。
+ 可靠的:表示可以保证发送者传输的数据会完整无缺地到达接收应用程序(假设网络 链接和接收者都不会崩溃)或收到一个传输失败的通知。
+ 双向的:表示数据可以在两个 socket 之间的任意方向上传输。
+ 字节流:表示与管道一样不存在消息边界的概念

一个流 socket 类似于使用一对允许在两个应用程序之间进行双向通信的管道，它们之间的差别在于(Internet domain)socket 允许在网络上进行通信。

流 socket 的正常工作需要一对相互连接的 socket，因此流 socket 通常被称为面向连接的。术语“对等 socket”是指连接另一端的 socket，“对等地址”表示该 socket 的地址，“对等应用 程序”表示利用这个对等 socket 的应用程序。有些时候，术语“远程”(或外部)是作为对等 的同义词使用。类似地，有些时候术语“本地”被用来指连接的这一端上的应用程序、socket 或地址。一个流 socket 只能与一个对等socket 进行连接。

数据报 socket(SOCK_DGRAM)允许数据以被称为数据报的消息的形式进行交换。在数据报 socket 中，消息边界得到了保留，但数据传输是不可靠的。消息的到达可能是无序的、重复的或者根本就无法到达。

数据报 socket 是更一般的无连接 socket 概念的一个示例。与流 socket 不同，一个数据报 socket 在使用时无需与另一个 socket 连接。(在 56.6.2 节中将会看到数据报 socket 可以与另一 个 socket 连接，但其语义与连接的流 socket 是不同的。)

在 Internet domain 中，数据报 socket 使用了用户数据报协议(UDP)，而流 socket 则(通常) 使用了传输控制协议(TCP)。一般来讲，在称呼这两种 socket 时不会使用术语“Internet domain 数据报 socket”和“Internet domain 流 socket”，而是分别使用术语“UDP socket”和“TCP socket”。

### protocol 参数
protocol 参数在本书描述的 socket 类型中总会被指定为 0。在一些 socket 类型中会使用非 零的 protocol 值，但本书并没有对这些 socket 类型进行描述。如在裸 socket(SOCK_RAW) 中会将 protocol 指定为 IPPROTO_RAW。

socket()在成功时返回一个引用在后续系统调用中会用到的新创建的 socket 的文件描述符。

## 将socket绑定到地址：bind()
```
#include <sys/socket.h>
int bind (int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```
sockfd 参数是在上一个 socket()调用中获得的文件描述符。  
addr 参数是一个指针，它指向了一个 指定该 socket 绑定到的地址的结构。传入这个参数的结构的类型取决于 socket domain。  
addrlen 参数 指定了地址结构的大小，addrlen 参数使用的 socklen_t 数据类型在 SUSv3 被规定为一个整数类型。

一般来讲，会将一个服务器的 socket 绑定到一个众所周知的地址——即一个固定的与服 务器进行通信的客户端应用程序提前就知道的地址。

### 通用 socket 地址结构:struct sockaddr
```
struct sockaddr {
	sa_family_t sa_family;
	char sa_data[14];
};
```
这个结构是所有 domain 特定的地址结构的模板，其中每个地址结构均以与 sockaddr 结构 中 sa_family 字段对应的 family 字段打头。(sa_family_t 数据类型在 SUSv3 中被规定成一个整数类型。)通过 family 字段的值足以确定存储在这个结构的剩余部分中的地址的大小和格式了。


## TCP Socket
流 socket 的运作与电话系统类似。
1. socket()系统调用将会创建一个 socket，这等价于安装一个电话。为使两个应用程序能够通信，每个应用程序都必须要创建一个 socket。
2. 通过一个流socket通信类似于一个电话呼叫。一个应用程序在进行通信之前必须要将其 socket 连接到另一个应用程序的 socket 上。两个 socket 的连接过程如下。
   1. 一个应用程序调用 bind()以将 socket 绑定到一个众所周知的地址上，然后调用 listen()通知内核它接受接入连接的意愿。这一步类似于已经有了一个为众人所知的电话号码并确保打开了电话，这样人们就可以打进电话了。
   2. 其他应用程序通过调用connect()建立连接，同时指定需连接的socket的地址。这类似于拨某人的电话号码。
   3. 调用 listen()的应用程序使用 accept()接受连接。这类似于在电话响起时拿起电话。如果在对等应用程序调用 connect()之前执行了 accept()，那么 accept()就会阻塞(“等待电话”)。
3. 一旦建立了一个连接之后就可以在应用程序之间(类似于两路电话会话)进行双向数据 传输直到其中一个使用 `close()`关闭连接为止。通信是通过传统的 read()和 write()系统调用或通过一些提供了额外功能的 socket 特定的系统调用(如 send()和 recv())来完成的。

<center><img src="pics/socket-stream.png" width="40%"></center>

### 监听接入连接:listen()
listen()系统调用将文件描述符 sockfd 引用的流 socket 标记为被动。这个 socket 后面会被用来接受来自其他(主动的)socket 的连接。
```
#include <sys/socket.h>
int listen(int sockfd, int backlog);
```
无法在一个已连接的 socket(即已经成功执行 connect()的 socket 或由 accept()调用返回的 socket)上执行 listen()。

要理解 backlog 参数的用途首先需要注意到客户端可能会在服务器调用 accept()之前调用 connect()。这种情况是有可能会发生的，如服务器可能正忙于处理其他客户端。这将会产生一个未决的连接，如图 56-2 所示。
<center><img src="pics/backlog.png" width="40%"></center>

**内核必须要记录所有未决的连接请求(半连接)的相关信息，这样后续的 accept()就能够处理这些请求了。backlog 参数允许限制这种未决连接（半连接）的数量。在这个限制之内的连接请求会立即成功。之外的连接请求就会阻塞直到一个未决的连接被接受(通过 accept())，并从未决连接队列（半连接队列）删除为止。**

通俗理解就是 backlog 设置半连接队列的长度，服务端未调用 accept() 之前，客户端的连接请求（connect()）会立即返回成功，请求会被放到半连接队列，如果队列已满，客户端的连接请求就会阻塞。服务端调用accept()后会处理半连接队列的请求，也会处理阻塞的请求。

SUSv3 允许一个实现为 backlog 的可取值规定一个上限并允许一个实现静默地将 backlog值向下舍入到这个限制值。SUSv3 规定实现应该通过在<sys/socket.h>中定义 SOMAXCONN 常量来发布这个限制。在 Linux 上，这个常量的值被定义成了 128。但从内核 2.4.25 起，Linux 允许在运行时通过 Linux 特有的/proc/sys/net/core/somaxconn 文件来调整这个限制。(在早期的 内核版本中，SOMAXCONN 限制是不可变的。)

#### java 实验 backlog
通过改变下述的 `serverSocket.bind(new InetSocketAddress(9999), 2);` backlog值，然后尝试用 client 连接可以发现，超过 backlog 后的client 不会连接上。
```
public class Server {
    public static void main(String[] args) throws IOException, InterruptedException {
        ServerSocket serverSocket = new ServerSocket();
        serverSocket.bind(new InetSocketAddress(9999), 2);
        Thread.sleep(10000000);
        Socket accept = serverSocket.accept();
        System.out.println("Accept");
        while (true) {
            Thread.sleep(10000);
        }
    }
}

public class Client {
    public static void main(String[] args) throws IOException {
        SocketAddress socketAddress = new InetSocketAddress("localhost", 9999);
        Socket socket = new Socket();
        socket.connect(socketAddress);
        System.out.println("Connected");
        OutputStream outputStream = socket.getOutputStream();
        InputStream inputStream = socket.getInputStream();
        int i = inputStream.read();
        System.out.println(i);
        ;
        if (inputStream.read() > 0) {
            System.out.println("read");
        }
    }
}
```

### 接受连接:accept()
accept()系统调用在文件描述符 sockfd 引用的监听流 socket 上接受一个接入连接。如果在调用 accept()时不存在未决的连接（半连接），那么调用就会阻塞直到有连接请求到达为止。
```
#include <sys/socket.h>
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```
理解 accept()的关键点是它会创建一个新 socket，并且正是这个新 socket 会与执行 connect() 的对等 socket 进行连接。accept()调用返回的函数结果是已连接的 socket 的文件描述符。监听 socket(sockfd)会保持打开状态，并且可以被用来接受后续的连接。一个典型的服务器应用 程序会创建一个监听 socket，将其绑定到一个众所周知的地址上，然后通过接受该 socket 上的 连接来处理所有客户端的请求。

传入 accept()的剩余参数会返回对端(客户端) socket 的地址。addr 参数指向了一个用来返回 socket 地址的结构。这个参数的类型取决于 socket domain(与 bind()一样)。  
addrlen 参数是一个值-结果参数。它指向一个整数，在调用被执行之前必须要将这个整数 初始化为 addr 指向的缓冲区的大小，这样内核就知道有多少空间可用于返回 socket 地址了。 当 accept()返回之后，这个整数会被设置成实际被复制进缓冲区中的数据的字节数。

### 连接到对等 socket:connect()
connect()系统调用将文件描述符 sockfd 引用的主动 socket 连接到地址通过 addr 和 addrlen指定的监听 socket 上。
```
#include <sys/socket.h>
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```
addr 和 addrlen 参数的指定方式与 bind()调用中对应参数的指定方式相同。  
如果 connect()失败并且希望重新进行连接，那么 SUSv3 规定完成这个任务的可移植的方法是关闭这个 socket，创建一个新 socket，在该新 socket 上重新进行连接。

### 流 socket I/O
一对连接的流 socket 在两个端点之间提供了一个双向通信信道，图 56-3 给出了 UNIX domain 的情形。
<center><img src="pics/tcp-io.png" width="40%"></center>

连接流 socket 上 I/O 的语义与管道上 I/O 的语义类似。
+ 要执行 I/O 需要使用 read()和 write()系统调用(或 socket 特有的 send() 和 recv()调用)。由于 socket 是双向的，因此在连接的两端都可以使用这两个调用。
+ 一个 socket 可以使用 close()系统调用来关闭或在应用程序终止之后关闭。之后当对等 应用程序试图从连接的另一端读取数据时将会收到文件结束(当所有缓冲数据都被读 取之后)。如果对等应用程序试图向其 socket 写入数据，那么它就会收到一个 SIGPIPE 信号，并且系统调用会返回 EPIPE 错误。在 44.2 节中曾提及过处理这种情况的常见方式是忽略 SIGPIPE 信号并通过 EPIPE 错误找出被关闭的连接。

### 关闭连接：close()
终止一个流 socket 连接的常见方式是调用 close()。如果多个文件描述符引用了同一个 socket，那么当所有描述符被关闭之后连接就会终止。  
假设在关闭一个连接之后，对等应用程序崩溃或没有读取或错误处理了之前发送给它的数据。在这种情况下就无法知道已经发生了一个错误。如果需要确保数据被成功地读取和处理，那么就必须要在应用程序中构建某种确认协议。这通常由一个从对等应用程序传过来的显式的确认消息构成。

## UDP Socket
数据报 socket 的运作类似于邮政系统：
1. socket()系统调用等价于创建一个邮箱。所有需要发送和接收数据报的应用程序都需要使用 socket()创建一个数据报 socket。
2. 为允许另一个应用程序发送其数据报(信)，一个应用程序需要使用 bind()将其 socket 绑定到一个众所周知的地址上。一般来讲，一个服务器会将其 socket 绑定到一个众所周知的地址上，而一个客户端会通过向该地址发送一个数据报来发起通信。(在一些 domain 中——特别是UNIX domain——客户端如果想要接受服务器发送来的数据报的话可能还需要使用bind()将一个地址赋给其 socket。)
3. 要发送一个数据报，一个应用程序需要调用 sendto()，它接收的其中一个参数是数据报发送到的 socket 的地址。这类似于将收信人的地址写到信件上并投递这封信。
4. 为接收一个数据报，一个应用程序需要调用 recvfrom()，它在没有数据报到达时会阻塞。由于 recvfrom()允许获取发送者的地址，因此可以在需要的时候发送一个响应。(这在发送 者的 socket 没有绑定到一个众所周知的地址上时是有用的，客户端通常是会碰到这种情况。)这里对这个比喻做了一点延伸，因为已投递的信件上是无需标记上发送者的地址的。
5. 当不再需要 socket 时，应用程序需要使用 close()关闭 socket。

<center><img src="pics/udpsys.png" width="40%"></center>

### 交换数据报:recvfrom 和 sendto()
```
#include <sys/socket.h>
ssize_t sendto (int sockfd, const void *buffer, size_t length, int flags, const struct sockaddr *des_addr, socklen_t addrlen);
ssize_t recvfrom (int sockfd, void *buffer, size_t length, int flags, struct sockaddr *src_addr, socklen_t *addrlen);
```
src_addr 和 addrlen 参数被用来获取或指定与之通信的对等 socket 的地址。如果不关心发送者的地址，那么可以将 src_addr 和 addrlen 都指定为 NULL。在这种情况 下，recvfrom()等价于使用 recv()来接收一个数据报。也可以使用 read()来读取一个数据报，这 等价于在使用 recv()时将 flags 参数指定为 0。

不管 length 的参数值是什么，recvfrom()只会从一个数据报 socket 中读取一条消息。如果消息的大小超过了 length 字节，那么消息会被静默地截断为 length 字节。

### 在数据报 socket 上使用 connect()
尽管数据报 socket 是无连接的，但在数据报 socket 上应用 connect()系统调用仍然是起作用的。在数据报 socket 上调用 connect()会导致内核记录这个 socket 的对等 socket 的地址。术语**已连接的数据报 socket 就是指此种 socket**。术语非连接的数据报 socket 是指那些没有调用 connect()的数据报 socket(即新数据报 socket 的默认行为)。

当一个数据报 socket 已连接之后:
+ 数据报的发送可在 socket 上使用 write()(或 send())来完成并且会自动被发送到同样的对等 socket 上。与 sendto()一样，每个 write()调用会发送一个独立的数据报;
+ 在这个 socket 上只能读取由对等 socket 发送的数据报。
注意 connect()的作用对数据报 socket 是不对称的。上面的论断只适用于调用了 connect()数据报 socket，并不适用于它连接的远程 socket(除非对等应用程序在其 socket 上也调用了 connect())。

通过再发起一个 connect()调用可以修改一个已连接的数据报 socket 的对等 socket。此外，通过指定一个地址族(如 UNIX domain 中的 sun_family 字段)为 AF_UNSPEC 的地址结构还可以解除对等关联关系。但需要注意的是，其他很多 UNIX 实现并不支持将 AF_UNSPEC 用于这种用途。

为一个数据报 socket 设置一个对等 socket，这种做法的一个明显优势是在该 socket 上传输数据时可以使用更简单的 I/O 系统调用，即无需使用指定了 dest_addr 和 addrlen 参数的 sendto()，而只需要使用 write()即可。设置一个对等 socket 主要对那些需要向单个对等 socket(通常是某种数据报客户端)发送多个数据报的应用程序是比较有用的。