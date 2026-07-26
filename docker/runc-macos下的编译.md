# runc
{docsify-updated}

> https://github.com/opencontainers/runc

## 编译
1. 使用 orbstack 创建一个 linux machine
2. 安装 golang 开发环境
3. 下载 runc 源码到 `go env` 中的 GOPATH 目录： `/root/go/runc`
4. `apt update && apt install -y make gcc linux-libc-dev libseccomp-dev pkg-config git` 安装必须的依赖包
5. `make RUNC_BUILDTAGS="-libpathrs" install`

## 运行一个容器
1. `mkdir /mycontainer`
2. `cd /mycontainer`
3. `mkdir rootfs` 
4. `mac docker export $(mac docker create busybox) -o busybox`
5. `tar -xvf  busybox -C rootfs/`
6. `runc spec`
7. `runc run mycontainerid`

## runc 命令
```
# run as root
cd /mycontainer
runc create mycontainerid

# view the container is created and in the "created" state
runc list

# start the process inside the container
runc start mycontainerid

# after 5 seconds view that the container has exited and is now in the stopped state
runc list

# now delete the container
runc delete mycontainerid
```