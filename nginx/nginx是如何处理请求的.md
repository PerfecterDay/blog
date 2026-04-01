
# nginx是如何处理请求的
{docsify-updated}



# nginx是如何处理请求的
--
## 基于名字的虚拟主机
首先，nginx会决定哪个server上下文处理请求。server_name指令为每个server上下文指定一个名字。
如下配置文件：

    server {
        listen      80;
        server_name example.org www.example.org;
        ...
    }

    server {
        listen      80;
        server_name example.net www.example.net;
        ...
    }

    server {
        listen      80;
        server_name example.com www.example.com;
        ...
    }
上述配置文件中，nginx会比较请求头"Host"和server_name的参数来决定将请求交给哪个server处理。如果，没有匹配到任何server上下文或者请求中没有HOST请求头，nginx会将请求转发给默认server处理。上述配置中第一个server是默认server。nginx中的默认server就是第一个server。要显示的声明一个默认server，在listen命令中使用default_server参数：

    server {
        listen      80 default_server;
        server_name example.net www.example.net;
        ...
    }

## 拒绝没有Host头的请求

    server {
        listen      80;
        server_name "";
        return      444;
    }
server_name设置为空将会匹配到没有"Host"头的请求，nginx将会返回444响应码并关闭连接。

## 混合名字和IP的虚拟主机
如下配置文件：

    server {
        listen      192.168.1.1:80;
        server_name example.org www.example.org;
        ...
    }

    server {
        listen      192.168.1.1:80;
        server_name example.net www.example.net;
        ...
    }

    server {
        listen      192.168.1.2:80;
        server_name example.com www.example.com;
        ...
    }
nginx首先会匹配IP和端口，然后在IP和端口匹配的server块中进一步匹配server_name，如果没有匹配到正确的server_name,nginx会将请求交给默认server处理。在上述配置文件中，假如在 192.168.1.1:80 收到一个www.example.com的请求，最终会由第一个server处理该请求。因为第一个换个第二个server中都没有www.example.com的server_name参数配置。
如果改成如下配置：

    server {
        listen      192.168.1.1:80;
        server_name example.org www.example.org;
        ...
    }

    server {
        listen      192.168.1.1:80 default_server;
        server_name example.net www.example.net;
        ...
    }

    server {
        listen      192.168.1.2:80 default_server;
        server_name example.com www.example.com;
        ...
    }
则上述请求会被第二个server处理。

