title: Day0：Docker容器入门基础
author: Stanley Wang
date: 2019-10-18 23:12:56
---
- - -
{% cq %}欢迎加入王导的VIP学习qq群：==>[<font color="FF7F50">932194668</font>](http://shang.qq.com/wpa/qunwpa?idkey=78869fddc5a661acb0639315eb52997c108de6625df5f0ee2f0372f176a032a6)<=={% endcq %}
- - -
# 实验环境
- 宿主机操作系统版本：
> CentOS Linux release 7.6.1810 (Core) 

- Docker版本：
> Docker-CE: v19.03.0

- 系统网络：

宿主机IP|Docker网络
-|-
10.4.7.5|172.7.5.0/24

# 安装部署
```
[root@docker ~]# yum install -y yum-utils
Installed:
  yum-utils.noarch 0:1.1.31-50.el7                                                                                                                                          
Dependency Installed:
  libxml2-python.x86_64 0:2.9.1-6.el7_2.3                    python-chardet.noarch 0:2.2.1-1.el7_1                    python-kitchen.noarch 0:1.1.1-5.el7                   

Complete!
[root@docker ~]# yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
Loaded plugins: fastestmirror
adding repo from: http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
grabbing file http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo to /etc/yum.repos.d/docker-ce.repo
repo saved to /etc/yum.repos.d/docker-ce.repo

[root@docker ~]# yum list docker-ce --showduplicates
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * epel: fedora.cs.nctu.edu.tw
docker-ce-stable                                                                                                                                     | 3.5 kB  00:00:00     
(1/2): docker-ce-stable/x86_64/updateinfo                                                                                                            |   55 B  00:00:00     
(2/2): docker-ce-stable/x86_64/primary_db                                                                                                            |  32 kB  00:00:00     
Available Packages
docker-ce.x86_64                                                          17.03.0.ce-1.el7.centos                                                           docker-ce-stable
docker-ce.x86_64                                                          17.03.1.ce-1.el7.centos                                                           docker-ce-stable
docker-ce.x86_64                                                          17.03.2.ce-1.el7.centos                                                           docker-ce-stable
docker-ce.x86_64                                                          17.03.3.ce-1.el7                                                                  docker-ce-stable
docker-ce.x86_64                                                          17.06.0.ce-1.el7.centos                                                           docker-ce-stable
docker-ce.x86_64                                                          17.06.1.ce-1.el7.centos                                                           docker-ce-stable
docker-ce.x86_64                                                          17.06.2.ce-1.el7.centos                                                           docker-ce-stable
docker-ce.x86_64                                                          17.09.0.ce-1.el7.centos                                                           docker-ce-stable
docker-ce.x86_64                                                          17.09.1.ce-1.el7.centos                                                           docker-ce-stable
docker-ce.x86_64                                                          17.12.0.ce-1.el7.centos                                                           docker-ce-stable
docker-ce.x86_64                                                          17.12.1.ce-1.el7.centos                                                           docker-ce-stable
docker-ce.x86_64                                                          18.03.0.ce-1.el7.centos                                                           docker-ce-stable
docker-ce.x86_64                                                          18.03.1.ce-1.el7.centos                                                           docker-ce-stable
docker-ce.x86_64                                                          18.06.0.ce-3.el7                                                                  docker-ce-stable
docker-ce.x86_64                                                          18.06.1.ce-3.el7                                                                  docker-ce-stable
docker-ce.x86_64                                                          18.06.2.ce-3.el7                                                                  docker-ce-stable
docker-ce.x86_64                                                          18.06.3.ce-3.el7                                                                  docker-ce-stable
docker-ce.x86_64                                                          3:18.09.0-3.el7                                                                   docker-ce-stable
docker-ce.x86_64                                                          3:18.09.1-3.el7                                                                   docker-ce-stable
docker-ce.x86_64                                                          3:18.09.2-3.el7                                                                   docker-ce-stable
docker-ce.x86_64                                                          3:18.09.3-3.el7                                                                   docker-ce-stable
docker-ce.x86_64                                                          3:18.09.4-3.el7                                                                   docker-ce-stable
docker-ce.x86_64                                                          3:18.09.5-3.el7                                                                   docker-ce-stable
docker-ce.x86_64                                                          3:18.09.6-3.el7                                                                   docker-ce-stable
docker-ce.x86_64                                                          3:18.09.7-3.el7                                                                   docker-ce-stable
docker-ce.x86_64                                                          3:18.09.8-3.el7                                                                   docker-ce-stable
docker-ce.x86_64                                                          3:19.03.0-3.el7                                                                   docker-ce-stable
docker-ce.x86_64                                                          3:19.03.1-3.el7                                                                   docker-ce-stable


```

## 配置
```vi /etc/docker/daemon.json
{
  "graph": "/data/docker",
  "storage-driver": "overlay2",
  "insecure-registries": ["registry.access.redhat.com","quay.io"],
  "registry-mirrors": ["https://q2gr04ke.mirror.aliyuncs.com"],
  "bip": "172.7.5.1/24",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "live-restore": true
}
```

## 启动Docker引擎
```
[root@docker ~]# systemctl start docker
[root@docker ~]# systemctl enable docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
[root@docker ~]# docker version
Client: Docker Engine - Community
 Version:           19.03.0
 API version:       1.40
 Go version:        go1.12.5
 Git commit:        aeac9490dc
 Built:             Wed Jul 17 18:15:40 2019
 OS/Arch:           linux/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          19.03.0
  API version:      1.40 (minimum version 1.12)
  Go version:       go1.12.5
  Git commit:       aeac9490dc
  Built:            Wed Jul 17 18:14:16 2019
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.2.6
  GitCommit:        894b81a4b802e4eb2a91d1ce216b8817763c29fb
 runc:
  Version:          1.0.0-rc8
  GitCommit:        425e105d5a03fabd737a126ad93d62a9eeede87f
 docker-init:
  Version:          0.18.0
  GitCommit:        fec3683
```

## 启动第一个Docker容器
```
[root@docker ~]# docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
1b930d010525: Already exists 
Digest: sha256:6540fc08ee6e6b7b63468dc3317e3303aae178cb8a45ed3123180328bcc1d20f
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

# Docker的镜像管理
## hub.docker.com
![dockerhub](/images/dockerhub.png "dockerhub")

## 注册一个dockerhub帐号
```
[root@docker ~]# docker login docker.io
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: xxxxxx
Password: xxxxxx
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

## 搜索一个镜像
- 命令行搜索

```
[root@docker ~]# docker search alpine
NAME                                   DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
alpine                                 A minimal Docker image based on Alpine Linux鈥?  5498                [OK]                
mhart/alpine-node                      Minimal Node.js built on Alpine Linux           435                                     
anapsix/alpine-java                    Oracle Java 8 (and 7) with GLIBC 2.28 over A鈥?  417                                     [OK]
frolvlad/alpine-glibc                  Alpine Docker image with glibc (~12MB)          209                                     [OK]
gliderlabs/alpine                      Image based on Alpine Linux will help you wi鈥?  180                                     
mvertes/alpine-mongo                   light MongoDB container                         105                                     [OK]
alpine/git                             A  simple git container running in alpine li鈥?  96                                      [OK]
yobasystems/alpine-mariadb             MariaDB running on Alpine Linux [docker] [am鈥?  44                                      [OK]
kiasaki/alpine-postgres                PostgreSQL docker image based on Alpine Linux   43                                      [OK]
davidcaste/alpine-tomcat               Apache Tomcat 7/8 using Oracle Java 7/8 with鈥?  36                                      [OK]
zzrot/alpine-caddy                     Caddy Server Docker Container running on Alp鈥?  35                                      [OK]
alpine/socat                           Run socat command in alpine container           33                                      [OK]
easypi/alpine-arm                      AlpineLinux for RaspberryPi                     32                                      
jfloff/alpine-python                   A small, more complete, Python Docker image 鈥?  26                                      [OK]
byrnedo/alpine-curl                    Alpine linux with curl installed and set as 鈥?  26                                      [OK]
hermsi/alpine-sshd                     Dockerize your OpenSSH-server with rsync and鈥?  23                                      [OK]
etopian/alpine-php-wordpress           Alpine WordPress Nginx PHP-FPM WP-CLI           21                                      [OK]
hermsi/alpine-fpm-php                  Dockerize your FPM PHP 7.4 upon a lightweigh鈥?  17                                      [OK]
zenika/alpine-chrome                   Chrome running in headless mode in a tiny Al鈥?  13                                      [OK]
davidcaste/alpine-java-unlimited-jce   Oracle Java 8 (and 7) with GLIBC 2.21 over A鈥?  13                                      [OK]
bashell/alpine-bash                    Alpine Linux with /bin/bash as a default she鈥?  13                                      [OK]
spotify/alpine                         Alpine image with `bash` and `curl`.            9                                       [OK]
tenstartups/alpine                     Alpine linux base docker image with useful p鈥?  8                                       [OK]
rawmind/alpine-traefik                 This image is the traefik base. It comes fro鈥?  5                                       [OK]
hermsi/alpine-varnish                  Dockerize Varnish upon a lightweight alpine-鈥?  1                                       [OK]
```

- 在网站搜索
![dockersearch](/images/dockersearch.png "dockersearch")

## 下载一个镜像
- 直接下载

```
[root@docker ~]# docker pull alpine
Using default tag: latest
latest: Pulling from library/alpine
050382585609: Pull complete 
Digest: sha256:6a92cd1fcdc8d8cdec60f33dda4db2cb1fcdcacf3410a8e05b3741f44a9b5998
Status: Downloaded newer image for alpine:latest
docker.io/library/alpine:latest
```

- 下载指定tag

```
[root@docker ~]# docker pull alpine:3.10.1
3.10.1: Pulling from library/alpine
Digest: sha256:6a92cd1fcdc8d8cdec60f33dda4db2cb1fcdcacf3410a8e05b3741f44a9b5998
Status: Downloaded newer image for alpine:3.10.1
docker.io/library/alpine:3.10.1
```

> 镜像的结构：registry_name/repository_name/image_name:tag_name
> 例如：docker.io/library/alpine:3.10.1

## 查看本地镜像
```
[root@docker ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
alpine              3.10.1              b7b28af77ffe        2 weeks ago         5.58MB
alpine              latest              b7b28af77ffe        2 weeks ago         5.58MB
hello-world         latest              fce289e99eb9        6 months ago        1.84kB
```

## 给镜像打标签
```
[root@docker ~]# docker tag b7b28af77ffe docker.io/stanleyws/alpine:v3.10.1
[root@docker ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
alpine              3.10.1              b7b28af77ffe        2 weeks ago         5.58MB
alpine              latest              b7b28af77ffe        2 weeks ago         5.58MB
stanleyws/alpine    v3.10.1             b7b28af77ffe        2 weeks ago         5.58MB
hello-world         latest              fce289e99eb9        6 months ago        1.84kB
```

## 推送镜像
```
[root@docker ~]# docker push stanleyws/alpine:v3.10.1
The push refers to repository [docker.io/stanleyws/alpine]
1bfeebd65323: Mounted from library/alpine
v3.10.1: digest: sha256:57334c50959f26ce1ee025d08f136c2292c128f84e7b229d1b0da5dac89e9866 size: 528
```
![dockerpush](/images/dockerpush.png "dockerpush")

## 给镜像再打一个标签
```
[root@docker ~]# docker images|grep alpine
stanleyws/alpine    v3.10.1             b7b28af77ffe        2 weeks ago         5.58MB
alpine              3.10.1              b7b28af77ffe        2 weeks ago         5.58MB
alpine              latest              b7b28af77ffe        2 weeks ago         5.58MB
[root@docker ~]# docker tag b7b28af77ffe stanleyws/alpine:latest
[root@docker ~]# docker images|grep alpine
stanleyws/alpine    latest              b7b28af77ffe        2 weeks ago         5.58MB
stanleyws/alpine    v3.10.1             b7b28af77ffe        2 weeks ago         5.58MB
alpine              3.10.1              b7b28af77ffe        2 weeks ago         5.58MB
alpine              latest              b7b28af77ffe        2 weeks ago         5.58MB
```

## 再次推送镜像
```
[root@docker ~]# docker push stanleyws/alpine:latest
The push refers to repository [docker.io/stanleyws/alpine]
1bfeebd65323: Layer already exists 
latest: digest: sha256:57334c50959f26ce1ee025d08f136c2292c128f84e7b229d1b0da5dac89e9866 size: 528
```
![dockerpushagain](/images/dockerpushagain.png "dockerpushagain")

## 删除镜像
- 删除镜像标签

```
[root@docker ~]# docker rmi stanleyws/alpine:latest
Untagged: stanleyws/alpine:latest
[root@docker ~]# docker images|grep alpine
alpine              3.10.1              b7b28af77ffe        2 weeks ago         5.58MB
alpine              latest              b7b28af77ffe        2 weeks ago         5.58MB
stanleyws/alpine    v3.10.1             b7b28af77ffe        2 weeks ago         5.58MB
```

- 删除镜像

```
[root@docker ~]# docker rmi -f b7b28af77ffe
Untagged: alpine:3.10.1
Untagged: alpine:latest
Untagged: alpine@sha256:6a92cd1fcdc8d8cdec60f33dda4db2cb1fcdcacf3410a8e05b3741f44a9b5998
Untagged: stanleyws/alpine:v3.10.1
Untagged: stanleyws/alpine@sha256:57334c50959f26ce1ee025d08f136c2292c128f84e7b229d1b0da5dac89e9866
Deleted: sha256:b7b28af77ffec6054d13378df4fdf02725830086c7444d9c278af25312aa39b9
Deleted: sha256:1bfeebd65323b8ddf5bd6a51cc7097b72788bc982e9ab3280d53d3c613adffa7
[root@docker ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
hello-world         latest              fce289e99eb9        6 months ago        1.84kB
```

![dockerrmi](/images/dockerrmi.png "dockerrmi")

# Docker容器的基本操作
## 查看本地的容器进程
```
[root@docker ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                   PORTS               NAMES
83f3560e01e9        hello-world         "/hello"            2 hours ago         Exited (0) 2 hours ago                       zen_dhawan
```

## 启动容器（运行镜像）
> docker run是日常用的最频繁用的命令之一，同样也是较为复杂的命令之一
> 命令格式：docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
> OPTIONS：选项
> -i：表示启动一个可交互的容器，并持续打开标准输入
> -t：表示使用终端关联到容器的标准输入输出上
> -d：表示将容器放置后台运行
> --rm: 退出后即删除容器
> --name：表示定义容器唯一名称
> IMAGE：表示要运行的镜像
> COMMAND：表示启动容器时要运行的命令*

### 交互式启动一个容器
```
[root@docker ~]# docker run -ti stanleyws/alpine:v3.10.1 /bin/sh
Unable to find image 'stanleyws/alpine:v3.10.1' locally
v3.10.1: Pulling from stanleyws/alpine
050382585609: Pull complete 
Digest: sha256:57334c50959f26ce1ee025d08f136c2292c128f84e7b229d1b0da5dac89e9866
Status: Downloaded newer image for stanleyws/alpine:v3.10.1
/ # cat /etc/issue 
Welcome to Alpine Linux 3.10
Kernel \r on an \m (\l)
```
### 非交互式启动一个容器
```
[root@docker ~]# docker run --rm stanleyws/alpine:v3.10.1 /bin/echo hello
hello
```

### 非交互式启动一个后台容器
```
[root@docker ~]# docker run -d --name myalpine stanleyws/alpine:v3.10.1 /bin/sleep 300
b42487bc0ead6e9f21db1719ea12658949f9f0c73b74441f7809e1841da9f8c1
```

## 查看容器
### 查看运行中的容器
```
[root@docker ~]# docker ps 
CONTAINER ID        IMAGE                      COMMAND             CREATED             STATUS              PORTS               NAMES
b42487bc0ead        stanleyws/alpine:v3.10.1   "/bin/sleep 300"    3 seconds ago       Up 3 seconds                            myalpine
```

### 查看所有容器
```
[root@docker ~]# docker ps -a
CONTAINER ID        IMAGE                      COMMAND             CREATED              STATUS                   PORTS               NAMES
b42487bc0ead        stanleyws/alpine:v3.10.1   "/bin/sleep 300"    About a minute ago   Up About a minute                          myalpine
83f3560e01e9        hello-world                "/hello"            5 mins ago          Exited (0) 5 mins ago                       zen_dhawan
```

### 查看宿主机进程
```
[root@docker ~]# ps aux|grep sleep|grep -v grep
root      49302  0.0  0.0   1540   244 ?        Ss   15:32   0:00 sleep 300
```

## 进入容器
```
[root@docker ~]# docker exec -ti myalpine /bin/sh
/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sleep 300
    6 root      0:00 /bin/sh
   11 root      0:00 ps aux
```
> docker attach不推荐使用

## 停止/启动/重启容器
### 停止容器
```
/ # exit
[root@docker ~]# docker stop myalpine
myalpine
[root@docker ~]# docker ps -a
CONTAINER ID        IMAGE                      COMMAND             CREATED             STATUS                       PORTS               NAMES
2dc026a2ae2d        stanleyws/alpine:v3.10.1   "/bin/sleep 3000"   21 seconds ago      Exited (137) 4 seconds ago                       myalpine
83f3560e01e9        hello-world                "/hello"            5 hours ago         Exited (0) 5 mins ago                            zen_dhawan
```

### 启动容器
```
[root@docker ~]# docker start myalpine
myalpine
[root@docker ~]# docker ps -a
CONTAINER ID        IMAGE                      COMMAND             CREATED             STATUS                   PORTS               NAMES
3614b532e596        stanleyws/alpine:v3.10.1   "/bin/sleep 3000"   28 seconds ago      Up 2 seconds                                 myalpine
```

### 重启容器
```
[root@docker ~]# docker restart myalpine
myalpine
[root@docker ~]# docker ps -a|grep alpine
3614b532e596        stanleyws/alpine:v3.10.1   "/bin/sleep 3000"   About a minute ago   Up 8 seconds                                 myalpine
```

## 删除容器
```
[root@docker ~]# docker rm myalpine
Error response from daemon: You cannot remove a running container 3614b532e596d2e59a054f1787929f62145649b98744e0efb23cac5beb0ac1f9. Stop the container before attempting removal or force remove
[root@docker ~]# docker rm -f myalpine
myalpine
```
> 如果没有停止的容器，需要用docker rm -f 进行强制删除

## 修改/提交容器
### 修改容器
```
[root@docker ~]# docker exec -ti 91abfbefd58 /bin/sh
/ # pwd
/
/ # echo Hello > 1.txt
/ # ls
1.txt  bin    dev    etc    home   lib    media  mnt    opt    proc   root   run    sbin   srv    sys    tmp    usr    var
/ # exit
```

### 提交容器
```
[root@docker ~]# docker commit -p 91abfbefd58 stanleyws/alpine:v3.10.1_with_1.txt
sha256:0890597dc4c9ce1be2e50d3121648cb3612500360c1d6c69545d1dcf5e851eee
```

### 验证
```
[root@docker ~]# docker run --rm stanleyws/alpine:v3.10.1_with_1.txt /bin/cat /1.txt
Hello
```
> 也可以使用docker export 91abfbefd58 > myalpine_with_1.txt.tar

## 导出/导入镜像
### 导出镜像
```
[root@docker ~]# docker images
REPOSITORY          TAG                  IMAGE ID            CREATED             SIZE
stanleyws/alpine    v3.10.1_with_1.txt   0890597dc4c9        2 minutes ago       5.58MB
stanleyws/alpine    v3.10.1              b7b28af77ffe        2 weeks ago         5.58MB
hello-world         latest               fce289e99eb9        6 months ago        1.84kB
[root@docker ~]# docker save 0890597dc4c9 > alpine_with_1.txt.tar
[root@docker ~]# ls
alpine_with_1.txt.tar  anaconda-ks.cfg
```

### 导入镜像
```
[root@docker ~]# docker rmi -f 0890597dc4c9
Untagged: stanleyws/alpine:v3.10.1_with_1.txt
Deleted: sha256:0890597dc4c9ce1be2e50d3121648cb3612500360c1d6c69545d1dcf5e851eee
Deleted: sha256:14b6e969e7d01c4d83f2a5a29b31717ec2c0805709f568ae7f8c7624208cc744
[root@docker ~]# docker load < ./alpine_with_1.txt.tar 
eb02c7007327: Loading layer [==================================================>]  3.584kB/3.584kB
Loaded image ID: sha256:0890597dc4c9ce1be2e50d3121648cb3612500360c1d6c69545d1dcf5e851eee
[root@docker ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
<none>              <none>              0890597dc4c9        3 minutes ago       5.58MB
stanleyws/alpine    v3.10.1             b7b28af77ffe        2 weeks ago         5.58MB
hello-world         latest              fce289e99eb9        6 months ago        1.84kB
[root@docker ~]# docker tag 0890597dc4c9 stanleyws/alpine:v3.10.1_with_1.txt
[root@docker ~]# docker images
REPOSITORY          TAG                  IMAGE ID            CREATED             SIZE
stanleyws/alpine    v3.10.1_with_1.txt   0890597dc4c9        4 minutes ago       5.58MB
stanleyws/alpine    v3.10.1              b7b28af77ffe        2 weeks ago         5.58MB
hello-world         latest               fce289e99eb9        6 months ago        1.84kB
```

## 查看容器的日志
```
[root@docker ~]# docker rm 83f3560e01e9
[root@docker ~]# docker run hello-world 2>&1 >>/dev/null
[root@docker ~]# docker ps -a|grep hello
42bd19e7d5d6        hello-world                "/hello"                 7 seconds ago       Exited (0) 6 seconds ago                          vigorous_banzai
[root@docker ~]# docker logs -f 42bd19e7d5d6

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/

[root@docker ~]# docker start 42bd19e7d5d6
42bd19e7d5d6
[root@docker ~]# docker logs -f 42bd19e7d5d6

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/


Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

# Docker容器的高级操作

## 下载nginx镜像
```
[root@docker ~]# docker pull nginx:1.12.2
1.12.2: Pulling from library/nginx
f2aa67a397c4: Pull complete 
e3eaf3d87fe0: Pull complete 
38cb13c1e4c9: Pull complete 
Digest: sha256:72daaf46f11cc753c4eab981cbf869919bd1fee3d2170a2adeac12400f494728
Status: Downloaded newer image for nginx:1.12.2
docker.io/library/nginx:1.12.2
[root@docker ~]# docker images|grep nginx
nginx               1.12.2                    4037a5562b03        15 months ago       108MB
[root@docker ~]# docker tag 4037a5562b03 stanleyws/nginx:v1.12.2
[root@docker ~]# docker images|grep nginx
nginx               1.12.2                    4037a5562b03        15 months ago       108MB
stanleyws/nginx     v1.12.2                   4037a5562b03        15 months ago       108MB
```

## 映射端口
```
[root@docker ~]# docker run -d -p81:80 stanleyws/nginx:v1.12.2 
cf47229ea987bbe0e6e456695ece7e4f31d51f1b0437b024917ff1ef5d3355ab
[root@docker ~]# netstat -luntp|grep 81
tcp6       0      0 :::81                   :::*                    LISTEN      52802/docker-proxy  
```

```
[root@docker ~]# curl localhost:81
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## 挂载数据卷

```
[root@docker ~]# mkdir html
[root@docker ~]# cd html/
[root@docker html]# wget www.baidu.com -O index.html
--2019-08-01 03:30:31--  http://www.baidu.com/
Resolving www.baidu.com (www.baidu.com)... 220.181.38.150, 220.181.38.149
Connecting to www.baidu.com (www.baidu.com)|220.181.38.150|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 2381 (2.3K) [text/html]
Saving to: 'index.html'

100%[=======================================================================================================>] 2,381       --.-K/s   in 0.04s   

2019-08-01 03:30:31 (57.2 KB/s) - 'index.html' saved [2381/2381]

[root@docker html]# docker run -d -p 82:80 -v /root/html:/usr/share/nginx/html stanleyws/nginx:v1.12.2
cbe1c0ddbc263ec17a9ee87819ae2d125b042876f47da91dcbb21eaf6eb1d8a8
[root@docker html]# docker ps -a|grep nginx
cbe1c0ddbc26        stanleyws/nginx:v1.12.2    "nginx -g 'daemon of??   5 seconds ago       Up 4 seconds              0.0.0.0:82->80/tcp   festive_cartwright
cf47229ea987        stanleyws/nginx:v1.12.2    "nginx -g 'daemon of??   2 minutes ago       Up 2 minutes              0.0.0.0:81->80/tcp   angry_sutherland
```

浏览器打开http://10.4.7.5:82
![baidu](/images/baidu_docker.png "baidu")

## 传递环境变量
```
[root@docker ~]# docker run --rm -e E_OPTS=abcdefg stanleyws/nginx:v1.12.2 printenv
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=f233ef7a0b8a
E_OPTS=abcdefg
NGINX_VERSION=1.12.2-1~stretch
NJS_VERSION=1.12.2.0.1.14-1~stretch
HOME=/root
```

## 容器内安装软件
```
[root@docker ~]# docker run -ti --rm stanleyws/nginx:v1.12.2 bash   
root@b5ff598aeee7:/# curl
bash: curl: command not found
root@b5ff598aeee7:/# tee /etc/apt/sources.list << EOF
> deb http://mirrors.163.com/debian/ jessie main non-free contrib
> deb http://mirrors.163.com/debian/ jessie-updates main non-free contrib
> EOF
deb http://mirrors.163.com/debian/ jessie main non-free contrib
deb http://mirrors.163.com/debian/ jessie-updates main non-free contrib

root@b5ff598aeee7:/# apt-get update && apt-get install curl -y
```

> 提交容器并推送至dockerhub，这个镜像以后还有用！

# Dockerfile
- Dockerfile是一个文本文件，文件名：Dockerfile
- 所有Dockerfile都是由FROM指令开始的
- 构建命令：docker build . -t \[registry\]/\[repository\]:\[tag\]

## USER/WORKDIR指令
- Dockerfile

```vi /data/dockerfile/Dockerfile
FROM stanleyws/nginx:v1.12.2
USER nginx
WORKDIR /usr/share/nginx/html
```

- 构建镜像

```
[root@docker dockerfile]# docker build . -t stanleyws/nginx:v1.12.2_with_user_workdir
Sending build context to Docker daemon  2.048kB
Step 1/3 : FROM stanleyws/nginx:v1.12.2
 ---> 4037a5562b03
Step 2/3 : USER nginx
 ---> Using cache
 ---> 501d118fae5f
Step 3/3 : WORKDIR /usr/share/nginx/html
 ---> Using cache
 ---> 1014c53fc0c1
Successfully built 1014c53fc0c1
Successfully tagged stanleyws/nginx:v1.12.2_with_user_workdir
```

- 运行容器

```
[root@docker dockerfile]# docker run -ti --rm stanleyws/nginx:v1.12.2_with_user_workdir bash
nginx@d77439235690:/usr/share/nginx/html$ pwd
/usr/share/nginx/html
nginx@d77439235690:/usr/share/nginx/html$ id
uid=101(nginx) gid=101(nginx) groups=101(nginx)
```

## ADD/EXPOSE指令
- Dockerfile

```vi /data/dockerfile/Dockerfile
FROM stanleyws/nginx:v1.12.2
ADD index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

- 构建镜像

```
[root@docker dockerfile]# docker build . -t stanleyws/nginx:v1.12.2_with_index_expose
Sending build context to Docker daemon   5.12kB
Step 1/3 : FROM stanleyws/nginx:v1.12.2
 ---> 4037a5562b03
Step 2/3 : ADD index.html /usr/share/nginx/html/index.html
 ---> a39ee166812b
Step 3/3 : EXPOSE 80
 ---> Running in b759b5a11471
Removing intermediate container b759b5a11471
 ---> aa982c654c06
Successfully built aa982c654c06
Successfully tagged stanleyws/nginx:v1.12.2_with_index_expose
```

-  运行容器

```
[root@docker dockerfile]# docker run -d -P stanleyws/nginx:v1.12.2_with_index_expose
a21c775757b55ef55f708a149580a44be08e91aab6e9565bd0ccbdbaf493e49a
[root@docker dockerfile]# docker ps -a|grep nginx
a21c775757b5        stanleyws/nginx:v1.12.2_with_index_expose   "nginx -g 'daemon of??   8 seconds ago       Up 7 seconds              0.0.0.0:4013->80/tcp   crazy_einstein
```

- 验证

浏览器打开http://10.4.7.5:4013

## RUN/ENV指令
- Dockerfile

```vi /data/dockerfile/Dockerfile
FROM centos
ENV VER 9.9.4-74.el7_6.1
RUN yum install bind-$VER -y
```

- 构建镜像

```
[root@docker dockerfile]# docker build . -t stanleyws/bind:v9.9.4_with_env_run
```

- 运行容器

```
[root@docker dockerfile]# docker run --rm stanleyws/bind:v9.9.4_with_env_run rpm -qa bind
bind-9.9.4-74.el7_6.1.x86_64
```

## CMD/ENTRYPOINT指令
> CMD指令和ENTRYPOINT指令作用相同，使用方法略有不同

### CMD指令

- Dockerfile

```vi /data/dockerfile/Dockerfile
FROM centos
RUN yum install httpd -y
CMD ["httpd","-D","FOREGROUND"]
```

- 构建镜像

```
[root@docker dockerfile]# docker build . -t stanleyws/httpd:myhttpd
Sending build context to Docker daemon  2.048kB
Step 1/3 : FROM centos
 ---> 9f38484d220f
Step 2/3 : RUN yum install httpd -y
 ---> Running in c6c82781dc44
Loaded plugins: fastestmirror, ovl
Determining fastest mirrors
 * base: mirrors.163.com
 * extras: mirrors.aliyun.com
 * updates: mirrors.163.com
Resolving Dependencies
--> Running transaction check
---> Package httpd.x86_64 0:2.4.6-89.el7.centos.1 will be installed
--> Processing Dependency: httpd-tools = 2.4.6-89.el7.centos.1 for package: httpd-2.4.6-89.el7.centos.1.x86_64
--> Processing Dependency: system-logos >= 7.92.1-1 for package: httpd-2.4.6-89.el7.centos.1.x86_64
--> Processing Dependency: /etc/mime.types for package: httpd-2.4.6-89.el7.centos.1.x86_64
--> Processing Dependency: libaprutil-1.so.0()(64bit) for package: httpd-2.4.6-89.el7.centos.1.x86_64
--> Processing Dependency: libapr-1.so.0()(64bit) for package: httpd-2.4.6-89.el7.centos.1.x86_64
--> Running transaction check
---> Package apr.x86_64 0:1.4.8-3.el7_4.1 will be installed
---> Package apr-util.x86_64 0:1.5.2-6.el7 will be installed
---> Package centos-logos.noarch 0:70.0.6-3.el7.centos will be installed
---> Package httpd-tools.x86_64 0:2.4.6-89.el7.centos.1 will be installed
---> Package mailcap.noarch 0:2.1.41-2.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package           Arch        Version                       Repository    Size
================================================================================
Installing:
 httpd             x86_64      2.4.6-89.el7.centos.1         updates      2.7 M
Installing for dependencies:
 apr               x86_64      1.4.8-3.el7_4.1               base         103 k
 apr-util          x86_64      1.5.2-6.el7                   base          92 k
 centos-logos      noarch      70.0.6-3.el7.centos           base          21 M
 httpd-tools       x86_64      2.4.6-89.el7.centos.1         updates       91 k
 mailcap           noarch      2.1.41-2.el7                  base          31 k

Transaction Summary
================================================================================
Install  1 Package (+5 Dependent packages)

Total download size: 24 M
Installed size: 31 M
Downloading packages:
warning: /var/cache/yum/x86_64/7/base/packages/apr-1.4.8-3.el7_4.1.x86_64.rpm: Header V3 RSA/SHA256 Signature, key ID f4a80eb5: NOKEY
Public key for apr-1.4.8-3.el7_4.1.x86_64.rpm is not installed
Public key for httpd-tools-2.4.6-89.el7.centos.1.x86_64.rpm is not installed
--------------------------------------------------------------------------------
Total                                              1.1 MB/s |  24 MB  00:21     
Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Importing GPG key 0xF4A80EB5:
 Userid     : "CentOS-7 Key (CentOS 7 Official Signing Key) <security@centos.org>"
 Fingerprint: 6341 ab27 53d7 8a78 a7c2 7bb1 24c6 a8a7 f4a8 0eb5
 Package    : centos-release-7-6.1810.2.el7.centos.x86_64 (@CentOS)
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : apr-1.4.8-3.el7_4.1.x86_64                                   1/6 
  Installing : apr-util-1.5.2-6.el7.x86_64                                  2/6 
  Installing : httpd-tools-2.4.6-89.el7.centos.1.x86_64                     3/6 
  Installing : centos-logos-70.0.6-3.el7.centos.noarch                      4/6 
  Installing : mailcap-2.1.41-2.el7.noarch                                  5/6 
  Installing : httpd-2.4.6-89.el7.centos.1.x86_64                           6/6 
  Verifying  : httpd-2.4.6-89.el7.centos.1.x86_64                           1/6 
  Verifying  : httpd-tools-2.4.6-89.el7.centos.1.x86_64                     2/6 
  Verifying  : mailcap-2.1.41-2.el7.noarch                                  3/6 
  Verifying  : apr-util-1.5.2-6.el7.x86_64                                  4/6 
  Verifying  : apr-1.4.8-3.el7_4.1.x86_64                                   5/6 
  Verifying  : centos-logos-70.0.6-3.el7.centos.noarch                      6/6 

Installed:
  httpd.x86_64 0:2.4.6-89.el7.centos.1                                          

Dependency Installed:
  apr.x86_64 0:1.4.8-3.el7_4.1                                                  
  apr-util.x86_64 0:1.5.2-6.el7                                                 
  centos-logos.noarch 0:70.0.6-3.el7.centos                                     
  httpd-tools.x86_64 0:2.4.6-89.el7.centos.1                                    
  mailcap.noarch 0:2.1.41-2.el7                                                 

Complete!
Removing intermediate container c6c82781dc44
 ---> 9d930be9c4c3
Step 3/3 : CMD ["httpd","-D","FOREGROUND"]
 ---> Running in 7be509de6df6
Removing intermediate container 7be509de6df6
 ---> 1c87162a8e3e
Successfully built 1c87162a8e3e
Successfully tagged stanleyws/httpd:myhttpd
```

- 运行容器

```
[root@docker dockerfile]# docker run -d -p80 stanleyws/httpd:myhttpd
6ae9da311ee977c24ae4c10ae6aa3f6baf80c80b67c324257020cd9b8c0b2c36
[root@docker dockerfile]# docker ps -a|grep httpd
5a8fff36428d        stanleyws/httpd:myhttpd              "httpd -D FOREGROUND"    7 seconds ago       Up 5 seconds                     0.0.0.0:4000->80/tcp   cool_moser
```

- 验证

浏览器打开：http://10.4.7.5:4000

### ENTRYPOINT指令

- Dockerfile

```vi /data/dockerfile/Dockerfile
FROM centos
ADD entrypoint.sh /entrypoint.sh
RUN yum install epel-release -q -y && yum install nginx -y
ENTRYPOINT /entrypoint.sh
```

- entrypoint.sh

```vi /data/dockerfile/entrypoint.sh
#!/bin/bash
/sbin/nginx -g "daemon off;"
```
**注意：**entrypoint.sh不要忘了给执行权限！

- 构建镜像

```
[root@docker dockerfile]# docker build . -t stanleyws/nginx:mynginx
```

- 运行容器

```
[root@docker dockerfile]# docker run -d -p8080:80 -v /root/html:/usr/share/nginx/html stanleyws/nginx:mynginx
```

- 验证

浏览器打开：http://10.4.7.5:8080

# 综合实验
> 运行一个docker容器，在浏览器打开demo.od.com能访问到百度首页

## 准备Docker镜像
- Dockerfile

```vi /data/dockerfile/nginx/Dockerfile
FROM stanleyws/nginx:v1.12.2
USER root
ENV WWW /usr/share/nginx/html
ENV CONF /etc/nginx/conf.d
RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime &&\ 
    echo 'Asia/Shanghai' >/etc/timezone
WORKDIR $WWW
ADD index.html $WWW/index.html
ADD demo.od.com.conf $CONF/demo.od.com.conf
EXPOSE 80
CMD ["nginx","-g","daemon off;"]
```

- index.html

```vi /data/dockerfile/nginx/index.html
[root@docker nginx]# wget www.baidu.com -O index.html
```

- demo.od.com.conf

```vi /data/dockerfile/nginx/demo.od.com.conf
server {
   listen 80;
   server_name demo.od.com;

   root /usr/share/nginx/html;
}
```

## 构建镜像
```
[root@docker nginx]# docker build . -t stanleyws/nginx:v1.12.2_with_index
Sending build context to Docker daemon  7.168kB
Step 1/10 : FROM stanleyws/nginx:v1.12.2
 ---> 4037a5562b03
Step 2/10 : USER root
 ---> Using cache
 ---> c339a693f253
Step 3/10 : ENV WWW /usr/share/nginx/html
 ---> Running in 48bd89045b24
Removing intermediate container 48bd89045b24
 ---> 8f7b5bdd5f3b
Step 4/10 : ENV CONF /etc/nginx/conf.d
 ---> Running in ba5aed9937e2
Removing intermediate container ba5aed9937e2
 ---> 11d853d9e494
Step 5/10 : RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime &&    echo 'Asia/Shanghai' >/etc/timezone
 ---> Running in 0f18a702f1ce
Removing intermediate container 0f18a702f1ce
 ---> 412f5d023783
Step 6/10 : WORKDIR $WWW
 ---> Running in 09a7d669c92f
Removing intermediate container 09a7d669c92f
 ---> 816b56406b0f
Step 7/10 : ADD index.html $WWW/index.html
 ---> c2897bca9502
Step 8/10 : ADD demo.od.com.conf $CONF/demo.od.com.conf
 ---> 5e1cdf39b689
Step 9/10 : EXPOSE 80
 ---> Running in 421582feae06
Removing intermediate container 421582feae06
 ---> 7f574abcc9ad
Step 10/10 : CMD ["nginx","-g","daemon off;"]
 ---> Running in 869f5eec9e82
Removing intermediate container 869f5eec9e82
 ---> d29aa9242b4d
Successfully built d29aa9242b4d
Successfully tagged nginx:v1.12.2_with_index
```

## 启动容器
```
[root@docker nginx]# docker run -d -p80:80 stanleyws/nginx:v1.12.2_with_index
```

## 验证
绑hosts，打开浏览器http://demo.od.com
![dockerdemo](/images/dockerdemo.png "dockerdemo")

# Docker的网络模型
## NAT（默认）
```
[root@docker ~]# docker run -ti --rm myalpine /bin/sh
/ # ip add
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
204: eth0@if205: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:07:05:06 brd ff:ff:ff:ff:ff:ff
    inet 172.7.5.6/24 brd 172.7.5.255 scope global eth0
       valid_lft forever preferred_lft forever
```

## None
```
[root@docker ~]# docker run -ti --rm --net=none myalpine /bin/sh
/ # ip add
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
```

## Host
```
[root@docker ~]# docker run -ti --rm --net=host myalpine /bin/sh
/ # ip add
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
    link/ether 00:15:5d:2d:54:02 brd ff:ff:ff:ff:ff:ff
    inet 10.4.7.5/24 brd 10.4.7.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::1ac7:8272:7cf7:a49c/64 scope link 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN 
    link/ether 02:42:31:67:ed:9d brd ff:ff:ff:ff:ff:ff
    inet 172.7.5.1/24 brd 172.7.5.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:31ff:fe67:ed9d/64 scope link 
       valid_lft forever preferred_lft forever
```

## 联合网络*
```
[root@docker ~]# docker run -d stanleyws/nginx:v1.12.2
1b18e7b1dcd4f629af234ffbd9fed703cdb8b8a5163feafd2c43113f26d270b4

[root@docker ~]# docker ps -a
CONTAINER ID        IMAGE                     COMMAND                  CREATED              STATUS              PORTS               NAMES
1b18e7b1dcd4        stanleyws/nginx:v1.12.2   "nginx -g 'daemon of??   About a minute ago   Up About a minute   80/tcp              condescending_aryabhata

[root@docker ~]# docker run -ti --rm --net=container:1b18e7b1 stanleyws/nginx:curl bash
root@1b18e7b1dcd4:/# curl localhost
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```