#  DNS
{docsify-updated}
> https://www.ruanyifeng.com/blog/2022/08/dns-query.html  
> https://ruanyifeng.com/blog/2016/06/dns.html
> https://coredns.io/manual/toc/

<center><img src="pics/dns.png" width="60%"></center>

### 域名的树状结构
最顶层的域名是根域名（root），然后是顶级域名（top-level domain，简写 TLD），再是一级域名、二级域名、三级域名。

#### 根域名
所有域名的起点都是根域名，它写作一个点.，放在域名的结尾。因为这部分对于所有域名都是相同的，所以就省略不写了，比如example.com等同于example.com.（结尾多一个点）。  
你可以试试，任何一个域名结尾加一个点，浏览器都可以正常解读。

#### 顶级域名
根域名的下一级是顶级域名。它分成两种：通用顶级域名（gTLD，比如 .com 和 .net）和国别顶级域名（ccTLD，比如 .cn 和 .us）。  
顶级域名由国际域名管理机构 ICANN 控制，它委托商业公司管理 gTLD，委托各国管理自己的国别域名。

#### 二级域名
二级域名是一级域名的子域名，是域名拥有者自行设置的，不用得到许可。比如，es6 就是 ruanyifeng.com 的二级域名。

### 域名解析过程
DNS 解析请求可以分为两类：递归和迭代。在一个递归请求中，请求者要求服务器处理整个解析任务，包括在必要的时候与其他 DNS 服务器进行通信的任务。

下面通过一个例子来解释迭代解析。假设本地 DNS 服务器需要解析一个名字www.otago.ac.nz。要完成这个任务，它首先与每个 DNS 服务器都知道的一小组根名字服务器中的一个进行通信。（使用命令dig . NS 或从网页http://www.root-servers.org/上可以获取这组服务器列表。）给定名字 www.otago.ac.nz，根名字服务器会告诉本 DNS 服务器到其中一台 nz DNS 服务器上查询。然后本地 DNS 服务器会在 nz 服务器上查询名字 www.otago.ac.nz，并收到一个到 ac.nz 服务器上查询的响应。之后本地 DNS 服务器会在 ac.nz 服务器上查询名字www.otago.ac.nz 并被告知查询otago.ac.nz 服务器。最后本地DNS 服务器会在otago.ac.nz 服务器上查询 www.otago.ac.nz 并获取所需的 IP 地址。 

这种树状结构的意义在于，只有上级域名，才知道下一级域名的 IP 地址，需要逐级查询。  
每一级域名都有自己的 DNS 服务器，存放下级域名的 IP 地址。  
所以，如果想要查询二级域名 es6.ruanyifeng.com 的 IP 地址，需要三个步骤。
1. 查询根域名服务器，获得顶级域名服务器.com（又称 TLD 服务器）的 IP 地址。
2. 查询 TLD 服务器 .com，获得一级域名服务器 ruanyifeng.com 的 IP 地址。
3. 查询一级域名服务器 ruanyifeng.com，获得二级域名 es6 的 IP 地址。

根域名服务器全世界一共有若干台（都是服务器集群）。根域名服务器的 IP 地址是不变的，集成在操作系统里面。可在 http://www.root-servers.org/ 查看。

DNS服务器一般分为四种：
+ 根域名服务器
+ 顶级域名服务器
+ 一级域名服务器（权威域名服务器）
+ 8.8.8.8 （递归域名服务器,一般电脑系统设置的DNS服务器地址）

前三种服务器都是用来查询下一级域名服务器的IP地址的，而 8.8.8.8 这种一般是把分步骤的查询过程自动化，方便用户一次性得到结果，所以它称为`递归 DNS 服务器（recursive DNS server）`，即可以自动递归查询。我们平常说的 DNS 服务器，一般都是指递归 DNS 服务器。它把 DNS 查询自动化了，只要向它查询就可以了。它内部有缓存，可以保存以前查询的结果，下次再有人查询，就直接返回缓存里面的结果。所以它能加快查询，减轻源头 DNS 服务器的负担。

### DNS的记录类型
+ A：地址记录（Address），返回域名指向的IP地址。
+ NS：域名服务器记录（Name Server），返回保存下一级域名信息的服务器地址。该记录只能设置为域名，不能设置为IP地址。
+ MX：邮件记录（Mail eXchange），返回接收电子邮件的服务器地址。
+ CNAME：规范名称记录（Canonical Name），返回另一个域名，即当前查询的域名是另一个域名的跳转，详见下文。
+ PTR：逆向查询记录（Pointer Record），只用于从IP地址查询域名，详见下文。

### dig命令
`dig @[DNS 服务器] [域名]`

如果DNS服务器中有域名的记录，就会在输出结果中的 `ANSWER SECTION` 块中给出结果，只否则就会返回一个 `AUTHORITY SECTION`，给出下一级域名的服务器的域名。并且在`ADDITIONAL SECTION` 块中给出下一级域名服务器的 IP 地址。

### Coredns 使用
```
brew intall coredns
/opt/homebrew/etc/coredns/Corefile //配置文件路径
/opt/homebrew/var/log/coredns.log //日志文件路径
```

#### Coredns 配置
每个 "服务器区块 "的开头都包含该服务器应授权的区域。在区域名称或区域名称列表（用空格分隔）之后，服务器块用开头括号打开。服务器块用收尾括号关闭。下面的服务器块指定了负责根区以下所有区的服务器：`.`；还可以为每个服务区块配置监听的端口，默认是53。基本上，该服务器应处理所有可能的查询：
```
coredns.io:5300 {
    file db.coredns.io
}

example.io:53 {
    log
    errors
    file db.example.io
}

example.net:53 {
    file db.example.net
}

.:53 {
    kubernetes
    forward . 8.8.8.8
    log
    errors
    cache
}
```

### MacOS DNS配置查看
```
查看
scutil --dns

配置
networksetup -getdnsservers Wi-Fi
networksetup -setdnsservers Wi-Fi 10.176.161.63 127.0.0.1 8.8.8.8
```


