# DockerFile

> Dockerfile是为快速构建docker image而设计的，当你使用docker build 命令的时候，docker 会读取当前目录下的命名为Dockerfile(首字母大写)的纯文本文件并执行里面的指令构建出一个docker image。这比SaltStack的配置管理要简单的多，不过还是要掌握一些简单的指令。 Dockerfile 由一行行命令语句组成，并且支持以#开头的注释行。指令是不区分大小写的，但是通常我们都大写，用于和后面的内容参数进行区分。

## 简单示例

下面我们通过构建一个Nginx的镜像来学习Dockerfile. 首先实现创建好目录，我这在opt下创建目录dockerfile，然后在dockerfile目录下创建nginx目录，在nginx目录下新建一个Dockerfile文件。Dcockerfile这个文件的首字母要大写，不然有可能不会被识别。

```shell
[root@linux-node1 ~]# cd /opt/
[root@linux-node1 opt]# cd dockerfile/
[root@linux-node1 dockerfile]# cd nginx/
[root@linux-node1 nginx]# echo "nginx in Docker ,hahahah" >index.html
[root@linux-node1 nginx]# cat Dockerfile 
# This Dockerfile is used to practice 
# Version: 1.0
# Author: lamber

# Base Image
FROM centos

# Who will take care of this image
MAINTAINER lamber 1020561033@qq.com

# Prepare Epel
RUN rpm -ivh https://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm

# Install Nginx and deal with the config file
RUN yum -y install nginx && yum clean all
RUN echo "daemon off;" >> /etc/nginx/nginx.conf
ADD index.html /usr/share/nginx/html/index.html

# Run
EXPOSE 80
CMD ["nginx"]
```

说明：

- FROM：这个镜像的妈是谁，指定基础镜像，除了注释外的第一行必须是他。如果本地没有这个镜像，它会给你pull下来。
- MAINTAINER：告诉别人谁负责养他，指定维护者的信息
- RUN：你想让他干啥，在命令前面加上RUN就可以了。
- ADD：给它点创业资金，copy文件进去，会自动解压
- WORKDIR：我是cd，今天刚化了妆（设置当前的工作目录）
- VOLUME：给它一个存放行李的地方，设置卷，挂载主机目录
- EXPOSE：它要打开的门是啥，指定对外的端口
- CMD：奔跑吧，兄弟，指定容器启动后要干的事情，这里双引一下，单引可能出现问题。。。。

构建docker镜像（后面那个点就是在当前目录，意思是告诉docker去哪里找这个dockerfile，一个点就是在当前目录找dockerfile，这个你也可以写绝对路径也是ok的），然后创建容器：

```shell
[root@linux-node1 nginx]# docker build -t mynginx:v2 .
[root@linux-node1 nginx]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
mynginx             v2                  5104f2ed9887        5 seconds ago       280.7 MB
nginx/mynginx       v1                  81f1607bb8a0        32 minutes ago      355 MB
docker.io/nginx     latest              5e69fe4b3c31        3 days ago          182.5 MB
docker.io/centos    latest              98d35105a391        2 weeks ago         192.5 MB
[root@linux-node1 nginx]# docker run --name mynginxv2 -d -p 82:80 mynginx:v2 nginx
a1efc632ba6d2412c3bc9fded592ca289c0f6c14dd1832118fc52539a2c1123f
```

起来以后我们就可以在网页端进行测试了！

### Docker Build

#### Docker build from Local

上面的命令使用docker build的同时为我们的新镜像打了tag，但是后面跟了一个`.`。这个点其实就是当前目录的意思，目的就是为了让docker build在当前目录下查找Dockerfile，但是这个路径并不是指定Dockerfile的路径，而是指定上下文的路径。

首先我们要理解 docker build 的工作原理。Docker 在运行时分为Docker引擎 （也就是服务端守护进程）和客户端工具。Docker 的引擎提供了一组 REST API， 被称为 Docker Remote API，而如 docker 命令这样的客户端工具，则是通过这组 API 与 Docker 引擎交互，从而完成各种功能。因此，虽然表面上我们好像是在 本机执行各种 docker 功能，但实际上，一切都是使用的远程调用形式在服务端 （Docker 引擎）完成。也因为这种 C/S 设计，让我们操作远程服务器的 Docker 引 擎变得轻而易举。

当我们进行镜像构建的时候，并非所有定制都会通过 RUN 指令完成，经常会需要 将一些本地文件复制进镜像，比如通过 COPY 指令、 ADD 指令等。而 docker build 命令构建镜像，其实并非在本地构建，而是在服务端，也就是 Docker 引擎中构建的。那么在这种客户端/服务端的架构中，如何才能让服务端获得本地文件呢？

这就引入了上下文的概念。当构建的时候，用户会指定构建镜像上下文的路径， docker build 命令得知这个路径后，会将路径下的所有内容打包，然后上传给Docker引擎。这样 Docker引擎收到这个上下文包后，展开就会获得构建镜像所需的一切文件，比如：

```dockerfile
COPY ./package.json /app/
```

这并不是要复制执行docker build命令所在的目录下的 package.json，也不是复制 Dockerfile 所在目录下的 package.json ,而是复制上下文（context） 目录下的package.json。

因此， COPY 这类指令中的源文件的路径都是相对路径。这也是初学者经常会问 的为什么 COPY ../package.json /app 或者 COPY /opt/xxxx /app 无法工作的原因，因为这些路径已经超出了上下文的范围，Docker 引擎无法获得这些位置的文件。如果真的需要那些文件，应该将它们复制到上下文目录中去。

现在就可以理解如上构建命令 中的这个 `.` ，实际 上是在指定上下文的目录， docker build 命令会将该目录下的内容打包交给 Docker 引擎以帮助构建镜像。（在构建过程中我们是可以看到发往Docker daemon的日志的）

一般的使用方法是，对应的项目有对应的目录存放，Dockerfile放到对应的项目里面去，如果有文件需要传到镜像里面去的话，那么就应该把对应的文件也复制一份放到这里。如果目录下有些东西确实不希望构建的时候传给Docker引擎，那么可以使用.gitignore一样的语法来写一个.dockerignore，该文件是用来剔除不需要作为上下文传递给Docker Engine的文件。

默认情况下，如果不指定Dockerfile的话，会将上下文目录下的名为Dockerfile的文件作为Dockerfile，但是实际上文件名不一定必须得是Dockerfile，也不是要求Dockerfile必须位于上下文目录，可以通过-f参数去指定某一个文件来作为Dockerfile，但是习惯性的都是默认使用Dockerfile为名称，而且放到上下午目录环境里。

#### Docker build from Remote

这个命令指定了构建所需要的Git Repo，并且默认执行master分支，构建目录为8.14，然后Docker会自己去clone这个项目，切换到指定分支，并进入到指定目录后开始创建。

```shell
$ docker build https://github.com/twang2218/gitlab-ce-zh.git #:8.14
docker build https://github.com/twang2218/gitlab-ce-zh.git\#:8.14
Sending build context to Docker daemon 2.048 kB
Step 1 : FROM gitlab/gitlab-ce:8.14.0-ce.0
8.14.0-ce.0: Pulling from gitlab/gitlab-ce
aed15891ba52: Already exists
773ae8583d14: Already exists
……
```

#### Docker build from tar

如果所给出的URL不是Git repo，而是个tar压缩包，那么Docker引擎会下载这个包，并自动解压缩，以其作为上下文，开始构建。

```shell
docker build http://server/context.tar.gz
```

#### Docker build from stdin

```shell
docker build - < Dockerfile
或者
cat Dockerfile | docker build -
```

如果标准输入传入的是文本文件，则将其视为 Dockerfile ，并开始构建。这种 形式由于直接从标准输入中读取 Dockerfile 的内容，它没有上下文，因此不可以像 其他方法那样可以将本地文件 COPY 进镜像之类的事情。

```shell
docker build - < context.tar.gz
```

如果发现标准输入的文件格式是 gzip 、 bzip2 以及 xz 的话，将会使其为上 下文压缩包，直接将其展开，将里面视为上下文，并开始构建。

## Dockerfile指令详解

### FROM

> 在 Docker Store 上有非常多的高质量的官方镜像，有可以直接拿来使用的服务类的 镜像，如 nginx 、 redis 、 mongo 、 mysql 、 httpd 、 php 、 tomcat等；也有一些方便开发、构建、运行各种语言应用的镜像，如 node 、 openjdk 、 python 、 ruby 、 golang 等。可以在其中寻找一个最符合我们最终目标的镜像为基础镜像进行定制。
>
> 如果没有找到对应服务的镜像，官方镜像中还提供了一些更为基础的操作系统镜 像，如 ubuntu 、 debian 、 centos 、 fedora 、 alpine 等，这些操作系统的软件库为我们提供了更广阔的扩展空间。
>
> 除了选择现有镜像为基础镜像外，Docker 还存在一个特殊的镜像，名为 scratch 。这个镜像是虚拟的概念，并不实际存在，它表示一个空白的镜像。
>
> ```dockerfile
> FROM scratch
> ...
> ```
>
> 如果你以 scratch 为基础镜像的话，意味着你不以任何镜像为基础，接下来所写 的指令将作为镜像第一层开始存在。不以任何系统为基础，直接将可执行文件复制进镜像的做法并不罕见，比如 swarm 、 coreos/etcd 。对于 Linux 下静态编译的程序来说，并不需要有操作 系统提供运行时支持，所需的一切库都已经在可执行文件里了，因此直接 FROM scratch 会让镜像体积更加小巧。使用 Go 语言 开发的应用很多会使用这种方式 来制作镜像，这也是为什么有人认为 Go 是特别适合容器微服务架构的语言的原因之一。
>

- 格式：FROM `<image>` 或FROM `<image>:<tag>`。
- 解释：FROM是Dockerfile里的第一条指令（必须是），后面跟有效的镜像名（如果该镜像你的本地仓库没有则会从远程仓库Pull取）。然后后面的其它指令FROM的镜像中执行。一般在FROM的时候建议带上image的tag，默认如果不带tag的话，tag就是latest。

### MAINTAINER

- 格式：MAINTAINER  `<name>`
- 解释：指定维护者信息。

### RUN

> RUN命令会在前一条命令创建出的镜像的基础上创建一个容器，并在容器中运行命令，在命令运行结束后commit容器为新的镜像。新的镜像会被Dockerfile中的下一条指令所使用。因此要合理使用RUN，避免建立不必要的层，Union FS 是有最大层数限制的，比如 AUFS，曾经是最大不得超过 42 层，现在是 不得超过 127 层。这里要相对合理使用，减少不必要的层数不意味着将所有的命令都写在一个RUN指令中，提交镜像本身是廉价的，合理的分层是符合Docker的核心概念，这很像源码版本控制。
>
> 这里合理的分层指的是一层就干一件事情，比如编译/安装redis，合成一层就可以了。编译安装过程涉及到的命令使用\\和\&\&合并起来。
>
> 最后进行无用数据得清理是一个好的习惯，比如你拿了一个tar.gz的源码包进去，那么安装一切就绪以后在最后应该做清理的工作，删掉无用的内容，比如为了编译构建所需要的软件，下载的压缩包，yum或者是apt-get的缓存文件，避免逐层构建起来的镜像文件是一个臃肿的镜像。
>
> RUN支持两种模式，RUN command的模式称为shell模式，调用的是/bin/sh -c来运行；第二种模式为exec模式，这种模式下，命令是直接运行的，而不调用shell程序，exec格式中的参数会被当成JSON数组被Docker解析，因此这里面的参数都要被双引号（双引号必须）引起来，单引号不是标准的JSON格式。
>
> exec模式下环境变量的参数不会被替换，因为exec模式并不调用shell，因此比如你`CMD ["echo","$HOME"]`的时候\$HOME并不会被变量替换掉。如果希望被变量替换掉可以调用shell，比如`CMD ["sh","-c","echo","$HOME"]`

- 格式：

  - RUN `<command>` 
  - RUN ["executable", "param1", "param2"]。

-  解释：运行命令，命令较长的时候，为了避免RUN多次，建立多个不必要的层（*很多运行时不需要的东西，都被装进了镜像里，比如编译环境、更新的软件包等等。结果就是产生 非常臃肿、非常多层的镜像，不仅仅增加了构建部署的时间，也很容易出错*），因此我们可以使用\来换行，综合成一行来处理，参考如下：(推荐使用exec格式)

  ```dockerfile
  FROM debian:jessie
  
  RUN buildDeps='gcc libc6-dev make' \
  	&& apt-get update \
  	&& apt-get install -y $buildDeps \
  	&& wget -O redis.tar.gz "http://download.redis.io/releases/redis-3.2.5.tar.gz" \
  	&& mkdir -p /usr/src/redis \
  	&& tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1 \
  	&& make -C /usr/src/redis \
  	&& make -C /usr/src/redis install \
  	&& rm -rf /var/lib/apt/lists/* \
  	&& rm redis.tar.gz \
  	&& rm -r /usr/src/redis \
  	&& apt-get purge -y --auto-remove $buildDeps
  ```

### CMD

```
格式：
   CMD ["executable","param1","param2"] 使用 exec 执行，推荐方式；
   CMD command param1 param2 在 /bin/sh 中执行，提供给需要交互的应用；
   CMD ["param1","param2"] 提供给ENTRYPOINT的默认参数；
```

解释： CMD指定容器启动是执行的命令，每个Dockerfile只能有一条CMD命令，如果指定了多条，只有最后一条会被执行。如果你在启动容器的时候也指定的命令，那么会覆盖Dockerfile构建的镜像里面的CMD命令。

#### ENTRYPOINT

```
格式：
   ENTRYPOINT ["executable", "param1","param2"]
   ENTRYPOINT command param1 param2（shell中执行）。
```

解释：和CMD类似都是配置容器启动后执行的命令，并且不可被 docker run 提供的参数覆盖。 每个 Dockerfile 中只能有一个ENTRYPOINT，当指定多个时，只有最后一个起效。ENTRYPOINT没有CMD的可替换特性，也就是你启动容器的时候增加运行的命令不会覆盖ENTRYPOINT指定的命令。

所以生产实践中我们可以同时使用ENTRYPOINT和CMD，例如：

```
ENTRYPOINT ["/usr/bin/rethinkdb"]
CMD ["--help"]
```

#### USER

- 格式：USER daemon
- 解释：指定运行容器时的用户名和UID，后续的RUN指令也会使用这里指定的用户。

#### EXPOSE

- 格式：EXPOSE `<port> [<port>…]`
- 解释：设置Docker容器内部暴露的端口号，如果需要外部访问，还需要启动容器时增加-p或者-P参数进行分配。

#### ENV

```
格式：
ENV <key> <value>
ENV <key>=<value> ...
```

解释：设置环境变量，可以在RUN之前使用，然后RUN命令时调用，容器启动时这些环境变量都会被指定

#### ADD

```
格式：
   ADD <src>... <dest>
   ADD ["<src>",... "<dest>"]
```

解释：将指定的\<src\>复制到容器文件系统中的\<dest\> 所有拷贝到container中的文件和文件夹权限为0755,uid和gid为0 如果文件是可识别的压缩格式，则docker会帮忙解压缩

#### VOLUME

- 格式：VOLUME ["/data"]
- 解释：可以将本地文件夹或者其他container的文件夹挂载到container中。

#### WORKDIR

- 格式：WORKDIR/path/to/workdir
- 解释：切换目录，为后续的RUN、CMD、ENTRYPOINT 指令配置工作目录。

可以多次切换(相当于cd命令)， 也可以使用多个WORKDIR 指令，后续命令如果参数是相对路径，则会基于之前命令指定的路径。

```
例如
WORKDIR /a
WORKDIR b
WORKDIR c
RUN pwd
则最终路径为 /a/b/c。
```

#### ONBUILD

ONBUILD 指定的命令在构建镜像时并不执行，而是在它的子镜像中执行

#### ARG

- 格式：ARG `<name>[=<default value>]`
- 解释：ARG指定了一个变量在docker build的时候使用，可以使用--build-arg  `<varname>=<value>`来指定参数的值，不过如果构建的时候不指定就会报错。

参考自运维社区：https://mp.weixin.qq.com/s/A8KSp3zmpL_u9a2aYv8WYw