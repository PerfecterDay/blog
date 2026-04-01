# docker 存储
{docsify-updated}

- [docker 存储](#docker-存储)
	- [Storage driver](#storage-driver)
	- [Data Volume](#data-volume)
	- [数据共享](#数据共享)

Docker 为容器提供了两种存放数据的策略：
1. 由 storage driver 管理的镜像层和容器层
2. Data Volume。

### Storage driver
就是前面介绍过的镜像层与容器层，容器运行时可以从镜像层中读出只读数据，并且可以将数据修改或者写一些数据到容器层。但是，保存在容器层的数据在容器删除时就会被删除，无法被长久持久化。

### Data Volume
Data Volume 本质上是宿主机文件系统中的目录或文件，能直接被 mount 到容器的文件系统中。Data Volume 有以下特点：
1. 是目录或文件而不是没有格式化的磁盘
2. 容器可以读写 volume 中的数据
3. volume 中的数据可以被用久保存，即使使用它的容器已经销毁

Docker 提供了两种类型的 volume：
1. bind mount
	bing mount 通过`-v <host_path>:<mount_path>:<read_write_control>`将host 上已存在的目录或文件挂载到容器的某个路径下，并且可以设置读写权限。如果`<mount_path>`本来就存在，则原有数据会被隐藏起来，取而代之的是 `<host_path>`中的数据。容器内对`<mount_path>`的读写会反映到宿主机的`<host_path>`中，并且即使容器被删除，宿主机的文件也会被保存下来。  
	但是，这种方式有个缺点，当容器迁移到新的宿主机时，新宿主机必须也要有挂载的路径`<host_path>`，否则容器会失败。

2. docker managed volume
	docker managed volume 与 bind mount 最大的区别就是不需要指定 mount 源，指定 mount point 就可以了，
	1. 容器启动时，简单地告诉docker 我需要一个 volume 存放数据，帮我 mount 到指定目录： `-v <mount_path>` 即可。
	2. docker 会在 /var/lib/docker/volumes 中生成一个随机目录作为 mount 源
	3. 如果容器内 `<mount_path>` 已经存在，则会把数据复制到 mount 源中
	4. 将 volume mount 到 `<mount_path>`
   
两种 volume 的异同：
<center><img src="pics/docker-volume.png" width="30%" style="inline"></center>

可以使用 `docker inspect <container>` 或者 `docker volume` 来查看 volume 信息，但是 `docker volume`只能看到 docker managed volume。

### 数据共享
1. 容器与宿主机之间分享数据
	使用 bind mount 时，直接可以共享 host 特定目录中的数据；使用 docker managed volume 时，因为宿主机的目录是在容器启动后创建的，所以要想共享数据必须在容器启动后，使用`docker cp <file/dir> <container>:<mount_path>`将数据拷贝到 volume 中，当然启动后可以通过 `docker inspect <containe>`获取到 volume 在宿主机的路径，然后直接使用 cp 命令拷贝数据到 volume 。

2. 容器之间分享数据
	1. 多个容器通过 bind mount 宿主机的同一个目录
	2. volume container