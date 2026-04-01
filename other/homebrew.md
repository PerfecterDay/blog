# Mac Homebrew 的使用
{docsify-updated}

>https://zhuanlan.zhihu.com/p/547898033  
>https://www.xiebruce.top/983.html

- [Mac Homebrew 的使用](#mac-homebrew-的使用)
	- [使用代理加速](#使用代理加速)
	- [安装软件的默认配置文件位置](#安装软件的默认配置文件位置)
	- [国内 homebrew 加速](#国内-homebrew-加速)
	- [常用命令](#常用命令)
	- [Macos 自带的服务管理 launchctl](#macos-自带的服务管理-launchctl)
	- [brew services](#brew-services)
		- [Jenv 进行JDK多版本管理](#jenv-进行jdk多版本管理)
		- [常用命令](#常用命令-1)

## 使用代理加速
`ALL_PROXY=socks5://127.0.0.1:1080 brew update && brew upgrade`

## 安装软件的默认配置文件位置
 ```
 /opt/homebrew/etc/proxychains.conf
```

## 国内 homebrew 加速
```
cd "$(brew --repo)/Library/Taps/"
mkdir homebrew && cd homebrew
git clone https://mirrors.ustc.edu.cn/homebrew-core.git
```

```
cd "$(brew --repo)/Library/Taps/"
cd homebrew
git clone https://mirrors.ustc.edu.cn/homebrew-cask.git
```

```
echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.tuna.tsinghua.edu.cn/homebrew-bottles' >> ~/.bash_profile
source ~/.bash_profile
```

## 常用命令
1. `brew install [formula]`:安装 formula
2. `brew uninstall [formula]`:卸载 formula
3. `brew list`: 显示所有安装的 formula
4. `brew search (text|/text/)`： 查找 formula
5. `brew cleanup`: 清理缓存
6. `brew pin jenkins` : 锁定指定的软件版本
7. `brew list --pinned` : 查看不升级的软件列表
8. `brew unpin jenkins` : 解锁软件版本

## Macos 自带的服务管理 launchctl
> 摘译自 [robots.thoughtbot.com](http://robots.thoughtbot.com/starting-and-stopping-background-services-with-homebrew)
`launchctl` 命令加载/卸载开机自动运行的服务，在 OS X 中，服务本身存储在 `.plist` 文件中（即 property list），这些文件的位置一般在 `~/Library/LaunchAgents` 或 `/Library/LaunchAgents`。可以使用 `launchctl load $PATH_TO_LIST` 和 `launchctl unload $PATH_TO_LIST` 命令来加载/卸载他们。加载就是允许这个程序开机执行，卸载反之。

如果你使用 `Homebrew` 安装过 `mysql` 那么下面的安装后提示你可能比较熟悉
 ```
To have launchd start mysql at login:
    ln -sfv /usr/local/opt/mysql/*.plist ~/Library/LaunchAgents
Then to load mysql now:
    launchctl load ~/Library/LaunchAgents/homebrew.mxcl.mysql.plist
Or, if you don't want/need launchctl, you can just run:
    mysql.server start
 ```
如果按上面的说明操作的话，未免太麻烦了，而且也很难记住 plist 的位置。还好 Homebrew 提供了一个易用的接口来管理 plist，然后你就不用再纠结什么 `ln`，`launchctl`，和 plist 的位置了。

## brew services
首先安装 `brew services` 命令
 ```
 brew tap gapple/services
 ```
下面举个例子
 ```
 $ brew services start mysql
==> Successfully started `mysql` (label: homebrew.mxcl.mysql)
 ```

在后台，`brew services start` 其实执行了最上面的安装后消息里面提到的所有命令，比如首先运行 `ln -sfv ...`，然后 `launchctl load ~/Library/LaunchAgents/homebrew.mxcl.mysql.plist` 。

假设突然 MySQL 出毛病了，我们要重启一下，那么执行下面的命令就行了
 ```
brew services restart mysql
Stopping `mysql`... (might take a while)
==> Successfully stopped `mysql` (label: homebrew.mxcl.mysql)
==> Successfully started `mysql` (label: homebrew.mxcl.mysql)
 ```

使用 `--verbose` 可以查看输出日志，排查问题的时候有用：
 ```
 brew services start squid --verbose
 ```

想看所有的已启用的服务的话：
 ```
 $ brew services list
redis      started      442 /Users/gabe/Library/LaunchAgents/homebrew.mxcl.redis.plist
postgresql started      443 /Users/gabe/Library/LaunchAgents/homebrew.mxcl.postgresql.plist
mongodb    started      444 /Users/gabe/Library/LaunchAgents/homebrew.mxcl.mongodb.plist
memcached  started      445 /Users/gabe/Library/LaunchAgents/homebrew.mxcl.memcached.plist
mysql      started    87538 /Users/gabe/Library/LaunchAgents/homebrew.mxcl.mysql.plist
 ```

要注意的是，这里不止显示通过 `brew services` 加载的服务，也包含 `launchctl load` 加载的。

如果你卸载了 MySQL 但是 `Homebrew` 没把 plist 文件删除的话，你可以
 ```
$ brew services cleanup
Removing unused plist /Users/gabe/Library/LaunchAgents/homebrew.mxcl.mysql.plist
 ```

`brew services list` 列出的 plist 文件在软件安装包中都会有一个对应的原始文件（在 `brew --cellar` 的对应目录中），每次执行 `brew services start` 启动服务，homebrew 都会将原始文件拷贝到 `~/Library/LaunchAgents`。所以直接边界 `~/Library/LaunchAgents` 目录下的 plist 文件不会生效。需要编辑原始文件。

### Jenv 进行JDK多版本管理
1. 安装Jenv ：`brew install jenv`
2. 启用Jenv，需要执行一下命令： add the following to your ~/.zshrc:
	```
	export PATH="$HOME/.jenv/bin:$PATH"
	eval "$(jenv init -)"
	```
3. 搜索安装需要的JDK
   ```
   brew install adoptopenjdk/openjdk/adoptopenjdk8 --cask
   brew install adoptopenjdk/openjdk/adoptopenjdk11 --cask
   ```
4. jenv versions 显示可用的 JDK 版本，如果没有的话，需要使用 `jenv add [pathToJDK]` 添加安装的JDK
5. `jenv local [javaversion]` 切换 JDK 版本

`sudo update-alternatives --config java`

### 常用命令
```
# 显示 Homebrew 本地的 Git 仓库
$ brew --repo

# 显示 Homebrew 安装路径
$ brew --prefix

# 显示 Homebrew Cellar 路径
$ brew --cellar

# 显示 Homebrew Caskroom 路径
$ brew --caskroom

# 缓存路径
$ brew --cache

# 删除下载的文件
brew cleanup --prune=all -n ##-n 会给出提示不会真的删除
brew cleanup --prune=all # 删除文件
```