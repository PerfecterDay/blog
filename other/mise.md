```
# 查看可用版本
mise ls-remote java
mise ls-remote java | grep 21

# 安装
mise install java@temurin-21

# 当前项目使用
mise use java@temurin-21

# 全局使用
mise use -g java@temurin-21

# 查看已安装工具
mise ls

# 查看当前项目实际版本
mise current

# 查看工具实际路径
mise which java

# 根据 mise.toml 安装整个项目环境
mise install
```