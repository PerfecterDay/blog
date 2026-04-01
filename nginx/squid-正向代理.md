# squid 正向代理安装使用
{docsify-updated}

linux/WSL 下安装
```
suod apt update && sudo apt install sqid
```

配置：
```
vim  /etc/squid/squid.conf
```

修改以下配置：
```
# http_access deny CONNECT !SSL_ports 注释掉
# http_access deny !Safe_ports 注释掉
http_access allow localnet
```

启动或重启服务：
```
sudo service squid start 
sudo service squid restart 
```