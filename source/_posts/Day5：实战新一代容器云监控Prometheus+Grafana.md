title: Day5：实战新一代容器云监控Prometheus+Grafana
author: Stanley Wang
date: 2019-6-18 18:12:56
---
- - -
{% cq %}欢迎加入王导的VIP学习qq群：==>[<font color="FF7F50">932194668</font>](http://shang.qq.com/wpa/qunwpa?idkey=78869fddc5a661acb0639315eb52997c108de6625df5f0ee2f0372f176a032a6)<=={% endcq %}
- - -
# 部署kube-state-metrics
运维主机`HDSS7-200.host.com`
## 准备kube-state-metrics镜像
[kube-state-metrics官方quay.io地址](https://quay.io/repository/coreos/kube-state-metrics?tab=info)
```
[root@hdss7-200 ~]# docker pull quay.io/coreos/kube-state-metrics:v1.5.0
v1.5.0: Pulling from coreos/kube-state-metrics
cd784148e348: Pull complete 
f622528a393e: Pull complete 
Digest: sha256:b7a3143bd1eb7130759c9259073b9f239d0eeda09f5210f1cd31f1a530599ea1
Status: Downloaded newer image for quay.io/coreos/kube-state-metrics:v1.5.0
[root@hdss7-200 ~]# docker tag 91599517197a harbor.od.com/public/kube-state-metrics:v1.5.0
[root@hdss7-200 ~]# docker push harbor.od.com/public/kube-state-metrics:v1.5.0
docker push harbor.od.com/public/kube-state-metrics:v1.5.0
The push refers to a repository [harbor.od.com/public/kube-state-metrics]
5b3c36501a0a: Pushed 
7bff100f35cb: Pushed 
v1.5.0: digest: sha256:0d9bea882f25586543254d9ceeb54703eb6f8f94c0dd88875df216b6a0f60589 size: 739
```

## 准备资源配置清单
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
\-\--
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
      - image: harbor.od.com/public/kube-state-metrics:v1.5.0
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
      restartPolicy: Always
      serviceAccount: kube-state-metrics
      serviceAccountName: kube-state-metrics
{% endcode %}
<!-- endtab -->
{% endtabs %}

## 应用资源配置清单
任意一台运算节点上：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/kube-state-metrics/rbac.yaml 
serviceaccount/kube-state-metrics created
clusterrole.rbac.authorization.k8s.io/kube-state-metrics created
clusterrolebinding.rbac.authorization.k8s.io/kube-state-metrics created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/kube-state-metrics/deployment.yaml 
deployment.extensions/kube-state-metrics created
```

## 检查启动情况
```
[root@hdss7-21 ~]# kubectl get pods -n kube-system|grep kube-state-metrics
kube-state-metrics-645bd94c55-wdtqh      1/1     Running   0          94s
```

# 部署node-exporter
运维主机`HDSS7-200.host.com`上：
## 准备node-exporter镜像
[node-exporter官方dockerhub地址](https://hub.docker.com/r/prom/node-exporter)
[node-expoerer官方github地址](https://github.com/prometheus/node_exporter)
```
[root@hdss7-200 ~]# docker pull prom/node-exporter:v0.15.0
v0.15.0: Pulling from prom/node-exporter
0de338cf4258: Pull complete 
f508012419d8: Pull complete 
d764f7880123: Pull complete 
Digest: sha256:c390c8fea4cd362a28ad5070aedd6515aacdfdffd21de6db42ead05e332be5a9
Status: Downloaded newer image for prom/node-exporter:v0.15.0
[root@hdss7-200 ~]# docker tag b3e7f67a1480 harbor.od.com/public/node-exporter:v0.15.0
[root@hdss7-200 ~]# docker push harbor.od.com/public/node-exporter:v0.15.0
docker push harbor.od.com/public/node-exporter:v0.15.0
The push refers to a repository [harbor.od.com/public/node-exporter]
0bf893ee7433: Pushed 
17ab2671f87e: Pushed 
b8873621dfbc: Pushed 
v0.15.0: digest: sha256:4e13dd75f00a6114675ea3e62e61dbd79dcb2205e8f3bbe1f8f8ef2fd3e28113 size: 949
```

## 准备资源配置清单
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
        image: harbor.od.com/public/node-exporter:v0.15.0
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
      hostNetwork: true
```

## 应用资源配置清单
任意运算节点上：
```
[root@hdss7-21 ~]# kubectl -f http://k8s-yaml.od.com/node-exporter/node-exporter-ds.yaml
daemonset.extensions/node-exporter created
```

# 部署cadvisor
运维主机`HDSS7-200.host.com`上：
## 准备cadvisor镜像
[cadvisor官方dockerhub地址](https://hub.docker.com/r/google/cadvisor)
[cadvisor官方github地址](https://github.com/google/cadvisor)
```
[root@hdss7-200 ~]# docker pull google/cadvisor:v0.28.3
v0.28.3: Pulling from google/cadvisor
49388a8c9c86: Pull complete 
21c698e54ae5: Pull complete 
fafc7cbc1edf: Pull complete 
Digest: sha256:09c8d73c9b799d30777763b7029cfd8624b8a1bd33af652ec3b51a6b827d492a
Status: Downloaded newer image for google/cadvisor:v0.28.3
[root@hdss7-200 ~]# docker tag d60fd8aeb74c harbor.od.com/public/cadvisor:v0.28.3
[root@hdss7-200 ~]# docker push !$
docker push harbor.od.com/public/cadvisor:v0.28.3
The push refers to a repository [harbor.od.com/public/cadvisor]
daab541dbbf0: Pushed 
ba67c95cca3d: Pushed 
ef763da74d91: Pushed 
v0.28.3: digest: sha256:319812db86e7677767bf6a51ea63b5bdb17dc18ffa576eb6c8a667e899d1329d size: 951
```

## 准备资源配置清单
{% tabs cadvisor%}
<!-- tab DaemonSet -->
vi /data/k8s-yaml/cadvisor/daemonset.yaml
{% code %}
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: cadvisor
  namespace: kube-system
  labels:
    app: cadvisor
spec:
  selector:
    matchLabels:
      name: cadvisor
  template:
    metadata:
      labels:
        name: cadvisor
    spec:
      hostNetwork: true
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
        key: enabledDiskSchedule
        value: "true"
        effect: NoSchedule
      containers:
      - name: cadvisor
        image: harbor.od.com/public/cadvisor:v0.28.3
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: rootfs
          mountPath: /rootfs
          readOnly: true
        - name: var-run
          mountPath: /var/run
        - name: sys
          mountPath: /sys
          readOnly: true
        - name: docker
          mountPath: /var/lib/docker
          readOnly: true
        ports:
          - name: http
            containerPort: 4194
            protocol: TCP
        readinessProbe:
          tcpSocket:
            port: 4194
          initialDelaySeconds: 5
          periodSeconds: 10
        args:
          - -\-housekeeping_interval=10s
          - -\-port=4194
      imagePullSecrets:
      - name: harbor
      terminationGracePeriodSeconds: 30
      volumes:
      - name: rootfs
        hostPath:
          path: /
      - name: var-run
        hostPath:
          path: /var/run
      - name: sys
        hostPath:
          path: /sys
      - name: docker
        hostPath:
          path: /data/docker
{% endcode %}
<!-- endtab -->
{% endtabs %}

## 修改运算节点软连接
所有运算节点上：
```
[root@hdss7-21 ~]# mount -o remount,rw /sys/fs/cgroup/
[root@hdss7-21 ~]# ln -s /sys/fs/cgroup/cpu,cpuacct/ /sys/fs/cgroup/cpuacct,cpu
[root@hdss7-21 ~]# ll /sys/fs/cgroup/ | grep cpu
total 0
lrwxrwxrwx 1 root root 11 Jan 28 22:41 cpu -> cpu,cpuacct
lrwxrwxrwx 1 root root 11 Jan 28 22:41 cpuacct -> cpu,cpuacct
lrwxrwxrwx 1 root root 27 May  5 11:15 cpuacct,cpu -> /sys/fs/cgroup/cpu,cpuacct/
drwxr-xr-x 8 root root  0 Apr 26 11:06 cpu,cpuacct
drwxr-xr-x 7 root root  0 Jan 28 22:41 cpuset
```

## 应用资源配置清单
任意运算节点上：
```
[root@hdss7-21 ~]# kubectl -f http://k8s-yaml.od.com/cadvisor/deamon.yaml
daemonset.apps/cadvisor created
[root@hdss7-21 ~]# netstat -luntp|grep 4194
tcp6       0      0 :::4194                 :::*                    LISTEN      49153/cadvisor
```

# 部署blackbox-exporter
运维主机`HDSS7-200.host.com`上：
## 准备blackbox-exporter镜像
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
[root@hdss7-200 ~]# docker tag d3a00aea3a01 harbor.od.com/public/blackbox-exporter:v0.14.0
[root@hdss7-200 ~]# docker push harbor.od.com/public/blackbox-exporter:v0.14.0
docker push harbor.od.com/public/blackbox-exporter:v0.14.0
The push refers to a repository [harbor.od.com/public/blackbox-exporter]
256c4aa8ebe5: Pushed 
4b6cc55de649: Pushed 
986894c42222: Mounted from infra/prometheus 
adab5d09ba79: Mounted from infra/prometheus 
v0.14.0: digest: sha256:b127897cf0f67c47496d5cbb7ecb86c001312bddd04192f6319d09292e880a5f size: 1155
```

## 准备资源配置清单
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
        image: harbor.od.com/public/blackbox-exporter:v0.14.0
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

## 解析域名
`HDSS7-11.host.com`上
```vi /var/named/od.com.zone
blackbox	60 IN A 10.4.7.10
```

## 应用资源配置清单
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

## 浏览器访问
http://blackbox.od.com

![blackbox-exporter](/images/blackbox.png "blackbox-exporter")


# 部署Prometheus
运维主机`HDSS7-200.host.com`上：
## 准备prometheus镜像
[prometheus官方dockerhub地址](https://hub.docker.com/r/prom/prometheus)
[prometheus官方github地址](https://github.com/prometheus/prometheus)
```
[root@hdss7-200 ~]# docker pull prom/prometheus:v2.12.0
v2.12.0: Pulling from prom/prometheus
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
Status: Downloaded newer image for prom/prometheus:v2.12.0
[root@hdss7-200 ~]# docker tag 4737a2d79d1a harbor.od.com/infra/prometheus:v2.12.0
[root@hdss7-200 ~]# docker push harbor.od.com/infra/prometheus:v2.12.0
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
v2.12.0: digest: sha256:2357e59541f5596dd90d9f4218deddecd9b4880c1e417a42592b00b30b47b0a9 size: 2198
```

## 准备资源配置清单
运维主机`HDSS7-200.host.com`上：
```pwd /data/k8s-yaml
[root@hdss7-200 k8s-yaml]# mkdir /data/k8s-yaml/prometheus && mkdir -p /data/nfs-volume/prometheus/etc && cd /data/k8s-yaml/prometheus 
```
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
      containers:
      - image: harbor.od.com/infra/prometheus:v2.12.0
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
      - name: data
        nfs:
          server: hdss7-200
          path: /data/nfs-volume/prometheus

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

## 准备prometheus的配置文件
运算节点`HDSS7-21.host.com`上：
- 拷贝证书

```
[root@hdss7-21 ~]# mkdir -pv /data/nfs-volume/prometheus/{etc,prom-db}
mkdir: created directory ‘/data/nfs-volume/prometheus/etc’
mkdir: created directory ‘/data/nfs-volume/prometheus/prom-db’
[root@hdss7-21 ~]# cd /data/nfs-volume/prometheus/etc
[root@hdss7-21 ~]# scp root@10.4.7.200:/opt/certs/ca.pem .
[root@hdss7-21 ~]# scp root@10.4.7.200:/opt/certs/client.pem .
[root@hdss7-21 ~]# scp root@10.4.7.200:/opt/certs/client-key.pem .
```
- 准备配置

```vi /data/nfs-volume/prometheus/etc/prometheus.yml
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
    replacement: blackbox-exporter.kube-system:9115
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
    replacement: blackbox-exporter.kube-system:9115
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

## 应用资源配置清单
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
## 解析域名
`HDSS7-11.host.com`上
```vi /var/named/od.com.zone
prometheus	60 IN A 10.4.7.10
```

## 浏览器访问
http://prometheus.od.com

## Prometheus监控内容
Targets（jobs）
### etcd
> 监控etcd服务

key|value
-|-
etcd_server_has_leader|1
etcd_http_failed_total|1
...|...

### kubernetes-apiserver
> 监控apiserver服务

### kubernetes-kubelet
> 监控kubelet服务

### kubernetes-kube-state
监控基本信息
- node-exporter
> 监控Node节点信息

- kube-state-metrics
> 监控pod信息

### traefik
> 监控traefik-ingress-controller

key|value
-|-
traefik_entrypoint_requests_total{code="200",entrypoint="http",method="PUT",protocol="http"}|138
traefik_entrypoint_requests_total{code="200",entrypoint="http",method="GET",protocol="http"}|285
traefik_entrypoint_open_connections{entrypoint="http",method="PUT",protocol="http"}|1
...|...

**注意：在traefik的pod控制器上加annotations，并重启pod，监控生效**
配置范例：
```
"annotations": {
  "prometheus_io_scheme": "traefik",
  "prometheus_io_path": "/metrics",
  "prometheus_io_port": "8080"
}
```

### blackbox*
监控服务是否存活
- blackbox_tcp_pod_porbe
> 监控tcp协议服务是否存活

key|value
-|-
probe_success|1
probe_ip_protocol|4
probe_failed_due_to_regex|0
probe_duration_seconds|0.000597546
probe_dns_lookup_time_seconds|0.00010898

**注意：在pod控制器上加annotations，并重启pod，监控生效**
配置范例：
```
"annotations": {
  "blackbox_port": "20880",
  "blackbox_scheme": "tcp"
}
```

- blackbox_http_pod_probe
> 监控http协议服务是否存活

key|value
-|-
probe_success|1
probe_ip_protocol|4
probe_http_version|1.1
probe_http_status_code|200
probe_http_ssl|0
probe_http_redirects|1
probe_http_last_modified_timestamp_seconds|1.553861888e+09
probe_http_duration_seconds{phase="transfer"}|0.000238343
probe_http_duration_seconds{phase="tls"}|0
probe_http_duration_seconds{phase="resolve"}|5.4095e-05
probe_http_duration_seconds{phase="processing"}|0.000966104
probe_http_duration_seconds{phase="connect"}|0.000520821
probe_http_content_length|716
probe_failed_due_to_regex|0
probe_duration_seconds|0.00272609
probe_dns_lookup_time_seconds|5.4095e-05

**注意：在pod控制器上加annotations，并重启pod，监控生效**
配置范例：
```
"annotations": {
  "blackbox_path": "/",
  "blackbox_port": "8080",
  "blackbox_scheme": "http"
}
```

### kubernetes-pods*
> 监控JVM信息

key|value
-|-
jvm_info{version="1.7.0_80-b15",vendor="Oracle Corporation",runtime="Java(TM) SE Runtime Environment",}|1.0
jmx_config_reload_success_total|0.0
process_resident_memory_bytes|4.693897216E9
process_virtual_memory_bytes|1.2138840064E10
process_max_fds|65536.0
process_open_fds|123.0
process_start_time_seconds|1.54331073249E9
process_cpu_seconds_total|196465.74
jvm_buffer_pool_used_buffers{pool="mapped",}|0.0
jvm_buffer_pool_used_buffers{pool="direct",}|150.0
jvm_buffer_pool_capacity_bytes{pool="mapped",}|0.0
jvm_buffer_pool_capacity_bytes{pool="direct",}|6216688.0
jvm_buffer_pool_used_bytes{pool="mapped",}|0.0
jvm_buffer_pool_used_bytes{pool="direct",}|6216688.0
jvm_gc_collection_seconds_sum{gc="PS MarkSweep",}|1.867
...|...

**注意：在pod控制器上加annotations，并重启pod，监控生效**
配置范例：
```
"annotations": {
  "prometheus_io_scrape": "true",
  "prometheus_io_port": "12346",
  "prometheus_io_path": "/"
}
```

## 修改traefik服务接入prometheus监控
`dashboard`上：
kube-system名称空间->daemonset->traefik-ingress-controller->spec->template->metadata下，添加
```
"annotations": {
  "prometheus_io_scheme": "traefik",
  "prometheus_io_path": "/metrics",
  "prometheus_io_port": "8080"
}
```
删除pod，重启traefik，观察监控

继续添加blackbox监控配置项
```
"annotations": {
  "prometheus_io_scheme": "traefik",
  "prometheus_io_path": "/metrics",
  "prometheus_io_port": "8080",
  "blackbox_path": "/",
  "blackbox_port": "8080",
  "blackbox_scheme": "http"
}
```
## 修改dubbo-service服务接入prometheus监控
`dashboard`上：
app名称空间->deployment->dubbo-demo-service->spec->template=>metadata下，添加
```
"annotations": {
  "prometheus_io_scrape": "true",
  "prometheus_io_path": "/",
  "prometheus_io_port": "12346",
  "blackbox_port": "20880",
  "blackbox_scheme": "tcp"
}
```
删除pod，重启traefik，观察监控

## 修改dubbo-consumer服务接入prometheus监控
app名称空间->deployment->dubbo-demo-consumer->spec->template->metadata下，添加
```
"annotations": {
  "prometheus_io_scrape": "true",
  "prometheus_io_path": "/",
  "prometheus_io_port": "12346",
  "blackbox_path": "/hello",
  "blackbox_port": "8080",
  "blackbox_scheme": "http"
}
```
删除pod，重启traefik，观察监控

# 部署Grafana
运维主机`HDSS7-200.host.com`上：
## 准备grafana镜像
[grafana官方dockerhub地址](https://hub.docker.com/r/grafana/grafana)
[grafana官方github地址](https://github.com/grafana/grafana)
[grafana官网](https://grafana.com/)
```
[root@hdss7-200 ~]# docker pull grafana/grafana:6.3.4
6.3.4: Pulling from grafana/grafana
27833a3ba0a5: Pull complete 
9412d126b236: Pull complete 
1b7d6aaa6217: Pull complete 
530d1110a8c8: Pull complete 
fdcf73917f64: Pull complete 
f5009e3ea28a: Pull complete 
Digest: sha256:c2100550937e7aa0f3e33c2fc46a8c9668c3b5f2f71a8885e304d35de9fea009
Status: Downloaded newer image for grafana/grafana:6.3.4
[root@hdss7-200 ~]# docker tag d9bdb6044027 harbor.od.com/infra/grafana:v6.3.4
[root@hdss7-200 ~]# docker push harbor.od.com/infra/grafana:v6.3.4
docker push harbor.od.com/infra/grafana:v6.3.4
The push refers to a repository [harbor.od.com/infra/grafana]
b57e9e94fc2d: Pushed 
3d4e16e25cba: Pushed 
9642e67d431a: Pushed 
af52591a894f: Pushed 
0a8c2d04bf65: Pushed 
5dacd731af1b: Pushed 
v6.3.4: digest: sha256:db87ab263f90bdae66be744ac7935f6980c4bbd30c9756308e7382e00d4eeae8 size: 1576
```

## 准备资源配置清单
{% tabs grafana%}
<!-- tab RBAC-->
vi /data/k8s-yaml/grafana/rbac.yaml
{% code %}
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/cluster-service: "true"
  name: grafana
rules:
- apiGroups:
  - "*"
  resources:
  - namespaces
  - deployments
  - pods
  verbs:
  - get
  - list
  - watch
-\--
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/cluster-service: "true"
  name: grafana
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: grafana
subjects:
- kind: User
  name: k8s-node
{% endcode %}
<!-- endtab -->
<!-- tab Deployment -->
vi /data/k8s-yaml/grafana/deployment.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: grafana
    name: grafana
  name: grafana
  namespace: infra
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 7
  selector:
    matchLabels:
      name: grafana
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: grafana
        name: grafana
    spec:
      containers:
      - image: harbor.od.com/infra/grafana:v6.3.4
        imagePullPolicy: IfNotPresent
        name: grafana
        ports:
        - containerPort: 3000
          protocol: TCP
        volumeMounts:
        - mountPath: /var/lib/grafana
          name: data
      imagePullSecrets:
      - name: harbor
      restartPolicy: Always
      securityContext:
        runAsUser: 0
      volumes:
      - nfs:
          server: hdss7-200
          path: /data/nfs-volume/grafana
        name: data
{% endcode %}
<!-- endtab -->
<!-- tab Service-->
vi /data/k8s-yaml/grafana/service.yaml
{% code %}
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: infra
spec:
  ports:
  - port: 3000
    protocol: TCP
  selector:
    app: grafana
  type: ClusterIP
{% endcode %}
<!-- endtab -->
<!-- tab Ingress-->
vi /data/k8s-yaml/grafana/ingress.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: grafana
  namespace: infra
spec:
  rules:
  - host: grafana.od.com
    http:
      paths:
      - path: /
        backend:
          serviceName: grafana
          servicePort: 3000
{% endcode %}
<!-- endtab -->
{% endtabs %}

## 应用资源配置清单
任意运算节点上：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/grafana/deployment.yaml 
deployment.extensions/grafana created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/grafana/service.yaml 
service/grafana created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/grafana/ingress.yaml 
ingress.extensions/grafana created
```

## 解析域名
`HDSS7-11.host.com`上
```vi /var/named/od.com.zone
grafana	60 IN A 10.4.7.10
```

## 浏览器访问
http://grafana.od.com

- 用户名：admin
- 密  码：admin

登录后需要修改管理员密码
![grafana首次登录](/images/grafana-password.png "修改管理员密码")

## 配置grafana页面
### 外观
Configuration -> Preferences
- UI Theme
> Light

- Home Dashboard
> Default

- Timezone
> Local browser time

save

### 插件
Configuration -> Plugins
- Kubernetes App

安装方法一：
```
grafana-cli plugins install grafana-kubernetes-app
```
安装方法二：
[下载地址](https://grafana.com/api/plugins/grafana-kubernetes-app/versions/1.0.1/download)
```pwd /data/nfs-volume/grafana/plugins
[root@hdss7-21 plugins]# wget https://grafana.com/api/plugins/grafana-kubernetes-app/versions/1.0.1/download -O grafana-kubernetes-app.zip
--2019-04-28 16:15:33--  https://grafana.com/api/plugins/grafana-kubernetes-app/versions/1.0.1/download
Resolving grafana.com (grafana.com)... 35.241.23.245
Connecting to grafana.com (grafana.com)|35.241.23.245|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://codeload.github.com/grafana/kubernetes-app/legacy.zip/31da38addc1d0ce5dfb154737c9da56e3b6692fc [following]
--2019-04-28 16:15:37--  https://codeload.github.com/grafana/kubernetes-app/legacy.zip/31da38addc1d0ce5dfb154737c9da56e3b6692fc
Resolving codeload.github.com (codeload.github.com)... 13.229.189.0, 13.250.162.133, 54.251.140.56
Connecting to codeload.github.com (codeload.github.com)|13.229.189.0|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3084524 (2.9M) [application/zip]
Saving to: ‘grafana-kubernetes-app.zip’

100%[===================================================================================================================>] 3,084,524    116KB/s   in 12s    

2019-04-28 16:15:51 (250 KB/s) - ‘grafana-kubernetes-app.zip’ saved [3084524/3084524]
[root@hdss7-200 plugins]# unzip grafana-kubernetes-app.zip
```

- Clock Pannel

安装方法一：
```
grafana-cli plugins install grafana-clock-panel
```

安装方法二：
[下载地址](https://grafana.com/api/plugins/grafana-clock-panel/versions/1.0.2/download)

- Pie Chart

安装方法一：
```
grafana-cli plugins install grafana-piechart-panel
```

安装方法二：
[下载地址](https://grafana.com/api/plugins/grafana-piechart-panel/versions/1.3.6/download)

- D3 Gauge

安装方法一：
```
grafana-cli plugins install briangann-gauge-panel
```

安装方法二：
[下载地址](https://grafana.com/api/plugins/briangann-gauge-panel/versions/0.0.6/download)

- Discrete

安装方法一：
```
grafana-cli plugins install natel-discrete-panel
```

安装方法二：
[下载地址](https://grafana.com/api/plugins/natel-discrete-panel/versions/0.0.9/download)

- 重启grafana的pod

- 依次enable插件

## 配置grafana数据源
Configuration -> Data Sources
选择prometheus
- HTTP

key|value
-|-
URL|http://prometheus.od.com
Access|Server(Default)

- Save & Test

![Grafana数据源](/images/grafana-datasource.png "grafana数据源")


## 配置Kubernetes集群Dashboard
kubernetes -> +New Cluster
- Add a new cluster

key|value
-|-
Name|myk8s

- HTTP

key|value
-|-
URL|https://10.4.7.10:7443
Access|Server(Default)

- Auth

key|value
-|-
TLS Client Auth| 勾选
With Ca Cert|勾选

将ca.pem、client.pem和client-key.pem粘贴至文本框内

- Prometheus Read

key|value
-|-
Datasource|Prometheus

- Save

**注意：**
- K8S Container中，所有Pannel的
> pod_name -> container_label_io_kubernetes_pod_name

{% tabs k8s-dashboard %}
<!-- tab K8S-Cluster -->
![k8s-cluster](/images/grafana-cluster.png "k8s-cluster")
<!-- endtab -->
<!-- tab K8S-Node -->
![k8s-node](/images/grafana-k8snode.png "k8s-node")
<!-- endtab -->
<!-- tab K8S-Deployment -->
![k8s-deployment](/images/grafana-deployment.png "k8s-deployment")
<!-- endtab -->
<!-- tab K8S-Container -->
![k8s-container](/images/grafana-container.png "k8s-container")
<!-- endtab -->
{% endtabs %}

## 配置自定义dashboard
根据Prometheus数据源里的数据，配置如下dashboard：
- generic dashboard
- etcd dashboard
- traefik dashboard
- JMX dashboard
- blackbox dashboard

{% tabs grafana-dashboard %}
<!-- tab Generic -->
![generic](/images/grafana-generic.png "generic")
<!-- endtab -->
<!-- tab Etcd-->
![Etcd](/images/grafana-etcd.png "Etcd")
<!-- endtab -->
<!-- tab Traefik -->
![Traefik](/images/grafana-traefik.png "Traefik")
<!-- endtab -->
<!-- tab JMX -->
![jvm监控](/images/grafana-jmx.png "jvm监控")
<!-- endtab -->
<!-- tab BlackBox -->
![BlackBox](/images/grafana-blackbox.png "BlackBox")
<!-- endtab -->
{% endtabs %}