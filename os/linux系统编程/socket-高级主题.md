# socket-高级主题
{docsify-updated}

## 零拷贝传输：sendfile()系统调用 
像 Web 服务器和文件服务器这样的应用程序常常需要将磁盘上的文件内容不做修改地通过（已连接）套接字传输出去。一种方法是通过循环按照如下方式处理：
```
while((n =read(diskfilefd,buf,BUF_SIZE)) > 0){
	write(sockfd,buf,n)
}
```
对于许多应用程序来说，这样的循环是完全可接受的。但是，如果我们需要通过套接字频繁地传输大文件的话，这种技术就显得很不高效。为了传输文件，我们必须使用两个系统调用（可能需要在循环中多次调用）：一个用来将文件内容从内核缓冲区 cache 中拷贝到用户空间，另一个用来将用户空间缓冲区拷贝回内核空间，以此才能通过套接字进行传输。图61-1左侧展示了这种场景。如果应用程序在发起传输之前根本不对文件内容做任何处理的话，那么这种两步式的处理就是一种浪费。

系统调用 sendfile()被设计为用来消除这种低效性。如图61-1 右侧所示，当应用程序调用sendfile()时，文件内容会直接传送到套接字上，而不会经过用户空间。这种技术被称为零拷贝传输（zero-copy transfer）。 

<center><img src="pics/zero-copy.png" width="40%"></center>

```
#include <sys/sendfile.h>
ssize t sendfile(int out_fd,int in_fd, off_t *offset, size_t count);
```
系统调用 sendfile()在代表输入文件的描述符 in_fd 和代表输出文件的描述符 out_fd 之间传送文件内容（字节）。描述符 out_fd 必须指向一个套接字。参数 in_fd 指向的文件必须是可以进行 mmap()操作的。在实践中，这通常表示一个普通文件。这些局限多少限制了sendfile()的使用。我们可以使用 sendfile()将数据从文件传递到套接字上，但反过来就不行。另外，我们也不能通过 sendfile()在两个套接字之间直接传送数据。 

## shutdown()系统调用
在套接字上调用`close()`会将双向通信通道的两端都关闭。有时候，只关闭连接的一端也是有用处的，这样数据只能在一个方向上通过套接字传输。系统调用`shutdown()`提供了这种功能。
```
#include <sys/socket.h>
int shutdown (int sockfd, int how);
```
系统调用 shutdown()可以根据参数 how 的值选择关闭套接字通道的一端还是两端。参数 how 的值可以指定为如下几种:
+ SHUT_RD   
  关闭连接的读端。之后的读操作将返回文件结尾(0)。数据仍然可以写入到套接字上。在 UNIX 域流式套接字上执行了 SHUT_RD 操作后，对端应用程序将接收到一个 SIGPIPE 信 号，如果继续尝试在对端套接字上做写操作的话将产生 EPIPE 错误。如 61.6.6 节中讨论的， SHUT_RD 对于 TCP 套接字来说没有什么意义。
+ SHUT_WR
  关闭连接的写端。一旦对端的应用程序已经将所有剩余的数据读取完毕，它就会检测到文件结尾。后续对本地套接字的写操作将产生 SIGPIPE 信号以及 EPIPE 错误。而由对端写入的数据仍然可以在套接字上读取。换句话说，这个操作允许我们在仍然能读取对端发回给我 们的数据时，通过文件结尾来通知对端应用程序本地的写端已经关闭了。SHUT_WR 操作在 ssh 和 rsh 中都有用到(参见[Stevens，1994]中的 18.5 节)。在 shutdown()中最常用到的操作就是 SHUT_WR，有时候也被称为半关闭套接字。
+ SHUT_RDWR
	将连接的读端和写端都关闭。这等同于先执行 SHUT_RD，跟着再执行一次 SHUT_WR操作。

除了参数 how 的语义之外，shutdown()同 close()之间的另一个重要区别是:无论该套接字上是否还关联有其他的文件描述符，shutdown()都会关闭套接字通道。(换句话说，shutdown()是根据 打开的文件描述(open file description)来执行操作，而同文件描述符无关。见图 <a href="#/os/linux系统编程/文件IO#file">5-1</a> 。)  
例如，假设sockfd 指向一个已连接的流式套接字，如果执行下列调用，那么连接依然会保持打开状态，我们仍然可以通过文件描述符fd2在该连接上做I/O 操作:
```
int fd2 = dup(sockfd);
close(sockfd);
```
但是，如果我们执行如下的调用，那么该连接的双向通道都会关闭，通过fd2 也无法再执行 I/O 操作了：
```
int fd2 = dup(sockfd);
shutdown(sockfd,SHUT_RDWR);
```

需要注意的是，shutdown()并不会关闭文件描述符，就算参数how 指定为SHUT_RDWR 时也是如此。要关闭文件描述符，我们必须另外调用 close()。

## 专用于套接字的 I/O 系统调用：recv()和 send() 