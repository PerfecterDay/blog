
# docker-compose
{docsify-updated}

- [docker-compose](#docker-compose)
  - [命令](#命令)
    - [命令选项](#命令选项)
    - [命令使用说明](#命令使用说明)
      - [build](#build)
      - [pull](#pull)
      - [run](#run)
    - [up](#up)
    - [ps](#ps)
    - [logs](#logs)
    - [stop](#stop)
    - [rm](#rm)
  - [Compose 模板文件](#compose-模板文件)
    - [services](#services)
    - [build](#build-1)
    - [command](#command)
    - [working\_dir](#working_dir)
    - [environment](#environment)
    - [volumes](#volumes)
    - [depends\_on](#depends_on)


我们知道使用一个 Dockerfile 模板文件，可以让用户很方便的定义一个单独的应用容器。然而，在日常工作中，经常会碰到需要多个容器相互配合来完成某项任务的情况。例如要实现一个 Web 项目，除了 Web 服务容器本身，往往还需要再加上后端的数据库服务容器，甚至还包括负载均衡容器等。

Compose 恰好满足了这样的需求。它允许用户通过一个单独的 docker compose.yml 模板文件（YAML 格式）来定义一组相关联的应用容器为一个项目（project）。

Compose 中有两个重要的概念：

+ 服务 (service)：一个应用的容器，实际上可以包括若干运行相同镜像的容器实例。
+ 项目 (project)：由一组关联的应用容器组成的一个完整业务单元，在 `docker-compose.yml` 文件中定义。

Compose 的默认管理对象是项目，通过子命令对项目中的一组容器进行便捷地生命周期管理。

Compose 项目由 Python 编写，实现上调用了 Docker 服务提供的 API 来对容器进行管理。因此，只要所操作的平台支持 Docker API，就可以在其上利用 Compose 来进行编排管理。


```
identidock: ➊  
  build: .  ➋ 
  ports:    ➌ 
  - "5000:5000"  
  environment: ➍ 
    ENV: DEV  
  volumes: ➎ 
    - ./app:/app
```

+ ➊ 第一行声明构建的容器名称。一个 YAML 文件中可以定义多个容器（在 Compose 的术语中称为服务）。
+ ➋ 这 里 的 build 关 键 字 告 诉 Compose， 这 个 容 器 的 镜 像 是 通 过 当 前 目 录（.） 下的Dockerﬁle 构建。每个容器的定义必须包括一个 build 或 image 关键字。image 关键字的值，是用于启动容器的镜像的标签或 ID，与 docker run 命令的参数相同。
+ ➌ ports 关键字相当于 docker run 命令的 -p 参数，用于声明对外开放的端口。这里我们把容器的 5000 端口映射到主机的 5000 端口。列出端口时可以不带引号，但最好避免这样做，因为当遇到像 56:56 这种值的时候，YAML 会把它解析为以 60 为基数的六十进制数字。
+ ➍ environment 关键字相当于 docker run 命令的 -e 参数，用来设置容器的环境变量。为了运行用于开发的 Flask Web 服务器，这里我们把 ENV 变量设成 DEV 。
+ ➎ volumes 关键字相当于 docker run 的 -v 参数，用于配置数据卷。这里与之前的做法一样，将 app 目录通过绑定挂载的方式挂载到容器，以便让我们能够从主机修改代码。

### 命令
`docker-compose [-f=<arg>...] [options] [COMMAND] [ARGS...]`

#### 命令选项
+ -f, --file FILE 指定使用的 Compose 模板文件，默认为 docker-compose.yml，可以多次指定。
+ -p, --project-name NAME 指定项目名称，默认将使用所在目录名称作为项目名。
+ --verbose 输出更多调试信息。
+ -v, --version 打印版本并退出。

#### 命令使用说明

##### build
格式为 `docker-compose build [options] [SERVICE...]`。重新建造由 Dockerﬁle 构建的镜像。除非镜像不存在，否则 up 命令不会执行构建的动作，因此需要更新镜像时便使用这个命令。服务容器一旦构建后，将会带上一个标记名，例如对于 web 项目中的一个 db 容器，可能是 web_db。可以随时在项目目录下运行 docker-compose build 来重新构建服务。  

选项包括：
+ `--force-rm` 删除构建过程中的临时容器。
+ `--no-cache` 构建镜像过程中不使用 cache（这将加长构建过程）。
+ `--pull` 始终尝试通过 pull 来获取更新版本的镜像。

##### pull
格式为 `docker-compose pull [options] [SERVICE...]`。 拉取服务依赖的镜像。  
选项：
+ `--ignore-pull-failures`: 忽略拉取镜像过程中的错误。

##### run
格式为 `docker-compose run [options] [-p PORT...] [-e KEY=VAL...] SERVICE [COMMAND] [ARGS...]`  
```
docker compose -f docker-compose-builder.yaml run --rm compile
```
在指定服务上执行一个命令。 启动一个容器，并运行一个一次性的命令。被依赖的容器会同时启动，除非用了 `--no-deps` 参数。  
请注意，如果运行时容器镜像已经存在，它不会重新构建镜像，如果要重新构建容器运行的镜像则需要重新 `build` 。
`--rm` 参数用来指示运行完成后删除容器，否则每次都会新建容器

#### up
启动所有在 Compose 文件中定义的容器，并且把它们的日志信息汇集一起。通常会使用 -d 参数使 Compose 在后台运行。

#### ps
获取由 Compose 管理的容器的状态信息。

#### logs
汇集由 Compose 管理的容器的日志，并以彩色输出。

#### stop
停止容器，但不会删除它们。

#### rm
删除已停止的容器。不要忘记使用 -v 参数来删除任何由 Docker 管理的数据卷。

一个普通的工作流程以 `docker compose up -d` 命令启动应用程序开始。`docker compose logs` 和 `ps` 命令可以用来验证应用程序的状态，还能帮助调试。修改代码后，先执行 `docker compose build` 构建新的镜像，然后执行 `docker compose up -d` 取代运行中的容器。注意，Compose 会保留原来容器中所有旧的数据卷，这意味着即使容器更新后，数据库和缓存也依旧在容器内（这很可能会造成混淆，因此要特别小心）。如果你修改了 Compose 的 YAML 文件，但不需要构建新镜像，可以通过 `up -d` 参数使Compose 以新的配置替换容器。如果想要强制停止 Compose 并重新创建所有容器，可以使用 `--force-recreate/-f` 选项来达到目的。

当你不再需要使用该应用时，可以执行 `docker compose stop` 来停止应用程序。假设代码没有变更，可以通过 `docker compose start` 或 `up` 来重启相同的容器。使用 `docker compose rm` 彻底把容器删除。


### Compose 模板文件
```
services:
  go_plugin_compile:
    build:
      context: simple
      dockerfile: ../../shared/golang/Dockerfile
      target: golang-base
    command: >
      bash -c "
      cd examples/golang-http/simple
      && go build -o simple.so -buildmode=c-shared .
      && cp ./simple.so /output"
    working_dir: /source
    environment:
    - GOFLAGS=-buildvcs=false
    volumes:
    - ../..:/source
    - ./lib:/output
```

#### services
指定服务（容器）

#### build
指定 Dockerfile 所在文件夹的路径（可以是绝对路径，或者相对 docker-compose.yml 文件的路径）。 Compose 将会利用它自动构建这个镜像，然后使用这个镜像。
+ 可以使用 `context` 指令指定构建上下文所在文件夹的路径，并且默认在该路径下寻找`Dockerfile`。
+ 使用 `dockerfile` 指令指定 `Dockerfile` 文件路径，如果是相对路径则是相对于 `context` 指定的路径。
+ 使用 `arg` 指令指定构建镜像时的变量。
+ 使用 `target` 指令指定多阶段构建镜像时的目标阶段。

#### command
覆盖容器启动后默认执行的命令。

#### working_dir
指定容器中工作目录。

#### environment
设置环境变量。你可以使用数组或字典两种格式。**只给定名称的变量会自动获取运行 Compose 主机上对应变量的值，可以用来防止泄露不必要的数据**。
```
environment:
  RACK_ENV: development
  SESSION_SECRET:

environment:
  - RACK_ENV=development
  - SESSION_SECRET
```

#### volumes
数据卷所挂载路径设置。可以设置为宿主机路径(`HOST:CONTAINER`)或者数据卷名称(`VOLUME:CONTAINER`)，并且可以设置访问模式 （`HOST:CONTAINER:ro`）。
该指令中路径支持相对路径。
```
volumes:
 - /var/lib/mysql
 - cache/:/tmp/cache
 - ~/configs:/etc/configs/:ro
```

如果路径为数据卷名称，必须在文件中配置数据卷。


#### depends_on
"service_started", "service_healthy", "service_completed_successfully"