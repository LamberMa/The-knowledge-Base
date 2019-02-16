# Docker

> Docker是Docker.Inc公司开源的一个基于LXC技术之上构建的Container容器引擎，源代码托管在github上，基于Go语言并遵循Apache2.0协议开源。
>
> Docker是通过内核虚拟化技术（namespaces以及cgroups等）来提供容器的资源隔离与安全保证等。由于Docker通过系统层的虚拟化实现隔离，所以Docker容器在运行的时候不需要类似虚拟机额外的操作系统的开销（经验来看一般一个完整的系统大概会消耗一个物理主机6%左右的性能），提高资源利用率。

**docker的核心理念**

- 构建（Build）
- 运输（Ship）
- 运行（Run）

## Docker介绍

### 容器VS虚拟化

![](http://tuku.dcgamer.top/18-5-11/23003726.jpg)

虚拟机是完整的资源隔离。容器是隔离不是虚拟，它不需要虚拟出来操作系统环境。

![](http://tuku.dcgamer.top/18-5-11/3645079.jpg)

### Docker可以做什么

![](http://tuku.dcgamer.top/18-5-11/87167288.jpg)

1. 简化配置
2. 代码流水线的管理，让开发，测试生产都跑在同样一个环境，保证开发环境的一致性。
3. 提高开发效率
4. 应用的隔离（虚拟机也可以做到）
5. 整合服务器，通过docker的隔离能力整合多个服务器降低成本。
6. 调试能力
7. 多租户环境
8. 快速部署

### Docker改变了什么？

- 面向产品：产品交付
- 面向开发：简化环境配置
- 面向测试：多版本测试
- 面向运维：环境一致性，可以环境和代码一起发布，回滚的时候也可以做到一起回滚。
- 面向架构：自动化扩容（微服务）

## Docker的安装

> 最新的安装信息请以官方文档的安装信息为准，[官方Doc文档地址](https://docs.docker.com/install/linux/docker-ce/centos/#set-up-the-repository)

以下是一个简单的安装步骤：

```shell
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install docker-ce
systemctl start docker
```

查看docker启动状态：

```shell
[root@localhost ~]# ps -ef | grep docker
root     31551     1  2 00:43 ?        00:00:00 /usr/bin/dockerd
root     31555 31551  0 00:43 ?        00:00:00 docker-containerd --config /var/run/docker/containerd/containerd.toml
root     31667 31082  0 00:43 pts/0    00:00:00 grep --color=auto docker
```

拉取docker镜像，这里如果直接拉的话可能会因为网速慢导致拉不下来，因此我们可以配置一下国内的docker镜像源

### 配置国内docker镜像源

> 国内的Docker镜像源有很多可以选择，[参考原文](https://www.cnblogs.com/anliven/p/6218741.html)
>
> - [DaoCloud - Docker加速器](https://dashboard.daocloud.io/)
> - [阿里云 - 开发者平台](https://dev.aliyun.com/)
> - [微镜像 - 希云cSphere](https://csphere.cn/hub)
> - [镜像广场 - 时速云](https://hub.tenxcloud.com/)
> - [灵雀云](https://hub.alauda.cn/)
> - [网易蜂巢](https://c.163.com/)

这里以阿里云的Docker加速器为例，注册并登陆[阿里云 - 开发者平台](https://dev.aliyun.com/)之后，在首页点击“创建我的容器镜像”，然后就会来到阿里云的服务面板。点击加速器标签。根据提示输入Docker登录时需要使用的密码（后期可更改），用户名就是登录阿里云的用户名。在出现的页面中，可以得到一个专属的镜像加速地址，类似于“[https://1234abcd.mirror.aliyuncs.com](https://1234abcd.mirror.aliyuncs.com/)”。根据页面中的“操作文档”信息，配置自己的Docker加速器。

或者，登录[阿里云 - 容器Hub服务控制台](https://cr.console.aliyun.com/)之后，点击加速器标签，也会出现相应信息。

![](http://tuku.dcgamer.top/18-5-11/49585375.jpg)

**其他国内Docker镜像的配置方法**

国内其他Docker加速器配置方法和阿里云的差不多：

- 注册账号，获取专属的镜像加速地址
- 根据提示和系统类型，升级，并配置Docker，然后重启。
- 验证操作结果

**手动配置Docker加速器**

配置Docker加速器的本质就是把Docker配置文件中的镜像下载地址由默认的Docker Hub地址变为国内镜像的加速地址。

```shell
/lib/systemd/system/docker.service
/etc/systemd/system/docker.service

# 比如centos7中
将 OPTIONS=--registry-mirror=http://abcd1234.m.daocloud.io加入到docker的配置文件/etc/sysconfig/docker中，然后重启Docker
```

## Docker镜像

### 镜像搜索

```shell
# docker有一个docker hub仓库，这个其实和github很相似。仓库中存了很多已经构建好的镜像。
# 大多数情况我们可以直接下载一个现成的镜像而不需要我们自己构建
COMAND:

$ sudo docker search TERM

OPTIONS:

--automated=false     是否仅显示自动创建的镜像  
--no-trunc=false      是否截断输出  
-s, --stars=0         仅显示至少有x颗星的镜像  

示例:

[root@maxiaoyu ~]# docker search nginx
INDEX       NAME                    DESCRIPTION               STARS     OFFICIAL   AUTOMATED
docker.io   docker.io/nginx    Official build of Nginx.       7127        [OK]
```

我这里截取了搜索出结果的第一行，当然如果你实际搜索的话会有很多行的内容的，其中包含了官方的以及各种自制的。

- NAME：镜像名称


- DESCRIPTION：也就是镜像的描述，


- STARS类似于github里面的stars，表示点赞和热度。


- OFFICIAL：指docker标准库, 由docker 官方建立. 用户建立的image则会有userid的prefix.


- automated builds ：则是通过代码版本管理网站结合docker hub提供的接口生成的, 例如github, bitbucket, 你需要注册docker hub, 然后使用github或bitbucket的在账户链接到docker hub, 然后就可以选择在github或bitbucket里面的项目自动build docker image, 这样的话只要代码版本管理网站的项目有更新, 就会触发自动创建image.对于的image属于什么版本，下面的“[OK]”就会打在什么地方

### 获取镜像

获取镜像的方式有两种，一种是直接去pull我们刚才用docker search搜索到的镜像：

```shell
[root@maxiaoyu ~]# docker pull docker.io/nginx   
Using default tag: latest
Trying to pull repository docker.io/library/nginx ... 
latest: Pulling from docker.io/library/nginx
bc95e04b23c0: Pull complete 
110767c6efff: Pull complete 
f081e0c4df75: Pull complete 
Digest: sha256:004ac1d5e791e705f12a17c80d7bb1e8f7f01aa7dca7deee6e65a03465392072
[root@maxiaoyu ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
docker.io/nginx     latest              1e5ab59102ce        2 weeks ago         108.3 MB
docker.io/centos    latest              328edcd84f1b        12 weeks ago        192.5 MB
```

导入本地的docker包：

```shell
docker load --input centos.tar
或者
docker load < nginx.tar
```

我们也可以通过docker的导出功能将我们pull下来的image导出成一个tar包，生成的tar包会保存在当前的目录：

```shell
docker save -o centos.tar centos
```

使用docker inspect去查看image的内容：

```shell
docker inspect docker.io/nginx:latest   
```

列出当前下载的所有的镜像：

```shell
[root@localhost ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               latest              ae513a47849c        10 days ago         109MB
centos              latest              e934aafc2206        4 weeks ago         199MB
```

如果我们在docker pull的时候不加额外的参数，那么下载的就是最新的，可以在tag中看到是latest的。但是我们可以指定版本，比如：

```shell
# 参考示例
docker pull centos:v1.0
```

### 删除镜像

```shell
docker rmi 镜像ID，比如：
[root@maxiaoyu ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
docker.io/nginx     latest              1e5ab59102ce        2 weeks ago         108.3 MB
docker.io/centos    latest              328edcd84f1b        12 weeks ago        192.5 MB
[root@maxiaoyu ~]# docker rmi 1e5ab59102ce
Untagged: docker.io/nginx:latest
Untagged: docker.io/nginx@sha256:004ac1d5e791e705f12a17c80d7bb1e8f7f01aa7dca7deee6e65a03465392072
Deleted: sha256:1e5ab59102ce46c277eda5ed77affaa4e3b06a59fe209fe0b05200606db3aa7a
Deleted: sha256:182a54bd28aa918a440f7edd3066ea838825c3d6a08cc73858ba070dc4f27211
Deleted: sha256:a527b2a06e67cab4b15e0fd831882f9e6a6485c24f3c56f870ac550b81938771
Deleted: sha256:cec7521cdf36a6d4ad8f2e92e41e3ac1b6fa6e05be07fa53cc84a63503bc5700
```

实际上是按照image的id去删除的，当然如果你的image属于被其他容器引用的情况下的话是无法删除的。会收到如下的报错信息：

```shell
[root@maxiaoyu ~]# docker rmi 328edcd84f1b
Error response from daemon: conflict: unable to delete 328edcd84f1b (must be forced) - image is being used by stopped container 1284da16efeb
```

## Docker容器

### 创建一个容器

```shell
[root@maxiaoyu ~]# docker run --name myfirstdocker -i -t centos /bin/bash
[root@2ce82d7a275e /]# uname -a
Linux 2ce82d7a275e 3.10.0-514.26.2.el7.x86_64 #1 SMP Tue Jul 4 15:04:05 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
[root@2ce82d7a275e /]# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.1  11764  1888 ?        Ss   09:40   0:00 /bin/bash
root        14  0.0  0.0  47436  1676 ?        R+   09:44   0:00 ps aux
```

这样就新建了一个容器，我们也随之会进入到容器的界面。docker容器的启动，即使没有把镜像pull下来，在执行如上的命令的时候依然可以执行，因为docker发现你没有这个镜像的时候会帮你把镜像pull下来。容器的主机名就是容器的container id。接下来看看如上的几个参数都代表什么意思：

- --name:指定这个生成的容器的名字，当然如果不指定的话name也会自动生成，我们指定名字只不过是为了更方便的去管理我们的容器。
- -i：允许标准输入 ，即确保容器的STDIN是开启的。可以看到，运行命令以后我们进入到了容器中，进程为PID的是/bin/bash，也就是刚才我们指定要运行的命令。因此docker其实是为进程执行隔离作用的，虚拟机是给操作系统做隔离的。
- -t：开启一个tty伪终端。
- -d：如果需要容器在后台运行的话，可以加上-d参数。范围值为容器的id

>以上操作其实是经历了这样一个过程：
>
>1. 执行上面操作首先会在系统本地去找有没有对应的这样一个image，如果没有的话那么就会去Docker Hub Registry去找，一旦找到以后就回去下载然后保存到本地宿主机器。
>2. Docker利用这个image创建了一个容器，这个容器拥有自己的网络，ip，以及桥接外部网络的接口。
>3. 我们告诉这个容器要去执行什么命令（/bin/bash），当然这个命令是可以在容器中指定的，指定容器起来的时候自动执行某个命令，那这里就可以不写了。

当前我们是在容器的内部，通过`ps aux`我们可以得知，pid的大树根并不是init（centos7的树根并不是init），而是我们运行的/bin/bash，如果此时我们使用exit退出容器的话，那么此时的容器就会被关掉，它就完成了它的使命。

```shell
[root@2ce82d7a275e /]# exit
exit
[root@maxiaoyu ~]# docker ps -a
CONTAINER ID  IMAGE    COMMAND      CREATED         STATUS                   PORTS     NAMES     
2ce82d7a275e  centos  "/bin/bash"  10 minutes ago  Exited (0) 5 seconds ago       myfirstdocker
```

Docker其实是单进程的，他需要一个进程在前台跑着，因此如果这个程序执行完了，那么整个容器也就退出了，就像我们刚才执行/bin/bash后进入到容器，当exit退出的时候这个进程结束以后整个容器也就跟着一起退出了。所以说不是所有的应用都适合docker。

### 启动并进入一个容器

那么如何启动已经关闭的容器呢？

**方法一**

```
docker start docker_name

比如（这样就一直跑起来了）：
[root@maxiaoyu ~]# docker start myfirstdocker
myfirstdocker
```

**方法二**

```
[root@maxiaoyu ~]# docker attach myfirstdocker
[root@2ce82d7a275e /]# 
# 这样就是直接进到容器里面去了，不过exit以后容器还是会停止，因此一般不会去这么操作的
# 而且多人同时进入到这个容器这个命令显示是同步的。
```

**方法三**

```shell
# 生产中建议使用的方法
[root@maxiaoyu ~]# yum -y install util-linux
[root@maxiaoyu ~]# docker start mydocker
mydocker
# 获取当前的docker的pid，如果获取到的pid=0，说明你这个容器没起来，后面可以接容器id也可以接容器名称。
[root@maxiaoyu ~]# docker inspect -f "{{ .State.Pid }}" mydocker
13500
[root@maxiaoyu ~]# nsenter -t 13500 -m -u -i -n -p 
# 使用这种方法进入容器，即使退出的话仍然可以保证容器是开启的，进程并不会结束，docker ps能看到
[root@1284da16efeb /]# exit
logout
# 那么为什么即使exit出去也不会退出容器呢？
[root@1284da16efeb /]# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0  11764  1676 ?        Ss+  10:01   0:00 /bin/bash
root        26  0.0  0.1  15192  1996 ?        S    10:05   0:00 -bash
root        39  0.0  0.0  50868  1816 ?        R+   10:05   0:00 ps aux
# 主要原因是因为多了一个-bash的进程，因此退出当前的进程这个docker不会退出，因为仍然还有一个/bin/bash的进程正在运行，因此整个容器是并不会退出的。
```

查看一下nsenter的帮助信息：

```
[root@maxiaoyu ~]# nsenter --help

Usage:
 nsenter [options] <program> [<argument>...]

Run a program with namespaces of other processes.

Options:
 -t, --target <pid>     target process to get namespaces from
 -m, --mount[=<file>]   enter mount namespace
 -u, --uts[=<file>]     enter UTS namespace (hostname etc)
 -i, --ipc[=<file>]     enter System V IPC namespace
 -n, --net[=<file>]     enter network namespace
 -p, --pid[=<file>]     enter pid namespace
 -U, --user[=<file>]    enter user namespace
 -S, --setuid <uid>     set uid in entered namespace
 -G, --setgid <gid>     set gid in entered namespace
     --preserve-credentials do not touch uids or gids
 -r, --root[=<dir>]     set the root directory
 -w, --wd[=<dir>]       set the working directory
 -F, --no-fork          do not fork before exec'ing <program>
 -Z, --follow-context   set SELinux context according to --target PID

 -h, --help     display this help and exit
 -V, --version  output version information and exit

For more details see nsenter(1).
```

因此我们可以针对第三种生产中使用的方法去写一个小脚本然后通过批量部署分发后使用：

```shell
# $1可以是容器的名称，或者容器的id
[root@maxiaoyu ~]# cat docker_in.sh 
#!/bin/bash

# Use nsenter to access docker 

docker_in (){
   NAME_ID=$1
   PID=$(docker inspect -f "{{ .State.Pid }}" $NAME_ID)
   nsenter -t $PID -m -u -i -n -p
}

docker_in $1
[root@maxiaoyu ~]# chmod +x docker_in.sh
```

**方法四**

```shell
[root@localhost ~]# docker exec mydocker uptime
 06:38:47 up  3:51,  0 users,  load average: 0.00, 0.03, 0.05
```

我不想真的进入容器，但是我还想让这个容器给我执行个命令，就可以使用exec，这个是exec命令的用途本意。不过通过exec也能进入容器，比如：

```shell
[root@linux-node1 ~]# docker exec -it mydocker /bin/bash
[root@e95a62d6770f /]# ps aux
USER        PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root          1  0.0  0.0  11768  1676 ?        Ss+  02:28   0:00 /bin/bash
root         50  0.5  0.0  11768  1884 ?        Ss   02:41   0:00 /bin/bash
root         62  0.0  0.0  47440  1676 ?        R+   02:41   0:00 ps aux
###有两个/bin/bash，因此这个容器技术退出了仍然还在运行，他退出的是pid=50的/bin/bash

[root@e95a62d6770f /]# exit
exit
[root@linux-node1 ~]# docker ps -a
CONTAINER ID    IMAGE    COMMAND     CREATED         STATUS        PORTS         NAMES
e95a62d6770f    centos  "/bin/bash" 33 minutes ago  Up 12 minutes              mydocker
78fc36ba1e5a    centos  "/bin/echo xx" 39 minutes ago  Exited (0) 39 minutes ago                       compassionate_rosalind
```

### 删除容器

```shell
docker rm 容器id
```

如果容器在使用的话那是不允许你删除的，不过你可以使用-f强制干掉。当然一般不建议这么做。一般来讲都是先把容器停掉以后再把容器干掉。

运行玩意后直接删除容器

```shell
## --rm参数可以使得一个容器运行完成以后就直接删除了。下面的内容执行完成以后再docker ps就看不到了
[root@linux-node1 ~]# docker run --rm centos /bin/echo "hello lamber"
hello lamber
```

### 停止容器

```shell
docker stop 容器ID
```



