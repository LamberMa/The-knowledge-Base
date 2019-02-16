# Docker-Registry

> DockerFile建议通过git来保存，



实验环境可以操作的，生产不建议这样去操作的内容：

- 容器停止后就自动删除：docker run —rm centos /bin/echo "hello world"
- 杀死所有正在运行的服务器：docker kill $(docker ps -a -q)
- 删除所有已经停止的容器：docker rm $(docker ps -a -q)
- 删除所有未打标签的镜像：docker rmi $(docker images -q -f dangling=true)





Docker registry功能比较单一，没有web界面。

Nginx + Docker registry 验证https（自签名证书）：

1. 手动创建证书

```shell
[root@localhost registry]# openssl req -x509 -days 3650 -nodes -newkey rsa:2048 -keyout ./docker-registry.key -out ./docker-registry.crt
Generating a 2048 bit RSA private key
...............................................................+++
...+++
writing new private key to './docker-registry.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:
State or Province Name (full name) []:
Locality Name (eg, city) [Default City]:
Organization Name (eg, company) [Default Company Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:reg.unixhot.com
Email Address []:
[root@localhost registry]# ll
总用量 8
-rw-r--r--. 1 root root 1289 5月  11 20:39 docker-registry.crt
-rw-r--r--. 1 root root 1708 5月  11 20:39 docker-registry.key
```

2. 安装httpd-tools htpasswd实现验证功能

```shell
# htpasswd是httpd-tools工具集中的工具，因此首先要安装httpd-tools
yum install -y httpd-tools
# 创建使用-c参数，加密码不要使用-c，指定用户为demo
[root@localhost registry]# htpasswd -c /opt/registry/docker-registry.htpasswd demo
New password: 
Re-type new password: 
Adding password for user demo
```

3. nginx proxy https  （有一个现成的镜像），启动一个docker-registry的容器，proxy到这里。

```shell
# --link可以让容器之间可以互访
docker run -d -p 443:443 \
--name docker-registry-proxy \
-e REGISTRY_HOST="docker-registry" \
-e REGISTRY_PORT="5000" \
-e SERVER_NAME="reg.unixhot.com" \
--link docker-registry:docker-registry \
-v /opt/registry/docker-registry.htpasswd:/etc/nginx/.htpasswd:ro \
-v /opt/registry:/etc/nginx/ssl:ro \
containersol/docker-registry-proxy
```

4. docker 配置使用证书

```shell
# 在/etc/hosts中针对reg.unixhot.com做域名的映射
192.168.56.101  reg.unixhot.com
# 配置docker使用证书，如果说不买证书的话那么所有的需要连接的服务器都要配置这一部分
# 将自签名证书放到/etc/docker下面
cd /etc/docker
mkdir -p certs.d/reg.unixhot.com
cd certs.d/reg.unixhot.com
cp /opt/registry/docker-registry.crt ca.crt
# 检测docker是否可以进行登录
[root@localhost ~]# docker login reg.unixhot.com
Username: demo
Password: 
Login Succeeded
```

5. 登录，push镜像

```shell
# Push镜像第一件需要做的事情就是打标签，这个标签是给docker入库我们创建这个registry的时候打的标签
[root@localhost ~]# docker tag unixhot/centos reg.unixhot.com/unixhot/centos
# push镜像到仓库
[root@localhost ~]# docker push reg.unixhot.com/unixhot/centos
The push refers to repository [reg.unixhot.com/unixhot/centos]
1e11f9cfa6aa: Pushed 
43e653f84b79: Pushed 
latest: digest: sha256:e0bf9b8009fdea481f4546d7139aa3beeabbe799d489843f1c6a61339ef11271 size: 3768
```

6. 查看，传上去了以后只能通过提供的api进行查看。

```shell
# 使用docker images也能查看到
[root@localhost ~]# curl -X GET https://demo:demo@reg.unixhot.com/v2/_catalog -k
{"repositories":["unixhot/centos"]}
```



如果是购买的证书的话只需要上述的第二步和第三步以及第五步即可。手动创建和配置docker使用证书就过了。



```shell
[root@localhost opt]# mkdir registry
[root@localhost opt]# cd registry/
[root@localhost registry]# docker run -d --name docker-registry -v /opt/registry/:/tmp/registry-dev registry:2.2.1
Unable to find image 'registry:2.2.1' locally
2.2.1: Pulling from library/registry
8387d9ff0016: Pull complete 
3b52deaaf0ed: Pull complete 
4bd501fad6de: Pull complete 
a3ed95caeb02: Pull complete 
b1f99b5938be: Pull complete 
82c34c0ec017: Pull complete 
5426c0c1c293: Pull complete 
Digest: sha256:30adb707d1b4d2ad694c38da3ea1e7d303fdbdce2538ab0372afe6f1523ae3c8
Status: Downloaded newer image for registry:2.2.1
4b19999b188ce8ed1a0ea7eab52c36a9a3e17ce78147f24a99d04d39624d9d87
```



## 使用Habor构建企业级Docker Registry

>Harbor是一个企业级Registry服务。它对开源的Docker Registry服务进行了扩展，添加了更多企业用户需要的功能。Harbor被设计用于部署一套组织内部使用的私有环境，这个私有Registry服务对于非常关心安全 的组织来说是十分重要的。另外，私有Registry服务可以通过避免从公域网下载镜像而提高企业生产力。这对于没有良好的Internet连接状态，使用Docker Container的用户是一个福音。
>
>Harbor是VMware公司最近开源的企业级Docker Registry项目(https://github.com/vmware/harbor) 。其目标是帮助用户迅速搭建一个企业级的Docker registry服务。它提供了管理UI, 基于角色的访问控制(Role Based Access Control)，AD/LDAP集成、以及审计日志(Audit logging) 等企业用户需求的功能，同时还原生支持中文。Harbor的每个组件都是以Docker容器的形式构建的，使用Docker Compose来对它进行部署。
>
>Harbor项目使用了go语言开发，WEB框架采用beego。容器应用的开发和运行离不开可靠的镜像管理。从安全和效率等方面考虑，在企业私有环境内部署的Registry服务是非常必要的。
>
>Harbor(https://github.com/vmware/harbor)由VMware中国研发团队为企业用户设计的Registry Server开源项目，包括了权限管理(RBAC)、图形管理界面、LDAP/AD集成、审计、自我注册、HA等企业必需的功能，同时针对中国用户的特点，原生支持中文，并计划实现镜像复制(roadmap)等功能。
>
>可参考内容：http://www.jiagoumi.com/work/1221.html

### 安装Docker-compose

```shell
# 安装过程中要用到docker-compose，这个玩意是用python写的，所以用pip来安装
yum -y install python-pip
pip install docker-compose
```

由于国内网速的问题这里我下载了harbor的离线包，离线包包含了所需的所有需要在线下载的镜像包，省去了很多麻烦的事情。[官方下载点](https://github.com/vmware/harbor/releases)&[百度云下载点](https://pan.baidu.com/s/1Ndp8Z_SHVD5AQ9J7ir7HYg)【提取码：5n52】

安装过程就异常简单了，我们可以照着[官方文档](https://github.com/vmware/harbor/blob/master/docs/installation_guide.md)直接去操作，官方文档详细的介绍了，我们在安装过程中需要调整的内容。这里其实就是简单的设置一下

### 修改harbor.cfg

这里我只简单的修改了3个配置，具体的配置在官方文档中，有非常详细的介绍，详细到介绍每一个配置项，大家可以根据自己的需要去仔细查看

```shell
# 修改harbor管理台UI的访问地址，不要用localhost或者127.0.0.1这种本地的地址，因为要对外提供服务
hostname = reg.unixhot.com
# 配置访问的方式为https，如果是自己在测试机上搞的话那么可以使用http，但是生产建议务必使用https
ui_url_protocol = https
# 如果有需要的话修改一下harbor的admin登录密码
harbor_admin_password = Harbor12345
```

初始化相关配置文件：

```shell
./prepare
```

运行install脚本进行安装部署harbor：

```shell
# 在线安装的过程太痛苦了，直接clone下来的包docker-compose.yml中不知道为啥，所有registry里面的镜像的tag都是__version__导致下载的时候直接找不到对应的version，不知道是不是只能手动改才行。
./install.sh
```

至此为止安装过程就结束了，离线版本包的安装是非常简单的，一步一步的操作即可，管理的话需要使用docker-compose来进行管理。

### 配置SSL

> 这里模拟使用https来进行访问，目前并没有购买证书，因此我们自己只能创建自签名证书来模拟这个环境。关于自签名证书的配置官方也是有教程的，直接按照教程去做就可以了。[跳转链接](https://github.com/vmware/harbor/blob/master/docs/configure_https.md)

具体的说明就不在这里赘述了，要看说明可以看官方文档，这里只是体现命令

```shell
openssl req -newkey rsa:4096 -nodes -sha256 -keyout ca.key -x509 -days 365 -out ca.crt
openssl req -newkey rsa:4096 -nodes -sha256 -keyout yourdomain.com.key -out reg.unixhot.com.csr
openssl x509 -req -days 365 -in reg.unixhot.com.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out reg.unixhot.com.crt
```

查看harbor.cfg配置文件中指定的ssl证书的位置：

```shell
#The path of cert and key files for nginx, they are applied only the protocol is set to https
ssl_cert = /data/cert/server.crt
ssl_cert_key = /data/cert/server.key
```

由于是默认配置，我也就没改，接下来就是把我们生成的key放到/data/cert/下并且按照配置文件中的重命名为server，注意这里是可以修改的，我图省事没有去动配置文件。

```shell
[root@localhost ~]# ll /data/cert/
total 8
-rw-r--r--. 1 root root 1866 May 12 13:22 server.crt
-rw-r--r--. 1 root root 3272 May 12 13:22 server.key
```

刷新配置文件：

```shell
./prepare
docker-compose down
# 启动harbor
docker-compose up -d
```

测试一下：

```shell
# 这里测试成功以后我们也就可以在网页端访问测试了。
[root@localhost harbor]# docker login reg.unixhot.com 
Username (admin): admin
Password: 
Login Succeeded
```

### Push镜像测试

将server.crt复制到/etc/docker/cert.d/reg.unixhot.com目录下，目录没有自己新建一个：

```shell
[root@localhost reg.unixhot.com]# pwd
/etc/docker/certs.d/reg.unixhot.com
[root@localhost reg.unixhot.com]# ll
total 4
-rw-r--r--. 1 root root 1866 May 12 14:25 server.crt
```

在web端Harbor中新建一个项目明为exercise

![](http://omk1n04i8.bkt.clouddn.com/18-5-22/97293193.jpg)

push镜像：

```shell
[root@localhost reg.unixhot.com]# docker login reg.unixhot.com -uadmin -pHarbor12345
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
Login Succeeded
# 首先记得打标签，docker tag src dst
# dockerhub域名/项目名/镜像名:TAG
[root@localhost reg.unixhot.com]# docker tag unixhot/centos reg.unixhot.com/exercise/centos
[root@localhost reg.unixhot.com]# docker push reg.unixhot.com/exercise/centos
The push refers to repository [reg.unixhot.com/exercise/centos]
1e11f9cfa6aa: Pushed 
43e653f84b79: Pushed 
latest: digest: sha256:792ead7dcec256fc00102cd0dd505c11b54a906ff2df2c090be972a92976ebc7 size: 741
```

在网页端查看，就能看到我们的镜像了。

![](http://omk1n04i8.bkt.clouddn.com/18-5-22/51015014.jpg)

从Harbor把镜像给Pull下来测试：

```shell
[root@localhost ~]# docker pull reg.unixhot.com/exercise/centos:latest
latest: Pulling from exercise/centos
469cfcc7a4b3: Already exists 
6863d4929975: Pull complete 
Digest: sha256:792ead7dcec256fc00102cd0dd505c11b54a906ff2df2c090be972a92976ebc7
Status: Downloaded newer image for reg.unixhot.com/exercise/centos:latest
```

测试成功。