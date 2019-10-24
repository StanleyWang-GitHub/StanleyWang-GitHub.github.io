title: Day6：使用ELK Stack对K8S内的应用进行日志分析
author: Stanley Wang
date: 2019-10-18 17:12:56
---
- - -
{% cq %}欢迎加入王导的VIP学习qq群：==>[<font color="FF7F50">932194668</font>](http://shang.qq.com/wpa/qunwpa?idkey=78869fddc5a661acb0639315eb52997c108de6625df5f0ee2f0372f176a032a6)<=={% endcq %}
- - -
# 改造dubbo-demo-web项目为Tomcat启动项目
[Tomcat官网](http://tomcat.apache.org/)
## 准备Tomcat的镜像底包
### 准备tomcat二进制包
运维主机`HDSS7-200.host.com`上：
[Tomcat8下载链接](http://mirrors.tuna.tsinghua.edu.cn/apache/tomcat/tomcat-8/v8.5.46/bin/apache-tomcat-8.5.46.tar.gz)
```pwd /opt/src
[root@hdss7-200 src]# ls -l|grep tomcat
-rw-r--r-- 1 root root   9690027 Apr 10 22:57 apache-tomcat-8.5.46.tar.gz
[root@hdss7-200 src]# mkdir -p /data/dockerfile/tomcat && tar xf apache-tomcat-8.5.46.tar.gz -C /data/dockerfile/tomcat
[root@hdss7-200 src]# cd /data/dockerfile/tomcat && rm -fr apache-tomcat-8.5.46/webapps/*
```

### 简单配置tomcat
1. 关闭AJP端口

```vi /data/dockerfile/tomcat/apache-tomcat-8.5.46/conf/server.xml
<!-- <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" /> -->
```
2. 配置日志

- 删除3manager，4host-manager的handlers

```vi /data/dockerfile/tomcat/apache-tomcat-8.5.46/conf/logging.properties
handlers = 1catalina.org.apache.juli.AsyncFileHandler, 2localhost.org.apache.juli.AsyncFileHandler,java.util.logging.ConsoleHandler
```
- 日志级别改为INFO

```vi /data/dockerfile/tomcat/apache-tomcat-8.5.46/conf/logging.properties
1catalina.org.apache.juli.AsyncFileHandler.level = INFO
2localhost.org.apache.juli.AsyncFileHandler.level = INFO
java.util.logging.ConsoleHandler.level = INFO
```
- 注释掉所有关于3manager，4host-manager日志的配置

```vi /data/dockerfile/tomcat/apache-tomcat-8.5.46/conf/logging.properties
#3manager.org.apache.juli.AsyncFileHandler.level = FINE
#3manager.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs
#3manager.org.apache.juli.AsyncFileHandler.prefix = manager.
#3manager.org.apache.juli.AsyncFileHandler.encoding = UTF-8

#4host-manager.org.apache.juli.AsyncFileHandler.level = FINE
#4host-manager.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs
#4host-manager.org.apache.juli.AsyncFileHandler.prefix = host-manager.
#4host-manager.org.apache.juli.AsyncFileHandler.encoding = UTF-8
```

### 准备Dockerfile
```vi /data/dockerfile/tomcat/Dockerfile
From stanleyws/jre8:8u112
RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime &&\ 
    echo 'Asia/Shanghai' >/etc/timezone
ENV CATALINA_HOME /opt/tomcat
ENV LANG zh_CN.UTF-8
ADD apache-tomcat-8.5.46/ /opt/tomcat
ADD config.yml /opt/prom/config.yml
ADD jmx_javaagent-0.3.1.jar /opt/prom/jmx_javaagent-0.3.1.jar
WORKDIR /opt/tomcat
ADD entrypoint.sh /entrypoint.sh
CMD ["/entrypoint.sh"]
```

{% tabs tomcat-dockerfile%}
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
#!/bin/bash
M_OPTS="-Duser.timezone=Asia/Shanghai -javaagent:/opt/prom/jmx_javaagent-0.3.1.jar=$(hostname -i):${M_PORT:-"12346"}:/opt/prom/config.yml"
C_OPTS=${C_OPTS}
MIN_HEAP=${MIN_HEAP:-"128m"}
MAX_HEAP=${MAX_HEAP:-"128m"}
JAVA_OPTS=${JAVA_OPTS:-"-Xmn384m -Xss256k -Duser.timezone=GMT+08  -XX:+DisableExplicitGC -XX:+UseConcMarkSweepGC -XX:+UseParNewGC -XX:+CMSParallelRemarkEnabled -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=0 -XX:+CMSClassUnloadingEnabled -XX:LargePageSizeInBytes=128m -XX:+UseFastAccessorMethods -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=80 -XX:SoftRefLRUPolicyMSPerMB=0 -XX:+PrintClassHistogram  -Dfile.encoding=UTF8 -Dsun.jnu.encoding=UTF8"}
CATALINA_OPTS="${CATALINA_OPTS}"
JAVA_OPTS="${M_OPTS} ${C_OPTS} -Xms${MIN_HEAP} -Xmx${MAX_HEAP} ${JAVA_OPTS}"
sed -i -e "1a\JAVA_OPTS=\"$JAVA_OPTS\"" -e "1a\CATALINA_OPTS=\"$CATALINA_OPTS\"" /opt/tomcat/bin/catalina.sh

cd /opt/tomcat && /opt/tomcat/bin/catalina.sh run 2>&1 >> /opt/tomcat/logs/stdout.log

{% endcode %}
<!-- endtab -->
{% endtabs %}

### 制作镜像并推送
```
[root@hdss7-200 tomcat]# docker build . -t harbor.od.com/base/tomcat:v8.5.46
Sending build context to Docker daemon 9.502 MB
Step 1 : FROM stanleyws/jre8:8u112
 ---> fa3a085d6ef1
Step 2 : RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime &&    echo 'Asia/Shanghai' >/etc/timezone
 ---> Using cache
 ---> 5da5ab0b1a48
Step 3 : ENV CATALINA_HOME /opt/tomcat
 ---> Running in edf8f2bbeae3
 ---> 05c7b829c8ab
Removing intermediate container edf8f2bbeae3
Step 4 : ENV LANG zh_CN.UTF-8
 ---> Running in 50516133f65b
 ---> 421d67c4c188
Removing intermediate container 50516133f65b
Step 5 : ADD apache-tomcat-8.5.46/ /opt/tomcat
 ---> 460a5d0ef93b
Removing intermediate container 0be2b4897d23
Step 6 : ADD config.yml /opt/prom/config.yml
 ---> 04c9d4679f0d
Removing intermediate container 0e88a4ed2088
Step 7 : ADD jmx_javaagent-0.3.1.jar /opt/prom/jmx_javaagent-0.3.1.jar
 ---> 06e4a78e9cbf
Removing intermediate container 1e6b2919be00
Step 8 : WORKDIR /opt/tomcat
 ---> Running in a51676d15a01
 ---> e8d164847eb3
Removing intermediate container a51676d15a01
Step 9 : ADD entrypoint.sh /entrypoint.sh
 ---> 0522db421536
Removing intermediate container 7f0be318fde8
Step 10 : CMD /entrypoint.sh
 ---> Running in c2df6e511c00
 ---> 8d735515bb42
Removing intermediate container c2df6e511c00
Successfully built 8d735515bb42

[root@hdss7-200 tomcat8]# docker push harbor.od.com/base/tomcat:v8.5.46
The push refers to a repository [harbor.od.com/base/tomcat]
498eaadf86a8: Pushed 
fab679acf269: Pushed 
0e65f86c3a75: Pushed 
938c8a5617cc: Pushed 
052016a734be: Mounted from app/dubbo-demo-web 
0690f10a63a5: Mounted from app/dubbo-demo-web 
c843b2cf4e12: Mounted from app/dubbo-demo-web 
fddd8887b725: Mounted from base/jre8 
42052a19230c: Mounted from base/jre8 
8d4d1ab5ff74: Mounted from base/jre8 
v8.5.46: digest: sha256:407c6abd7c4fa5119376efa71090b49637d7a52ef2fc202f4019ab4c91f6dc50 size: 2409
```
## 改造dubbo-demo-web项目
略，直接使用tomcat分支

## 新建Jenkins的pipeline
### 配置New job
- 使用admin登录
- New Item
- create new jobs
- Enter an item name
> tomcat-demo

- Pipeline -> OK
- Discard old builds
> Days to keep builds : 3
> Max # of builds to keep : 30

- This project is parameterized
1. Add Parameter -> String Parameter
> Name : app_name
> Default Value : 
> Description : project name. e.g: dubbo-demo-web

2. Add Parameter -> String Parameter
> Name : image_name
> Default Value : 
> Description : project docker image name. e.g:  app/dubbo-demo-web

3. Add Parameter -> String Parameter
> Name : git_repo
> Default Value : 
> Description : project git repository. e.g: git@gitee.com:stanleywang/dubbo-demo-web.git

4. Add Parameter -> String Parameter
> Name : git_ver
> Default Value : tomcat
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
> Default Value : ./dubbo-client/target
> Description : the relative path of target file such as .jar or .war package. e.g: ./dubbo-client/target

8. Add Parameter -> String Parameter
> Name : mvn_cmd
> Default Value : mvn clean package -Dmaven.test.skip=true
> Description : maven command. e.g: mvn clean package -e -q -Dmaven.test.skip=true

9. Add Parameter -> Choice Parameter
> Name : base_image
> Default Value : 
> - base/tomcat:v7.0.94
> - base/tomcat:v8.5.46
> - base/tomcat:v9.0.17
> Description : project base image list in harbor.od.com.

10. Add Parameter -> Choice Parameter
> Name : maven
> Default Value : 
> - 3.6.1-8u212
> - 3.2.5-6u025
> - 2.2.1-6u025
> Description : different maven edition.

11. Add Parameter -> String Parameter
> Name : root_url
> Default Value : ROOT
> Description : webapp dir.

![jenkins新增流水线](/images/jenkins-2projects.png "jenkins新增流水线")

### Pipeline Script
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
    stage('unzip') { //unzip  target/*.war -c target/project_dir
      steps {
        sh "cd ${params.app_name}/${env.BUILD_NUMBER} && cd ${params.target_dir} && mkdir project_dir && unzip *.war -d ./project_dir"
      }
    }
    stage('image') { //build image and push to registry
      steps {
        writeFile file: "${params.app_name}/${env.BUILD_NUMBER}/Dockerfile", text: """FROM harbor.od.com/${params.base_image}
ADD ${params.target_dir}/project_dir /opt/tomcat/webapps/${params.root_url}"""
        sh "cd  ${params.app_name}/${env.BUILD_NUMBER} && docker build -t harbor.od.com/${params.image_name}:${params.git_ver}_${params.add_tag} . && docker push harbor.od.com/${params.image_name}:${params.git_ver}_${params.add_tag}"
      }
    }
  }
}
```

## 构建应用镜像
使用Jenkins进行CI，并查看harbor仓库
![jenkins构建](/images/jenkins-tomcat.png "jenkins构建")
![harbor仓库](/images/harbor-tomcat.png "harbor仓库")

## 准备k8s的资源配置清单
不再需要单独准备资源配置清单

## 应用资源配置清单
k8s的dashboard上直接修改image的值为jenkins打包出来的镜像
文档里的例子是：`harbor.od.com/app/dubbo-demo-web:tomcat_190119_1920`

## 浏览器访问
http://demo.od.com?hello=wangdao

## 检查tomcat运行情况
任意一台运算节点主机上：
```
[root@hdss7-22 ~]# kubectl get pods -n app
NAME                                                    READY     STATUS    RESTARTS   AGE
dubbo-demo-consumer-v025-htfx8                          2/2       Running   0          1h
[root@hdss7-22 ~]# kubectl exec -ti dubbo-demo-consumer-v025-htfx8 -n app bash
dubbo-demo-consumer-v025-htfx8:/opt/tomcat#  ls -lsr logs/
total 16
-rw-r----- 1 root root  435 Jan 19 19:26 catalina.2019-01-19.log
-rw-r----- 1 root root  629 Jan 19 19:26 localhost.2019-01-19.log
-rw-r----- 1 root root  249 Jan 15 19:27 localhost_access_log.2019-01-19.txt
-rw-r----- 1 root root 7355 Jan 15 19:28 stdout.log
```

# 部署ElasticSearch
[官网](https://www.elastic.co/)
[官方github地址](https://github.com/elastic/elasticsearch)
[下载地址](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.8.3.tar.gz)
`HDSS7-12.host.com`上：
## 安装
```pwd /opt/src
[root@hdss7-12 src]# ls -l|grep elasticsearch-6.8.3.tar.gz
-rw-r--r-- 1 root root  72262257 Jan 30 15:18 elasticsearch-6.8.3.tar.gz
[root@hdss7-12 src]# tar xf elasticsearch-6.8.3.tar.gz -C /opt
[root@hdss7-12 src]# ln -s /opt/elasticsearch-6.8.3/ /opt/elasticsearch
```
## 配置
### elasticsearch.yml

```
[root@hdss7-12 src]# mkdir -p /data/elasticsearch/{data,logs}
[root@hdss7-12 elasticsearch]# vi config/elasticsearch.yml
cluster.name: es.od.com
node.name: hdss7-12.host.com
path.data: /data/elasticsearch/data
path.logs: /data/elasticsearch/logs
bootstrap.memory_lock: true
network.host: 10.4.7.12
http.port: 9200
```
### jvm.options
```
[root@hdss7-12 elasticsearch]# vi config/jvm.options
-Xms512m
-Xmx512m
```
### 创建普通用户
```
[root@hdss7-12 elasticsearch]# useradd -s /bin/bash -M es
[root@hdss7-12 elasticsearch]# chown -R es.es /opt/elasticsearch-6.8.3
[root@hdss7-12 elasticsearch]# chown -R es.es /data/elasticsearch
```

### 文件描述符
```vi /etc/security/limits.d/es.conf
es hard nofile 65536
es soft fsize unlimited
es hard memlock unlimited
es soft memlock unlimited
```

### 调整内核参数
```
[root@hdss7-12 elasticsearch]# sysctl -w vm.max_map_count=262144
or
[root@hdss7-12 elasticsearch]# echo "vm.max_map_count=262144" >> /etc/sysctl.conf
[root@hdss7-12 elasticsearch]# sysctl -p
```

## 启动
```
[root@hdss7-12 elasticsearch]# su - es
su: warning: cannot change directory to /home/es: No such file or directory
-bash-4.2$ /opt/elasticsearch/bin/elasticsearch -d
[1] 8714
[root@hdss7-12 elasticsearch]# ps -ef|grep elastic|grep -v grep
es        8714     1 58 10:29 pts/0    00:00:19 /usr/java/jdk/bin/java -Xms512m -Xmx512m -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -XX:+AlwaysPreTouch -server -Xss1m -Djava.awt.headless=true -Dfile.encoding=UTF-8 -Djna.nosys=true -Djdk.io.permissionsUseCanonicalPath=true -Dio.netty.noUnsafe=true -Dio.netty.noKeySetOptimization=true -Dio.netty.recycler.maxCapacityPerThread=0 -Dlog4j.shutdownHookEnabled=false -Dlog4j2.disable.jmx=true -Dlog4j.skipJansi=true -XX:+HeapDumpOnOutOfMemoryError -Des.path.home=/opt/elasticsearch -cp /opt/elasticsearch/lib/* org.elasticsearch.bootstrap.Elasticsearch
```
## 调整ES日志模板
```
[root@hdss7-12 elasticsearch]# curl -H "Content-Type:application/json" -XPUT http://10.4.7.12:9200/_template/k8s -d '{
  "template" : "k8s*",
  "index_patterns": ["k8s*"],  
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 0
  }
}'
```

# 部署kafka
[官网](http://kafka.apache.org/)
[官方github地址](https://github.com/apache/kafka)
[下载地址](http://mirrors.tuna.tsinghua.edu.cn/apache/kafka/2.2.0/kafka_2.12-2.2.0.tgz)
`HDSS7-11.host.com`上：
## 安装
```pwd /opt/src
[root@hdss7-11 src]# ls -l|grep kafka
-rw-r--r-- 1 root root  57028557 Mar 23 08:57 kafka_2.12-2.2.0.tgz
[root@hdss7-11 src]# tar xf kafka_2.12-2.2.0.tgz -C /opt
[root@hdss7-11 src]# ln -s /opt/kafka_2.12-2.2.0/ /opt/kafka
```
## 配置
```vi /opt/kafka/config/server.properties
log.dirs=/data/kafka/logs
zookeeper.connect=localhost:2181
log.flush.interval.messages=10000
log.flush.interval.ms=1000
delete.topic.enable=true
host.name=hdss7-11.host.com
```
## 启动
```pwd /opt/kafka
[root@hdss7-11 kafka]# bin/kafka-server-start.sh -daemon config/server.properties
[root@hdss7-11 kafka]# netstat -luntp|grep 9092
tcp6       0      0 10.4.7.12:9092         :::*                    LISTEN      17543/java
```

# 部署kafka-manager
[官方github地址](https://github.com/yahoo/kafka-manager)
[源码下载地址](https://github.com/yahoo/kafka-manager/archive/2.0.0.2.tar.gz)
运维主机`HDSS7-200.host.com`上：
## 方法一：自定义构建
### 准备Dockerfile
```vi /data/dockerfile/kafka-manager/Dockerfile
FROM hseeberger/scala-sbt

ENV ZK_HOSTS=10.4.7.11:2181 \
     KM_VERSION=2.0.0.2

RUN mkdir -p /tmp && \
    cd /tmp && \
    wget https://github.com/yahoo/kafka-manager/archive/${KM_VERSION}.tar.gz && \
    tar xxf ${KM_VERSION}.tar.gz && \
    cd /tmp/kafka-manager-${KM_VERSION} && \
    sbt clean dist && \
    unzip  -d / ./target/universal/kafka-manager-${KM_VERSION}.zip && \
    rm -fr /tmp/${KM_VERSION} /tmp/kafka-manager-${KM_VERSION}

WORKDIR /kafka-manager-${KM_VERSION}

EXPOSE 9000
ENTRYPOINT ["./bin/kafka-manager","-Dconfig.file=conf/application.conf"]
```
### 制作docker镜像
```pwd /data/dockerfile/kafka-manager
[root@hdss7-200 kafka-manager]# docker build . -t harbor.od.com/infra/kafka-manager:v2.0.0.2
(漫长的过程)
```

## 方法二：直接下载docker镜像
[镜像下载地址](https://hub.docker.com/r/sheepkiller/kafka-manager/tags)
```
[root@hdss7-200 ~]# docker pull sheepkiller/kafka-manager:stable
[root@hdss7-200 ~]# docker tag 34627743836f harbor.od.com/infra/kafka-manager:stable
[root@hdss7-200 ~]# docker push harbor.od.com/infra/kafka-manager:stable
docker push harbor.od.com/infra/kafka-manager:stable
The push refers to a repository [harbor.od.com/infra/kafka]
ef97dbc2670b: Pushed 
ec01aa005e59: Pushed 
de05a1bdf878: Pushed 
9c553f3feafd: Pushed 
581533427a4f: Pushed 
f6b229974fdd: Pushed 
ae150883d6e2: Pushed 
3df7be729841: Pushed 
f231cc200afe: Pushed 
9752c15164a8: Pushed 
9ab7eda5c826: Pushed 
402964b3d72e: Pushed 
6b3f8ebf864c: Pushed 
stable: digest: sha256:57fd46a3751284818f1bc6c0fdf097250bc0feed03e77135fb8b0a93aa8c6cc7 size: 3056
```

## 准备资源配置清单
{% tabs kafka-manager%}
<!-- tab Deployment -->
vi /data/k8s-yaml/kafka-manager/dp.yaml
{% code %}
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: kafka-manager
  namespace: infra
  labels: 
    name: kafka-manager
spec:
  replicas: 1
  selector:
    matchLabels: 
      app: kafka-manager
  strategy:
    type: RollingUpdate
    rollingUpdate: 
      maxUnavailable: 1
      maxSurge: 1
  revisionHistoryLimit: 7
  progressDeadlineSeconds: 600
  template:
    metadata:
      labels: 
        app: kafka-manager
    spec:
      containers:
      - name: kafka-manager
        image: harbor.od.com/infra/kafka-manager:v2.0.0.2
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9000
          protocol: TCP
        env:
        - name: ZK_HOSTS
          value: zk1.od.com:2181
        - name: APPLICATION_SECRET
          value: letmein
      imagePullSecrets:
      - name: harbor
      terminationGracePeriodSeconds: 30
      securityContext: 
        runAsUser: 0
{% endcode %}
<!-- endtab -->
<!-- tab Service-->
vi /data/k8s-yaml/kafka-manager/svc.yaml
{% code %}
kind: Service
apiVersion: v1
metadata: 
  name: kafka-manager
  namespace: infra
spec:
  ports:
  - protocol: TCP
    port: 9000
    targetPort: 9000
  selector: 
    app: kafka-manager
{% endcode %}
<!-- endtab -->
<!-- tab Ingress -->
vi /data/k8s-yaml/kafka-manager/ingress.yaml
{% code %}
kind: Ingress
apiVersion: extensions/v1beta1
metadata: 
  name: kafka-manager
  namespace: infra
spec:
  rules:
  - host: km.od.com
    http:
      paths:
      - path: /
        backend: 
          serviceName: kafka-manager
          servicePort: 9000
{% endcode %}
<!-- endtab -->
{% endtabs %}
## 应用资源配置清单
任意一台运算节点上：
```
[root@hdss7-21 kafka-manager]# kubectl apply -f http://k8s-yaml.od.com/kafka-manager/dp.yaml 
deployment.extensions/kafka-manager created
[root@hdss7-21 kafka-manager]# kubectl apply -f http://k8s-yaml.od.com/kafka-manager/svc.yaml 
service/kafka-manager created
[root@hdss7-21 kafka-manager]# kubectl apply -f http://k8s-yaml.od.com/kafka-manager/ingress.yaml 
ingress.extensions/kafka-manager created
```
## 解析域名
`HDSS7-11.host.com`上
```vi /var/named/od.com.zone
km                 A    10.4.7.10
```

## 浏览器访问
http://km.od.com

![kafka-manager](/images/kafka-manager.png "kafka-manager")

# 部署filebeat
[官方下载地址](https://www.elastic.co/downloads/beats/filebeat)
运维主机`HDSS7-200.host.com`上：
## 制作docker镜像
### 准备Dockerfile
{% tabs filebeat-dockerfile%}
<!-- tab Dockerfile -->
vi /data/dockerfile/filebeat/Dockerfile
{% code %}
FROM debian:jessie

ENV FILEBEAT_VERSION=7.4.0 \
    FILEBEAT_SHA1=c63bb1e16f7f85f71568041c78f11b57de58d497ba733e398fa4b2d071270a86dbab19d5cb35da5d3579f35cb5b5f3c46e6e08cdf840afb7c347777aae5c4e11

RUN set -x && \
  apt-get update && \
  apt-get install -y wget && \
  wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-${FILEBEAT_VERSION}-linux-x86_64.tar.gz -O /opt/filebeat.tar.gz && \
  cd /opt && \
  echo "${FILEBEAT_SHA1}  filebeat.tar.gz" | sha512sum -c - && \
  tar xzvf filebeat.tar.gz && \
  cd filebeat-* && \
  cp filebeat /bin && \
  cd /opt && \
  rm -rf filebeat* && \
  apt-get purge -y wget && \
  apt-get autoremove -y && \
  apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

COPY docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]
{% endcode %}
<!-- endtab -->
<!-- tab Entrypoint -->
vi /data/dockerfile/filebeat/docker-entrypoint.sh
{% code %}
#!/bin/bash

ENV=${ENV:-"test"}
PROJ_NAME=${PROJ_NAME:-"no-define"}
MULTILINE=${MULTILINE:-"^\d{2}"}

cat > /etc/filebeat.yaml << EOF
filebeat.inputs:
- type: log
  fields_under_root: true
  fields:
    topic: logm-${PROJ_NAME}
  paths:
    - /logm/*.log
    - /logm/*/*.log
    - /logm/*/*/*.log
    - /logm/*/*/*/*.log
    - /logm/*/*/*/*/*.log
  scan_frequency: 120s
  max_bytes: 10485760
  multiline.pattern: '$MULTILINE'
  multiline.negate: true
  multiline.match: after
  multiline.max_lines: 100
- type: log
  fields_under_root: true
  fields:
    topic: logu-${PROJ_NAME}
  paths:
    - /logu/*.log
    - /logu/*/*.log
    - /logu/*/*/*.log
    - /logu/*/*/*/*.log
    - /logu/*/*/*/*/*.log
    - /logu/*/*/*/*/*/*.log
output.kafka:
  hosts: ["10.4.7.11:9092"]
  topic: k8s-fb-$ENV-%{[topic]}
  version: 2.0.0
  required_acks: 0
  max_message_bytes: 10485760
EOF

set -xe

# If user don't provide any command
# Run filebeat
if [[ "$1" == "" ]]; then
     exec filebeat  -c /etc/filebeat.yaml 
else
    # Else allow the user to run arbitrarily commands like bash
    exec "$@"
fi

{% endcode %}
<!-- endtab -->
{% endtabs %}

### 制作镜像
```pwd /data/dockerfile/filebeat
[root@hdss7-200 filebeat]# docker build . -t harbor.od.com/infra/filebeat:v7.4.0
...
+ apt-get autoremove -y
Reading package lists...
Building dependency tree...
Reading state information...
The following packages will be REMOVED:
  ca-certificates libicu52 libidn11 libpsl0 libssl1.0.0 openssl
0 upgraded, 0 newly installed, 6 to remove and 6 not upgraded.
After this operation, 33.6 MB disk space will be freed.
(Reading database ... 7962 files and directories currently installed.)
Removing ca-certificates (20141019+deb8u4) ...
Removing dangling symlinks from /etc/ssl/certs... done.
Removing libpsl0:amd64 (0.5.1-1) ...
Removing libicu52:amd64 (52.1-8+deb8u7) ...
Removing libidn11:amd64 (1.29-1+deb8u3) ...
Removing openssl (1.0.1t-1+deb8u11) ...
Removing libssl1.0.0:amd64 (1.0.1t-1+deb8u11) ...
Processing triggers for libc-bin (2.19-18+deb8u10) ...
+ apt-get clean
+ rm -rf /var/lib/apt/lists/deb.debian.org_debian_dists_jessie_Release /var/lib/apt/lists/deb.debian.org_debian_dists_jessie_Release.gpg /var/lib/apt/lists/deb.debian.org_debian_dists_jessie_main_binary-amd64_Packages.gz /var/lib/apt/lists/lock /var/lib/apt/lists/partial /var/lib/apt/lists/security.debian.org_debian-security_dists_jessie_updates_InRelease /var/lib/apt/lists/security.debian.org_debian-security_dists_jessie_updates_main_binary-amd64_Packages.gz /tmp/* /var/tmp/*
 ---> a78678659f2c
Removing intermediate container 2cfff130c15c
Step 4 : COPY docker-entrypoint.sh /
 ---> 62dc7fe5a98f
Removing intermediate container 87e271482593
Step 5 : ENTRYPOINT /docker-entrypoint.sh
 ---> Running in d367b6e3bb5a
 ---> 23c8fbdc088a
Removing intermediate container d367b6e3bb5a
Successfully built 23c8fbdc088a
[root@hdss7-200 filebeat]# docker tag 23c8fbdc088a harbor.od.com/infra/filebeat:v7.4.0
[root@hdss7-200 filebeat]# docker push !$
docker push harbor.od.com/infra/filebeat:v7.4.0
The push refers to a repository [harbor.od.com/infra/filebeat]
6a765e653161: Pushed 
8e89ae5c6fc2: Pushed 
9abb3997a540: Pushed 
v7.4.0: digest: sha256:c35d7cdba29d8555388ad41ac2fc1b063ed9ec488082e33b5d0e91864b3bb35c size: 948
```
## 修改资源配置清单
**使用dubbo-demo-consumer的Tomcat版镜像**
```vi /data/k8s-yaml/dubbo-demo-consumer/dp.yaml
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: dubbo-demo-consumer
  namespace: test
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
        image: harbor.od.com/app/dubbo-demo-web:tomcat
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          protocol: TCP
        env:
        - name: C_OPTS
          value: -Denv=fat -Dapollo.meta=http://config.od.com
        volumeMounts:
        - mountPath: /opt/tomcat/logs
          name: logm
      - name: filebeat
        image: harbor.od.com/infra/filebeat:v7.4.0
        imagePullPolicy: IfNotPresent
        env:
        - name: ENV
          value: test
        - name: PROJ_NAME
          value: dubbo-demo-web
        volumeMounts:
        - mountPath: /logm
          name: logm
      volumes:
      - emptyDir: {}
        name: logm
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
## 浏览器访问http://km.od.com
看到kafaka-manager里，topic打进来，即为成功。
![kafka-topic](/images/kafka-topic.png "kafka-topic")

## 验证数据
```pwd /opt/kafka/bin
./kafka-console-consumer.sh --bootstrap-server 10.4.7.11:9092 --topic k8s-fb-test-logm-dubbo-demo-web --from-beginning
```

# 部署logstash
运维主机`HDSS7-200.host.com`上：

## 选版本
[logstash选型](https://www.elastic.co/support/matrix)
![compatibility](/images/es-logstash.png "compatibility")

## 准备docker镜像
- 下载官方镜像

```
[root@hdss7-200 ~]# docker pull logstash:6.8.3
6.8.3: Pulling from library/logstash
8ba884070f61: Pull complete 
063405b57b96: Pull complete 
0a0115b8825c: Pull complete 
825b124900cb: Pull complete 
ed736d9e3c41: Pull complete 
6efb3934697a: Pull complete 
c0a3a0c8ebc2: Pull complete 
f55d8adfbc3d: Pull complete 
3a6c56fbbef8: Pull complete 
ca45dc8946a2: Pull complete 
8b852a079ea9: Pull complete 
Digest: sha256:46eaff19af5e14edd9b78e1d5bf16f6abcd9ad50e0338cbaf3024f2aadb2903b
Status: Downloaded newer image for logstash:6.8.3

[root@hdss7-200 ~]# docker tag 857d9a1f8221 harbor.od.com/public/logstash:v6.8.3

[root@hdss7-200 ~]# docker push harbor.od.com/public/logstash:v6.8.3
docker push harbor.od.com/public/logstash:v6.8.3
The push refers to a repository [harbor.od.com/public/logstash]
c2b00f70cade: Mounted from public/filebeat 
9a2c2851596d: Mounted from public/filebeat 
86564f3ca0e2: Mounted from public/filebeat 
e9bc8faf823a: Mounted from public/filebeat 
3f3bcfa85067: Mounted from public/filebeat 
65d3b56913bd: Mounted from public/filebeat 
9b48a60607ee: Mounted from public/filebeat 
df859db41dd0: Mounted from public/filebeat 
1cbe912d7c00: Mounted from public/filebeat 
ab5fbaca7e70: Mounted from public/filebeat 
d69483a6face: Mounted from public/filebeat 
v6.8.3: digest: sha256:6aacf97dfbcc5c64c2f1a12f43ee48a8dadb98657b9b8d4149d0fee0ec18be81 size: 2823
```
- 自定义Dockerfile

{% tabs logstash %}
<!-- tab Dockerfile -->
{% code %}
From harbor.od.com/public/logstash:v6.8.3
ADD logstash.yml /usr/share/logstash/config
{% endcode %}
<!-- endtab -->
<!-- tab logstash.yml-->
{% code %}
http.host: "0.0.0.0"
path.config: /etc/logstash
xpack.monitoring.enabled: false
{% endcode %}
<!-- endtab -->
{% endtabs %}

- 创建自定义镜像

```pwd /data/dockerfile/logstash
[root@hdss7-200 logstash]# docker build . -t harbor.od.com/infra/logstash:v6.8.3
Sending build context to Docker daemon 249.3 MB
Step 1 : FROM harbor.od.com/public/logstash:v6.8.3
v6.8.3: Pulling from public/logstash
e60be144a6a5: Pull complete 
6292c2b22d35: Pull complete 
1bb4586d90e7: Pull complete 
3f11f6d21132: Pull complete 
f0aaeeafebd3: Pull complete 
38c751a91f0f: Pull complete 
b80e2bad9681: Pull complete 
e3c97754ddde: Pull complete 
840f2d82a9fb: Pull complete 
32063a9aaefb: Pull complete 
e87b22bf50f5: Pull complete 
Digest: sha256:6aacf97dfbcc5c64c2f1a12f43ee48a8dadb98657b9b8d4149d0fee0ec18be81
Status: Downloaded newer image for harbor.od.com/public/logstash:v6.8.3
 ---> 857d9a1f8221
Step 2 : ADD logstash.yml /usr/share/logstash/config
 ---> 2ad32d3f5fef
Removing intermediate container 1d3a1488c1b7
Successfully built 2ad32d3f5fef
[root@hdss7-200 logstash]# docker push harbor.od.com/infra/logstash:v6.8.3
```

## 启动docker镜像
- 创建配置

```vi /etc/logstash/logstash-test.conf
input {
  kafka {
    bootstrap_servers => "10.4.7.11:9092"
    client_id => "10.4.7.200"
    consumer_threads => 4
    group_id => "k8s_test"
    topics_pattern => "k8s-fb-test-.*"
  }
}

filter {
  json {
    source => "message"
  }
}

output {
  elasticsearch {
    hosts => ["10.4.7.12:9200"]
    index => "k8s-test-%{+YYYY.MM}"
  }
}
```

- 启动logstash镜像

```
[root@hdss7-200 ~]# docker run -d --name logstash-test -v /etc/logstash:/etc/logstash harbor.od.com/infra/logstash:v6.8.3 -f /etc/logstash/logstash-test.conf
[root@hdss7-200 ~]# docker ps -a|grep logstash
a5dcf56faa9a        harbor.od.com/infra/logstash:v6.8.3                                                                                "/usr/local/bin/docke"   8 seconds ago       Up 6 seconds                5044/tcp, 9600/tcp           jovial_swanson
```

- 验证ElasticSearch里的索引

```
[root@hdss7-200 ~]# curl http://10.4.7.12:9200/_cat/indices?v
health status index            uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   k8s-test-2019.04 H3MY9d8WSbqQ6uL0DFhenQ   5   0         55            0    249.9kb        249.9kb
```

# 部署Kibana
运维主机`HDSS7-200.host.com`上：
## 准备docker镜像
[kibana官方镜像下载地址](https://hub.docker.com/_/kibana?tab=tags)
```
[root@hdss7-200 ~]# docker pull kibana:6.8.3
6.8.3: Pulling from library/kibana

8014725ed6f5: Pull complete 
19b590251e94: Pull complete 
7fa3e54c0b5a: Pull complete 
60ee9811b356: Pull complete 
d18bafa420f4: Pull complete 
8cee55751899: Pull complete 
c395be470eb2: Pull complete 
Digest: sha256:71f776596244877597fd679b2fa6fb0f1fa9c5b11388199997781d1ce77b73b1
Status: Downloaded newer image for kibana:6.8.3
[root@hdss7-200 ~]# docker tag 62079cf74c23 harbor.od.com/infra/kibana:v6.8.3
[root@hdss7-200 ~]# docker push harbor.od.com/infra/kibana:v6.8.3
docker push harbor.od.com/infra/kibana:v6.8.3
The push refers to a repository [harbor.od.com/infra/kibana]
be94745c5390: Pushed 
652dcbd52cdd: Pushed 
a3508a095ca7: Pushed 
56d52080e2fe: Pushed 
dbce28d91bf0: Pushed 
dcddc432ebdf: Pushed 
3e466685bf43: Pushed 
d69483a6face: Mounted from infra/logstash 
v6.8.3: digest: sha256:17dd243d6cc4e572f74f3de83eafc981e54c1710f8fe2d0bf74357b28bddaf08 size: 1991
```
## 解析域名
`HDSS7-11.host.com`上
```vi /var/named/od.com.zone
kibana             A    10.4.7.10
```
## 准备资源配置清单
{% tabs kibana %}
<!-- tab Deployment -->
vi /data/k8s-yaml/kibana/dp.yaml
{% code %}
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: kibana
  namespace: infra
  labels: 
    name: kibana
spec:
  replicas: 1
  selector:
    matchLabels: 
      name: kibana
  template:
    metadata:
      labels: 
        app: kibana
        name: kibana
    spec:
      containers:
      - name: kibana
        image: harbor.od.com/infra/kibana:v6.8.3
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5601
          protocol: TCP
        env:
        - name: ELASTICSEARCH_URL
          value: http://10.4.7.12:9200
      imagePullSecrets:
      - name: harbor
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
<!-- tab Service -->
vi /data/k8s-yaml/kibana/svc.yaml
{% code %}
kind: Service
apiVersion: v1
metadata: 
  name: kibana
  namespace: infra
spec:
  ports:
  - protocol: TCP
    port: 5601
    targetPort: 5601
  selector: 
    app: kibana
{% endcode %}
<!-- endtab -->
<!-- tab Ingress -->
vi /data/k8s-yaml/kibana/ingress.yaml
{% code %}
kind: Ingress
apiVersion: extensions/v1beta1
metadata: 
  name: kibana
  namespace: infra
spec:
  rules:
  - host: kibana.od.com
    http:
      paths:
      - path: /
        backend: 
          serviceName: kibana
          servicePort: 5601
{% endcode %}
<!-- endtab -->
{% endtabs %}
## 应用资源配置清单
任意运算节点上：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/kibana/dp.yaml
deployment.extensions/kibana created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/kibana/svc.yaml
service/kibana created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/kibana/ingress.yaml
ingress.extensions/kibana created
```
## 浏览器访问
http://kibana.od.com
![kibana页面](/images/kibana.png "kibana页面")

# kibana的使用
![kibana用法](/images/kibana-usage.png "kibana用法")
## 选择区域
- @timestamp
> 对应日志的时间戳

- log.file.path
> 对应日志文件名

- message
> 对应日志内容

## 时间选择器
- 选择日志时间
> 快速时间
> 绝对时间
> 相对时间

## 环境选择器
- 选择对应环境的日志
> k8s-test-*
> k8s-prod-*

## 项目选择器
- 对应filebeat的PROJ_NAME值
- Add a fillter
- topic is ${PROJ_NAME}
> dubbo-demo-service
> dubbo-demo-web

## 关键字选择器
- exception
- error
