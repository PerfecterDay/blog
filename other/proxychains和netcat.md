# ProxyChains 和 netcat
{docsify-updated}

> https://mp.weixin.qq.com/s/n7ZeWXIpchZ0_QGLZdrfpg

- [ProxyChains 和 netcat](#proxychains-和-netcat)
  - [ProxyChains](#proxychains)
    - [proxychain 安装](#proxychain-安装)
    - [使用配置](#使用配置)
  - [netcat](#netcat)


## ProxyChains
Proxychains.exe 是一个适用于 Win32(Windows) 和 Cygwin 平台的命令行强制代理工具（Proxifier）。它能够截获大多数 Win32 或 Cygwin 程序的 TCP 连接，强制它们通过一个或多个 SOCKS5 代理隧道。

Proxychains.exe 通过给动态链接的程序注入一个 DLL，对 Ws2_32.dll 的 Winsock 函数挂钩子的方式来将应用程序的连接重定向到 SOCKS5 代理。
警告：此程序只对动态链接的程序有用。同时，Proxychains.exe 和需要运行的目标程序必须是同一架构和平台（用 proxychains_x86.exe 运行 x86 程序，用 proxychains_x64.exe 运行 x64 程序；用 Cygwin 下构建的版本来运行 Cygwin 程序）。

使用场景就是当我们希望某个程序需要访问某些网络，但是由于各种原因必须使用代理的时候，就可以使用这个程序。通过配置文件，可以对指定的 IP/域名 配置代理/直连模式。但是该程序本身并不提供代理服务。也就是说，它能控制的是对某个域名是直连还是走代理。至于，代理服务器（如VPN）则需要自己搭建。

### proxychain 安装
`scoop install proxychain`

### 使用配置

1. Proxychains.exe 按照以下顺序寻找配置：
    在 Win32 环境中:
        + 环境变量 %PROXYCHAINS_CONF_FILE% 或通过 -f 命令行参数指定的文件
        + %USERPROFILE%\.proxychains\proxychains.conf（Win32 用户主目录）
        + (CSIDL_APPDATA)\Proxychains\proxychains.conf（在现代 Windows 版本中，典型的路径如 C:\Users\<用户名>\AppData\Roaming\Proxychains\proxychains.conf）
        + (CSIDL_COMMON_APPDATA)\Proxychains\proxychains.conf（在现代 Windows 版本中，典型的路径如 C:\ProgramData\Proxychains\proxychains.conf）
    在 Macos ：
        `/opt/homebrew/etc/proxychains.conf`
  
2. `proxychains -l V -f D:\applications\Scoop\apps\proxychains\0.6.8\proxychains.conf curl https://www.google.com.hk/`
3. `proxychains4 brew update && brew upgrade`

## netcat
nc -l -t localhost 91111