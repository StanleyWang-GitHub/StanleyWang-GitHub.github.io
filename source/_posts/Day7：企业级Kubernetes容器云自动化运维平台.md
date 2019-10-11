title: Day7：企业级Kubernetes容器云自动化运维平台
author: Stanley Wang
date: 2019-6-18 16:12:56
---
- - -
{% cq %}欢迎加入王导的VIP学习qq群：==>[<font color="FF7F50">932194668</font>](http://shang.qq.com/wpa/qunwpa?idkey=78869fddc5a661acb0639315eb52997c108de6625df5f0ee2f0372f176a032a6)<=={% endcq %}
- - -
# 部署对象式存储组件——minio
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
[root@hdss7-200 ~]# docker tag ed5e2b2624bf harbor.od.com/armory/minio:latest
[root@hdss7-200 ~]# docker push harbor.od.com/armory/minio:latest
The push refers to a repository [harbor.od.com/armory/minio]
e8c528077536: Pushed 
ad774a44659e: Pushed 
1a734f934625: Pushed 
f1b5933fe4b5: Pushed 
latest: digest: sha256:a676c2f71ad261d10cba2527094810e8a37f98d79086694ec162cb7fea64f5e0 size: 1158
```

## 准备资源配置清单
{% tabs armory-minio %}
<!-- tab Deployment -->
vi /data/k8s-yaml/armory/minio/dp.yaml
{% code %}
kind: Deployment
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    name: minio
  name: minio
  namespace: armory
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 7
  selector:
    matchLabels:
      name: minio
  template:
    metadata:
      labels:
        app: minio
        name: minio
    spec:
      containers:
      - name: minio
        image: harbor.od.com/armory/minio:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9000
          protocol: TCP
        args:
        - server
        - /data
        env:
        - name: MINIO_ACCESS_KEY
          value: admin
        - name: MINIO_SECRET_KEY
          value: admin123
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
        volumeMounts:
        - mountPath: /data
          name: data
      imagePullSecrets:
      - name: harbor
      volumes:
      - nfs:
          server: hdss7-200
          path: /data/nfs-volume/minio
        name: data
{% endcode %}
<!-- endtab -->
<!-- tab Service -->
vi /data/k8s-yaml/armory/minio/svc.yaml
{% code %}
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: armory
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 9000
  selector:
    app: minio
{% endcode %}
<!-- endtab -->
<!-- tab Ingress -->
vi /data/k8s-yaml/armory/minio/ingress.yaml
{% code %}
kind: Ingress
apiVersion: extensions/v1beta1
metadata: 
  name: minio
  namespace: armory
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
minio              A    10.4.7.10
```
## 应用资源配置清单
任意运算节点上：
```
[root@hdss7-21 ~]# kubectl apply -f https://k8s-yaml.od.com/armory/minio/dp.yaml 
deployment.extensions/minio created
[root@hdss7-21 ~]# kubectl apply -f https://k8s-yaml.od.com/armory/minio/svc.yaml 
service/minio created
[root@hdss7-21 ~]# kubectl apply -f https://k8s-yaml.od.com/armory/minio/ingress.yaml 
ingress.extensions/minio created
```
## 浏览器访问
http://minio.od.com

# 部署缓存组件——Redis
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
[root@hdss7-200 ~]# docker tag 720768125f4f harbor.od.com/armory/redis:v4.0.14
[root@hdss7-200 ~]# docker push harbor.od.com/armory/redis:v4.0.14
docker push harbor.od.com/armory/redis:v4.0.14
The push refers to a repository [harbor.od.com/armory/redis]
675b01d6b533: Pushed 
a6104c28b530: Pushed 
49be6cb1f430: Pushed 
03eafa792876: Pushed 
f99f83132c0a: Pushed 
6270adb5794c: Pushed 
v4.0.14: digest: sha256:b1aaa4a4e2c0ae115c1bae57ce0f957dd9440f52fae42af9e1c2e165ea7e0aba size: 1571
```
## 准备资源配置清单
{% tabs armory-redis %}
<!-- tab Deployment -->
vi /data/k8s-yaml/armory/redis/dp.yaml
{% code %}
kind: Deployment
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    name: redis
  name: redis
  namespace: armory
spec:
  replicas: 1
  revisionHistoryLimit: 7
  selector:
    matchLabels:
      name: redis
  template:
    metadata:
      labels:
        app: redis
        name: redis
    spec:
      containers:
      - name: redis
        image: harbor.od.com/armory/redis:v4.0.14
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 6379
          protocol: TCP
      imagePullSecrets:
      - name: harbor
{% endcode %}
<!-- endtab -->
<!-- tab Service -->
vi /data/k8s-yaml/armory/redis/svc.yaml
{% code %}
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: armory
spec:
  ports:
  - port: 6379
    protocol: TCP
    targetPort: 6379
  selector:
    app: redis
{% endcode %}
<!-- endtab -->
{% endtabs %}

## 应用资源配置清单
任意运算节点上：
```
[root@hdss7-21 ~]# kubectl apply -f https://k8s-yaml.od.com/armory/redis/dp.yaml 
deployment.extensions/redis created
[root@hdss7-21 ~]# kubectl apply -f https://k8s-yaml.od.com/armory/redis/svc.yaml 
service/redis created
```

# 部署K8S云驱动组件——CloudDriver
运维主机`HDSS7-200.host.com`上：
## 准备docker镜像
[镜像下载地址](https://hub.docker.io/armory/spinnaker-clouddriver-slim)
```
[root@hdss7-200 ~]# docker pull docker.io/armory/spinnaker-clouddriver-slim:release-1.8.x-14c9664
release-1.8.x-14c9664: Pulling from armory/spinnaker-clouddriver-slim
8e3ba11ec2a2: Pull complete 
311ad0da4533: Pull complete 
391a6a6b3651: Pull complete 
e06ac5cf1eba: Pull complete 
310ea36c6b4a: Pull complete 
46928b6e4f49: Pull complete 
91d309cd3217: Pull complete 
Digest: sha256:968fcae9ba2bd5456cb672a85a776c7fe535ed4b6e000ad457f2486245d51273
Status: Downloaded newer image for armory/spinnaker-clouddriver-slim:release-1.8.x-14c966
[root@hdss7-200 ~]# docker tag edb2507fdb62 harbor.od.com/armory/clouddriver:v1.8.x
[root@hdss7-200 ~]# docker push harbor.od.com/armory/clouddriver:v1.8.x
The push refers to a repository [harbor.od.com/armory/clouddriver]
2d512d83ffd6: Pushed 
658a102be22d: Pushed 
6852d72bcb09: Pushed 
c77bb5c0e352: Pushed 
8bc7bbcd76b6: Pushed 
298c3bb2664f: Pushed 
73046094a9b8: Pushed 
v1.8.x: digest: sha256:e8fdea9e6838d2c2b5fe07beac46a0450148f9ef37167971fb18e3b4e6b470a8 size: 1792
```
## 准备minio的secret
- 准备配置文件

运维主机`HDSS7-200.host.com`上：
```vi /data/k8s-yaml/armory/clouddriver/credentials
[default]
aws_access_key_id=admin
aws_secret_access_key=admin123
```
- 创建secret

任意运算节点上：
```
[root@hdss7-21 ~]# wget http://k8s-yaml.od.com/armory/clouddriver/credentials
[root@hdss7-21 ~]# kubectl create secret generic credentials --from-file=./credentials -n armory
secret/credentials created
```

## 准备cluster-admin用户配置
运维主机`HDSS7-200.host.com`上：
- 签发admin.pem、admin-key.pem
> 参考实验文档1

- 做admin.kubeconfig

任意运算节点上：
```
[root@hdss7-21 ~]# scp root@hdss7-200:/opt/certs/ca.pem .
[root@hdss7-21 ~]# scp root@hdss7-200:/opt/certs/admin.pem .
[root@hdss7-21 ~]# scp root@hdss7-200:/opt/certs/admin-key.pem .
[root@hdss7-21 ~]# kubectl config set-cluster myk8s --certificate-authority=./ca.pem --embed-certs=true --server=https://10.4.7.10:7443 --kubeconfig=config
Cluster "myk8s" set.
[root@hdss7-21 ~]# kubectl config set-credentials cluster-admin --client-certificate=./admin.pem --client-key=./admin-key.pem --embed-certs=true --kubeconfig=config
User "cluster-admin" set.
[root@hdss7-21 ~]# kubectl config set-context myk8s-context --cluster=myk8s --user=cluster-admin --kubeconfig=config
Context "myk8s-context" created.
[root@hdss7-21 ~]# kubectl config use-context myk8s-context --kubeconfig=config
Switched to context "myk8s-context".
[root@hdss7-21 ~]# kubectl create clusterrolebinding myk8s-admin --clusterrole=cluster-admin --user=cluster-admin
clusterrolebinding.rbac.authorization.k8s.io/myk8s-admin created
```

- 验证
> 将config文件拷贝至任意运算节点/root/.kube下，使用kubectl验证

- 创建cm

```pwd /root/.kube
[root@hdss7-21 .kube]# kubectl create cm default-kubeconfig --from-file=default-kubeconfig -n armory
```

## 准备资源配置清单
{% tabs armory-clouddriver %}
<!-- tab ConfigMap1 -->
vi /data/k8s-yaml/armory/clouddriver/init-env.yaml
{% code %}
kind: ConfigMap
apiVersion: v1
metadata:
  name: init-env
  namespace: armory
data:
  API_HOST: http://spinnaker.od.com/api
  ARMORY_ID: c02f0781-92f5-4e80-86db-0ba8fe7b8544
  ARMORYSPINNAKER_CONF_STORE_BUCKET: armory-platform
  ARMORYSPINNAKER_CONF_STORE_PREFIX: front50
  ARMORYSPINNAKER_GCS_ENABLED: "false"
  ARMORYSPINNAKER_S3_ENABLED: "true"
  AUTH_ENABLED: "false"
  AWS_REGION: us-east-1
  BASE_IP: 127.0.0.1
  CLOUDDRIVER_OPTS: -Dspring.profiles.active=armory,configurator,local
  CONFIGURATOR_ENABLED: "false"
  DECK_HOST: http://spinnaker.od.com
  ECHO_OPTS: -Dspring.profiles.active=armory,configurator,local
  GATE_OPTS: -Dspring.profiles.active=armory,configurator,local
  IGOR_OPTS: -Dspring.profiles.active=armory,configurator,local
  PLATFORM_ARCHITECTURE: k8s
  REDIS_HOST: redis://redis:6379
  SERVER_ADDRESS: 0.0.0.0
  SPINNAKER_AWS_DEFAULT_REGION: us-east-1
  SPINNAKER_AWS_ENABLED: "false"
  SPINNAKER_CONFIG_DIR: /home/spinnaker/config
  SPINNAKER_GOOGLE_PROJECT_CREDENTIALS_PATH: ""
  SPINNAKER_HOME: /home/spinnaker
  SPRING_PROFILES_ACTIVE: armory,configurator,local
{% endcode %}
<!-- endtab -->
<!-- tab ConfigMap2 -->
vi /data/k8s-yaml/armory/clouddriver/default-config.yaml
{% code %}
kind: ConfigMap
apiVersion: v1
metadata:
  name: default-config
  namespace: armory
data:
  barometer.yml: |
    server:
      port: 9092

    spinnaker:
      redis:
        host: ${services.redis.host}
        port: ${services.redis.port}
  clouddriver-armory.yml: |
    aws:
      defaultAssumeRole: role/${SPINNAKER_AWS_DEFAULT_ASSUME_ROLE:SpinnakerManagedProfile}
      accounts:
        - name: default-aws-account
          accountId: ${SPINNAKER_AWS_DEFAULT_ACCOUNT_ID:none}

      client:
        maxErrorRetry: 20

    serviceLimits:
      cloudProviderOverrides:
        aws:
          rateLimit: 15.0

      implementationLimits:
        AmazonAutoScaling:
          defaults:
            rateLimit: 3.0
        AmazonElasticLoadBalancing:
          defaults:
            rateLimit: 5.0

    security.basic.enabled: false
    management.security.enabled: false
  clouddriver-dev.yml: |
    #
    # Limit cloud provider polling as to not hit rate limits.
    #

    serviceLimits:
      defaults:
        rateLimit: 2
  clouddriver.yml: |
    server:
      port: ${services.clouddriver.port:7002}
      address: ${services.clouddriver.host:localhost}

    redis:
      connection: ${REDIS_HOST:redis://localhost:6379}

    udf:
      # Controls whether UserDataProviders are used to populate user data of new
      # server groups. If false, user data is copied over from ancestor server
      # groups on both CopyLastAsgAtomicOperation and
      # ModifyAsgLaunchConfigurationOperation (only if no user data is provided
      # on the given request).
      enabled: ${services.clouddriver.aws.udf.enabled:true}
      udfRoot: /opt/spinnaker/config/udf
      defaultLegacyUdf: false

    default:
      account:
        env: ${providers.aws.primaryCredentials.name}

    aws:
      # AWS Credentials are passed either as environment variables or through
      # a standard AWS file $HOME/.aws/credentials
      # See providers.aws.primaryCredentials in spinnaker.yml for more
      # info on setting environment variables.
      enabled: ${providers.aws.enabled:false}
      defaults:
        iamRole: ${providers.aws.defaultIAMRole:BaseIAMRole}
      defaultRegions:
        - name: ${providers.aws.defaultRegion:us-east-1}
      defaultFront50Template: ${services.front50.baseUrl}
      defaultKeyPairTemplate: ${providers.aws.defaultKeyPairTemplate}

    azure:
      enabled: ${providers.azure.enabled:false}

      accounts:
        - name: ${providers.azure.primaryCredentials.name}
          clientId: ${providers.azure.primaryCredentials.clientId}
          appKey: ${providers.azure.primaryCredentials.appKey}
          tenantId: ${providers.azure.primaryCredentials.tenantId}
          subscriptionId: ${providers.azure.primaryCredentials.subscriptionId}

    google:
      enabled: ${providers.google.enabled:false}

      accounts:
        - name: ${providers.google.primaryCredentials.name}
          project: ${providers.google.primaryCredentials.project}
          jsonPath: ${providers.google.primaryCredentials.jsonPath}
          consul:
            enabled: ${providers.google.primaryCredentials.consul.enabled:false}

    cf:
      enabled: ${providers.cf.enabled:false}

      accounts:
        - name: ${providers.cf.primaryCredentials.name}
          api: ${providers.cf.primaryCredentials.api}
          console: ${providers.cf.primaryCredentials.console}
          org: ${providers.cf.defaultOrg}
          space: ${providers.cf.defaultSpace}
          username: ${providers.cf.account.name:}
          password: ${providers.cf.account.password:}

    kubernetes:
      enabled: ${providers.kubernetes.enabled:false}
      accounts:
        - name: ${providers.kubernetes.primaryCredentials.name}
          dockerRegistries:
            - accountName: ${providers.kubernetes.primaryCredentials.dockerRegistryAccount}

    openstack:
      enabled: ${providers.openstack.enabled:false}
      accounts:
        - name: ${providers.openstack.primaryCredentials.name}
          authUrl: ${providers.openstack.primaryCredentials.authUrl}
          username: ${providers.openstack.primaryCredentials.username}
          password: ${providers.openstack.primaryCredentials.password}
          projectName: ${providers.openstack.primaryCredentials.projectName}
          domainName: ${providers.openstack.primaryCredentials.domainName:Default}
          regions: ${providers.openstack.primaryCredentials.regions}
          insecure: ${providers.openstack.primaryCredentials.insecure:false}
          userDataFile: ${providers.openstack.primaryCredentials.userDataFile:}

          # The Openstack API requires that the load balancer be in an ACTIVE
          # state for it to create associated relationships (i.e. listeners,
          # pools, monitors). Each modification will cause the load balancer to
          # go into a PENDING state and back to ACTIVE once the change has been
          # made. Depending on your implementation, the timeout and polling
          # intervals would need to be adjusted, especially if testing out
          # Spinnaker with Devstack or another resource constrained enviroment
          lbaas:
            pollTimeout: 60
            pollInterval: 5

    dockerRegistry:
      enabled: ${providers.dockerRegistry.enabled:false}
      accounts:
        - name: ${providers.dockerRegistry.primaryCredentials.name}
          address: ${providers.dockerRegistry.primaryCredentials.address}
          username: ${providers.dockerRegistry.primaryCredentials.username:}
          passwordFile: ${providers.dockerRegistry.primaryCredentials.passwordFile}

    credentials:
      primaryAccountTypes: ${providers.aws.primaryCredentials.name}, ${providers.google.primaryCredentials.name}, ${providers.cf.primaryCredentials.name}, ${providers.azure.primaryCredentials.name}
      challengeDestructiveActionsEnvironments: ${providers.aws.primaryCredentials.name}, ${providers.google.primaryCredentials.name}, ${providers.cf.primaryCredentials.name}, ${providers.azure.primaryCredentials.name}


    spectator:
      applicationName: ${spring.application.name}
      webEndpoint:
        enabled: ${services.spectator.webEndpoint.enabled:false}
        prototypeFilter:
          path: ${services.spectator.webEndpoint.prototypeFilter.path:}

      stackdriver:
        enabled: ${services.stackdriver.enabled}
        projectName: ${services.stackdriver.projectName}
        credentialsPath: ${services.stackdriver.credentialsPath}

    stackdriver:
      hints:
        - name: controller.invocations
          labels:
          - account
          - region
  dinghy.yml: ""
  echo-armory.yml: |
    diagnostics:
      enabled: true
      id: ${ARMORY_ID:unknown}

    armorywebhooks:
      enabled: false
      forwarding:
        baseUrl: http://armory-dinghy:8081
        endpoint: v1/webhooks
  echo-noncron.yml: |
    scheduler:
      enabled: false
  echo.yml: |
    server:
      port: ${services.echo.port:8089}
      address: ${services.echo.host:localhost}

    cassandra:
      enabled: ${services.echo.cassandra.enabled:false}
      embedded: ${services.cassandra.embedded:false}
      host: ${services.cassandra.host:localhost}

    spinnaker:
      baseUrl: ${services.deck.baseUrl}
      cassandra:
         enabled: ${services.echo.cassandra.enabled:false}
      inMemory:
         enabled: ${services.echo.inMemory.enabled:true}

    front50:
      baseUrl: ${services.front50.baseUrl:http://localhost:8080 }

    orca:
      baseUrl: ${services.orca.baseUrl:http://localhost:8083 }

    endpoints.health.sensitive: false

    slack:
      enabled: ${services.echo.notifications.slack.enabled:false}
      token: ${services.echo.notifications.slack.token}

    spring:
      mail:
        host: ${mail.host}

    mail:
      enabled: ${services.echo.notifications.mail.enabled:false}
      host: ${services.echo.notifications.mail.host}
      from: ${services.echo.notifications.mail.fromAddress}

    hipchat:
      enabled: ${services.echo.notifications.hipchat.enabled:false}
      baseUrl: ${services.echo.notifications.hipchat.url}
      token: ${services.echo.notifications.hipchat.token}

    twilio:
      enabled: ${services.echo.notifications.sms.enabled:false}
      baseUrl: ${services.echo.notifications.sms.url:https://api.twilio.com/ }
      account: ${services.echo.notifications.sms.account}
      token: ${services.echo.notifications.sms.token}
      from: ${services.echo.notifications.sms.from}

    scheduler:
      enabled: ${services.echo.cron.enabled:true}
      threadPoolSize: 20
      triggeringEnabled: true
      pipelineConfigsPoller:
        enabled: true
        pollingIntervalMs: 30000
      cron:
        timezone: ${services.echo.cron.timezone}

    spectator:
      applicationName: ${spring.application.name}
      webEndpoint:
        enabled: ${services.spectator.webEndpoint.enabled:false}
        prototypeFilter:
          path: ${services.spectator.webEndpoint.prototypeFilter.path:}

      stackdriver:
        enabled: ${services.stackdriver.enabled}
        projectName: ${services.stackdriver.projectName}
        credentialsPath: ${services.stackdriver.credentialsPath}

    webhooks:
      artifacts:
        enabled: true
  fetch.sh: |+
    #!/bin/bash -xe

    CONFIG_LOCATION=${SPINNAKER_HOME:-"/opt/spinnaker"}/config
    CONTAINER=$1

    rm -f /opt/spinnaker/config/*.yml

    mkdir -p ${CONFIG_LOCATION}

    # Setup the default configuration that comes with a distribution
    for filename in /opt/spinnaker/config/default/*.yml; do
        cp $filename ${CONFIG_LOCATION}
    done

    # User specific config
    if [ -d /opt/spinnaker/config/custom ]; then
        for filename in /opt/spinnaker/config/custom/*; do
            cp $filename ${CONFIG_LOCATION}
        done
    fi

    add_ca_certs() {
      # if CA exists, mount it into the default JKS store
      ca_cert_path="$1"
      jks_path="$2"
      alias="$3"

      if [[ "$(whoami)" != "root" ]]; then
        echo "INFO: I do not have proper permisions to add CA roots"
        return
      fi

      if [[ ! -f ${ca_cert_path} ]]; then
        echo "INFO: No CA cert found at ${ca_cert_path}"
        return
      fi
      keytool -importcert \
          -file ${ca_cert_path} \
          -keystore ${jks_path} \
          -alias ${alias} \
          -storepass changeit \
          -noprompt
    }

    if [ `which keytool` ]; then
      echo "INFO: Keytool found adding certs where appropriate"
      add_ca_certs "${CONFIG_LOCATION}/ca.crt" "/etc/ssl/certs/java/cacerts" "custom-ca"
      #we'll want to add saml, oauth, authn/authz stuff here too
    else
      echo "INFO: Keytool not found, not adding any certs/private keys"
    fi

    saml_pem_path="/opt/spinnaker/config/custom/saml.pem"
    saml_pkcs12_path="/tmp/saml.pkcs12"
    saml_jks_path="${CONFIG_LOCATION}/saml.jks"

    # for x509
    x509_ca_cert_path="/opt/spinnaker/config/custom/x509ca.crt"
    x509_client_cert_path="/opt/spinnaker/config/custom/x509client.crt"
    x509_jks_path="${CONFIG_LOCATION}/x509.jks"
    x509_nginx_cert_path="/opt/nginx/certs/ssl.crt"

    if [ "${CONTAINER}" == "gate" ]; then
        if [ -f ${saml_pem_path} ]; then
            echo "Loading ${saml_pem_path} into ${saml_jks_path}"
            # Convert PEM to PKCS12 with a password.
            openssl pkcs12 -export -out ${saml_pkcs12_path} -in ${saml_pem_path} -password pass:changeit -name saml
            keytool -genkey -v -keystore ${saml_jks_path} -alias saml \
                    -keyalg RSA -keysize 2048 -validity 10000 \
                    -storepass changeit -keypass changeit -dname "CN=armory"
            keytool -importkeystore \
                    -srckeystore ${saml_pkcs12_path} \
                    -srcstoretype PKCS12 \
                    -srcstorepass changeit \
                    -destkeystore ${saml_jks_path} \
                    -deststoretype JKS \
                    -storepass changeit \
                    -alias saml \
                    -destalias saml \
                    -noprompt
        else
            echo "No SAML IDP pemfile found at ${saml_pem_path}"
        fi
        if [ -f ${x509_ca_cert_path} ]; then
            echo "Loading ${x509_ca_cert_path} into ${x509_jks_path}"
            add_ca_certs ${x509_ca_cert_path} ${x509_jks_path} "ca"
        else
            echo "No x509 CA cert found at ${x509_ca_cert_path}"
        fi
        if [ -f ${x509_client_cert_path} ]; then
            echo "Loading ${x509_client_cert_path} into ${x509_jks_path}"
            add_ca_certs ${x509_client_cert_path} ${x509_jks_path} "client"
        else
            echo "No x509 Client cert found at ${x509_client_cert_path}"
        fi


        if [ -f ${x509_nginx_cert_path} ]; then
            echo "Creating a self-signed CA (EXPIRES IN 360 DAYS) with java keystore: ${x509_jks_path}"
            echo -e "\n\n\n\n\n\ny\n" | keytool -genkey -keyalg RSA -alias server -keystore keystore.jks -storepass changeit -validity 360 -keysize 2048
            keytool -importkeystore \
                    -srckeystore keystore.jks \
                    -srcstorepass changeit \
                    -destkeystore "${x509_jks_path}" \
                    -storepass changeit \
                    -srcalias server \
                    -destalias server \
                    -noprompt
        else
            echo "No x509 nginx cert found at ${x509_nginx_cert_path}"
        fi
    fi

    if [ "${CONTAINER}" == "nginx" ]; then
        nginx_conf_path="/opt/spinnaker/config/default/nginx.conf"
        if [ -f ${nginx_conf_path} ]; then
            cp ${nginx_conf_path} /etc/nginx/nginx.conf
        fi
    fi



  fiat.yml: |-
    server:
      port: ${services.fiat.port:7003}
      address: ${services.fiat.host:localhost}

    redis:
      connection: ${services.redis.connection:redis://localhost:6379}

    spectator:
      applicationName: ${spring.application.name}
      webEndpoint:
        enabled: ${services.spectator.webEndpoint.enabled:false}
        prototypeFilter:
          path: ${services.spectator.webEndpoint.prototypeFilter.path:}

      stackdriver:
        enabled: ${services.stackdriver.enabled}
        projectName: ${services.stackdriver.projectName}
        credentialsPath: ${services.stackdriver.credentialsPath}

    hystrix:
     command:
       default.execution.isolation.thread.timeoutInMilliseconds: 20000

    logging:
      level:
        com.netflix.spinnaker.fiat: DEBUG
  front50-armory.yml: |
    spinnaker:
      redis:
        enabled: true
        host: redis
  front50.yml: |
    server:
      port: ${services.front50.port:8080}
      address: ${services.front50.host:localhost}

    hystrix:
      command:
        default.execution.isolation.thread.timeoutInMilliseconds: 15000

    cassandra:
      enabled: ${services.front50.cassandra.enabled:false}
      embedded: ${services.cassandra.embedded:false}
      host: ${services.cassandra.host:localhost}

    aws:
      simpleDBEnabled: ${providers.aws.simpleDBEnabled:false}
      defaultSimpleDBDomain: ${providers.aws.defaultSimpleDBDomain}

    spinnaker:
      cassandra:
        enabled: ${services.front50.cassandra.enabled:false}
        host: ${services.cassandra.host:localhost}
        port: ${services.cassandra.port:9042}
        cluster: ${services.cassandra.cluster:CASS_SPINNAKER}
        keyspace: front50
        name: global

      redis:
        enabled: ${services.front50.redis.enabled:false}

      gcs:
        enabled: ${services.front50.gcs.enabled:false}
        bucket: ${services.front50.storage_bucket:}
        # See https://cloud.google.com/storage/docs/managing-buckets#manage-class-location
        bucketLocation: ${services.front50.bucket_location:}
        rootFolder: ${services.front50.rootFolder:front50}
        project: ${providers.google.primaryCredentials.project}
        jsonPath: ${providers.google.primaryCredentials.jsonPath}

      s3:
        enabled: ${services.front50.s3.enabled:false}
        bucket: ${services.front50.storage_bucket:}
        rootFolder: ${services.front50.rootFolder:front50}

    spectator:
      applicationName: ${spring.application.name}
      webEndpoint:
        enabled: ${services.spectator.webEndpoint.enabled:false}
        prototypeFilter:
          path: ${services.spectator.webEndpoint.prototypeFilter.path:}

      stackdriver:
        enabled: ${services.stackdriver.enabled}
        projectName: ${services.stackdriver.projectName}
        credentialsPath: ${services.stackdriver.credentialsPath}

    stackdriver:
      hints:
        - name: controller.invocations
          labels:
          - application
          - cause
        - name: aws.request.httpRequestTime
          labels:
          - status
          - exception
          - AWSErrorCode
        - name: aws.request.requestSigningTime
          labels:
          - exception
  gate-armory.yml: |+
    lighthouse:
        baseUrl: http://${DEFAULT_DNS_NAME:lighthouse}:5000

  gate.yml: |
    server:
      port: ${services.gate.port:8084}
      address: ${services.gate.host:localhost}

    # Circular references since we're already using 'services'
    # services:
    #   clouddriver:
    #     baseUrl: ${services.clouddriver.baseUrl:localhost:7002}
    #   orca:
    #     baseUrl: ${services.orca.baseUrl:localhost:8083}
    #   front50:
    #     baseUrl: ${services.front50.baseUrl:localhost:8080}
    # #optional services:
    #   echo:
    #     enabled: ${services.echo.enabled:true}
    #     baseUrl: ${services.echo.baseUrl:8089}
    #   flapjack:
    #     enabled: ${services.flapjack.enabled:false}
    #   igor:
    #     enabled: ${services.igor.enabled:false}
    #     baseUrl: ${services.igor.baseUrl:8088}

    redis:
      connection: ${REDIS_HOST:redis://localhost:6379}
      configuration:
        secure: true

    spectator:
      applicationName: ${spring.application.name}
      webEndpoint:
        enabled: ${services.spectator.webEndpoint.enabled:false}
        prototypeFilter:
          path: ${services.spectator.webEndpoint.prototypeFilter.path:}

      stackdriver:
        enabled: ${services.stackdriver.enabled}
        projectName: ${services.stackdriver.projectName}
        credentialsPath: ${services.stackdriver.credentialsPath}

    stackdriver:
      hints:
        - name: EurekaOkClient_Request
          labels:
          - cause
          - reason
          - status
  igor-nonpolling.yml: |
    jenkins:
      polling:
        enabled: false
  igor.yml: |
    server:
      port: ${services.igor.port:8088}
      address: ${services.igor.host:localhost}

    jenkins:
      enabled: ${services.jenkins.enabled:false}
      masters:
        - name: ${services.jenkins.defaultMaster.name}
          address: ${services.jenkins.defaultMaster.baseUrl}
          username: ${services.jenkins.defaultMaster.username}
          password: ${services.jenkins.defaultMaster.password}
          csrf: ${services.jenkins.defaultMaster.csrf:false}

    travis:
      enabled: ${services.travis.enabled:false}
      masters:
        - name: ${services.travis.defaultMaster.name}
          baseUrl: ${services.travis.defaultMaster.baseUrl}
          address: ${services.travis.defaultMaster.address}
          githubToken: ${services.travis.defaultMaster.githubToken}

    dockerRegistry:
      enabled: ${providers.dockerRegistry.enabled:false}

    redis:
      connection: ${REDIS_HOST:redis://localhost:6379}

    # Igor depends on Clouddriver and Echo. These are normally configured
    # in spinnaker[-local].yml (if present), otherwise, uncomment this.
    # services:
    #   clouddriver:
    #     baseUrl: ${services.clouddriver.baseUrl}
    #   echo:
    #     baseUrl: ${services.echo.baseUrl}

    spectator:
      applicationName: ${spring.application.name}
      webEndpoint:
        enabled: ${services.spectator.webEndpoint.enabled:false}
        prototypeFilter:
          path: ${services.spectator.webEndpoint.prototypeFilter.path:}

      stackdriver:
        enabled: ${services.stackdriver.enabled}
        projectName: ${services.stackdriver.projectName}
        credentialsPath: ${services.stackdriver.credentialsPath}

    stackdriver:
      hints:
        - name: controller.invocations
          labels:
          - master
  kayenta-armory.yml: |
    kayenta:
      aws:
        enabled: ${ARMORYSPINNAKER_S3_ENABLED:false}
        accounts:
          - name: aws-s3-storage
            bucket: ${ARMORYSPINNAKER_CONF_STORE_BUCKET}
            rootFolder: kayenta
            supportedTypes:
              - OBJECT_STORE
              - CONFIGURATION_STORE

      s3:
        enabled: ${ARMORYSPINNAKER_S3_ENABLED:false}

      google:
        enabled: ${ARMORYSPINNAKER_GCS_ENABLED:false}
        accounts:
          - name: cloud-armory
            # project: myproject
            # jsonPath: /opt/spinnaker/credentials/gcp.json
            bucket: ${ARMORYSPINNAKER_CONF_STORE_BUCKET}
            rootFolder: kayenta-prod
            supportedTypes:
              - METRICS_STORE
              - OBJECT_STORE
              - CONFIGURATION_STORE

      gcs:
        enabled: ${ARMORYSPINNAKER_GCS_ENABLED:false}
  kayenta.yml: |2

    server:
      port: 8090

    kayenta:
      atlas:
        enabled: false
    #    stageTimeoutMinutes: 3
    #    maxBackoffPeriodSeconds: 30
    #    accounts:
    #      - name:
    #        endpoint:
    #          baseUrl: http://localhost:7101
    #        namespace:
    #        supportedTypes:
    #          - METRICS_STORE

      google:
        enabled: false
    #    accounts:
    #      - name:
    #        project:
    #        jsonPath:
    #        bucket:
    #        rootFolder: kayenta
    #        supportedTypes:
    #          - METRICS_STORE
    #          - OBJECT_STORE
    #          - CONFIGURATION_STORE

      aws:
        enabled: false
    #    accounts:
    #      - name:
    #        bucket:
    #        rootFolder: kayenta
    #        supportedTypes:
    #          - OBJECT_STORE
    #          - CONFIGURATION_STORE

      datadog:
        enabled: false
    #    accounts:
    #      - name: my-datadog-account
    #        apiKey: xxxx
    #        applicationKey: xxxx
    #        supportedTypes:
    #          - METRICS_STORE
    #        endpoint.baseUrl: https://app.datadoghq.com

      prometheus:
        enabled: false
    #    accounts:
    #      - name: my-prometheus-account
    #        endpoint:
    #          baseUrl: http://localhost:9090
    #        supportedTypes:
    #          - METRICS_STORE

      gcs:
        enabled: false

      s3:
        enabled: false

      stackdriver:
        enabled: false

      memory:
        enabled: false

      configbin:
        enabled: false

    keiko:
      queue:
        redis:
          queueName: kayenta.keiko.queue
          deadLetterQueueName: kayenta.keiko.queue.deadLetters

    redis:
      connection: ${REDIS_HOST:redis://localhost:6379}

    spectator:
      applicationName: ${spring.application.name}
      webEndpoint:
        enabled: true

    swagger:
      enabled: true
      title: Kayenta API
      description:
      contact:
      patterns:
        - /admin.*
        - /canary.*
        - /canaryConfig.*
        - /canaryJudgeResult.*
        - /credentials.*
        - /fetch.*
        - /health
        - /judges.*
        - /metadata.*
        - /metricSetList.*
        - /metricSetPairList.*
        - /pipeline.*

    security.basic.enabled: false
    management.security.enabled: false
  nginx.conf: |
    user  nginx;
    worker_processes  1;

    error_log  /var/log/nginx/error.log warn;
    pid        /var/run/nginx.pid;


    events {
        worker_connections  1024;
    }


    http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;

        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';

        access_log  /var/log/nginx/access.log  main;

        sendfile        on;
        #tcp_nopush     on;

        keepalive_timeout  65;

        #gzip  on;

        include /etc/nginx/conf.d/*.conf;
    }

    stream {
        upstream gate_api {
            server armory-gate:8085;
        }

        server {
            listen 8085;
            proxy_pass gate_api;
        }
    }
  nginx.http.conf: |
    gzip on;
    gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript application/vnd.ms-fontobject application/x-font-ttf font/opentype image/svg+xml image/x-icon;

    server {
           listen 80;
           listen [::]:80;

           location / {
                proxy_pass http://armory-deck/;
           }

           location /api/ {
                proxy_pass http://armory-gate:8084/;
           }

           location /slack/ {
               proxy_pass http://armory-platform:10000/;
           }

           rewrite ^/login(.*)$ /api/login$1 last;
           rewrite ^/auth(.*)$ /api/auth$1 last;
    }
  nginx.https.conf: |
    gzip on;
    gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript application/vnd.ms-fontobject application/x-font-ttf font/opentype image/svg+xml image/x-icon;

    server {
        listen 80;
        listen [::]:80;
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;
        listen [::]:443 ssl;

        ssl on;

        ssl_certificate /opt/nginx/certs/ssl.crt;
        ssl_certificate_key /opt/nginx/certs/ssl.key;

        location / {
            proxy_pass http://armory-deck/;
        }

        location /api/ {
            proxy_pass http://armory-gate:8084/;
            proxy_set_header Host            $host;
            proxy_set_header X-Real-IP       $proxy_protocol_addr;
            proxy_set_header X-Forwarded-For $proxy_protocol_addr;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /slack/ {
            proxy_pass http://armory-platform:10000/;
        }

        rewrite ^/login(.*)$ /api/login$1 last;
        rewrite ^/auth(.*)$ /api/auth$1 last;
    }
  orca-armory.yml: |
    mine:
      baseUrl: http://${services.barometer.host}:${services.barometer.port}

    pipelineTemplate:
      enabled: ${features.pipelineTemplates.enabled:false}
      jinja:
        enabled: true

    kayenta:
      enabled: ${services.kayenta.enabled:false}
      baseUrl: ${services.kayenta.baseUrl}

    jira:
      enabled: ${features.jira.enabled:false}
      # Fill in your basic auth:  Base64("user:pass")
      basicAuth:  "Basic ${features.jira.basicAuthToken}"
      # ex. https://myjira.atlassian.net/rest/api/2/issue/
      url: ${features.jira.createIssueUrl}

    webhook:
      preconfigured:
        - label: Enforce Pipeline Policy
          description: Checks pipeline configuration against policy requirements
          type: enforcePipelinePolicy
          enabled: ${features.certifiedPipelines.enabled:false}
          url: "http://lighthouse:5000/v1/pipelines/${execution.application}/${execution.pipelineConfigId}?check_policy=yes"
          headers:
            Accept:
              - application/json
          method: GET
          waitForCompletion: true
          statusUrlResolution: getMethod
          statusJsonPath: $.status
          successStatuses: pass
          canceledStatuses:
          terminalStatuses: TERMINAL

        - label: "Jira: Create Issue"
          description:  Enter a Jira ticket when this pipeline runs
          type: createJiraIssue
          enabled: ${jira.enabled}
          url:  ${jira.url}
          customHeaders:
            "Content-Type": application/json
            Authorization: ${jira.basicAuth}
          method: POST
          parameters:
            - name: summary
              label: Issue Summary
              description: A short summary of your issue.
            - name: description
              label: Issue Description
              description: A longer description of your issue.
            - name: projectKey
              label: Project key
              description: The key of your JIRA project.
            - name: type
              label: Issue Type
              description: The type of your issue, e.g. "Task", "Story", etc.
          payload: |
            {
              "fields" : {
                "description": "${parameterValues['description']}",
                "issuetype": {
                   "name": "${parameterValues['type']}"
                },
                "project": {
                   "key": "${parameterValues['projectKey']}"
                },
                "summary":  "${parameterValues['summary']}"
              }
            }
          waitForCompletion: false

        - label: "Jira: Update Issue"
          description:  Update a previously created Jira Issue
          type: updateJiraIssue
          enabled: ${jira.enabled}
          url: "${execution.stages.?[type == 'createJiraIssue'][0]['context']['buildInfo']['self']}"
          customHeaders:
            "Content-Type": application/json
            Authorization: ${jira.basicAuth}
          method: PUT
          parameters:
            - name: summary
              label: Issue Summary
              description: A short summary of your issue.
            - name: description
              label: Issue Description
              description: A longer description of your issue.
          payload: |
            {
              "fields" : {
                "description": "${parameterValues['description']}",
                "summary": "${parameterValues['summary']}"
              }
            }
          waitForCompletion: false

        - label: "Jira: Transition Issue"
          description:  Change state of existing Jira Issue
          type: transitionJiraIssue
          enabled: ${jira.enabled}
          url: "${execution.stages.?[type == 'createJiraIssue'][0]['context']['buildInfo']['self']}/transitions"
          customHeaders:
            "Content-Type": application/json
            Authorization: ${jira.basicAuth}
          method: POST
          parameters:
            - name: newStateID
              label: New State ID
              description: The ID of the state you want to transition the issue to.
          payload: |
            {
              "transition" : {
                "id" : "${parameterValues['newStateID']}"
              }
            }
          waitForCompletion: false

        - label: "Jira: Add Comment"
          description:  Add a comment to an existing Jira Issue
          type: commentJiraIssue
          enabled: ${jira.enabled}
          url: "${execution.stages.?[type == 'createJiraIssue'][0]['context']['buildInfo']['self']}/comment"
          customHeaders:
            "Content-Type": application/json
            Authorization: ${jira.basicAuth}
          method: POST
          parameters:
            - name: body
              label: Comment body
              description: The text body of the component.
          payload: |
            {
              "body" : "${parameterValues['body']}"
            }
          waitForCompletion: false

  orca.yml: |
    server:
        port: ${services.orca.port:8083}
        address: ${services.orca.host:localhost}

    oort:
        baseUrl: ${services.oort.baseUrl:localhost:7002}
    front50:
        baseUrl: ${services.front50.baseUrl:localhost:8080}
    mort:
        baseUrl: ${services.mort.baseUrl:localhost:7002}
    kato:
        baseUrl: ${services.kato.baseUrl:localhost:7002}
    bakery:
        baseUrl: ${services.bakery.baseUrl:localhost:8087}
        extractBuildDetails: ${services.bakery.extractBuildDetails:true}
        allowMissingPackageInstallation: ${services.bakery.allowMissingPackageInstallation:true}
    echo:
        enabled: ${services.echo.enabled:false}
        baseUrl: ${services.echo.baseUrl:8089}

    igor:
        baseUrl: ${services.igor.baseUrl:8088}

    flex:
      baseUrl: http://not-a-host

    default:
      bake:
        account: ${providers.aws.primaryCredentials.name}
      securityGroups:
      vpc:
        securityGroups:

    redis:
      connection: ${REDIS_HOST:redis://localhost:6379}

    tasks:
      executionWindow:
        timezone: ${services.orca.timezone}

    spectator:
      applicationName: ${spring.application.name}
      webEndpoint:
        enabled: ${services.spectator.webEndpoint.enabled:false}
        prototypeFilter:
          path: ${services.spectator.webEndpoint.prototypeFilter.path:}

      stackdriver:
        enabled: ${services.stackdriver.enabled}
        projectName: ${services.stackdriver.projectName}
        credentialsPath: ${services.stackdriver.credentialsPath}

    stackdriver:
      hints:
        - name: controller.invocations
          labels:
          - application
  rosco-armory.yml: |
    redis:
      timeout: 50000

    rosco:
      jobs:
        local:
          timeoutMinutes: 60
  rosco.yml: |
    server:
      port: ${services.rosco.port:8087}
      address: ${services.rosco.host:localhost}

    redis:
      connection: ${REDIS_HOST:redis://localhost:6379}

    aws:
      enabled: ${providers.aws.enabled:false}

    docker:
      enabled: ${services.docker.enabled:false}
      bakeryDefaults:
        targetRepository: ${services.docker.targetRepository}

    google:
      enabled: ${providers.google.enabled:false}
      accounts:
        - name: ${providers.google.primaryCredentials.name}
          project: ${providers.google.primaryCredentials.project}
          jsonPath: ${providers.google.primaryCredentials.jsonPath}
      gce:
        bakeryDefaults:
          zone: ${providers.google.defaultZone}

    rosco:
      configDir: ${services.rosco.configDir}
      jobs:
        local:
          timeoutMinutes: 30

    spectator:
      applicationName: ${spring.application.name}
      webEndpoint:
        enabled: ${services.spectator.webEndpoint.enabled:false}
        prototypeFilter:
          path: ${services.spectator.webEndpoint.prototypeFilter.path:}

      stackdriver:
        enabled: ${services.stackdriver.enabled}
        projectName: ${services.stackdriver.projectName}
        credentialsPath: ${services.stackdriver.credentialsPath}

    stackdriver:
      hints:
        - name: bakes
          labels:
          - success
  spinnaker-armory.yml: |
    armory:
      architecture: 'k8s'

    features:
      artifacts:
        enabled: true
      # features are should be turned on in {ENV}.env. ex: prod.env
      pipelineTemplates:
        enabled: ${PIPELINE_TEMPLATES_ENABLED:false}
      infrastructureStages:
        enabled: ${INFRA_ENABLED:false}
      certifiedPipelines:
        enabled: ${CERTIFIED_PIPELINES_ENABLED:false}
      configuratorEnabled:
        enabled: true
      configuratorWizard:
        enabled: true
      configuratorCerts:
        enabled: true
      loadtestStage:
        enabled: ${LOADTEST_ENABLED:false}
      jira:
        # These settings are for the Jira Stages (webhook-based):
        enabled: ${JIRA_ENABLED:false}
        # Should be the basic Authorization header value token, the Base64
        # encoded version of "username:password".
        basicAuthToken: ${JIRA_BASIC_AUTH}
        # Should be the "create issue" endpoint, for example:
        #    https://armory.atlassian.net/rest/api/2/issue/
        url: ${JIRA_URL}

        # These setings are for Echo's Jira integration
        login: ${JIRA_LOGIN}
        password: ${JIRA_PASSWORD}

      slaEnabled:
        enabled: ${SLA_ENABLED:false}
      chaosMonkey:
        enabled: ${CHAOS_ENABLED:false}

      armoryPlatform:
        enabled: ${PLATFORM_ENABLED:false}
        uiEnabled: ${PLATFORM_UI_ENABLED:false}


    services:
      default:
        host: ${DEFAULT_DNS_NAME:localhost}

      clouddriver:
        host: ${DEFAULT_DNS_NAME:armory-clouddriver}
        entityTags:
          enabled: false

      configurator:
        baseUrl: http://${CONFIGURATOR_HOST:armory-configurator}:8069

      echo:
        host: ${DEFAULT_DNS_NAME:armory-echo}

      deck:
        gateUrl: ${API_HOST:service.default.host}
        baseUrl: ${DECK_HOST:armory-deck}

      dinghy:
        enabled: ${DINGHY_ENABLED:false}
        host: ${DEFAULT_DNS_NAME:armory-dinghy}
        baseUrl: ${services.default.protocol}://${services.dinghy.host}:${services.dinghy.port}
        port: 8081

      front50:
        host: ${DEFAULT_DNS_NAME:armory-front50}
        cassandra:
          enabled: false
        redis:
          enabled: true
        gcs:
          enabled: ${ARMORYSPINNAKER_GCS_ENABLED:false}
        s3:
          enabled: ${ARMORYSPINNAKER_S3_ENABLED:false}
        storage_bucket: ${ARMORYSPINNAKER_CONF_STORE_BUCKET}
        rootFolder: ${ARMORYSPINNAKER_CONF_STORE_PREFIX:front50}

      gate:
        host: ${DEFAULT_DNS_NAME:armory-gate}

      igor:
        host: ${DEFAULT_DNS_NAME:armory-igor}

      kayenta:
        enabled: true
        host: ${DEFAULT_DNS_NAME:armory-kayenta}
        canaryConfigStore: true
        port: 8090
        baseUrl: ${services.default.protocol}://${services.kayenta.host}:${services.kayenta.port}
        metricsStore: ${METRICS_STORE:stackdriver}
        metricsAccountName: ${METRICS_ACCOUNT_NAME}
        storageAccountName: ${STORAGE_ACCOUNT_NAME}
        atlasWebComponentsUrl: ${ATLAS_COMPONENTS_URL:}

      lighthouse:
        host: ${DEFAULT_DNS_NAME:armory-lighthouse}
        port: 5000
        baseUrl: ${services.default.protocol}://${services.lighthouse.host}:${services.lighthouse.port}

      orca:
        host: ${DEFAULT_DNS_NAME:armory-orca}

      platform:
        enabled: ${PLATFORM_ENABLED:false}
        host: ${DEFAULT_DNS_NAME:armory-platform}
        baseUrl: ${services.default.protocol}://${services.platform.host}:${services.platform.port}
        port: 5001

      rosco:
        host: ${DEFAULT_DNS_NAME:armory-rosco}
        enabled: true
        configDir: /opt/spinnaker/config/packer

      bakery:
        allowMissingPackageInstallation: true

      barometer:
        enabled: ${BAROMETER_ENABLED:false}
        host: ${DEFAULT_DNS_NAME:armory-barometer}
        baseUrl: ${services.default.protocol}://${services.barometer.host}:${services.barometer.port}
        port: 9092
        newRelicEnabled: ${NEW_RELIC_ENABLED:false}

      redis:
        # If you are using a remote redis server, you can set the host here.
        # If the remote server is on a different port or url, you can add
        # a "port" or "baseUrl" field here instead.
        host: redis
        port: 6379
        connection: ${REDIS_HOST:redis://localhost:6379}

      fiat:
        enabled: ${FIAT_ENABLED:false}
        host: ${DEFAULT_DNS_NAME:armory-fiat}
        port: 7003
        baseUrl: ${services.default.protocol}://${services.fiat.host}:${services.fiat.port}

    providers:
      aws:
        enabled: ${SPINNAKER_AWS_ENABLED:true}
        defaultRegion: ${SPINNAKER_AWS_DEFAULT_REGION:us-west-2}
        defaultIAMRole: ${SPINNAKER_AWS_DEFAULT_IAM_ROLE:SpinnakerInstanceProfile}
        defaultAssumeRole: ${SPINNAKER_AWS_DEFAULT_ASSUME_ROLE:SpinnakerManagedProfile}
        primaryCredentials:
          name: ${SPINNAKER_AWS_DEFAULT_ACCOUNT:default-aws-account}
          # The actual credentials are set using a standard AWS client library mechanism
          # http://docs.aws.amazon.com/AWSSdkDocsJava/latest/DeveloperGuide/credentials.html
          # Typically this is a $HOME/.aws/credentials file (remember, a standard
          # spinnaker installation runs as user "spinnaker" whose $HOME is
          # /home/spinnaker). The primaryCredentials.name will identify which profile
          # to use (for .aws/credentials).

      kubernetes:
        proxy: localhost:8001
        apiPrefix: api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard/#
  spinnaker.yml: |2

    # This file is intended to serve as a master configuration for a Spinnaker
    # deployment. Customizations to the deployment should be made in another file
    # named "spinnaker-local.yml". The distribution has a prototype called
    # "default-spinnaker-local.yml" which calls out the subset of attributes of
    # general interest. It can be copied into a "spinnaker-local.yml" to start
    # with. The prototype does not change any of the default values here, it just
    # surfaces the more critical attributes.

    global:
      spinnaker:
        timezone: 'America/Los_Angeles'
        architecture: ${PLATFORM_ARCHITECTURE}

    services:
      default:
        # These defaults can be modified to change all the spinnaker subsystems
        # (clouddriver, gate, etc) at once, but not external systems (jenkins, etc).
        # Individual systems can still be overridden using their own section entry
        # directly under 'services'.
        host: localhost
        protocol: http

      clouddriver:
        host: ${services.default.host}
        port: 7002
        baseUrl: ${services.default.protocol}://${services.clouddriver.host}:${services.clouddriver.port}
        aws:
          udf:
            # Controls whether UserDataProviders are used to populate user data of
            # new server groups. If false, user data is copied over from ancestor
            # server groups on both CopyLastAsgAtomicOperation and
            # ModifyAsgLaunchConfigurationOperation (only if no user data is
            # provided on the given request).
            enabled: true

      echo:
        enabled: true
        host: ${services.default.host}
        port: 8089
        baseUrl: ${services.default.protocol}://${services.echo.host}:${services.echo.port}

        # Persistence mechanism to use
        cassandra:
          enabled: false
        inMemory:
          enabled: true

        cron:
          # Allow pipeline triggers to run periodically via cron expressions.
          enabled: true
          timezone: ${global.spinnaker.timezone}

        notifications:
          # The following blocks can enable Spinnaker to send notifications
          # using the corresponding mechanism.
          # See http://www.spinnaker.io/docs/notifications-and-events-guide
          # for more information.
          mail:
            enabled: false
            host: # the smtp host
            fromAddress: # the address for which emails are sent from
          hipchat:
            enabled: false
            url: # the hipchat server to connect to
            token: # the hipchat auth token
            botName: # the username of the bot
          sms:
            enabled: false
            account: # twilio account id
            token: # twilio auth token
            from: # phone number by which sms messages are sent
          slack:
            # See https://api.slack.com/bot-users for details about using bots
            # and how to create your own bot user.
            enabled: false
            token: # the API token for the bot
            botName: # the username of the bot

      deck:
        # Frontend configuration.
        # If you are proxying Spinnaker behind a single host, you may want to
        # override these values. Remember to run `reconfigure_spinnaker.sh` after.
        host: ${services.default.host}
        port: 9000
        baseUrl: ${services.default.protocol}://${services.deck.host}:${services.deck.port}
        gateUrl: ${API_HOST:services.gate.baseUrl}
        bakeryUrl: ${services.bakery.baseUrl}
        timezone: ${global.spinnaker.timezone}
        auth:
          enabled: ${AUTH_ENABLED:false}

      fiat:
        enabled: false
        host: ${services.default.host}
        port: 7003
        baseUrl: ${services.default.protocol}://${services.fiat.host}:${services.fiat.port}

      front50:
        host: ${services.default.host}
        port: 8080
        baseUrl: ${services.default.protocol}://${services.front50.host}:${services.front50.port}

        # To use a cloud storage bucket on Amazon S3 or Google Cloud Storage instead
        # of cassandra, set the storage_bucket, disable cassandra, and enable one of
        # the service providers.
        storage_bucket: ${SPINNAKER_DEFAULT_STORAGE_BUCKET:}
        # (GCS Only) Location for bucket.
        bucket_location:
        bucket_root: front50

        cassandra:
          enabled: false
        redis:
          enabled: false
        gcs:
          enabled: false
        s3:
          enabled: false

      gate:
        host: ${services.default.host}
        port: 8084
        baseUrl: ${services.default.protocol}://${services.gate.host}:${services.gate.port}

      igor:
        # If you are integrating Jenkins then you must also enable Spinnaker's
        # "igor" subsystem.
        enabled: false
        host: ${services.default.host}
        port: 8088
        baseUrl: ${services.default.protocol}://${services.igor.host}:${services.igor.port}

      kato:
        host: ${services.clouddriver.host}
        port: ${services.clouddriver.port}
        baseUrl: ${services.clouddriver.baseUrl}

      mort:
        host: ${services.clouddriver.host}
        port: ${services.clouddriver.port}
        baseUrl: ${services.clouddriver.baseUrl}

      orca:
        host: ${services.default.host}
        port: 8083
        baseUrl: ${services.default.protocol}://${services.orca.host}:${services.orca.port}
        timezone: ${global.spinnaker.timezone}
        enabled: true

      oort:
        host: ${services.clouddriver.host}
        port: ${services.clouddriver.port}
        baseUrl: ${services.clouddriver.baseUrl}

      rosco:
        host: ${services.default.host}
        port: 8087
        baseUrl: ${services.default.protocol}://${services.rosco.host}:${services.rosco.port}
        # You need to provide the fully-qualified path to the directory containing
        # the packer templates.
        # They typically live in rosco's config/packer directory.
        configDir: /opt/rosco/config/packer

      bakery:
        host: ${services.rosco.host}
        port: ${services.rosco.port}
        baseUrl: ${services.rosco.baseUrl}
        extractBuildDetails: true
        allowMissingPackageInstallation: false

      docker:
        # This target repository is used by the bakery to publish baked docker images.
        # Do not include http://.
        targetRepository: # Optional, but expected in spinnaker-local.yml if specified.

      jenkins:
        # If you are integrating Jenkins, set its location here using the baseUrl
        # field and provide the username/password credentials.
        # You must also enable the "igor" service listed separately.
        # The "name" entry is used for the display name when selecting
        # this server.
        #
        # If you have multiple jenkins servers, you will need to list
        # them in an igor-local.yml. See jenkins.masters in config/igor.yml.
        #
        # Note that jenkins is not installed with Spinnaker so you must obtain this
        # on your own if you are interested.
        enabled: ${services.igor.enabled:false}
        defaultMaster:
          name: Jenkins
          baseUrl:   # Expected in spinnaker-local.yml
          username:  # Expected in spinnaker-local.yml
          password:  # Expected in spinnaker-local.yml

      redis:
        host: redis
        port: 6379
        connection: ${REDIS_HOST:redis://localhost:6379}

      cassandra:
        # cassandra.enabled is no longer used
        # cassandra is enabled or disabled on a per-service basis
        # where the alternative persistence mechanism for that service
        # can be enabled.
        host: ${services.default.host}
        port: 9042
        embedded: false
        cluster: CASS_SPINNAKER

      travis:
        # If you are integrating Travis, set its location here using the baseUrl
        # and adress fields and provide the githubToken for authentication.
        # You must also enable the "igor" service listed separately.
        #
        # If you have multiple travis servers, you will need to list
        # them in an igor-local.yml. See travis.masters in config/igor.yml.
        #
        # Note that travis is not installed with Spinnaker so you must obtain this
        # on your own if you are interested.
        enabled: false
        defaultMaster:
          name: ci # The display name for this server. Gets prefixed with "travis-"
          baseUrl: https://travis-ci.com
          address: https://api.travis-ci.org
          githubToken: # GitHub scopes currently required by Travis is required.

      spectator:
        webEndpoint:
          enabled: false

      stackdriver:
        enabled: ${SPINNAKER_STACKDRIVER_ENABLED:false}
        projectName: ${SPINNAKER_STACKDRIVER_PROJECT_NAME:${providers.google.primaryCredentials.project}}
        credentialsPath: ${SPINNAKER_STACKDRIVER_CREDENTIALS_PATH:${providers.google.primaryCredentials.jsonPath}}


    providers:
      aws:
        # For more information on configuring Amazon Web Services (aws), see
        # http://www.spinnaker.io/v1.0/docs/target-deployment-setup#section-amazon-web-services-setup

        enabled: ${SPINNAKER_AWS_ENABLED:false}
        simpleDBEnabled: false
        defaultRegion: ${SPINNAKER_AWS_DEFAULT_REGION:us-west-2}
        defaultIAMRole: BaseIAMRole
        defaultSimpleDBDomain: CLOUD_APPLICATIONS
        primaryCredentials:
          name: default
          # The actual credentials are set using a standard AWS client library mechanism
          # http://docs.aws.amazon.com/AWSSdkDocsJava/latest/DeveloperGuide/credentials.html
          # Typically this is a $HOME/.aws/credentials file (remember, a standard
          # spinnaker installation runs as user "spinnaker" whose $HOME is
          # /home/spinnaker). The primaryCredentials.name will identify which profile
          # to use (for .aws/credentials).

        # {{name}} will be interpolated with the aws account name (e.g. "my-aws-account-keypair").
        defaultKeyPairTemplate: "{\{name}\}-keypair"

      google:
        # For more information on configuring Google Cloud Platform (google), see
        # http://www.spinnaker.io/v1.0/docs/target-deployment-setup#section-google-cloud-platform-setup

        enabled: ${SPINNAKER_GOOGLE_ENABLED:false}
        defaultRegion: ${SPINNAKER_GOOGLE_DEFAULT_REGION:us-central1}
        defaultZone: ${SPINNAKER_GOOGLE_DEFAULT_ZONE:us-central1-f}

        primaryCredentials:
          name: my-account-name
          # The project is the Google Project ID for the project to manage with
          # Spinnaker. The jsonPath is a path to the JSON service credentials
          # downloaded from the Google Developer's Console.
          project: ${SPINNAKER_GOOGLE_PROJECT_ID:}
          jsonPath: ${SPINNAKER_GOOGLE_PROJECT_CREDENTIALS_PATH:}
          consul:
            enabled: ${SPINNAKER_GOOGLE_CONSUL_ENABLED:false}

      cf:
        # For more information on configuring Cloud Foundry (cf) support, see
        # http://www.spinnaker.io/v1.0/docs/target-deployment-setup#section-cloud-foundry-platform-setup

        enabled: false
        defaultOrg: spinnaker-cf-org
        defaultSpace: spinnaker-cf-space
        primaryCredentials:
          name: my-cf-account
          api: my-cf-api-uri
          console: my-cf-console-base-url
          # You must also supply cf.account.username and cf.account.password through env properties

      azure:
        # For more information on configuring Microsoft Azure (azure), see
        # http://www.spinnaker.io/v1.0/docs/target-deployment-setup#section-azure-cloud-platform-setup

        enabled: ${SPINNAKER_AZURE_ENABLED:false}
        defaultRegion: ${SPINNAKER_AZURE_DEFAULT_REGION:westus}
        primaryCredentials:
          name: my-azure-account

          # To set Azure credentials, enter your Azure supscription values for:
          # clientId, appKey, tenantId, and subscriptionId.
          clientId:
          appKey:
          tenantId:
          subscriptionId:

      titan:
        # If you want to deploy some services to titan,
        # set enabled and provide primary credentials for deploying.
        # Enabling titan is independent of other providers.
        enabled: false
        defaultRegion: us-east-1
        primaryCredentials:
          name: my-titan-account

      kubernetes:
        # For more information on configuring Kubernetes clusters (kubernetes), see
        # http://www.spinnaker.io/v1.0/docs/target-deployment-setup#section-kubernetes-cluster-setup

        # NOTE: enabling kubernetes also requires enabling dockerRegistry.
        enabled: ${SPINNAKER_KUBERNETES_ENABLED:false}
        primaryCredentials:
          name: my-kubernetes-account
          namespace: default
          dockerRegistryAccount: ${providers.dockerRegistry.primaryCredentials.name}

      dockerRegistry:
        # If you want to use a container based provider, you need to configure and
        # enable this provider to cache images.
        enabled: ${SPINNAKER_KUBERNETES_ENABLED:false}

        primaryCredentials:
          name: my-docker-registry-account
          address: ${SPINNAKER_DOCKER_REGISTRY:https://index.docker.io/ }
          repository: ${SPINNAKER_DOCKER_REPOSITORY:}
          username: ${SPINNAKER_DOCKER_USERNAME:}
          # A path to a plain text file containing the user's password
          passwordFile: ${SPINNAKER_DOCKER_PASSWORD_FILE:}

      openstack:
        # This default configuration uses the same environment variable names set in
        # the OpenStack RC file. See
        # http://docs.openstack.org/user-guide/common/cli-set-environment-variables-using-openstack-rc.html
        # for details on the OpenStack RC file.
        enabled: false
        defaultRegion: ${SPINNAKER_OPENSTACK_DEFAULT_REGION:RegionOne}
        primaryCredentials:
          name: my-openstack-account
          authUrl: ${OS_AUTH_URL}
          username: ${OS_USERNAME}
          password: ${OS_PASSWORD}
          projectName: ${OS_PROJECT_NAME}
          domainName: ${OS_USER_DOMAIN_NAME:Default}
          regions: ${OS_REGION_NAME:RegionOne}
          insecure: false
{% endcode %}
<!-- endtab -->
<!-- tab ConfigMap3 -->
vi /data/k8s-yaml/armory/clouddriver/custom-config.yaml
{% code %}
kind: ConfigMap
apiVersion: v1
metadata:
  name: custom-config
  namespace: armory
data:
  clouddriver-local.yml: |
    kubernetes:
      enabled: true
      accounts:
        - name: cluster-admin
          serviceAccount: false
          dockerRegistries:
            - accountName: harbor
              namespace: []
          namespaces:
            - test
            - prod
          kubeconfigFile: /opt/spinnaker/credentials/custom/default-kubeconfig
      primaryAccount: cluster-admin
    dockerRegistry:
      enabled: true
      accounts:
        - name: harbor
          requiredGroupMembership: []
          providerVersion: V1
          insecureRegistry: true
          address: http://harbor.od.com
          username: admin
          password: Harbor12345
      primaryAccount: harbor
    artifacts:
      s3:
        enabled: true
        accounts:
        - name: armory-config-s3-account
          apiEndpoint: http://minio
          apiRegion: us-east-1
      gcs:
        enabled: false
        accounts:
        - name: armory-config-gcs-account
  custom-config.json: ""
  echo-configurator.yml: |
    diagnostics:
      enabled: true
  front50-local.yml: |
    spinnaker:
      s3:
        endpoint: http://minio
  igor-local.yml: |
    jenkins:
      enabled: true
      masters:
        - name: jenkins-admin
          address: http://jenkins.od.com
          username: admin
          password: admin123
      primaryAccount: jenkins-admin
  nginx.conf: |
    gzip on;
    gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript application/vnd.ms-fontobject application/x-font-ttf font/opentype image/svg+xml image/x-icon;

    server {
           listen 80;

           location / {
                proxy_pass http://armory-deck/;
           }

           location /api/ {
                proxy_pass http://armory-gate:8084/;
           }

           rewrite ^/login(.*)$ /api/login$1 last;
           rewrite ^/auth(.*)$ /api/auth$1 last;
    }
  spinnaker-local.yml: |
    services:
      igor:
        enabled: true
{% endcode %}
<!-- endtab -->
<!-- tab Deployment -->
vi /data/k8s-yaml/armory/clouddriver/dp.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: armory-clouddriver
  name: armory-clouddriver
  namespace: armory
spec:
  replicas: 1
  revisionHistoryLimit: 7
  selector:
    matchLabels:
      app: armory-clouddriver
  template:
    metadata:
      annotations:
        artifact.spinnaker.io/location: '"armory"'
        artifact.spinnaker.io/name: '"armory-clouddriver"'
        artifact.spinnaker.io/type: '"kubernetes/deployment"'
        moniker.spinnaker.io/application: '"armory"'
        moniker.spinnaker.io/cluster: '"clouddriver"'
      labels:
        app: armory-clouddriver
    spec:
      containers:
      - name: armory-clouddriver
        image: harbor.od.com/armory/clouddriver:v1.8.x
        imagePullPolicy: IfNotPresent
        command:
        - bash
        - -c
        args:
        - bash /opt/spinnaker/config/default/fetch.sh && cd /home/spinnaker/config
          && /opt/clouddriver/bin/clouddriver
        ports:
        - containerPort: 7002
          protocol: TCP
        env:
        - name: JAVA_OPTS
          value: -Xmx2000M
        envFrom:
        - configMapRef:
            name: init-env
        livenessProbe:
          failureThreshold: 5
          httpGet:
            path: /health
            port: 7002
            scheme: HTTP
          initialDelaySeconds: 600
          periodSeconds: 3
          successThreshold: 1
          timeoutSeconds: 1
        readinessProbe:
          failureThreshold: 5
          httpGet:
            path: /health
            port: 7002
            scheme: HTTP
          initialDelaySeconds: 180
          periodSeconds: 3
          successThreshold: 5
          timeoutSeconds: 1
        securityContext: 
          runAsUser: 0
        volumeMounts:
        - mountPath: /etc/podinfo
          name: podinfo
        - mountPath: /home/spinnaker/.aws
          name: credentials
        - mountPath: /opt/spinnaker/credentials/custom
          name: default-kubeconfig
        - mountPath: /opt/spinnaker/config/default
          name: default-config
        - mountPath: /opt/spinnaker/config/custom
          name: custom-config
      imagePullSecrets:
      - name: harbor
      volumes:
      - configMap:
          defaultMode: 420
          name: default-kubeconfig
        name: default-kubeconfig
      - configMap:
          defaultMode: 420
          name: custom-config
        name: custom-config
      - configMap:
          defaultMode: 420
          name: default-config
        name: default-config
      - name: credentials
        secret:
          defaultMode: 420
          secretName: credentials
      - downwardAPI:
          defaultMode: 420
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.labels
            path: labels
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.annotations
            path: annotations
        name: podinfo
{% endcode %}
<!-- endtab -->
<!-- tab Service -->
vi /data/k8s-yaml/armory/clouddriver/svc.yaml
{% code %}
apiVersion: v1
kind: Service
metadata:
  name: armory-clouddriver
  namespace: armory
spec:
  ports:
  - port: 7002
    protocol: TCP
    targetPort: 7002
  selector:
    app: armory-clouddriver
{% endcode %}
<!-- endtab -->
{% endtabs %}

## 应用资源配置清单
任意一台运算节点上：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/armory/clouddriver/init-env.yaml 
configmap/init-env created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/armory/clouddriver/default-config.yaml
configmap/default-config created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/armory/clouddriver/custom-config.yaml
configmap/custom-config created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/armory/clouddriver/dp.yaml 
deployment.extensions/armory-clouddriver created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/armory/clouddriver/svc.yaml 
service/armory-clouddriver created
```

# 部署对象存储管理组件——Front50
运维主机`HDSS7-200.host.com`上：
## 准备docker镜像
[镜像下载地址](https://hub.docker.io/armory/spinnaker-front50-slim)
```
[root@hdss7-200 ~]# docker pull docker.io/armory/spinnaker-front50-slim:release-1.8.x-93febf2
0.15.0-20190123154713: Pulling from armory/spinnaker-front50-slim

cd784148e348: Already exists 
1a5149a464dd: Pull complete 
26bf3384364c: Pull complete 
69dd8d165987: Pull complete 
32fe51b2b4d9: Pull complete 
Digest: sha256:c8ef16b8af2d19d600cca5e8cc23804bd2c1d5514388773a10b6066f94618a72
Status: Downloaded newer image for armory/spinnaker-front50-slim:release-1.8.x-93febf2
[root@hdss7-200 ~]# docker tag 0d353788f4f2 harbor.od.com/armory/front50:v1.8.x
[root@hdss7-200 ~]# docker push harbor.od.com/spinnaker/front50:v0.15.0
The push refers to a repository [harbor.od.com/armory/front50]
dfaf560918e4: Pushed 
44956a013f38: Pushed 
78bd58e6921a: Pushed 
12c374f8270a: Pushed 
0c3170905795: Pushed
df64d3292fd6: Pushed  
v1.8.x: digest: sha256:c8b66fbe74f5e2ca09ecea5232826790bc4715dd0147144de2bdc4b9656aa185 size: 1579
```

## 准备资源配置清单
{% tabs armory-front50 %}
<!-- tab Deployment -->
vi /data/k8s-yaml/armory/front50/dp.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: armory-front50
  name: armory-front50
  namespace: armory
spec:
  replicas: 1
  revisionHistoryLimit: 7
  selector:
    matchLabels:
      app: armory-front50
  template:
    metadata:
      annotations:
        artifact.spinnaker.io/location: '"armory"'
        artifact.spinnaker.io/name: '"armory-front50"'
        artifact.spinnaker.io/type: '"kubernetes/deployment"'
        moniker.spinnaker.io/application: '"armory"'
        moniker.spinnaker.io/cluster: '"front50"'
      labels:
        app: armory-front50
    spec:
      containers:
      - name: armory-front50
        image: harbor.od.com/armory/front50:v1.8.x
        imagePullPolicy: IfNotPresent
        command:
        - bash
        - -c
        args:
        - bash /opt/spinnaker/config/default/fetch.sh && cd /home/spinnaker/config
          && /opt/front50/bin/front50
        ports:
        - containerPort: 8080
          protocol: TCP
        env:
        - name: JAVA_OPTS
          value: -javaagent:/opt/front50/lib/jamm-0.2.5.jar -Xmx1000M
        envFrom:
        - configMapRef:
            name: init-env
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 600
          periodSeconds: 3
          successThreshold: 1
          timeoutSeconds: 1
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 180
          periodSeconds: 5
          successThreshold: 8
          timeoutSeconds: 1
        volumeMounts:
        - mountPath: /etc/podinfo
          name: podinfo
        - mountPath: /home/spinnaker/.aws
          name: credentials
        - mountPath: /opt/spinnaker/config/default
          name: default-config
        - mountPath: /opt/spinnaker/config/custom
          name: custom-config
      imagePullSecrets:
      - name: harbor
      volumes:
      - configMap:
          defaultMode: 420
          name: custom-config
        name: custom-config
      - configMap:
          defaultMode: 420
          name: default-config
        name: default-config
      - name: credentials
        secret:
          defaultMode: 420
          secretName: credentials
      - downwardAPI:
          defaultMode: 420
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.labels
            path: labels
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.annotations
            path: annotations
        name: podinfo
{% endcode %}
<!-- endtab -->
<!-- tab Service -->
vi /data/k8s-yaml/armory/front50/svc.yaml
{% code %}
apiVersion: v1
kind: Service
metadata:
  name: armory-front50
  namespace: armory
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: armory-front50
{% endcode %}
<!-- endtab -->
{% endtabs %}

## 应用资源配置清单
任意一台运算节点上：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/armory/front50/dp.yaml 
deployment.extensions/armory-front50 created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/armory/front50/svc.yaml 
service/armory-front50 created
```

## 浏览器访问
http://minio.od.com
登录并观察存储是否创建

# 部署资源编排组件——Orca
运维主机`HDSS7-200.host.com`上：
## 准备docker镜像
[镜像下载地址](https://hub.docker.io/armory/spinnaker-orca-slim:release-1.8.x-de4ab55)
```
[root@hdss7-200 ~]# docker pull docker.io/armory/spinnaker-orca-slim:release-1.8.x-de4ab55
release-1.8.x-de4ab55: Pulling from armory/spinnaker-orca-slim

4fe2ade4980c: Already exists 
6fc58a8d4ae4: Already exists 
6c70af887bc7: Pull complete 
c4b6e637d6e8: Pull complete 
da01b2afaa26: Pull complete 
Digest: sha256:aeb403299da62e26c018d5ff3ce7ba20a6a92d9dc0c48fa16edef37e20316bb3
Status: Downloaded newer image for armory/spinnaker-orca-slim:release-1.8.x-de4ab55
[root@hdss7-200 ~]# docker tag 5103b1f73e04 harbor.od.com/armory/orca:v1.8.x
[root@hdss7-200 ~]# docker push harbor.od.com/spinnaker/orca:v2.3.0
The push refers to a repository [harbor.od.com/armory/orca]
fc691dbda20f: Pushed 
df3bd4d73885: Pushed 
c5165988c0bd: Pushed 
12c374f8270a: Mounted from armory/front50 
0c3170905795: Mounted from armory/front50 
df64d3292fd6: Mounted from armory/front50 
v1.8.x: digest: sha256:337900b721558115c748255ffb2ecf471805cfe7bc5cfe93c52ac90482f2eff4 size: 1578
```

## 准备资源配置清单
{% tabs armory-orca %}
<!-- tab Deployment -->
vi /data/k8s-yaml/armory/orca/dp.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: armory-orca
  name: armory-orca
  namespace: armory
spec:
  replicas: 1
  revisionHistoryLimit: 7
  selector:
    matchLabels:
      app: armory-orca
  template:
    metadata:
      annotations:
        artifact.spinnaker.io/location: '"armory"'
        artifact.spinnaker.io/name: '"armory-orca"'
        artifact.spinnaker.io/type: '"kubernetes/deployment"'
        moniker.spinnaker.io/application: '"armory"'
        moniker.spinnaker.io/cluster: '"orca"'
      labels:
        app: armory-orca
    spec:
      containers:
      - name: armory-orca
        image: harbor.od.com/armory/orca:v1.8.x
        imagePullPolicy: IfNotPresent
        command:
        - bash
        - -c
        args:
        - bash /opt/spinnaker/config/default/fetch.sh && cd /home/spinnaker/config
          && /opt/orca/bin/orca
        ports:
        - containerPort: 8083
          protocol: TCP
        env:
        - name: JAVA_OPTS
          value: -Xmx1000M
        envFrom:
        - configMapRef:
            name: init-env
        livenessProbe:
          failureThreshold: 5
          httpGet:
            path: /health
            port: 8083
            scheme: HTTP
          initialDelaySeconds: 600
          periodSeconds: 5
          successThreshold: 1
          timeoutSeconds: 1
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /health
            port: 8083
            scheme: HTTP
          initialDelaySeconds: 180
          periodSeconds: 3
          successThreshold: 5
          timeoutSeconds: 1
        volumeMounts:
        - mountPath: /etc/podinfo
          name: podinfo
        - mountPath: /opt/spinnaker/config/default
          name: default-config
        - mountPath: /opt/spinnaker/config/custom
          name: custom-config
      imagePullSecrets:
      - name: harbor
      volumes:
      - configMap:
          defaultMode: 420
          name: custom-config
        name: custom-config
      - configMap:
          defaultMode: 420
          name: default-config
        name: default-config
      - downwardAPI:
          defaultMode: 420
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.labels
            path: labels
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.annotations
            path: annotations
        name: podinfo
{% endcode %}
<!-- endtab -->
<!-- tab Service -->
vi /data/k8s-yaml/armory/orca/svc.yaml
{% code %}
apiVersion: v1
kind: Service
metadata:
  name: armory-orca
  namespace: armory
spec:
  ports:
  - port: 8083
    protocol: TCP
    targetPort: 8083
  selector:
    app: armory-orca
{% endcode %}
<!-- endtab -->
{% endtabs %}

## 应用资源配置清单
任意一台运算节点上：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/armory/orca/dp.yaml 
deployment.extensions/armory-orca created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/armory/orca/svc.yaml 
service/armory-orca created
```
# 部署消息调度组件——Echo
运维主机`HDSS7-200.host.com`上：
## 准备docker镜像
[镜像下载地址](https://hub.docker.io/armory/echo-armory)
```
[root@hdss7-200 ~]# docker pull docker.io/armory/echo-armory:c36d576-release-1.8.x-617c567
c36d576-release-1.8.x-617c567: Pulling from armory/echo-armory
12a7970a6783: Pull complete 
38a1c0aaa6fd: Pull complete 
fb7693893388: Pull complete 
c29ed2a5f1b6: Pull complete 
69c3c33e23e9: Pull complete 
5662e03596af: Pull complete 
Digest: sha256:33f6d25aa536d245bc1181a9d6f42eceb8ce59c9daa954fa9e4a64095acf8356
Status: Downloaded newer image for armory/echo-armory:c36d576-release-1.8.x-617c567
[root@hdss7-200 ~]# docker tag 415efd46f474 harbor.od.com/armory/echo:v1.8.x
[root@hdss7-200 ~]# docker push harbor.od.com/armory/echo:v1.8.x
The push refers to a repository [harbor.od.com/armory/echo]
1800ebd4bcda: Pushed 
e1f2ca83d794: Pushed 
7f4fe63acda3: Pushed 
20dd87a4c2ab: Pushed 
78075328e0da: Pushed 
9f8566ee5135: Pushed 
v1.8.x: digest: sha256:3972fe52095a8f68f4f872ee56ff120b22c660a3cd3a8d9e7a824e3ef2b73e37 size: 1579
```

## 准备资源配置清单
{% tabs armory-echo %}
<!-- tab Deployment -->
vi /data/k8s-yaml/armory/echo/dp.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: armory-echo
  name: armory-echo
  namespace: armory
spec:
  replicas: 1
  revisionHistoryLimit: 7
  selector:
    matchLabels:
      app: armory-echo
  template:
    metadata:
      annotations:
        artifact.spinnaker.io/location: '"armory"'
        artifact.spinnaker.io/name: '"armory-echo"'
        artifact.spinnaker.io/type: '"kubernetes/deployment"'
        moniker.spinnaker.io/application: '"armory"'
        moniker.spinnaker.io/cluster: '"echo"'
      labels:
        app: armory-echo
    spec:
      containers:
      - name: armory-echo
        image: harbor.od.com/armory/echo:v1.8.x
        imagePullPolicy: IfNotPresent
        command:
        - bash
        - -c
        args:
        - bash /opt/spinnaker/config/default/fetch.sh && cd /home/spinnaker/config
          && /opt/echo/bin/echo
        ports:
        - containerPort: 8089
          protocol: TCP
        env:
        - name: JAVA_OPTS
          value: -javaagent:/opt/echo/lib/jamm-0.2.5.jar -Xmx1000M
        envFrom:
        - configMapRef:
            name: init-env
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /health
            port: 8089
            scheme: HTTP
          initialDelaySeconds: 600
          periodSeconds: 3
          successThreshold: 1
          timeoutSeconds: 1
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /health
            port: 8089
            scheme: HTTP
          initialDelaySeconds: 180
          periodSeconds: 3
          successThreshold: 5
          timeoutSeconds: 1
        volumeMounts:
        - mountPath: /etc/podinfo
          name: podinfo
        - mountPath: /opt/spinnaker/config/default
          name: default-config
        - mountPath: /opt/spinnaker/config/custom
          name: custom-config
      imagePullSecrets:
      - name: harbor
      volumes:
      - configMap:
          defaultMode: 420
          name: custom-config
        name: custom-config
      - configMap:
          defaultMode: 420
          name: default-config
        name: default-config
      - downwardAPI:
          defaultMode: 420
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.labels
            path: labels
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.annotations
            path: annotations
        name: podinfo
{% endcode %}
<!-- endtab -->
<!-- tab Service -->
vi /data/k8s-yaml/armory/echo/svc.yaml
{% code %}
apiVersion: v1
kind: Service
metadata:
  name: armory-echo
  namespace: armory
spec:
  ports:
  - port: 8089
    protocol: TCP
    targetPort: 8089
  selector:
    app: armory-echo
{% endcode %}
<!-- endtab -->
{% endtabs %}

## 应用资源配置清单
任意一台运算节点上：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/armory/echo/dp.yaml 
deployment.extensions/armory-echo created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/armory/echo/svc.yaml 
service/armory-echo created
```

# 部署流水线交互组件——Igor
运维主机`HDSS7-200.host.com`上：
## 准备docker镜像
[镜像下载地址](https://hub.docker.io/armory/spinnaker-igor-slim)
```
[root@hdss7-200 ~]# docker pull docker.io/armory/spinnaker-igor-slim:release-1.8-x-new-install-healthy-ae2b329
release-1.8-x-new-install-healthy-ae2b329: Pulling from armory/spinnaker-igor-slim

a8906544047d: Pull complete 
590b87a38029: Pull complete 
246dc9ee5476: Pull complete 
04efcaf9f873: Pull complete 
da906be06326: Pull complete 
Digest: sha256:2a487385908647f24ffa6cd11071ad571bec717008b7f16bc470ba754a7ad258
Status: Downloaded newer image for armory/spinnaker-igor-slim:release-1.8-x-new-install-healthy-ae2b329
[root@hdss7-200 ~]# docker tag 23984f5b43f6 harbor.od.com/armory/igor:v1.8.x
[root@hdss7-200 ~]# docker push harbor.od.com/spinnaker/igor:v1.1.0
The push refers to a repository [harbor.od.com/armory/igor]
86e21a74a10c: Pushed 
f18424cc1043: Pushed 
aa52633c7e64: Pushed 
8fcf61ed46a1: Pushed 
a8cc3712c14a: Pushed 
cd7100a72410: Mounted from spinnaker/clouddriver 
v1.8.x: digest: sha256:a12945b48ff428a2f1b5e58c9e1aba25ee1c4199f42f3f43910b9e009bc2cc94 size: 1578
```

## 准备资源配置清单
{% tabs armory-igor %}
<!-- tab Deployment -->
vi /data/k8s-yaml/armory/igor/dp.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: armory-igor
  name: armory-igor
  namespace: armory
spec:
  replicas: 1
  revisionHistoryLimit: 7
  selector:
    matchLabels:
      app: armory-igor
  template:
    metadata:
      annotations:
        artifact.spinnaker.io/location: '"armory"'
        artifact.spinnaker.io/name: '"armory-igor"'
        artifact.spinnaker.io/type: '"kubernetes/deployment"'
        moniker.spinnaker.io/application: '"armory"'
        moniker.spinnaker.io/cluster: '"igor"'
      labels:
        app: armory-igor
    spec:
      containers:
      - name: armory-igor
        image: harbor.od.com/armory/igor:v1.8.x
        imagePullPolicy: IfNotPresent
        command:
        - bash
        - -c
        args:
        - bash /opt/spinnaker/config/default/fetch.sh && cd /home/spinnaker/config
          && /opt/igor/bin/igor
        ports:
        - containerPort: 8088
          protocol: TCP
        env:
        - name: IGOR_PORT_MAPPING
          value: -8088:8088
        - name: JAVA_OPTS
          value: -Xmx1000M
        envFrom:
        - configMapRef:
            name: init-env
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /health
            port: 8088
            scheme: HTTP
          initialDelaySeconds: 600
          periodSeconds: 3
          successThreshold: 1
          timeoutSeconds: 1
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /health
            port: 8088
            scheme: HTTP
          initialDelaySeconds: 180
          periodSeconds: 5
          successThreshold: 5
          timeoutSeconds: 1
        volumeMounts:
        - mountPath: /etc/podinfo
          name: podinfo
        - mountPath: /opt/spinnaker/config/default
          name: default-config
        - mountPath: /opt/spinnaker/config/custom
          name: custom-config
      imagePullSecrets:
      - name: harbor
      securityContext:
        runAsUser: 0
      volumes:
      - configMap:
          defaultMode: 420
          name: custom-config
        name: custom-config
      - configMap:
          defaultMode: 420
          name: default-config
        name: default-config
      - downwardAPI:
          defaultMode: 420
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.labels
            path: labels
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.annotations
            path: annotations
        name: podinfo
{% endcode %}
<!-- endtab -->
<!-- tab Service -->
vi /data/k8s-yaml/armory/igor/svc.yaml
{% code %}
apiVersion: v1
kind: Service
metadata:
  name: armory-igor
  namespace: armory
spec:
  ports:
  - port: 8088
    protocol: TCP
    targetPort: 8088
  selector:
    app: armory-igor
{% endcode %}
<!-- endtab -->
{% endtabs %}

## 应用资源配置清单
任意一台运算节点上：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/armory/igor/dp.yaml 
deployment.extensions/armory-igor created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/armory/igor/svc.yaml 
service/armory-igor created
```

# 部署API组件——Gate
运维主机`HDSS7-200.host.com`上：
## 准备docker镜像
[镜像下载地址](https://hub.docker.io/armory/gate-armory)
```
[root@hdss7-200 ~]# docker pull docker.io/armory/gate-armory:dfafe73-release-1.8.x-5d505ca
dfafe73-release-1.8.x-5d505ca: Pulling from armory/gate-armory

12a7970a6783: Already exists 
38a1c0aaa6fd: Already exists 
f6e81adf2fc6: Pull complete 
d81889909516: Pull complete 
804d515d9470: Pull complete 
Digest: sha256:e3ea88c29023bce211a1b0772cc6cb631f3db45c81a4c0394c4fc9999a417c1f
Status: Downloaded newer image for armory/gate-armory:dfafe73-release-1.8.x-5d505ca
[root@hdss7-200 ~]# docker tag b092d4665301 harbor.od.com/armory/gate:v1.8.x
[root@hdss7-200 ~]# docker push harbor.od.com/armory/gate:v1.8.x
The push refers to a repository [harbor.od.com/armory/gate]
6d2409165e36: Pushed 
25907606e564: Pushed 
f75150ad7e46: Pushed 
20dd87a4c2ab: Mounted from armory/echo 
78075328e0da: Mounted from armory/echo 
9f8566ee5135: Mounted from armory/echo 
v1.8.x: digest: sha256:efbe278ea128806b6fe620e7b4e4d7441ca7dd94a7fa98760ffa086625b8d5f3 size: 1577
```

## 准备资源配置清单
{% tabs armory-gate %}
<!-- tab Deployment -->
vi /data/k8s-yaml/armory/gate/dp.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: armory-gate
  name: armory-gate
  namespace: armory
spec:
  replicas: 1
  revisionHistoryLimit: 7
  selector:
    matchLabels:
      app: armory-gate
  template:
    metadata:
      annotations:
        artifact.spinnaker.io/location: '"armory"'
        artifact.spinnaker.io/name: '"armory-gate"'
        artifact.spinnaker.io/type: '"kubernetes/deployment"'
        moniker.spinnaker.io/application: '"armory"'
        moniker.spinnaker.io/cluster: '"gate"'
      labels:
        app: armory-gate
    spec:
      containers:
      - name: armory-gate
        image: harbor.od.com/armory/gate:v1.8.x
        imagePullPolicy: IfNotPresent
        command:
        - bash
        - -c
        args:
        - bash /opt/spinnaker/config/default/fetch.sh gate && cd /home/spinnaker/config
          && /opt/gate/bin/gate
        ports:
        - containerPort: 8084
          name: gate-port
          protocol: TCP
        - containerPort: 8085
          name: gate-api-port
          protocol: TCP
        env:
        - name: GATE_PORT_MAPPING
          value: -8084:8084
        - name: GATE_API_PORT_MAPPING
          value: -8085:8085
        - name: JAVA_OPTS
          value: -Xmx1000M
        envFrom:
        - configMapRef:
            name: init-env
        livenessProbe:
          exec:
            command:
            - /bin/bash
            - -c
            - wget -O - http://localhost:8084/health || wget -O - https://localhost:8084/health
          failureThreshold: 5
          initialDelaySeconds: 600
          periodSeconds: 5
          successThreshold: 1
          timeoutSeconds: 1
        readinessProbe:
          exec:
            command:
            - /bin/bash
            - -c
            - wget -O - http://localhost:8084/health?checkDownstreamServices=true&downstreamServices=true
              || wget -O - https://localhost:8084/health?checkDownstreamServices=true&downstreamServices=true
          failureThreshold: 3
          initialDelaySeconds: 180
          periodSeconds: 5
          successThreshold: 10
          timeoutSeconds: 1
        volumeMounts:
        - mountPath: /etc/podinfo
          name: podinfo
        - mountPath: /opt/spinnaker/config/default
          name: default-config
        - mountPath: /opt/spinnaker/config/custom
          name: custom-config
      imagePullSecrets:
      - name: harbor
      securityContext:
        runAsUser: 0
      volumes:
      - configMap:
          defaultMode: 420
          name: custom-config
        name: custom-config
      - configMap:
          defaultMode: 420
          name: default-config
        name: default-config
      - downwardAPI:
          defaultMode: 420
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.labels
            path: labels
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.annotations
            path: annotations
        name: podinfo
{% endcode %}
<!-- endtab -->
<!-- tab Service -->
vi /data/k8s-yaml/armory/gate/svc.yaml
{% code %}
apiVersion: v1
kind: Service
metadata:
  name: armory-gate
  namespace: armory
spec:
  ports:
  - name: gate-port
    port: 8084
    protocol: TCP
    targetPort: 8084
  - name: gate-api-port
    port: 8085
    protocol: TCP
    targetPort: 8085
  selector:
    app: armory-gate
{% endcode %}
<!-- endtab -->
{% endtabs %}

## 应用资源配置清单
任意一台运算节点上：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/armory/gate/dp.yaml 
deployment.extensions/armory-gate created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/armory/gate/svc.yaml 
service/armory-gate created
```

#  部署前端网站项目——Deck
运维主机`HDSS7-200.host.com`上：
## 准备docker镜像
[镜像下载地址](https://hub.docker.io/armory/deck-armory)
```
[root@hdss7-200 ~]# docker pull docker.io/armory/deck-armory:d4bf0cf-release-1.8.x-0a33f94
d4bf0cf-release-1.8.x-0a33f94: Pulling from armory/deck-armory
cf0a75889057: Pull complete 
c8de9902faf0: Pull complete 
a3c0f7711c5e: Pull complete 
e6391432e12c: Pull complete 
624ce029a17f: Pull complete 
c7b18362c9ae: Pull complete 
7b9914ffa0d3: Pull complete 
f212d95417f6: Pull complete 
e7de6608a940: Pull complete 
532b97612378: Pull complete 
b15dbf201531: Pull complete 
ca28775a6e92: Pull complete 
Digest: sha256:ad85eb8e1ada327ab0b98471d10ed2a4e5eada3c154a2f17b6b23a089c74839f
Status: Downloaded newer image for armory/deck-armory:d4bf0cf-release-1.8.x-0a33f94
[root@hdss7-200 ~]# docker tag 9a87ba3b319f harbor.od.com/armory/deck:v1.8.x
[root@hdss7-200 ~]# docker push harbor.od.com/armory/deck:v1.8.x
The push refers to a repository [harbor.od.com/armory/deck]
7a83eddefe6b: Pushed 
28a427580c04: Pushed 
0371824b6e1b: Pushed 
1338fd1b3201: Pushed 
14e3d0d46256: Pushed 
c426a599a7c7: Pushed 
b9e54f6f12bd: Pushed 
776d5289b76e: Pushed 
0fb55a72eab2: Pushed 
a30ab2bcda94: Pushed 
99840408c5ea: Pushed 
a8e78858b03b: Pushed 
v1.8.x: digest: sha256:299b6a10fbca18beda4c7d71472be3daf390b2d45447f43682879178fa2b2c19 size: 2822
```

## 准备资源配置清单
{% tabs armory-deck %}
<!-- tab Deployment -->
vi /data/k8s-yaml/armory/deck/dp.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: armory-deck
  name: armory-deck
  namespace: armory
spec:
  replicas: 1
  revisionHistoryLimit: 7
  selector:
    matchLabels:
      app: armory-deck
  template:
    metadata:
      annotations:
        artifact.spinnaker.io/location: '"armory"'
        artifact.spinnaker.io/name: '"armory-deck"'
        artifact.spinnaker.io/type: '"kubernetes/deployment"'
        moniker.spinnaker.io/application: '"armory"'
        moniker.spinnaker.io/cluster: '"deck"'
      labels:
        app: armory-deck
    spec:
      containers:
      - name: armory-deck
        image: harbor.od.com/armory/deck:v1.8.x
        imagePullPolicy: IfNotPresent
        command:
        - bash
        - -c
        args:
        - bash /opt/spinnaker/config/default/fetch.sh && /entrypoint.sh
        ports:
        - containerPort: 9000
          protocol: TCP
        envFrom:
        - configMapRef:
            name: init-env
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /
            port: 9000
            scheme: HTTP
          initialDelaySeconds: 180
          periodSeconds: 3
          successThreshold: 1
          timeoutSeconds: 1
        readinessProbe:
          failureThreshold: 5
          httpGet:
            path: /
            port: 9000
            scheme: HTTP
          initialDelaySeconds: 30
          periodSeconds: 3
          successThreshold: 5
          timeoutSeconds: 1
        volumeMounts:
        - mountPath: /etc/podinfo
          name: podinfo
        - mountPath: /opt/spinnaker/config/default
          name: default-config
        - mountPath: /opt/spinnaker/config/custom
          name: custom-config
      imagePullSecrets:
      - name: harbor
      volumes:
      - configMap:
          defaultMode: 420
          name: custom-config
        name: custom-config
      - configMap:
          defaultMode: 420
          name: default-config
        name: default-config
      - downwardAPI:
          defaultMode: 420
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.labels
            path: labels
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.annotations
            path: annotations
        name: podinfo
{% endcode %}
<!-- endtab -->
<!-- tab Service -->
vi /data/k8s-yaml/armory/deck/svc.yaml
{% code %}
apiVersion: v1
kind: Service
metadata:
  name: armory-deck
  namespace: armory
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 9000
  selector:
    app: armory-deck
{% endcode %}
<!-- endtab -->
{% endtabs %}

## 应用资源配置清单
任意一台运算节点上：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/armory/deck/dp.yaml 
deployment.extensions/armory-deck created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/armory/deck/svc.yaml 
service/armory-deck created
```

#  部署前端代理——Nginx
运维主机`HDSS7-200.host.com`上：
## 准备docker镜像
[镜像下载地址](https://hub.docker.io/library/nginx)
```
[root@hdss7-200 ~]# docker pull nginx:1.12.2
Using default tag: 1.12.2
latest: Pulling from library/nginx
fc7181108d40: Pull complete 
c4277fc40ec2: Pull complete 
780053e98559: Pull complete 
Digest: sha256:bdbf36b7f1f77ffe7bd2a32e59235dff6ecf131e3b6b5b96061c652f30685f3a
Status: Downloaded newer image for nginx:1.12.2
[root@hdss7-200 ~]# docker tag 719cd2e3ed04 harbor.od.com/armory/nginx:v1.12.2
[root@hdss7-200 ~]# docker push harbor.od.com/armory/nginx:v1.12.2
The push refers to a repository [harbor.od.com/armory/nginx]
d7acf794921f: Pushed 
d9569ca04881: Pushed 
cf5b3c6798f7: Pushed 
v1.12.2: digest: sha256:671d8dbb5fc0e03626167acaa95cf21439d34761d110e081a0be98aa53513776 size: 948
```

## 准备资源配置清单
{% tabs armory-nginx %}
<!-- tab Deployment -->
vi /data/k8s-yaml/armory/nginx/dp.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: armory-nginx
  name: armory-nginx
  namespace: armory
spec:
  replicas: 1
  revisionHistoryLimit: 7
  selector:
    matchLabels:
      app: armory-nginx
  template:
    metadata:
      annotations:
        artifact.spinnaker.io/location: '"armory"'
        artifact.spinnaker.io/name: '"armory-nginx"'
        artifact.spinnaker.io/type: '"kubernetes/deployment"'
        moniker.spinnaker.io/application: '"armory"'
        moniker.spinnaker.io/cluster: '"nginx"'
      labels:
        app: armory-nginx
    spec:
      containers:
      - name: armory-nginx
        image: harbor.od.com/armory/nginx:v1.12.2
        imagePullPolicy: Always
        command:
        - bash
        - -c
        args:
        - bash /opt/spinnaker/config/default/fetch.sh nginx && nginx -g 'daemon off;'
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
        - containerPort: 443
          name: https
          protocol: TCP
        - containerPort: 8085
          name: api
          protocol: TCP
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /
            port: 80
            scheme: HTTP
          initialDelaySeconds: 180
          periodSeconds: 3
          successThreshold: 1
          timeoutSeconds: 1
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /
            port: 80
            scheme: HTTP
          initialDelaySeconds: 30
          periodSeconds: 3
          successThreshold: 5
          timeoutSeconds: 1
        volumeMounts:
        - mountPath: /opt/spinnaker/config/default
          name: default-config
        - mountPath: /etc/nginx/conf.d
          name: custom-config
      imagePullSecrets:
      - name: harbor
      volumes:
      - configMap:
          defaultMode: 420
          name: custom-config
        name: custom-config
      - configMap:
          defaultMode: 420
          name: default-config
        name: default-config
{% endcode %}
<!-- endtab -->
<!-- tab Service -->
vi /data/k8s-yaml/armory/nginx/svc.yaml
{% code %}
apiVersion: v1
kind: Service
metadata:
  name: armory-nginx
  namespace: armory
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  - name: https
    port: 443
    protocol: TCP
    targetPort: 443
  - name: api
    port: 8085
    protocol: TCP
    targetPort: 8085
  selector:
    app: armory-nginx
{% endcode %}
<!-- endtab -->
<!-- tab Ingress -->
vi /data/k8s-yaml/armory/nginx/ingress.yaml
{% code %}
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  labels:
    app: spinnaker
    web: spinnaker.od.com
  name: armory-nginx
  namespace: armory
spec:
  rules:
  - host: spinnaker.od.com
    http:
      paths:
      - backend:
          serviceName: armory-nginx
          servicePort: 80
{% endcode %}
<!-- endtab -->
{% endtabs %}

## 应用资源配置清单
任意一台运算节点上：
```
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/armory/nginx/dp.yaml 
deployment.extensions/armory-nginx created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/armory/nginx/svc.yaml 
service/armory-nginx created
[root@hdss7-21 ~]# kubectl apply -f http://k8s-yaml.od.com/armory/nginx/ingress.yaml 
ingress.extensions/armory-nginx created
```