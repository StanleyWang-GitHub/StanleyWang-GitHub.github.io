title: Day3：实战交付一套dubbo微服务到kubernetes集群
author: Stanley Wang
date: 2019-10-18 20:12:56
---
- - -
{% cq %}欢迎加入王导的VIP学习qq群：==>[<font color="FF7F50">932194668</font>](http://shang.qq.com/wpa/qunwpa?idkey=78869fddc5a661acb0639315eb52997c108de6625df5f0ee2f0372f176a032a6)<=={% endcq %}
- - -
# 基础架构
主机名|角色|ip
-|-|-
HDSS7-11.host.com|k8s代理节点1，zk1|10.4.7.11
HDSS7-12.host.com|k8s代理节点2，zk2|10.4.7.12
HDSS7-21.host.com|k8s运算节点1，zk3|10.4.7.21
HDSS7-22.host.com|k8s运算节点2，jenkins|10.4.7.22
HDSS7-200.host.com|k8s运维节点(docker仓库)|10.4.7.200

# 部署zookeeper
## 安装jdk1.8（3台zk角色主机）
> jdk下载地址
> [jdk1.6](https://www.oracle.com/technetwork/java/javase/downloads/java-archive-downloads-javase6-419409.html)
> [jdk1.7](https://www.oracle.com/technetwork/java/javase/downloads/java-archive-downloads-javase7-521261.html)
> [jdk1.8](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)

```pwd /opt/src
[root@hdss7-11 src]# ls -l|grep jdk
-rw-r--r-- 1 root root 153530841 Jan 17 17:49 jdk-8u201-linux-x64.tar.gz
[root@hdss7-11 src]# mkdir /usr/java
[root@hdss7-11 src]# tar xf jdk-8u201-linux-x64.tar.gz -C /usr/java
[root@hdss7-11 src]# ln -s /usr/java/jdk1.8.0_201 /usr/java/jdk
[root@hdss7-11 src]# vi /etc/profile
export JAVA_HOME=/usr/java/jdk
export PATH=$JAVA_HOME/bin:$JAVA_HOME/bin:$PATH
export CLASSPATH=$CLASSPATH:$JAVA_HOME/lib:$JAVA_HOME/lib/tools.jar
```
## 安装zookeeper（3台zk角色主机）
> zk下载地址
> [zookeeper](https://archive.apache.org/dist/zookeeper/)

### 解压、配置
```pwd /opt/src
[root@hdss7-11 src]# ls -l|grep zoo
-rw-r--r-- 1 root root 153530841 Jan 17 18:10 zookeeper-3.4.14.tar.gz
[root@hdss7-11 src]# tar xf /opt/src/zookeeper-3.4.14.tar.gz -C /opt
[root@hdss7-11 opt]# ln -s /opt/zookeeper-3.4.14/ /opt/zookeeper
[root@hdss7-11 opt]# mkdir -pv /data/zookeeper/data /data/zookeeper/logs
[root@hdss7-11 opt]# vi /opt/zookeeper/conf/zoo.cfg
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/data/zookeeper/data
dataLogDir=/data/zookeeper/logs
clientPort=2181
server.1=zk1.od.com:2888:3888
server.2=zk2.od.com:2888:3888
server.3=zk3.od.com:2888:3888
```
**注意：**各节点zk配置相同。

### myid
`HDSS7-11.host.com`上：
```vi /data/zookeeper/data/myid
1
```
`HDSS7-12.host.com`上：
```vi /data/zookeeper/data/myid
2
```
`HDSS7-21.host.com`上：
```vi /data/zookeeper/data/myid
3
```

### 做dns解析
`HDSS7-11.host.com`上
```vi /var/named/od.com.zone
zk1	60 IN A 10.4.7.11
zk2	60 IN A 10.4.7.12
zk3	60 IN A 10.4.7.21
```

### 依次启动
```
[root@hdss7-11 opt]# /opt/zookeeper/bin/zkServer.sh start
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED
```

# 部署jenkins
## 准备镜像
> [jenkins官网](https://jenkins.io/download/)
> [jenkins镜像](https://hub.docker.com/r/jenkins/jenkins)

在运维主机下载官网上的稳定版（这里下载2.164.3）
```
[root@hdss7-200 ~]#  docker pull jenkins/jenkins:2.164.3
2.164.3: Pulling from jenkins/jenkins
22dbe790f715: Pull complete 
0250231711a0: Pull complete 
6fba9447437b: Pull complete 
c2b4d327b352: Pull complete 
cddb9bb0d37c: Pull complete 
b535486c968f: Pull complete 
f3e976e6210c: Pull complete 
b2c11b10291d: Pull complete 
f4c0181e1976: Pull complete 
924c8e712392: Pull complete 
d13006b7c9dd: Pull complete 
fc80aeb92627: Pull complete 
36a6e96ba1b5: Pull complete 
f50f33dc1d0a: Pull complete 
b10642432117: Pull complete 
850c260511d8: Pull complete 
47f95e65a629: Pull complete 
3b33ce546dc6: Pull complete 
051c7665e760: Pull complete 
fe379aecc538: Pull complete 
Digest: sha256:12fd14965de7274b5201653b2bffa62700c5f5f336ec75c945321e2cb70d7af0
Status: Downloaded newer image for jenkins/jenkins:2.164.3

[root@hdss7-200 ~]# docker tag 256cb12e72d6 harbor.od.com/public/jenkins:v2.164.3
[root@hdss7-200 ~]# docker push harbor.od.com/public/jenkins:v2.164.3
```
## 自定义Dockerfile
在运维主机`HDSS7-200.host.com`上编辑自定义dockerfile
```vi /data/dockerfile/jenkins/Dockerfile
FROM harbor.od.com/public/jenkins:v2.164.3
USER root
RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime &&\ 
    echo 'Asia/Shanghai' >/etc/timezone
ADD id_rsa /root/.ssh/id_rsa
ADD config.json /root/.docker/config.json
ADD get-docker.sh /get-docker.sh
RUN echo "    StrictHostKeyChecking no" >> /etc/ssh/ssh_config &&\
    /get-docker.sh
```
这个Dockerfile里我们主要做了以下几件事
- 设置容器用户为root
- 设置容器内的时区
- 将ssh私钥加入（使用git拉代码时要用到，配对的公钥应配置在gitlab中）
- 加入了登录自建harbor仓库的config文件
- 修改了ssh客户端的配置
- 安装一个docker的客户端

生成ssh密钥对：
```
[root@hdss7-200 ~]# ssh-keygen -t rsa -b 2048 -C "xxx@xx.xx" -N "" -f /root/.ssh/id_rsa
```

{% tabs jenkins-dockerfile%}
<!-- tab config.json -->
{% code %}
{
	"auths": {
		"harbor.od.com": {
			"auth": "YWRtaW46SGFyYm9yMTIzNDU="
		}
	}
}
{% endcode %}
<!-- endtab -->
<!-- tab get-docker.sh-->
{% code %}
[root@hdss7-200 jenkins]# curl -fsSL get.docker.com -o get-docker.sh
[root@hdss7-200 jenkins]# chmod +x get-docker.sh
{% endcode %}
<!-- endtab -->
{% endtabs %}

## 制作自定义镜像
```pwd /data/dockerfile/jenkins
[root@hdss7-200 jenkins]# ls -l
total 24
-rw------- 1 root root    98 Jan  17 15:58 config.json
-rw-r--r-- 1 root root   158 Jan  17 15:59 Dockerfile
-rwxr-xr-x 1 root root 13847 Jan  17 15:37 get-docker.sh
-rw------- 1 root root  1679 Jan  17 15:39 id_rsa
[root@hdss7-200 jenkins]# docker build . -t harbor.od.com/infra/jenkins:v2.164.3
Sending build context to Docker daemon 19.46 kB
Step 1 : FROM harbor.od.com/public/jenkins:v2.164.3
 ---> 256cb12e72d6
Step 2 : USER root
 ---> Running in d600e9db8305
 ---> 03687cf21cb3
Removing intermediate container d600e9db8305
Step 3 : RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime &&    echo 'Asia/Shanghai' >/etc/timezone
 ---> Running in 3d79b4025e97
 ---> e4790b3bb6d9
Removing intermediate container 3d79b4025e97
Step 4 : ADD id_rsa /root/.ssh/id_rsa
 ---> 39d80713d43c
Removing intermediate container 7b4e66e726dd
Step 5 : ADD config.json /root/.docker/config.json
 ---> a44402fd4bc1
Removing intermediate container f1ae1871d035
Step 6 : ADD get-docker.sh /get-docker.sh
 ---> 189ccca429e4
Removing intermediate container a0ff59237fe5
Step 7 : RUN /get-docker.sh
 ---> Running in 5a7d69c1af45
# Executing docker install script, commit: cfba462
+ sh -c apt-get update -qq >/dev/null
+ sh -c apt-get install -y -qq apt-transport-https ca-certificates curl >/dev/null
debconf: delaying package configuration, since apt-utils is not installed
+ sh -c curl -fsSL "https://download.docker.com/linux/debian/gpg" | apt-key add -qq - >/dev/null
Warning: apt-key output should not be parsed (stdout is not a terminal)
+ sh -c echo "deb [arch=amd64] https://download.docker.com/linux/debian stretch stable" > /etc/apt/sources.list.d/docker.list
+ sh -c apt-get update -qq >/dev/null
+ sh -c apt-get install -y -qq --no-install-recommends docker-ce >/dev/null
debconf: delaying package configuration, since apt-utils is not installed
If you would like to use Docker as a non-root user, you should now consider
adding your user to the "docker" group with something like:

  sudo usermod -aG docker your-user

Remember that you will have to log out and back in for this to take effect!

WARNING: Adding a user to the "docker" group will grant the ability to run
         containers which can be used to obtain root privileges on the
         docker host.
         Refer to https://docs.docker.com/engine/security/security/#docker-daemon-attack-surface
         for more information.

** DOCKER ENGINE - ENTERPRISE **

If you’re ready for production workloads, Docker Engine - Enterprise also includes:

  * SLA-backed technical support
  * Extended lifecycle maintenance policy for patches and hotfixes
  * Access to certified ecosystem content

** Learn more at https://dockr.ly/engine2 **

ACTIVATE your own engine to Docker Engine - Enterprise using:

  sudo docker engine activate

 ---> 64c74242ee28
Removing intermediate container 5a7d69c1af45
Successfully built 64c74242ee28
```

## 创建infra仓库
在Harbor页面，创建infra仓库，**注意：私有仓库**

## 创建kubernetes命名空间
任意运算节点上：
```
[root@hdss7-21 ~]# kubectl create namespace infra
namespace/infra created
[root@hdss7-21 ~]# kubectl create secret docker-registry harbor --docker-server=harbor.od.com --docker-username=admin --docker-password=Harbor12345 -n infra
secret/harbor created
```

## 推送镜像
```
[root@hdss7-200 jenkins]# docker push harbor.od.com/infra/jenkins:v2.164.3
```

## 准备共享存储
运维主机，以及所有运算节点上：
```
# yum install nfs-utils -y
```
- 配置NFS服务

运维主机`HDSS7-200.host.com`上：
```vi /etc/exports
/data/nfs-volume 10.4.7.0/24(rw,no_root_squash)
```
- 启动NFS服务

运维主机`HDSS7-200.host.com`上：
```
[root@hdss7-200 ~]# mkdir -p /data/nfs-volume
[root@hdss7-200 ~]# systemctl start nfs
[root@hdss7-200 ~]# systemctl enable nfs
```

## 准备资源配置清单
运维主机`HDSS7-200.host.com`上：
```pwd /data/k8s-yaml
[root@hdss7-200 k8s-yaml]# mkdir /data/k8s-yaml/jenkins && mkdir /data/nfs-volume/jenkins_home && cd /data/k8s-yaml/jenkins 
```
{% tabs jenkins-yaml%}
<!-- tab Deployment -->
vi dp.yaml
{% code %}
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: jenkins
  namespace: infra
  labels: 
    name: jenkins
spec:
  replicas: 1
  selector:
    matchLabels: 
      name: jenkins
  template:
    metadata:
      labels: 
        app: jenkins 
        name: jenkins
    spec:
      volumes:
      - name: data
        nfs: 
          server: hdss7-200
          path: /data/nfs-volume/jenkins_home
      - name: docker
        hostPath: 
          path: /run/docker.sock
          type: ''
      containers:
      - name: jenkins
        image: harbor.od.com/infra/jenkins:v2.164.3
        ports:
        - containerPort: 8080
          protocol: TCP
        env:
        - name: JAVA_OPTS
          value: -Xmx512m -Xms512m
        resources:
          limits: 
            cpu: 500m
            memory: 1Gi
          requests: 
            cpu: 500m
            memory: 1Gi
        volumeMounts:
        - name: data
          mountPath: /var/jenkins_home
        - name: docker
          mountPath: /run/docker.sock
        imagePullPolicy: IfNotPresent
      imagePullSecrets:
      - name: harbor
      restartPolicy: Always
      securityContext: 
        runAsUser: 0
  strategy:
    type: RollingUpdate
    rollingUpdate: 
      maxUnavailable: 1
      maxSurge: 1
  revisionHistoryLimit: 7
  progressDeadlineSeconds: 600
{% endcode %}
<!-- endtab -->
<!-- tab Service-->
vi svc.yaml
{% code %}
kind: Service
apiVersion: v1
metadata: 
  name: jenkins
  namespace: infra
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  selector:
    app: jenkins
{% endcode %}
<!-- endtab -->
<!-- tab Ingress-->
vi ingress.yaml
{% code %}
kind: Ingress
apiVersion: extensions/v1beta1
metadata: 
  name: jenkins
  namespace: infra
spec:
  rules:
  - host: jenkins.od.com
    http:
      paths:
      - path: /
        backend: 
          serviceName: jenkins
          servicePort: 80
{% endcode %}
<!-- endtab -->
{% endtabs %}

## 应用资源配置清单
任意一个k8s运算节点上
```
[root@hdss7-21 ~]# kubectl create namespace infra
[root@hdss7-21 ~]# kubectl apply -f  http://k8s-yaml.od.com/jenkins/dp.yaml
[root@hdss7-21 ~]# kubectl apply -f  http://k8s-yaml.od.com/jenkins/svc.yaml
[root@hdss7-21 ~]# kubectl apply -f  http://k8s-yaml.od.com/jenkins/ingress.yaml

[root@hdss7-21 ~]# kubectl get pods -n infra|grep jenkins
NAME                                READY     STATUS    RESTARTS   AGE
jenkins-84455f9675-jpkr8            1/1       Running   0          0d

[root@hdss7-21 ~]# kubectl get svc -n infra|grep jenkins
NAME           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
jenkins        ClusterIP   None             <none>        8080/TCP   0d

[root@hdss7-21 ~]# kubectl get ingress -n infra|grep jenkins
NAME           HOSTS                          ADDRESS   PORTS     AGE
jenkins        jenkins.od.com                           80        0d
```

## 解析域名
`HDSS7-11.host.com`上
```vi /var/named/od.com.zone
jenkins	60 IN A 10.4.7.10
```

## 浏览器访问
http://jenkins.od.com

## 页面配置jenkins
![jenkins初始化页面](/images/jenkins-init.png "jenkins初始化页面")
### 初始化密码
```cat /data/nfs-volume/jenkins_home/secrets/initialAdminPassword
[root@hdss7-200 secrets]# cat initialAdminPassword 
08d17edc125444a28ad6141ffdfd5c69
```
### 安装插件
![jenkins安装页面](/images/jenkins-install.png "jenkins安装页面")

### 设置用户
![jenkins设置用户](/images/jenkins-user.png "jenkins设置用户")

### 完成安装
![jenkins完成安装1](/images/jenkins-url.png "jenkins完成安装1")
![jenkins完成安装2](/images/jenkins-ready.png "jenkins完成安装2")

### 使用admin登录
![jenkins登录](/images/jenkins-welcome.png "jenkins登录")

### 安装Blue Ocean插件
- Manage Jenkins
- Manage Plugins
- Available
- Blue Ocean

### 调整安全选项
- Manage Jenkins
- Configure Global Security
- Allow anonymous read access

## 配置New job
- create new jobs
- Enter an item name
> dubbo-demo

- Pipeline -> OK
- Discard old builds
> Days to keep builds : 3
> Max # of builds to keep : 30

- This project is parameterized
1. Add Parameter -> String Parameter
> Name : app_name
> Default Value : 
> Description : project name. e.g: dubbo-demo-service

2. Add Parameter -> String Parameter
> Name : image_name
> Default Value : 
> Description : project docker image name. e.g:  app/dubbo-demo-service

3. Add Parameter -> String Parameter
> Name : git_repo
> Default Value : 
> Description : project git repository. e.g: https://gitee.com/stanleywang/dubbo-demo-service.git

4. Add Parameter -> String Parameter
> Name : git_ver
> Default Value : 
> Description : git commit id of the project.

5. Add Parameter -> String Parameter
> Name : add_tag
> Default Value : 
> Description : project docker image tag, date_timestamp recommended. e.g: 190117_1920

6. Add Parameter -> String Parameter
> Name : mvn_dir
> Default Value : ./
> Description : project maven directory. e.g: ./

7. Add Parameter -> String Parameter
> Name : target_dir
> Default Value : ./target
> Description : the relative path of target file such as .jar or .war package. e.g: ./dubbo-server/target

8. Add Parameter -> String Parameter
> Name : mvn_cmd
> Default Value : mvn clean package -Dmaven.test.skip=true
> Description : maven command. e.g: mvn clean package -e -q -Dmaven.test.skip=true

9. Add Parameter -> Choice Parameter
> Name : base_image
> Default Value : 
> - base/jre7:7u80
> - base/jre8:8u112
> Description : project base image list in harbor.od.com.

10. Add Parameter -> Choice Parameter
> Name : maven
> Default Value : 
> - 3.6.1-8u212
> - 3.2.5-6u025
> - 2.2.1-6u025
> Description : different maven edition.

## Pipeline Script
```
pipeline {
  agent any 
    stages {
      stage('pull') { //get project code from repo 
        steps {
          sh "git clone ${params.git_repo} ${params.app_name}/${env.BUILD_NUMBER} && cd ${params.app_name}/${env.BUILD_NUMBER} && git checkout ${params.git_ver}"
        }
      }
      stage('build') { //exec mvn cmd
        steps {
          sh "cd ${params.app_name}/${env.BUILD_NUMBER}  && /var/jenkins_home/maven-${params.maven}/bin/${params.mvn_cmd}"
        }
      }
      stage('package') { //move jar file into project_dir
        steps {
          sh "cd ${params.app_name}/${env.BUILD_NUMBER} && cd ${params.target_dir} && mkdir project_dir && mv *.jar ./project_dir"
        }
      }
      stage('image') { //build image and push to registry
        steps {
          writeFile file: "${params.app_name}/${env.BUILD_NUMBER}/Dockerfile", text: """FROM harbor.od.com/${params.base_image}
ADD ${params.target_dir}/project_dir /opt/project_dir"""
          sh "cd  ${params.app_name}/${env.BUILD_NUMBER} && docker build -t harbor.od.com/${params.image_name}:${params.git_ver}_${params.add_tag} . && docker push harbor.od.com/${params.image_name}:${params.git_ver}_${params.add_tag}"
        }
      }
    }
}
```

# 最后的准备工作
## 检查jenkins容器里的docker客户端
进入jenkins的docker容器里，检查docker客户端是否可用。
```
[root@hdss7-22 ~]# docker exec -ti 52e250789b78 bash
root@52e250789b78:/# docker ps -a
```
## 检查jenkins容器里的SSH key
进入jenkins的docker容器里，检查ssh连接git仓库，确认是否能拉到代码。
```
[root@hdss7-22 ~]# docker exec -ti 52e250789b78 bash
root@52e250789b78:/# ssh -i /root/.ssh/id_rsa -T git@gitee.com                                                                                              
Hi Anonymous! You've successfully authenticated, but GITEE.COM does not provide shell access.
Note: Perhaps the current use is DeployKey.
Note: DeployKey only supports pull/fetch operations
```
## 部署maven软件
[maven官方下载地址](http://maven.apache.org/docs/history.html)
在运维主机`HDSS7-200.host.com`上二进制部署，这里部署maven-3.6.1版
```pwd /opt/src
[root@hdss7-22 src]# ls -l
total 8852
-rw-r--r-- 1 root root 9063587 Jan  17 19:57 apache-maven-3.6.1-bin.tar.gz
[root@hdss7-200 src]# tar xf apache-maven-3.6.1-bin.tar.gz -C /data/nfs-volume/jenkins_home/maven-3.6.1-8u212
[root@hdss7-200 src]# mv /data/nfs-volume/jenkins_home/apache-maven-3.6.1/ /data/nfs-volume/jenkins_home/maven-3.6.1-8u212
[root@hdss7-200 src]# ls -ld /data/nfs-volume/jenkins_home/maven-3.6.1-8u212
drwxr-xr-x 6 root root 99 Jan  17 19:58 /data/nfs-volume/jenkins_home/maven-3.6.1-8u212
```
设置国内镜像源
```vi /data/nfs-volume/jenkins_home/maven-3.6.1-8u212/conf/setting.xml
    <mirror>
      <id>nexus-aliyun</id>
      <mirrorOf>*</mirrorOf>
      <name>Nexus aliyun</name>
      <url>http://maven.aliyun.com/nexus/content/groups/public</url>
    </mirror>
```
其他版本略

## 制作dubbo微服务的底包镜像
运维主机`HDSS7-200.host.com`上
1. 自定义Dockerfile

```vi /data/dockerfile/jre8/Dockerfile
FROM stanleyws/jre8:8u112
RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime &&\
    echo 'Asia/Shanghai' >/etc/timezone
ADD config.yml /opt/prom/config.yml
ADD jmx_javaagent-0.3.1.jar /opt/prom/
WORKDIR /opt/project_dir
ADD entrypoint.sh /entrypoint.sh
CMD ["/entrypoint.sh"]
```
{% tabs jre-dockerfile%}
<!-- tab config.yml -->
vi config.yml
{% code %}
---
rules:
  - pattern: '.*'
{% endcode %}
<!-- endtab -->
<!-- tab jmx_javaagent-0.3.1.jar-->
{% code %}
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.3.1/jmx_prometheus_javaagent-0.3.1.jar -O jmx_javaagent-0.3.1.jar
{% endcode %}
<!-- endtab -->
<!-- tab entrypoint.sh-->
vi entrypoint.sh  (不要忘了给执行权限)
{% code %}
#!/bin/sh
M_OPTS="-Duser.timezone=Asia/Shanghai -javaagent:/opt/prom/jmx_javaagent-0.3.1.jar=$(hostname -i):${M_PORT:-"12346"}:/opt/prom/config.yml"
C_OPTS=${C_OPTS}
JAR_BALL=${JAR_BALL}
exec java -jar ${M_OPTS} ${C_OPTS} ${JAR_BALL}
{% endcode %}
<!-- endtab -->
{% endtabs %}

2. 制作dubbo服务docker底包

```pwd /data/dockerfile/jre8
[root@hdss7-200 jre8]# ls -l
total 372
-rw-r--r-- 1 root root     29 Jan  17 19:09 config.yml
-rw-r--r-- 1 root root    287 Jan  17 19:06 Dockerfile
-rwxr--r-- 1 root root    250 Jan  17 19:11 entrypoint.sh
-rw-r--r-- 1 root root 367417 May 10  2018 jmx_javaagent-0.3.1.jar

[root@hdss7-200 jre8]# docker build . -t harbor.od.com/base/jre8:8u112
Sending build context to Docker daemon 372.2 kB
Step 1 : FROM stanleyws/jre8:8u112
 ---> fa3a085d6ef1
Step 2 : RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime &&    echo 'Asia/Shanghai' >/etc/timezone
 ---> Using cache
 ---> 5da5ab0b1a48
Step 3 : ADD config.yml /opt/prom/config.yml
 ---> Using cache
 ---> 70d3ebfe88f5
Step 4 : ADD jmx_javaagent-0.3.1.jar /opt/prom/
 ---> Using cache
 ---> 08b38a0684a8
Step 5 : WORKDIR /opt/project_dir
 ---> Using cache
 ---> f06adf17fb69
Step 6 : ADD entrypoint.sh /entrypoint.sh
 ---> e34f185d5c52
Removing intermediate container ee213576ca0e
Step 7 : CMD /entrypoint.sh
 ---> Running in 655f594bcbe2
 ---> 47852bc0ade9
Removing intermediate container 655f594bcbe2
Successfully built 47852bc0ade9

[root@hdss7-200 jre8]# docker push harbor.od.com/base/jre8:8u112
The push refers to a repository [harbor.od.com/base/jre8]
0b2b753b122e: Pushed 
67e1b844d09c: Pushed 
ad4fa4673d87: Pushed 
0ef3a1b4ca9f: Pushed 
052016a734be: Pushed 
0690f10a63a5: Pushed 
c843b2cf4e12: Pushed 
fddd8887b725: Pushed 
42052a19230c: Pushed 
8d4d1ab5ff74: Pushed 
8u112: digest: sha256:252e3e869039ee6242c39bdfee0809242e83c8c3a06830f1224435935aeded28 size: 2405
```
**注意：**jre7底包制作类似，这里略

# 交付dubbo微服务至kubernetes集群
## dubbo服务提供者（dubbo-demo-service）
### 通过jenkins进行一次CI
打开jenkins页面，使用admin登录，准备构建`dubbo-demo`项目

![jenkins构建](/images/jenkins-firstbuild.png "jenkins构建")
点`Build with Parameters`

![jenkins构建详情](/images/jenkins-builddetail.png "jenkins构建详情")
依次填入/选择：
- app_name
> dubbo-demo-service

- image_name
> app/dubbo-demo-service

- git_repo
> https://gitee.com/stanleywang/dubbo-demo-service.git

- git_ver
> master

- add_tag
> 190117_1920

- mvn_dir
> /

- target_dir
> ./dubbo-server/target

- mvn_cmd
> mvn clean package -Dmaven.test.skip=true

- base_image
> base/jre8:8u112

- maven
> 3.6.1-8u212

点击`Build`进行构建，等待构建完成。

> test $? -eq 0 && {% label success@成功，进行下一步 %} || {% label danger@失败，排错直到成功 %}

### 检查harbor仓库内镜像
![harbor仓库内镜像](/images/harbor-firstci.png "harbor仓库内镜像")

### 准备k8s资源配置清单
运维主机`HDSS7-200.host.com`上，准备资源配置清单：
```vi /data/k8s-yaml/dubbo-demo-service/dp.yaml
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: dubbo-demo-service
  namespace: app
  labels: 
    name: dubbo-demo-service
spec:
  replicas: 1
  selector:
    matchLabels: 
      name: dubbo-demo-service
  template:
    metadata:
      labels: 
        app: dubbo-demo-service
        name: dubbo-demo-service
    spec:
      containers:
      - name: dubbo-demo-service
        image: harbor.od.com/app/dubbo-demo-service:master_190117_1920
        ports:
        - containerPort: 20880
          protocol: TCP
        env:
        - name: JAR_BALL
          value: dubbo-server.jar
        imagePullPolicy: IfNotPresent
      imagePullSecrets:
      - name: harbor
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      securityContext: 
        runAsUser: 0
      schedulerName: default-scheduler
  strategy:
    type: RollingUpdate
    rollingUpdate: 
      maxUnavailable: 1
      maxSurge: 1
  revisionHistoryLimit: 7
  progressDeadlineSeconds: 600
```

### 应用资源配置清单
在任意一台k8s运算节点执行：
```
[root@hdss7-21 ~]# kubectl create namespace app
namespace/app created
[root@hdss7-21 ~]# kubectl create secret docker-registry harbor --docker-server=harbor.od.com --docker-username=admin --docker-password=Harbor12345 -n app
secret/harbor created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dubbo-demo-service/dp.yaml
deployment.extensions/dubbo-demo-service created
```

### 检查docker运行情况及zk里的信息
```vi /opt/zookeeper/bin/zkCli.sh
[root@hdss7-11 ~]# /opt/zookeeper/bin/zkCli.sh -server localhost
[zk: localhost(CONNECTED) 0] ls /dubbo
[com.od.dubbotest.api.HelloService]
```

## dubbo-monitor工具
[dubbo-monitor源码包](https://github.com/Jeromefromcn/dubbo-monitor.git)
### 准备docker镜像
#### 下载源码
下载到运维主机`HDSS7-200.host.com`上
```pwd /opt/src
[root@hdss7-200 src]# ls -l|grep dubbo-monitor
drwxr-xr-x 4 root root      81 Jan  17 13:58 dubbo-monitor
```
#### 修改配置
```vi /opt/src/dubbo-monitor/dubbo-monitor-simple/conf/dubbo_origin.properties
dubbo.registry.address=zookeeper://zk1.od.com:2181?backup=zk2.od.com:2181,zk3.od.com:2181
dubbo.protocol.port=20880
dubbo.jetty.port=8080
dubbo.jetty.directory=/dubbo-monitor-simple/monitor
dubbo.statistics.directory=/dubbo-monitor-simple/statistics
dubbo.charts.directory=/dubbo-monitor-simple/charts
dubbo.log4j.file=logs/dubbo-monitor.log
```

#### 制作镜像
1. 准备环境
```
[root@hdss7-200 src]# mkdir /data/dockerfile/dubbo-monitor
[root@hdss7-200 src]# cp -a dubbo-monitor/* /data/dockerfile/dubbo-monitor/
[root@hdss7-200 src]# cd /data/dockerfile/dubbo-monitor/
[root@hdss7-200 dubbo-monitor]# sed -r -i -e '/^nohup/{p;:a;N;$!ba;d}'  ./dubbo-monitor-simple/bin/start.sh && sed  -r -i -e "s%^nohup(.*)%exec \1%"  ./dubbo-monitor-simple/bin/start.sh
```
2. 准备Dockerfile
```vi /data/dockerfile/dubbo-monitor/Dockerfile
FROM jeromefromcn/docker-alpine-java-bash
MAINTAINER Jerome Jiang
COPY dubbo-monitor-simple/ /dubbo-monitor-simple/
CMD /dubbo-monitor-simple/bin/start.sh
```
3. build镜像
```
[root@hdss7-200 dubbo-monitor]# docker build . -t harbor.od.com/infra/dubbo-monitor:latest
Sending build context to Docker daemon 26.21 MB
Step 1 : FROM harbor.od.com/base/jre7:7u80
 ---> dbba4641da57
Step 2 : MAINTAINER Stanley Wang
 ---> Running in 8851a3c55d4b
 ---> 6266a6f15dc5
Removing intermediate container 8851a3c55d4b
Step 3 : COPY dubbo-monitor-simple/ /opt/dubbo-monitor/
 ---> f4e0a9067c5c
Removing intermediate container f1038ecb1055
Step 4 : WORKDIR /opt/dubbo-monitor
 ---> Running in 4056339d1b5a
 ---> e496e2d3079e
Removing intermediate container 4056339d1b5a
Step 5 : CMD /opt/dubbo-monitor/bin/start.sh
 ---> Running in c33b8fb98326
 ---> 97e40c179bbe
Removing intermediate container c33b8fb98326
Successfully built 97e40c179bbe

[root@hdss7-200 dubbo-monitor]# docker push harbor.od.com/infra/dubbo-monitor:latest
The push refers to a repository [harbor.od.com/infra/dubbo-monitor]
750135a87545: Pushed 
0b2b753b122e: Pushed 
5b1f1b5295ff: Pushed 
d54f1d9d76d3: Pushed 
8d51c20d6553: Pushed 
106b765202e9: Pushed 
c6698ca565d0: Pushed 
50ecb880731d: Pushed 
fddd8887b725: Pushed 
42052a19230c: Pushed 
8d4d1ab5ff74: Pushed 
190107_1930: digest: sha256:73007a37a55ecd5fd72bc5b36d2ab0bb639c96b32b7879984d5cdbc759778790 size: 2617
```

### 解析域名
在DNS主机`HDSS7-11.host.com`上：
```vi /var/named/od.com.zone
dubbo-monitor      A    10.4.7.10
```

### 准备k8s资源配置清单
运维主机`HDSS7-200.host.com`上
{% tabs dubbo-monitor%}
<!-- tab Deployment -->
vi /data/k8s-yaml/dubbo-monitor/dp.yaml
{% code %}
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: dubbo-monitor
  namespace: infra
  labels: 
    name: dubbo-monitor
spec:
  replicas: 1
  selector:
    matchLabels: 
      name: dubbo-monitor
  template:
    metadata:
      labels: 
        app: dubbo-monitor
        name: dubbo-monitor
    spec:
      containers:
      - name: dubbo-monitor
        image: harbor.od.com/infra/dubbo-monitor:latest
        ports:
        - containerPort: 8080
          protocol: TCP
        - containerPort: 20880
          protocol: TCP
        imagePullPolicy: IfNotPresent
      imagePullSecrets:
      - name: harbor
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      securityContext: 
        runAsUser: 0
      schedulerName: default-scheduler
  strategy:
    type: RollingUpdate
    rollingUpdate: 
      maxUnavailable: 1
      maxSurge: 1
  revisionHistoryLimit: 7
  progressDeadlineSeconds: 600
{% endcode %}
<!-- endtab -->
<!-- tab Service-->
vi /data/k8s-yaml/dubbo-monitor/svc.yaml
{% code %}
kind: Service
apiVersion: v1
metadata: 
  name: dubbo-monitor
  namespace: infra
spec:
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
  selector: 
    app: dubbo-monitor
{% endcode %}
<!-- endtab -->
<!-- tab Ingress -->
vi /data/k8s-yaml/dubbo-monitor/ingress.yaml
{% code %}
kind: Ingress
apiVersion: extensions/v1beta1
metadata: 
  name: dubbo-monitor
  namespace: infra
spec:
  rules:
  - host: dubbo-monitor.od.com
    http:
      paths:
      - path: /
        backend: 
          serviceName: dubbo-monitor
          servicePort: 8080
{% endcode %}
<!-- endtab -->
{% endtabs %}

### 应用资源配置清单
在任意一台k8s运算节点执行：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dubbo-monitor/dp.yaml
deployment.extensions/dubbo-monitor created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dubbo-monitor/svc.yaml
service/dubbo-monitor created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dubbo-monitor/ingress.yaml
ingress.extensions/dubbo-monitor created
```

### 浏览器访问
http://dubbo-monitor.od.com

## dubbo服务消费者（dubbo-demo-consumer）
### 通过jenkins进行一次CI
打开jenkins页面，使用admin登录，准备构建`dubbo-demo`项目

![jenkins构建](/images/jenkins-firstbuild.png "jenkins构建")
点`Build with Parameters`

![jenkins构建详情](/images/jenkins-builddetail.png "jenkins构建详情")
依次填入/选择：
- app_name
> dubbo-demo-consumer

- image_name
> app/dubbo-demo-consumer

- git_repo
> git@gitee.com:stanleywang/dubbo-demo-web.git

- git_ver
> master

- add_tag
> 190117_1950

- mvn_dir
> /

- target_dir
> ./dubbo-client/target

- mvn_cmd
> mvn clean package -Dmaven.test.skip=true

- base_image
> base/jre8:8u112

- maven
> 3.6.1-8u212

点击`Build`进行构建，等待构建完成。

> test $? -eq 0 && {% label success@成功，进行下一步 %} || {% label danger@失败，排错直到成功 %}

### 检查harbor仓库内镜像
![harbor仓库内镜像](/images/harbor-secondci.png "harbor仓库内镜像")

### 解析域名
在DNS主机`HDSS7-11.host.com`上：
```vi /var/named/od.com.zone
demo      A    10.9.7.10
```

### 准备k8s资源配置清单
运维主机`HDSS7-200.host.com`上，准备资源配置清单
{% tabs dubbo-demo-consumer%}
<!-- tab Deployment -->
vi /data/k8s-yaml/dubbo-demo-consumer/dp.yaml
{% code %}
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: dubbo-demo-consumer
  namespace: app
  labels: 
    name: dubbo-demo-consumer
spec:
  replicas: 1
  selector:
    matchLabels: 
      name: dubbo-demo-consumer
  template:
    metadata:
      labels: 
        app: dubbo-demo-consumer
        name: dubbo-demo-consumer
    spec:
      containers:
      - name: dubbo-demo-consumer
        image: harbor.od.com/app/dubbo-demo-consumer:master_190119_2015
        ports:
        - containerPort: 8080
          protocol: TCP
        - containerPort: 20880
          protocol: TCP
        env:
        - name: JAR_BALL
          value: dubbo-client.jar
        imagePullPolicy: IfNotPresent
      imagePullSecrets:
      - name: harbor
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      securityContext: 
        runAsUser: 0
      schedulerName: default-scheduler
  strategy:
    type: RollingUpdate
    rollingUpdate: 
      maxUnavailable: 1
      maxSurge: 1
  revisionHistoryLimit: 7
  progressDeadlineSeconds: 600
{% endcode %}
<!-- endtab -->
<!-- tab Service -->
vi /data/k8s-yaml/dubbo-demo-consumer/svc.yaml
{% code %}
kind: Service
apiVersion: v1
metadata: 
  name: dubbo-demo-consumer
  namespace: app
spec:
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
  selector: 
    app: dubbo-demo-consumer
{% endcode %}
<!-- endtab -->
<!-- tab Ingress -->
vi /data/k8s-yaml/dubbo-demo-consumer/ingress.yaml
{% code %}
kind: Ingress
apiVersion: extensions/v1beta1
metadata: 
  name: dubbo-demo-consumer
  namespace: app
spec:
  rules:
  - host: demo.od.com
    http:
      paths:
      - path: /
        backend: 
          serviceName: dubbo-demo-consumer
          servicePort: 8080
{% endcode %}
<!-- endtab -->
{% endtabs %}

### 应用资源配置清单
在任意一台k8s运算节点执行：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dubbo-demo-consumer/dp.yaml
deployment.extensions/dubbo-demo-consumer created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dubbo-demo-consumer/svc.yaml
service/dubbo-demo-consumer created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dubbo-demo-consumer/ingress.yaml
ingress.extensions/dubbo-demo-consumer created
```
### 检查docker运行情况及dubbo-monitor
http://dubbo-monitor.od.com

### 浏览器访问
http://demo.od.com/hello?name=wangdao

# 实战维护dubbo微服务集群
## 更新（rolling update）
- 修改代码提git（发版）
- 使用jenkins进行CI
- 修改并应用k8s资源配置清单
> 或者在k8s的dashboard上直接操作

## 扩容（scaling）
- k8s的dashboard上直接操作

