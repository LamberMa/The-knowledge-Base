# Docker数据存储&网络访问

## Docker的网络应用

### 通过端口映射的方式访问Docker

我们安装了docker后会多了一个桥接网卡docker0：

```shell
[root@localhost ~]# ifconfig docker0
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        inet6 fe80::42:baff:fea1:dfa8  prefixlen 64  scopeid 0x20<link>
        ether 02:42:ba:a1:df:a8  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 10  bytes 1876 (1.8 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

[root@localhost ~]# yum install bridge-utils

[root@localhost ~]# brctl show
bridge name	bridge id		STP enabled	interfaces
docker0		8000.0242baa1dfa8	no		veth92ad03f
```

那么现在启动一个容器并映射端口，端口的映射分为随机映射和指定映射：

- 随机映射
  - docker run -P
- 指定映射
  - -p hostPort:containerPort       # 映射所有ip的指定端口到容器ip的指定端口
  - -p ip:hostPort:containerPort   # 如果物理机有多个ip地址，我们也可以指定一个ip
  - -p ip::containerPort   # 映射到指定地址的任意端口
  - -p hostPort:containerPort:udp # 默认是tcp的，可以指定协议为udp
  - -p 80:81 -p 443:443 ##同时映射多个端口

#### 随机端口映射

由于之前下载了一个nginx的镜像，那么我们就直接先来启用一个nginx容器：

```shell
[root@localhost ~]# docker run -d -P --name nginx-test1 nginx
WARNING: IPv4 forwarding is disabled. Networking will not work.
0b3299999a96eb216b05f2a5c68f32a2775751a49d240034a2f89305bb7bd236

# 可以看到容器启动，返回了容器的id的，但是报了一个警报说如果不打开ipv4转发的话那么网络将不会有效。那么我们就先把系统的网络转发打开
echo 1 > /proc/sys/net/ipv4/ip_forward

# 查看容器的状态
[root@localhost ~]# docker ps
CONTAINER ID IMAGE   COMMAND              CREATED             STATUS              PORTS                   NAMES
0b3299999a96 nginx "nginx -g 'daemon of…"   3 minutes ago       Up 3 minutes        0.0.0.0:32768->80/tcp   nginx-test1
```

从上面可以看到将nginx容器的80端口映射到了宿主机的32768端口。此时我们就可以通过访问宿主机的32768端口来访问docker了，具体我还可以通过iptables来查看，`iptables -t nat nvL`。可以通过docker logs查看日志：

```shell
[root@localhost ~]# docker logs nginx-test1
192.168.56.1 - - [11/May/2018:07:32:23 +0000] "GET / HTTP/1.1" 200 612 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.139 Safari/537.36" "-"
2018/05/11 07:32:24 [error] 5#5: *1 open() "/usr/share/nginx/html/favicon.ico" failed (2: No such file or directory), client: 192.168.56.1, server: localhost, request: "GET /favicon.ico HTTP/1.1", host: "192.168.56.101:32768", referrer: "http://192.168.56.101:32768/"
192.168.56.1 - - [11/May/2018:07:32:24 +0000] "GET /favicon.ico HTTP/1.1" 404 572 "http://192.168.56.101:32768/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.139 Safari/537.36" "-"
```

随机映射是非常简单的，但是管理起来也不是很方便的，因为我们需要知道每个容器的端口的映射关系，好处就是端口不会重复，启用很方便。

#### 指定映射

```shell
[root@localhost ~]# docker run -d -p 192.168.56.101:88:80 --name nginx-test2 nginx

f2eccbad48a94111e7844562ce8750584bced374894cd4dad0af7b201eef8ffc
[root@localhost ~]# curl 192.168.56.101:88 -I
HTTP/1.1 200 OK
Server: nginx/1.13.12
Date: Fri, 11 May 2018 07:45:34 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Mon, 09 Apr 2018 16:01:09 GMT
Connection: keep-alive
ETag: "5acb8e45-264"
Accept-Ranges: bytes
```

查看容器映射：

```shell
# -l参数指的是查看最后一个最新创建的容器
[root@localhost ~]# docker ps -l
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                       NAMES
f2eccbad48a9        nginx               "nginx -g 'daemon of…"   2 minutes ago       Up 2 minutes        192.168.56.101:88->80/tcp   nginx-test2
```

查看一下iptables的nat规则：

```shell
Chain DOCKER (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0           
    2   128 DNAT       tcp  --  !docker0 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:32768 to:172.17.0.3:80
    1    60 DNAT       tcp  --  !docker0 *       0.0.0.0/0            192.168.56.101       tcp dpt:88 to:172.17.0.4:80
```

经过NAT转换所以对于吞吐量比较大的业务还是多少有一些影响的。

## Docker数据存储

> 生产中不建议直接在docker容器中去写数据，建议使用数据卷或者数据卷容器写入

### 数据卷

> - Docker镜像由多个只读层叠加而成，当启动容器的时候，Docker会加载只读镜像层，并在镜像栈顶部添加一个读写层
> - 如果运行中的容器修改了现有的一个已经存在的文件，那么该文件将会从读写层下面的只读层复制到读写层，该文件的只读版本仍然存在，只是已经被读写层中该文件的副本所隐藏，此即”写时复制（COW）“机制，鉴于这种问题，层数那么多，在写的方面效率相对较低。
> - 数据卷使得即使容器脱离它的生命周期，数据仍然存在。

我们新建一个容器，并把物理机的一个区域给它mount到容器中的data目录下，使用-v参数：

```shell
[root@localhost ~]# docker run -d --name nginx-volume-test1 -v /data nginx
7cb33525f1027208402a86c95d0deedc4bf592264d113ddb47901d46797daa94
```

为了方便测试，我们直接去容器内的data目录下新建了一个hehe文件。那么这个文件被映射到物理机的什么地方呢？

```shell
[root@localhost ~]# docker inspect -f {{.Mounts}} nginx-volume-test1
[{volume c2316666b28d0875aa66ec90ef511e61c0e044e531d56a54e835789ac9792229 /var/lib/docker/volumes/c2316666b28d0875aa66ec90ef511e61c0e044e531d56a54e835789ac9792229/_data /data local  true }]
[root@localhost ~]# cd /var/lib/docker/volumes/c2316666b28d0875aa66ec90ef511e61c0e044e531d56a54e835789ac9792229/_data
[root@localhost _data]# ll
总用量 0
-rw-r--r--. 1 root root 0 5月  11 04:14 hehe
```

另外一种挂载方式，指定我们的映射目录，一个源一个目的：

```shell
# 源是物理机的，目的是docker容器的。注意对应关系
mkdir /data/docker-volume-nginx -p
docker run -d --name nginx-volume-test2 -v /data/docker-volume-nginx/:/data nginx
6b739fc4d20f1064abf5c2fb619a394772f445f15636b303b0527bb148bd2818

touch /data/docker-volume-nginx/hehehe
docker_in.sh nginx-volume-test2
# 进入容器
root@6b739fc4d20f:/# ls -l /data/
total 0
-rw-r--r-- 1 root root 0 Mar 31 08:20 hehehe
```

但是上面的这种方法在dockerfile中就是不支持的，试想如果这样操作了，那么可移植性就下降了，你要确保移植的位置也有你这个物理机的目录和数据。

其他的几个选项：

```shell
docker run -d --name nginx-volume-test2 -v /data/docker-volume-nginx/:/data:ro  只读
```

挂载文件：

```shell
[root@linux-node1 ~]# docker run --rm -it -v /root/docker_in.sh:/root/ nginx /bin/bash
root@239820f3e6bd:/# ls /root/ -l --color
total 4
-rwxr-xr-x 1 root root 179 Mar 31 07:48 docker_in.sh

###记得前后文件名要对应
```

### 数据卷容器

实现数据在多个容器之间共享

```shell
[root@linux-node1 ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                               NAMES
6b739fc4d20f        nginx               "nginx -g 'daemon off"   11 minutes ago      Up 11 minutes       80/tcp, 443/tcp                     nginx-volume-test2
925564d95872        nginx               "nginx -g 'daemon off"   20 minutes ago      Up 20 minutes       80/tcp, 443/tcp                     nginx-volumn-test1
dce2e78b22be        nginx               "nginx -g 'daemon off"   36 minutes ago      Up 36 minutes       443/tcp, 192.168.56.11:81->80/tcp   nginx2
[root@linux-node1 ~]# docker run -it --name volume-test3 --volumes-from nginx-volume-test2 centos /bin/bash
[root@034018de2dc4 /]# ls /data/
hehehe
```

即使把nginx-volume-test2给停掉了仍然不影响volume-test3的访问，因为这个数据卷是mount上去的。而且我们这个时候占用的这个共享卷的时候，这个nginx-volume-test2你是删除不掉的。

**实际应用**

比如创建一个容器挂载数据卷，然后其他容器都共享它的。比如可以用到日志统一管理，多个容器挂载一个日志数据卷，然后用logstash统一收集管理。

```shell
[root@linux-node1 ~]# docker run -d --name nfs-test -v /data/nfs-data/:/data nginx
[root@linux-node1 ~]# docker run --rm -it --volumes-from nfs-test centos /bin/bash
[root@ac637a64910f /]# cd /data/
[root@ac637a64910f data]# ls -l
total 0
-rw-r--r-- 1 root root 0 Mar 31 08:41 iamatest
此时我们在物理机上创建的测试文件就显示出来了。
```

