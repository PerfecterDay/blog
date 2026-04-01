# Nginx安装与启动及命令行参数
{docsify-updated}

## Nginx安装
下载[nginx官网](http://nginx.org)下载安装即可。

## Nginx启动、停止及重新加载配置文件
nginx有一个master线程和若干个工作线程，master线程主要用来读取配置文件、加载配置以及维护工作线程，工作线程才会真正的处理请求。

1. 安装目录下，直接双击nginx启动。或者，cmd模式切换到安装目录，运行nginx.exe文件。
   1. windows 设置了环境变量的情况下，可以使用：`nginx -p "%NGINX_HOME%"` 启动nginx
2. 一旦nginx启动后，可以使用以下命令：
>nginx -s signal

signal可以是以下参数：

* *stop* — 快速停止
* *quit* — 优雅停止
* *reload* — 重新加载配置文件
* *reopen* — 重新打开日志文件

mac homebrew 相关路径： 
```
/opt/homebrew/var/www
/opt/homebrew/etc/nginx/nginx.conf
/opt/homebrew/etc/nginx/servers/.
/opt/homebrew/opt/nginx/bin/nginx

~/Library/LaunchAgents/homebrew.mxcl.nginx.plist --> /opt/homebrew/Cellar/nginx/1.29.1/homebrew.mxcl.nginx.plist

/opt/homebrew/var/log/nginx/error.log
/opt/homebrew/var/log/nginx/access.log
```

## Nginx命令行参数

* *-h* 显示命令行参数帮助信息
* *-c file* 指定file为nginx的配置文件
* *-g directives* 设置全局的配置指令，如：

>nginx -g "pid /var/run/nginx.pid; worker_processes `sysctl -n hw.ncpu`;"

* *-p prefix* 设置nginx路径前缀，即保存nginx服务文件的文件夹
* *-t* 测试配置文件，检查配置文件语法，然后加载配置文件
* *-T* 与-t相同，但是会在标准输出打印出配置文件
* *-q* 测试配置文件，不输出费错误信息
* *-v* 输出nginx的版本
* *-V* 输出nginx的版本，编译器版本及配置参数
