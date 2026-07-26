# 容器工具
{docsify-updated}

podman 安装：
```
	brew install podman
	podman machine init /  podman machine init --memory=8192 --cpus=2
	podman machine start
	podman info
```

`podman machine set --rootful` 设置 root 权限

`podman machine ssh`;
`sudo sysctl -w vm.max_map_count=262144`


## Orbstack
> https://docs.orbstack.dev/

<center><img src="pics/arch-orbstack.png" alt=""></center>

```
Start: orb start k8s
Stop: orb stop k8s
Restart: orb restart k8s
Delete: orb delete k8s
```

### Machines

#### 与mac宿主机以及其它machine之间的文件访问及共享
Mac files are available in Linux machines at `/mnt/mac` , and Linux files are available from Mac at `~/OrbStack` or the OrbStack tab in Finder. See File sharing for more information.

You can access files from other Linux machines at `/mnt/machines`: `ls /mnt/machines/foo`

#### 执行mac 命令