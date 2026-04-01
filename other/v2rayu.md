#  代理与V2rayU
V2RayU 无法打开问题

证书过期问题导致的：

1. 执行
```sudo codesign --force --deep --sign - /Applications/V2rayU.app```
2. 在应用程序中找到V2rayU，右键，显示简介，勾选覆盖恶意软件保护(我的没有此选项)
3. 打开软件
4. 执行

```
sudo codesign --force --deep --sign - ~/.V2rayU/V2rayUTool
sudo codesign --force --deep --sign - ~/.V2rayU/v2ray-core/v2ray
```
然后就能正常运行了。
来自：https://discussionschinese.apple.com/thread/254968153

v2ray 无法使用？
删除 `~/.V2rayU` 下的所有文件，重启app


### PAC
代理自动配置（PAC）文件是一个 JavaScript 脚本，其核心是一个 JavaScript 函数，用来决定网页浏览请求（HTTP、HTTPS，和 FTP）应当直连目标地址，还是被转发给一个网页代理服务器并通过代理连接。PAC 文件中的核心 JavaScript 函数通常是这样定义的：
```
function FindProxyForURL(url, host) {
  // ...
}
```
+ url
    要访问的 URL。https:// URL 中的路径和查询组件已被去除。在 Chrome 浏览器（版本 52 至 73）中，你可以通过设置 PacHttpsUrlStrippingEnabled 为 false 来禁止这种行为，或者以 --unsafe-pac-url 命令行参数启动（自 Chrome 74 起，仅命令行参数有效，且在 Chrome 75 及之后的版本中无法禁用这种行为；至于 Chrome 81，路径剥离对 HTTP URL 不适用，但有意改变这一行为以适应 HTTPS）；在 Firefox 浏览器中，对应的选项是 network.proxy.autoconfig_url.include_path。

+ host
    从 URL 中提取得到的主机名。这只是为了方便；它与 :// 之后到第一个 : 或 / 之前的字符串相同。端口号不包括在此参数中，必要时可以自行从 URL 中提取。