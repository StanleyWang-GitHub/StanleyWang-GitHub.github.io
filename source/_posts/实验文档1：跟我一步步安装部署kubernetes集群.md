title: 实验文档1：跟我一步步安装部署kubernetes集群
author: Stanley Wang
categories: Kubernetes容器云专题
date: 2019-1-16 22:12:56
---
- - -
{% cq %}欢迎加入王导的VIP学习qq群：==>[<font color="FF7F50">932194668</font>](http://shang.qq.com/wpa/qunwpa?idkey=78869fddc5a661acb0639315eb52997c108de6625df5f0ee2f0372f176a032a6)<=={% endcq %}
- - -
# 实验环境
## 基础架构
主机名|角色|ip
-|-|-
HDSS7-11.host.com|k8s代理节点1|10.4.7.11
HDSS7-12.host.com|k8s代理节点2|10.4.7.12
HDSS7-21.host.com|k8s运算节点1|10.4.7.21
HDSS7-22.host.com|k8s运算节点2|10.4.7.22
HDSS7-200.host.com|k8s运维节点(docker仓库)|10.4.7.200

## 硬件环境
- 5台vm，每台至少2c2g

## 软件环境
- OS: CentOS Linux release 7.6.1810 (Core)
- docker: v1.12.6
> [docker引擎官方下载地址](https://yum.dockerproject.org/repo/main/centos/7/Packages/docker-engine-1.12.6-1.el7.centos.x86_64.rpm)
> [docker引擎官方selinux包](https://yum.dockerproject.org/repo/main/centos/7/Packages/docker-engine-selinux-1.12.6-1.el7.centos.noarch.rpm)
- kubernetes: v1.13.2
> [kubernetes官方下载地址](https://github.com/kubernetes/kubernetes/releases)
- etcd: v3.1.18
> [etcd官方下载地址](https://github.com/etcd-io/etcd/releases)
- flannel: v0.10.0
> [flannel官方下载地址](https://github.com/coreos/flannel/releases)
- bind9: v9.9.4
> [bind9官方下载地址](https://www.isc.org/downloads/#)
- harbor: v1.7.1
> [harbor官方下载地址](https://github.com/goharbor/harbor/releases)
- 证书签发工具CFSSL: R1.2
> [cfssl下载地址](https://pkg.cfssl.org/R1.2/cfssl_linux-amd64)
> [cfssljson下载地址](https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64)
> [cfssl-certinfo下载地址](https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64)
- 其他
> 其他可能用到的软件，均使用操作系统自带的yum源和epel源进行安装

# 安装部署
## BIND9安装部署
- 创建主机域host.com
- 创建业务域od.com
- 主辅同步(10.4.7.11主、10.4.7.12辅)
- 客户端配置指向自建DNS

略

## k8s运算节点和运维节点安装docker环境(3台)
```
# ls -l|grep docker-engine
-rw-r--r--  1 root root  20013304 Jan 22 18:16 docker-engine-1.12.6-1.el7.centos.x86_64.rpm
-rw-r--r--  1 root root     29112 Jan 22 18:15 docker-engine-selinux-1.12.6-1.el7.centos.noarch.rpm
# yum localinstall *.rpm
```

## 配置并启动docker(3台)
### 配置
```vi /etc/docker/daemon.json 
# vi /etc/docker/daemon.json 
{
  "graph": "/data/docker",
  "storage-driver": "overlay",
  "insecure-registries": ["registry.access.redhat.com","quay.io","harbor.od.com"],
  "bip": "172.7.21.1/24",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "live-restore": true
}
```
**注意：**这里bip要根据宿主机ip变化

### 启动脚本
```vi /usr/lib/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network.target

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
#TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process

[Install]
WantedBy=multi-user.target
```

### 启动
```
# systemctl enable docker.service
# systemctl start docker.service
```

## 部署docker镜像私有仓库harbor
**`HDSS7-200.host.com`上**
### 下载软件二进制包并解压
```pwd /opt/harbor
[root@hdss7-200 harbor]# tar xf harbor-offline-installer-v1.7.1.tgz -C /opt/harbor

[root@hdss7-200 harbor]# ll
total 583848
drwxr-xr-x 3 root root       242 Jan 23 15:28 harbor
-rw-r--r-- 1 root root 597857483 Jan 17 14:58 harbor-offline-installer-v1.7.1.tgz
```

### 配置
```vi /opt/harbor/harbor.cfg
hostname = harbor.od.com
```
```vi /opt/harbor/docker-compose.yml
ports:
  - 180:80
  - 1443:443
  - 4443:4443
```

### 安装docker-compose
```
[root@hdss7-200 harbor]# yum install docker-compose -y
[root@hdss7-200 harbor]# rpm -qa docker-compose
docker-compose-1.18.0-2.el7.noarch
```

### 安装harbor
```pwd /opt/harbor
[root@hdss7-200 harbor]# ./install.sh 
```

### 检查harbor启动情况
```
[root@hdss7-200 harbor]# docker-compose ps
       Name                     Command               State                                 Ports                               
--------------------------------------------------------------------------------------------------------------------------------
harbor-adminserver   /harbor/start.sh                 Up                                                                        
harbor-core          /harbor/start.sh                 Up                                                                        
harbor-db            /entrypoint.sh postgres          Up      5432/tcp                                                          
harbor-jobservice    /harbor/start.sh                 Up                                                                        
harbor-log           /bin/sh -c /usr/local/bin/ ...   Up      127.0.0.1:1514->10514/tcp                                         
harbor-portal        nginx -g daemon off;             Up      80/tcp                                                            
nginx                nginx -g daemon off;             Up      0.0.0.0:1443->443/tcp, 0.0.0.0:4443->4443/tcp, 0.0.0.0:180->80/tcp
redis                docker-entrypoint.sh redis ...   Up      6379/tcp                                                          
registry             /entrypoint.sh /etc/regist ...   Up      5000/tcp                                                          
registryctl          /harbor/start.sh                 Up     
```

### 配置harbor的dns内网解析
```vi /var/named/od.com.zone
harbor	60 IN A 10.4.7.200
```
检查
```
[root@hdss7-200 harbor]# dig -t A harbor.od.com @10.4.7.11 +short
10.4.7.200
```

### 安装nginx并配置
#### 安装
```
[root@hdss7-200 harbor]# yum install nginx -y
[root@hdss7-200 harbor]# rpm -qa nginx
nginx-1.12.2-2.el7.x86_64
```
#### 配置
```vi /etc/nginx/conf.d/harbor.od.com.conf
server {
    listen       80;
    server_name  harbor.od.com;

    client_max_body_size 1000m;

    location / {
        proxy_pass http://127.0.0.1:180;
    }
}
server {
    listen       443 ssl;
    server_name  harbor.od.com;

    ssl_certificate "certs/harbor.od.com.pem";
    ssl_certificate_key "certs/harbor.od.com-key.pem";
    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout  10m;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    client_max_body_size 1000m;

    location / {
        proxy_pass http://127.0.0.1:180;
    }
}
```
**注意：**这里需要自签ssl证书，自签过程略
#### 启动
```
[root@hdss7-200 harbor]# nginx

[root@hdss7-200 harbor]# netstat -luntp|grep nginx
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      6590/nginx: master  
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      6590/nginx: master  
```

### 浏览器打开`harbor.od.com`并登陆
- 用户名：admin
- 密码： Harbor12345

## 部署kubernetes
kubernetes集群一共包含以下几个核心组件：
- etcd（注册中心）
- kube-apiserver（master）
- controller-manager（master）
- kube-scheduler（master）
- kubelet（node）
- kube-proxy（node）
kubernetes集群的附件：
- flannel（网络）
- coredns（dns）
- traefik（ingress）
- dashboard（UI）

### 准备工作，部署签发证书环境
####  在运维主机安装CFSSL
**HDSS7-200.host.com上**
```
[root@hdss7-200 ~]# curl -s -L -o /bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 
[root@hdss7-200 ~]# curl -s -L -o /bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 
[root@hdss7-200 ~]# curl -s -L -o /bin/cfssl-certinfo https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 
[root@hdss7-200 ~]# chmod +x /bin/cfssl*
```
#### 创建生成CA证书的JSON配置文件
```vi /opt/certs/ca-config.json
{
    "signing": {
        "default": {
            "expiry": "175200h"
        },
        "profiles": {
            "server": {
                "expiry": "175200h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            },
            "client": {
                "expiry": "175200h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            },
            "peer": {
                "expiry": "175200h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
```
> 证书类型
> client certificate： 用于服务端认证客户端,例如etcdctl、etcd proxy、fleetctl、docker客户端
> server certificate: 服务端使用，客户端以此验证服务端身份,例如docker服务端、kube-apiserver
> peer certificate: 双向证书，用于etcd集群成员间通信

#### 创建生成CA证书签名请求（csr）的JSON配置文件
```vi /opt/certs/ca-csr.json
{
    "CN": "kubernetes-ca",
    "hosts": [
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "beijing",
            "L": "beijing",
            "O": "od",
            "OU": "ops"
        }
    ],
    "ca": {
        "expiry": "175200h"
    }
}
```
> CN: Common Name，浏览器使用该字段验证网站是否合法，一般写的是域名。非常重要。浏览器使用该字段验证网站是否合法
> C: Country， 国家
> ST: State，州，省
> L: Locality，地区，城市
> O: Organization Name，组织名称，公司名称
> OU: Organization Unit Name，组织单位名称，公司部门

#### 生成CA证书和私钥
```pwd /opt/certs
[root@hdss7-200 certs]# cfssl gencert -initca ca-csr.json | cfssljson -bare ca - 
```
生成ca.pem、ca.csr、ca-key.pem(CA私钥,需妥善保管)
```pwd /opt/certs
[root@hdss7-200 certs]# ls -l
-rw-r--r-- 1 root root  836 Jan 18 11:04 ca-config.json
-rw-r--r-- 1 root root  332 Jan 18 11:10 ca-csr.json
-rw------- 1 root root 1675 Jan 18 11:17 ca-key.pem
-rw-r--r-- 1 root root 1001 Jan 18 11:17 ca.csr
-rw-r--r-- 1 root root 1354 Jan 18 11:17 ca.pem
```
### 部署etcd集群
#### 集群规划
主机名|角色|ip
-|-|-
HDSS7-12.host.com|etcd lead|10.4.7.12
HDSS7-21.host.com|etcd follow|10.4.7.21
HDSS7-22.host.com|etcd follow|10.4.7.22

**注意：**这里部署文档以`HDSS7-12.host.com`主机为例，另外两台主机安装部署方法类似

#### 创建生成证书签名请求（csr）的JSON配置文件
```vi /opt/certs/etcd-peer-csr.json
{
    "CN": "etcd-peer",
    "hosts": [
        "127.0.0.1",
        "10.4.7.11",
        "10.4.7.12",
        "10.4.7.21",
        "10.4.7.22"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "beijing",
            "L": "beijing",
            "O": "od",
            "OU": "ops"
        }
    ]
}
```
#### 生成etcd证书和私钥
```pwd /opt/certs
[root@hdss7-200 certs]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer etcd-peer-csr.json | cfssljson -bare etcd-peer
```
#### 检查生成的证书、私钥
```ll /opt/certs
[root@hdss7-200 certs]# ls -l|grep etcd
-rw-r--r-- 1 root root  387 Jan 18 12:32 etcd-peer-csr.json
-rw------- 1 root root 1679 Jan 18 12:32 etcd-peer-key.pem
-rw-r--r-- 1 root root 1074 Jan 18 12:32 etcd-peer.csr
-rw-r--r-- 1 root root 1432 Jan 18 12:32 etcd-peer.pem
```

#### 创建etcd用户
```
[root@hdss7-12 ~]# useradd -s /sbin/nologin -M etcd
```
#### 下载软件，解压，做软连接
```pwd /opt/src
[root@hdss7-12 src]# ls -l
total 9604
-rw-r--r-- 1 root root 9831476 Jan 18 10:45 etcd-v3.1.18-linux-amd64.tar.gz
[root@hdss7-12 src]# tar xf etcd-v3.1.18-linux-amd64.tar.gz -C /opt
[root@hdss7-12 src]# ln -s /opt/etcd-v3.1.18-linux-amd64 /opt/etcd
[root@hdss7-12 src]# ls -l /opt
total 0
lrwxrwxrwx 1 root   root   24 Jan 18 14:21 etcd -> etcd-v3.1.18-linux-amd64
drwxr-xr-x 4 478493 89939 166 Jun 16  2018 etcd-v3.1.18-linux-amd64
drwxr-xr-x 2 root   root   45 Jan 18 14:21 src
```
#### 创建目录，拷贝证书、私钥
```
[root@hdss7-12 src]# mkdir -p /data/etcd /data/logs/etcd-server 
[root@hdss7-12 src]# chown -R etcd.etcd /data/etcd /data/logs/etcd-server/
[root@hdss7-12 src]# mkdir -p /opt/etcd/certs
```
将运维主机上生成的`ca.pem`、`etcd-peer-key.pem`、`etcd-peer.pem`拷贝到`/opt/etcd/certs`目录中，注意私钥文件权限600
```pwd /opt/etcd/certs
[root@hdss7-12 certs]# chmod 600 etcd-peer-key.pem
[root@hdss7-12 certs]# chown -R etcd.etcd /opt/etcd/certs/
[root@hdss7-12 certs]# ls -l
total 12
-rw-r--r-- 1 etcd etcd 1354 Jan 18 14:45 ca.pem
-rw------- 1 etcd etcd 1679 Jan 18 17:00 etcd-peer-key.pem
-rw-r--r-- 1 etcd etcd 1444 Jan 18 17:02 etcd-peer.pem
```
#### 创建etcd服务启动脚本
```vi /opt/etcd/etcd-server-startup.sh
#!/bin/sh
./etcd --name etcd-server-7-12 \
       --data-dir /data/etcd/etcd-server \
       --listen-peer-urls https://10.4.7.12:2380 \
       --listen-client-urls https://10.4.7.12:2379,http://127.0.0.1:2379 \
       --quota-backend-bytes 8000000000 \
       --initial-advertise-peer-urls https://10.4.7.12:2380 \
       --advertise-client-urls https://10.4.7.12:2379,http://127.0.0.1:2379 \
       --initial-cluster  etcd-server-7-12=https://10.4.7.12:2380,etcd-server-7-21=https://10.4.7.21:2380,etcd-server-7-22=https://10.4.7.22:2380 \
       --ca-file ./certs/ca.pem \
       --cert-file ./certs/etcd-peer.pem \
       --key-file ./certs/etcd-peer-key.pem \
       --client-cert-auth  \
       --trusted-ca-file ./certs/ca.pem \
       --peer-ca-file ./certs/ca.pem \
       --peer-cert-file ./certs/etcd-peer.pem \
       --peer-key-file ./certs/etcd-peer-key.pem \
       --peer-client-cert-auth \
       --peer-trusted-ca-file ./certs/ca.pem \
       --log-output stdout
```
**注意：**etcd集群各主机的启动脚本略有不同，部署其他节点时注意修改。
#### 调整权限和目录
```
[root@hdss7-12 certs]# chmod +x /opt/etcd/etcd-server-startup.sh
[root@hdss7-12 certs]# mkdir -p /data/logs/etcd-server
```
#### 安装supervisor软件
```
[root@hdss7-12 certs]# yum install supervisor -y
[root@hdss7-12 certs]# systemctl start supervisord
[root@hdss7-12 certs]# systemctl enable supervisord
```
#### 创建etcd-server的启动配置
```vi /etc/supervisord.d/etcd-server.ini
[program:etcd-server-7-12]
command=/opt/etcd/etcd-server-startup.sh                        ; the program (relative uses PATH, can take args)
numprocs=1                                                      ; number of processes copies to start (def 1)
directory=/opt/etcd                                             ; directory to cwd to before exec (def no cwd)
autostart=true                                                  ; start at supervisord start (default: true)
autorestart=true                                                ; retstart at unexpected quit (default: true)
startsecs=22                                                    ; number of secs prog must stay running (def. 1)
startretries=3                                                  ; max # of serial start failures (default 3)
exitcodes=0,2                                                   ; 'expected' exit codes for process (default 0,2)
stopsignal=QUIT                                                 ; signal used to kill process (default TERM)
stopwaitsecs=10                                                 ; max num secs to wait b4 SIGKILL (default 10)
user=etcd                                                       ; setuid to this UNIX account to run the program
redirect_stderr=false                                           ; redirect proc stderr to stdout (default false)
stdout_logfile=/data/logs/etcd-server/etcd.stdout.log           ; stdout log path, NONE for none; default AUTO
stdout_logfile_maxbytes=64MB                                    ; max # logfile bytes b4 rotation (default 50MB)
stdout_logfile_backups=4                                        ; # of stdout logfile backups (default 10)
stdout_capture_maxbytes=1MB                                     ; number of bytes in 'capturemode' (default 0)
stdout_events_enabled=false                                     ; emit events on stdout writes (default false)
stderr_logfile=/data/logs/etcd-server/etcd.stderr.log           ; stderr log path, NONE for none; default AUTO
stderr_logfile_maxbytes=64MB                                    ; max # logfile bytes b4 rotation (default 50MB)
stderr_logfile_backups=4                                        ; # of stderr logfile backups (default 10)
stderr_capture_maxbytes=1MB                                     ; number of bytes in 'capturemode' (default 0)
stderr_events_enabled=false                                     ; emit events on stderr writes (default false)
```
**注意：**etcd集群各主机启动配置略有不同，配置其他节点时注意修改。

#### 启动etcd服务并检查
```
[root@hdss7-12 certs]# supervisorctl start all
[root@hdss7-12 certs]# supervisorctl status   
etcd-server-7-12                 RUNNING   pid 6692, uptime 5:37:25
```

#### 安装部署启动检查所有集群规划主机上的etcd服务
略

#### 检查集群状态
3台均启动后，检查集群状态
```
[root@hdss7-12 ~]# /opt/etcd/etcdctl cluster-health
member 988139385f78284 is healthy: got healthy result from http://127.0.0.1:2379
member 5a0ef2a004fc4349 is healthy: got healthy result from http://127.0.0.1:2379
member f4a0cb0a765574a8 is healthy: got healthy result from http://127.0.0.1:2379
cluster is healthy

[root@hdss7-12 ~]# /opt/etcd/etcdctl member list
988139385f78284: name=etcd-server-7-22 peerURLs=https://10.4.7.22:2380 clientURLs=http://127.0.0.1:2379,https://10.4.7.22:2379 isLeader=false
5a0ef2a004fc4349: name=etcd-server-7-21 peerURLs=https://10.4.7.21:2380 clientURLs=http://127.0.0.1:2379,https://10.4.7.21:2379 isLeader=false
f4a0cb0a765574a8: name=etcd-server-7-12 peerURLs=https://10.4.7.12:2380 clientURLs=http://127.0.0.1:2379,https://10.4.7.12:2379 isLeader=true
```

###  部署kube-apiserver集群
#### 集群规划
主机名|角色|ip
-|-|-
HDSS7-21.host.com|kube-apiserver|10.4.7.21
HDSS7-22.host.com|kube-apiserver|10.4.7.22
HDSS7-11.host.com|4层负载均衡|10.4.7.11
HDSS7-12.host.com|4层负载均衡|10.4.7.12
**注意：**这里`10.4.7.11`和`10.4.7.12`使用nginx做4层负载均衡器，用keepalived跑一个vip：10.4.7.10，代理两个kube-apiserver，实现高可用

#### 下载软件，解压，做软连接
这里部署文档以`HDSS7-21.host.com`主机为例，另外一台运算节点安装部署方法类似
```pwd /opt/src
[root@hdss7-21 src]# ls -l|grep kubernetes
-rw-r--r-- 1 root root 417761204 Jan 17 16:46 kubernetes-server-linux-amd64.tar.gz
[root@hdss7-21 src]# tar xf kubernetes-server-linux-amd64.tar.gz -C /opt
[root@hdss7-21 src]# ln -s /opt/kubernetes-v1.13.2-linux-amd64 /opt/kubernetes
[root@hdss7-21 src]# ls -l /opt|grep kubernetes
lrwxrwxrwx 1 root   root         31 Jan 18 10:49 kubernetes -> kubernetes-v1.13.2-linux-amd64/
drwxr-xr-x 4 root   root         50 Jan 17 17:40 kubernetes-v1.13.2-linux-amd64
```
#### 在运维主机，签发etcd client证书
##### 创建生成证书签名请求（csr）的JSON配置文件
```vi /opt/certs/client-csr.json
{
    "CN": "client",
    "hosts": [
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "beijing",
            "L": "beijing",
            "O": "od",
            "OU": "ops"
        }
    ]
}
```
##### 生成etcd-client证书和私钥
```
[root@hdss7-200 certs]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client client-csr.json | cfssljson -bare etcd-client
```
##### 检查生成的证书、私钥
```
[root@hdss7-200 certs]# ls -l|grep etcd-client
-rw------- 1 root root 1679 Jan 21 11:13 etcd-client-key.pem
-rw-r--r-- 1 root root  989 Jan 21 11:13 etcd-client.csr
-rw-r--r-- 1 root root 1367 Jan 21 11:13 etcd-client.pem
```

#### 在运维主机，签发kube-apiserver证书
##### 创建生成证书签名请求（csr）的JSON配置文件
```vi /opt/apiserver-csr.json 
{
    "CN": "apiserver",
    "hosts": [
        "127.0.0.1",
        "192.168.0.1",
        "kubernetes.default",
        "kubernetes.default.svc",
        "kubernetes.default.svc.cluster",
        "kubernetes.default.svc.cluster.local",
        "10.4.7.21",
        "10.4.7.22",
        "10.4.7.23",
        "10.4.7.24"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "beijing",
            "L": "beijing",
            "O": "od",
            "OU": "ops"
        }
    ]
}
```
##### 生成kube-apiserver证书和私钥
```
[root@hdss7-200 certs]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server apiserver-csr.json | cfssljson -bare apiserver 
```
##### 检查生成的证书、私钥
```
[root@hdss7-200 certs]# ls -l|grep apiserver
total 72
-rw-r--r-- 1 root root  406 Jan 21 14:10 apiserver-csr.json
-rw------- 1 root root 1675 Jan 21 14:11 apiserver-key.pem
-rw-r--r-- 1 root root 1082 Jan 21 14:11 apiserver.csr
-rw-r--r-- 1 root root 1599 Jan 21 14:11 apiserver.pem
```

#### 拷贝证书至各运算节点，并创建配置
##### 拷贝证书、私钥，注意私钥文件属性600
```ls /opt/kubernetes/server/bin/cert
[root@hdss7-21 cert]# ls -l /opt/kubernetes/server/bin/cert
total 40
-rw------- 1 root root 1676 Jan 21 16:39 apiserver-key.pem
-rw-r--r-- 1 root root 1599 Jan 21 16:36 apiserver.pem
-rw------- 1 root root 1675 Jan 21 13:55 ca-key.pem
-rw-r--r-- 1 root root 1354 Jan 21 13:50 ca.pem
-rw------- 1 root root 1679 Jan 21 13:53 etcd-client-key.pem
-rw-r--r-- 1 root root 1368 Jan 21 13:53 etcd-client.pem
```
##### 创建配置
```vi /opt/kubernetes/server/bin/conf/audit.yaml
apiVersion: audit.k8s.io/v1beta1 # This is required.
kind: Policy
# Don't generate audit events for all requests in RequestReceived stage.
omitStages:
  - "RequestReceived"
rules:
  # Log pod changes at RequestResponse level
  - level: RequestResponse
    resources:
    - group: ""
      # Resource "pods" doesn't match requests to any subresource of pods,
      # which is consistent with the RBAC policy.
      resources: ["pods"]
  # Log "pods/log", "pods/status" at Metadata level
  - level: Metadata
    resources:
    - group: ""
      resources: ["pods/log", "pods/status"]

  # Don't log requests to a configmap called "controller-leader"
  - level: None
    resources:
    - group: ""
      resources: ["configmaps"]
      resourceNames: ["controller-leader"]

  # Don't log watch requests by the "system:kube-proxy" on endpoints or services
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
    - group: "" # core API group
      resources: ["endpoints", "services"]

  # Don't log authenticated requests to certain non-resource URL paths.
  - level: None
    userGroups: ["system:authenticated"]
    nonResourceURLs:
    - "/api*" # Wildcard matching.
    - "/version"

  # Log the request body of configmap changes in kube-system.
  - level: Request
    resources:
    - group: "" # core API group
      resources: ["configmaps"]
    # This rule only applies to resources in the "kube-system" namespace.
    # The empty string "" can be used to select non-namespaced resources.
    namespaces: ["kube-system"]

  # Log configmap and secret changes in all other namespaces at the Metadata level.
  - level: Metadata
    resources:
    - group: "" # core API group
      resources: ["secrets", "configmaps"]

  # Log all other resources in core and extensions at the Request level.
  - level: Request
    resources:
    - group: "" # core API group
    - group: "extensions" # Version of group should NOT be included.

  # A catch-all rule to log all other requests at the Metadata level.
  - level: Metadata
    # Long-running requests like watches that fall under this rule will not
    # generate an audit event in RequestReceived.
    omitStages:
      - "RequestReceived"
```

#### 创建启动脚本
```vi /opt/kubernetes/server/bin/kube-apiserver.sh
#!/bin/bash
./kube-apiserver \
  --apiserver-count 2 \
  --audit-log-path /data/logs/kubernetes/kube-apiserver/audit-log \
  --audit-policy-file ./conf/audit.yaml \
  --authorization-mode RBAC \
  --client-ca-file ./cert/ca.pem \
  --requestheader-client-ca-file ./cert/ca.pem \
  --enable-admission-plugins NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota \
  --etcd-cafile ./cert/ca.pem \
  --etcd-certfile ./cert/etcd-client.pem \
  --etcd-keyfile ./cert/etcd-client-key.pem \
  --etcd-servers https://10.4.7.12:2379,https://10.4.7.21:2379,https://10.4.7.22:2379 \
  --service-account-key-file ./cert/ca-key.pem \
  --service-cluster-ip-range 192.168.0.0/16 \
  --service-node-port-range 3000-29999 \
  --target-ram-mb=1024 \
  --kubelet-client-certificate ./cert/etcd-client.pem \
  --kubelet-client-key ./cert/etcd-client-key.pem \
  --log-dir  /data/logs/kubernetes/kube-apiserver \
  --tls-cert-file ./cert/apiserver.pem \
  --tls-private-key-file ./cert/apiserver-key.pem \
  --v 2
```

#### 调整权限和目录
```pwd /opt/kubernetes/server/bin
[root@hdss7-21 bin]# chmod +x /opt/kubernetes/server/bin/kube-apiserver.sh
[root@hdss7-21 bin]# mkdir -p /data/logs/kubernetes/kube-apiserver
```
#### 创建supervisor配置
```vi /etc/supervisord.d/kube-apiserver.ini
[program:kube-apiserver]
command=/opt/kubernetes/server/bin/kube-apiserver.sh            ; the program (relative uses PATH, can take args)
numprocs=1                                                      ; number of processes copies to start (def 1)
directory=/opt/kubernetes/server/bin                            ; directory to cwd to before exec (def no cwd)
autostart=true                                                  ; start at supervisord start (default: true)
autorestart=true                                                ; retstart at unexpected quit (default: true)
startsecs=22                                                    ; number of secs prog must stay running (def. 1)
startretries=3                                                  ; max # of serial start failures (default 3)
exitcodes=0,2                                                   ; 'expected' exit codes for process (default 0,2)
stopsignal=QUIT                                                 ; signal used to kill process (default TERM)
stopwaitsecs=10                                                 ; max num secs to wait b4 SIGKILL (default 10)
user=root                                                       ; setuid to this UNIX account to run the program
redirect_stderr=false                                           ; redirect proc stderr to stdout (default false)
stdout_logfile=/data/logs/kubernetes/kube-apiserver/apiserver.stdout.log        ; stdout log path, NONE for none; default AUTO
stdout_logfile_maxbytes=64MB                                    ; max # logfile bytes b4 rotation (default 50MB)
stdout_logfile_backups=4                                        ; # of stdout logfile backups (default 10)
stdout_capture_maxbytes=1MB                                     ; number of bytes in 'capturemode' (default 0)
stdout_events_enabled=false                                     ; emit events on stdout writes (default false)
stderr_logfile=/data/logs/kubernetes/kube-apiserver/apiserver.stderr.log        ; stderr log path, NONE for none; default AUTO
stderr_logfile_maxbytes=64MB                                    ; max # logfile bytes b4 rotation (default 50MB)
stderr_logfile_backups=4                                        ; # of stderr logfile backups (default 10)
stderr_capture_maxbytes=1MB                                     ; number of bytes in 'capturemode' (default 0)
stderr_events_enabled=false                                     ; emit events on stderr writes (default false)
```
#### 启动服务并检查
```
[root@hdss7-21 bin]# supervisorctl update
kube-apiserverr: added process group
[root@hdss7-21 bin]# supervisorctl status
etcd-server-7-21                 RUNNING   pid 6661, uptime 1 day, 8:41:13
kube-apiserver                   RUNNING   pid 43765, uptime 2:09:41
```
#### 安装部署启动检查所有集群规划主机上的kube-apiserver
略

#### 配4层反向代理
##### nginx配置（两台都要配置）
```vi /etc/nginx/nginx.conf
stream {
    upstream kube-apiserver {
        server 10.4.7.21:6443     max_fails=3 fail_timeout=30s;
        server 10.4.7.22:6443     max_fails=3 fail_timeout=30s;
    }
    server {
        listen 7443;
        proxy_connect_timeout 2s;
        proxy_timeout 900s;
        proxy_pass kube-apiserver;
    }
}
```
##### keepalived配置
###### check_port.sh（两台都要创建）
```vi /etc/keepalived/check_port.sh 
#!/bin/bash
#keepalived 监控端口脚本
#使用方法：
#在keepalived的配置文件中
#vrrp_script check_port {#创建一个vrrp_script脚本,检查配置
#    script "/etc/keepalived/check_port.sh 6379" #配置监听的端口
#    interval 2 #检查脚本的频率,单位（秒）
#}
CHK_PORT=$1
echo $CHK_PORT
if [ "$CHK_PORT" != "" ];then

	PORT_PROCESS=`lsof -i:$CHK_PORT|wc -l`
	if [ $PORT_PROCESS -eq 0 ];then
	echo "Port $CHK_PORT Is Not Used,End."
        exit 1	
	fi
else
	echo "Check Port Cant Be Empty!"
fi
```
###### `HDSS7-11.host.com`上
```
[root@hdss7-11 ~]# rpm -qa keepalived
keepalived-1.3.5-6.el7.x86_64
```

```vi /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
   router_id 10.4.7.11

}

vrrp_script chk_nginx {
    script "/etc/keepalived/check_port.sh 80"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state MASTER
    interface vir0
    virtual_router_id 251
    priority 100
    advert_int 1
    mcast_src_ip 10.4.7.11
    nopreempt

    authentication {
        auth_type PASS
        auth_pass 11111111
    }
    track_script {
         chk_nginx
    }
    virtual_ipaddress {
        10.4.7.10
    }
}
```
###### `HDSS7-12.host.com`上
```
[root@hdss7-12 ~]# rpm -qa keepalived
keepalived-1.3.5-6.el7.x86_64
```

```vi /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
	router_id 10.4.7.12
}
vrrp_script chk_nginx {
	script "/etc/keepalived/check_port.sh 80"
	interval 2
	weight -20
}
vrrp_instance VI_1 {
	state BACKUP
	interface vir0
	virtual_router_id 251
	mcast_src_ip 10.4.7.12
	priority 90
	advert_int 1
	authentication {
		auth_type PASS
		auth_pass 11111111
	}
	track_script {
		chk_nginx
	}
	virtual_ipaddress {
		10.4.7.10
	}
}
```

#### 启动代理并检查
启动
```
[root@hdss7-11 ~]# systemctl start keepalived
[root@hdss7-11 ~]# systemctl enable keepalived
[root@hdss7-11 ~]# nginx -s reload

[root@hdss7-12 ~]# systemctl start keepalived
[root@hdss7-12 ~]# systemctl enable keepalived
[root@hdss7-12 ~]# nginx -s reload
```
检查
```
[root@hdss7-11 ~]## netstat -luntp|grep 7443
tcp        0      0 0.0.0.0:7443            0.0.0.0:*               LISTEN      17970/nginx: master
[root@hdss7-12 ~]## netstat -luntp|grep 7443
tcp        0      0 0.0.0.0:7443            0.0.0.0:*               LISTEN      17970/nginx: master
[root@hdss7-11 ~]# ip add|grep 10.4.9.10
    inet 10.9.7.10/32 scope global vir0
[root@hdss7-11 ~]# ip add|grep 10.4.9.10
    （空）
```

### 部署controller-manager
#### 集群规划
主机名|角色|ip
-|-|-
HDSS7-21.host.com|controller-manager|10.4.7.21
HDSS7-22.host.com|controller-manager|10.4.7.22

**注意：**这里部署文档以`HDSS7-21.host.com`主机为例，另外一台运算节点安装部署方法类似

#### 创建启动脚本
```vi /opt/kubernetes/server/bin/kube-controller-manager.sh
#!/bin/sh
./kube-controller-manager \
  --cluster-cidr 172.7.0.0/16 \
  --leader-elect true \
  --log-dir /data/logs/kubernetes/kube-controller-manager \
  --master http://127.0.0.1:8080 \
  --service-account-private-key-file ./cert/ca-key.pem \
  --service-cluster-ip-range 192.168.0.0/16 \
  --root-ca-file ./cert/ca.pem \
  --v 2
```
#### 调整文件权限，创建目录
```pwd /opt/kubernetes/server/bin
[root@hdss7-21 bin]# chmod +x /opt/kubernetes/server/bin/kube-controller-manager.sh
[root@hdss7-21 bin]# mkdir -p /data/logs/kubernetes/kube-controller-manager
```
#### 创建supervisor配置
```vi /etc/supervisord.d/kube-conntroller-manager.ini
[program:kube-controller-manager]
command=/opt/kubernetes/server/bin/kube-controller-manager.sh                     ; the program (relative uses PATH, can take args)
numprocs=1                                                                        ; number of processes copies to start (def 1)
directory=/opt/kubernetes/server/bin                                              ; directory to cwd to before exec (def no cwd)
autostart=true                                                                    ; start at supervisord start (default: true)
autorestart=true                                                                  ; retstart at unexpected quit (default: true)
startsecs=22                                                                      ; number of secs prog must stay running (def. 1)
startretries=3                                                                    ; max # of serial start failures (default 3)
exitcodes=0,2                                                                     ; 'expected' exit codes for process (default 0,2)
stopsignal=QUIT                                                                   ; signal used to kill process (default TERM)
stopwaitsecs=10                                                                   ; max num secs to wait b4 SIGKILL (default 10)
user=root                                                                         ; setuid to this UNIX account to run the program
redirect_stderr=false                                                             ; redirect proc stderr to stdout (default false)
stdout_logfile=/data/logs/kubernetes/kube-controller-manager/controll.stdout.log  ; stdout log path, NONE for none; default AUTO
stdout_logfile_maxbytes=64MB                                                      ; max # logfile bytes b4 rotation (default 50MB)
stdout_logfile_backups=4                                                          ; # of stdout logfile backups (default 10)
stdout_capture_maxbytes=1MB                                                       ; number of bytes in 'capturemode' (default 0)
stdout_events_enabled=false                                                       ; emit events on stdout writes (default false)
stderr_logfile=/data/logs/kubernetes/kube-controller-manager/controll.stderr.log  ; stderr log path, NONE for none; default AUTO
stderr_logfile_maxbytes=64MB                                                      ; max # logfile bytes b4 rotation (default 50MB)
stderr_logfile_backups=4                                                          ; # of stderr logfile backups (default 10)
stderr_capture_maxbytes=1MB                                                       ; number of bytes in 'capturemode' (default 0)
stderr_events_enabled=false                                                       ; emit events on stderr writes (default false)
```
#### 启动服务并检查
```
[root@hdss7-21 bin]# supervisorctl update
kube-controller-manager: added process group
[root@hdss7-21 bin]# supervisorctl status   
etcd-server-7-21                 RUNNING   pid 6661, uptime 1 day, 8:41:13
kube-apiserver                   RUNNING   pid 43765, uptime 2:09:41
kube-controller-manager          RUNNING   pid 44230, uptime 2:05:01
```

#### 安装部署启动检查所有集群规划主机上的kube-controller-manager服务
略

### 部署kube-scheduler
#### 集群规划
主机名|角色|ip
-|-|-
HDSS7-21.host.com|kube-scheduler|10.4.7.21
HDSS7-22.host.com|kube-scheduler|10.4.7.22

**注意：**这里部署文档以`HDSS7-21.host.com`主机为例，另外一台运算节点安装部署方法类似

#### 创建启动脚本
```vi /opt/kubernetes/server/bin/kube-scheduler.sh
#!/bin/sh
./kube-scheduler \
  --leader-elect  \
  --log-dir /data/logs/kubernetes/kube-scheduler \
  --master http://127.0.0.1:8080 \
  --v 2
```

#### 调整文件权限，创建目录
```pwd /opt/kubernetes/server/bin
[root@hdss7-21 bin]# chmod +x /opt/kubernetes/server/bin/kube-scheduler.sh
[root@hdss7-21 bin]# mkdir -p /data/logs/kubernetes/kube-scheduler
```

#### 创建supervisor配置
```vi /etc/supervisord.d/kube-scheduler.ini
[program:kube-scheduler]
command=/opt/kubernetes/server/bin/kube-scheduler.sh                     ; the program (relative uses PATH, can take args)
numprocs=1                                                               ; number of processes copies to start (def 1)
directory=/opt/kubernetes/server/bin                                     ; directory to cwd to before exec (def no cwd)
autostart=true                                                           ; start at supervisord start (default: true)
autorestart=true                                                         ; retstart at unexpected quit (default: true)
startsecs=22                                                             ; number of secs prog must stay running (def. 1)
startretries=3                                                           ; max # of serial start failures (default 3)
exitcodes=0,2                                                            ; 'expected' exit codes for process (default 0,2)
stopsignal=QUIT                                                          ; signal used to kill process (default TERM)
stopwaitsecs=10                                                          ; max num secs to wait b4 SIGKILL (default 10)
user=root                                                                ; setuid to this UNIX account to run the program
redirect_stderr=false                                                    ; redirect proc stderr to stdout (default false)
stdout_logfile=/data/logs/kubernetes/kube-scheduler/scheduler.stdout.log ; stdout log path, NONE for none; default AUTO
stdout_logfile_maxbytes=64MB                                             ; max # logfile bytes b4 rotation (default 50MB)
stdout_logfile_backups=4                                                 ; # of stdout logfile backups (default 10)
stdout_capture_maxbytes=1MB                                              ; number of bytes in 'capturemode' (default 0)
stdout_events_enabled=false                                              ; emit events on stdout writes (default false)
stderr_logfile=/data/logs/kubernetes/kube-scheduler/scheduler.stderr.log ; stderr log path, NONE for none; default AUTO
stderr_logfile_maxbytes=64MB                                             ; max # logfile bytes b4 rotation (default 50MB)
stderr_logfile_backups=4                                                 ; # of stderr logfile backups (default 10)
stderr_capture_maxbytes=1MB                                              ; number of bytes in 'capturemode' (default 0)
stderr_events_enabled=false                                              ; emit events on stderr writes (default false)
```

#### 启动服务并检查
```
[root@hdss7-21 bin]# supervisorctl update
kube-scheduler: added process group
[root@hdss7-21 bin]# supervisorctl status
etcd-server-7-21                 RUNNING   pid 6661, uptime 1 day, 8:41:13
kube-apiserver                   RUNNING   pid 43765, uptime 2:09:41
kube-controller-manager          RUNNING   pid 44230, uptime 2:05:01
kube-scheduler                   RUNNING   pid 44779, uptime 2:02:27
```

#### 安装部署启动检查所有集群规划主机上的kube-scheduler服务
略

### 部署kubelet
#### 集群规划
主机名|角色|ip
-|-|-
HDSS7-21.host.com|kubelet|10.4.7.21
HDSS7-22.host.com|kubelet|10.4.7.22

**注意：**这里部署文档以`HDSS7-21.host.com`主机为例，另外一台运算节点安装部署方法类似

#### 在运维主机，签发kubelet证书
##### 创建生成证书签名请求（csr）的JSON配置文件
```vi kubelet-peer-csr.json
{
    "CN": "kubelet-node",
    "hosts": [
		"127.0.0.1",
		"10.4.7.10",
		"10.4.7.21",
		"10.4.7.22",
		"10.4.7.23",
		"10.4.7.24"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "beijing",
            "L": "beijing",
            "O": "od",
            "OU": "ops"
        }
    ]
}
```
##### 生成kubelet证书和私钥
```pwd /opt/certs
[root@hdss7-200 certs]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer kubelet-peer-csr.json | cfssljson -bare kubelet-peer
```
##### 检查生成的证书、私钥
```pwd /opt/certs
[root@hdss7-200 certs]# ls -l|grep kubelet
total 88
-rw-r--r-- 1 root root  415 Jan 22 16:58 kubelet-peer-csr.json
-rw------- 1 root root 1679 Jan 22 17:00 kubelet-peer-key.pem
-rw-r--r-- 1 root root 1086 Jan 22 17:00 kubelet-peer.csr
-rw-r--r-- 1 root root 1456 Jan 22 17:00 kubelet-peer.pem
```

#### 拷贝证书至各运算节点，并创建配置
##### 拷贝证书、私钥，注意私钥文件属性600
```ls /opt/kubernetes/server/bin/cert
[root@hdss7-21 cert]# ls -l /opt/kubernetes/server/bin/cert
total 40
-rw------- 1 root root 1676 Jan 21 16:39 apiserver-key.pem
-rw-r--r-- 1 root root 1599 Jan 21 16:36 apiserver.pem
-rw------- 1 root root 1675 Jan 21 13:55 ca-key.pem
-rw-r--r-- 1 root root 1354 Jan 21 13:50 ca.pem
-rw------- 1 root root 1679 Jan 21 13:53 etcd-client-key.pem
-rw-r--r-- 1 root root 1368 Jan 21 13:53 etcd-client.pem
-rw------- 1 root root 1679 Jan 22 17:00 kubelet-peer-key.pem
-rw-r--r-- 1 root root 1456 Jan 22 17:00 kubelet-peer.pem
```
##### 创建配置
###### 给kubectl创建软连接
```pwd /opt/kubernetes/server/bin
[root@hdss7-21 bin]# ln -s /opt/kubernetes/server/bin/kubectl /usr/bin/kubectl
[root@hdss7-21 bin]# which kubectl
/usr/bin/kubectl
```
###### set-cluster
**注意：**在conf目录下
```pwd /opt/kubernetes/server/conf
[root@hdss7-21 conf]# kubectl config set-cluster myk8s \
  --certificate-authority=/opt/kubernetes/server/bin/cert/ca.pem \
  --embed-certs=true \
  --server=https://10.4.7.10:7443 \
  --kubeconfig=kubelet.kubeconfig
```

###### set-credentials
**注意：**在conf目录下
```pwd /opt/kubernetes/server/conf
[root@hdss7-21 conf]# kubectl config set-credentials kubelet-node --client-certificate=/opt/kubernetes/server/bin/cert/kubelet-peer.pem --client-key=/opt/kubernetes/server/bin/cert/kubelet-peer-key.pem --embed-certs=true --kubeconfig=kubelet.kubeconfig 

User "kubelet-node" set.
```

###### set-context
**注意：**在conf目录下
```pwd /opt/kubernetes/server/conf
[root@hdss7-21 conf]# kubectl config set-context myk8s-context \
  --cluster=myk8s \
  --user=kubelet-node \
  --kubeconfig=kubelet.kubeconfig
```

###### use-context
**注意：**在conf目录下
```pwd /opt/kubernetes/server/conf
[root@hdss7-21 conf]# kubectl config use-context myk8s-context --kubeconfig=kubelet.kubeconfig
```
###### kube-node.yaml
创建资源配置文件
```vi /opt/kubernetes/server/conf/kube-node.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubelet-node
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:node
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: kubelet-node
```
应用资源配置文件
```pwd /opt/kubernetes/server/conf
[root@hdss7-21 conf]# kubectl create -f kube-node.yaml
```
检查
```pwd /opt/kubernetes/server/conf
[root@hdss7-21 conf]# kubectl get clusterrolebinding kubelet-node      
NAME           AGE
kubelet-node   3m
```
##### 准备infra_pod基础镜像
###### 下载
```
[root@hdss7-21 conf]# docker pull xplenty/rhel7-pod-infrastructure:v3.4
Trying to pull repository docker.io/xplenty/rhel7-pod-infrastructure ... 
sha256:9314554780673b821cb7113d8c048a90d15077c6e7bfeebddb92a054a1f84843: Pulling from docker.io/xplenty/rhel7-pod-infrastructure
615bc035f9f8: Pull complete 
1c5fd9dfeaa8: Pull complete 
7653a8c7f937: Pull complete 
Digest: sha256:9314554780673b821cb7113d8c048a90d15077c6e7bfeebddb92a054a1f84843
Status: Downloaded newer image for docker.io/xplenty/rhel7-pod-infrastructure:v3.4
```
###### 提交至私有仓库（harbor）中
- 配置主机登录私有仓库

```vi /root/.docker/config.json
{
	"auths": {
		"harbor.od.com": {
			"auth": "YWRtaW46SGFyYm9yMTIzNDU="
		}
	}
}
```
> 这里代表：用户名admin，密码Harbor12345
> [root@hdss7-21 ~]# echo YWRtaW46SGFyYm9yMTIzNDU=|base64 -d
> admin:Harbor12345

**注意：**也可以在各运算节点使用docker login harbor.od.com，输入用户名，密码

- 给镜像打tag

```
[root@hdss7-21 conf]# docker images|grep v3.4
xplenty/rhel7-pod-infrastructure   v3.4                34d3450d733b        2 years ago         205 MB
[root@hdss7-21 conf]# docker tag 34d3450d733b harbor.od.com/k8s/pod:v3.4
```

- push到harbor

```
[root@hdss7-21 conf]# docker push harbor.od.com/k8s/pod:v3.4
```

##### 创建kubelet启动脚本
```vi /opt/kubernetes/server/bin/kubelet-721.sh
#!/bin/sh
./kubelet \
  --anonymous-auth=false \
  --cgroup-driver systemd \
  --cluster-dns 192.168.0.2 \
  --cluster-domain cluster.local \
  --runtime-cgroups=/systemd/system.slice --kubelet-cgroups=/systemd/system.slice \
  --fail-swap-on="false" \
  --client-ca-file ./cert/ca.pem \
  --tls-cert-file ./cert/kubelet-peer.pem \
  --tls-private-key-file ./cert/kubelet-peer-key.pem \
  --hostname-override 10.4.7.21 \
  --image-gc-high-threshold 20 \
  --image-gc-low-threshold 10 \
  --kubeconfig ./conf/kubelet.kubeconfig \
  --log-dir /data/logs/kubernetes/kubelet \
  --pod-infra-container-image harbor.od.com/k8s/pod:v3.4 \
  --root-dir /data/kubelet
```
**注意：**kubelet集群各主机的启动脚本略有不同，部署其他节点时注意修改。

##### 检查配置，权限，创建日志目录
```pwd /opt/kubernetes/server/conf
[root@hdss7-21 conf]# ls -l|grep kubelet.kubeconfig 
-rw------- 1 root root 6471 Jan 22 17:33 kubelet.kubeconfig

[root@hdss7-21 conf]# chmod +x /opt/kubernetes/server/bin/kubelet.sh
[root@hdss7-21 conf]# mkdir -p /data/logs/kubernetes/kubelet /data/kubelet
```

#### 创建supervisor配置
```vi /etc/supervisord.d/kube-kubelet.ini
[program:kube-kubelet]
command=/opt/kubernetes/server/bin/kubelet-721.sh                 ; the program (relative uses PATH, can take args)
numprocs=1                                                        ; number of processes copies to start (def 1)
directory=/opt/kubernetes/server/bin                              ; directory to cwd to before exec (def no cwd)
autostart=true                                                    ; start at supervisord start (default: true)
autorestart=true              									  ; retstart at unexpected quit (default: true)
startsecs=22                  									  ; number of secs prog must stay running (def. 1)
startretries=3                									  ; max # of serial start failures (default 3)
exitcodes=0,2                 									  ; 'expected' exit codes for process (default 0,2)
stopsignal=QUIT               									  ; signal used to kill process (default TERM)
stopwaitsecs=10               									  ; max num secs to wait b4 SIGKILL (default 10)
user=root                                                         ; setuid to this UNIX account to run the program
redirect_stderr=false                                             ; redirect proc stderr to stdout (default false)
stdout_logfile=/data/logs/kubernetes/kubelet/kubelet.stdout.log   ; stdout log path, NONE for none; default AUTO
stdout_logfile_maxbytes=64MB                                      ; max # logfile bytes b4 rotation (default 50MB)
stdout_logfile_backups=4                                          ; # of stdout logfile backups (default 10)
stdout_capture_maxbytes=1MB                                       ; number of bytes in 'capturemode' (default 0)
stdout_events_enabled=false                                       ; emit events on stdout writes (default false)
stderr_logfile=/data/logs/kubernetes/kubelet/kubelet.stderr.log   ; stderr log path, NONE for none; default AUTO
stderr_logfile_maxbytes=64MB                                      ; max # logfile bytes b4 rotation (default 50MB)
stderr_logfile_backups=4                                          ; # of stderr logfile backups (default 10)
stderr_capture_maxbytes=1MB   									  ; number of bytes in 'capturemode' (default 0)
stderr_events_enabled=false   									  ; emit events on stderr writes (default false)
```

#### 启动服务并检查
```
[root@hdss7-21 bin]# supervisorctl update
kube-kubelet: added process group
[root@hdss7-21 bin]# supervisorctl status
etcd-server-7-21                 RUNNING   pid 9507, uptime 22:44:48
kube-apiserver                   RUNNING   pid 9770, uptime 21:10:49
kube-controller-manager          RUNNING   pid 10048, uptime 18:22:10
kube-kubelet                     STARTING  
kube-scheduler                   RUNNING   pid 10041, uptime 18:22:13
```

#### 检查运算节点
```
[root@hdss7-21 bin]# kubectl get node
NAME        STATUS   ROLES    AGE   VERSION
10.4.7.21   Ready    <none>   3m   v1.13.2
```

#### 安装部署启动检查所有集群规划主机上的kubelet服务
略

### 部署kube-proxy
#### 集群规划
主机名|角色|ip
-|-|-
HDSS7-21.host.com|kube-proxy|10.4.7.21
HDSS7-22.host.com|kube-proxy|10.4.7.22

**注意：**这里部署文档以`HDSS7-21.host.com`主机为例，另外一台运算节点安装部署方法类似

#### 在运维主机，签发kube-proxy证书
##### 创建生成证书签名请求（csr）的JSON配置文件
```vi /opt/certs/kube-proxy-csr.json
{
    "CN": "system:kube-proxy",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "beijing",
            "L": "beijing",
            "O": "od",
            "OU": "ops"
        }
    ]
}
```
##### 生成kube-proxy证书和私钥
```pwd /opt/certs
[root@hdss7-200 certs]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client kube-proxy-csr.json | cfssljson -bare kube-proxy-client
```
##### 检查生成的证书、私钥
```pwd /opt/certs
[root@hdss7-200 certs]# ls -l|grep kube-proxy
-rw------- 1 root root 1679 Jan 22 17:31 kube-proxy-client-key.pem
-rw-r--r-- 1 root root 1005 Jan 22 17:31 kube-proxy-client.csr
-rw-r--r-- 1 root root 1383 Jan 22 17:31 kube-proxy-client.pem
-rw-r--r-- 1 root root  268 Jan 22 17:23 kube-proxy-csr.json
```
#### 拷贝证书至各运算节点，并创建配置
##### 拷贝证书、私钥，注意私钥文件属性600
```ls /opt/kubernetes/server/bin/cert
[root@hdss7-21 cert]# ls -l /opt/kubernetes/server/bin/cert
total 40
-rw------- 1 root root 1676 Jan 21 16:39 apiserver-key.pem
-rw-r--r-- 1 root root 1599 Jan 21 16:36 apiserver.pem
-rw------- 1 root root 1675 Jan 21 13:55 ca-key.pem
-rw-r--r-- 1 root root 1354 Jan 21 13:50 ca.pem
-rw------- 1 root root 1679 Jan 21 13:53 etcd-client-key.pem
-rw-r--r-- 1 root root 1368 Jan 21 13:53 etcd-client.pem
-rw------- 1 root root 1679 Jan 22 17:00 kubelet-peer-key.pem
-rw-r--r-- 1 root root 1456 Jan 22 17:00 kubelet-peer.pem
-rw------- 1 root root 1679 Jan 22 17:31 kube-proxy-client-key.pem
-rw-r--r-- 1 root root 1383 Jan 22 17:31 kube-proxy-client.pem
```

##### 创建配置
###### set-cluster
**注意：**在conf目录下
```pwd /opt/kubernetes/server/bin/conf
[root@hdss7-21 conf]# kubectl config set-cluster myk8s \
  --certificate-authority=/opt/kubernetes/server/bin/cert/ca.pem \
  --embed-certs=true \
  --server=https://10.4.7.10:7443 \
  --kubeconfig=kube-proxy.kubeconfig
```
###### set-credentials
**注意：**在conf目录下
```pwd /opt/kubernetes/server/bin/conf
[root@hdss7-21 conf]# kubectl config set-credentials kube-proxy \
  --client-certificate=/opt/kubernetes/server/bin/cert/kube-proxy-client.pem \
  --client-key=/opt/kubernetes/server/bin/cert/kube-proxy-client-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
```
###### set-context
**注意：**在conf目录下
```pwd /opt/kubernetes/server/bin/conf
[root@hdss7-21 conf]# kubectl config set-context myk8s-context \
  --cluster=myk8s \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
```
###### use-context
**注意：**在conf目录下
```pwd /opt/kubernetes/server/bin/conf
[root@hdss7-21 conf]# kubectl config use-context myk8s-context --kubeconfig=kube-proxy.kubeconfig
```
##### 创建kube-proxy启动脚本
```vi /opt/kubernetes/server/bin/kube-proxy-721.sh
#!/bin/sh
./kube-proxy \
  --cluster-cidr 172.7.0.0/16 \
  --hostname-override 10.4.7.21 \
  --kubeconfig ./conf/kube-proxy.kubeconfig
```
**注意：**kube-proxy集群各主机的启动脚本略有不同，部署其他节点时注意修改。

##### 检查配置，权限，创建日志目录
```pwd /opt/kubernetes/server/conf
[root@hdss7-21 conf]# ls -l|grep kube-proxy.kubeconfig 
-rw------- 1 root root 6471 Jan 22 17:33 kube-proxy.kubeconfig

[root@hdss7-21 conf]# chmod +x /opt/kubernetes/server/bin/kube-proxy.sh
[root@hdss7-21 conf]# mkdir -p /data/logs/kubernetes/kube-proxy
```
#### 创建supervisor配置
```vi /etc/supervisord.d/kube-proxy.ini
[program:kube-proxy]
command=/opt/kubernetes/server/bin/kube-proxy-721.sh                     ; the program (relative uses PATH, can take args)
numprocs=1                                                               ; number of processes copies to start (def 1)
directory=/opt/kubernetes/server/bin                                     ; directory to cwd to before exec (def no cwd)
autostart=true                                                           ; start at supervisord start (default: true)
autorestart=true                                                         ; retstart at unexpected quit (default: true)
startsecs=22                                                             ; number of secs prog must stay running (def. 1)
startretries=3                                                           ; max # of serial start failures (default 3)
exitcodes=0,2                                                            ; 'expected' exit codes for process (default 0,2)
stopsignal=QUIT                                                          ; signal used to kill process (default TERM)
stopwaitsecs=10                                                          ; max num secs to wait b4 SIGKILL (default 10)
user=root                                                		         ; setuid to this UNIX account to run the program
redirect_stderr=false                                           		 ; redirect proc stderr to stdout (default false)
stdout_logfile=/data/logs/kubernetes/kube-proxy/proxy.stdout.log         ; stdout log path, NONE for none; default AUTO
stdout_logfile_maxbytes=64MB                                    		 ; max # logfile bytes b4 rotation (default 50MB)
stdout_logfile_backups=4                                        		 ; # of stdout logfile backups (default 10)
stdout_capture_maxbytes=1MB                                     		 ; number of bytes in 'capturemode' (default 0)
stdout_events_enabled=false                                     		 ; emit events on stdout writes (default false)
stderr_logfile=/data/logs/kubernetes/kube-proxy/proxy.stderr.log         ; stderr log path, NONE for none; default AUTO
stderr_logfile_maxbytes=64MB                                    		 ; max # logfile bytes b4 rotation (default 50MB)
stderr_logfile_backups=4                                        		 ; # of stderr logfile backups (default 10)
stderr_capture_maxbytes=1MB   						                     ; number of bytes in 'capturemode' (default 0)
stderr_events_enabled=false   						                     ; emit events on stderr writes (default false)
```
#### 启动服务并检查
```
[root@hdss7-21 bin]# supervisorctl update
kube-proxy: added process group
[root@hdss7-21 bin]# supervisorctl status
etcd-server-7-21                 RUNNING   pid 9507, uptime 22:44:48
kube-apiserver                   RUNNING   pid 9770, uptime 21:10:49
kube-controller-manager          RUNNING   pid 10048, uptime 18:22:10
kube-kubelet                     RUNNING   pid 14597, uptime 0:32:59
kube-proxy                       STARTING  
kube-scheduler                   RUNNING   pid 10041, uptime 18:22:13
```
#### 安装部署启动检查所有集群规划主机上的kube-proxy服务
略

### 部署flannel
#### 集群规划
主机名|角色|ip
-|-|-
HDSS7-21.host.com|flannel|10.4.7.21
HDSS7-22.host.com|flannel|10.4.7.22

**注意：**这里部署文档以`HDSS7-21.host.com`主机为例，另外一台运算节点安装部署方法类似

#### 在各运算节点上增加iptables规则
**注意：**iptables规则各主机的略有不同，其他运算节点上执行时注意修改。
优化SNAT规则，各运算节点之间的各POD之间的网络通信不再出网
```
# iptables -t nat -D POSTROUTING -s 172.7.21.0/24 ! -o docker0 -j MASQUERADE
# iptables -t nat -I POSTROUTING -s 172.7.21.0/24 ! -d 172.7.0.0/16 ! -o docker0 -j MASQUERADE
```
> 10.4.7.21主机上的，来源是172.7.21.0/24段的docker的ip，目标ip不是172.7.0.0/16段，网络发包不从docker0桥设备出站的，进行SNAT转换

#### 各运算节点保存iptables规则
```
[root@hdss7-21 ~]# iptables-save > /etc/sysconfig/iptables
```
#### 下载软件，解压，做软连接
```pwd /opt/src
[root@hdss7-21 src]# ls -l|grep flannel
-rw-r--r-- 1 root root 417761204 Jan 17 18:46 flannel-v0.10.0-linux-amd64.tar.gz
[root@hdss7-21 src]# tar xf flannel-v0.10.0-linux-amd64.tar.gz -C /opt
[root@hdss7-21 src]# ln -s /opt/flannel-v0.10.0-linux-amd64.tar.gz /opt/flannel
[root@hdss7-21 src]# ls -l /opt|grep flannel
lrwxrwxrwx 1 root   root         31 Jan 17 18:49 flannel -> flannel-v0.10.0-linux-amd64/
drwxr-xr-x 4 root   root         50 Jan 17 18:47 flannel-v0.10.0-linux-amd64
```
#### 最终目录结构
```pwd /opt
[root@hdss7-21 opt]# tree -L 2
.
|-- etcd -> etcd-v3.1.18-linux-amd64
|-- etcd-v3.1.18-linux-amd64
|   |-- Documentation
|   |-- README-etcdctl.md
|   |-- README.md
|   |-- READMEv2-etcdctl.md
|   |-- certs
|   |-- etcd
|   |-- etcd-server-startup.sh
|   `-- etcdctl
|-- flannel -> flannel-v0.10.0/
|-- flannel-v0.10.0
|   |-- README.md
|   |-- cert
|   |-- flanneld
|   `-- mk-docker-opts.sh
|-- kubernetes -> kubernetes-v1.13.2-linux-amd64/
|-- kubernetes-v1.13.2-linux-amd64
|   |-- LICENSES
|   |-- addons
|   `-- server
`-- src
    |-- etcd-v3.1.18-linux-amd64.tar.gz
    |-- flannel-v0.10.0-linux-amd64.tar.gz
    `-- kubernetes-server-linux-amd64.tar.gz
```
#### 操作etcd，增加host-gw
```pwd /opt/etcd
[root@hdss7-21 etcd]# ./etcdctl set /coreos.com/network/config '{"Network": "172.7.0.0/16", "Backend": {"Type": "host-gw"}}'
{"Network": "172.7.0.0/16", "Backend": {"Type": "host-gw"}}
```

#### 创建配置
```vi /opt/flannel/subnet.env
FLANNEL_NETWORK=172.7.0.0/16
FLANNEL_SUBNET=172.7.21.1/24
FLANNEL_MTU=1500
FLANNEL_IPMASQ=false
```
**注意：**flannel集群各主机的配置略有不同，部署其他节点时注意修改。

#### 创建启动脚本
```vi /opt/flannelflanneld.sh
#!/bin/sh
./flanneld \
  --public-ip=10.4.7.21 \
  --etcd-endpoints=https://10.4.7.12:2379,https://10.4.7.21:2379,https://10.4.7.22:2379 \
  --etcd-keyfile=./cert/etcd-client-key.pem \
  --etcd-certfile=./cert/etcd-client.pem \
  --etcd-cafile=./cert/ca.pem \
  --iface=eth0 \
  --subnet-file=./subnet.env \
  --healthz-port=2401 
```
**注意：**flannel集群各主机的启动脚本略有不同，部署其他节点时注意修改。

#### 检查配置，权限，创建日志目录
```pwd /opt/flannel
[root@hdss7-21 flannel]# chmod +x /opt/flannel/flanneld.sh 

[root@hdss7-21 flannel]# mkdir -p /data/logs/flanneld
```

#### 创建supervisor配置
```vi /etc/supervisord.d/flanneld.ini
[program:flanneld]
command=/opt/flannel/flanneld.sh                             ; the program (relative uses PATH, can take args)
numprocs=1                                                   ; number of processes copies to start (def 1)
directory=/opt/flannel                                       ; directory to cwd to before exec (def no cwd)
autostart=true                                               ; start at supervisord start (default: true)
autorestart=true                                             ; retstart at unexpected quit (default: true)
startsecs=22       										     ; number of secs prog must stay running (def. 1)
startretries=3     										     ; max # of serial start failures (default 3)
exitcodes=0,2      										     ; 'expected' exit codes for process (default 0,2)
stopsignal=QUIT    										     ; signal used to kill process (default TERM)
stopwaitsecs=10    										     ; max num secs to wait b4 SIGKILL (default 10)
user=root                                                    ; setuid to this UNIX account to run the program
redirect_stderr=false                                        ; redirect proc stderr to stdout (default false)
stdout_logfile=/data/logs/flanneld/flanneld.stdout.log       ; stdout log path, NONE for none; default AUTO
stdout_logfile_maxbytes=64MB                                 ; max # logfile bytes b4 rotation (default 50MB)
stdout_logfile_backups=4                                     ; # of stdout logfile backups (default 10)
stdout_capture_maxbytes=1MB                                  ; number of bytes in 'capturemode' (default 0)
stdout_events_enabled=false                                  ; emit events on stdout writes (default false)
stderr_logfile=/data/logs/flanneld/flanneld.stderr.log       ; stderr log path, NONE for none; default AUTO
stderr_logfile_maxbytes=64MB                                 ; max # logfile bytes b4 rotation (default 50MB)
stderr_logfile_backups=4                                     ; # of stderr logfile backups (default 10)
stderr_capture_maxbytes=1MB                                  ; number of bytes in 'capturemode' (default 0)
stderr_events_enabled=false                                  ; emit events on stderr writes (default false)
```

#### 启动服务并检查
```
[root@hdss7-21 flanneld]# supervisorctl update
flanneld: added process group
[root@hdss7-21 flanneld]# supervisorctl status
etcd-server-7-21                 RUNNING   pid 9507, uptime 1 day, 20:35:42
flanneld                         STARTING  
kube-apiserver                   RUNNING   pid 9770, uptime 1 day, 19:01:43
kube-controller-manager          RUNNING   pid 37646, uptime 0:58:48
kube-kubelet                     RUNNING   pid 32640, uptime 17:16:36
kube-proxy                       RUNNING   pid 15097, uptime 17:55:36
kube-scheduler                   RUNNING   pid 37803, uptime 0:55:47
```
#### 安装部署启动检查所有集群规划主机上的flannel服务
略

### 验证kubernetes集群
#### 在任意一个运算节点，创建一个资源配置清单
这里我们选择`HDSS7-21.host.com`主机
```vi /root/nginx-ds.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-ds
  labels:
    app: nginx-ds
spec:
  type: NodePort
  selector:
    app: nginx-ds
  ports:
  - name: http
    port: 80
    targetPort: 80

---

apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: nginx-ds
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  template:
    metadata:
      labels:
        app: nginx-ds
    spec:
      containers:
      - name: my-nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```
#### 应用资源配置，并检查
```pwd /root
[root@hdss7-21 ~]# kubectl create -f nginx-ds.yaml
[root@hdss7-21 ~]# kubectl get pods
NAME             READY   STATUS    RESTARTS   AGE
nginx-ds-6hnc7   1/1     Running   0          99m
nginx-ds-m5q6j   1/1     Running   0          18h
```

#### 验证
```
[root@hdss7-21 ~]# docker exec -ti 63c0 bash
root@nginx-ds-m5q6j:/# ping 172.7.22.2
PING 172.7.22.2 (172.7.22.2): 48 data bytes
56 bytes from 172.7.22.2: icmp_seq=0 ttl=62 time=24.438 ms
56 bytes from 172.7.22.2: icmp_seq=1 ttl=62 time=0.695 ms
56 bytes from 172.7.22.2: icmp_seq=2 ttl=62 time=0.325 ms
^C--- 172.7.22.2 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.325/8.486/24.438/11.281 ms

[root@hdss7-22 ~]# docker exec -ti 1964 bash
root@nginx-ds-6hnc7:/# ping 172.7.21.2
PING 172.7.21.2 (172.7.21.2): 48 data bytes
56 bytes from 172.7.21.2: icmp_seq=0 ttl=62 time=455.370 ms
56 bytes from 172.7.21.2: icmp_seq=1 ttl=62 time=0.395 ms
56 bytes from 172.7.21.2: icmp_seq=2 ttl=62 time=0.514 ms
^C--- 172.7.21.2 ping statistics ---
13 packets transmitted, 13 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.309/61.144/455.370/136.546 ms

[root@hdss7-21 ~]# kubectl get nodes
NAME        STATUS   ROLES    AGE    VERSION
10.4.7.21   Ready    <none>   21h    v1.13.2
10.4.7.22   Ready    <none>   101m   v1.13.2
[root@hdss7-21 ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)       AGE
kubernetes   ClusterIP   192.168.0.1     <none>        443/TCP       45h
nginx-ds     NodePort    192.168.62.22   <none>        80:5801/TCP   18h
```

### 部署k8s资源配置清单的内网http服务
#### 在运维主机`HDSS7-200.host.com`上，配置一个nginx虚拟主机，用以提供k8s统一的资源配置清单访问入口
```vi /etc/nginx/conf.d/k8s-yaml.od.com.conf
server {
    listen       80;
    server_name  k8s-yaml.od.com;

    location / {
        autoindex on;
        default_type text/plain;
        root /data/k8s-yaml;
    }
}
```
#### 配置内网DNS解析
`HDSS7-11.host.com`上
```vi /var/named/od.com.zone
k8s-yaml	60 IN A 10.4.7.200
```
以后所有的资源配置清单统一放置在运维主机的`/data/k8s-yaml`目录下即可
```
[root@hdss7-200 ~]# nginx -s reload
```

### 部署kube-dns(coredns)
#### 运维主机上准备coredns-v1.3.1镜像
```
[root@hdss7-200 ~]# docker pull coredns/coredns:1.3.1
[root@hdss7-200 ~]# docker tag eb516548c180 harbor.od.com/k8s/coredns:1.3.1
[root@hdss7-200 ~]# docker push harbor.od.com/k8s/coredns:1.3.1
```
#### 运维主机上准备资源配置清单
```
[root@hdss7-200 ~]# mkdir -p /data/k8s-yaml/coredns && cd /data/k8s-yaml/coredns
```

{% tabs coredns%}
<!-- tab RoleBinding -->
vi /data/k8s-yaml/coredns/rolebinding.yaml
{% code %}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
  labels:
      kubernetes.io/cluster-service: "true"
      addonmanager.kubernetes.io/mode: Reconcile
\--\-
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: Reconcile
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
\--\-
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: EnsureExists
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
{% endcode %}
<!-- endtab -->
<!-- tab ConfigMap-->
vi /data/k8s-yaml/coredns/configmap.yaml
{% code %}
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        log
        health
        kubernetes cluster.local 192.168.0.0/16
        proxy . /etc/resolv.conf
        cache 30
       }
{% endcode %}
<!-- endtab -->
<!-- tab Deployment-->
vi /data/k8s-yaml/coredns/deployment.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "CoreDNS"
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: coredns
  template:
    metadata:
      labels:
        k8s-app: coredns
    spec:
      serviceAccountName: coredns
      containers:
      - name: coredns
        image: harbor.od.com/k8s/coredns:1.3.1
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
{% endcode %}
<!-- endtab -->
<!-- tab Service-->
vi /data/k8s-yaml/coredns/svc.yaml
{% code %}
apiVersion: v1
kind: Service
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: coredns
  clusterIP: 192.168.0.2
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
{% endcode %}
<!-- endtab -->
{% endtabs %}

#### 依次执行创建
浏览器打开：http://k8s-yaml.od.com/coredns 检查资源配置清单文件是否正确创建
在任意运算节点应用资源配置清单
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/coredns/rolebinding.yaml
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/coredns/configmap.yaml
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/coredns/deployment.yaml
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/coredns/sva.yaml
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/coredns/svc.yaml
```
#### 检查
```
[root@hdss7-21 ~]# kubectl get pods -n kube-system
NAME                       READY   STATUS    RESTARTS   AGE
coredns-7ccccdf57c-5b9ch   1/1     Running   0          3m4s
[root@hdss7-21 coredns]# kubectl get svc -n kube-system
NAME      TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
coredns   ClusterIP   192.168.0.2   <none>        53/UDP,53/TCP   29s
```

### 部署taefik（ingress）
#### 运维主机上准备traefik镜像
```
[root@hdss7-200 ~]# docker pull traefik:v1.7-alpine
[root@hdss7-200 ~]# docker tag a8958b0d0447 harbor.od.com/k8s/traefik:1.7       
[root@hdss7-200 ~]# docker push harbor.od.com/k8s/traefik:1.7
```
#### 运维主机上准备资源配置清单
```
[root@hdss7-200 ~]# mkdir -p /data/k8s-yaml/traefik && cd /data/k8s-yaml/traefik
```
{% tabs traefik%}
<!-- tab Rolebinding -->
vi /data/k8s-yaml/traefik/rolebinding.yaml
{% code %}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
\--\-
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
\--\-
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: kube-system
{% endcode %}
<!-- endtab -->
<!-- tab DaemonSet-->
vi /data/k8s-yaml/traefik/daemonset.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      containers:
      - image: harbor.od.com/k8s/traefik:1.7
        name: traefik-ingress-lb
        ports:
        - name: http
          containerPort: 80
          hostPort: 81
        - name: admin
          containerPort: 8080
        securityContext:
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
        args:
        - --api
        - --kubernetes
        - --logLevel=INFO
		- --insecureskipverify=true
{% endcode %}
<!-- endtab -->
<!-- tab Service-->
vi /data/k8s-yaml/traefik/svc.yaml
{% code %}
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
    - protocol: TCP
      port: 80
      name: web
    - protocol: TCP
      port: 8080
      name: admin
{% endcode %}
<!-- endtab -->
<!-- tab Ingress-->
vi /data/k8s-yaml/traefik/ingress.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: traefik.od.com
    http:
      paths:
      - backend:
          serviceName: traefik-ingress-service
          servicePort: 8080
{% endcode %}
<!-- endtab -->
{% endtabs %}

#### 依次执行创建
浏览器打开：http://k8s-yaml.od.com/traefik 检查资源配置清单文件是否正确创建
在任意运算节点应用资源配置清单
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/traefik/rolebinding.yaml 
serviceaccount/traefik-ingress-controller created
clusterrole.rbac.authorization.k8s.io/traefik-ingress-controller created
clusterrolebinding.rbac.authorization.k8s.io/traefik-ingress-controller created

[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/traefik/daemonset.yaml 
daemonset.extensions/traefik-ingress-controller created

[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/traefik/svc.yaml 
service/traefik-ingress-service created

[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/traefik/ingress.yaml 
ingress.extensions/traefik-web-ui created
```

#### 解析域名
`HDSS7-11.host.com`上
```vi /var/named/od.com.zone
traefik	60 IN A 10.4.7.10
```

#### 配置反代
`HDSS7-11.host.com`和`HDSS7-12.host.com`两台主机上的nginx均需要配置，这里可以考虑使用saltstack或者ansible进行统一配置管理
```vi /etc/nginx/conf.d/od.com.conf
upstream default_backend_traefik {
    server 10.4.7.21:81    max_fails=3 fail_timeout=10s;
    server 10.4.7.22:81    max_fails=3 fail_timeout=10s;
}
server {
    server_name *.od.com;
  
    location / {
        proxy_pass http://default_backend_traefik;
        proxy_set_header Host       $http_host;
        proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for;
    }
} 
```

#### 浏览器访问
http://traefik.od.com

### 部署dashboard
#### 运维主机上准备dashboard镜像
```
[root@hdss7-200 ~]# docker pull k8scn/kubernetes-dashboard-amd64:v1.8.3
v1.8.3: Pulling from k8scn/kubernetes-dashboard-amd64
a4026007c47e: Pull complete 
Digest: sha256:ebc993303f8a42c301592639770bd1944d80c88be8036e2d4d0aa116148264ff
Status: Downloaded newer image for k8scn/kubernetes-dashboard-amd64:v1.8.3
[root@hdss7-200 ~]# docker tag 0c60bcf89900 harbor.od.com/k8s/dashboard:1.8.3
[root@hdss7-200 ~]# docker push harbor.od.com/k8s/dashboard:1.8.3
docker push harbor.od.com/k8s/dashboard:1.8.3
The push refers to a repository [harbor.od.com/k8s/dashboard]
23ddb8cbb75a: Pushed 
1.8.3: digest: sha256:e76c5fe6886c99873898e4c8c0945261709024c4bea773fc477629455631e472 size: 529
```
#### 运维主机上准备资源配置清单
```
[root@hdss7-200 ~]# mkdir -p /data/k8s-yaml/dashboard && cd /data/k8s-yaml/dashboard
```
{% tabs dashboard%}
<!-- tab RoleBinding -->
vi /data/k8s-yaml/dashboard/rolebinding.yaml
{% code %}
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
    addonmanager.kubernetes.io/mode: Reconcile
  name: kubernetes-dashboard
  namespace: kube-system
\--\-
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
    addonmanager.kubernetes.io/mode: Reconcile
  name: kubernetes-dashboard-admin
  namespace: kube-system
rules:
  # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs"]
  verbs: ["get", "update", "delete"]
  # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["kubernetes-dashboard-settings"]
  verbs: ["get", "update"]
  # Allow Dashboard to get metrics from heapster.
- apiGroups: [""]
  resources: ["services"]
  resourceNames: ["heapster"]
  verbs: ["proxy"]
- apiGroups: [""]
  resources: ["services/proxy"]
  resourceNames: ["heapster", "http:heapster:", "https:heapster:"]
  verbs: ["get"]
\--\-
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-admin
  namespace: kube-system
  labels:
    k8s-app: kubernetes-dashboard
    addonmanager.kubernetes.io/mode: Reconcile
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
{% endcode %}
<!-- endtab -->
<!-- tab Secret-->
vi /data/k8s-yaml/dashboard/secret.yaml
{% code %}
apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
    # Allows editing resource and makes sure it is created first.
    addonmanager.kubernetes.io/mode: EnsureExists
  name: kubernetes-dashboard-certs
  namespace: kube-system
type: Opaque
\--\-
apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
    # Allows editing resource and makes sure it is created first.
    addonmanager.kubernetes.io/mode: EnsureExists
  name: kubernetes-dashboard-key-holder
  namespace: kube-system
type: Opaque
{% endcode %}
<!-- endtab -->
<!-- tab ConfigMap-->
vi /data/k8s-yaml/dashboard/configmap.yaml
{% code %}
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    k8s-app: kubernetes-dashboard
    # Allows editing resource and makes sure it is created first.
    addonmanager.kubernetes.io/mode: EnsureExists
  name: kubernetes-dashboard-settings
  namespace: kube-system
{% endcode %}
<!-- endtab -->
<!-- tab Service-->
vi /data/k8s-yaml/dashboard/svc.yaml
{% code %}
apiVersion: v1
kind: Service
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
  labels:
    k8s-app: kubernetes-dashboard
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    k8s-app: kubernetes-dashboard
  ports:
  - port: 443
    targetPort: 8443
{% endcode %}
<!-- endtab -->
<!-- tab Ingress-->
vi /data/k8s-yaml/dashboard/ingress.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: dashboard.od.com
    http:
      paths:
      - backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
{% endcode %}
<!-- endtab -->
<!-- tab Deployment-->
vi /data/k8s-yaml/dashboard/deployment.yaml
{% code %}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
  labels:
    k8s-app: kubernetes-dashboard
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      priorityClassName: system-cluster-critical
      containers:
      - name: kubernetes-dashboard
        image: harbor.od.com/k8s/dashboard:1.8.3
        resources:
          limits:
            cpu: 100m
            memory: 300Mi
          requests:
            cpu: 50m
            memory: 100Mi
        ports:
        - containerPort: 8443
          protocol: TCP
        args:
          # PLATFORM-SPECIFIC ARGS HERE
          - --auto-generate-certificates
        volumeMounts:
        - name: kubernetes-dashboard-certs
          mountPath: /certs
        - name: tmp-volume
          mountPath: /tmp
        livenessProbe:
          httpGet:
            scheme: HTTPS
            path: /
            port: 8443
          initialDelaySeconds: 30
          timeoutSeconds: 30
      volumes:
      - name: kubernetes-dashboard-certs
        secret:
          secretName: kubernetes-dashboard-certs
      - name: tmp-volume
        emptyDir: {}
      serviceAccountName: kubernetes-dashboard
      tolerations:
      - key: "CriticalAddonsOnly"
        operator: "Exists"
{% endcode %}
<!-- endtab -->
{% endtabs %}

#### 依次执行创建
浏览器打开：http://k8s-yaml.od.com/dashboard 检查资源配置清单文件是否正确创建
在任意运算节点应用资源配置清单
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dashboard/rolebinding.yaml 
serviceaccount/kubernetes-dashboard created
role.rbac.authorization.k8s.io/kubernetes-dashboard-admin created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-admin created

[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dashboard/secret.yaml 
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-key-holder created

[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dashboard/configmap.yaml 
configmap/kubernetes-dashboard-settings created

[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dashboard/svc.yaml 
service/kubernetes-dashboard created

[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dashboard/ingress.yaml 
ingress.extensions/kubernetes-dashboard created

[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dashboard/deployment.yaml 
deployment.apps/kubernetes-dashboard created
```
#### 解析域名
`HDSS7-11.host.com`上
```vi /var/named/od.com.zone
dashboard	60 IN A 10.4.7.10
```

#### 浏览器访问
http://dashboard.od.com