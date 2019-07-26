title: Day0：Docker容器入门基础
author: Stanley Wang
date: 2019-6-18 23:12:56
---
- - -
{% cq %}欢迎加入王导的VIP学习qq群：==>[<font color="FF7F50">932194668</font>](http://shang.qq.com/wpa/qunwpa?idkey=78869fddc5a661acb0639315eb52997c108de6625df5f0ee2f0372f176a032a6)<=={% endcq %}
- - -
# Docker引擎安装部署
## 实验环境
- 宿主机操作系统版本：
> CentOS Linux release 7.6.1810 (Core) 

- Docker版本：
> Docker-CE-19.03

- 系统网络：
宿主机|Docker
-|-
10.4.7.5|172.7.5.0/24

## 安装部署
```
[root@docker ~]# curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
# Executing docker install script, commit: 6bf300318ebaab958c4adc341a8c7bb9f3a54a1a
+ sh -c 'yum install -y -q yum-utils'
+ sh -c 'yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo'
Loaded plugins: fastestmirror
adding repo from: https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
grabbing file https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo to /etc/yum.repos.d/docker-ce.repo
repo saved to /etc/yum.repos.d/docker-ce.repo
+ '[' stable '!=' stable ']'
+ sh -c 'yum makecache'
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
epel/x86_64/metalink                                                                                                      | 8.5 kB  00:00:00     
 * epel: mirrors.tuna.tsinghua.edu.cn
base                                                                                                                      | 3.6 kB  00:00:00     
docker-ce-stable                                                                                                          | 3.5 kB  00:00:00     
epel                                                                                                                      | 5.3 kB  00:00:00     
extras                                                                                                                    | 3.4 kB  00:00:00     
updates                                                                                                                   | 3.4 kB  00:00:00     
(1/14): docker-ce-stable/x86_64/updateinfo                                                                                |   55 B  00:00:00     
(2/14): docker-ce-stable/x86_64/filelists_db                                                                              |  15 kB  00:00:00     
(3/14): docker-ce-stable/x86_64/primary_db                                                                                |  31 kB  00:00:00     
(4/14): docker-ce-stable/x86_64/other_db                                                                                  | 112 kB  00:00:00     
(5/14): epel/x86_64/filelists_db                                                                                          |  12 MB  00:00:05     
(6/14): epel/x86_64/prestodelta                                                                                           | 1.7 kB  00:00:00     
(7/14): epel/x86_64/other_db                                                                                              | 3.3 MB  00:00:01     
(8/14): epel/x86_64/updateinfo_zck                                                                                        | 1.5 MB  00:00:00     
(9/14): extras/7/x86_64/prestodelta                                                                                       |  65 kB  00:00:00     
(10/14): extras/7/x86_64/other_db                                                                                         | 127 kB  00:00:00     
(11/14): extras/7/x86_64/filelists_db                                                                                     | 246 kB  00:00:00     
(12/14): updates/7/x86_64/prestodelta                                                                                     | 857 kB  00:00:00     
(13/14): updates/7/x86_64/other_db                                                                                        | 672 kB  00:00:00     
(14/14): updates/7/x86_64/filelists_db                                                                                    | 4.9 MB  00:00:02     
Metadata Cache Created
+ '[' -n '' ']'
+ sh -c 'yum install -y -q docker-ce'
warning: /var/cache/yum/x86_64/7/docker-ce-stable/packages/docker-ce-19.03.0-3.el7.x86_64.rpm: Header V4 RSA/SHA512 Signature, key ID 621e9f35: NOKEY
Public key for docker-ce-19.03.0-3.el7.x86_64.rpm is not installed
Importing GPG key 0x621E9F35:
 Userid     : "Docker Release (CE rpm) <docker@docker.com>"
 Fingerprint: 060a 61c5 1b55 8a7f 742b 77aa c52f eb6b 621e 9f35
 From       : https://mirrors.aliyun.com/docker-ce/linux/centos/gpg
setsebool:  SELinux is disabled.
If you would like to use Docker as a non-root user, you should now consider
adding your user to the "docker" group with something like:

  sudo usermod -aG docker your-user

Remember that you will have to log out and back in for this to take effect!

WARNING: Adding a user to the "docker" group will grant the ability to run
         containers which can be used to obtain root privileges on the
         docker host.
         Refer to https://docs.docker.com/engine/security/security/#docker-daemon-attack-surface
         for more information.
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
## 启动
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

# 容器的基本操作
## 查看本地的容器进程
```
[root@docker ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                   PORTS               NAMES
83f3560e01e9        hello-world         "/hello"            2 hours ago         Exited (0) 2 hours ago                       zen_dhawan
```

## 运行镜像（启动容器）
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
[root@docker ~]# docker run -d --name myalpine stanleyws/alpine:v3.10.1 /bin/sleep 3000
b42487bc0ead6e9f21db1719ea12658949f9f0c73b74441f7809e1841da9f8c1
```

### 查看容器
#### 查看运行中的容器
```
[root@docker ~]# docker ps 
CONTAINER ID        IMAGE                      COMMAND             CREATED             STATUS              PORTS               NAMES
b42487bc0ead        stanleyws/alpine:v3.10.1   "/bin/sleep 300"    3 seconds ago       Up 3 seconds                            myalpine
```
#### 查看所有容器
```
[root@docker ~]# docker ps -a
CONTAINER ID        IMAGE                      COMMAND             CREATED              STATUS                   PORTS               NAMES
b42487bc0ead        stanleyws/alpine:v3.10.1   "/bin/sleep 300"    About a minute ago   Up About a minute                          myalpine
83f3560e01e9        hello-world                "/hello"            5 mins ago          Exited (0) 5 mins ago                       zen_dhawan
```

### 进入容器
```
[root@docker ~]# docker exec -ti myalpine /bin/sh
/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sleep 3000
    6 root      0:00 /bin/sh
   11 root      0:00 ps aux
```

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

### 删除容器
```
[root@docker ~]# docker rm myalpine
Error response from daemon: You cannot remove a running container 3614b532e596d2e59a054f1787929f62145649b98744e0efb23cac5beb0ac1f9. Stop the container before attempting removal or force remove
[root@docker ~]# docker rm -f myalpine
myalpine
```
> 如果没有停止的容器，需要用docker rm -f 进行强制删除

### 修改并提交容器
```
[root@docker ~]# docker exec -ti 91abfbefd58 /bin/sh
/ # pwd
/
/ # echo Hello > 1.txt
/ # ls
1.txt  bin    dev    etc    home   lib    media  mnt    opt    proc   root   run    sbin   srv    sys    tmp    usr    var
/ # exit
[root@docker ~]# docker commit -p 91abfbefd58 stanleyws/alpine:v3.10.1_with_1.txt
sha256:0890597dc4c9ce1be2e50d3121648cb3612500360c1d6c69545d1dcf5e851eee
```
验证
```
[root@docker ~]# docker run --rm stanleyws/alpine:v3.10.1_with_1.txt /bin/cat /1.txt
Hello
```
> 也可以使用docker export 91abfbefd58 > myalpine_with_1.txt.tar

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

