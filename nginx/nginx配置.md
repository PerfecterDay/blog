# nginx基本配置、负载均衡与反向代理
{docsify-updated}

## nginx基本配置
通常来说，nginx配置文件包含若干个server上下文，不同的server上下文通过监听的端口和名字来区分。一旦nginx决定了用哪个server上下文来处理请求，nginx就会用请求URI与server块指令中配置的location指令的参数相比较，一旦匹配到某个location的参数，URI就会被添加到location块内的root指令参数后边，构成请求的静态内容的访问路径。如果有多个location匹配，nginx选取最长的匹配。

基本配置：
```
user www www;
worker_processes 10; 

events {
	use epoll;  # 设置使用 epoll IO 网络模型，Linux推荐 epoll，FreeBSD 推荐kqueue
	worker_connections  1024; #允许的连接数
}

http{

	# 虚拟主机1
	server {
		# 监听的IP和端口，如果省略IP则监听主机所有网卡的端口
		listen 192.168.0.1 80; 
		server_name www.mysite.com mysite.com; #匹配的域名，请求的域名与这个相同则用该虚拟主机处理请求

		access_log logs/access_log.log combined; #访问日志存放路径,combined设置日志使用的格式名称
		location / {
			root /server1/htdocs; # HTML 文件存放的目录
			index index.html index.htm; #默认首页文件，如果找不到index.html，就找index.htm，从左到右以此类推
		}

		location ~ \.(gif|jpg|png)$ {
			root html/images;
		}

		location /app{
			proxy_pass http://localhost:8080/;
		}
	}

	#虚拟主机2
	server {
		# 监听的IP和端口
		listen 8080;
		server_name bbb.com;

		root html/proxy;

		location / {
		}
	}
}
```
1. localtion 指令
	语法：location[=|~|~*|^~]/uri{}
	该指令允许对不同的URI进行不同的配置，既可以使用字符串，也可以使用正则表达式，使用正则表达式，必须使用以下前缀：
		+ ～* ： 表示不区分大小写的匹配
		+ ～：区分大小写的匹配
    在匹配过程中，nginx **首先匹配字符串，然后匹配正则表达式。匹配字符串以最长匹配到的字符串为结果；正则匹配时，匹配到第一个正则表达式后就会停止搜索，后边的正则匹配将不再进行，所以正则匹配与配置文件中的顺序有关。如果匹配到正则表达式，就使用正则表达式的结果，否则使用字符串匹配的结果。**
	使用`^~`可以禁止优先使用正则匹配的行为，如果字符串匹配的最长结果是以`^~`开头，则会停止正则匹配过程，直接使用字符串匹配的结果。
	nginx默认的是前缀匹配，也就是只要URI中包含了待匹配的内容就会成功匹配，使用`=`可以进行**精确匹配**，例如`location=\`只会匹配'\'，而`\test.html`不会匹配。精确匹配即使不是最长匹配也会被立即使用，剩余字符串和正则匹配都会停止。
2. max_fails
	`max_fails` 指令设置在 `fail_timeout` 时间内，通讯失败的次数。如果被设置为0，表示关闭健康检查功能，默认值是1。如果设置为2，则表示在 `fail_timeout`时间内，2次没有与服务器正常通信，则标记该服务器为非正常服务器。然后每隔 `fail_timeout` 时间就检测一次服务器状态，如果能正常通信，则标记为存活服务器。

## Nginx 负载均衡
Nginx 负载均衡使用 upstream 指令来配置，支持以下几种负载均衡算法：  
+ round-robin :轮询算法——轮流将请求发送至各个服务器，nginx的默认算法
+ least-connected :最少连接算法——将请求转发至活动连接最少的服务器
+ ip-hash :算法——基于客户端的请求IP，使用hash算法决定将请求转发至哪台服务器

```
user www www;
worker_processes 10; 

events {
	use epoll;  # 设置使用 epoll IO 网络模型，Linux推荐 epoll，FreeBSD 推荐kqueue
	worker_connections  1024; #允许的连接数
}

http{

	# 虚拟主机1
	server {
		# 监听的IP和端口，如果省略IP则监听主机所有网卡的端口
		listen 192.168.0.1 80; 
		server_name www.mysite.com mysite.com; #匹配的域名，请求的域名与这个相同则用该虚拟主机处理请求

		access_log logs/access_log.log combined; #访问日志存放路径,combined设置日志使用的格式名称

		upstream backend1{
			ip_hash; # iphash 负载算法
			server 192.168.0.1:8000 weight=4 max_fails=3 fail_timeout=30s; #max_fails 设置失败次数
			server example.com:8000 weight=5 max_fails=3 fail_timeout=30s;
			server 192.168.2:8000 down; # 标记服务器为永久离线状态，配合ip_hash 使用
			server 192.168.2:8000 backup; # 仅在其它非backup 机器全宕机后使用
			server unix:/tmp/backend; 
		}

		upstream backend2{
			least_conn; # least_conn 负载算法
			server 192.168.0.1:8000 weight=4 max_fails=3 fail_timeout=30s; #max_fails 设置失败次数
			server example.com:8000 weight=5 max_fails=3 fail_timeout=30s;
			server 192.168.2:8000 backup; # 仅在其它非backup 机器全宕机后使用
			server unix:/tmp/backend; 
		}

		# 路径匹配
		location /app1 {
			proxy_pass https://backend1;
		}

		# 路径匹配
		location /app2 {
			proxy_pass https://backend2;
		}

		location ~ \.(gif|jpg|png)$ {
			root html/images;
		}

		location /app{
			proxy_pass http://localhost:8080/;
		}
	}

	#虚拟主机2
	server {
		# 监听的IP和端口
		listen 8080;
		server_name bbb.com;

		root html/proxy;

		location / {
		}
	}
}
```

## Nginx 反向代理

nginx一个最常用的功能就是作为一个反向代理服务器，即收到客户端的请求后，将请求转发到被代理的服务器上，收到被代理的服务器响应后，将其发送到客户端。 反向代理一般**只会代理集群内部的一些服务器，而正向代理则可以代替客户端去请求各个互联完资源（VPN翻墙），它没有配置请求转发到一组特定的服务器上。** 
nginx转发指令是 `proxy_pass` ，参数转发的完整地址，包括协议、url和端口号。

```
stream { 
    upstream stream_backend { 
        server srv1.example.com weight=75;
        server srv2.example.com weight=25; 
    } 
    server { 
        listen 80; 
        proxy_pass stream_backend; 
    } 
}

# 路径匹配
location /app1 {
	proxy_pass https://backend1;
	proxy_set_header X-real-ip $remote_addr;
	proxy_set_header Host $proxy_host;
	proxy_next_upstream http_500 http_404 error timeout invalid_header;
	proxy_http_version 1.1;
}
```

### 正向代理与反向代理的区别
1. 反向向代理通常只能代理若干服务器的服务，nginx 配置中配置了多个 location -> proxy_pass，用户到特定服务的访问会被反向代理代理掉。但是正向代理往往并不限制用户对服务的访问（或者说能代理用户所有的访问流量），然后代替用户去访问目标服务器。

<center><img src="pics/proxy-reverseproxy.png" width="50%"></center>


## nginx 自定义错误页面隐藏版本信息
```
http {
    server_tokens off;
}

server{
	error_page 404 /404.html;
	location = /404.html {
	root /usr/share/nginx/html;
	}
}
```

## 添加响应头
```
# 路径匹配
location /app1 {
	proxy_pass https://backend1;
	add_header Access-Control-Allow-Origin *;
	add_header Access-Control-Allow-Headers X-Requested-With;
	add_header Access-Control-Allow-Methods PUT,POST,GET,DELETE,OPTIONS;
}
```
add_header 只对 200,201，204,206,301,302,303,304,307 这些状态码生效，对于 401 405 403 这些状态码是不生效的。
解决办法，加上 alwasy 代表总是返回头:
```
# 路径匹配
location /app1 {
	proxy_pass https://backend1;
	add_header Access-Control-Allow-Origin * always;
	add_header Access-Control-Allow-Headers X-Requested-With always;
	add_header Access-Control-Allow-Methods PUT,POST,GET,DELETE,OPTIONS always;
}
```

## 跟踪重定向
```
proxy_redirect    http://upstream:port/    http://$http_host/;
```