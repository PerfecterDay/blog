# docker 容器
{docsify-updated}

> https://medium.com/@furkan.turkal/how-does-docker-actually-work-the-hard-way-a-technical-deep-diving-c5b8ea2f0422

## 容器操作的命令

<center>
	<img src="/pics/container-state.jpg">
</center>

1. 创建容器  
	1. 创建并启动容器执行 command 命令 `docker run [options] image_name [command]` ，实际上是 `docker create` 和 `docker start` 的组合，容器是否**长久运行和 command 命令有关,只要命令不结束，容器也不会退出（exit 状态）**，相关选项:
		+ `-d` :代表后台守护进程运行
		+ `-p 宿主机IP:宿主机端口：容器端口`：将容器端口映射到宿主机端口
		+ `--restart=always`：自动重启容器
		+ `--name=yourname`：指定容器名字
		+ `-v 宿主机目录(文件)：容器目录（文件）：ro|rw`： 将宿主机的目录或文件作为卷，挂载到容器对应目录或文件
		+ `-e, --env=[]`: 设置环境变量  
	示例：`docker run -p 3316:3306 -e MYSQL_ROOT_PASSWORD=123456 -d --name=mysql-master  mysql`
	2. 创建一个新的容器但不启动它： `docker create [options] image_name [command]` 示例：`docker create  --name myrunoob  nginx:latest`

2. 查看容器  
	1. 查看正在运行的容器: `docker ps` 或者 `docker container ls`
	2. 查看所有容器， `-a` 选项: `docker ps -a` 或者`docker container ls -a `
	3. 查看指定容器的详细信息: `docker inspect container_id`
	4. 查看镜像、容器、数据卷所占用的空间: `docker system df`
	5. 删除指定容器: `docker container rm container_id` 或者 `docker rm container_id`
	6. 清理所有处于终止状态的容器: `docker container prune` 

3. 启动/停止容器  
	1. 创建并启动容器: `docker run`
	2. 重启已经停止的容器: `docker start container_id`
	3.  停止容器: `docker stop container_id` 
	4.  重启容器: `docker restart container_id|tag`，  示例-`docker restart master slave`

4. 进入正在运行的容器
	1.  `docker attach [containerid]`：可通过 Ctrl+p 然后 Ctrl+q 组合键退出 attach 终端
	2.  `docker exec -it [containerid]`：`-it` 以交互模式打开 pseudo-TTY，执行 bash，其结果就是打开了一个 bash 终端。
	3.  启动时直接进入容器，适用于要保持容器后台运行的情况：`docker run -it [imageid] [command]` -> `docker run -it nginx /bin/bash`
		```
				docker run -dit --name user-center-com-gmas -v /Users/coder_wang/Workspace/uat-docker/user-com-gmas:/usr/local/demo/com_gmas -p 8910:8910  golang:1.17.7-stretch
		```

		attach 与 exec 主要区别如下:
    	+ attach 直接进入容器 启动命令 的终端，不会启动新的进程。
    	+ exec 则是在容器中打开新的终端，并且可以启动新的进程。
    	+ 如果想直接在终端中查看启动命令的输出，用 attach；其他情况使用 exec。
    	+ 当然，如果只是为了查看启动命令的输出，可以使用 docker logs 命令

	4. 覆盖 entry-point 启动容器：
		```
		docker run -it -p 8101:8100 \
		-v ./config:/home/config/ \
		-e SPRING_PROFILES_ACTIVE=uat \
		--entrypoint sh \
		token:1.0.0
		```

5. 查看容器启动命令的输出日志
	1.  `docker logs -f`:持续输出docker启动命令的输出日志
6. 容器与宿主机之间拷贝文件 
   1. `docker cp ~/Documents/demo/com_gmas/com_gmas.tar.gz ubuntu:/home`：主机到容器
   2. `docker cp ubuntu:/home/test.txt ~/Documents/demo/com_gmas/`：容器到主机
7. `docker run -it -p 8999:8999 --name user-center -v /Users/coder_wang/Workspace/com_gmas:/usr/local/demo/com_gmas  docker.io/library/golang:1.17.7`
8. `docker update --restart=always --cpus=2 --memory=1g my-container`: 用于动态修改一个或多个已运行容器的资源限制或重启策略，而不会中断容器运行。

## 实现容器的底层技术
cgroup 和 namespace 是最重要的两种技术
+ cgroup 实现资源限额，
+ namespace 实现资源隔离。

1. cgroup(全称Control Group) : 提供资源分配与管理功能。

   Linux 操作系统通过 cgroup 可以设置进程使用 CPU、内存 和 IO 资源的限额。相信你已经猜到了：前面我们看到的`--cpu-shares`、`-m`、`--device-write-bps` 实际上就是在配置 cgroup。
   cgroup 到底长什么样子呢？我们可以在 /sys/fs/cgroup 中找到它。在 /sys/fs/cgroup/cpu/docker 目录中，保存了对CPU资源的限制配置，Linux 会为每个容器创建一个 cgroup 目录，以容器长ID 命名。
   目录中包含所有与 cpu 相关的 cgroup 配置，文件 cpu.shares 保存的就是 --cpu-shares 的配置，值为 512。
   同样的，/sys/fs/cgroup/memory/docker 和 /sys/fs/cgroup/blkio/docker 中保存的是内存以及 Block IO 的 cgroup 配置。

2. namespace : 提供环境隔离功能。  

    在每个容器中，我们都可以看到文件系统，网卡等资源，这些资源看上去是容器自己的。拿网卡来说，每个容器都会认为自己有一块独立的网卡，即使 host 上只有一块物理网卡。这种方式非常好，它使得容器更像一个独立的计算机。
    Linux 实现这种方式的技术是 namespace。namespace 管理着 host 中全局唯一的资源，并可以让每个容器都觉得只有自己在使用它。换句话说，namespace 实现了容器间资源的隔离。
	Linux 使用了六种 namespace，分别对应六种资源： `Mount` 、 `UTS` 、 `IPC` 、 `PID` 、`Network` 和 `User`，下面我们分别讨论。

	1. Mount namespace  
		Mount namespace 让容器看上去拥有整个文件系统。容器有自己的 / 目录，可以执行 mount 和 umount 命令。当然我们知道这些操作只在当前容器中生效，不会影响到 host 和其他容器。
	2. UTS namespace  
		简单的说，UTS namespace 让容器有自己的 hostname。 默认情况下，容器的 hostname 是它的短ID，可以通过 -h 或 --hostname 参数设置。
	3. IPC namespace  
		IPC namespace 让容器拥有自己的共享内存和信号量（semaphore）来实现进程间通信，而不会与 host 和其他容器的 IPC 混在一起。
	4. PID namespace  
		我们前面提到过，容器在 host 中以进程的形式运行。例如当前 host 中运行了两个容器：
	5. Network namespace  
		Network namespace 让容器拥有自己独立的网卡、IP、路由等资源。我们会在后面网络章节详细讨论。
	6. User namespace  
		User namespace 让容器能够管理自己的用户，host 不能看到容器中创建的用户。

## 查看容器对应的宿主机进程
```
docker run -it --rm --privileged --pid=host --net=host --ipc=host \
  justincormack/nsenter1
```