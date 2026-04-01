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
```
Start: orb start k8s
Stop: orb stop k8s
Restart: orb restart k8s
Delete: orb delete k8s
```