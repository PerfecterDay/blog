#  sdkman
{docsify-updated}

> https://sdkman.io/usage


安装： `curl -s "https://get.sdkman.io" | bash`


+ `sdk list`: 列出所有可用软件及版本
+ `sdk list java`： 列出所有可用 java 版本
+ `sdk install java`： 安装 java 最新版本
+ `sdk install java 11.0.17-tem `： 安装 java 特定版本
+ `sdk current java`： 查看当前使用的java版本
+ `sdk current`：查看当前所有软件包使用的版本
+ `sdk use java 11.0.19-tem`: 在当前 shell 环境临时切换java 版本，如果切换shell，用的还是全局版本的java
+ `sdk default java 11.0.19-tem`：切换默认的 java 版本，全局生效
+ `sdk uninstall scala 3.2.2`: 卸载某个版本
+ `sdk update` : 更新仓库，查看新的可用软件包
+ `sdk upgrade` ： 升级所有安装的软件
+ `sdk upgrade java` ： 升级 java 
+ `sdk selfupdate` ： 升级sdkman
+ `sdk selfupdate force`: 重新安装sdkman
+ `sdk flush`: 删除缓存