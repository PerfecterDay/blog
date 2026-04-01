# SOCKET-Internet Domain
{docsify-updated}

## /etc/services 文件 
众所周知的端口号是由 IANA 集中注册的，其中每个端口都有一个对应的服务名。由于服务号是集中管理并且不会像IP 地址那样频繁变化，因此没有必要采用DNS 服务器来管理它们。相反，端口号和服务名会记录在文件/etc/services 中。getaddrinfo()和getnameinfo()函数会使用这个文件中的信息在服务名和端口号之间进行转换。  

<center><img src="pics/services-file.png" width="40%"></center>