---
title: Docker 基础
categories:
- Docker
feature_image: "https://picsum.photos/2560/600"
---

### 概述
Docker 解决了机器间环境配置不统一，弥补虚拟机的资源占用多（独占一部分内存和硬盘空间），冗余步骤多（虚拟机即是一个完整的操作系统），启动慢（启动操作系统）等问题。Linux 容器不是模拟一个完整的操作系统，而是对进程进行隔离。对于容器里面的进程来说，它接触到的各种资源都是虚拟的，从而实现与底层系统的隔离。由于容器是进程级别，具有启动快，资源占用少，体积小等优势。

Docker 属于 Linux 容器的一种封装，提供简单易用的容器使用接口。Docker 将应用程序与该程序的依赖，打包在一个文件里面。运行这个文件，就会生成一个虚拟容器。程序在这个虚拟容器里运行，就好像在真实的物理机上运行一样。有了 Docker，就不用担心环境问题。

### Image 镜像
Docker 把应用程序及其依赖打包到 image 文件中。只有通过这个文件，才能生成 Docker 容器。image 文件可以看成是容器的模板。Docker 根据 image 文件生成容器的实例。同一个 image 文件，可以生成多个同时运行的容器实例。

image 是二进制文件。实际开发中，一个 image 文件往往通过继承另一个 image 文件，加上一些个性化设置而生成。

### 容器文件
image 文件生产的容器实例，本身也是一个文件，称为容器文件。一旦容器生成，会同时存在两个文件：image 文件和容器文件。关闭容器并不会删除容器文件，只是容器暂停运行而已。
```bash
# 列出本机正在运行的容器
$ docker container ls

# 列出本机所有容器，包括终止运行的容器
$ docker container ls --all

# 终止容器运行
$ docker container kill [containerID]

# 移除容器
docker container rm [containerID]
```

### Dockerfile 文件
它是一个文本文件，用来配置 image。Docker 根据 该文件生成二进制的 image 文件。

`.dockerignore` 文件表示不要打包进入 image 文件。

### 实现 Docker 容器

```bash
# Dockerfile
# 设置基础镜像，必须为第一个命令
FROM node
# 将当前目录下的所有文件（除了.dockerignore排除的路径），都拷贝进入 image 文件的/app目录。
COPY . /app
# 指定接下来的工作路径为/app
WORKDIR /app
# 在/app目录下，运行npm install命令安装依赖。注意，安装后所有的依赖，都将打包进入 image 文件。
RUN npm install --registry=https://registry.npm.taobao.org
# 将容器 3000 端口暴露出来， 允许外部连接这个端口。
EXPOSE 3000
# 容器启动后自动执行
CMD node app.js
```

1. 新建 `.dockerignore` 文件，排除要打包进 image 文件
2. 新建 `Dockerfile` 文件，根据指令生成镜像的文件
3. 使用 `docker image build -t [container_name:1.0.0] .` 创建 image 文件，最后那个点表示当前目录
4. 使用 `docker containr run -p 8000:8000 -it [container_name:1.0.0] /bin/bash` 生成及运行容器
```bash
-p参数：容器的 3000 端口映射到本机的 8000 端口。
-it参数：容器的 Shell 映射到当前的 Shell，然后你在本机窗口输入的命令，就会传入容器。
container_name:1.0.0 文件的名字（如果有标签，还需要提供标签，默认是 latest 标签）。
/bin/bash：容器启动以后，内部第一个执行的命令。这里是启动 Bash，保证用户可以使用 Shell。
```

RUN命令与CMD命令的区别在哪里？简单说，RUN命令在 image 文件的构建阶段执行，执行结果都会打包进入 image 文件；CMD命令则是在容器启动后执行。另外，一个 Dockerfile 可以包含多个RUN命令，但是只能有一个CMD命令。

### 发布 image 文件
```bash
docker login
docker image build -t [username]/[repository]:[tag] .
docker image tag [imageName] [username]/[repository]:[tag]
docker image push [username]/[repository]:[tag]
```

### 其他命令
* `docker container run` 新建容器，每运行一次，就会新建一个容器
* `docker container start` 启动已生成但已经停止运行的容器
* `docker cotainer kill` 强行终止容器运行，相当于向容器里面的主进程发出 SIGKILL 信号。
* `docker container stop` 用来终止容器运行，相当于向容器里面的主进程发出 SIGTERM 信号，然后过一段时间再发出 SIGKILL 信号。应用程序收到 SIGTERM 信号以后，可以自行进行收尾清理工作，但也可以不理会这个信号。如果收到 SIGKILL 信号，就会强行立即终止，那些正在进行中的操作会全部丢失。
* `docker container logs` 查看 docker 容器的输出
* `docker container exec` 用于进入一个正在运行的 docker.如果docker run命令运行容器的时候，没有使用-it参数，就要用这个命令进入容器。一旦进入了容器，就可以在容器的 Shell 执行命令了。
* `docker container cp` 用于从正在运行的 Docker 容器里面，将文件拷贝到本机。



### 微服务
Docker 是一个容器工具，提供了虚拟环境。站在 Docker 的角度，软件就是容器的组合：业务逻辑容器、数据库容器、存储容器、队列容器等，使得软件可以拆分成若干个标准化容器，然后像搭积木一样组合起来。
这正是微服务（microservices）的思想：软件把任务外包出去，让各种外部服务完成任务，软件本身只是底层服务的调度中心和组装层。微服务很适合用 Docker 容器实现，每个容器承载一个服务。一台计算机同时运行多个容器，从而就能很轻松地模拟出复杂的微服务架构。

### Compose 文件
`Compose` 是 Docker 公司推出的一个工具软件，可以管理多个 Docker 容器组成的应用。定义一个 YAML 格式的配置文件，写好多个容器间的调用关系。只有一个命令，就能同时启动/关闭这些容器。
```bash
# 启动所有服务
$ docker-compose up
# 关闭所有服务
$ docker-compose stop
# 移除容器文件
$ docker-compose rm
```

### Shell 和 Exec 格式
Shell 格式: `RUN apt-get install python3`，`CMD echo "Hello world"` 或 `ENTRYPOINT echo "Hello world"`, 当指令执行时，shell 格式底层会调用 `/bin/sh -c <command>`

Exec 格式：`RUN ["apt-get", "install", "python3"]`, `CMD ["/bin/echo", "Hello world"]`, `ENTRYPOINT ["/bin/echo", "Hello world"]` 当指令执行时，会直接调用 `<command>`, 不会被 shell 解析。

CMD 和 ENTRYPOINT 推荐使用 Exec 格式，因为指令可读性更强，更容易理解。RUN 则两种格式都可以。

### RUN、CMD 和 ENTRYPOINT 区别
* RUN 执行命令并创建新的镜像层，RUN 经常用于安装软件包。
* CMD 设置容器启动后默认执行的命令及其参数，但 CMD 能够被 docker run 后面跟的命令行参数替换。
* ENTRYPOINT 配置容器启动时运行的命令。

RUN、 CMD、 ENTRYPOINT 三个指令的用途非常相识，不同在于，RUN指令是在容器被构建时运行的命令，而CMD、ENTRYPOINT 是启动容器时执行 shell 命令，而 RUN 会被 docker run 命令覆盖，但是ENTRYPOINT 不会被覆盖。事实上，docker run 命令指定的任何参数都会被当作参数再次传递给 ENTRYPOINT 指令。CMD、ENTRYPOINT 两个指令之间也可以一起使用。例如，我们 可以使用 ENTRYPOINT 的 exec 形式设置固定的默认命令和参数，然后使用任一形式的 CMD 来设置可能更改的其他默认值。

#### RUN
RUN 指令通常用于安装应用和软件包。
RUN 在当前镜像的顶部执行命令，并通过创建新的镜像层。Dockerfile 中常常包含多个 RUN 指令。

#### CMD
CMD 指令允许用户指定容器的默认执行的命令。此命令会在容器启动且 docker run 没有指定其他命令时运行。如果 docker run 指定了其他命令，CMD 指定的默认命令将被忽略。如果 Dockerfile 中有多个 CMD 指令，只有最后一个 CMD 有效。

#### ENTRYPOINT
ENTRYPOINT 指令可让容器以应用程序或者服务的形式运行。ENTRYPOINT 看上去与 CMD 很像，它们都可以指定要执行的命令及其参数。不同的地方在于 ENTRYPOINT 不会被忽略，一定会被执行，即使运行 docker run 时指定了其他命令。

### 最佳实践
* 使用 RUN 指令安装应用和软件包，构建镜像。
* 如果 Docker 镜像的用途是运行应用程序或服务，比如运行一个 MySQL，应该优先使用 Exec 格式的 ENTRYPOINT 指令。CMD 可为 ENTRYPOINT 提供额外的默认参数，同时可利用 docker run 命令行替换默认参数。
* 如果想为容器设置默认的启动命令，可使用 CMD 指令。用户可在 docker run 命令行中替换此默认命令。


### Dockerfile 常用指令
* FROM 指定基础镜像，必须为第一个命令
* RUN 构建镜像时执行 shell 命令
* EXPOSE 指定容器运行的服务端口
* CMD 启动容器时执行的 shell 命令（如果多个，最后一个生效）
* ENTRYPOINT 启动容器时执行 shell 命令（如果多个，最后一个生效）
* ADD 拷贝本地文件到容器中，对压缩文件自动解压
* COPY 拷贝本地文件到容器中，不支持解压
* ENV 设置环境变量
* VOLUME 用于指定持久化目录
* WORKDIR 设置 RUN、CMD、ENTRYPOINT、ADD、COPY 工作目录
* USER 设置 RUN、CMD、ENTRYPOINT 运行容器时运行用户
* OUTBUILD 设置镜像触发器
* LABLE 添加元数据，建议放在一个 LABLE 指令中执行
* ARG 指定默认参数
* STOPSIGNAL 设置停止容器时发送什么系统调用信号

