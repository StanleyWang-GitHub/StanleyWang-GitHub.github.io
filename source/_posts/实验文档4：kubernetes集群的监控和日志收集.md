title: 实验文档4：kubernetes集群的监控和日志收集
author: Stanley Wang
categories: Kubernetes容器云技术专题
date: 2019-1-18 19:12:56
---
- - -
{% cq %}欢迎加入王导的VIP学习qq群：==>[<font color="FF7F50">932194668</font>](http://shang.qq.com/wpa/qunwpa?idkey=78869fddc5a661acb0639315eb52997c108de6625df5f0ee2f0372f176a032a6)<=={% endcq %}
- - -
# 改造dubbo-demo-web项目为Tomcat启动项目
[Tomcat官网](http://tomcat.apache.org/)
## 准备Tomcat的镜像底包
### 准备tomcat二进制包
运维主机`HDSS7-200.host.com`上：
[Tomcat8下载链接](http://mirrors.tuna.tsinghua.edu.cn/apache/tomcat/tomcat-8/v8.5.40/bin/apache-tomcat-8.5.40.tar.gz)
```pwd /opt/src
[root@hdss7-200 src]# ls -l|grep tomcat
-rw-r--r-- 1 root root   9690027 Apr 10 22:57 apache-tomcat-8.5.40.tar.gz
[root@hdss7-200 src]# mkdir -p /data/dockerfile/tomcat8 && tar xf apache-tomcat-8.5.40.tar.gz -C /data/dockerfile/tomcat
[root@hdss7-200 src]# cd /data/dockerfile/tomcat8 && rm -fr apache-tomcat-8.5.40/webapps/*
```

### 简单配置tomcat
1. 关闭AJP端口

```vi /data/dockerfile/tomcat/apache-tomcat-8.5.40/conf/server.xml
<!-- <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" /> -->
```
2. 配置日志

- 删除3manager，4host-manager的handlers

```vi /data/dockerfile/tomcat/apache-tomcat-8.5.40/conf/logging.properties
handlers = 1catalina.org.apache.juli.AsyncFileHandler, 2localhost.org.apache.juli.AsyncFileHandler,java.util.logging.ConsoleHandler
```
- 日志级别改为INFO

```vi /data/dockerfile/tomcat/apache-tomcat-8.5.40/conf/logging.properties
1catalina.org.apache.juli.AsyncFileHandler.level = INFO
2localhost.org.apache.juli.AsyncFileHandler.level = INFO
java.util.logging.ConsoleHandler.level = INFO
```
- 注释掉所有关于3manager，4host-manager日志的配置

```vi /data/dockerfile/tomcat/apache-tomcat-8.5.40/conf/logging.properties
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
ADD apache-tomcat-8.5.40/ /opt/tomcat
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
JAVA_OPTS="${E_OPTS} ${C_OPTS} -Xms${MIN_HEAP} -Xmx${MAX_HEAP} ${JAVA_OPTS}"
sed -i -e "1a\JAVA_OPTS=\"$JAVA_OPTS\"" -e "1a\CATALINA_OPTS=\"$CATALINA_OPTS\"" /opt/tomcat/bin/catalina.sh

cd /opt/tomcat && /opt/tomcat/bin/catalina.sh run

{% endcode %}
<!-- endtab -->
{% endtabs %}

### 制作镜像并推送
```
[root@hdss7-200 tomcat]# docker build . -t harbor.od.com/base/tomcat:v8.5.40
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
Step 5 : ADD apache-tomcat-8.5.40/ /opt/tomcat
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

[root@hdss7-200 tomcat8]# docker push harbor.od.com/base/tomcat:v8.5.40
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
v8.5.40: digest: sha256:407c6abd7c4fa5119376efa71090b49637d7a52ef2fc202f4019ab4c91f6dc50 size: 2409
```
## 改造dubbo-demo-web项目
### 修改dubbo-client/pom.xml
```vi /d/workspace/dubbo-demo-web/dubbo-client/pom.xml
<packaging>war</packaging>

<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
  <exclusions>
    <exclusion>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-tomcat</artifactId>
    </exclusion>
  </exclusions>
</dependency>

<dependency>
  <groupId>org.apache.tomcat</groupId> 
  <artifactId>tomcat-servlet-api</artifactId>
  <version>8.0.36</version>
  <scope>provided</scope>
</dependency>
```

### 修改Application.java
```vi /d/workspace/dubbo-demo-web/dubbo-client/src/main/java/com/od/dubbotest/Application.java
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration;
import org.springframework.context.annotation.ImportResource;

@ImportResource(value={"classpath*:spring-config.xml"})
@EnableAutoConfiguration(exclude={DataSourceAutoConfiguration.class})
```

### 创建ServletInitializer.java
```vi /d/workspace/dubbo-demo-web/dubbo-client/src/main/java/com/od/dubbotest/ServletInitializer.java
package com.od.dubbotest;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.builder.SpringApplicationBuilder;
import org.springframework.boot.context.web.SpringBootServletInitializer;
import com.od.dubbotest.Application;

public class ServletInitializer extends SpringBootServletInitializer {

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        return builder.sources(Application.class);
    }
}
```

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
> - base/tomcat:v8.5.40
> - base/tomcat:v9.0.17
> Description : project base image list in harbor.od.com.

10. Add Parameter -> Choice Parameter
> Name : maven
> Default Value : 
> - 3.6.0-8u181
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
-rw-r----- 1 root root 7435 Jan 19 19:26 catalina.2019-01-19.log
-rw-r----- 1 root root  629 Jan 19 19:26 localhost.2019-01-19.log
-rw-r----- 1 root root  249 Jan 15 19:27 localhost_access_log.2019-01-19.txt
```

# 使用Prometheus和Grafana监控kubernetes集群
## 部署kube-state-metrics
运维主机`HDSS7-200.host.com`
### 准备kube-state-metrics镜像
[kube-state-metrics官方quay.io地址](https://quay.io/repository/coreos/kube-state-metrics?tab=info)
```
[root@hdss7-200 ~]# docker pull quay.io/coreos/kube-state-metrics:v1.5.0
v1.5.0: Pulling from coreos/kube-state-metrics
cd784148e348: Pull complete 
f622528a393e: Pull complete 
Digest: sha256:b7a3143bd1eb7130759c9259073b9f239d0eeda09f5210f1cd31f1a530599ea1
Status: Downloaded newer image for quay.io/coreos/kube-state-metrics:v1.5.0
[root@hdss7-200 ~]# docker tag 91599517197a harbor.od.com/k8s/kube-state-metrics:v1.5.0
[root@hdss7-200 ~]# docker push harbor.od.com/k8s/kube-state-metrics:v1.5.0
docker push harbor.od.com/k8s/kube-state-metrics:v1.5.0
The push refers to a repository [harbor.od.com/k8s/kube-state-metrics]
5b3c36501a0a: Pushed 
7bff100f35cb: Pushed 
v1.5.0: digest: sha256:0d9bea882f25586543254d9ceeb54703eb6f8f94c0dd88875df216b6a0f60589 size: 739
```

### 准备资源配置清单
{% tabs kube-state-metrics%}
<!-- tab RBAC -->
vi /data/k8s-yaml/kube-state-metrics/rbac.yaml
{% code %}
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/cluster-service: "true"
  name: kube-state-metrics
  namespace: kube-system
-\--
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/cluster-service: "true"
  name: kube-state-metrics
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  - secrets
  - nodes
  - pods
  - services
  - resourcequotas
  - replicationcontrollers
  - limitranges
  - persistentvolumeclaims
  - persistentvolumes
  - namespaces
  - endpoints
  verbs:
  - list
  - watch
- apiGroups:
  - extensions
  resources:
  - daemonsets
  - deployments
  - replicasets
  verbs:
  - list
  - watch
- apiGroups:
  - apps
  resources:
  - statefulsets
  verbs:
  - list
  - watch
- apiGroups:
  - batch
  resources:
  - cronjobs
  - jobs
  verbs:
  - list
  - watch
- apiGroups:
  - autoscaling
  resources:
  - horizontalpodautoscalers
  verbs:
  - list
  - watch
-\--
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/cluster-service: "true"
  name: kube-state-metrics
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-state-metrics
subjects:
- kind: ServiceAccount
  name: kube-state-metrics
  namespace: kube-system
{% endcode %}
<!-- endtab -->
<!-- tab Deployment-->
vi /data/k8s-yaml/kube-state-metrics/deployment.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "2"
  labels:
    grafanak8sapp: "true"
    app: kube-state-metrics
  name: kube-state-metrics
  namespace: kube-system
spec:
  selector:
    matchLabels:
      grafanak8sapp: "true"
      app: kube-state-metrics
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        grafanak8sapp: "true"
        app: kube-state-metrics
    spec:
      containers:
      - image: harbor.od.com/k8s/kube-state-metrics:v1.5.0
        name: kube-state-metrics
        ports:
        - containerPort: 8080
          name: http-metrics
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 5
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 5
        imagePullPolicy: IfNotPresent
      imagePullSecrets:
      - name: harbor
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      serviceAccount: kube-state-metrics
      serviceAccountName: kube-state-metrics
{% endcode %}
<!-- endtab -->
{% endtabs %}

### 应用资源配置清单
任意一台运算节点上：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/kube-state-metrics/rbac.yaml 
serviceaccount/kube-state-metrics created
clusterrole.rbac.authorization.k8s.io/kube-state-metrics created
clusterrolebinding.rbac.authorization.k8s.io/kube-state-metrics created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/kube-state-metrics/deployment.yaml 
deployment.extensions/kube-state-metrics created
```

### 检查启动情况
```
[root@hdss7-21 ~]# kubectl get pods -n kube-system|grep kube-state-metrics
kube-state-metrics-645bd94c55-wdtqh      1/1     Running   0          94s
```

## 部署node-exporter
运维主机`HDSS7-200.host.com`上：
### 准备node-exporter镜像
[node-exporter官方dockerhub地址](https://hub.docker.com/r/prom/node-exporter)
[node-expoerer官方github地址](https://github.com/prometheus/node_exporter)
```
[root@hdss7-200 ~]# docker pull prom/node-exporter:v0.17.0
v0.17.0: Pulling from prom/node-exporter
0de338cf4258: Pull complete 
f508012419d8: Pull complete 
d764f7880123: Pull complete 
Digest: sha256:c390c8fea4cd362a28ad5070aedd6515aacdfdffd21de6db42ead05e332be5a9
Status: Downloaded newer image for prom/node-exporter:v0.17.0
[root@hdss7-200 ~]# docker tag b3e7f67a1480 harbor.od.com/k8s/node-exporter:v0.17.0
[root@hdss7-200 ~]# docker push harbor.od.com/k8s/node-exporter:v0.17.0
docker push harbor.od.com/k8s/node-exporter:v0.17.0
The push refers to a repository [harbor.od.com/k8s/node-exporter]
0bf893ee7433: Pushed 
17ab2671f87e: Pushed 
b8873621dfbc: Pushed 
v0.17.0: digest: sha256:4e13dd75f00a6114675ea3e62e61dbd79dcb2205e8f3bbe1f8f8ef2fd3e28113 size: 949
```

### 准备资源配置清单
```vi /data/k8s-yaml/node-exporter/node-exporter-ds.yaml
kind: DaemonSet
apiVersion: extensions/v1beta1
metadata:
  name: node-exporter
  namespace: kube-system
  labels:
    daemon: "node-exporter"
    grafanak8sapp: "true"
spec:
  selector:
    matchLabels:
      daemon: "node-exporter"
      grafanak8sapp: "true"
  template:
    metadata:
      name: node-exporter
      labels:
        daemon: "node-exporter"
        grafanak8sapp: "true"
    spec:
      volumes:
      - name: proc
        hostPath: 
          path: /proc
          type: ""
      - name: sys
        hostPath:
          path: /sys
          type: ""
      containers:
      - name: node-exporter
        image: harbor.od.com/k8s/node-exporter:v0.17.0
        args:
        - --path.procfs=/host_proc
        - --path.sysfs=/host_sys
        ports:
        - name: node-exporter
          hostPort: 9100
          containerPort: 9100
          protocol: TCP
        volumeMounts:
        - name: sys
          readOnly: true
          mountPath: /host_sys
        - name: proc
          readOnly: true
          mountPath: /host_proc
      imagePullSecrets:
      - name: harbor
      restartPolicy: Always
      dnsPolicy: ClusterFirst
      hostNetwork: true
```

### 应用资源配置清单
任意运算节点上：
```
[root@hdss7-21 ~]# kubectl -f http://k8s-yaml.od.com/node-exporter/node-exporter-ds.yaml
daemonset.extensions/node-exporter created
```

## 部署blackbox-exporter
运维主机`HDSS7-200.host.com`上：
### 准备blackbox-exporter镜像
[blackbox-exporter官方dockerhub地址](https://hub.docker.com/r/prom/blackbox-exporter)
[blackbox-exporter官方github地址](https://github.com/prometheus/blackbox_exporter)
```
[root@hdss7-200 ~]# docker pull prom/blackbox-exporter:v0.14.0
v0.14.0: Pulling from prom/blackbox-exporter
697743189b6d: Pull complete 
f1989cfd335b: Pull complete 
8918f7b8f34f: Pull complete 
a128dce6256a: Pull complete 
Digest: sha256:c20445e0cc628fa4b227fe2f694c22a314beb43fd8297095b6ee6cbc67161336
Status: Downloaded newer image for prom/blackbox-exporter:v0.14.0
[root@hdss7-200 ~]# docker tag d3a00aea3a01 harbor.od.com/k8s/blackbox-exporter:v0.14.0
[root@hdss7-200 ~]# docker push harbor.od.com/k8s/blackbox-exporter:v0.14.0
docker push harbor.od.com/k8s/blackbox-exporter:v0.14.0
The push refers to a repository [harbor.od.com/k8s/blackbox-exporter]
256c4aa8ebe5: Pushed 
4b6cc55de649: Pushed 
986894c42222: Mounted from infra/prometheus 
adab5d09ba79: Mounted from infra/prometheus 
v0.14.0: digest: sha256:b127897cf0f67c47496d5cbb7ecb86c001312bddd04192f6319d09292e880a5f size: 1155
```

### 准备资源配置清单
{% tabs blackbox-exporter%}
<!-- tab ConfigMap -->
vi /data/k8s-yaml/blackbox-exporter/configmap.yaml
{% code %}
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app: blackbox-exporter
  name: blackbox-exporter
  namespace: kube-system
data:
  blackbox.yml: |-
    modules:
      http_2xx:
        prober: http
        timeout: 2s
        http:
          valid_http_versions: ["HTTP/1.1", "HTTP/2"]
          valid_status_codes: [200,301,302]
          method: GET
          preferred_ip_protocol: "ip4"
      tcp_connect:
        prober: tcp
        timeout: 2s
{% endcode %}
<!-- endtab -->
<!-- tab Deployment -->
vi /data/k8s-yaml/blackbox-exporter/deployment.yaml
{% code %}
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: blackbox-exporter
  namespace: kube-system
  labels:
    app: blackbox-exporter
  annotations:
    deployment.kubernetes.io/revision: 1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blackbox-exporter
  template:
    metadata:
      labels:
        app: blackbox-exporter
    spec:
      volumes:
      - name: config
        configMap:
          name: blackbox-exporter
          defaultMode: 420
      containers:
      - name: blackbox-exporter
        image: harbor.od.com/k8s/blackbox-exporter:v0.14.0
        args:
        - -\-config.file=/etc/blackbox_exporter/blackbox.yml
        - -\-log.level=debug
        - -\-web.listen-address=:9115
        ports:
        - name: blackbox-port
          containerPort: 9115
          protocol: TCP
        resources:
          limits:
            cpu: 200m
            memory: 256Mi
          requests:
            cpu: 100m
            memory: 50Mi
        volumeMounts:
        - name: config
          mountPath: /etc/blackbox_exporter
        readinessProbe:
          tcpSocket:
            port: 9115
          initialDelaySeconds: 5
          timeoutSeconds: 5
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
        imagePullPolicy: IfNotPresent
      imagePullSecrets:
      - name: harbor
      restartPolicy: Always
      dnsPolicy: ClusterFirst
{% endcode %}
<!-- endtab -->
<!-- tab Service-->
vi /data/k8s-yaml/blackbox-exporter/service.yaml
{% code %}
kind: Service
apiVersion: v1
metadata:
  name: blackbox-exporter
  namespace: kube-system
spec:
  selector:
    app: blackbox-exporter
  ports:
    - protocol: TCP
      port: 9115
      name: http
{% endcode %}
<!-- endtab -->
<!-- tab Ingress-->
vi /data/k8s-yaml/blackbox-exporter/ingress.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: blackbox-exporter
  namespace: kube-system
spec:
  rules:
  - host: blackbox.od.com
    http:
      paths:
      - backend:
          serviceName: blackbox-exporter
          servicePort: 9115
{% endcode %}
<!-- endtab -->
{% endtabs %}

### 解析域名
`HDSS7-11.host.com`上
```vi /var/named/od.com.zone
blackbox	60 IN A 10.4.7.10
```

### 应用资源配置清单
任意运算节点上：
```
[root@hdss7-21 ~]# kubectl -f http://k8s-yaml.od.com/blackbox-exporter/configmap.yaml
configmap/blackbox-exporter created
[root@hdss7-21 ~]# kubectl -f http://k8s-yaml.od.com/blackbox-exporter/deployment.yaml
deployment.extensions/blackbox-exporter created
[root@hdss7-21 ~]# kubectl -f http://k8s-yaml.od.com/blackbox-exporter/service.yaml
service/blackbox-exporter created
[root@hdss7-21 ~]# kubectl -f http://k8s-yaml.od.com/blackbox-exporter/ingress.yaml
ingress.extensions/blackbox-exporter created
```

### 浏览器访问
http://blackbox.od.com

![blackbox-exporter](/images/blackbox.png "blackbox-exporter")


## 部署prometheus
运维主机`HDSS7-200.host.com`上：
### 准备prometheus镜像
[prometheus官方dockerhub地址](https://hub.docker.com/r/prom/prometheus)
[prometheus官方github地址](https://github.com/prometheus/prometheus)
```
[root@hdss7-200 ~]# docker pull prom/prometheus:v2.9.1
v2.9.1: Pulling from prom/prometheus
697743189b6d: Pull complete 
f1989cfd335b: Pull complete 
b60c2f039ea7: Pull complete 
6a189f2c500c: Pull complete 
bd6be4aea906: Pull complete 
81b69caae2b5: Pull complete 
5e7226eda004: Pull complete 
564568254ec8: Pull complete 
9bd07902fc4b: Pull complete 
Digest: sha256:e02bb3dec47631b4d31cede2d35ff901d892b57f33144406ee7994e8c94fb2d7
Status: Downloaded newer image for prom/prometheus:v2.9.1
[root@hdss7-200 ~]# docker tag 4737a2d79d1a harbor.od.com/infra/prometheus:v2.9.1
[root@hdss7-200 ~]# docker push harbor.od.com/infra/prometheus:v2.9.1
The push refers to a repository [harbor.od.com/infra/prometheus]
a67e5326fa35: Pushed 
02c0c0b3065f: Pushed 
b7b1f5015c12: Pushed 
38be466f60e1: Pushed 
9c3fb6c27da7: Pushed 
a06c616e78e9: Pushed 
05c0d5c2ae72: Pushed 
986894c42222: Pushed 
adab5d09ba79: Pushed 
v2.9.1: digest: sha256:2357e59541f5596dd90d9f4218deddecd9b4880c1e417a42592b00b30b47b0a9 size: 2198
```

### 准备资源配置清单
{% tabs prometheus%}
<!-- tab RBAC-->
vi /data/k8s-yaml/prometheus/rbac.yaml
{% code %}
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/cluster-service: "true"
  name: prometheus
  namespace: infra
-\--
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/cluster-service: "true"
  name: prometheus
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  - nodes/metrics
  - services
  - endpoints
  - pods
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - configmaps
  verbs:
  - get
- nonResourceURLs:
  - /metrics
  verbs:
  - get
-\--
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/cluster-service: "true"
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: infra
{% endcode %}
<!-- endtab -->
<!-- tab Deployment -->
vi /data/k8s-yaml/prometheus/deployment.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "5"
  labels:
    name: prometheus
  name: prometheus
  namespace: infra
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 7
  selector:
    matchLabels:
      app: prometheus
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      nodeName: 10.4.7.21
      containers:
      - image: harbor.od.com/infra/prometheus:v2.9.1
        args:
        - -\-config.file=/data/etc/prometheus.yml
        - -\-storage.tsdb.path=/data/prom-db
        - -\-storage.tsdb.retention=72h
        command:
        - /bin/prometheus
        name: prometheus
        ports:
        - containerPort: 9090
          protocol: TCP
        resources:
          limits:
            cpu: 500m
            memory: 2500Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - mountPath: /data
          name: data
        imagePullPolicy: IfNotPresent
      imagePullSecrets:
      - name: harbor
      securityContext:
        runAsUser: 0
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      serviceAccount: prometheus
      serviceAccountName: prometheus
      volumes:
      - hostPath:
          path: /data/k8s-volume/prometheus
          type: ""
        name: data

{% endcode %}
<!-- endtab -->
<!-- tab Service-->
vi /data/k8s-yaml/prometheus/service.yaml
{% code %}
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: infra
spec:
  ports:
  - port: 9090
    protocol: TCP
    name: prometheus
  selector:
    app: prometheus
  type: ClusterIP
{% endcode %}
<!-- endtab -->
<!-- tab Ingress-->
vi /data/k8s-yaml/prometheus/ingress.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: traefik
  name: prometheus
  namespace: infra
spec:
  rules:
  - host: prometheus.od.com
    http:
      paths:
      - backend:
          serviceName: prometheus
          servicePort: 9090
{% endcode %}
<!-- endtab -->
{% endtabs %}

### 准备prometheus的配置文件
运算节点`HDSS7-21.host.com`上：
- 拷贝证书

```
[root@hdss7-21 ~]# mkdir -pv /data/k8s-volume/prometheus/{etc,prom-db}
mkdir: created directory ‘/data/k8s-volume/prometheus/etc’
mkdir: created directory ‘/data/k8s-volume/prometheus/prom-db’
[root@hdss7-21 ~]# cd /data/k8s-volume/prometheus/etc
[root@hdss7-21 ~]# scp root@10.4.7.200:/opt/certs/ca.pem .
[root@hdss7-21 ~]# scp root@10.4.7.200:/opt/certs/client.pem .
[root@hdss7-21 ~]# scp root@10.4.7.200:/opt/certs/client-key.pem .
```
- 准备配置

```vi /data/k8s-volume/promethues/etc/prometheus.yml
global:
  scrape_interval:     15s
  evaluation_interval: 15s
scrape_configs:
- job_name: 'etcd'
  tls_config:
    ca_file: /data/etc/ca.pem
    cert_file: /data/etc/client.pem
    key_file: /data/etc/client-key.pem
  scheme: https
  static_configs:
  - targets:
    - '10.4.7.12:2379'
    - '10.4.7.21:2379'
    - '10.4.7.22:2379'
- job_name: 'kubernetes-apiservers'
  kubernetes_sd_configs:
  - role: endpoints
  scheme: https
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  relabel_configs:
  - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
    action: keep
    regex: default;kubernetes;https
- job_name: 'kubernetes-pods'
  kubernetes_sd_configs:
  - role: pod
  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    action: keep
    regex: true
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
    action: replace
    target_label: __metrics_path__
    regex: (.+)
  - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
    action: replace
    regex: ([^:]+)(?::\d+)?;(\d+)
    replacement: $1:$2
    target_label: __address__
  - action: labelmap
    regex: __meta_kubernetes_pod_label_(.+)
  - source_labels: [__meta_kubernetes_namespace]
    action: replace
    target_label: kubernetes_namespace
  - source_labels: [__meta_kubernetes_pod_name]
    action: replace
    target_label: kubernetes_pod_name
- job_name: 'kubernetes-kubelet'
  kubernetes_sd_configs:
  - role: node
  relabel_configs:
  - action: labelmap
    regex: __meta_kubernetes_node_label_(.+)
  - source_labels: [__meta_kubernetes_node_name]
    regex: (.+)
    target_label: __address__
    replacement: ${1}:10255
- job_name: 'kubernetes-cadvisor'
  kubernetes_sd_configs:
  - role: node
  relabel_configs:
  - action: labelmap
    regex: __meta_kubernetes_node_label_(.+)
  - source_labels: [__meta_kubernetes_node_name]
    regex: (.+)
    target_label: __address__
    replacement: ${1}:4194
- job_name: 'kubernetes-kube-state'
  kubernetes_sd_configs:
  - role: pod
  relabel_configs:
  - action: labelmap
    regex: __meta_kubernetes_pod_label_(.+)
  - source_labels: [__meta_kubernetes_namespace]
    action: replace
    target_label: kubernetes_namespace
  - source_labels: [__meta_kubernetes_pod_name]
    action: replace
    target_label: kubernetes_pod_name
  - source_labels: [__meta_kubernetes_pod_label_grafanak8sapp]
    regex: .*true.*
    action: keep
  - source_labels: ['__meta_kubernetes_pod_label_daemon', '__meta_kubernetes_pod_node_name']
    regex: 'node-exporter;(.*)'
    action: replace
    target_label: nodename
- job_name: 'blackbox_http_pod_probe'
  metrics_path: /probe
  kubernetes_sd_configs:
  - role: pod
  params:
    module: [http_2xx]
  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_annotation_blackbox_scheme]
    action: keep
    regex: http
  - source_labels: [__address__, __meta_kubernetes_pod_annotation_blackbox_port,  __meta_kubernetes_pod_annotation_blackbox_path]
    action: replace
    regex: ([^:]+)(?::\d+)?;(\d+);(.+)
    replacement: $1:$2$3
    target_label: __param_target
  - action: replace
    target_label: __address__
    replacement: http://blackbox.od.com
  - source_labels: [__param_target]
    target_label: instance
  - action: labelmap
    regex: __meta_kubernetes_pod_label_(.+)
  - source_labels: [__meta_kubernetes_namespace]
    action: replace
    target_label: kubernetes_namespace
  - source_labels: [__meta_kubernetes_pod_name]
    action: replace
    target_label: kubernetes_pod_name
- job_name: 'blackbox_tcp_pod_probe'
  metrics_path: /probe
  kubernetes_sd_configs:
  - role: pod
  params:
    module: [tcp_connect]
  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_annotation_blackbox_scheme]
    action: keep
    regex: tcp
  - source_labels: [__address__, __meta_kubernetes_pod_annotation_blackbox_port]
    action: replace
    regex: ([^:]+)(?::\d+)?;(\d+)
    replacement: $1:$2
    target_label: __param_target
  - action: replace
    target_label: __address__
    replacement: http://blackbox.od.com
  - source_labels: [__param_target]
    target_label: instance
  - action: labelmap
    regex: __meta_kubernetes_pod_label_(.+)
  - source_labels: [__meta_kubernetes_namespace]
    action: replace
    target_label: kubernetes_namespace
  - source_labels: [__meta_kubernetes_pod_name]
    action: replace
    target_label: kubernetes_pod_name
- job_name: 'traefik'
  kubernetes_sd_configs:
  - role: pod
  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scheme]
    action: keep
    regex: traefik
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
    action: replace
    target_label: __metrics_path__
    regex: (.+)
  - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
    action: replace
    regex: ([^:]+)(?::\d+)?;(\d+)
    replacement: $1:$2
    target_label: __address__
  - action: labelmap
    regex: __meta_kubernetes_pod_label_(.+)
  - source_labels: [__meta_kubernetes_namespace]
    action: replace
    target_label: kubernetes_namespace
  - source_labels: [__meta_kubernetes_pod_name]
    action: replace
    target_label: kubernetes_pod_name
```

### 应用资源配置清单
任意运算节点上：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/prometheus/rbac.yaml
serviceaccount/prometheus created
clusterrole.rbac.authorization.k8s.io/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/prometheus/deployment.yaml
deployment.extensions/prometheus created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/prometheus/service.yaml
service/prometheus created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/prometheus/ingress.yaml
ingress.extensions/prometheus created
```
## 部署Grafana

# 使用ELK Stack收集kubernetes集群内的应用日志
## 系统架构
## 安装部署配置
## Kibana的使用
