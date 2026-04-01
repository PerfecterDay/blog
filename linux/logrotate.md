# logrotate
{docsify-updated}

> https://linux.die.net/man/8/logrotate

logrotate 旨在简化大量日志文件生成系统的管理。它支持日志文件的自动轮换、压缩、删除及邮件发送功能。每个日志文件可设置为每日、每周、每月处理，或在文件过大时自动处理。

在大多数 Linux 系统中，logrotate 通过 cron 或 systemd timer 自动运行：
```
cat /etc/cron.daily/logrotate
systemctl list-timers --all | grep logrotate
```

logrotate作为每日 `cron` 任务运行。除非日志的处理条件基于文件大小且logrotate每日运行多次，或使用了`-f`或`--force`选项，否则不会在一天内多次修改同一日志文件。

命令行可指定任意数量的配置文件。后续配置文件将覆盖先前文件的选项，因此配置文件的排列顺序至关重要。通常应使用单个包含所有必要配置文件的总配置文件。可以使用include指令实现此功能。若命令行指定目录，则该目录下所有文件均作为配置文件处理。

若未提供命令行参数，logrotate将显示版本信息、版权声明及简要用法说明。日志轮换过程中若发生任何错误，logrotate将以非零状态退出。

## 命令行语法
```
logrotate [-dv] [-f|--force] [-s|--state file] config_file ..
```

+ `-d, --debug` : 开启调试模式，默认会使用 `-v`。在调试模式下，不会更改日志或 `logrotate` 状态文件。
+ `-f, --force` : 告诉 logrotate 强制轮换，即使它认为没有必要。有时，在向 logrotate 配置文件添加新条目后，或在手动删除旧日志文件后，这很有用，因为新文件将被创建，日志记录将继续正常进行。
+ `-s, --state <statefile>` : 告诉 logrotate 使用另一个状态文件。这在 logrotate 以不同用户身份运行不同日志文件集时非常有用。默认状态文件是 `/var/lib/logrotate.status` 。
+ `--usage` : 打印使用帮助
+ `--?, --help` : 打开帮助
+ `-v, --verbose` : 打开 verbose 


logrotate 需要状态文件控制频率（比如 daily）。默认，logrotate 会使用 `/var/lib/logrotate/status`（或 `/var/lib/logrotate/status.log` ）来记录上次执行时间。因此，如果你刚运行完一次， logrotate 不会再次执行 `daily` 配置，除非你：
+ 删除状态文件
+ 使用 `-f` 强制忽略状态文件

新建一个日志文件后，立即运行 logrotate 也不会立即轮转日志文件：
1. logrotate 检查状态文件。如果这个日志文件之前从未被轮转过，状态文件中没有它的记录。
2. 对于新的、没有记录的文件，logrotate 不会将其视为需要立即轮转，而是将当前时间记录为它的首次轮转时间，然后等待 24 小时后的下一次运行。

## 配置文件
```
/etc/logrotate.conf
/etc/logrotate.d/
```

```
# see "man logrotate" for details
# rotate log files weekly/daily/monthly/yearly
weekly

# use the syslog group by default, since this is the owning group
# of /var/log/syslog.
su root syslog

# keep 4 weeks worth of backlogs
rotate 4

# create new (empty) log files after rotating old ones
create

# uncomment this if you want your log files compressed
compress

# packages drop log rotation information into this directory
include /etc/logrotate.d

/var/log/messages {
    rotate 5
    weekly
    postrotate
        /usr/bin/killall -HUP syslogd
    endscript
}

"/var/log/httpd/access.log" /var/log/httpd/error.log {
    rotate 5
    mail www@my.org
    size 100k
    sharedscripts
    postrotate
        /usr/bin/killall -HUP httpd
    endscript
}

/var/log/news/* {
    monthly
    rotate 2
    olddir /var/log/news/old
    missingok
    postrotate
        kill -HUP 'cat /var/run/inn.pid'
    endscript
    nocompress
}
```

+ `notifempty` : 如果日志文件是空的则不会切割文件
+ `ifempty` : 即使是空文件也会切割
+ `create [mode] [owner] [group]` : 在轮换操作完成后（在执行`postrotate`脚本之前），系统会立即创建日志文件（文件名与刚轮换的日志文件相同）。**这种情况下会有个问题，如果程序不重启，会将日志文件写入到分割归档的日志文件中而不是新建的日志文件中，因为logrotate是给源文件重命名后又创建了一个新文件**。`mode`参数以八进制形式指定日志文件的权限模式（与`chmod`命令相同），`owner`参数指定日志文件的所有者用户名，`group`参数指定日志文件所属的用户组。日志文件的任何属性均可省略，此时新文件的对应属性将继承原始日志文件的默认值。可通过 `nocreate` 选项禁用此功能。
+ `copytruncate` : 创建副本后就地截断原始日志文件，而非移动旧日志文件并可选创建新文件。当某些程序无法被要求关闭其日志文件，从而可能无限期地继续向旧日志文件追加写入时，可使用此选项。需注意文件复制与截断之间存在极短时间差，部分日志数据可能因此丢失(**复制后，程序又写日志文件了，然后截断导致新写的日志丢失**)。启用此选项时，`create`选项将失效，因旧日志文件仍保留在原位置。

## 被 SELinux（Security-Enhanced Linux） 拦截的问题
SELinux（Security-Enhanced Linux）是 Linux 的一个强制访问控制（MAC）安全系统，由美国 NSA 开发，后来开源并集成到 Linux 内核中。它提供了比传统权限（chmod / chown）更严格、更细粒度的安全控制。

一句话总结：**SELinux 是 Linux 下最强的安全机制，用来限制进程能访问哪些文件、目录、端口。即使进程被黑，也只能在受限范围内活动。**

```
[root@LXUATGMAA2 redis]# journalctl -t setroubleshoot
-- Logs begin at Fri 2023-12-08 04:12:34 CST, end at Thu 2025-12-11 10:50:09 CST. --
Dec 11 03:26:06 LXUATGMAA2.demo.COM.UAT setroubleshoot[1964098]: AnalyzeThread.run(): Cancel pending alarm
Dec 11 03:26:07 LXUATGMAA2.demo.COM.UAT setroubleshoot[1964098]: SELinux is preventing logrotate from read access on the directory redis. For complete SELinux messages run: sealert -l f9f76ccb-fe44-4e13-8872-c7c728b65d8e
Dec 11 03:26:07 LXUATGMAA2.demo.COM.UAT setroubleshoot[1964098]: SELinux is preventing logrotate from read access on the directory redis.
```

解决方案:
1. 新建一个 te 文件，内容如下：
```

module logrotate-redis 1.0;

require {
	type redis_var_lib_t;
	type logrotate_t;
        class dir { getattr search add_name create open remove_name rmdir setattr write };
        class file { entrypoint execute getattr open read map setattr write create link unlink append };
}

#============= logrotate_t ==============

#!!!! This avc is allowed in the current policy
# 允许logrotate_t访问redis_var_lib_t目录
allow logrotate_t redis_var_lib_t:dir { getattr search add_name create open remove_name rmdir setattr write };;
# 允许logrotate_t访问redis_var_lib_t文件（核心轮转权限）
allow logrotate_t redis_var_lib_t:file { entrypoint execute getattr open read map setattr write create link unlink append };
```
2. 然后编译
```
# 1. 编译模块
checkmodule -M -m -o logrotate-redis.mod logrotate-redis.te
# 2. 打包策略包
semodule_package -o logrotate-redis.pp -m logrotate-redis.mod
# 3. 加载（覆盖旧版本）
semodule -i logrotate-redis.pp
# 4. 验证加载状态
semodule -l | grep logrotate-redis
# 5.
semanage fcontext -a -t var_log_t "/var/lib/redis(/.*)?"
# 6. 
restorecon -Rv /var/lib/redis
```