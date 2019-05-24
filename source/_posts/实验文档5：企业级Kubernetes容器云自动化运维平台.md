title: 实验文档5：企业级kubernetes容器云自动化运维平台
author: Stanley Wang
categories: Kubernetes容器云技术专题
date: 2019-1-18 18:12:56
---
- - -
{% cq %}欢迎加入王导的VIP学习qq群：==>[<font color="FF7F50">932194668</font>](http://shang.qq.com/wpa/qunwpa?idkey=78869fddc5a661acb0639315eb52997c108de6625df5f0ee2f0372f176a032a6)<=={% endcq %}
- - -
# 部署对象式存储minio
运维主机`HDSS7-200.host.com`上：
## 准备docker镜像
[镜像下载地址](https://hub.docker.com/r/minio/minio)
```
[root@hdss7-200 ~]# docker pull minio/minio:latest
latest: Pulling from minio/minio
e7c96db7181b: Pull complete 
b17880043800: Pull complete 
e5fc8b080393: Pull complete 
3b5383aad564: Pull complete 
Digest: sha256:1a594faffab833866e43154c31b943d49ca1146d2c164ec3279382d7530d2ede
Status: Downloaded newer image for minio/minio:latest
[root@hdss7-200 ~]# docker tag ed5e2b2624bf harbor.od.com/spinnaker/minio:latest
[root@hdss7-200 ~]# docker push harbor.od.com/spinnaker/minio:latest
The push refers to a repository [harbor.od.com/spinnaker/minio]
e8c528077536: Pushed 
ad774a44659e: Pushed 
1a734f934625: Pushed 
f1b5933fe4b5: Pushed 
latest: digest: sha256:a676c2f71ad261d10cba2527094810e8a37f98d79086694ec162cb7fea64f5e0 size: 1158
```

## 准备资源配置清单
{% tabs minio %}
<!-- tab Deployment -->
vi /data/k8s-yaml/minio/deployment.yaml
{% code %}
kind: Deployment
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    name: minio
  name: minio
  namespace: spinnaker
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 7
  selector:
    matchLabels:
      name: minio
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: minio
        name: minio
    spec:
      containers:
      - args:
        - server
        - /data
        env:
        - name: MINIO_ACCESS_KEY
          value: admin
        - name: MINIO_SECRET_KEY
          value: poiuytrewq
        image: harbor.od.com/spinnaker/minio:latest
        imagePullPolicy: IfNotPresent
        name: minio
        ports:
        - containerPort: 9000
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /minio/health/ready
            port: 9000
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 5
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /data
          name: data
      imagePullSecrets:
      - name: harbor
      restartPolicy: Always
      schedulerName: default-scheduler
      terminationGracePeriodSeconds: 30
      volumes:
      - nfs:
          server: hdss7-200
          path: /data/nfs-volume/minio
        name: data
{% endcode %}
<!-- endtab -->
<!-- tab Service -->
vi /data/k8s-yaml/minio/svc.yaml
{% code %}
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: spinnaker
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 9000
  selector:
    app: minio
  type: ClusterIP
{% endcode %}
<!-- endtab -->
<!-- tab Ingress -->
vi /data/k8s-yaml/minio/ingress.yaml
{% code %}
kind: Ingress
apiVersion: extensions/v1beta1
metadata: 
  name: minio
  namespace: spinnaker
spec:
  rules:
  - host: minio.od.com
    http:
      paths:
      - path: /
        backend: 
          serviceName: minio
          servicePort: 80
{% endcode %}
<!-- endtab -->
{% endtabs %}

## 解析域名
`HDSS7-11.host.com`上
```vi /var/named/od.com.zone
minio 	60 IN A 10.4.7.10
```
## 应用资源配置清单
任意运算节点上：
```
[root@hdss7-21 ~]# kubectl apply -f https://k8s-yaml.od.com/minio/deployment.yaml 
deployment.extensions/minio created
[root@hdss7-21 ~]# kubectl apply -f https://k8s-yaml.od.com/minio/svc.yaml 
service/minio created
[root@hdss7-21 ~]# kubectl apply -f https://k8s-yaml.od.com/minio/ingress.yaml 
ingress.extensions/minio created
```
## 浏览器访问
http://minio.od.com

# 部署Redis
## 准备docker镜像
运维主机`HDSS7-200.host.com`上：
[镜像下载地址](https://hub.docker.com/_/redis)
```
[root@hdss7-200 ~]# docker pull redis:4.0.14
4.0.14: Pulling from library/redis
743f2d6c1f65: Pull complete 
171658c5966d: Pull complete 
fbef10bd7a65: Pull complete 
1111aefffd16: Pull complete 
3b9af921a085: Pull complete 
e0659698d44a: Pull complete 
Digest: sha256:2e82b9fc2c0396f6056b180743f5cf05219908b9e0c9e7046041588ebd1363d1
Status: Downloaded newer image for redis:4.0.14
[root@hdss7-200 ~]# docker tag 720768125f4f harbor.od.com/spinnaker/redis:v4.0.14
[root@hdss7-200 ~]# docker push harbor.od.com/spinnaker/redis:v4.0.14
docker push harbor.od.com/spinnaker/redis:v4.0.14
The push refers to a repository [harbor.od.com/spinnaker/redis]
675b01d6b533: Pushed 
a6104c28b530: Pushed 
49be6cb1f430: Pushed 
03eafa792876: Pushed 
f99f83132c0a: Pushed 
6270adb5794c: Pushed 
v4.0.14: digest: sha256:b1aaa4a4e2c0ae115c1bae57ce0f957dd9440f52fae42af9e1c2e165ea7e0aba size: 1571
```
## 准备资源配置清单
{% tabs redis %}
<!-- tab Deployment -->
vi /data/k8s-yaml/redis/deployment.yaml
{% code %}
kind: Deployment
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    name: redis
  name: redis
  namespace: spinnaker
spec:
  replicas: 1
  revisionHistoryLimit: 7
  selector:
    matchLabels:
      name: redis
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: redis
        name: redis
    spec:
      containers:
      - name: redis
        image: harbor.od.com/spinnaker/redis:v4.0.14
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 6379
          protocol: TCP
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      imagePullSecrets:
      - name: harbor
      restartPolicy: Always
      schedulerName: default-scheduler
      terminationGracePeriodSeconds: 30
{% endcode %}
<!-- endtab -->
<!-- tab Service -->
vi /data/k8s-yaml/redis/svc.yaml
{% code %}
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: spinnaker
spec:
  ports:
  - port: 6379
    protocol: TCP
    targetPort: 6379
  selector:
    app: redis
  type: ClusterIP
{% endcode %}
<!-- endtab -->
{% endtabs %}

## 应用资源配置清单
任意运算节点上：
```
[root@hdss7-21 ~]# kubectl apply -f https://k8s-yaml.od.com/redis/deployment.yaml 
deployment.extensions/redis created
[root@hdss7-21 ~]# kubectl apply -f https://k8s-yaml.od.com/redis/svc.yaml 
service/redis created
```

# 部署CloudDriver
运维主机`HDSS7-200.host.com`上：
## 准备docker镜像
[镜像下载地址](https://quay.io/repository/container-image/spinnaker-clouddriver)
```
[root@hdss7-200 ~]# docker pull quay.io/container-image/spinnaker-clouddriver:4.3.0-20190128134206
4.3.0-20190128134206: Pulling from container-image/spinnaker-clouddriver

35920a071f91: Pull complete 
88b05767cad1: Pull complete 
3f8bd0101cd3: Pull complete 
1f055e83a236: Pull complete 
144fd385f584: Pull complete 
f6c8f5d7f704: Pull complete 
613069df7b32: Pull complete 
e4ae8f830423: Pull complete 
Digest: sha256:34cf9c03911fc2fc8ec728f8c3eecfe04a82705bc5d6ce70223235bf337c2fc2
Status: Downloaded newer image for quay.io/container-image/spinnaker-clouddriver:4.3.0-20190128134206
[root@hdss7-200 ~]# docker tag be10393ea45e harbor.od.com/spinnaker/clouddriver:v4.3.0
[root@hdss7-200 ~]# docker push harbor.od.com/spinnaker/clouddriver:v4.3.0
The push refers to a repository [harbor.od.com/spinnaker/clouddriver]
060c6a71a4b8: Pushed 
fe8ea4bcebb0: Pushed 
34cab728765a: Pushed 
a7c1f802e045: Pushed 
46abf18f621c: Pushed 
7bbd7cb9c0eb: Pushed 
b845f393fc2b: Pushed 
dbc783c89851: Pushed 
7bff100f35cb: Pushed 
v4.3.0: digest: sha256:ff075168d339ada5a474d09f9b875bbd713eb3f3815b81f15a10b2a442deaa00 size: 2216
```
## 准备cluster-admin用户配置
运维主机`HDSS7-200.host.com`上：
- 签发admin.pem、admin-key.pem
> 参考实验文档1

- 做admin.kubeconfig

```pwd /opt/certs
[root@hdss7-200 certs]# kubectl config set-cluster myk8s --certificate-authority=./ca.pem --embed-certs=true --server=https://10.4.7.10:7443 --kubeconfig=config
Cluster "myk8s" set.
[root@hdss7-200 certs]# kubectl config set-credentials cluster-admin --client-certificate=./admin.pem --client-key=./admin-key.pem --embed-certs=true --kubeconfig=config
User "cluster-admin" set.
[root@hdss7-200 certs]# kubectl config set-context myk8s-context --cluster=myk8s --user=cluster-admin --kubeconfig=config
Context "myk8s-context" created.
[root@hdss7-200 certs]# kubectl config use-context myk8s-context --kubeconfig=config
Switched to context "myk8s-context".
[root@hdss7-200 certs]# kubectl create clusterrolebinding myk8s-admin --clusterrole=cluster-admin --user=cluster-admin
clusterrolebinding.rbac.authorization.k8s.io/myk8s-admin created
```

- 验证
> 将config文件拷贝至任意运算节点/root/.kube下，使用kubectl验证

- 创建cm

```pwd /root/.kube
[root@hdss7-21 .kube]# kubectl create cm kubeconfig --from-file=config -n spinnaker
```

## 准备资源配置清单
{% tabs clouddriver %}
<!-- tab ConfigMap -->
vi /data/k8s-yaml/clouddriver/cm.yaml
{% code %}
kind: ConfigMap
apiVersion: v1
metadata:
  name: clouddriver
  namespace: spinnaker
data:
  clouddriver.yml: |
    server:
      port: 7002
      ssl:
        enabled: false
    redis:
      connection: redis://redis:6379
      scheduler: default
      parallelism: -1

    services:
      front50:
        baseUrl: http://front50

    management.health.elasticsearch.enabled: false
    kubernetes:
      enabled: true
      accounts:
        - name: cluster-admin
          serviceAccount: false
          dockerRegistries:
            - accountName: harbor-admin
              namespaces: []
          namespaces:
            - spinnaker
            - infra
            - app-test
            - app-prod
          kubeconfigFile:  /etc/kubernetes/config
      primaryAccount: cluster-admin
    dockerRegistry:
      enabled: true
      accounts:
        - name: harbor-admin
          requiredGroupMembership: []
          providerVersion: V1
          insecureRegistry: true
          address: http://harbor.od.com
          username: admin
          password: Harbor12345
      primaryAccount: harbor-admin
{% endcode %}
<!-- endtab -->
<!-- tab Deployment -->
vi /data/k8s-yaml/clouddriver/deployment.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: clouddriver
  name: clouddriver
  namespace: spinnaker
spec:
  replicas: 1
  revisionHistoryLimit: 7
  selector:
    matchLabels:
      app: clouddriver
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: clouddriver
    spec:
      containers:
      - image: harbor.od.com/spinnaker/clouddriver:v4.3.0
        imagePullPolicy: IfNotPresent
        name: clouddriver
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /etc/kubernetes
          name: kubeconfig
        - mountPath: /opt/clouddriver/config
          name: clouddriver
      imagePullSecrets:
      - name: harbor
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      volumes:
      - configMap:
          defaultMode: 420
          name: kubeconfig
        name: kubeconfig
      - configMap:
          defaultMode: 420
          name: clouddriver
        name: clouddriver
{% endcode %}
<!-- endtab -->
<!-- tab Service -->
vi /data/k8s-yaml/clouddriver/svc.yaml
{% code %}
apiVersion: v1
kind: Service
metadata:
  name: clouddriver
  namespace: spinnaker
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 7002
  selector:
    app: clouddriver
  type: ClusterIP
{% endcode %}
<!-- endtab -->
{% endtabs %}

## 应用资源配置清单
任意一台运算节点上：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/clouddriver/cm.yaml 
configmap/clouddriver created
configmap/kubeconfig created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/clouddriver/deployment.yaml 
deployment.extensions/clouddriver created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/clouddriver/svc.yaml 
service/clouddriver created
```

# 部署Front50
运维主机`HDSS7-200.host.com`上：
## 准备docker镜像
[镜像下载地址](https://quay.io/repository/container-image/spinnaker-front50)
```
[root@hdss7-200 ~]# docker pull quay.io/container-image/spinnaker-front50:0.15.0-20190123154713
0.15.0-20190123154713: Pulling from container-image/spinnaker-front50

cd784148e348: Already exists 
1a5149a464dd: Pull complete 
26bf3384364c: Pull complete 
69dd8d165987: Pull complete 
32fe51b2b4d9: Pull complete 
Digest: sha256:c8ef16b8af2d19d600cca5e8cc23804bd2c1d5514388773a10b6066f94618a72
Status: Downloaded newer image for quay.io/container-image/spinnaker-front50:0.15.0-20190123154713
[root@hdss7-200 ~]# docker tag dc9a643cca87 harbor.od.com/spinnaker/front50:v0.15.0
[root@hdss7-200 ~]# docker push harbor.od.com/spinnaker/front50:v0.15.0
The push refers to a repository [harbor.od.com/spinnaker/front50]
eabce1ea6140: Pushed 
90b2472b7159: Pushed 
891f51ecb4e3: Pushed 
382d47ad6dc1: Pushed 
dbc783c89851: Mounted from spinnaker/clouddriver 
7bff100f35cb: Mounted from spinnaker/clouddriver 
v0.15.0: digest: sha256:4567e45433305ebed2f66f079db3d17e62675740e9a19006fa4d33c0af492720 size: 1579
```

## 准备资源配置清单
{% tabs front50 %}
<!-- tab ConfigMap -->
vi /data/k8s-yaml/front50/cm.yaml
{% code %}
kind: ConfigMap
apiVersion: v1
metadata:
  name: front50
  namespace: spinnaker
data:
  credentials: |
    [default]
    aws_access_key_id=admin
    aws_secret_access_key=poiuytrewq
  front50.yml: |
    server:
      port: 8080

    management:
      health:
        redis:
          enabled: false

    cassandra:
      enabled: false

    spinnaker:
      cassandra:
        enabled: false
      redis:
        enabled: false
      s3:
        endpoint: http://minio
        enabled: true
        bucket: spinnaker
      rootFolder: front50
    swagger:
      enabled: true
      title: Spinnaker Front50 API
      description:
      contact:
      patterns:
        - /default/.*
        - /credentials.*
        - /global/.*
        - /notifications.*
        - /pipelines.*
        - /strategies.*
        - /v2/.*
{% endcode %}
<!-- endtab -->
<!-- tab Deployment -->
vi /data/k8s-yaml/front50/deployment.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: front50
  name: front50
  namespace: spinnaker
spec:
  replicas: 1
  revisionHistoryLimit: 7
  selector:
    matchLabels:
      app: front50
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: front50
    spec:
      containers:
      - image: harbor.od.com/spinnaker/front50:v0.15.0
        imagePullPolicy: IfNotPresent
        name: front50
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        securityContext: 
          runAsUser: 0
        volumeMounts:
        - mountPath: /opt/front50/config
          name: front
        - mountPath: /root/.aws
          name: aws
      imagePullSecrets:
      - name: harbor
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      volumes:
      - configMap:
          defaultMode: 420
          items:
          - key: front50.yml
            path: front50.yml
          name: front50
        name: front
      - configMap:
          defaultMode: 420
          items:
          - key: credentials
            path: credentials
          name: front50
        name: aws
{% endcode %}
<!-- endtab -->
<!-- tab Service -->
vi /data/k8s-yaml/front50/svc.yaml
{% code %}
apiVersion: v1
kind: Service
metadata:
  name: front50
  namespace: spinnaker
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: front50
  type: ClusterIP
{% endcode %}
<!-- endtab -->
{% endtabs %}

## 应用资源配置清单
任意一台运算节点上：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/front50/cm.yaml 
configmap/front50 created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/front50/deployment.yaml 
deployment.extensions/front50 created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/front50/svc.yaml 
service/front50 created
```

## 浏览器访问
http://minio.od.com
登录并观察存储是否创建

# 部署Orca
运维主机`HDSS7-200.host.com`上：
## 准备docker镜像
[镜像下载地址](https://quay.io/repository/container-image/spinnaker-orca)
```
[root@hdss7-200 ~]# docker pull quay.io/container-image/spinnaker-orca:2.3.0-20190128134206
2.3.0-20190128134206: Pulling from container-image/spinnaker-orca

cd784148e348: Already exists 
35920a071f91: Already exists 
42e1846e4485: Pull complete 
bc6a60341cf7: Pull complete 
19c9805c6a8e: Pull complete 
Digest: sha256:c5994af4100caa9d55ec81aebbd0ca8121520ca48c29daa43757ae3750db2f48
Status: Downloaded newer image for quay.io/container-image/spinnaker-orca:2.3.0-20190128134206
[root@hdss7-200 ~]# docker tag 619c3af0b837 harbor.od.com/spinnaker/orca:v2.3.0
[root@hdss7-200 ~]# docker push harbor.od.com/spinnaker/orca:v2.3.0
The push refers to a repository [harbor.od.com/spinnaker/orca]
3af99de9eeaf: Pushed 
70d500537a7c: Pushed 
39aba6e22a54: Pushed 
382d47ad6dc1: Mounted from spinnaker/front50 
dbc783c89851: Mounted from spinnaker/front50 
7bff100f35cb: Mounted from spinnaker/front50 
v2.3.0: digest: sha256:5d4109a424dd79048ea7b58a6dd42b5a5678b9a6a7299eb1783794ba597a8e83 size: 1578
```

## 准备资源配置清单
{% tabs orca %}
<!-- tab ConfigMap -->
vi /data/k8s-yaml/orca/cm.yaml
{% code %}
kind: ConfigMap
apiVersion: v1
metadata:
  name: orca
  namespace: spinnaker
data:
  orca.yml: |
    server:
      port: 8083

    default:
      bake:
        account: default

    front50:
      enabled: true
      baseUrl: http://front50
    tide:
      enabled: false
    oort:
      baseUrl: http://clouddriver
    mort:
      baseUrl: http://clouddriver
    kato:
      baseUrl: http://clouddriver
    echo:
      enabled: true
      baseUrl: http://echo
    igor:
      enabled: true
      baseUrl: http://igor
    bakery:
      enabled: false
      # Rosco exposes more endpoints than Netflix's internal bakery. This disables
      # the use of those endpoints.
      roscoApisEnabled: true
      allowMissingPackageInstallation: false

    redis:
      connection: redis://redis:6379

    keiko:
      queue:
        redis:
          queueName: orca.task.queue
          deadLetterQueueName: orca.task.deadLetterQueue

    tasks:
      useWaitForAllNetflixAWSInstancesDownTask: false

    logging:
      config: classpath:logback-defaults.xml
{% endcode %}
<!-- endtab -->
<!-- tab Deployment -->
vi /data/k8s-yaml/orca/deployment.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: orca
  name: orca
  namespace: spinnaker
spec:
  replicas: 1
  revisionHistoryLimit: 7
  selector:
    matchLabels:
      app: orca
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: orca
    spec:
      containers:
      - image: harbor.od.com/spinnaker/orca:v2.3.0
        imagePullPolicy: IfNotPresent
        name: orca
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        securityContext: 
          runAsUser: 0
        volumeMounts:
        - mountPath: /opt/orca/config
          name: orca
      imagePullSecrets:
      - name: harbor
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      volumes:
      - configMap:
          defaultMode: 420
          name: orca
        name: orca
{% endcode %}
<!-- endtab -->
<!-- tab Service -->
vi /data/k8s-yaml/orca/svc.yaml
{% code %}
apiVersion: v1
kind: Service
metadata:
  name: orca
  namespace: spinnaker
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8083
  selector:
    app: orca
  type: ClusterIP
{% endcode %}
<!-- endtab -->
{% endtabs %}

## 应用资源配置清单
任意一台运算节点上：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/orca/cm.yaml 
configmap/orca created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/orca/deployment.yaml 
deployment.extensions/orca created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/orca/svc.yaml 
service/orca created
```

# 部署Echo
运维主机`HDSS7-200.host.com`上：
## 准备docker镜像
[镜像下载地址](https://quay.io/repository/container-image/spinnaker-echo)
```
[root@hdss7-200 ~]# docker pull quay.io/container-image/spinnaker-echo:2.3.0-20190123200115
2.3.0-20190123200115: Pulling from container-image/spinnaker-echo
ab1fc7e4bf91: Pull complete 
35fba333ff52: Pull complete 
f0cb1fa13079: Pull complete 
3d1dd648b5ad: Pull complete 
a9f886e483d6: Pull complete 
4346341d3c49: Pull complete 
006f2208d67a: Pull complete 
fb85cf26717d: Pull complete 
845bb5ae3b56: Pull complete 
5d242638cdfc: Pull complete 
90f2c113bf33: Pull complete 
Digest: sha256:2829f5d27e80177aed357d0159daf308fe380c3fad883c4531d580b37617ad43
Status: Downloaded newer image for quay.io/container-image/spinnaker-echo:2.3.0-20190123200115
[root@hdss7-200 ~]# docker tag e56354dc5c3b harbor.od.com/spinnaker/echo:v2.3.0
[root@hdss7-200 ~]# docker push harbor.od.com/spinnaker/echo:v2.3.0
The push refers to a repository [harbor.od.com/spinnaker/echo]
8752d9b1121d: Pushed 
6c6ced81271d: Pushed 
eb72703b9af7: Pushed 
5144cdc9e024: Pushed 
31c8272db455: Pushed 
8e1025180705: Pushed 
0715026f2ff1: Pushed 
a89464ad2a8f: Pushed 
76dfa41f0a1d: Pushed 
c240c542ed55: Pushed 
badfbcebf7f8: Pushed 
v2.3.0: digest: sha256:fae7162a35d18b12692ad36c5dfa3e53691e8bca260d47c179c7c562b886441a size: 2634
```

## 准备资源配置清单
{% tabs echo %}
<!-- tab ConfigMap -->
vi /data/k8s-yaml/echo/cm.yaml
{% code %}
kind: ConfigMap
apiVersion: v1
metadata:
  name: echo
  namespace: spinnaker
data:
  echo.yml: |
    server:
      port: 8089

    cassandra:
      enabled: false

    spinnaker:
      baseUrl: http://deck
      cassandra:
        enabled: false
      inMemory:
        enabled: true

    front50:
      baseUrl: http://front50

    orca:
      baseUrl: http://orca

    endpoints.health.sensitive: false

    slack:
      enabled: false

    spring:
      mail:
        host: hdss7-200.host.com

    mail:
      enabled: true
      host: hdss7-200.host.com
      from: ws2319@163.com

    hipchat:
      enabled: false

    twilio:
      enabled: false

    scheduler:
      enabled: true
      threadPoolSize: 20
      triggeringEnabled: true
      pipelineConfigsPoller:
        enabled: true
        pollingIntervalMs: 30000
      cron:
        timezone: Asia/Shanghai

    redis:
      connection: redis://redis:6379

    webhooks:
      artifacts:
        enabled: true
{% endcode %}
<!-- endtab -->
<!-- tab Deployment -->
vi /data/k8s-yaml/echo/deployment.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: echo
  name: echo
  namespace: spinnaker
spec:
  replicas: 1
  revisionHistoryLimit: 7
  selector:
    matchLabels:
      app: echo
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - image: harbor.od.com/spinnaker/echo:v2.3.0
        imagePullPolicy: IfNotPresent
        name: echo
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        securityContext: 
          runAsUser: 0
        volumeMounts:
        - mountPath: /opt/echo/config
          name: echo
      imagePullSecrets:
      - name: harbor
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      volumes:
      - configMap:
          defaultMode: 420
          name: echo
        name: echo
{% endcode %}
<!-- endtab -->
<!-- tab Service -->
vi /data/k8s-yaml/echo/svc.yaml
{% code %}
apiVersion: v1
kind: Service
metadata:
  name: echo
  namespace: spinnaker
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8089
  selector:
    app: echo
  type: ClusterIP
{% endcode %}
<!-- endtab -->
{% endtabs %}

## 应用资源配置清单
任意一台运算节点上：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/echo/cm.yaml 
configmap/echo created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/echo/deployment.yaml 
deployment.extensions/echo created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/echo/svc.yaml 
service/echo created
```

# 部署Igor
运维主机`HDSS7-200.host.com`上：
## 准备docker镜像
[镜像下载地址](https://quay.io/repository/container-image/spinnaker-igor)
```
[root@hdss7-200 ~]# docker pull quay.io/container-image/spinnaker-igor:1.1.0-20190123154713
1.1.0-20190123154713: Pulling from container-image/spinnaker-igor

cd784148e348: Already exists 
35920a071f91: Already exists 
3a14a57f2792: Pull complete 
3f7df0dd19ae: Pull complete 
e340dbc6f4bf: Pull complete 
Digest: sha256:e653d7085be245bf6dded93fe41bf2c757a8413ba03ccae6959d667a78e5061b
Status: Downloaded newer image for quay.io/container-image/spinnaker-igor:1.1.0-20190123154713
[root@hdss7-200 ~]# docker tag 719a43aa8718 harbor.od.com/spinnaker/igor:v1.1.0
[root@hdss7-200 ~]# docker push harbor.od.com/spinnaker/igor:v1.1.0
The push refers to a repository [harbor.od.com/spinnaker/igor]
5355896bc30a: Pushed 
7024e25c1a5a: Pushed 
faaa6bf89d9e: Pushed 
382d47ad6dc1: Mounted from spinnaker/orca 
dbc783c89851: Mounted from spinnaker/orca 
7bff100f35cb: Mounted from spinnaker/orca 
v1.1.0: digest: sha256:997843cf6f26c6f3e647cca882b0393297dba41dc2a70b3f86acdaee337c5c33 size: 1578
```

## 准备资源配置清单
{% tabs igor %}
<!-- tab ConfigMap -->
vi /data/k8s-yaml/igor/cm.yaml
{% code %}
kind: ConfigMap
apiVersion: v1
metadata:
  name: igor
  namespace: spinnaker
data:
  igor.yml: |
    server:
      port: 8088

    redis:
      connection: redis://redis:6379
      
    jenkins:
      enabled: true
      masters:
        -
          address: http://jenkins.infra
          name: jenkins
          password: admin123
          username: admin
{% endcode %}
<!-- endtab -->
<!-- tab Deployment -->
vi /data/k8s-yaml/igor/deployment.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: igor
  name: igor
  namespace: spinnaker
spec:
  replicas: 1
  revisionHistoryLimit: 7
  selector:
    matchLabels:
      app: igor
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: igor
    spec:
      containers:
      - image: harbor.od.com/spinnaker/igor:v1.1.0
        imagePullPolicy: IfNotPresent
        name: igor
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        securityContext: 
          runAsUser: 0
        volumeMounts:
        - mountPath: /opt/igor/config
          name: igor
      imagePullSecrets:
      - name: harbor
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      volumes:
      - configMap:
          defaultMode: 420
          name: igor
        name: igor
{% endcode %}
<!-- endtab -->
<!-- tab Service -->
vi /data/k8s-yaml/igor/svc.yaml
{% code %}
apiVersion: v1
kind: Service
metadata:
  name: igor
  namespace: spinnaker
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8088
  selector:
    app: igor
  type: ClusterIP
{% endcode %}
<!-- endtab -->
{% endtabs %}

## 应用资源配置清单
任意一台运算节点上：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/igor/cm.yaml 
configmap/igor created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/igor/deployment.yaml 
deployment.extensions/igor created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/igor/svc.yaml 
service/igor created
```

# 部署Gate
运维主机`HDSS7-200.host.com`上：
## 准备docker镜像
[镜像下载地址](https://quay.io/repository/container-image/spinnaker-gate)
```
[root@hdss7-200 ~]# docker pull quay.io/container-image/spinnaker-gate:1.5.0-20190123154713
1.5.0-20190123154713: Pulling from container-image/spinnaker-gate

cd784148e348: Already exists 
35920a071f91: Already exists 
5abffd2fef8e: Pull complete 
eaebfee37cc8: Pull complete 
d29593b154a0: Pull complete 
Digest: sha256:7d46cebaf06c5496cec03fddab43646a9a4b1d42ecd8e32fc8b2ca1ed1e2fb2f
Status: Downloaded newer image for quay.io/container-image/spinnaker-gate:1.5.0-20190123154713
[root@hdss7-200 ~]# docker tag 15edc59823a3 harbor.od.com/spinnaker/gate:v1.5.0
[root@hdss7-200 ~]# docker push harbor.od.com/spinnaker/gate:v1.5.0
The push refers to a repository [harbor.od.com/spinnaker/gate]
0f729725e45a: Pushed 
cfbf10ed46bf: Pushed 
6c65665dab4c: Pushed 
382d47ad6dc1: Mounted from spinnaker/igor 
dbc783c89851: Mounted from spinnaker/igor 
7bff100f35cb: Mounted from spinnaker/igor 
v1.5.0: digest: sha256:0ed100d855506a1df7fa97bd8cb0ae77266cca556d0a8f5f877427089eb6be23 size: 1578
```

## 准备资源配置清单
{% tabs gate %}
<!-- tab ConfigMap -->
vi /data/k8s-yaml/gate/cm.yaml
{% code %}
kind: ConfigMap
apiVersion: v1
metadata:
  name: gate
  namespace: spinnaker
data:
  gate.yml: |
    server:
      port: 8084

    services:
      deck:
        baseUrl: http://deck
      orca:
        baseUrl: http://orca
      front50:
        canaryConfigStore: false
        baseUrl: http://front50
      clouddriver:
        baseUrl: http://clouddriver
      rosco:
        enabled: false
    # optional services:
      echo:
        enabled: false
      fiat:
        enabled: false
      flapjack:
        enabled: false
      igor:
        enabled: false
      mahe:
        enabled: false
      kayenta:
        enabled: false
        canaryConfigStore: false

    redis:
      connection: redis://redis:6379
    ldap:
      enabled: false
    swagger:
      enabled: true
      title: Spinnaker API
      description:
      contact:
      patterns:
        - .*tasks.*
        - .*auth.*
        - .*applications.*
        - .*securityGroups.*
        - /search
        - .*pipelines.*
        - .*loadBalancers.*
        - .*instances.*
        - .*images.*
        - .*elasticIps.*
        - .*credentials.*
        - .*events.*
        - .*builds.*
        - .*instanceTypes.*
        - .*vpcs.*
        - .*subnets.*
        - .*networks.*
        - .*bakery.*
        - .*executions.*
        - .*webhooks.*
        - /v2/.*
    hystrix:
      command:
    ## Hystrix Global Defaults
        default:
          execution.isolation.thread.timeoutInMilliseconds: 60000
    ## Command-specific overrides
        fetchGlobalAccounts:
          execution.isolation.thread.timeoutInMilliseconds: 2000
    spring:
      jackson:
        mapper:
          SORT_PROPERTIES_ALPHABETICALLY: true
        serialization:
          ORDER_MAP_ENTRIES_BY_KEYS: true
    spring.session.store-type: redis
{% endcode %}
<!-- endtab -->
<!-- tab Deployment -->
vi /data/k8s-yaml/gate/deployment.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: gate
  name: gate
  namespace: spinnaker
spec:
  replicas: 1
  revisionHistoryLimit: 7
  selector:
    matchLabels:
      app: gate
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: gate
    spec:
      containers:
      - image: harbor.od.com/spinnaker/gate:v1.5.0
        imagePullPolicy: IfNotPresent
        name: gate
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        securityContext: 
          runAsUser: 0
        volumeMounts:
        - mountPath: /opt/gate/config
          name: gate
      imagePullSecrets:
      - name: harbor
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      volumes:
      - configMap:
          defaultMode: 420
          name: gate
        name: gate
{% endcode %}
<!-- endtab -->
<!-- tab Service -->
vi /data/k8s-yaml/gate/svc.yaml
{% code %}
apiVersion: v1
kind: Service
metadata:
  name: gate
  namespace: spinnaker
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8084
  selector:
    app: gate
  type: ClusterIP
{% endcode %}
<!-- endtab -->
<!-- tab Ingress -->
vi /data/k8s-yaml/gate/ingress.yaml
{% code %}
kind: Ingress
apiVersion: extensions/v1beta1
metadata: 
  name: deck
  namespace: spinnaker
spec:
  rules:
  - host: gate.od.com
    http:
      paths:
      - path: /
        backend: 
          serviceName: gate
          servicePort: 80
{% endcode %}
<!-- endtab -->
{% endtabs %}

## 应用资源配置清单
任意一台运算节点上：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/gate/cm.yaml 
configmap/gate created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/gate/deployment.yaml 
deployment.extensions/gate created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/gate/svc.yaml 
service/gate created
```

#  部署Deck
运维主机`HDSS7-200.host.com`上：
## 准备docker镜像
[镜像下载地址](https://quay.io/repository/container-image/spinnaker-deck)
```
[root@hdss7-200 ~]# docker pull quay.io/container-image/spinnaker-deck:2.7.0-20190123200115
2.7.0-20190123200115: Pulling from container-image/spinnaker-deck
a4aa50fa9594: Pull complete 
209b18efd318: Pull complete 
26cdadfbf7fc: Pull complete 
9ce01d906dbd: Pull complete 
9c615f32c6da: Pull complete 
Digest: sha256:36e82f2fa95572be5bb6d7a6cb43bde6c0e120c369506d752ebdd5ae378e21f8
Status: Downloaded newer image for quay.io/container-image/spinnaker-deck:2.7.0-20190123200115
[root@hdss7-200 ~]# docker tag 13cb501943e7 harbor.od.com/spinnaker/deck:v2.7.0
[root@hdss7-200 ~]# docker push harbor.od.com/spinnaker/gate:v1.5.0
The push refers to a repository [harbor.od.com/spinnaker/deck]
37c745de781e: Pushed 
6472be60f4ca: Pushed 
904c7ac769d2: Pushed 
b50cd40f4a93: Pushed 
9493909cb400: Pushed 
v2.7.0: digest: sha256:bfc65b79405f405d332999bcfb979917e1b6b6824d178efe3ba6cb9f65ba718f size: 1373
```

## 准备资源配置清单
{% tabs deck %}
<!-- tab ConfigMap1 -->
vi /root/setttings.js
{% code %}
'use strict';

var apiHost = process.env.API_HOST || 'http://gate';
var artifactsEnabled = process.env.ARTIFACTS_ENABLED === 'true';
var artifactsRewriteEnabled = process.env.ARTIFACTS_REWRITE_ENABLED === 'true';
var atlasWebComponentsUrl = process.env.ATLAS_WEB_COMPONENTS_URL;
var authEndpoint = process.env.AUTH_ENDPOINT || apiHost + '/auth/user';
var authEnabled = process.env.AUTH_ENABLED === 'false' ? false : true;
var bakeryDetailUrl =
  process.env.BAKERY_DETAIL_URL || apiHost + '/bakery/logs/{{context.region}}/{{context.status.resourceId}}';
var canaryAccount = process.env.CANARY_ACCOUNT || '';
var canaryEnabled = process.env.CANARY_ENABLED === 'false';
var canaryFeatureDisabled = process.env.CANARY_FEATURE_ENABLED !== 'true';
var canaryStagesEnabled = process.env.CANARY_STAGES_ENABLED === 'false';
var chaosEnabled = process.env.CHAOS_ENABLED === 'true' ? true : false;
var debugEnabled = process.env.DEBUG_ENABLED === 'false' ? false : true;
var defaultMetricStore = process.env.METRIC_STORE || 'atlas';
var displayTimestampsInUserLocalTime = process.env.DISPLAY_TIMESTAMPS_IN_USER_LOCAL_TIME === 'true';
var dryRunEnabled = process.env.DRYRUN_ENABLED === 'true' ? true : false;
var entityTagsEnabled = process.env.ENTITY_TAGS_ENABLED === 'true' ? true : false;
var fiatEnabled = process.env.FIAT_ENABLED === 'true' ? true : false;
var gremlinEnabled = process.env.GREMLIN_ENABLED === 'false' ? false : true;
var iapRefresherEnabled = process.env.IAP_REFRESHER_ENABLED === 'true' ? true : false;
var infrastructureEnabled = process.env.INFRA_ENABLED === 'true' ? true : false;
var managedPipelineTemplatesV2UIEnabled = process.env.MANAGED_PIPELINE_TEMPLATES_V2_UI_ENABLED === 'true';
var managedServiceAccountsEnabled = process.env.MANAGED_SERVICE_ACCOUNTS_ENABLED === 'true';
var onDemandClusterThreshold = process.env.ON_DEMAND_CLUSTER_THRESHOLD || '350';
var reduxLoggerEnabled = process.env.REDUX_LOGGER === 'true';
var templatesEnabled = process.env.TEMPLATES_ENABLED === 'true';
var useClassicFirewallLabels = process.env.USE_CLASSIC_FIREWALL_LABELS === 'true';

window.spinnakerSettings = {
  authEnabled: authEnabled,
  authEndpoint: authEndpoint,
  authTtl: 600000,
  bakeryDetailUrl: bakeryDetailUrl,
  canary: {
    atlasWebComponentsUrl: atlasWebComponentsUrl,
    defaultJudge: 'NetflixACAJudge-v1.0',
    featureDisabled: canaryFeatureDisabled,
    metricsAccountName: canaryAccount,
    metricStore: defaultMetricStore,
    reduxLogger: reduxLoggerEnabled,
    showAllConfigs: true,
    stagesEnabled: canaryStagesEnabled,
    storageAccountName: canaryAccount,
    templatesEnabled: templatesEnabled,
  },
  checkForUpdates: true,
  debugEnabled: debugEnabled,
  defaultCategory: 'Kubernetes',
  defaultInstancePort: 80,
  defaultProviders: ['kubernetes'],
  defaultTimeZone: process.env.TIMEZONE || 'Asia/Shanghai', // see http://momentjs.com/timezone/docs/#/data-utilities/
  feature: {
    artifacts: artifactsEnabled,
    artifactsRewrite: artifactsRewriteEnabled,
    canary: "false",
    chaosMonkey: "false",
    displayTimestampsInUserLocalTime: displayTimestampsInUserLocalTime,
    dryRunEnabled: dryRunEnabled,
    entityTags: entityTagsEnabled,
    fiatEnabled: "false",
    gremlinEnabled: "false",
    iapRefresherEnabled: iapRefresherEnabled,
    // whether stages affecting infrastructure (like "Create Load Balancer") should be enabled or not
    infrastructureStages: infrastructureEnabled,
    jobs: false,
    managedPipelineTemplatesV2UI: managedPipelineTemplatesV2UIEnabled,
    managedServiceAccounts: managedServiceAccountsEnabled,
    notifications: false,
    pagerDuty: false,
    pipelineTemplates: false,
    pipelines: true,
    quietPeriod: false,
    roscoMode: false,
    snapshots: false,
    travis: false,
    versionedProviders: true,
    wercker: false,
  },
  gateUrl: apiHost,
  gitSources: ['stash', 'github', 'bitbucket', 'gitlab'],
  maxPipelineAgeDays: 14,
  newApplicationDefaults: {
    chaosMonkey: false,
  },
  notifications: {
    bearychat: {
      enabled: false,
    },
    email: {
      enabled: true,
    },
    githubStatus: {
      enabled: true,
    },
    googlechat: {
      enabled: false,
    },
    pubsub: {
      enabled: false,
    },
    slack: {
      botName: 'spinnakerbot',
      enabled: true,
    },
    sms: {
      enabled: false,
    },
  },
  onDemandClusterThreshold: Number(onDemandClusterThreshold),
  pollSchedule: 30000,
  providers: {
    appengine: {
      defaults: {
        account: 'my-appengine-account',
      },
    },
    kubernetes: {
      defaults: {
        account: 'cluster-admin',
        apiPrefix: 'api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard/#',
        instanceLinkTemplate: '{{host}}/api/v1/proxy/namespaces/{{namespace}}/pods/{{name}}',
        internalDNSNameTemplate: '{{name}}.{{namespace}}.svc.cluster.local',
        namespace: 'spinnaker',
        proxy: '10.9.6.200:6443',
      },
    },
  },
  pagerDuty: {
    required: false,
  },
  pubsubProviders: ['google'], // TODO(joonlim): Add amazon once it is confirmed that amazon pub/sub works.
  searchVersion: 1,
  triggerTypes: [
    'artifactory',
    'concourse',
    'cron',
    'docker',
    'git',
    'jenkins',
    'pipeline',
    'pubsub',
    'travis',
    'webhook',
    'wercker',
  ],
  useClassicFirewallLabels: useClassicFirewallLabels,
  whatsNew: {
    fileName: 'news.md',
    gistId: '32526cd608db3d811b38',
  },
};
{% endcode %}
<!-- endtab -->
<!-- tab ConfigMap2 -->
vi /root/spinnaker.conf.gen
{% code %}
<VirtualHost 0.0.0.0:9000>
  DocumentRoot /opt/deck/html
  ServerName spinnaker.od.com

  ProxyPass "/gate" "http://gate" retry=0
  ProxyPassReverse "/gate" "http://gate"
  ProxyPreserveHost On

  <Directory "/opt/deck/html/">
    Require all granted
  </Directory>
</VirtualHost>
{% endcode %}
<!-- endtab -->
<!-- tab Deployment -->
vi /data/k8s-yaml/deck/deployment.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: deck
  name: deck
  namespace: spinnaker
spec:
  replicas: 1
  revisionHistoryLimit: 7
  selector:
    matchLabels:
      app: deck
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: deck
    spec:
      containers:
      - image: harbor.od.com/spinnaker/deck:v2.7.0
        imagePullPolicy: IfNotPresent
        name: deck
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        securityContext: 
          runAsUser: 0
        volumeMounts:
        - mountPath: /opt/spinnaker/config
          name: deck
      imagePullSecrets:
      - name: harbor
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      volumes:
      - configMap:
          defaultMode: 420
          name: deck
        name: deck
{% endcode %}
<!-- endtab -->
<!-- tab Service -->
vi /data/k8s-yaml/deck/svc.yaml
{% code %}
apiVersion: v1
kind: Service
metadata:
  name: deck
  namespace: spinnaker
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 9000
  selector:
    app: deck
  type: ClusterIP
{% endcode %}
<!-- endtab -->
<!-- tab Ingress -->
vi /data/k8s-yaml/deck/ingress.yaml
{% code %}
kind: Ingress
apiVersion: extensions/v1beta1
metadata: 
  name: deck
  namespace: spinnaker
spec:
  rules:
  - host: spinnaker.od.com
    http:
      paths:
      - path: /
        backend: 
          serviceName: deck
          servicePort: 80
{% endcode %}
<!-- endtab -->
{% endtabs %}

## 应用资源配置清单
任意一台运算节点上：
```
[root@hdss7-21 ~]# kubectl create cm deck --from-file=settings.js --from-file=spinnaker.conf.gen -n spinnaker
configmap/deck created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/deck/deployment.yaml 
deployment.extensions/deck created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/deck/svc.yaml 
service/deck created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/deck/ingress.yaml 
ingress/deck created
```
