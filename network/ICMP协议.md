# ICMP(Internet Control Message Protocol)协议
{docsify-updated}

- [ICMP(Internet Control Message Protocol)协议](#icmpinternet-control-message-protocol协议)
      - [ICMP报文格式](#icmp报文格式)

#### ICMP报文格式
<center><img src="pics/icmp.png" alt="" width="40%"></center>

类型字段可以有 15个不同的值，以描述特定类型的 ICMP报文。某些ICMP报文还使用代码字段的值来进一步描述不同的条件。

检验和字段会检验整个ICMP报文。

ICMP报文的种类有两种：**ICMP查询报文**和**ICMP差错报告报文**。

ICMP的具体类型如下：
<center><img src="pics/icmp_type.png" alt="" width="40%"></center>

常用的ICMP差错报文主要有上述的五大类：

1. 目的不可达
2. 源端被关闭
3. 重定向
4. 超时
5. 参数问题

常用 ICMP 询问报文有两种：
1. 回送请求和回答（type 8 和 0）

   ICMP回送请求报文是由主机或路由器向一个特定的目的主机发出的询问。收到此报文的主机必须给源主机或路由器发送ICMP回送回答报文。这种询问报文用来测试目的站是否可达以及了解其有关状态。
2. 时间戳请求和回答（type 13 和 14）

   ICMP时间戳请求报文是请求某个主机或路由器回答当前的日期和和时间。这种报文可用来进行始终同步和测量时间。