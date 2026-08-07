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
6. `systemctl status demo-thrift-proxy`
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