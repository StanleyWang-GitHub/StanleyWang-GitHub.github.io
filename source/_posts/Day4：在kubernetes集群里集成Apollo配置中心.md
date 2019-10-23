title: Day4：在kubernetes集群里集成Apollo配置中心
author: Stanley Wang
date: 2019-10-18 19:12:56
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
vi /data/k8s-yaml/dubbo-monitor/cm.yaml
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
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dubbo-monitor/cm.yaml
configmap/dubbo-monitor-cm created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dubbo-monitor/dp.yaml
deployment.extensions/dubbo-monitor configured
```

## 重新发版，修改dubbo项目的配置文件
### 修改项目源代码(仅演示)
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
[下载官方release包](https://github.com/ctripcorp/apollo/releases/download/v1.4.0/apollo-configservice-1.4.0-github.zip)
```pwd /opt/src
[root@hdss7-200 src]# ls -l|grep apollo
-rw-r--r-- 1 root root 52713404 Feb 16 23:29 apollo-configservice-1.4.0-github.zip
[root@hdss7-200 src]# mkdir /data/dockerfile/apollo-configservice && unzip -o apollo-configservice-1.4.0-github.zip -d /data/dockerfile/apollo-configservice
Archive:  apollo-configservice-1.4.0-github.zip
   creating: /data/dockerfile/apollo-configservice/scripts/
  inflating: /data/dockerfile/apollo-configservice/config/application-github.properties  
  inflating: /data/dockerfile/apollo-configservice/scripts/shutdown.sh  
  inflating: /data/dockerfile/apollo-configservice/apollo-configservice-1.4.0-sources.jar  
  inflating: /data/dockerfile/apollo-configservice/scripts/startup.sh  
  inflating: /data/dockerfile/apollo-configservice/config/app.properties  
  inflating: /data/dockerfile/apollo-configservice/apollo-configservice-1.4.0.jar  
  inflating: /data/dockerfile/apollo-configservice/apollo-configservice.conf
```
### 安装部署MySQL数据库
在数据库主机`HDSS7-11.host.com`上：
**注意：**MySQL版本应为5.6或以上！
- 更新yum源

```vi /etc/yum.repos.d/MariaDB.repo
[mariadb]
name = MariaDB
baseurl = https://mirrors.ustc.edu.cn/mariadb/yum/10.1/centos7-amd64/
gpgkey=https://mirrors.ustc.edu.cn/mariadb/yum/RPM-GPG-KEY-MariaDB
gpgcheck=1
```

- 导入GPG-KEY

```
[root@hdss7-11 ~]# rpm --import https://mirrors.ustc.edu.cn/mariadb/yum/RPM-GPG-KEY-MariaDB
```

- 更新数据库版本

```
[root@hdss7-11 ~]# yum install MariaDB-server -y
```

- 配置my.cnf

```vi /etc/my.cnf
[mysql]
default-character-set = utf8mb4
[mysqld]
character_set_server = utf8mb4
collation_server = utf8mb4_general_ci
init_connect = "SET NAMES 'utf8mb4'"
```

- 启动数据库

```
[root@hdss7-11 ~]# systemctl start mariadb
[root@hdss7-11 ~]# netstat -luntp|grep 3306
tcp6       0      0 :::3306                 :::*                    LISTEN      43437/mysqld
[root@hdss7-11 ~]# systemctl enable mariadb
```

- 修改数据库管理员密码

```
[root@hdss7-11 ~]# mysqladmin -u root password
New password: (123456)
Confirm new password: (123456) 
```

### 执行数据库脚本

[数据库脚本地址](https://raw.githubusercontent.com/ctripcorp/apollo/master/scripts/db/migration/configdb/V1.0.0__initialization.sql)
```
[root@hdss7-11 ~]# mysql -uroot -p
mysql> create database ApolloConfigDB;
mysql> source ./apolloconfig.sql
```

### 数据库用户授权
```
mysql> grant INSERT,DELETE,UPDATE,SELECT on ApolloConfigDB.* to "apolloconfig"@"10.4.7.%" identified by "123456";
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
- 查看数据库连接串（仅演示）

```pwd /data/dockerfile/apollo-configservice
[root@hdss7-200 apollo-configservice]# cat config/application-github.properties
# DataSource
spring.datasource.url = jdbc:mysql://fill-in-the-correct-server:3306/ApolloConfigDB?characterEncoding=utf8
spring.datasource.username = FillInCorrectUser
spring.datasource.password = FillInCorrectPassword
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
FROM stanleyws/jre8:8u112

ENV VERSION 1.4.0

RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime &&\
    echo "Asia/Shanghai" > /etc/timezone

ADD apollo-configservice-${VERSION}.jar /apollo-configservice/apollo-configservice.jar
ADD config/ /apollo-configservice/config
ADD scripts/ /apollo-configservice/scripts

CMD ["/apollo-configservice/scripts/startup.sh"]
```

- 制作镜像并推送

```
[root@hdss7-200 apollo-configservice]# docker build . -t harbor.od.com/infra/apollo-configservice:v1.4.0
Sending build context to Docker daemon 61.91 MB
Step 1 : FROM stanleyws/jre8:8u112
 ---> fa3a085d6ef1
Step 2 : ENV VERSION 1.4.0
 ---> [Warning] IPv4 forwarding is disabled. Networking will not work.
 ---> Running in 685d51b5adb4
 ---> feb4c0289f04
Removing intermediate container 685d51b5adb4
Step 3 : RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime &&    echo "Asia/Shanghai" > /etc/timezone
 ---> [Warning] IPv4 forwarding is disabled. Networking will not work.
 ---> Running in eaa05073feeb
 ---> a3e3fd61ae35
Removing intermediate container eaa05073feeb
Step 4 : ADD apollo-configservice-${VERSION}.jar /apollo-configservice/apollo-configservice.jar
 ---> be09a59b83a2
Removing intermediate container ac6b8af3979b
Step 5 : ADD config/ /apollo-configservice/config
 ---> fb64fc0f3194
Removing intermediate container b73c5315ad20
Step 6 : ADD scripts/ /apollo-configservice/scripts
 ---> 96ff3d9b9456
Removing intermediate container 67ba203b3101
Step 7 : CMD /apollo-configservice/scripts/startup.sh
 ---> [Warning] IPv4 forwarding is disabled. Networking will not work.
 ---> Running in 80bd3f53fefc
 ---> 551ea2ba8de3
Removing intermediate container 80bd3f53fefc
Successfully built 551ea2ba8de3

[root@hdss7-200 apollo-configservice]# docker push harbor.od.com/infra/apollo-configservice:v1.4.0
The push refers to a repository [harbor.od.com/infra/apollo-configservice]
25efb9a44683: Pushed 
b3572bb46247: Pushed 
e7994b936025: Pushed 
0ff1d078cbc4: Pushed 
ebfb473df5c2: Pushed 
aae5c057d1b6: Pushed 
dee6aef5c2b6: Pushed 
a464c54f93a9: Pushed 
v1.4.0: digest: sha256:6a8e4fdda58de0dfba9985ebbf91c4d6f46f5274983d2efa8853b03f4e45fa06 size: 1992
```

### 解析域名
DNS主机`HDSS7-11.host.com`上：
```vi /var/named/od.com.zone
mysql              A    10.4.7.11
config             A    10.4.7.10
```

### 准备资源配置清单
在运维主机`HDSS7-200.host.com`上
```pwd /data/k8s-yaml
[root@hdss7-200 k8s-yaml]# mkdir /data/k8s-yaml/apollo-configservice && cd /data/k8s-yaml/apollo-configservice
```
{% tabs apollo-configservice-yaml%}
<!-- tab Deployment -->
vi dp.yaml
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
        image: harbor.od.com/infra/apollo-configservice:v1.4.0
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
vi cm.yaml
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
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/apollo-configservice/cm.yaml
configmap/apollo-configservice-cm created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/apollo-configservice/dp.yaml
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
[下载官方release包](https://github.com/ctripcorp/apollo/releases/download/v1.4.0/apollo-adminservice-1.4.0-github.zip)
```
[root@hdss7-200 src]# ls -l|grep apollo
-rw-r--r-- 1 root root 52713404 Feb 16 08:47 apollo-configservice-1.4.0-github.zip
-rw-r--r-- 1 root root 49418246 Feb 16 09:54 apollo-adminservice-1.4.0-github.zip

[root@hdss7-200 src]# mkdir /data/dockerfile/apollo-adminservice && unzip -o apollo-adminservice-1.4.0-github.zip -d /data/dockerfile/apollo-adminservice
```

### 制作Docker镜像
在运维主机`HDSS7-200.host.com`上：
- 查看数据库连接串（仅演示）

```pwd /data/dockerfile/apollo-adminservice
[root@hdss7-200 apollo-adminservice]# cat config/application-github.properties
# DataSource
spring.datasource.url = jdbc:mysql://fill-in-the-correct-server:3306/ApolloConfigDB?characterEncoding=utf8
spring.datasource.username = FillInCorrectUser
spring.datasource.password = FillInCorrectPassword
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
FROM stanleyws/jre8:8u112

ENV VERSION 1.4.0

RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime &&\
    echo "Asia/Shanghai" > /etc/timezone

ADD apollo-adminservice-${VERSION}.jar /apollo-adminservice/apollo-adminservice.jar
ADD config/ /apollo-adminservice/config
ADD scripts/ /apollo-adminservice/scripts

CMD ["/apollo-adminservice/scripts/startup.sh"]
```

- 制作镜像并推送

```
[root@hdss7-200 apollo-adminservice]# docker build . -t harbor.od.com/infra/apollo-adminservice:v1.4.0
Sending build context to Docker daemon 58.31 MB
Step 1 : FROM stanleyws/jre8:8u112
 ---> fa3a085d6ef1
Step 2 : ENV VERSION 1.4.0
 ---> Using cache
 ---> feb4c0289f04
Step 3 : RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime &&    echo "Asia/Shanghai" > /etc/timezone
 ---> Using cache
 ---> a3e3fd61ae35
Step 4 : ADD apollo-adminservice-${VERSION}.jar /apollo-adminservice/apollo-adminservice.jar
 ---> 6a1eb9565777
Removing intermediate container 7196df9af6af
Step 5 : ADD config/ /apollo-adminservice/config
 ---> 9f364b732d46
Removing intermediate container 9b24669c6c78
Step 6 : ADD scripts/ /apollo-adminservice/scripts
 ---> b7bc5517b0fc
Removing intermediate container f3e34e759148
Step 7 : CMD /apollo-adminservice/scripts/startup.sh
 ---> [Warning] IPv4 forwarding is disabled. Networking will not work.
 ---> Running in 18c6597914b4
 ---> 82145db3ee88
Removing intermediate container 18c6597914b4
Successfully built 82145db3ee88

[root@hdss7-200 apollo-adminservice]# docker push harbor.od.com/infra/apollo-adminservice:v1.4.0
docker push harbor.od.com/infra/apollo-adminservice:v1.4.0
The push refers to a repository [harbor.od.com/infra/apollo-adminservice]
19b1ca6c066d: Pushed 
8fa6cde49908: Pushed 
0b2c9b9226cc: Pushed 
ebfb473df5c2: Mounted from infra/apollo-configservice 
aae5c057d1b6: Mounted from infra/apollo-configservice 
dee6aef5c2b6: Mounted from infra/apollo-configservice 
a464c54f93a9: Mounted from infra/apollo-configservice 
v1.4.0: digest: sha256:75367caab9bad3d0d281eb3324451a0734e84b6aa3ee860e38ad758d7166a7d1 size: 1785
```
### 准备资源配置清单
在运维主机`HDSS7-200.host.com`上
```pwd /data/k8s-yaml
[root@hdss7-200 k8s-yaml]# mkdir /data/k8s-yaml/apollo-adminservice && cd /data/k8s-yaml/apollo-adminservice
```
{% tabs apollo-adminservice-yaml%}
<!-- tab Deployment -->
vi dp.yaml
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
        image: harbor.od.com/infra/apollo-adminservice:v1.4.0
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
vi cm.yaml
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
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/apollo-adminservice/cm.yaml
configmap/apollo-adminservice-cm created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/apollo-adminservice/dp.yaml
deployment.extensions/apollo-adminservice created
```
### 浏览器访问
http://config.od.com
![apollo注册中心](/images/eureka-ready.png "apollo注册中心")

## 交付apollo-portal
### 准备软件包
在运维主机`HDSS7-200.host.com`上：
[下载官方release包](https://github.com/ctripcorp/apollo/releases/download/v1.4.0/apollo-portal-1.4.0-github.zip)
```
[root@hdss7-200 src]# ls -l|grep apollo
-rw-r--r-- 1 root root 52713404 Feb 16 08:37 apollo-configservice-1.4.0-github.zip
-rw-r--r-- 1 root root 49418246 Feb 16 09:54 apollo-adminservice-1.4.0-github.zip
-rw-r--r-- 1 root root 36459359 Feb 16 10:00 apollo-portal-1.4.0-github.zip

[root@hdss7-200 src]# mkdir /data/dockerfile/apollo-portal && unzip -o apollo-portal-1.4.0-github.zip -d /data/dockerfile/apollo-portal
Archive:  apollo-portal-1.4.0-github.zip
  inflating: /data/dockerfile/apollo-portal/scripts/shutdown.sh  
  inflating: /data/dockerfile/apollo-portal/apollo-portal.conf  
  inflating: /data/dockerfile/apollo-portal/apollo-portal-1.4.0-sources.jar  
   creating: /data/dockerfile/apollo-portal/config/
  inflating: /data/dockerfile/apollo-portal/config/application-github.properties  
  inflating: /data/dockerfile/apollo-portal/scripts/startup.sh  
  inflating: /data/dockerfile/apollo-portal/config/apollo-env.properties  
  inflating: /data/dockerfile/apollo-portal/config/app.properties  
  inflating: /data/dockerfile/apollo-portal/apollo-portal-1.4.0.jar
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
mysql> grant INSERT,DELETE,UPDATE,SELECT on ApolloPortalDB.* to "apolloportal"@"10.4.7.%" identified by "123456";
```
### 制作Docker镜像
在运维主机`HDSS7-200.host.com`上：
- 查看数据库连接串（仅演示）

```pwd /data/dockerfile/apollo-portal
[root@hdss7-200 apollo-portal]# cat config/application-github.properties
# DataSource
spring.datasource.url = jdbc:mysql://fill-in-the-correct-server:3306/ApolloPortalDB?characterEncoding=utf8
spring.datasource.username = FillInCorrectUser
spring.datasource.password = FillInCorrectPassword
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
FROM stanleyws/jre8:8u112

ENV VERSION 1.4.0

RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime &&\
    echo "Asia/Shanghai" > /etc/timezone

ADD apollo-portal-${VERSION}.jar /apollo-portal/apollo-portal.jar
ADD config/ /apollo-portal/config
ADD scripts/ /apollo-portal/scripts

CMD ["/apollo-portal/scripts/startup.sh"]
```
- 制作镜像并推送

```
[root@hdss7-200 apollo-portal]# docker build . -t harbor.od.com/infra/apollo-portal:v1.4.0
Sending build context to Docker daemon 43.35 MB
Step 1 : FROM stanleyws/jre8:8u112
 ---> fa3a085d6ef1
Step 2 : ENV VERSION 1.4.0
 ---> Using cache
 ---> feb4c0289f04
Step 3 : RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime &&    echo "Asia/Shanghai" > /etc/timezone
 ---> Using cache
 ---> a3e3fd61ae35
Step 4 : ADD apollo-portal-${VERSION}.jar /apollo-portal/apollo-portal.jar
 ---> cfcf63e8eedc
Removing intermediate container 860b55bd3fc5
Step 5 : ADD config/ /apollo-portal/config
 ---> 3ee780369431
Removing intermediate container 6b67ee4224b5
Step 6 : ADD scripts/ /apollo-portal/scripts
 ---> 42c9aea2e9e3
Removing intermediate container 2dcf8d1bf4cf
Step 7 : CMD /apollo-portal/scripts/startup.sh
 ---> [Warning] IPv4 forwarding is disabled. Networking will not work.
 ---> Running in 9162dab8b63a
 ---> 0c020b79c36f
Removing intermediate container 9162dab8b63a
Successfully built 0c020b79c36f
[root@hdss7-200 apollo-portal]# docker push harbor.od.com/infra/apollo-portal:v1.4.0
docker push harbor.od.com/infra/apollo-portal:v1.4.0
The push refers to a repository [harbor.od.com/infra/apollo-portal]
e7c0e96ded4e: Pushed 
0076c5344476: Pushed 
3851a45d7440: Pushed 
ebfb473df5c2: Mounted from infra/apollo-adminservice 
aae5c057d1b6: Mounted from infra/apollo-adminservice 
dee6aef5c2b6: Mounted from infra/apollo-adminservice 
a464c54f93a9: Mounted from infra/apollo-adminservice 
v1.4.0: digest: sha256:1aa30aac8642cceb97c053b7d74632240af08f64c49b65d8729021fef65628a4 size: 1785
```

### 解析域名
DNS主机`HDSS7-11.host.com`上：
```vi /var/named/od.com.zone
portal             A    10.4.7.10
```
### 准备资源配置清单
在运维主机`HDSS7-200.host.com`上
```pwd /data/k8s-yaml
[root@hdss7-200 k8s-yaml]# mkdir /data/k8s-yaml/apollo-portal && cd /data/k8s-yaml/apollo-portal
```
{% tabs apollo-portal-yaml%}
<!-- tab Deployment -->
vi dp.yaml
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
        image: harbor.od.com/infra/apollo-portal:v1.4.0
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
vi cm.yaml
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
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/apollo-portal/cm.yaml
configmap/apollo-portal-cm created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/apollo-portal/dp.yaml
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
## 改造dubbo-demo-service项目（仅演示）
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
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dubbo-demo-service/dp.yaml
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
```vi /data/k8s-yaml/dubbo-demo-consumer/dp.yaml
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
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dubbo-demo-web/dp.yaml
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
测试环境（FAT）|test|apollo-config，apollo-admin
测试环境（FAT）|test|dubbo-demo-service，dubbo-demo-web
生产环境（PRO）|prod|apollo-config，apollo-admin
生产环境（PRO）|prod|dubbo-demo-service，dubbo-demo-web
ops环境|infra|jenkins，dubbo-monitor，apollo-portal

## 修改/添加域名解析
DNS主机`HDSS7-11.host.com`上：
```vi /var/named/od.com.zone
zk-test            A    10.4.7.11
zk-prod            A    10.4.7.12
config-test        A    10.4.7.10
config-prod        A    10.4.7.10
demo-test          A    10.4.7.10
demo-prod          A    10.4.7.10
```

## Apollo的k8s应用配置
- 删除app命名空间内应用，创建test命名空间，创建prod命名空间
- 删除infra命名空间内apollo-configservice，apollo-adminservice应用
- 数据库内删除ApolloConfigDB，创建ApolloConfigTestDB，创建ApolloConfigProdDB

```
mysql> drop database ApolloConfigDB;

mysql> create database ApolloConfigTestDB;
mysql> use ApolloConfigTestDB;
mysql> source ./apolloconfig.sql
mysql> update ApolloConfigTestDB.ServerConfig set ServerConfig.Value="http://config-test.od.com/eureka" where ServerConfig.Key="eureka.service.url";
mysql> grant INSERT,DELETE,UPDATE,SELECT on ApolloConfigTestDB.* to "apolloconfig"@"10.4.7.%" identified by "123456";

mysql> create database ApolloConfigProdDB;
mysql> use ApolloConfigProdDB;
mysql> source ./apolloconfig.sql
mysql> update ApolloConfigProdDB.ServerConfig set ServerConfig.Value="http://config-prod.od.com/eureka" where ServerConfig.Key="eureka.service.url";
mysql> grant INSERT,DELETE,UPDATE,SELECT on ApolloConfigProdDB.* to "apolloconfig"@"10.4.7.%" identified by "123456";
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

- 依次应用，分别发布在test和prod命名空间

- 修改apollo-portal的configmap并重启portal

```
apollo-env.properties: |
    fat.meta=http://config-test.od.com
    pro.meta=http://config-prod.od.com
```

## Apollo的portal配置
### 管理员工具
删除应用、集群、AppNamespace，将已配置应用删除
### 系统参数
- Key
> apollo.portal.envs

- Value
> fat,pro

**查询**
- Value
> fat,pro

**保存**

## 新建dubbo-demo-service和dubbo-demo-web项目
在FAT/PRO环境分别增加配置项并发布

## 发布dubbo微服务
- 准备dubbo-demo-service和dubbo-demo-web的资源配置清单（各2套）
- 依次应用，分别发布至test和prod命名空间
- 使用dubbo-monitor查验
- 浏览器依次打开http://demo-test.od.com 和 http://demo-prod.od.com

# 项目迭代流程（仅演示）
## 开发提测（提交代码到apollo分支）
## 打包发版至测试环境进行测试（http://demo-test.od.com）
## 直接将通过测试的docker镜像，应用到生产环境（不再进行打包操作）
## 项目投产完成