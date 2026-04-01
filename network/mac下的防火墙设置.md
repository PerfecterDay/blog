# MacOs 的防火墙配置

## pfctl
```
编辑配置文件：
sudo vim /etc/pf.conf

添加：
block drop proto tcp from any to app.allfunds.com

应用规则：
sudo pfctl -ef /etc/pf.conf
```

+ `pfctl -e` 启用 pf
+ `pfctl -d` 禁用 pf
+ `pfctl -f /etc/pf.conf` 载入配置文件
+ `pfctl -nf /etc/pf.conf` 解析文件，但不载入
+ `pfctl -Nf /etc/pf.conf` 只载入文件中的NAT规则
+ `pfctl -Rf /etc/pf.conf` 只载入文件中的过滤规则
+ `pfctl -sn` # 显示当前的NAT规则
+ `pfctl -sr` # 显示当前的过滤规则
+ `pfctl -ss` # 显示当前的状态表
+ `pfctl -si` # 显示过滤状态和计数
+ `pfctl -sa` # 显示任何可显示的

### 配置文件
pf.conf文件有7个部分：

+ 宏：用户定义的变量，包括IP地址，接口名称等等。
+ 表：一种用来保存IP地址列表的结构。
+ 选项：控制PF如何工作的变量。
+ 整形：重新处理数据包，进行正常化和碎片整理。
+ 排队：提供带宽控制和数据包优先级控制。
+ 转换：控制网络地址转换和数据包重定向。
+ 过滤规则：在数据包通过接口时允许进行选择性的过滤和阻止。

空行会被忽略，以#开头的行被认为是注释。

