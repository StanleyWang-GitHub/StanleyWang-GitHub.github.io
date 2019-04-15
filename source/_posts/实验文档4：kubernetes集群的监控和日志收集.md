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

# 使用Promethus和Grafana监控kubernetes集群
## 安装部署
## 监控集群
## 监控dubbo微服务

# 使用ELK Stack收集kubernetes集群内的应用日志
## 系统架构
## 安装部署配置
## Kibana的使用
