---
title: docker2-dockerfile
date: 2018-09-03 11:29:57
tags: docker
categories: docker
---

## Dockerfile简介

如果我们可以把镜像的每一层修改、安装、构建、操作的命令都写入一个脚本，用这个脚本来构建、定制镜像，那么之前提及的无法重复的问题、镜像构建透明性的问题、体积的问题就都会解决。这个脚本就是 Dockerfile。

Dockerfile 是一个文本文件，其内包含了一条条的**指令(Instruction)**，每一条指令构建一层，因此每一条指令的内容，就是描述该层应当如何构建。

在一个空白目录中，建立一个文本文件，并命名为 `Dockerfile`：

```
$ mkdir mynginx
$ cd mynginx
$ touch Dockerfile
```

其内容为：

```
FROM nginx
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```

这个 Dockerfile 很简单，一共就两行。涉及到了两条指令，`FROM` 和 `RUN`。

接下来就是在dockerfile目录下执行`docker build -t zhq/static_web .`

`-t`为新镜像设置仓库和名称，

`.`dockerfile在当前路径

## FROM 指定基础镜像

所谓定制镜像，那一定是以一个镜像为基础，在其上进行定制。就像之前运行了一个 `nginx` 镜像的容器，再进行修改一样，基础镜像是必须指定的。而 `FROM` 就是指定**基础镜像**，因此一个 `Dockerfile` 中 `FROM` 是必备的指令，并且**必须是第一条指令**。

在 [Docker Store](https://store.docker.com/) 上有非常多的高质量的官方镜像，有可以直接拿来使用的服务类的镜像，如 [`nginx`](https://store.docker.com/images/nginx/)、[`redis`](https://store.docker.com/images/redis/)、[`mongo`](https://store.docker.com/images/mongo/)、[`mysql`](https://store.docker.com/images/mysql/)、[`httpd`](https://store.docker.com/images/httpd/)、[`php`](https://store.docker.com/images/php/)、[`tomcat`](https://store.docker.com/images/tomcat/) 等；也有一些方便开发、构建、运行各种语言应用的镜像，如 [`node`](https://store.docker.com/images/node)、[`openjdk`](https://store.docker.com/images/openjdk/)、[`python`](https://store.docker.com/images/python/)、[`ruby`](https://store.docker.com/images/ruby/)、[`golang`](https://store.docker.com/images/golang/) 等。可以在其中寻找一个最符合我们最终目标的镜像为基础镜像进行定制。

## RUN 执行命令

`RUN` 指令是用来执行命令行命令的。由于命令行的强大能力，`RUN` 指令在定制镜像时是最常用的指令之一。其格式有两种：

- *shell* 格式：`RUN <命令>`，就像直接在命令行中输入的命令一样。刚才写的 Dockerfile 中的 `RUN` 指令就是这种格式。

```
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```

- *exec* 格式：`RUN ["可执行文件", "参数1", "参数2"]`，这更像是函数调用中的格式。

**但是RUN虽然像shell脚本一样可以执行命令,但是不能像 Shell 脚本一样把每个命令对应一个 RUN**

从Docker1.0起，Dockerfile 中每一个指令都会建立一层，`RUN` 也不例外。每一个 `RUN` 的行为，就和刚才我们手工建立镜像的过程一样：新建立一层，在其上执行这些命令，执行结束后，`commit` 这一层的修改，构成新的镜像。

下图是docke文件系统图示

![](docker2-dockerfile/DockerFileSystem.png)



可以看到FROM，add,VOLUME,CMD这些Dockerfile命令都会在docker镜像上添加一层，如果每一个命令对应一个RUN,则镜像的层数就会特别多，导致镜像特别臃肿和繁杂，也容易出错

所以解决办法一般是通过`RUN`和`&&`结合

```dockerfile
FROM ubuntu
RUN apt-get update && apt-get install vim
```

## CMD 容器启动命令

`CMD` 指令的格式和 `RUN` 相似，也是两种格式：

- `shell` 格式：`CMD <命令>`
- `exec` 格式：`CMD ["可执行文件", "参数1", "参数2"...]`
- 参数列表格式：`CMD ["参数1", "参数2"...]`。在指定了 `ENTRYPOINT` 指令后，用 `CMD` 指定具体的参数。

CMD指令用于指定一个容器启动时要运行的命令，和RUN不同的是，RUN是指定镜像被构建时要运行的命令，而CMD是指容器被启动时运行的命令。这和docker run 命令启动容器时指定要运行的命令非常类似

`docker run -it jamtyr01/static_web /bin/true`和

在dockerfile中使用 `CMD ["/bin/true"]`

也可以给要运行的命令指定参数 `CMD ["/bin/bash","-l"]`

这样就把-l标志穿个了`/bin/bash`

使用CMD命令,内部其实是在命令前加了/bin/sh -c，所以其实是

**在dockerfile中只能指定一条CMD指令，如果指定了多条CMD指令，业之后最后一个会被使用**。

## ENTRYPOINT

ENTRYPOINT和CMD非常类似，

ENTRYPOINT命令相对与CMD命令不容易在启动容器的时候被覆盖。docker run命令行中指定的任何参数都会被当做参数再次传递给ENTRYPOINT指令中指定的指令

比如:

`ENTRYPOINT ["/usr/sbin/nginx"]`







## COPY 复制文件

格式：

- `COPY <源路径>... <目标路径>`
- `COPY ["<源路径1>",... "<目标路径>"]`

和 `RUN` 指令一样，也有两种格式，一种类似于命令行，一种类似于函数调用。

## ADD 更高级的复制文件

`ADD` 指令和 `COPY` 的格式和性质基本一致。但是在 `COPY` 基础上增加了一些功能。

如果 `<源路径>` 为一个 `tar` 压缩文件的话，压缩格式为 `gzip`, `bzip2` 以及 `xz` 的情况下，`ADD` 指令将会自动解压缩这个压缩文件到 `<目标路径>` 去。

在 Docker 官方的 [Dockerfile 最佳实践文档](https://yeasy.gitbooks.io/docker_practice/appendix/best_practices.html) 中要求，尽可能的使用 `COPY`，因为 `COPY` 的语义很明确，就是复制文件而已，而 `ADD` 则包含了更复杂的功能，其行为也不一定很清晰。最适合使用 `ADD` 的场合，就是所提及的需要自动解压缩的场合。 

## MAINTAINER 作者

MAINTAINER这条指令告诉docker镜像的作者以及电子邮件，方便联系

`MAINTAINER zhqqqy "123@gmail.com"`

## ENV

ENV是在build的时候，可以定义一些变数，让后面指令在执行时候可以参考...

```
From resin/rpi-raspbian
ENV NODE_VER=node-v5.9.1-linux-armv7l
ADD ./${NODE_VER} /
RUN ln -s /${NODE_VER} /node
ENV PATH=/node/bin:$PATH
CMD ["node"]
```

上面Dockerfile的第二行，定义NODE_VER变数，让第三行中的ADD参考，因此ADD会在执行时间，使用ENV所定义的值来做ADD的动作。

注意：在Dockerfile中若没有第二行的指定，则第三行会参考到空字串。

## ARG

ARG在build时候是可以从外部以--build-arg带入的变数，让build的动作可结合外部的指令给定一些建构时候所需的参数...（ARG参数只在docker build的时候存在，docker run的时候不存在了，所以，在docker build的时候可以使用${}）

```
From resin/rpi-raspbian
ARG NODE_VER
ADD ./${NODE_VER:-node-v5.9.1-linux-armv7l} /
RUN ln -s /${NODE_VER} /node
ENV PATH=/node/bin:$PATH
CMD ["node"]
```

上面Dockerfile在第二行的部分，会指定一个NODE_VER的变数，而在第三行时候，会引用该变数在本地端找到指令列所指定的字串目录进行复制到根目录(node-v5.9.0 -linux-armv7l)，如果未给定时候，则是以预设的node-v5.9.1-linux-armv7l为值...

Build的时候，可以这样下参数：

```
docker build --build-arg NODE_VER=node-v5.9.0-linux-armv7l .
```

 