# ES 集群的安装部署

## Docker 安装
1. `docker pull docker.elastic.co/elasticsearch/elasticsearch:8.4.2`
2. `docker network create elastic`
3. `docker run -d -e ES_JAVA_OPTS="-Xms4g -Xmx4g" --name es01 --net elastic -p 9200:9200 -p 9300:9300 -it docker.elastic.co/elasticsearch/elasticsearch:8.4.2`
4. `docker cp es01:/usr/share/elasticsearch/config/certs/http_ca_es01.crt .`
5. `curl --cacert http_ca.crt -u elastic https://localhost:9200` -〉测试
6. 添加节点：`docker run -e ENROLLMENT_TOKEN="<token>" --name es02 --net elastic -it docker.elastic.co/elasticsearch/elasticsearch:8.4.2`


重置密码：
`docker exec -it es01 /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic`

重新生成 enrollment token（加入集群时用）：  
+ 别的node 加入集群：`docker exec -it es01 /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s node`
+ Kibana: `docker exec -it es01 /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana`

Turns out exit code 137 with docker containers is commonly due to memory allocation issues!


## Docker-compose 安装
1. `docker pull docker.elastic.co/elasticsearch/elasticsearch:8.4.2`
2. 下载 .env 和 docker-compose.yml ：https://github.com/elastic/elasticsearch/tree/master/docs/reference/setup/install/docker
3. 修改 .env 文件中的配置
4. 后台启动es集群：`docker-compose up -d` 
5. 停止并删除es集群：`docker-compose down`、
6. 删除es集群（容器、网络、卷）：`docker-compose down -v` 


## 生产环境推荐配置
1. 修改mmap 限制 >= 262144
	+ linux: `sysctl vm.max_map_count`可以查看mmap值，修改`sudo sysctl -w vm.max_map_count=262144`
	+ macos: 
    	+ podman : `podman machine ssh; sudo sysctl -w vm.max_map_count=262144`
    	+ docker :  from the docker desktop GUI - `Preferences -> Resources -> Swap -> Apply and Restart`
2. Elasticsearch 在容器中运行时的用户是 elasticsearch，uid:gid ——> 1000:0.当挂载卷到本机文件时，必须确保 elasticsearch 用户对本地文件有读写权限，推荐对本地文件夹授予 gid 0 读写权限：
	mkdir esdatadir
	chmod g+rwx esdatadir
	chgrp 0 esdatadir
3. 为nofile和noproc增加 ulimits：
	+ 查看 ulimits 默认设置：`docker run --rm docker.elastic.co/elasticsearch/elasticsearch:{version} /bin/bash -c 'ulimit -Hn && ulimit -Sn && ulimit -Hu && ulimit -Su'`
	+ 容器启动时设置配置：`--ulimit nofile=65535:65535`
4. 禁用 swapping ：`-e "bootstrap.memory_lock=true" --ulimit memlock=-1:-1`
5. 设置随机端口：`--publish-all`
6. 手动设置JVM：
   docker run -e ES_JAVA_OPTS="-Xms4g -Xmx1g" -e ENROLLMENT_TOKEN="<token>" --name es02 -p 9201:9200 --net elastic -it docker.elastic.co/elasticsearch/elasticsearch:docker.elastic.co/elasticsearch/elasticsearch:8.4.2
7. 总是使用volume映射 `/usr/share/elasticsearch/data` 到主机，防止数据丢失
8. 挂载卷映射配置文件到本地：`-v full_path_to/custom_elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml`