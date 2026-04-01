# Helm 命令行
{docsify-updated}

> https://helm.sh/zh/docs/helm/

## Helm 仓库
+ `helm repo add` - 添加chart仓库
+ `helm repo index` - 基于包含打包chart的目录，生成索引文件
+ `helm repo list` - 列举chart仓库
+ `helm repo remove` - 删除一个或多个仓库
+ `helm repo update` - 从chart仓库中更新本地可用chart的信息

## chart
+ `helm pull` - 从仓库下载并（可选）在本地目录解压

## 安装
+ `helm install` - 安装
+ `helm uninstall` - 卸载


1. 从 Artifact Hub 添加一个chart 仓库 ：`helm repo add bitnami https://charts.bitnami.com/bitnami`
2. 更新仓库 ：`helm repo update`
3. 搜索 Charts: `helm search repo bitnami`
4. 查看某个 Chart 的信息: `helm show [all] chart bitnami/mysql`
5. 如果你想下载和查看一个发布的chart，但不安装它： `helm pull chartrepo/chartname`,执行完会在当前目录看到对应的文件
6. 上传Chart到阿里云： `helm cm-push consul-1.1.1.tgz demo`
7. Helm可以通过多种途径查找和安装chart， 但最简单的是安装官方的bitnami charts: `helm install bitnami/mysql --generate-name`
   每当您执行 helm install 的时候，都会创建一个新的发布版本。 所以一个chart在同一个集群里面可以被安装多次，每一个都可以被独立的管理和升级。
8. 查看发布的列表： `helm list`,`helm list -n namespace`
9. 卸载版本： `helm uninstall mysql-1612624192`
10. 帮助：`helm help` 或者在任意命令后添加 -h 选项：`helm get -h`