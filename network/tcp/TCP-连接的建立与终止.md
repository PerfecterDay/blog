# TCP连接的建立与终止——三次握手与四次挥手
{docsify-updated}

<center><img src="pics/tcp_open_close.png" alt="" width=40%></center>


## 三次握手建立连接

<center><img src="pics/three-handshake.png" alt="" width=40%></center>

1. 请求端（通常称为客户）发送一个 SYN段指明客户打算连接的服务器的端口，以及初始序号。这个SYN段为报文段1。
2. 服务器发回包含服务器的初始序号的 SYN报文段（报文段2）作为应答。同时，将确认序号设置为客户的ISN加1以对客户的SYN报文段进行确认。**一个SYN将占用一个序号。**
3. 客户必须将确认序号设置为服务器的 ISN加1以对服务器的SYN报文段进行确认（报文段3）。

## 四次挥手断开连接

<center><img src="pics/four-close.png" alt="" width=40%></center>

1. 首先进行关闭的一方发送第一个 FIN执行主动关闭，而另一方（收到这个 FIN）执行被动关闭。通常一方完成主动关闭而另一方完成被动关闭（也可以双方都执行主动关闭）。
2. 当服务器收到这个 FIN，它发回一个ACK，确认序号为收到的序号加 1（报文段5）。**和SYN一样，一个FIN将占用一个序号。**
3. 接着这个服务器程序就关闭它的连接，导致它的 TCP端发送一个FIN（报文段6），客户必须发回一个确认，并将确认序号设置为收到序号加1（报文段7）。

发送FIN将导致应用程序关闭它们的连接，这些FIN的ACK是由TCP软件自动产生的。

## TCP半关闭
TCP提供了连接的一端在结束它的发送后还能接收来自另一端数据的能力，这就是所谓的半关闭。
<center><img src="pics/half-close.jpg" alt="" width=40%></center>

初始端发出的FIN，接着是另一端对这个FIN的ACK报文段。接收半关闭FIN的一方仍能发送数据。我们只显示一个数据报文段和一个ACK报文段，但可能发送了许多数据报文段。当收到半关闭的一端在完成它的数据传送后，将发送一个FIN关闭这个方向的连接，这将传送一个文件结束符给发起这个半关闭的应用进程。当对第二个FIN进行确认后，这个连接便彻底关闭了。

没有半关闭，需要其他的一些技术让客户通知服务器,客户端已经完成了它的数据传送，但仍要接收来自服务器的数据。使用两个TCP连接也可作为一个选择，但使用半关闭的单连接更好。

## TCP状态变迁
<center><img src="pics/tcp-status.png" alt="" width=45%></center>

## 2MSL（TIME_WAIT）等待状态
`TIME_WAIT` 状态也称为 2MSL 等待状态。每个具体 TCP实现必须选择一个报文段最大生存时间MSL(Maximum Segment Lifetime) 。它是任何报文段被丢弃前在网络内的最长时间。我们知道这个时间是有限的，因为 TCP 报文段以IP数据报在网络内传输，而IP数据报则有限制其生存时间的 TTL字段。RFC 793 指出MSL为2分钟。然而，实现中的常用值是30秒，1分钟，或2分钟。

为什么需要 2MSL ?  
当 TCP 执行一个主动关闭，并发回最后一个ACK，该连接必须在 TIME_WAIT 状态停留的时间为2倍的MSL。**这样可让TCP再次发送最后的ACK以防这个ACK丢失（另一端超时并重发最后的 FIN）。**

我们说图18-13中客户执行主动关闭并进入TIME_WAIT是正常的。服务器通常执行被动关闭，不会进入TIME_WAIT状态。这暗示如果我们终止一个客户程序，并立即重新启动这个客户程序，则这个新客户程序将不能重用相同的本地端口。这不会带来什么问题，因为客户使用本地端口，而并不关心这个端口号是什么。  
然而，对于服务器，情况就有所不同，因为服务器使用熟知端口。如果我们终止一个已经建立连接的服务器程序，并试图立即重新启动这个服务器程序，服务器程序将不能把它的这个熟知端口赋值给它的端点，因为那个端口是处于2MSL连接的一部分。在重新启动服务器程序前，它需要在1~4分钟。

这种2MSL等待的另一个结果是**这个TCP连接在2MSL等待期间，定义这个连接的插口（客户的IP地址和端口号，服务器的 IP地址和端口号）不能再被使用。**这个连接只能在 2MSL结束后才能再被使用。大多数 TCP实现（如伯克利版）强加了更为严格的限制。在 2MSL等待期间，插口中使用的本地端口在默认情况下不能再被使用。

大多数TCP实现(如伯克利版)强加了更为严格的限制。在2MSL等待期间，插口中使用的**本地端口在默认情况下不能再被使用。（端口占用）**
某些实现和API提供了一种避开这个限制的方法。使用插口API时，可说明其中`SO_REUSEADDR`选项。它将让调用者对处于2MSL等待的本地端口进行赋值，但我们将看到TCP原则上仍将避免使用仍处于2MSL连接中的端口。

在连接处于2MSL等待时，任何迟到的报文段将被丢弃。因为处于2MSL等待的、由该插口对(socket pair)定义的连接在这段时间内不能被再用，因此当要建立一个有效的连接时，来自该连接的一个较早替身（incarnation）的迟到报文段作为新连接的一部分不可能不被曲解（一个连接由一个插口对来定义。一个连接的新的实例（instance）称为该连接的替身）。

需要再次强调2MSL等待的一个效果，因为我们将在第27章的文件传输协议FTP中遇到它。和以前介绍的一样，一个插口对（即包含本地IP地址、本地端口、远端IP地址和远端端口的4元组）在它处于2MSL等待时，将不能再被使用。尽管许多具体的实现中允许一个进程重新使用仍处于2MSL等待的端口（通常是设置选项SO_REUSEADDR），但TCP不能允许一个新的连接建立在相同的插口对上。 实验验证:

<center><img src="pics/2msl.png" alt=""></center>

在第1次运行sock程序中，我们将它作为服务器程序，端口号为6666，并从主机bsdi上的一个客户程序与它连接，这个客户程序使用的端口为1098。我们终止服务器程序，因此它将执行主动关闭。这将导致4元组`[140.252.13.33（本地IP地址）,6666（本地端口号）,140.252.13.35（另一端IP地址）,1098（另一端的端口号)]`在服务器主机进入2MSL等待。

在第2次运行sock程序时，我们将它作为客户程序，并试图将它的本地端口号指明为6666，同时与主机bsdi在端口1098上进行连接。但这个程序在试图将它的本地端口号赋值为6666时产生了一个差错，因为这个端口是处于2MSL等待4元组的一部分。

为了避免这个差错，我们再次运行这个程序，并使用选项`-A`来设置前面提到的`SO_REUSEADDR`。这将让sock程序能将它的本地端口号设置为6666，但当我们试图进行主动打开时，又出现了一个差错。即使它能将它的本地端口设置为6666，但它仍不能和主机bsdi在端口1098上进行连接，因为定义这个连接的插口对仍处于2MSL等待状态。

如果我们试图从其他主机来建立这个连接会如何？首先我们必须在sun上以-A标记来重新启动服务器程序，因为它需要的端口（6666）是还处于2MSL等待连接的一部分。
```
sun % sock -A -s 6666   启动服务器程序，在端口 6666监听
```
接着，在2MSL等待结束前，我们在bsdi上启动客户程序：
```
bsdi % sock -b1098 sun 6666

connected on 140.252.13.35.1098 to 140.252.13.33.6666
```

不幸的是它成功了！这违反了TCP规范，但被大多数的伯克利版实现所支持。这些实现允许一个新的连接请求到达仍处于TIME_WAIT状态的连接，只要新的序号大于该连接前一个替身的最后序号。在这个例子中，新替身的ISN被设置为前一个替身最后序号与128000的和。附录的RFC1185[Jacobsan、Braden和Zhang1990]指出了这项技术仍可能存在缺陷。

对于同一连接的前一个替身，这个具体实现中的特性让客户程序和服务器程序能连续地重用每一端的相同端口号，但这只有在服务器执行主动关闭才有效。

### 平静时间的概念
对于来自某个连接的较早替身的迟到报文段，2MSL等待可防止将它解释成使用相同插口对的新连接的一部分。但这只有在处于2MSL等待连接中的主机处于正常工作状态时才有效。
如果使用处于2MSL等待端口的主机出现故障，它会在MSL秒内重新启动，并立即使用故障前仍处于2MSL的插口对来建立一个新的连接吗？如果是这样，在故障前从这个连接发出而迟到的报文段会被错误地当作属于重启后新连接的报文段。无论如何选择重启后新连接的初始序号，都会发生这种情况。

为了防止这种情况，RFC793指出TCP在重启动后的MSL秒内不能建立任何连接。这就称为平静时间(quiettime)。只要等待 MSL 时间，就能保证不会有迟到的报文段到达，因为已经超过了报文段的最大生存时间。

只有极少的实现版遵守这一原则，因为大多数主机重启动的时间都比MSL秒要长

### FIN_WAIT_2 状态
在 `FIN_WAIT_2` 状态我们已经发出了 `FIN` ，并且另一端也已对它进行确认。除非我们在实行半关闭，否则将等待另一端的应用层意识到它已收到一个文件结束符说明，并向我们发一个`FIN`来关闭另一方向的连接。只有当另一端的进程完成这个关闭，我们这端才会从 `FIN_WAIT_2` 状态进入 `TIME_WAIT` 状态。

这意味着我们这端可能永远保持这个状态。另一端也将处于 `CLOSE_WAIT` 状态，并一直保持这个状态直到应用层决定进行关闭。

许多伯克利实现采用如下方式来防止这种在 `FIN_WAIT_2` 状态的无限等待。如果执行主动关闭的应用层将进行全关闭，而不是半关闭来说明它还想接收数据，就设置一个定时器。如果这个连接空闲10分钟75秒，TCP将进入 `CLOSED` 状态。在实现代码的注释中确认这个实现代码违背协议的规范。

## 复位报文段
TCP首部中的RST比特是用于“复位”的。一般说来，无论何时一个报文段发往基准的连接（referenced connection）出现错误，TCP都会发出一个复位报文段（这里提到的“基准的连接”是指由目的IP地址和目的端口号以及源IP地址和源端口号指明的连接。这就是为什么RFC793称之为插口）。

### 到不存在的端口的连接请求
产生复位的一种常见情况是当连接请求到达时，目的端口没有进程正在听。对于 U D P，当一个数据报到达目的端口时，该端口没在使用，它将产生一个I C M P端口不可达的信息。而T C P则使用复位。

实验, telnet 到一个没有在监听的端口： `telnet localhost 10000`，抓包结果如下：
```
tcpdump -i lo0 port 10000
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on lo0, link-type NULL (BSD loopback), snapshot length 262144 bytes

08:48:47.265605 IP6 localhost.52452 > localhost.ndmp: Flags [S], seq 900085527, win 65535, options [mss 16324,nop,wscale 6,nop,nop,TS val 3892278877 ecr 0,sackOK,eol], length 0
08:48:47.265683 IP6 localhost.ndmp > localhost.52452: Flags [R.], seq 0, ack 900085528, win 0, length 0
08:48:47.265933 IP localhost.52453 > localhost.ndmp: Flags [S], seq 1953933074, win 65535, options [mss 16344,nop,wscale 6,nop,nop,TS val 1538706092 ecr 0,sackOK,eol], length 0
08:48:47.265982 IP localhost.ndmp > localhost.52453: Flags [R.], seq 0, ack 1953933075, win 0, length 0
```

同样，看看UDP 的情况，使用 nc 向一个不存在的 UDP 服务端口发送数据 : `echo "123" | nc -u localhost 10000`, 结果如下：
```
sudo tcpdump -nn -i lo0 icmp
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on lo0, link-type NULL (BSD loopback), snapshot length 262144 bytes

08:54:31.241731 IP 127.0.0.1 > 127.0.0.1: ICMP 127.0.0.1 udp port 10000 unreachable, length 36
```

### 异常终止一个连接
终止一个连接的正常方式是一方发送FIN。有时这也称为有序释放（orderly release），因为在所有排队数据都已发送之后才发送FIN，正常情况下没有任何数据丢失。但也有可能发送一个复位报文段而不是FIN来中途释放一个连接。有时称这为异常释放（abortive release）。异常终止一个连接对应用程序来说有两个优点：
1. 丢弃任何待发数据并立即发送复位报文段；
2. RST的接收方会区分另一端执行的是异常关闭还是正常关闭。

应用程序使用的API必须提供产生异常关闭而不是正常关闭的手段。使用sock程序能够观察这种异常关闭的过程。SocketAPI通过“lingeronclose”选项（SO_LINGER）提供了这种异常关闭的能力。我们加上-L选项并将停留时间设为0。这将导致连接关闭时进行复位而不是正常的FIN。我们连接到处于服务器上的sock程序，并键入一输入行：
```
bsdi % sock -L0 svr4 8888 这是客户程序，服务器程序显示后面
hello, world 键入一行输入，它被发往到另一端
^ D 键入文件结束符，终止客户程序
```

tcpdump 的抓包结果如下：
<center><img src="pics/tcp-rst.png" alt=""></center>

在服务器上得到下面的差错信息：
```
svr4 % sock -s 8888   作为服务器进程运行，在端口 8 8 8 8监听
hello, world    这行是客户端发送的
read error: Connection reset by peer
```

### 半打开连接
如果一方已经关闭或异常终止连接而另一方却还不知道，我们将这样的TCP连接称为半打开（Half-Open）的。任何一端的主机异常都可能导致发生这种情况。只要不打算在半打开连接上传输数据，仍处于连接状态的一方就不会检测另一方已经出现异常。半打开连接的另一个常见原因是当客户主机突然掉电而不是正常的结束客户应用程序后再关机。这可能发生在使用PC机作为Telnet的客户主机上，例如，用户在一天工作结束时关闭PC机的电源。当关闭PC机电源时，如果已不再有要向服务器发送的数据，服务器将永远不知道客户程序已经消失了。当用户在第二天到来时，打开PC机，并启动新的Telnet客户程序，在服务器主机上会启动一个新的服务器程序。这样会导致服务器主机中产生许多半打开的TCP连接（在第23章中我们将看到使用TCP的keepalive选项能使TCP的一端发现另一端已经消失）。

请注意：半关闭（Half-Close） 指一个方向已经发送FIN，另一个方向仍可继续发送数据；半开连接（Half-Open Connection） 通常指一端崩溃、重启或状态丢失后，另一端仍以为连接存在。

## 服务器设计
大多数的TCP服务器进程是并发的。当一个新的连接请求到达服务器时，服务器接受这个请求，并调用一个新进程来处理这个新的客户请求。不同的操作系统使用不同的技术来调用新的服务器进程。在Unix系统下，常用的技术是使用fork函数来创建新的进程。如果系统支持，也可使用轻型进程，即线程（thread）。

我们感兴趣的是TCP与若干并发服务器的交互作用。需要回答下面的问题：当一个服务器进程接受一来自客户进程的服务请求时是如何处理端口的？如果多个连接请求几乎同时到达会发生什么情况？

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

当队列已满时，TCP将不理会传入的SYN，也不发回RST作为应答，因为这是一个软错误，而不是一个硬错误。通常队列已满是由于应用程序或操作系统忙造成的，这样可防止应用程序对传入的连接进行服务。这个条件在一个很短的时间内可以改变。但如果服务器的TCP以系统复位作为响应，客户进程的主动打开将被废弃（如果服务器程序没有启动我们就会遇到）。由于不应答SYN，服务器程序迫使客户TCP随后重传SYN，以等待连接队列有空间接受新的连接。

这个例子中有一个巧妙之处，这在大多TCP/IP的具体实现中都能见到，就是如果服务器的连接队列未满时，TCP将接受传入的连接请求（即SYN），但并不让应用层了解该连接源于何处（即不告知源IP地址和源端口）。这不是TCP所要求的，而只是共同的实现技术（如伯克利源代码通常都这么做）。如果一个API如TLI（见1.15节）向应用程序提供了解连接请求的到来的方法，并允许应用程序选择是否接受连接。当应用程序假定被告知连接请求已经到来时，TCP的三次握手已经结束！其他运输层的实现可能将连接请求的到达与接受分开（如OSI的运输层），但TCP不是这样。