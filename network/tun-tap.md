# TUN 
{docisyf-updated}

> https://www.baeldung.com/linux/tun-interface-purpose

在计算机网络领域，无需硬件适配器支持的虚拟设备为我们开辟了新的可能性。利用这一概念的一种方式是通过 TUN——这是内核中的一种网络接口，它使我们能够在操作系统中建立虚拟连接。

## TUN 是什么
TUN（ `network TUNnel`-网络隧道 ）的缩写是一种虚拟接口，它通过模拟以太网或Wi-Fi网卡等物理设备的行为，实现了对网络的软件抽象。它在OSI模型的第3层运行，负责IP数据包的发送和接收。因此，与同样在第2层运行的另一种接口 `TAP` 不同， `TUN` 与以太网帧和MAC地址没有任何关联。

与TAP不同，TUN不具备连接不同局域网的能力。不过，TUN可用于通过隧道路由流量，因此适合用于VPN服务。

从本质上讲，TUN 提供了一个接口，供用户空间应用程序处理原始网络流量。程序可以挂载到该网络接口上。因此，它们可以像操作物理接口一样，从该接口读取数据并向其写入数据。

## 创建一个 TUN
有多种工具可用于创建 TUN 接口，包括 `tunctl` 和 `OpenVPN` 。以下示例将使用 Linux 的 `ip` 命令行工具及其 `tuntap` 接口类型。

首先，我们创建一个名为 tun0 的 TUN 设备，配置其 IP 地址和子网掩码，并使用 sudo 权限将其启用：
```
sudo ip tuntap add dev tun0 mode tun
sudo ip addr add 10.0.0.1/24 dev tun0
sudo ip link set up dev tun0
```

然后查看接口详情：
```
ip addr show
...
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:37:cd:ab:f8 brd ff:ff:ff:ff:ff:ff
    inet 10.0.4.11/24 brd 10.0.4.255 scope global dynamic noprefixroute enp0s3
       valid_lft 86263sec preferred_lft 86263sec
    inet6 fe82::3597:47e3:1e77:5a3c/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: tun0: <NO-CARRIER,POINTOPOINT,MULTICAST,NOARP,UP> mtu 1500 qdisc fq_codel state DOWN group default qlen 500
    link/none 
    inet 10.0.0.1/24 scope global tun0
       valid_lft forever preferred_lft forever
```

## 测试程序
```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <linux/if.h>
#include <linux/if_tun.h>

#define IFNAMSIZ 16

int main() {
  int tun_fd = open("/dev/net/tun", O_RDWR);
  struct ifreq ifr;
  memset(&ifr, 0, sizeof(ifr));
  ifr.ifr_flags = IFF_TUN | IFF_NO_PI;
  strcpy(ifr.ifr_name, "tun0");
  ioctl(tun_fd, TUNSETIFF, (void *)&ifr);

  while (1) {
    char buffer[1500];
    int nread = read(tun_fd, buffer, sizeof(buffer));
    printf("Read %d bytes from device %s\n", nread, ifr.ifr_name);
  }

  close(tun_fd);
  return 0;
}
```

编译并运行上述程序，然后使用 `ping/tcpdump` 测试：
```
[Terminal 1]
./tun_program 
Read 84 bytes from device tun0
Read 84 bytes from device tun0
Read 84 bytes from device tun0
Read 84 bytes from device tun0
Read 84 bytes from device tun0
Read 84 bytes from device tun0

[Terminal 2]
sudo tcpdump -i tun0 -v
tcpdump: listening on tun0, link-type RAW (Raw IP), capture size 262144 bytes
23:36:25.137290 IP6 (hlim 255, next-header ICMPv6 (58) payload length: 8) baeldung > ip6-allrouters: [icmp6 sum ok] ICMP6, router solicitation, length 8
23:36:26.064591 IP (tos 0x0, ttl 64, id 3637, offset 0, flags [DF], proto ICMP (1), length 84)
    10.0.0.1 > 10.0.0.2: ICMP echo request, id 4, seq 1, length 64
23:36:27.088521 IP (tos 0x0, ttl 64, id 3843, offset 0, flags [DF], proto ICMP (1), length 84)
    10.0.0.1 > 10.0.0.2: ICMP echo request, id 4, seq 2, length 64
23:36:28.115681 IP (tos 0x0, ttl 64, id 3851, offset 0, flags [DF], proto ICMP (1), length 84)
    10.0.0.1 > 10.0.0.2: ICMP echo request, id 4, seq 3, length 64
23:36:29.138589 IP (tos 0x0, ttl 64, id 3924, offset 0, flags [DF], proto ICMP (1), length 84)
    10.0.0.1 > 10.0.0.2: ICMP echo request, id 4, seq 4, length 64
23:36:30.163580 IP (tos 0x0, ttl 64, id 4006, offset 0, flags [DF], proto ICMP (1), length 84)
    10.0.0.1 > 10.0.0.2: ICMP echo request, id 4, seq 5, length 64
6 packets captured
6 packets received by filter
0 packets dropped by kernel

[Terminal 3]
ping 10.0.0.2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
--- 10.0.0.2 ping statistics ---
6 packets transmitted, 0 received, 100% packet loss, time 4705ms
```