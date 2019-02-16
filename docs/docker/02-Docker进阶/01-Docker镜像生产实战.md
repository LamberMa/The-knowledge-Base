# Docker镜像生产规划与实战

> 公开的dockerfile：https://github.com/dockerfile

## 分层的设计

![](http://omk1n04i8.bkt.clouddn.com/18-5-15/1214172.jpg)

```
A(系统层)-->B(运行环境层)-->C(应用服务层)
```



```
[root@linux-node1 ~]# mkdir docker
[root@linux-node1 ~]# cd docker/
[root@linux-node1 docker]# mkdir system
[root@linux-node1 docker]# mkdir runtime
[root@linux-node1 docker]# mkdir app
[root@linux-node1 docker]# cd system/
[root@linux-node1 system]# mkdir centos
[root@linux-node1 system]# mkdir ubuntu
[root@linux-node1 system]# mkdir centos-ssh
[root@linux-node1 system]# cd ../runtime/
[root@linux-node1 runtime]# mkdir php
[root@linux-node1 runtime]# mkdir java
[root@linux-node1 runtime]# mkdir python
[root@linux-node1 runtime]# cd ../app/
[root@linux-node1 app]# mkdir xxx.api
[root@linux-node1 app]# mkdir xxx.admin
[root@linux-node1 docker]# tree .
.
├── app
│   ├── xxx.admin
│   └── xxx.api
├── runtime
│   ├── java
│   ├── php
│   └── python
└── system
    ├── centos
    ├── centos-ssh
    └── ubuntu

```
## 模拟案例
### 规划：
requirement.txt

pip install -r requirement.txt

创建dockerfile然后build镜像

### 构建一个基础的centos镜像
```
[root@linux-node1 docker]# cd system/centos
[root@linux-node1 centos]# pwd
/root/docker/system/centos
[root@linux-node1 centos]# wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
--2017-03-31 18:13:42--  http://mirrors.aliyun.com/repo/epel-7.repo
正在解析主机 mirrors.aliyun.com (mirrors.aliyun.com)... 115.28.122.210, 112.124.140.210
正在连接 mirrors.aliyun.com (mirrors.aliyun.com)|115.28.122.210|:80... 已连接。
已发出 HTTP 请求，正在等待回应... 200 OK
长度：1084 (1.1K) [application/octet-stream]
正在保存至: “/etc/yum.repos.d/epel.repo”

100%[====================================================================================>] 1,084       --.-K/s 用时 0s      

2017-03-31 18:13:42 (16.7 MB/s) - 已保存 “/etc/yum.repos.d/epel.repo” [1084/1084])

[root@linux-node1 centos]# cp /etc/yum.repos.d/epel.repo .

```


```
[root@linux-node1 centos]# vim Dockerfile
# Docker for CentOS

FROM centos

MAINTAINER lamber 1020561033@qq.com

# EPEL
ADD epel.repo /etc/yum.repos.d/

# Base pkg
RUN yum install -y wget mysql-devel supervisor git redis tree net-tools sudo psmisc && yum clean all

[root@linux-node1 centos]# docker build -t lamber/centos:base .
[root@linux-node1 centos]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
lamber/centos       base                6a427d784875        6 seconds ago       283.1 MB
mynginx             v2                  5104f2ed9887        42 minutes ago      280.7 MB
nginx/mynginx       v1                  81f1607bb8a0        About an hour ago   355 MB
docker.io/nginx     latest              5e69fe4b3c31        3 days ago          182.5 MB
docker.io/centos    latest              98d35105a391        2 weeks ago         192.5 MB

```
再创建运行环境的时候就应该from上面我们创建的镜像了：

```
[root@linux-node1 python]# cat Dockerfile 
FROM lamber/centos:base

MAINTAINER lamber 1020561033@qq.com

# Python env
RUN yum install -y python-devel python-pip supervisor

# Upgrade pip
RUN pip install --upgrade pip


```
这里引用的就是上面我们创建好的镜像。

构建docker镜像

```
[root@linux-node1 python]# docker build -t lamber/python .
[root@linux-node1 app]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
lamber/python       latest              94747e30d7b4        2 minutes ago       413.3 MB
lamber/centos       base                6a427d784875        About an hour ago   283.1 MB
mynginx             v2                  5104f2ed9887        2 hours ago         280.7 MB
nginx/mynginx       v1                  81f1607bb8a0        2 hours ago         355 MB
docker.io/nginx     latest              5e69fe4b3c31        3 days ago          182.5 MB
docker.io/centos    latest              98d35105a391        2 weeks ago         192.5 MB


```
## 构建一个带ssh的centos docker镜像

```
[root@linux-node1 centos-ssh]# pwd
/root/docker/system/centos-ssh
[root@linux-node1 centos-ssh]# cat Dockerfile 
# Docker for CentOS

FROM centos

MAINTAINER lamber 1020561033@qq.com

# EPEL
ADD epel.repo /etc/yum.repos.d/

# Base pkg
RUN yum install -y openssh-clients openssl-devel openssh-server wget mysql-devel supervisor git redis tree net-tools sudo psmisc && yum clean all

# For SSHD
RUN ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key
RUN ssh-keygen -t ecdsa -f /etc/ssh/ssh_host_ecdsa_key
RUN echo "root:redhat" | chpasswd
[root@linux-node1 centos-ssh]# docker build -t lamber/centos-ssh .
[root@linux-node1 centos-ssh]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
lamber/centos-ssh   latest              41db0593e915        2 minutes ago       284.1 MB
lamber/python       latest              94747e30d7b4        16 minutes ago      413.3 MB
lamber/centos       base                6a427d784875        About an hour ago   283.1 MB
mynginx             v2                  5104f2ed9887        2 hours ago         280.7 MB
nginx/mynginx       v1                  81f1607bb8a0        3 hours ago         355 MB
docker.io/nginx     latest              5e69fe4b3c31        3 days ago          182.5 MB
docker.io/centos    latest              98d35105a391        2 weeks ago         192.5 MB

```
部署环境层：

```
[root@linux-node1 runtime]# cp -r python/ python-ssh
[root@linux-node1 runtime]# ll
总用量 0
drwxr-xr-x 2 root root  6 3月  31 18:08 java
drwxr-xr-x 2 root root  6 3月  31 18:08 php
drwxr-xr-x 2 root root 24 3月  31 19:52 python
drwxr-xr-x 2 root root 24 3月  31 22:39 python-ssh
[root@linux-node1 runtime]# cd python
[root@linux-node1 python]# cat Dockerfile 
FROM lamber/centos-ssh

MAINTAINER lamber 1020561033@qq.com

# Python env
RUN yum install -y python-devel python-pip supervisor

# Upgrade pip
RUN pip install --upgrade pip
[root@linux-node1 python]# docker build -t lamber/python-ssh .
[root@linux-node1 python]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
lamber/python-ssh   latest              657b57f25bdd        24 seconds ago      414.3 MB
lamber/centos-ssh   latest              41db0593e915        2 hours ago         284.1 MB
lamber/python       latest              94747e30d7b4        2 hours ago         413.3 MB
lamber/centos       base                6a427d784875        4 hours ago         283.1 MB
mynginx             v2                  5104f2ed9887        5 hours ago         280.7 MB
nginx/mynginx       v1                  81f1607bb8a0        5 hours ago         355 MB
docker.io/nginx     latest              5e69fe4b3c31        3 days ago          182.5 MB
docker.io/centos    latest              98d35105a391        2 weeks ago         192.5 MB

```
部署应用层：

测试脚本：

```
[root@linux-node1 shop-api]# pwd
/root/docker/app/shop-api

[root@linux-node1 shop-api]# cat app.py 
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return 'Hello World!' 

if __name__ == "__main__":
    app.run(host="0.0.0.0", debug=True)

```
测试脚本在本机进行测试（要首先确保本地跑没问题，然后再封装进docker镜像里面去）：

```
[root@linux-node1 shop-api]# yum -y install python-pip
[root@linux-node1 python-ssh]# pip install flask
[root@linux-node1 shop-api]# python app.py 
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
 * Restarting with stat
 * Debugger is active!
 * Debugger PIN: 248-731-527

起到了5000端口，访问测试
[root@linux-node1 shop-api]# curl  127.0.0.1:5000 
Hello World!
```
创建Dockerfile：

```
[root@linux-node1 shop-api]# cat Dockerfile 
#Base image
FROM lamber/python-ssh

#Maintainer
MAINTAINER lamber 1020561033@qq.com

# Add www user
RUN useradd -s /sbin/nologin -M www

# ADD file
ADD app.py /opt/app.py
ADD requirements.txt /opt/
ADD supervisord.conf /etc/supervisord.conf
ADD app-supervisor.ini /etc/supervisord.d/

# pip
RUN /usr/bin/pip2.7 install -r /opt/requirements.txt

# Port
EXPOSE 22 5000

#CMD
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]
##有参数是上面这种写法
```

创建supervisor的配置文件：

```
[root@linux-node1 shop-api]# cat app-supervisor.ini 
[program:shop-api]
command=/usr/bin/python2.7 /opt/app.py
process_name=%(program_name)s
autostart=true
user=www
stdout_logfile=/tmp/app.log
stderr_logfile=/tmp/app.error

[program:sshd]
command=/usr/sbin/sshd -D
process_name=%(program_name)s
autostart=true

```

创建依赖文件：

```
[root@linux-node1 shop-api]# cat requirements.txt 
flask

```
修改supervisor.conf的配置文件：

```
[root@linux-node1 shop-api]# cat supervisord.conf | grep nodaemon
nodaemon=true              ; (start in foreground if true;default false)
将nodaemon改为true，这样supervisor就在前台运行了，如此创建容器以后就能起来了。
```

补全文件：

```
[root@linux-node1 shop-api]# ll
总用量 24
-rw-r--r-- 1 root root  172 9月  10 2016 app.py
-rw-r--r-- 1 root root  257 9月  10 2016 app-supervisor.ini
-rw-r--r-- 1 root root  433 9月  10 2016 Dockerfile
-rw-r--r-- 1 root root    6 9月  10 2016 requirements.txt
-rw-r--r-- 1 root root 7952 9月  10 2016 supervisord.conf
```
生成docker镜像：

```
[root@linux-node1 shop-api]# docker build -t lamber/shop-api .
[root@linux-node1 shop-api]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
lamber/shop-api     latest              9a9470aa92eb        32 seconds ago      420.3 MB
lamber/python-ssh   latest              f1c4bcf54bad        19 minutes ago      414.3 MB
lamber/python       latest              570e81c8e8bf        23 minutes ago      413.3 MB
lamber/centos-ssh   latest              41db0593e915        3 hours ago         284.1 MB
lamber/centos       base                6a427d784875        4 hours ago         283.1 MB
mynginx             v2                  5104f2ed9887        5 hours ago         280.7 MB
nginx/mynginx       v1                  81f1607bb8a0        6 hours ago         355 MB
docker.io/nginx     latest              5e69fe4b3c31        3 days ago          182.5 MB
docker.io/centos    latest              98d35105a391        2 weeks ago         192.5 MB

```
启动镜像容器：

```
[root@linux-node1 shop-api]# docker run --name shop-api -d -p 88:5000 -p 8022:22 lamber/shop-api
f38807176f44970632f0156bd02a614c102343749a627a1eb945a6e745c2e95b
[root@linux-node1 shop-api]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                        NAMES
f38807176f44        lamber/shop-api     "/usr/bin/supervisord"   7 seconds ago       Up 6 seconds        0.0.0.0:8022->22/tcp, 0.0.0.0:88->5000/tcp   shop-api
a1efc632ba6d        mynginx:v2          "nginx"                  5 hours ago         Up 5 hours          0.0.0.0:82->80/tcp                           mynginxv2
604ac1c6443a        nginx/mynginx:v1    "nginx"                  6 hours ago         Up 6 hours          0.0.0.0:81->80/tcp                           mynginxv1

```
测试结果：

```
[root@linux-node1 shop-api]# curl 127.0.0.1:88
Hello World!
[root@linux-node1 shop-api]# /root/docker_in.sh shop-api
[root@f38807176f44 /]# ps aux
USER        PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root          1  0.0  0.5 117256 14796 ?        Ss   15:21   0:00 /usr/bin/python /usr/bin/supervisord -c /etc/supervisord.con
root          7  0.0  0.1  82524  3592 ?        S    15:21   0:00 /usr/sbin/sshd -D
www           8  0.0  0.6 119760 17328 ?        S    15:21   0:00 /usr/bin/python2.7 /opt/app.py
www          13  0.6  0.6 196036 18052 ?        Sl   15:21   0:02 /usr/bin/python2.7 /opt/app.py
root         19  0.0  0.0  15172  1896 ?        S    15:27   0:00 -bash
root         32  0.0  0.0  50844  1700 ?        R+   15:27   0:00 ps aux

```
访问容器的22端口：

密码就是我们在创建centos-ssh时候指定的root：redhat

![](http://omk1n04i8.bkt.clouddn.com/17-3-31/36142994-file_1490974494177_17425.jpg)
#### 总结
这就是分层的便利性，生产情况下都是使用supervisor来启动，即使是一个也要用supervisor来启动，python的依赖文件要写好。实际生产过程中，需要在真机上测试好了然后再进行镜像的封装。

```
[root@linux-node1 docker]# tree
.
├── app
│   ├── shop-api
│   │   ├── app.py
│   │   ├── app-supervisor.ini
│   │   ├── Dockerfile
│   │   ├── requirements.txt
│   │   └── supervisord.conf
│   ├── xxx.admin
│   └── xxx.api
├── runtime
│   ├── java
│   ├── php
│   ├── python
│   │   └── Dockerfile
│   └── python-ssh
│       └── Dockerfile
└── system
    ├── centos
    │   ├── Dockerfile
    │   └── epel.repo
    ├── centos-ssh
    │   ├── Dockerfile
    │   └── epel.repo
    └── ubuntu

```

### 使用supervisor管理进程
#### 参考文档

> http://supervisord.org/
>
> http://liyangliang.me/posts/2015/06/using-supervisor/



安装supervisor

```
[root@linux-node1 ~]# yum install supervisor -y
[root@linux-node1 ~]# rpm -ql supervisor | head -2
/etc/logrotate.d/supervisor
/etc/supervisord.conf
[root@linux-node1 ~]# tail -2 /etc/supervisord.conf 
[include]
files = supervisord.d/*.ini

```




```
[program:sshd]
command=/usr/sbin/sshd -D
process_name=%(program_name)s
autostart=true
```



```
# docker容器统一使用supervisor进行管理，是管理操作标准化起来。
[root@6dd90ccf92d0 ~]# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 18:01 ?        00:00:00 /usr/bin/python /usr/bin/supervisord -c /
root         7     1  0 18:01 ?        00:00:00 /usr/sbin/sshd -D
root         8     7  0 18:02 ?        00:00:00 sshd: root@pts/0
root        10     8  0 18:02 pts/0    00:00:00 -bash
root        23    10  0 18:03 pts/0    00:00:00 ps -ef
[root@6dd90ccf92d0 ~]# supervisorctl status
sshd                             RUNNING   pid 7, uptime 0:01:28
```

