title: Day2：跟我一步步部署kubernetes集群（下）
author: Stanley Wang
date: 2019-6-18 21:12:56
---
- - -
{% cq %}欢迎加入王导的VIP学习qq群：==>[<font color="FF7F50">932194668</font>](http://shang.qq.com/wpa/qunwpa?idkey=78869fddc5a661acb0639315eb52997c108de6625df5f0ee2f0372f176a032a6)<=={% endcq %}
- - -

# 部署flannel
## 集群规划
主机名|角色|ip
-|-|-
HDSS7-21.host.com|flannel|10.4.7.21
HDSS7-22.host.com|flannel|10.4.7.22

**注意：**这里部署文档以`HDSS7-21.host.com`主机为例，另外一台运算节点安装部署方法类似

## 下载软件，解压，做软连接
[flannel官方下载地址](https://github.com/coreos/flannel/releases)
`HDSS7-21.host.com`上：
```pwd /opt/src
[root@hdss7-21 src]# ls -l|grep flannel
-rw-r--r-- 1 root root 417761204 Jan 17 18:46 flannel-v0.10.0-linux-amd64.tar.gz
[root@hdss7-21 src]# mkdir -p /opt/flannel-v0.10.0-linux-amd64/cert
[root@hdss7-21 src]# tar xf flannel-v0.10.0-linux-amd64.tar.gz -C /opt/flannel-v0.10.0-linux-amd64
[root@hdss7-21 src]# ln -s /opt/flannel-v0.10.0-linux-amd64 /opt/flannel
[root@hdss7-21 src]# ls -l /opt|grep flannel
lrwxrwxrwx 1 root   root         31 Jan 17 18:49 flannel -> flannel-v0.10.0-linux-amd64/
drwxr-xr-x 4 root   root         50 Jan 17 18:47 flannel-v0.10.0-linux-amd64
```

## 最终目录结构
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
    |-- docker
    |-- etcd
    |-- flannel
    `-- kubernetes
```

## 拷贝证书
各运算节点上：
```pwd /opt/flannel/cert
[root@hdss7-21 cert]# scp hdss7-200:/opt/certs/ca.pem .
[root@hdss7-21 cert]# scp hdss7-200:/opt/certs/client.pem .
[root@hdss7-21 cert]# scp hdss7-200:/opt/certs/client-key.pem .
```

## 创建配置
`HDSS7-21.host.com`上：
```vi /opt/flannel/subnet.env
FLANNEL_NETWORK=172.7.0.0/16
FLANNEL_SUBNET=172.7.21.1/24
FLANNEL_MTU=1500
FLANNEL_IPMASQ=false
```
**注意：**flannel集群各主机的配置略有不同，部署其他节点时注意修改。

## 创建启动脚本
`HDSS7-21.host.com`上：
```vi /opt/flannel/flanneld.sh
#!/bin/sh
./flanneld \
  --public-ip=10.4.7.21 \
  --etcd-endpoints=https://10.4.7.12:2379,https://10.4.7.21:2379,https://10.4.7.22:2379 \
  --etcd-keyfile=./cert/client-key.pem \
  --etcd-certfile=./cert/client.pem \
  --etcd-cafile=./cert/ca.pem \
  --iface=eth0 \
  --subnet-file=./subnet.env \
  --healthz-port=2401 
```
**注意：**flannel集群各主机的启动脚本略有不同，部署其他节点时注意修改。

## 检查配置，权限，创建日志目录
`HDSS7-21.host.com`上：
```pwd /opt/flannel
[root@hdss7-21 flannel]# chmod +x /opt/flannel/flanneld.sh 

[root@hdss7-21 flannel]# mkdir -p /data/logs/flanneld
```

## 创建supervisor配置
`HDSS7-21.host.com`上：
```vi /etc/supervisord.d/flanneld.ini
[program:flanneld-7-21]
command=/opt/flannel/flanneld.sh                             ; the program (relative uses PATH, can take args)
numprocs=1                                                   ; number of processes copies to start (def 1)
directory=/opt/flannel                                       ; directory to cwd to before exec (def no cwd)
autostart=true                                               ; start at supervisord start (default: true)
autorestart=true                                             ; retstart at unexpected quit (default: true)
startsecs=22                   ; number of secs prog must stay running (def. 1)
startretries=3     				     ; max # of serial start failures (default 3)
exitcodes=0,2      				     ; 'expected' exit codes for process (default 0,2)
stopsignal=QUIT    				     ; signal used to kill process (default TERM)
stopwaitsecs=10    				     ; max num secs to wait b4 SIGKILL (default 10)
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

## 操作etcd，增加host-gw
`HDSS7-21.host.com`上：
```pwd /opt/etcd
[root@hdss7-21 etcd]# ./etcdctl set /coreos.com/network/config '{"Network": "172.7.0.0/16", "Backend": {"Type": "host-gw"}}'
{"Network": "172.7.0.0/16", "Backend": {"Type": "host-gw"}}
```

## 启动服务并检查
`HDSS7-21.host.com`上：
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
## 安装部署启动检查所有集群规划主机上的flannel服务
略

## 再次验证集群，POD之间网络互通

## 在各运算节点上优化iptables规则
**注意：**iptables规则各主机的略有不同，其他运算节点上执行时注意修改。
- 优化SNAT规则，各运算节点之间的各POD之间的网络通信不再出网

```
# iptables -t nat -D POSTROUTING -s 172.7.21.0/24 ! -o docker0 -j MASQUERADE
# iptables -t nat -I POSTROUTING -s 172.7.21.0/24 ! -d 172.7.0.0/16 ! -o docker0 -j MASQUERADE
```
> 10.4.7.21主机上的，来源是172.7.21.0/24段的docker的ip，目标ip不是172.7.0.0/16段，网络发包不从docker0桥设备出站的，才进行SNAT转换

## 各运算节点保存iptables规则
```
[root@hdss7-21 ~]# iptables-save > /etc/sysconfig/iptables
```

# 部署k8s资源配置清单的内网http服务
## 在运维主机`HDSS7-200.host.com`上，配置一个nginx虚拟主机，用以提供k8s统一的资源配置清单访问入口
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
## 配置内网DNS解析
`HDSS7-11.host.com`上
```vi /var/named/od.com.zone
k8s-yaml	60 IN A 10.4.7.200
```
以后所有的资源配置清单统一放置在运维主机的`/data/k8s-yaml`目录下即可
```
[root@hdss7-200 ~]# nginx -s reload
```


# 部署kube-dns(coredns)
[coredns官方GitHub](https://github.com/coredns/coredns)
## 准备coredns-v1.3.1镜像
运维主机`HDSS7-200.host.com`上：
```
[root@hdss7-200 ~]# docker pull coredns/coredns:1.3.1
1.3.1: Pulling from coredns/coredns
e0daa8927b68: Pull complete 
3928e47de029: Pull complete 
Digest: sha256:02382353821b12c21b062c59184e227e001079bb13ebd01f9d3270ba0fcbf1e4
Status: Downloaded newer image for coredns/coredns:1.3.1
[root@hdss7-200 ~]# docker tag eb516548c180 harbor.od.com/public/coredns:v1.3.1
[root@hdss7-200 ~]# docker push harbor.od.com/public/coredns:v1.3.1
docker push harbor.od.com/public/coredns:v1.3.1
The push refers to a repository [harbor.od.com/public/coredns]
c6a5fc8a3f01: Pushed 
fb61a074724d: Pushed 
v1.3.1: digest: sha256:e077b9680c32be06fc9652d57f64aa54770dd6554eb87e7a00b97cf8e9431fda size: 739
```

## 准备资源配置清单
运维主机`HDSS7-200.host.com`上：
```
[root@hdss7-200 ~]# mkdir -p /data/k8s-yaml/coredns && cd /data/k8s-yaml/coredns
```

{% tabs coredns%}
<!-- tab RBAC -->
vi /data/k8s-yaml/coredns/rbac.yaml
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
vi /data/k8s-yaml/coredns/cm.yaml
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
vi /data/k8s-yaml/coredns/dp.yaml
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
        image: harbor.od.com/public/coredns:v1.3.1
        args:
        - -conf
        - /etc/coredns/Corefile
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

## 依次执行创建
浏览器打开：http://k8s-yaml.od.com/coredns 检查资源配置清单文件是否正确创建
在任意运算节点上应用资源配置清单
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/coredns/rbac.yaml
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/coredns/cm.yaml
configmap/coredns created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/coredns/dp.yaml
deployment.extensions/coredns created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/coredns/svc.yaml
service/coredns created
```
## 检查
```
[root@hdss7-21 ~]# kubectl get pods -n kube-system -o wide
NAME                       READY   STATUS    RESTARTS   AGE
coredns-7ccccdf57c-5b9ch   1/1     Running   0          3m4s

[root@hdss7-21 coredns]# kubectl get svc -n kube-system
NAME      TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
coredns   ClusterIP   192.168.0.2   <none>        53/UDP,53/TCP   29s

[root@hdss7-21 ~]# dig -t A nginx-ds.default.svc.cluster.local. @192.168.0.2 +short
192.168.0.3
```

# 部署traefik（ingress）
[traefik官方GitHub](https://github.com/containous/traefik)
## 准备traefik镜像
运维主机`HDSS7-200.host.com`上：
```
[root@hdss7-200 ~]# docker pull traefik:v1.7-alpine
v1.7-alpine: Pulling from library/traefik
bdf0201b3a05: Pull complete 
9dfd896cc066: Pull complete 
de06b5685128: Pull complete 
c4d82a21fa27: Pull complete 
Digest: sha256:0531581bde9da0670fc2c7a4e419e1cc38abff74e7ba06410bf2b1b55c70ef15
Status: Downloaded newer image for traefik:v1.7-alpine
[root@hdss7-200 ~]# docker tag 1930b7508541 harbor.od.com/public/traefik:v1.7.2       
[root@hdss7-200 ~]# docker push harbor.od.com/public/traefik:v1.7.2
The push refers to a repository [harbor.od.com/public/traefik]
a3e3d574f6ae: Pushed 
a7c355c1a104: Pushed 
e89059911fc9: Pushed 
a464c54f93a9: Pushed
v1.7.2: digest: sha256:8f92899f5feb08db600c89d3016145e838fa7ff0d316ee21ecd63d9623643410 size: 1157
```
## 准备资源配置清单
运维主机`HDSS7-200.host.com`上：
```
[root@hdss7-200 ~]# mkdir -p /data/k8s-yaml/traefik && cd /data/k8s-yaml/traefik
```
{% tabs traefik%}
<!-- tab RBAC -->
vi /data/k8s-yaml/traefik/rbac.yaml
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
vi /data/k8s-yaml/traefik/ds.yaml
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
      - image: harbor.od.com/public/traefik:v1.7.2
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
        - -\-api
        - -\-kubernetes
        - -\-logLevel=INFO
        - -\-insecureskipverify=true
        - -\-kubernetes.endpoint=https://10.4.7.10:7443
        - -\-accesslog
        - -\-accesslog.filepath=/var/log/traefik_access.log
        - -\-traefiklog
        - -\-traefiklog.filepath=/var/log/traefik.log
        - -\-metrics.prometheus
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

## 解析域名
`HDSS7-11.host.com`上
```vi /var/named/od.com.zone
traefik	60 IN A 10.4.7.10
```

## 依次执行创建
浏览器打开：http://k8s-yaml.od.com/traefik 检查资源配置清单文件是否正确创建
在任意运算节点应用资源配置清单
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/traefik/rbac.yaml 
serviceaccount/traefik-ingress-controller created
clusterrole.rbac.authorization.k8s.io/traefik-ingress-controller created
clusterrolebinding.rbac.authorization.k8s.io/traefik-ingress-controller created

[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/traefik/ds.yaml 
daemonset.extensions/traefik-ingress-controller created

[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/traefik/svc.yaml 
service/traefik-ingress-service created

[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/traefik/ingress.yaml 
ingress.extensions/traefik-web-ui created
```

## 配置反代
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

## 浏览器访问
http://traefik.od.com

# 部署dashboard
[dashboard官方GitHub](https://github.com/kubernetes/dashboard)
## 准备dashboard镜像
运维主机`HDSS7-200.host.com`上：
```
[root@hdss7-200 ~]# docker pull k8scn/kubernetes-dashboard-amd64:v1.8.3
v1.8.3: Pulling from k8scn/kubernetes-dashboard-amd64
a4026007c47e: Pull complete 
Digest: sha256:ebc993303f8a42c301592639770bd1944d80c88be8036e2d4d0aa116148264ff
Status: Downloaded newer image for k8scn/kubernetes-dashboard-amd64:v1.8.3
[root@hdss7-200 ~]# docker tag 0c60bcf89900 harbor.od.com/public/dashboard:v1.8.3
[root@hdss7-200 ~]# docker push harbor.od.com/public/dashboard:v1.8.3
docker push harbor.od.com/public/dashboard:v1.8.3
The push refers to a repository [harbor.od.com/public/dashboard]
23ddb8cbb75a: Pushed 
v1.8.3: digest: sha256:e76c5fe6886c99873898e4c8c0945261709024c4bea773fc477629455631e472 size: 529
```
## 准备资源配置清单
运维主机`HDSS7-200.host.com`上：
```
[root@hdss7-200 ~]# mkdir -p /data/k8s-yaml/dashboard && cd /data/k8s-yaml/dashboard
```
{% tabs dashboard%}
<!-- tab RBAC -->
vi /data/k8s-yaml/dashboard/rbac.yaml
{% code %}
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
    addonmanager.kubernetes.io/mode: Reconcile
  name: kubernetes-dashboard-admin
  namespace: kube-system
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
  name: kubernetes-dashboard-admin
  namespace: kube-system
{% endcode %}
<!-- endtab -->
<!-- tab Deployment-->
vi /data/k8s-yaml/dashboard/dp.yaml
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
        image: harbor.od.com/public/dashboard:v1.8.3
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
          - -\-auto-generate-certificates
        volumeMounts:
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
      - name: tmp-volume
        emptyDir: {}
      serviceAccountName: kubernetes-dashboard-admin
      tolerations:
      - key: "CriticalAddonsOnly"
        operator: "Exists"
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

{% endtabs %}

## 解析域名
`HDSS7-11.host.com`上
```vi /var/named/od.com.zone
dashboard	60 IN A 10.4.7.10
```

## 依次执行创建
浏览器打开：http://k8s-yaml.od.com/dashboard 检查资源配置清单文件是否正确创建
在任意运算节点应用资源配置清单
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dashboard/rbac.yaml 
serviceaccount/kubernetes-dashboard created
role.rbac.authorization.k8s.io/kubernetes-dashboard-admin created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-admin created

[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dashboard/dp.yaml 
deployment.apps/kubernetes-dashboard created

[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dashboard/svc.yaml 
service/kubernetes-dashboard created

[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dashboard/ingress.yaml 
ingress.extensions/kubernetes-dashboard created
```

## 浏览器访问
http://dashboard.od.com

## 配置认证
- 下载新版dashboard

```
[root@hdss7-200 ~]# docker pull hexun/kubernetes-dashboard-amd64:v1.10.1
[root@hdss7-200 ~]# docker tag f9aed6605b81 harbor.od.com/public/dashboard:v1.10.1
[root@hdss7-200 ~]# docker push harbor.od.com/public/dashboard:v1.10.1
The push refers to a repository [harbor.od.com/public/dashboard]
fbdfe08b001c: Pushed 
v1.10.1: digest: sha256:d6b4e5d77c1cdcb54cd5697a9fe164bc08581a7020d6463986fe1366d36060e8 size: 529
```

- 应用新版dashboard

- 自签ssl证书

运维主机`HDSS7-200.host.com`上：
```pwd /opt/certs
[root@hdss7-200 certs]# (umask 077; openssl genrsa -out dashboard.od.com.key 2048)
Generating RSA private key, 2048 bit long modulus
.........................................................................................................................................................+++
.......................................................................+++

[root@hdss7-200 certs]# openssl req -new -key dashboard.od.com.key -out dashboard.od.com.csr -subj "/CN=*.od.com/ST=Beijing/L=beijing/O=od/OU=ops"
[root@hdss7-200 certs]# openssl x509 -req -in dashboard.od.com.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out dashboard.od.com.crt -days 365
Signature ok
subject=/CN=*.od.com/ST=Beijing/L=beijing/O=od/OU=ops
Getting CA Private Key
[root@hdss7-200 certs]# ls -l|grep dashboard.od.com
-rw-r--r-- 1 root root 1168 Jun 24 14:21 dashboard.od.com.crt
-rw-r--r-- 1 root root  976 Jun 24 14:21 dashboard.od.com.csr
-rw------- 1 root root 1679 Jun 24 14:21 dashboard.od.com.key
```

- 修改nginx配置，走https

代理节点`HDSS7-11.host.com`和`HDSS7-12.host.com`上：
```vi /etc/nginx/conf.d/dashboard.od.com.conf
server {
    listen       80;
    server_name  dashboard.od.com;

    rewrite ^(.*)$ https://${server_name}$1 permanent;
}
server {
    listen       443 ssl;
    server_name  dashboard.od.com;

    ssl_certificate "certs/dashboard.od.com.crt";
    ssl_certificate_key "certs/dashboard.od.com.key";
    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout  10m;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://default_backend_traefik;
        proxy_set_header Host       $http_host;
        proxy_set_header x-forwarded-for $proxy_add_x_forwarded_for;
    }
}
```

- 获取token

运算节点`HDSS7-21.host.com`上：
```
[root@hdsss7-21 ~]# kubectl describe secret kubernetes-dashboard-admin-token-rhr62 -n kube-system
Name:         kubernetes-dashboard-admin-token-rhr62
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: kubernetes-dashboard-admin
              kubernetes.io/service-account.uid: cdd3c552-856d-11e9-ae34-782bcb321c07

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1354 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbi10b2tlbi1yaHI2MiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImNkZDNjNTUyLTg1NmQtMTFlOS1hZTM0LTc4MmJjYjMyMWMwNyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTprdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiJ9.72OcJZCm_3I-7QZcEJTRPyIJSxQwSwZfVsB6Bx_RAZRJLOv3-BXy88PclYgxRy2dDqeX6cpjvFPBrmNOGQoxT9oD8_H49pvBnqdCdNuoJbXK7aBIZdkZxATzXd-63zmhHhUBsM3Ybgwy5XxD3vj8VUYfux5c5Mr4TzU_rnGLCj1H5mq_JJ3hNabv0rwil-ZAV-3HLikOMiIRhEK7RdMs1bfXF2yvse4VOabe9xv47TvbEYns97S4OlZvsurmOk0B8dD85OSaREEtqa8n_ND9GrHeeL4CcALqWYJHLrr7vLfndXi1QHDVrUzFKvgkAeYpDVAzGwIWL7rgHwp3sQguGA
```

- 使用token登录dashboard

## 部署heapster
[heapster官方github地址](https://github.com/kubernetes-retired/heapster)
### 准备heapster镜像
运维主机`HDSS7-200.host.com`上
```
[root@hdss7-200 ~]# docker pull quay.io/bitnami/heapster:1.5.4
1.5.4: Pulling from bitnami/heapster
4018396ca1ba: Pull complete 
0e4723f815c4: Pull complete 
d8569f30adeb: Pull complete 
Digest: sha256:6d891479611ca06a5502bc36e280802cbf9e0426ce4c008dd2919c2294ce0324
Status: Downloaded newer image for quay.io/bitnami/heapster:1.5.4
[root@hdss7--200 ~]# docker tag c359b95ad38b harbor.od.com/public/heapster:v1.5.4
[root@hdss7--200 ~]# docker push harbor.od.com/public/heapster:v1.5.4
docker push harbor.od.com/public/heapster:v1.5.4
The push refers to a repository [harbor.od.com/public/heapster]
20d37d828804: Pushed 
b9b192015e25: Pushed 
b76dba5a0109: Pushed 
v1.5.4: digest: sha256:1203b49f2b2b07e02e77263bce8bb30563a91e1d7ee7c6742e9d125abcb3abe6 size: 952
```

### 准备资源配置清单
```
[root@hdss7-200 ~]# mkdir -p /data/k8s-yaml/dashboard/heapster && cd /data/k8s-yaml/dashboard/heapster
```
{% tabs heapster%}
<!-- tab RBAC -->
vi /data/k8s-yaml/dashboard/heapster/rbac.yaml
{% code %}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: heapster
  namespace: kube-system
\--\-
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: heapster
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:heapster
subjects:
- kind: ServiceAccount
  name: heapster
  namespace: kube-system
{% endcode %}
<!-- endtab -->
<!-- tab Deployment-->
vi /data/k8s-yaml/dashboard/heapster/dp.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: heapster
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        task: monitoring
        k8s-app: heapster
    spec:
      serviceAccountName: heapster
      containers:
      - name: heapster
        image: harbor.od.com/k8s/heapster:v1.5.4
        imagePullPolicy: IfNotPresent
        command:
        - /opt/bitnami/heapster/bin/heapster
        - \--source=kubernetes:https://kubernetes.default
{% endcode %}
<!-- endtab -->
<!-- tab Service-->
vi /data/k8s-yaml/dashboard/heapster/svc.yaml
{% code %}
apiVersion: v1
kind: Service
metadata:
  labels:
    task: monitoring
    # For use as a Cluster add-on (https://github.com/kubernetes/kubernetes/tree/master/cluster/addons)
    # If you are NOT using this as an addon, you should comment out this line.
    kubernetes.io/cluster-service: 'true'
    kubernetes.io/name: Heapster
  name: heapster
  namespace: kube-system
spec:
  ports:
  - port: 80
    targetPort: 8082
  selector:
    k8s-app: heapster
{% endcode %}
<!-- endtab -->
{% endtabs %}

### 应用资源配置清单
任意运算节点上：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dashboard/heapster/rbac.yaml 
serviceaccount/heapster created
clusterrolebinding.rbac.authorization.k8s.io/heapster created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dashboard/heapster/dp.yaml 
deployment.extensions/heapster created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/dashboard/heapster/svc.yaml 
service/heapster created
```

### 重启dashboard
浏览器访问：http://dashboard.od.com
![加入heapster插件的dashboard](/images/heapster.png "加入heapster插件的dashboard")

