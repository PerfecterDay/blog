# Mysql配置与系统状态变量
{docsify-updated}

## 命令行上使用启动选项

`mysqld_safe --启动选项1[=值1] --启动选项2[=值2] ... --启动选项n[=值n]`

可以将各个启动选项写到一行中，各个启动选项之间使用空白字符隔开，在每一个启动选项名称前边添加-- 。对于不需要值的启动选项，比方说 skip-networking ，它们就不需要指定对应的值。对于需要指定值的启动选项，比如 default-storage-engine 我们在指定这个设置项的时候需要显式的指定它的值，比方说InnoDB 、 MyISAM 啦什么的～ 在命令行上指定有值的启动选项时需要注意，选项名、=、选项值之间不可以有空白字符。

每个MySQL程序都有许多不同的选项。大多数程序提供了一个 `--help` 选项，你可以查看该程序支持的全部启动选项以及它们的默认值。

+ `mysql --help` 可以看到 `mysql` 客户端程序支持的启动选项
+ `mysqld_safe --help` 可以看到 `mysqld_safe` 程序支持的启动选项。
+ `mysqld --verbose --help` 可以看到 `mysqld` 程序支持的启动选项。
+ `mysql.server --help` 可以看到 `mysql.server` 程序支持的启动选项。

## 配置文件
在命令行中设置启动选项只对当次启动生效，也就是说如果下一次重启程序的时候我们还想保留这些启动选项的话，还得重复把这些选项写到启动命令行中，这样很麻烦。把需要设置的启动选项都写在这个配置文件中，每次启动服务器的时候都从这个文件里加载相应的启动选项。由于这个配置文件可以长久的保存在计算机的硬盘里，所以只需我们配置一次，以后就都不用显式的把启动选项都写在启动命令行中了，所以我们推荐使用配置文件的方式来设置启动选项。

### 配置文件的路径
以 macos homebrew 安装的方式为例，相应的程序会在一下路径查找配置文件：

+ /etc/my.cnf
+ /etc/mysql/my.cnf
+ /opt/homebrew/etc/my.cnf
+ ~/.my.cnf

我们可以使用 `mysql --help | hrep my.cnf` 命令来查看 mysql 查找配置文件的路径。

### 配置文件的内容

与在命令行中指定启动选项不同的是，配置文件中的启动选项被划分为若干个组，每个组有一个组名，用中括号[] 扩起来：

```
[server]
option1 #这是option1，该选项不需要选项值
option2 = value2 #这是option2，该选项需要选项值
...

[mysqld]
option1 #这是option1，该选项不需要选项值
option2 = value2 #这是option2，该选项需要选项值
....

[mysqld_safe]
option1 #这是option1，该选项不需要选项值
option2 = value2 #这是option2，该选项需要选项值
...

[client]
option1 #这是option1，该选项不需要选项值
option2 = value2 #这是option2，该选项需要选项值
...

[mysql]
option1 #这是option1，该选项不需要选项值
option2 = value2 #这是option2，该选项需要选项值
...

[mysqladmin]
option1 #这是option1，该选项不需要选项值
option2 = value2 #这是option2，该选项需要选项值
...
```

是用mysql 命令时，会读取配置文件中的不同组，以下列举了一些常见命令读取的组：
| 启动命令 | 类别 | 能读取的组 |
| :---        |    :----:   |          ---: |
|mysqld | 启动服务器 | [mysqld]、 [server]|
|mysqld_safe |启动服务器| [mysqld]、 [server] 、 [mysqld_safe]|
|mysql.server |启动服务器| [mysqld]、 [server] 、 [mysql.server]|
|mysql |启动客户端| [mysql]、 [client]|
|mysqladmin |启动客户端| [mysqladmin]、 [client]|
|mysqldump |启动客户端| [mysqldump]、 [client]|

### 配置的优先级
MySQL 将在某些固定的路径下搜索配置文件，我们也可以通过在命令行上指定 `defaults-extra-file` 启动选项来指定额外的配置文件路径。 MySQL 将按照我们在上表中给定的顺序依次读取各个配置文件，如果该文件不存在则忽略。值得注意的是，**如果我们在多个配置文件中设置了相同的启动选项，那以最后一个配置文件中的为准**。

比方说 /etc/my.cnf 文件的内容是这样的：
```
[server]
default-storage-engine=InnoDB
```
而 ~/.my.cnf 文件中的内容是这样的：
```
[server]
default-storage-engine=MyISAM
```
因为 `~/.my.cnf` 比 `/etc/my.cnf` 顺序靠后，所以如果两个配置文件中出现相同的启动选项，以 `~/.my.cnf` 中的为准，所以 MySQL 服务器程序启动之后， `default-storage-engine` 的值就是 `MyISAM` 。

另外，因为同一个命令可以访问配置文件中的多个组，如果每个组中的配置不一致的话，**以配置文件中最后一个出现的组中的启动选项为准**。

**如果同一个启动选项既出现在命令行中，又出现在配置文件中，那么以命令行中的启动选项为准。**

## 系统变量
MySQL 服务器程序运行过程中会用到许多影响程序行为的变量，它们被称为 MySQL 系统变量，比如允许同时连入的客户端数量用系统变量 `max_connections` 表示，表的默认存储引擎用系统变量 `default_storage_engine` 表示，查询缓存的大小用系统变量 `query_cache_size` 表示， MySQL 服务器程序的系统变量有好几百条，我们就不一一列举了。每个系统变量都有一个默认值，我们可以使用命令行或者配置文件中的选项在启动服务器时改变一些系统变量的值。大多数的系统变量的值也可以在程序运行过程中修改，而无需停止并重新启动它。

可以使用下列命令查看 MySQL 服务器程序支持的系统变量以及它们的当前值：
```
SHOW VARIABLES [LIKE 匹配的模式];
SHOW [GLOBAL|SESSION] VARIABLES [LIKE 匹配的模式];
```
由于 **系统变量** 实在太多了，如果我们直接使用 `SHOW VARIABLES` 查看的话就直接刷屏了，所以通常都会带一个 `LIKE` 过滤条件来查看我们需要的系统变量的值。

### 系统变量的设置
系统变量比较牛逼的一点就是，对于大部分系统变量来说，它们的值可以在服务器程序运行过程中进行动态修改而无需停止并重启服务器。不过系统变量有作用范围之分。

大部分的 系统变量 都可以通过启动服务器时传送启动选项的方式来进行设置:
```
mysqld --default-storage-engine=MyISAM --max-connections=10
```

系统变量有 **作用范围** 的概念，具体来说作用范围分为这两种：
+ `GLOBAL` : 全局变量，影响服务器的整体操作。
+ `SESSION` : 会话变量，影响某个客户端连接的操作。

在服务器启动时，会将每个全局变量初始化为其默认值（可以通过命令行或选项文件中指定的选项更改这些默认值）。然后服务器还为每个连接的客户端维护一组会话变量，客户端的会话变量在连接时使用相应全局变量的当前值初始化。通过启动选项设置的系统变量的作用范围都是 GLOBAL 的，也就是对所有客户端都有效的。

在服务器程序运行期间通过客户端程序设置系统变量的语法：
```
SET [GLOBAL|SESSION] 系统变量名 = 值;
SET [@@(GLOBAL|SESSION).]var_name = XXX;
```

如果在设置系统变量的语句中省略了作用范围，默认的作用范围就是 SESSION 。所以以下三个设置语句是等效的：
```
SET SESSION default_storage_engine = MyISAM;
SET @@SESSION.default_storage_engine = MyISAM;
SET default_storage_engine = MyISAM;
```


## 状态变量
为了让我们更好的了解服务器程序的运行情况， MySQL 服务器程序中维护了好多关于程序运行状态的变量，它们被称为 **状态变量** 。比方说`Threads_connected` 表示当前有多少客户端与服务器建立了连接， `Handler_update` 表示已经更新了多少行记录吧啦吧啦，像这样显示服务器程序状态信息的状态变量还有好几百个，我们就不一一唠叨了，等遇到了会详细说它们的作用的。

由于 **状态变量** 是用来显示服务器程序运行状况的，所以它们的值只能由服务器程序自己来设置，我们程序员是不能设置的。与系统变量类似， 状态变量也有 `GLOBAL` 和 `SESSION` 两个作用范围的，所以查看状态变量的语句可以这么写：
```
SHOW [GLOBAL|SESSION] STATUS [LIKE 匹配的模式];
```