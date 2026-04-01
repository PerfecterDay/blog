# java基础-网络编程基础
{docsify-updated}

## Java的基本网络支持

### IP地址：InetAddress
Java 使用 `InetAddress` 来代表 IP 地址， `InetAddress` 有两个子类： `Inet4Address` 和 `Inet6Address` 。分别代表 IPv4 和 IPv6 地址。

`InetAddress` 没有提供构造器，而是提供了如下两个静态方法来获取 `InetAddress` 对象：
+ `InetAddress getByName(String name)` ：根据主机域名获取对应 `InetAddress` 对象；
+ `InetAddress[] getAllByName(String name)`: 有时候一个域名对应许多个IP，这个方法能返回对应的所有IP
+ `InetAddress getByAdress(byte[] addr)`: 根据原始 IP 地址获取 `InetAddress` 对象；
+ `InetAddress getLocalHost()`: 获取本机IP地址对应的 `InetAddress` 对象;

当获取了一个 `InetAddress` 对象后，可以使用下述方法获取该 `InetAddress` 对象的相关信息：
+ `String getCanonicalHostName()`: 获取IP地址的全限定域名
+ `String getHostAddress()`: 返回地址实例对应的 IP 地址字符串
+ `String getHostName()`: 返回该 `InetAddress` 实例的主机名
+ `boolean isReachable(int timeout)`: 测试该地址的主机是否可达

### URLEncoder 和 URLDecoder
`URLEncoder` 和 `URLDecoder` 提供了静态的工具方法 `encode` 和 `decode` 来进行 URL 的编码和解码。

### URL、URLConnection 和 URLPermission
URL 对象代表统一资源定位器，通常而言URL由协议名、主机、端口、路径和查询参数组成：  
`protocol://host:port/resource?queryParam1=value1&queryParam2=value2...`

URL类提供了多个构造器用于创建URL对象，一旦构建一个URL对象，可以使用它提供的方法来获取资源：
1. `String getFile()`: 获取URL的资源名
2. `String getHost()`: 获取URL的主机名
3. `String getPath()`: 获取URL的路径
4. `int getPort()`: 获取URL的端口号
5. `String getQuery()`: 获取URL的查询参数
6. `URLConnection openConnection()`: 打开一个到 URL 远程资源的连接，返回一个 URLConnection 对象
7. `InputStream openStream()`: 打开一个到 URL 远程资源的连接，并返回一个读取 URL 资源的 InputStream 对象

## 基于TCP的网络编程
1. 使用 `ServerSocker` 创建TCP服务器端

	Java使用 `ServerSocket` 来创建服务端用于监听客户端连接的套接字。有如下构造方法：
	+ `ServerSocket(int port)`：使用指定端口创建一个 `ServerSocket` ，端口应该在 `0～65535` 之间。
	+ `ServerSocket(int port, int backlog)`:除了端口，增加一个指定连接队列长度的 `backlog` 参数
	+ `ServerSocket(int port, int backlog,InetAddress localAddress)`:当主机有多个IP地址时，可以指定ServerSocket绑定到的IP地址。
  
	一旦构造了 `ServerSocket` 对象后，该对象会在指定端口监听连接请求，使用 `accept()` 方法接受来自客户端的请求，如果接收到了客户端请求，方法将会返回一个与客户端 `Socket` 连接的 `Socket` 对象；如果没有没有客户端请求，该方法将会阻塞线程。当 `ServerSocket` 对象使用完毕，应该使用 `close()` 方法关闭 `ServerSocket` 。

2. 使用Socket进行通信

	客户端通常可以使用 Socket 来连接指定服务器， Socket 通常有以下两个构造器：
	+ `Socket(InetAddress/String remoteAddress, int remotePort)`: 创建连接到指定的主机和端口的 Socket，默认使用本地主机的默认IP和随机分配的端口
	+ `Socket(InetAddress/String remoteAddress, int remotePort, InetAddress localAddr, int localPort)`:额外指定本地的IP和端口，适用于本地主机有多个IP的情况。
	+ `Socket(Proxy proxy)`：使用指定代理服务器创建一个没有连接的Socket对象

	当客户端和服务器端创建了对应的Socket之后，可以使用 Socket 提供的下述两个方法来获取输入/输出流从而进行通信：
	1. `InputStream getInputStream()`: 获取输入流，用于读取数据
	2. `OuputStream getOutputStream()`: 获取输出流，用于发送数据
	3. `setSoTimeout(int timeouut)`:设置socket **读写数据的超时时间**，读写数据超过该时长将会抛出 SocketTimeoutException.
	4. 如果要设置**连接的超时时长**，则必须新建一个无连接的 Socket，然后调用 `connect(InetSocketAddress remoteAddress,int timeout)`来指定连接超时时间：
	```
	Socket socket = new Socket(); 
	socket.connect(new InetSocketAddress("testHost",8080),1000)
	//使用socket 发送 HTTP demo
	public static void main(String[] args) throws IOException {
		//Socket s = SSLSocketFactory.getDefault().createSocket();
        //s.connect(new InetSocketAddress("www.baidu.com", 443), 2000);

        Socket s = new Socket();
        s.connect(new InetSocketAddress("127.0.0.1", 8500), 2000);
        PrintWriter wtr = new PrintWriter(s.getOutputStream());

        //Prints the request string to the output stream
        wtr.println("GET / HTTP/1.1");
        wtr.println("Host: localhost");
        wtr.println("");
        wtr.flush();

        BufferedReader bufRead = new BufferedReader(new InputStreamReader(s.getInputStream()));
        String outStr;

        //Prints each line of the response
        while((outStr = bufRead.readLine()) != null){
            System.out.println(outStr);
        }

        //Closes out buffer and writer
        bufRead.close();
        wtr.close();
    }
	```

## 基于UDP的网络编程

### 使用 DatagramSocket 发送、接收 UDP 数据

### 使用 MulticastSocket 实现多点广播

## 使用代理服务器
JDK1.5 开始，提供了 Proxy 和 ProxySelector 两个类，其中 Proxy 代表一个代理服务器，可以在打开 URLConnection 连接时指定所用的 Proxy 实例，也可以在创建 Socket 连接时指定Proxy 实例。而 ProxySelector 代表一个代理选择器，它提供了对代理服务器更加灵活的控制，可以对不同协议如 HTTP、HTTPS、FTP、SOCKS等分别设置代理，还以为某些地址设置不走代理。

### Proxy
Proxy 有一个构造器： Proxy(Proxy.Type type, SocketAddress sa)，sa指定代理服务器的地址，type指定代理服务器的类型，类型有如下三种：
1. `Proxy.Type.DIRECT`： 表示直连
2. `Proxy.Type.HTTP`： 表示高级协议的代理，如HTTP或FTP
3. `Proxy.Type.SOCKS`： 表示 SOCKS（V4或V5)代理

一旦创建了 Proxy 对象，就可以在打开 URLConnection 或者创建 Socket 时使用了：
+ URL的 `URLConnection openConnection(Proxy proxy)`方法
+ Socket 的构造方法： `Socket(Proxy proxy)`

### ProxySelector
基于 Proxy 的代理，需要在每次打开 URLConnection 或者 Socket 时指定代理服务器，如果想让系统在每次打开连接时都是用相关的代理服务器，而不用显示的为每个连接指定，则可以使用 ProxySelector ，它可以为不同的连接指定不同的代理服务器。

系统提供了默认的 ProxySelector 子类作为代理选择器，默认的 ProxySelector 会检测各种系统属性和URL协议，然后决定是否使用指定的代理服务器，程序可以使用 System 类来设置系统的代理服务器属性，关于代理服务器常用的属性有以下几个：
1. http.proxyHost: 设置 HTTP 访问时的代理服务器主机地址，HTTP前缀可以更改为HTTPS/FTPHTTPS/FTP等，下同
2. http.proxyPort: 设置 HTTP 访问时的代理服务器连接端口
3. http.nonProxyHosts: 设置不需要代理的主机地址，连接这些地址时不需要代理，多个主机地址使用 | 分隔，可以使用通配符 * 。

通过系统属性来配置代理服务器适用于简单的代理配置，如果有更加灵活的配置需求，如连接指定主机时适用指定代理，通过系统属性无法完成这一需求。因此，开发者可以通过继承 ProxySelector 子类实现自己的代理选择器，需要重写两个方法：
1. `List<Proxy> select(URI uri)` ： 该方法让代理选择器根据不同的 URI 自定义的选择代理服务器；
2. `void connectFailed(URI uri, SocketAddress sa, IOException ioe)` : 当连接代理服务器失败时，会调用此方法

实现了自定义的 ProxySelector 后，还需要将其设置为默认的代理选择器才会生效， ProxySelector 提供了下述静态方法来获取和设置默认的 ProxySelector：
+ `static ProxySelector getDefault()`: 获取默认的 ProxySelector
+ `static void setDefault(ProxySelector ps)`： 设置默认的 ProxySelector


## 实现请求代理转发功能
```
package com.minigod.controller.user;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.net.URI;
import java.net.URISyntaxException;
import java.util.Enumeration;

@RestController
@RequestMapping("/app/user")
public class ProxyController {

    @Value("${app.user.url}")
    private String url;

    @PostMapping("/**")
    public ResponseEntity<String> proxy(@RequestBody String body, HttpServletRequest servletRequest, HttpServletResponse response) {
        try {
            HttpMethod method = HttpMethod.resolve(servletRequest.getMethod());
            String uriStr = servletRequest.getRequestURI().replace("/app/user", "");
            URI uri = new URI(url + uriStr);

            HttpHeaders headers = new HttpHeaders();
            Enumeration<String> headerNames = servletRequest.getHeaderNames();

            while (headerNames.hasMoreElements()) {
                String headerName = headerNames.nextElement();
                headers.set(headerName, servletRequest.getHeader(headerName));
            }
            headers.remove(HttpHeaders.ACCEPT_ENCODING); // 不加这行在gzip 下会有问题

            HttpEntity<String> httpEntity = new HttpEntity<>(body, headers);
            RestTemplate restTemplate = new RestTemplate();
            ResponseEntity<String> serverResponse = restTemplate.exchange(uri, method, httpEntity, String.class);
            return serverResponse;

        } catch (URISyntaxException e) {
            throw new RuntimeException(e);
        }
    }
}
```