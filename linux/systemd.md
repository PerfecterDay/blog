#  Systemd
{docsify-updated}

> https://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html  
> https://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-part-two.html


1. 创建一个Systemd服务单元文件,即在 `/etc/systemd/system/` 目录下创建一个以 `demo-thrift-proxy.service` 的文件
2. 修改 `/root/demo-thrift-proxy/start.sh` ：加上 `cd /root/demo-thrift-proxy` 。 systemd 默认的执行路径不是脚本所在的路径， `java -jar xxx.jar` 会找不到
3. 
```
[Unit]
Description=cap-app

[Service]
Type=forking
WorkingDirectory=/root/cap
ExecStart=/bin/sh /root/cap/cap-restart.sh start
ExecStop=/bin/sh /root/cap/cap-restart.sh stop
Restart=always

TimeoutStartSec=90
TimeoutStopSec=45

[Install]
WantedBy=multi-user.target
```
4. `systemctl enable demo-thrift-proxy`
5. `systemctl start demo-thrift-proxy`
6. `systemctl status demo-thrift-proxy/systemctl status cap.service --no-pager`
7. 修改了 service 文件之后，要重载： `systemctl daemon-reload`
8. `systemctl restart demo-thrift-proxy` ： 重启服务，默认会先执行 `ExecStop`，然后执行 `ExecStart`
9. `systemctl cat <service-name>/systemctl list-unit-files | grep <service-name>` ： 查看服务定义文件的目录及内容，比如我要查找 mysql 自启动服务的描述文件，可以 `systemctl cat mysqld`
10. `journalctl -u my.service` ： 查看服务日志
11. `journalctl -u my.service -b` :查看最近一次启动的日志
12. `journalctl -xeu cap.service` : 查看最近一次启动的日志
13. `journalctl -fu nginx` : 实时查看日志
14. 查看所有日志： `journalctl -xe`


| 类型 | 文件/目录 | crontab 命令能否看到 | 说明 |
| :--- | :--- | :--- | :--- |
| 用户 crontab | /var/spool/cron/<username> | 可以用 crontab -l 查看 | 由 crontab -e 编辑的任务 |
| 系统 crontab | /etc/crontab | 不在用户 crontab 里，但可以直接查看文件 | 系统级计划任务，root 用户可编辑 |
| cron.daily / weekly / monthly | /etc/cron.daily/ 等 | 不在 crontab -l 输出中 | 通过系统 crontab 或 anacron 调用 run-parts 执行的脚本 |


## systemd 配置文件的区块

1. `[Unit]` : 通用配置
```
Description：简短描述
Documentation：文档地址
Requires：当前 Unit 依赖的其他 Unit，如果它们没有运行，当前 Unit 会启动失败
Wants：与当前 Unit 配合的其他 Unit，如果它们没有运行，当前 Unit 不会启动失败
BindsTo：与Requires类似，它指定的 Unit 如果退出，会导致当前 Unit 停止运行
Before：如果该字段指定的 Unit 也要启动，那么必须在当前 Unit 之后启动
After：如果该字段指定的 Unit 也要启动，那么必须在当前 Unit 之前启动
Conflicts：这里指定的 Unit 不能与当前 Unit 同时运行
Condition...：当前 Unit 运行必须满足的条件，否则不会运行
Assert...：当前 Unit 运行必须满足的条件，否则会报启动失败
```

2. `[Service]` : 服务配置
```
WantedBy：它的值是一个或多个 Target，当前 Unit 激活时（enable）符号链接会放入/etc/systemd/system目录下面以 Target 名 + .wants后缀构成的子目录中
RequiredBy：它的值是一个或多个 Target，当前 Unit 激活时，符号链接会放入/etc/systemd/system目录下面以 Target 名 + .required后缀构成的子目录中
Alias：当前 Unit 可用于启动的别名
Also：当前 Unit 激活（enable）时，会被同时激活的其他 Unit
```

3. `[Install]` : 安装配置
```
Type：定义启动时的进程行为。它有以下几种值。
Type=simple：默认值，执行ExecStart指定的命令，启动主进程
Type=forking：以 fork 方式从父进程创建子进程，创建后父进程会立即退出
Type=oneshot：一次性进程，Systemd 会等当前服务退出，再继续往下执行
Type=dbus：当前服务通过D-Bus启动
Type=notify：当前服务启动完毕，会通知Systemd，再继续往下执行
Type=idle：若有其他任务执行完毕，当前服务才会运行
ExecStart：启动当前服务的命令
ExecStartPre：启动当前服务之前执行的命令
ExecStartPost：启动当前服务之后执行的命令
ExecReload：重启当前服务时执行的命令
ExecStop：停止当前服务时执行的命令
ExecStopPost：停止当其服务之后执行的命令
RestartSec：自动重启当前服务间隔的秒数
Restart：定义何种情况 Systemd 会自动重启当前服务，可能的值包括always（总是重启）、on-success、on-failure、on-abnormal、on-abort、on-watchdog
TimeoutSec：定义 Systemd 停止当前服务之前等待的秒数
Environment：指定环境变量
```