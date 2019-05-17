title: 实验文档3：在kubernetes集群里集成Apollo配置中心
author: Stanley Wang
categories: Kubernetes容器云技术专题
date: 2019-1-18 20:12:56
---
- - -
{% cq %}欢迎加入王导的VIP学习qq群：==>[<font color="FF7F50">932194668</font>](http://shang.qq.com/wpa/qunwpa?idkey=78869fddc5a661acb0639315eb52997c108de6625df5f0ee2f0372f176a032a6)<=={% endcq %}
- - -
# 使用ConfigMap管理应用配置
## 拆分环境
主机名|角色|ip
-|-|-
HDSS7-11.host.com|zk1.od.com(Test环境)|10.4.7.11
HDSS7-12.host.com|zk2.od.com(Prod环境)|10.4.7.12

## 重配zookeeper
`HDSS7-11.host.com`上：
```vi /opt/zookeeper/conf/zoo.cfg
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/data/zookeeper/data
dataLogDir=/data/zookeeper/logs
clientPort=2181
```
`HDSS7-12.host.com`上：
```vi /opt/zookeeper/conf/zoo.cfg
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/data/zookeeper/data
dataLogDir=/data/zookeeper/logs
clientPort=2181
```
重启zk(删除数据文件)
```
[root@hdss7-11 ~]# /opt/zookeeper/bin/zkServer.sh restart && /opt/zookeeper/bin/zkServer.sh status
[root@hdss7-12 ~]# /opt/zookeeper/bin/zkServer.sh restart && /opt/zookeeper/bin/zkServer.sh status
[root@hdss7-21 ~]# /opt/zookeeper/bin/zkServer.sh stop
```

## 准备资源配置清单(dubbo-monitor)
在运维主机`HDSS7-200.host.com`上：
{% tabs dubbo-monitor%}
<!-- tab ConfigMap -->
vi /data/k8s-yaml/dubbo-monitor/configmap.yaml
{% code %}
apiVersion: v1
kind: ConfigMap
metadata:
  name: dubbo-monitor-cm
  namespace: infra
data:
  dubbo.properties: |
    dubbo.container=log4j,spring,registry,jetty
    dubbo.application.name=simple-monitor
    dubbo.application.owner=
    dubbo.registry.address=zookeeper://zk1.od.com:2181
    dubbo.protocol.port=20880
    dubbo.jetty.port=8080
    dubbo.jetty.directory=/dubbo-monitor-simple/monitor
    dubbo.charts.directory=/dubbo-monitor-simple/charts
    dubbo.statistics.directory=/dubbo-monitor-simple/statistics
    dubbo.log4j.file=/dubbo-monitor-simple/logs/dubbo-monitor.log
    dubbo.log4j.level=WARN
{% endcode %}
<!-- endtab -->
<!-- tab Deployment-->
vi /data/k8s-yaml/dubbo-monitor/deployment.yaml
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
        volumeMounts:
          - name: configmap-volume
            mountPath: /dubbo-monitor-simple/conf
      volumes:
        - name: configmap-volume
          configMap:
            name: dubbo-monitor-cm
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
{% endtabs %}

## 应用资源配置清单
在任意一台k8s运算节点执行：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dubbo-monitor/configmap.yaml
configmap/dubbo-monitor-cm created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dubbo-monitor/deployment.yaml
deployment.extensions/dubbo-monitor configured
```

## 重新发版，修改dubbo项目的配置文件
### 修改项目源代码
- duboo-demo-service
```vi dubbo-server/src/main/java/config.properties
dubbo.registry=zookeeper://zk1.od.com:2181
dubbo.port=28080
```
- dubbo-demo-web
```vi dubbo-client/src/main/java/config.properties
dubbo.registry=zookeeper://zk1.od.com:2181
```
### 使用Jenkins进行CI
略

### 修改/应用资源配置清单
k8s的dashboard上，修改deployment使用的容器版本，提交应用

## 验证configmap的配置
在K8S的dashboard上，修改dubbo-monitor的configmap配置为不同的zk，重启POD，浏览器打开http://dubbo-monitor.od.com 观察效果

# 交付Apollo至Kubernetes集群
## Apollo简介
Apollo（阿波罗）是携程框架部门研发的分布式配置中心，能够集中化管理应用不同环境、不同集群的配置，配置修改后能够实时推送到应用端，并且具备规范的权限、流程治理等特性，适用于微服务配置管理场景。

### 官方GitHub地址
[Apollo官方地址](https://github.com/ctripcorp/apollo)
[官方release包](https://github.com/ctripcorp/apollo/releases)

### 基础架构
![apollo基础架构](/images/apollo.png "apollo基础架构")

### 简化模型
![apollo简化架构](/images/apollo-simple.png "apollo简化架构")

## 交付apollo-configservice
### 准备软件包
在运维主机`HDSS7-200.host.com`上：
[下载官方release包](https://github.com/ctripcorp/apollo/releases/download/v1.3.0/apollo-configservice-1.3.0-github.zip)
```pwd /opt/src
[root@hdss7-200 src]# ls -l|grep apollo
-rw-r--r-- 1 root root 52713404 Feb 16 23:29 apollo-configservice-1.3.0-github.zip
[root@hdss7-200 src]# mkdir /data/dockerfile/apollo-configservice && unzip -o apollo-configservice-1.3.0-github.zip -d /data/dockerfile/apollo-configservice
Archive:  apollo-configservice-1.3.0-github.zip
   creating: /data/dockerfile/apollo-configservice/scripts/
  inflating: /data/dockerfile/apollo-configservice/config/application-github.properties  
  inflating: /data/dockerfile/apollo-configservice/scripts/shutdown.sh  
  inflating: /data/dockerfile/apollo-configservice/apollo-configservice-1.3.0-sources.jar  
  inflating: /data/dockerfile/apollo-configservice/scripts/startup.sh  
  inflating: /data/dockerfile/apollo-configservice/config/app.properties  
  inflating: /data/dockerfile/apollo-configservice/apollo-configservice-1.3.0.jar  
  inflating: /data/dockerfile/apollo-configservice/apollo-configservice.conf
```
### 执行数据库脚本
在数据库主机`HDSS7-11.host.com`上：
[数据库脚本地址](https://raw.githubusercontent.com/ctripcorp/apollo/master/scripts/db/migration/configdb/V1.0.0__initialization.sql)
```
[root@hdss7-11 ~]# mysql -uroot -p
mysql> create database ApolloConfigDB;
mysql> source ./apolloconfig.sql
```


### 数据库用户授权
```
mysql> grant INSERT,DELETE,UPDATE,SELECT on ApolloConfigDB.* to "apolloconfig"@"172.7.%" identified by "123456";
```

### 修改初始数据
```
mysql> update ApolloConfigDB.ServerConfig set ServerConfig.Value="http://config.od.com/eureka" where ServerConfig.Key="eureka.service.url";
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select * from ServerConfig\G
*************************** 1. row ***************************
                       Id: 1
                      Key: eureka.service.url
                  Cluster: default
                    Value: http://config.od.com/eureka
                  Comment: Eureka服务Url，多个service以英文逗号分隔
                IsDeleted:  
     DataChange_CreatedBy: default
   DataChange_CreatedTime: 2019-04-10 15:07:34
DataChange_LastModifiedBy: 
      DataChange_LastTime: 2019-04-11 16:28:57
```

### 制作Docker镜像
在运维主机`HDSS7-200.host.com`上：
- 配置数据库连接串
```pwd /data/dockerfile/apollo-configservice
[root@hdss7-200 apollo-configservice]# cat config/application-github.properties

```

- 更新startup.sh
```vi /data/dockerfile/apollo-configservice/scripts/startup.sh
#!/bin/bash
SERVICE_NAME=apollo-configservice
## Adjust log dir if necessary
LOG_DIR=/opt/logs/apollo-config-server
## Adjust server port if necessary
SERVER_PORT=8080
APOLLO_CONFIG_SERVICE_NAME=$(hostname -i)
SERVER_URL="http://${APOLLO_CONFIG_SERVICE_NAME}:${SERVER_PORT}"

## Adjust memory settings if necessary
#export JAVA_OPTS="-Xms6144m -Xmx6144m -Xss256k -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=384m -XX:NewSize=4096m -XX:MaxNewSize=4096m -XX:SurvivorRatio=8"

## Only uncomment the following when you are using server jvm
#export JAVA_OPTS="$JAVA_OPTS -server -XX:-ReduceInitialCardMarks"

########### The following is the same for configservice, adminservice, portal ###########
export JAVA_OPTS="$JAVA_OPTS -XX:ParallelGCThreads=4 -XX:MaxTenuringThreshold=9 -XX:+DisableExplicitGC -XX:+ScavengeBeforeFullGC -XX:SoftRefLRUPolicyMSPerMB=0 -XX:+ExplicitGCInvokesConcurrent -XX:+PrintGCDetails -XX:+HeapDumpOnOutOfMemoryError -XX:-OmitStackTraceInFastThrow -Duser.timezone=Asia/Shanghai -Dclient.encoding.override=UTF-8 -Dfile.encoding=UTF-8 -Djava.security.egd=file:/dev/./urandom"
export JAVA_OPTS="$JAVA_OPTS -Dserver.port=$SERVER_PORT -Dlogging.file=$LOG_DIR/$SERVICE_NAME.log -XX:HeapDumpPath=$LOG_DIR/HeapDumpOnOutOfMemoryError/"

# Find Java
if [[ -n "$JAVA_HOME" ]] && [[ -x "$JAVA_HOME/bin/java" ]]; then
    javaexe="$JAVA_HOME/bin/java"
elif type -p java > /dev/null 2>&1; then
    javaexe=$(type -p java)
elif [[ -x "/usr/bin/java" ]];  then
    javaexe="/usr/bin/java"
else
    echo "Unable to find Java"
    exit 1
fi

if [[ "$javaexe" ]]; then
    version=$("$javaexe" -version 2>&1 | awk -F '"' '/version/ {print $2}')
    version=$(echo "$version" | awk -F. '{printf("%03d%03d",$1,$2);}')
    # now version is of format 009003 (9.3.x)
    if [ $version -ge 011000 ]; then
        JAVA_OPTS="$JAVA_OPTS -Xlog:gc*:$LOG_DIR/gc.log:time,level,tags -Xlog:safepoint -Xlog:gc+heap=trace"
    elif [ $version -ge 010000 ]; then
        JAVA_OPTS="$JAVA_OPTS -Xlog:gc*:$LOG_DIR/gc.log:time,level,tags -Xlog:safepoint -Xlog:gc+heap=trace"
    elif [ $version -ge 009000 ]; then
        JAVA_OPTS="$JAVA_OPTS -Xlog:gc*:$LOG_DIR/gc.log:time,level,tags -Xlog:safepoint -Xlog:gc+heap=trace"
    else
        JAVA_OPTS="$JAVA_OPTS -XX:+UseParNewGC"
        JAVA_OPTS="$JAVA_OPTS -Xloggc:$LOG_DIR/gc.log -XX:+PrintGCDetails"
        JAVA_OPTS="$JAVA_OPTS -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=60 -XX:+CMSClassUnloadingEnabled -XX:+CMSParallelRemarkEnabled -XX:CMSFullGCsBeforeCompaction=9 -XX:+CMSClassUnloadingEnabled  -XX:+PrintGCDateStamps -XX:+PrintGCApplicationConcurrentTime -XX:+PrintHeapAtGC -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=5M"
    fi
fi

printf "$(date) ==== Starting ==== \n"

cd `dirname $0`/..
chmod 755 $SERVICE_NAME".jar"
./$SERVICE_NAME".jar" start

rc=$?;

if [[ $rc != 0 ]];
then
    echo "$(date) Failed to start $SERVICE_NAME.jar, return code: $rc"
    exit $rc;
fi

tail -f /dev/null
```

- 写Dockerfile
```vi /data/dockerfile/apollo-configservice/Dockerfile
FROM openjdk:8-jre-alpine

ENV VERSION 1.3.0

RUN echo "http://mirrors.aliyun.com/alpine/v3.8/main" > /etc/apk/repositories \
    && echo "http://mirrors.aliyun.com/alpine/v3.8/community" >> /etc/apk/repositories \
    && apk update upgrade \
    && apk add --no-cache procps unzip curl bash tzdata \
    && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && echo "Asia/Shanghai" > /etc/timezone

ADD apollo-configservice-${VERSION}.jar /apollo-configservice/apollo-configservice.jar
ADD config/ /apollo-configservice/config
ADD scripts/ /apollo-configservice/scripts

CMD ["/apollo-configservice/scripts/startup.sh"]
```
- 制作镜像并推送
```
[root@hdss7-200 apollo-configservice]# docker build . -t harbor.od.com/infra/apollo-configservice:v1.3.0
Sending build context to Docker daemon 111.9 MB
Step 1 : FROM openjdk:8-jre-alpine
 ---> ce8477c7d086
Step 2 : ENV VERSION 1.3.0
 ---> Running in 1b216dfe65f4
 ---> 02291afc5c51
Removing intermediate container 1b216dfe65f4
Step 3 : RUN echo "http://mirrors.aliyun.com/alpine/v3.8/main" > /etc/apk/repositories     && echo "http://mirrors.aliyun.com/alpine/v3.8/community" >> /etc/apk/repositories     && apk update upgrade     && apk add --no-cache procps unzip curl bash tzdata     && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime     && echo "Asia/Shanghai" > /etc/timezone
 ---> Running in ef3bf63405ba
fetch http://mirrors.aliyun.com/alpine/v3.8/main/x86_64/APKINDEX.tar.gz
fetch http://mirrors.aliyun.com/alpine/v3.8/community/x86_64/APKINDEX.tar.gz
v3.8.4-22-g92c8ab7e2e [http://mirrors.aliyun.com/alpine/v3.8/main]
v3.8.4-22-g92c8ab7e2e [http://mirrors.aliyun.com/alpine/v3.8/community]
OK: 9597 distinct packages available
fetch http://mirrors.aliyun.com/alpine/v3.8/main/x86_64/APKINDEX.tar.gz
fetch http://mirrors.aliyun.com/alpine/v3.8/community/x86_64/APKINDEX.tar.gz
(1/16) Installing ncurses-terminfo-base (6.1_p20180818-r1)
(2/16) Installing ncurses-terminfo (6.1_p20180818-r1)
(3/16) Installing ncurses-libs (6.1_p20180818-r1)
(4/16) Installing readline (7.0.003-r0)
(5/16) Installing bash (4.4.19-r1)
Executing bash-4.4.19-r1.post-install
(6/16) Installing libressl2.7-libcrypto (2.7.5-r0)
(7/16) Installing nghttp2-libs (1.32.0-r0)
(8/16) Installing libssh2 (1.8.2-r0)
(9/16) Installing libressl2.7-libssl (2.7.5-r0)
(10/16) Installing libcurl (7.61.1-r2)
(11/16) Installing curl (7.61.1-r2)
(12/16) Installing libintl (0.19.8.1-r2)
(13/16) Installing libproc (3.3.15-r0)
(14/16) Installing procps (3.3.15-r0)
(15/16) Installing tzdata (2018f-r0)
(16/16) Installing unzip (6.0-r4)
Executing busybox-1.29.3-r10.trigger
Executing ca-certificates-20190108-r0.trigger
OK: 100 MiB in 69 packages
 ---> bfcc40fdf2a2
Removing intermediate container ef3bf63405ba
Step 4 : ADD apollo-configservice-${VERSION}.jar /apollo-configservice/apollo-configservice.jar
 ---> 47446152080a
Removing intermediate container 535a15e62c53
Step 5 : ADD apollo-configservice.conf /apollo-configservice/apollo-configservice.conf
 ---> 8133e438abe6
Removing intermediate container 623a17ff715c
Step 6 : ADD config/ /apollo-configservice/config
 ---> 7c78e8a039c4
Removing intermediate container 5b5d21ee39ef
Step 7 : ADD scripts/ /apollo-configservice/scripts
 ---> 3c5e73da046e
Removing intermediate container 1517d8a51322
Step 8 : CMD /apollo-configservice/scripts/startup.sh
 ---> Running in 5cb6b42ba6ba
 ---> a498f9f1e57b
Removing intermediate container 5cb6b42ba6ba
Successfully built a498f9f1e57b

[root@hdss7-200 apollo-configservice]# docker push harbor.od.com/infra/apollo-configservice:v1.3.0
The push refers to a repository [harbor.od.com/infra/apollo-configservice]
25efb9a44683: Pushed 
b3572bb46247: Pushed 
e7994b936025: Pushed 
0ff1d078cbc4: Pushed 
ebfb473df5c2: Pushed 
aae5c057d1b6: Pushed 
dee6aef5c2b6: Pushed 
a464c54f93a9: Pushed 
v1.3.0: digest: sha256:6a8e4fdda58de0dfba9985ebbf91c4d6f46f5274983d2efa8853b03f4e45fa06 size: 1992
```

### 解析域名
DNS主机`HDSS7-11.host.com`上：
```vi /var/named/od.com.zone
mysql   60 IN A 10.4.7.11
config	60 IN A 10.4.7.10
```

### 准备资源配置清单
在运维主机`HDSS7-200.host.com`上
```pwd /data/k8s-yaml
[root@hdss7-200 k8s-yaml]# mkdir /data/k8s-yaml/apollo-configservice && cd /data/k8s-yaml/apollo-configservice
```
{% tabs apollo-configservice-yaml%}
<!-- tab Deployment -->
vi deployment.yaml
{% code %}
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: apollo-configservice
  namespace: infra
  labels: 
    name: apollo-configservice
spec:
  replicas: 1
  selector:
    matchLabels: 
      name: apollo-configservice
  template:
    metadata:
      labels: 
        app: apollo-configservice 
        name: apollo-configservice
    spec:
      volumes:
      - name: configmap-volume
        configMap:
          name: apollo-configservice-cm
      containers:
      - name: apollo-configservice
        image: harbor.od.com/infra/apollo-configservice:v1.3.0
        ports:
        - containerPort: 8080
          protocol: TCP
        volumeMounts:
        - name: configmap-volume
          mountPath: /apollo-configservice/config
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
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
vi svc.yaml
{% code %}
kind: Service
apiVersion: v1
metadata: 
  name: apollo-configservice
  namespace: infra
spec:
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
  selector: 
    app: apollo-configservice
  clusterIP: None
  type: ClusterIP
  sessionAffinity: None
{% endcode %}
<!-- endtab -->
<!-- tab Ingress-->
vi ingress.yaml
{% code %}
kind: Ingress
apiVersion: extensions/v1beta1
metadata: 
  name: apollo-configservice
  namespace: infra
spec:
  rules:
  - host: config.od.com
    http:
      paths:
      - path: /
        backend: 
          serviceName: apollo-configservice
          servicePort: 8080
{% endcode %}
<!-- endtab -->
<!-- tab ConfigMap -->
vi configmap.yaml
{% code %}
apiVersion: v1
kind: ConfigMap
metadata:
  name: apollo-configservice-cm
  namespace: infra
data:
  application-github.properties: |
    # DataSource
    spring.datasource.url = jdbc:mysql://mysql.od.com:3306/ApolloConfigDB?characterEncoding=utf8
    spring.datasource.username = apolloconfig
    spring.datasource.password = 123456
    eureka.service.url = http://config.od.com/eureka
  app.properties: |
    appId=100003171
{% endcode %}
<!-- endtab -->
{% endtabs %}

### 应用资源配置清单
在任意一台k8s运算节点执行：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/apollo-configservice/configmap.yaml
configmap/apollo-configservice-cm created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/apollo-configservice/deployment.yaml
deployment.extensions/apollo-configservice created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/apollo-configservice/svc.yaml
service/apollo-configservice created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/apollo-configservice/ingress.yaml
ingress.extensions/apollo-configservice created
```

### 浏览器访问
http://config.od.com

## 交付apollo-adminservice
### 准备软件包
在运维主机`HDSS7-200.host.com`上：
[下载官方release包](https://github.com/ctripcorp/apollo/releases/download/v1.3.0/apollo-adminservice-1.3.0-github.zip)
```
[root@hdss7-200 src]# ls -l|grep apollo
-rw-r--r-- 1 root root 52713404 Feb 16 08:47 apollo-configservice-1.3.0-github.zip
-rw-r--r-- 1 root root 49418246 Feb 16 09:54 apollo-adminservice-1.3.0-github.zip

[root@hdss7-200 src]# mkdir /data/dockerfile/apollo-adminservice && unzip -o apollo-adminservice-1.3.0-github.zip -d /data/dockerfile/apollo-adminservice
```

### 制作Docker镜像
在运维主机`HDSS7-200.host.com`上：
- 配置数据库连接串
```pwd /data/dockerfile/apollo-adminservice
[root@hdss7-200 apollo-adminservice]# cat config/application-github.properties

```
- 更新starup.sh
```vi /data/dockerfile/apollo-adminservice/scripts/startup.sh
#!/bin/bash
SERVICE_NAME=apollo-adminservice
## Adjust log dir if necessary
LOG_DIR=/opt/logs/apollo-adminservice
## Adjust server port if necessary
SERVER_PORT=8080
APOLLO_ADMIN_SERVICE_NAME=$(hostname -i)
# SERVER_URL="http://localhost:${SERVER_PORT}"
SERVER_URL="http://${APOLLO_ADMIN_SERVICE_NAME}:${SERVER_PORT}"

## Adjust memory settings if necessary
#export JAVA_OPTS="-Xms2560m -Xmx2560m -Xss256k -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=384m -XX:NewSize=1536m -XX:MaxNewSize=1536m -XX:SurvivorRatio=8"

## Only uncomment the following when you are using server jvm
#export JAVA_OPTS="$JAVA_OPTS -server -XX:-ReduceInitialCardMarks"

########### The following is the same for configservice, adminservice, portal ###########
export JAVA_OPTS="$JAVA_OPTS -XX:ParallelGCThreads=4 -XX:MaxTenuringThreshold=9 -XX:+DisableExplicitGC -XX:+ScavengeBeforeFullGC -XX:SoftRefLRUPolicyMSPerMB=0 -XX:+ExplicitGCInvokesConcurrent -XX:+PrintGCDetails -XX:+HeapDumpOnOutOfMemoryError -XX:-OmitStackTraceInFastThrow -Duser.timezone=Asia/Shanghai -Dclient.encoding.override=UTF-8 -Dfile.encoding=UTF-8 -Djava.security.egd=file:/dev/./urandom"
export JAVA_OPTS="$JAVA_OPTS -Dserver.port=$SERVER_PORT -Dlogging.file=$LOG_DIR/$SERVICE_NAME.log -XX:HeapDumpPath=$LOG_DIR/HeapDumpOnOutOfMemoryError/"

# Find Java
if [[ -n "$JAVA_HOME" ]] && [[ -x "$JAVA_HOME/bin/java" ]]; then
    javaexe="$JAVA_HOME/bin/java"
elif type -p java > /dev/null 2>&1; then
    javaexe=$(type -p java)
elif [[ -x "/usr/bin/java" ]];  then
    javaexe="/usr/bin/java"
else
    echo "Unable to find Java"
    exit 1
fi

if [[ "$javaexe" ]]; then
    version=$("$javaexe" -version 2>&1 | awk -F '"' '/version/ {print $2}')
    version=$(echo "$version" | awk -F. '{printf("%03d%03d",$1,$2);}')
    # now version is of format 009003 (9.3.x)
    if [ $version -ge 011000 ]; then
        JAVA_OPTS="$JAVA_OPTS -Xlog:gc*:$LOG_DIR/gc.log:time,level,tags -Xlog:safepoint -Xlog:gc+heap=trace"
    elif [ $version -ge 010000 ]; then
        JAVA_OPTS="$JAVA_OPTS -Xlog:gc*:$LOG_DIR/gc.log:time,level,tags -Xlog:safepoint -Xlog:gc+heap=trace"
    elif [ $version -ge 009000 ]; then
        JAVA_OPTS="$JAVA_OPTS -Xlog:gc*:$LOG_DIR/gc.log:time,level,tags -Xlog:safepoint -Xlog:gc+heap=trace"
    else
        JAVA_OPTS="$JAVA_OPTS -XX:+UseParNewGC"
        JAVA_OPTS="$JAVA_OPTS -Xloggc:$LOG_DIR/gc.log -XX:+PrintGCDetails"
        JAVA_OPTS="$JAVA_OPTS -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=60 -XX:+CMSClassUnloadingEnabled -XX:+CMSParallelRemarkEnabled -XX:CMSFullGCsBeforeCompaction=9 -XX:+CMSClassUnloadingEnabled  -XX:+PrintGCDateStamps -XX:+PrintGCApplicationConcurrentTime -XX:+PrintHeapAtGC -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=5M"
    fi
fi

printf "$(date) ==== Starting ==== \n"

cd `dirname $0`/..
chmod 755 $SERVICE_NAME".jar"
./$SERVICE_NAME".jar" start

rc=$?;

if [[ $rc != 0 ]];
then
    echo "$(date) Failed to start $SERVICE_NAME.jar, return code: $rc"
    exit $rc;
fi

tail -f /dev/null
```

- 写Dockerfile
```vi /data/dockerfile/apollo-adminservice/Dockerfile
FROM openjdk:8-jre-alpine

ENV VERSION 1.3.0

RUN echo "http://mirrors.aliyun.com/alpine/v3.8/main" > /etc/apk/repositories \
    && echo "http://mirrors.aliyun.com/alpine/v3.8/community" >> /etc/apk/repositories \
    && apk update upgrade \
    && apk add --no-cache procps unzip curl bash tzdata \
    && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && echo "Asia/Shanghai" > /etc/timezone

ADD apollo-adminservice-${VERSION}.jar /apollo-adminservice/apollo-adminservice.jar
ADD config/ /apollo-adminservice/config
ADD scripts/ /apollo-adminservice/scripts

CMD ["/apollo-adminservice/scripts/startup.sh"]
```

- 制作镜像并推送
```
[root@hdss7-200 apollo-adminservice]# docker build . -t harbor.od.com/infra/apollo-adminservice:v1.3.0
Sending build context to Docker daemon 55.57 MB
Step 1 : FROM openjdk:8-jre-alpine
 ---> ce8477c7d086
Step 2 : ENV VERSION 1.3.0
 ---> Using cache
 ---> 02291afc5c51
Step 3 : RUN echo "http://mirrors.aliyun.com/alpine/v3.8/main" > /etc/apk/repositories     && echo "http://mirrors.aliyun.com/alpine/v3.8/community" >> /etc/apk/repositories     && apk update upgrade     && apk add --no-cache procps unzip curl bash tzdata     && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime     && echo "Asia/Shanghai" > /etc/timezone
 ---> Using cache
 ---> bfcc40fdf2a2
Step 4 : ADD apollo-adminservice-${VERSION}.jar /apollo-adminservice/apollo-adminservice.jar
 ---> c9f1cacd3b3f
Removing intermediate container d12af621413e
Step 5 : ADD config/ /apollo-adminservice/config
 ---> e3f8f596fdf7
Removing intermediate container 759fe7344b6b
Step 6 : ADD scripts/ /apollo-adminservice/scripts
 ---> 383e3be822ca
Removing intermediate container a66111ed5ea6
Step 7 : CMD /apollo-adminservice/scripts/startup.sh
 ---> Running in 8a7736aecf0c
 ---> c79bd0ef7a9a
Removing intermediate container 8a7736aecf0c
Successfully built c79bd0ef7a9a

[root@hdss7-200 apollo-adminservice]# docker push harbor.od.com/infra/apollo-adminservice:v1.3.0
docker push harbor.od.com/infra/apollo-adminservice:v1.3.0
The push refers to a repository [harbor.od.com/infra/apollo-adminservice]
19b1ca6c066d: Pushed 
8fa6cde49908: Pushed 
0b2c9b9226cc: Pushed 
ebfb473df5c2: Mounted from infra/apollo-configservice 
aae5c057d1b6: Mounted from infra/apollo-configservice 
dee6aef5c2b6: Mounted from infra/apollo-configservice 
a464c54f93a9: Mounted from infra/apollo-configservice 
v1.3.0: digest: sha256:75367caab9bad3d0d281eb3324451a0734e84b6aa3ee860e38ad758d7166a7d1 size: 1785
```
### 准备资源配置清单
在运维主机`HDSS7-200.host.com`上
```pwd /data/k8s-yaml
[root@hdss7-200 k8s-yaml]# mkdir /data/k8s-yaml/apollo-adminservice && cd /data/k8s-yaml/apollo-adminservice
```
{% tabs apollo-adminservice-yaml%}
<!-- tab Deployment -->
vi deployment.yaml
{% code %}
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: apollo-adminservice
  namespace: infra
  labels: 
    name: apollo-adminservice
spec:
  replicas: 1
  selector:
    matchLabels: 
      name: apollo-adminservice
  template:
    metadata:
      labels: 
        app: apollo-adminservice 
        name: apollo-adminservice
    spec:
      volumes:
      - name: configmap-volume
        configMap:
          name: apollo-adminservice-cm
      containers:
      - name: apollo-adminservice
        image: harbor.od.com/infra/apollo-adminservice:v1.3.0
        ports:
        - containerPort: 8080
          protocol: TCP
        volumeMounts:
        - name: configmap-volume
          mountPath: /apollo-adminservice/config
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
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
<!-- tab ConfigMap -->
vi configmap.yaml
{% code %}
apiVersion: v1
kind: ConfigMap
metadata:
  name: apollo-adminservice-cm
  namespace: infra
data:
  application-github.properties: |
    # DataSource
    spring.datasource.url = jdbc:mysql://mysql.od.com:3306/ApolloConfigDB?characterEncoding=utf8
    spring.datasource.username = apolloconfig
    spring.datasource.password = 123456
    eureka.service.url = http://config.od.com/eureka
  app.properties: |
    appId=100003172
{% endcode %}
<!-- endtab -->
{% endtabs %}

### 应用资源配置清单
在任意一台k8s运算节点执行：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/apollo-adminservice/configmap.yaml
configmap/apollo-adminservice-cm created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/apollo-adminservice/deployment.yaml
deployment.extensions/apollo-adminservice created
```
### 浏览器访问
http://config.od.com
![apollo注册中心](/images/eureka-ready.png "apollo注册中心")

## 交付apollo-portal
### 准备软件包
在运维主机`HDSS7-200.host.com`上：
[下载官方release包](https://github.com/ctripcorp/apollo/releases/download/v1.3.0/apollo-portal-1.3.0-github.zip)
```
[root@hdss7-200 src]# ls -l|grep apollo
-rw-r--r-- 1 root root 52713404 Feb 16 08:37 apollo-configservice-1.3.0-github.zip
-rw-r--r-- 1 root root 49418246 Feb 16 09:54 apollo-adminservice-1.3.0-github.zip
-rw-r--r-- 1 root root 36459359 Feb 16 10:00 apollo-portal-1.3.0-github.zip

[root@hdss7-200 src]# mkdir /data/dockerfile/apollo-portal && unzip -o apollo-portal-1.3.0-github.zip -d /data/dockerfile/apollo-portal
Archive:  apollo-portal-1.3.0-github.zip
  inflating: /data/dockerfile/apollo-portal/scripts/shutdown.sh  
  inflating: /data/dockerfile/apollo-portal/apollo-portal.conf  
  inflating: /data/dockerfile/apollo-portal/apollo-portal-1.3.0-sources.jar  
   creating: /data/dockerfile/apollo-portal/config/
  inflating: /data/dockerfile/apollo-portal/config/application-github.properties  
  inflating: /data/dockerfile/apollo-portal/scripts/startup.sh  
  inflating: /data/dockerfile/apollo-portal/config/apollo-env.properties  
  inflating: /data/dockerfile/apollo-portal/config/app.properties  
  inflating: /data/dockerfile/apollo-portal/apollo-portal-1.3.0.jar
```
### 执行数据库脚本
在数据库主机`HDSS7-11.host.com`上：
[数据库脚本地址](https://raw.githubusercontent.com/ctripcorp/apollo/master/scripts/db/migration/portaldb/V1.0.0__initialization.sql)
```
[root@hdss7-11 ~]# mysql -uroot -p
mysql> create database ApolloPortalDB;
mysql> source ./apolloportal.sql
```
### 数据库用户授权
```
mysql> grant INSERT,DELETE,UPDATE,SELECT on ApolloPortalDB.* to "apolloportal"@"172.7.%" identified by "123456";
```
### 制作Docker镜像
在运维主机`HDSS7-200.host.com`上：
- 配置数据库连接串
```pwd /data/dockerfile/apollo-portal
[root@hdss7-200 apollo-portal]# cat config/application-github.properties

```
- 配置Portal的meta service
```vi /data/dockerfile/apollo-portal/config/apollo-env.properties
dev.meta=http://config.od.com
```
- 更新starup.sh
```vi /data/dockerfile/apollo-portal/scripts/startup.sh
#!/bin/bash
SERVICE_NAME=apollo-portal
## Adjust log dir if necessary
LOG_DIR=/opt/logs/apollo-portal-server
## Adjust server port if necessary
SERVER_PORT=8080
APOLLO_PORTAL_SERVICE_NAME=$(hostname -i)
# SERVER_URL="http://localhost:$SERVER_PORT"
SERVER_URL="http://${APOLLO_PORTAL_SERVICE_NAME}:${SERVER_PORT}"

## Adjust memory settings if necessary
#export JAVA_OPTS="-Xms2560m -Xmx2560m -Xss256k -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=384m -XX:NewSize=1536m -XX:MaxNewSize=1536m -XX:SurvivorRatio=8"

## Only uncomment the following when you are using server jvm
#export JAVA_OPTS="$JAVA_OPTS -server -XX:-ReduceInitialCardMarks"

########### The following is the same for configservice, adminservice, portal ###########
export JAVA_OPTS="$JAVA_OPTS -XX:ParallelGCThreads=4 -XX:MaxTenuringThreshold=9 -XX:+DisableExplicitGC -XX:+ScavengeBeforeFullGC -XX:SoftRefLRUPolicyMSPerMB=0 -XX:+ExplicitGCInvokesConcurrent -XX:+PrintGCDetails -XX:+HeapDumpOnOutOfMemoryError -XX:-OmitStackTraceInFastThrow -Duser.timezone=Asia/Shanghai -Dclient.encoding.override=UTF-8 -Dfile.encoding=UTF-8 -Djava.security.egd=file:/dev/./urandom"
export JAVA_OPTS="$JAVA_OPTS -Dserver.port=$SERVER_PORT -Dlogging.file=$LOG_DIR/$SERVICE_NAME.log -XX:HeapDumpPath=$LOG_DIR/HeapDumpOnOutOfMemoryError/"

# Find Java
if [[ -n "$JAVA_HOME" ]] && [[ -x "$JAVA_HOME/bin/java" ]]; then
    javaexe="$JAVA_HOME/bin/java"
elif type -p java > /dev/null 2>&1; then
    javaexe=$(type -p java)
elif [[ -x "/usr/bin/java" ]];  then
    javaexe="/usr/bin/java"
else
    echo "Unable to find Java"
    exit 1
fi

if [[ "$javaexe" ]]; then
    version=$("$javaexe" -version 2>&1 | awk -F '"' '/version/ {print $2}')
    version=$(echo "$version" | awk -F. '{printf("%03d%03d",$1,$2);}')
    # now version is of format 009003 (9.3.x)
    if [ $version -ge 011000 ]; then
        JAVA_OPTS="$JAVA_OPTS -Xlog:gc*:$LOG_DIR/gc.log:time,level,tags -Xlog:safepoint -Xlog:gc+heap=trace"
    elif [ $version -ge 010000 ]; then
        JAVA_OPTS="$JAVA_OPTS -Xlog:gc*:$LOG_DIR/gc.log:time,level,tags -Xlog:safepoint -Xlog:gc+heap=trace"
    elif [ $version -ge 009000 ]; then
        JAVA_OPTS="$JAVA_OPTS -Xlog:gc*:$LOG_DIR/gc.log:time,level,tags -Xlog:safepoint -Xlog:gc+heap=trace"
    else
        JAVA_OPTS="$JAVA_OPTS -XX:+UseParNewGC"
        JAVA_OPTS="$JAVA_OPTS -Xloggc:$LOG_DIR/gc.log -XX:+PrintGCDetails"
        JAVA_OPTS="$JAVA_OPTS -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=60 -XX:+CMSClassUnloadingEnabled -XX:+CMSParallelRemarkEnabled -XX:CMSFullGCsBeforeCompaction=9 -XX:+CMSClassUnloadingEnabled  -XX:+PrintGCDateStamps -XX:+PrintGCApplicationConcurrentTime -XX:+PrintHeapAtGC -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=5M"
    fi
fi

printf "$(date) ==== Starting ==== \n"

cd `dirname $0`/..
chmod 755 $SERVICE_NAME".jar"
./$SERVICE_NAME".jar" start

rc=$?;

if [[ $rc != 0 ]];
then
    echo "$(date) Failed to start $SERVICE_NAME.jar, return code: $rc"
    exit $rc;
fi

tail -f /dev/null
```
- 写Dockerfile
```vi /data/dockerfile/apollo-portal/Dockerfile
FROM openjdk:8-jre-alpine

ENV VERSION 1.3.0

RUN echo "http://mirrors.aliyun.com/alpine/v3.8/main" > /etc/apk/repositories \
    && echo "http://mirrors.aliyun.com/alpine/v3.8/community" >> /etc/apk/repositories \
    && apk update upgrade \
    && apk add --no-cache procps unzip curl bash tzdata \
    && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && echo "Asia/Shanghai" > /etc/timezone

ADD apollo-portal-${VERSION}.jar /apollo-portal/apollo-portal.jar
ADD config/ /apollo-portal/config
ADD scripts/ /apollo-portal/scripts

CMD ["/apollo-portal/scripts/startup.sh"]
```
- 制作镜像并推送
```
[root@hdss7-200 apollo-portal]# docker build . -t harbor.od.com/infra/apollo-portal:v1.3.0
Sending build context to Docker daemon  40.6 MB
Step 1 : FROM openjdk:8-jre-alpine
 ---> ce8477c7d086
Step 2 : ENV VERSION 1.3.0
 ---> Using cache
 ---> 02291afc5c51
Step 3 : RUN echo "http://mirrors.aliyun.com/alpine/v3.8/main" > /etc/apk/repositories     && echo "http://mirrors.aliyun.com/alpine/v3.8/community" >> /etc/apk/repositories     && apk update upgrade     && apk add --no-cache procps unzip curl bash tzdata     && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime     && echo "Asia/Shanghai" > /etc/timezone
 ---> Using cache
 ---> bfcc40fdf2a2
Step 4 : ADD apollo-portal-${VERSION}.jar /apollo-portal/apollo-portal.jar
 ---> f18e4617638c
Removing intermediate container 5c1ba8001ff3
Step 5 : ADD config/ /apollo-portal/config
 ---> 201162bf7bf8
Removing intermediate container a5d89acc0779
Step 6 : ADD scripts/ /apollo-portal/scripts
 ---> 17acba32839f
Removing intermediate container fc702b1bb605
Step 7 : CMD /apollo-portal/scripts/startup.sh
 ---> Running in b88cf490171f
 ---> 76409781a792
Removing intermediate container b88cf490171f
Successfully built 76409781a792
[root@hdss7-200 apollo-portal]# docker push harbor.od.com/infra/apollo-portal:v1.3.0
docker push harbor.od.com/infra/apollo-portal:v1.3.0
The push refers to a repository [harbor.od.com/infra/apollo-portal]
e7c0e96ded4e: Pushed 
0076c5344476: Pushed 
3851a45d7440: Pushed 
ebfb473df5c2: Mounted from infra/apollo-adminservice 
aae5c057d1b6: Mounted from infra/apollo-adminservice 
dee6aef5c2b6: Mounted from infra/apollo-adminservice 
a464c54f93a9: Mounted from infra/apollo-adminservice 
v1.3.0: digest: sha256:1aa30aac8642cceb97c053b7d74632240af08f64c49b65d8729021fef65628a4 size: 1785
```
### 解析域名
DNS主机`HDSS7-11.host.com`上：
```vi /var/named/od.com.zone
portal	60 IN A 10.4.7.10
```
### 准备资源配置清单
在运维主机`HDSS7-200.host.com`上
```pwd /data/k8s-yaml
[root@hdss7-200 k8s-yaml]# mkdir /data/k8s-yaml/apollo-portal && cd /data/k8s-yaml/apollo-portal
```
{% tabs apollo-portal-yaml%}
<!-- tab Deployment -->
vi deployment.yaml
{% code %}
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: apollo-portal
  namespace: infra
  labels: 
    name: apollo-portal
spec:
  replicas: 1
  selector:
    matchLabels: 
      name: apollo-portal
  template:
    metadata:
      labels: 
        app: apollo-portal 
        name: apollo-portal
    spec:
      volumes:
      - name: configmap-volume
        configMap:
          name: apollo-portal-cm
      containers:
      - name: apollo-portal
        image: harbor.od.com/infra/apollo-portal:v1.3.0
        ports:
        - containerPort: 8080
          protocol: TCP
        volumeMounts:
        - name: configmap-volume
          mountPath: /apollo-portal/config
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
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
vi svc.yaml
{% code %}
kind: Service
apiVersion: v1
metadata: 
  name: apollo-portal
  namespace: infra
spec:
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
  selector: 
    app: apollo-portal
  clusterIP: None
  type: ClusterIP
  sessionAffinity: None
{% endcode %}
<!-- endtab -->
<!-- tab Ingress-->
vi ingress.yaml
{% code %}
kind: Ingress
apiVersion: extensions/v1beta1
metadata: 
  name: apollo-portal
  namespace: infra
spec:
  rules:
  - host: portal.od.com
    http:
      paths:
      - path: /
        backend: 
          serviceName: apollo-portal
          servicePort: 8080
{% endcode %}
<!-- endtab -->
<!-- tab ConfigMap -->
vi configmap.yaml
{% code %}
apiVersion: v1
kind: ConfigMap
metadata:
  name: apollo-portal-cm
  namespace: infra
data:
  application-github.properties: |
    # DataSource
    spring.datasource.url = jdbc:mysql://mysql.od.com:3306/ApolloPortalDB?characterEncoding=utf8
    spring.datasource.username = apolloportal
    spring.datasource.password = 123456
  app.properties: |
    appId=100003173
  apollo-env.properties: |
    dev.meta=http://config.od.com
{% endcode %}
<!-- endtab -->
{% endtabs %}

### 应用资源配置清单
在任意一台k8s运算节点执行：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/apollo-portal/configmap.yaml
configmap/apollo-portal-cm created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/apollo-portal/deployment.yaml
deployment.extensions/apollo-portal created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/apollo-portal/svc.yaml
service/apollo-portal created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/apollo-portal/ingress.yaml
ingress.extensions/apollo-portal created
```
### 浏览器访问
http://portal.od.com
- 用户名：apollo
- 密码： admin

![apollo-portal](/images/portal-ready.png "apollo-portal")

# 实战dubbo微服务接入Apollo配置中心
## 改造dubbo-demo-service项目
### 使用IDE拉取项目（这里使用git bash作为范例）
```
$ git clone git@gitee.com/stanleywang/dubbo-demo-service.git
```
### 切到apollo分支
```
$ git checkout -b apollo
```
### 修改pom.xml
- 加入apollo客户端jar包的依赖

```vi dubbo-server/pom.xml
<dependency>
  <groupId>com.ctrip.framework.apollo</groupId>
  <artifactId>apollo-client</artifactId>
  <version>1.1.0</version>
</dependency>
```
- 修改resource段

```vi dubbo-server/pom.xml
<resource>
  <directory>src/main/resources</directory>
  <includes>
  <include>**/*</include>
  </includes>
  <filtering>false</filtering>
</resource>
```
### 增加resources目录
```pwd /d/workspace/dubbo-demo-service/dubbo-server/src/main
$ mkdir -pv resources/META-INF
mkdir: created directory 'resources'
mkdir: created directory 'resources/META-INF'
```
### 修改config.properties文件
```vi /d/workspace/dubbo-demo-service/dubbo-server/src/main/resources/config.properties
dubbo.registry=${dubbo.registry}
dubbo.port=${dubbo.port}
```
### 修改srping-config.xml文件
- beans段新增属性

```vi /d/workspace/dubbo-demo-service/dubbo-server/src/main/resources/spring-config.xml
xmlns:apollo="http://www.ctrip.com/schema/apollo"
```
- xsi:schemaLocation段内新增属性

```vi /d/workspace/dubbo-demo-service/dubbo-server/src/main/resources/spring-config.xml
http://www.ctrip.com/schema/apollo http://www.ctrip.com/schema/apollo.xsd
```
- 新增配置项

```vi /d/workspace/dubbo-demo-service/dubbo-server/src/main/resources/spring-config.xml
<apollo:config/>
```
- 删除配置项（注释）

```vi /d/workspace/dubbo-demo-service/dubbo-server/src/main/resources/spring-config.xml
<!-- <context:property-placeholder location="classpath:config.properties"/> -->
```

### 增加app.properties文件
```vi /d/workspace/dubbo-demo-service/dubbo-server/src/main/resources/META-INF/app.properties
app.id=dubbo-demo-service
```

### 提交git中心仓库（gitee）
```
$ git push origin apollo
```

## 配置apollo-portal
### 创建项目
- 部门
> 样例部门1（TEST1）

- **应用id**
> dubbo-demo-service

- 应用名称
> dubbo服务提供者

- 应用负责人
> apollo|apollo

- 项目管理员
> apollo|apollo

**提交**

### 进入配置页面
#### 新增配置项1
- Key
> dubbo.registry

- Value
> zookeeper://zk1.od.com:2181

- 选择集群
> DEV

**提交**

#### 新增配置项2
- Key
> dubbo.port

- Value
> 20880

- 选择集群
> DEV

**提交**

### 发布配置
点击**发布**，配置生效
![apollo-release](/images/apollo-release.png "apollo发布配置")

## 使用jenkins进行CI
略（注意记录镜像的tag）

## 上线新构建的项目
### 准备资源配置清单
运维主机`HDSS7-200.host.com`上：
```vi /data/k8s-yaml/dubbo-demo-service/deployment.yaml
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
        image: harbor.od.com/app/dubbo-demo-service:apollo_190119_1815
        ports:
        - containerPort: 20880
          protocol: TCP
        env:
        - name: C_OPTS
          value: -Denv=dev -Dapollo.meta=http://config.od.com
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

**注意：**增加了env段配置
**注意：**docker镜像新版的tag

### 应用资源配置清单
在任意一台k8s运算节点上执行：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dubbo-demo-service/deployment.yaml
deployment.extensions/dubbo-demo-service configured
```
### 观察项目运行情况
http://dubbo-monitor.od.com

## 改造dubbo-demo-web
略

## 配置apollo-portal
### 创建项目
- 部门
> 样例部门1（TEST1）

- **应用id**
> dubbo-demo-web

- 应用名称
> dubbo服务消费者

- 应用负责人
> apollo|apollo

- 项目管理员
> apollo|apollo

**提交**

### 进入配置页面
#### 新增配置项1
- Key
> dubbo.registry

- Value
> zookeeper://zk1.od.com:2181

- 选择集群
> DEV

**提交**

### 发布配置
点击**发布**，配置生效

## 使用jenkins进行CI
略（注意记录镜像的tag）

## 上线新构建的项目
### 准备资源配置清单
运维主机`HDSS7-200.host.com`上：
```vi /data/k8s-yaml/dubbo-demo-consumer/deployment.yaml
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
        image: harbor.od.com/app/dubbo-demo-consumer:apllo_190120_1815
        ports:
        - containerPort: 20880
          protocol: TCP
        - containerPort: 8080
          protocol: TCP
        env:
        - name: C_OPTS
          value: -Denv=dev -Dapollo.meta=http://config.od.com
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
```

**注意：**增加了env段配置
**注意：**docker镜像新版的tag

### 应用资源配置清单
在任意一台k8s运算节点上执行：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dubbo-demo-web/deployment.yaml
deployment.extensions/dubbo-demo-consumer configured
```

## 通过Apollo配置中心动态维护项目的配置
以dubbo-demo-service项目为例，不用修改代码
- 在http://portal.od.com 里修改dubbo.port配置项
- 重启dubbo-demo-service项目
- 配置生效

# 实战维护多套dubbo微服务环境
## 生产实践
1. 迭代新需求/修复BUG（编码->提GIT）
2. 测试环境发版，测试（应用通过编译打包发布至TEST命名空间）
3. 测试通过，上线（应用镜像直接发布至PROD命名空间）

## 系统架构
- 物理架构

主机名|角色|ip
-|-|-
HDSS7-11.host.com|zk-test(测试环境Test)|10.4.7.11
HDSS7-12.host.com|zk-prod(生产环境Prod)|10.4.7.12
HDSS7-21.host.com|kubernetes运算节点|10.4.7.21
HDSS7-22.host.com|kubernetes运算节点|10.4.7.22
HDSS7-200.host.com|运维主机，harbor仓库|10.4.7.200

- K8S内系统架构

环境|命名空间|应用
-|-|-
测试环境（TEST）|infra-test|apollo-config，apollo-admin
测试环境（TEST）|app-test|dubbo-demo-service，dubbo-demo-web
生产环境（PROD）|infra-prod|apollo-config，apollo-admin
生产环境（PROD）|app-prod|dubbo-demo-service，dubbo-demo-web
ops环境（infra）|infra|jenkins，dubbo-monitor，apollo-portal

## 修改/添加域名解析
DNS主机`HDSS7-11.host.com`上：
```vi /var/named/od.com.zone
zk-test 60 IN A 10.4.7.11
zk-prod 60 IN A 10.4.7.12
config-test	60 IN A 10.4.7.10
config-prod	60 IN A 10.4.7.10
demo-test 60 IN A 10.4.7.10
demo-prod 60 IN A 10.4.7.10
```

## Apollo的k8s应用配置
- 删除app命名空间内应用，创建infra-test命名空间，创建infra-prod命名空间
- 删除infra命名空间内apollo-configservice，apollo-adminservice应用
- 数据库内删除ApolloConfigDB，创建ApolloConfigTestDB，创建ApolloConfigProdDB

```
mysql> drop database ApolloConfigDB;

mysql> create database ApolloConfigTestDB;
mysql> use ApolloConfigTestDB;
mysql> source ./apolloconfig.sql
mysql> update ApolloConfigTestDB.ServerConfig set ServerConfig.Value="http://config-test.od.com/eureka" where ServerConfig.Key="eureka.service.url";
mysql> grant INSERT,DELETE,UPDATE,SELECT on ApolloConfigTestDB.* to "apolloconfig"@"172.7.%" identified by "123456";

mysql> create database ApolloConfigProdDB;
mysql> use ApolloConfigProdDB;
mysql> source ./apolloconfig.sql
mysql> update ApolloConfigProdDB.ServerConfig set ServerConfig.Value="http://config-prod.od.com/eureka" where ServerConfig.Key="eureka.service.url";
mysql> grant INSERT,DELETE,UPDATE,SELECT on ApolloConfigProdDB.* to "apolloconfig"@"172.7.%" identified by "123456";
```
- 准备apollo-config，apollo-admin的资源配置清单（各2套）

**注：**apollo-config/apollo-admin的configmap配置要点
- Test环境

```
application-github.properties: |
    # DataSource
    spring.datasource.url = jdbc:mysql://mysql.od.com:3306/ApolloConfigTestDB?characterEncoding=utf8
    spring.datasource.username = apolloconfig
    spring.datasource.password = 123456
    eureka.service.url = http://config-test.od.com/eureka
```
- Prod环境

```
application-github.properties: |
    # DataSource
    spring.datasource.url = jdbc:mysql://mysql.od.com:3306/ApolloConfigProdDB?characterEncoding=utf8
    spring.datasource.username = apolloconfig
    spring.datasource.password = 123456
    eureka.service.url = http://config-prod.od.com/eureka
```

- 依次应用，分别发布在infra-test和infra-prod命名空间
- 修改apollo-portal的configmap并重启portal

```
apollo-env.properties: |
    TEST.meta=http://config-test.od.com
    PROD.meta=http://config-prod.od.com
```

## Apollo的portal配置
### 管理员工具
删除应用、集群、AppNamespace，将已配置应用删除
### 系统参数
- Key
> apollo.portal.envs

- Value
> TEST,PROD

**查询**
- Value
> TEST,PROD

**保存**

## 新建dubbo-demo-service和dubbo-demo-web项目
在TEST/PROD环境分别增加配置项并发布

## 发布dubbo微服务
- 准备dubbo-demo-service和dubbo-demo-web的资源配置清单（各2套）
- 依次应用，分别发布至app-test和app-prod命名空间
- 使用dubbo-monitor查验

# 互联网公司技术部的日常
- 产品经理整理需求，需求评审，出产品原型
- 开发同学夜以继日的开发，提测
- 测试同学使用Jenkins持续集成，并发布至测试环境
- 验证功能，通过->待上线or打回->修改代码
- 提交发版申请，运维同学将测试后的包发往生产环境
- 无尽的BUG修复（笑cry）
