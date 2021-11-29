---
title: zadig-kubernetes-deploy
date: 2021-10-26 11:02:34
tags:
- kubernetes
- ci
---



# 前言

公众号看到zadig非常强大，可以集成jenkins, gitlab, kubernetes, ...



<!--more-->
# 部署

https://docs.koderover.com/zadig/quick-start/introduction/#%E4%B8%9A%E5%8A%A1%E6%9E%B6%E6%9E%84





# yaml

## zadig的yaml

`zadig.yaml`

```bash
helm template zadig --namespace zadig ./ --set endpoint.FQDN=zadig.mykernel.cn --set tags.ingressController=false > zadig-domain.yaml
```

```yaml
---
# Source: zadig/charts/minio/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: zadig-minio
  namespace: zadig
  labels:
    app.kubernetes.io/name: minio
    helm.sh/chart: minio-7.0.0
    app.kubernetes.io/instance: zadig
    app.kubernetes.io/managed-by: Helm
secrets:
  - name: zadig-minio
---
# Source: zadig/charts/mongodb/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: zadig-mongodb
  namespace: zadig
  labels:
    app.kubernetes.io/name: mongodb
    helm.sh/chart: mongodb-10.20.0
    app.kubernetes.io/instance: zadig
    app.kubernetes.io/managed-by: Helm
secrets:
  - name: zadig-mongodb
---
# Source: zadig/templates/rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-zadig
---
# Source: zadig/charts/minio/templates/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: zadig-minio
  namespace: zadig
  labels:
    app.kubernetes.io/name: minio
    helm.sh/chart: minio-7.0.0
    app.kubernetes.io/instance: zadig
    app.kubernetes.io/managed-by: Helm
type: Opaque
data:
  access-key: "QUtJQUlPU0ZPRE5ONzIwMTlFWEFNUExF"
  secret-key: "d0phbHJYVXRuRkVNSTIwMTlLN01ERU5HYlB4UmZpQ1lFWEFNUExFS0VZ"
  key.json: ""
---
# Source: zadig/templates/encryption_secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: zadig-aes-key
type: Opaque
data:
  aesKey: "OUYxMUI0RTUwM0M3RjJCNTc3RTVGOTM2NkJEREFCNjQ="
---
# Source: zadig/templates/pull-secret.yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/dockerconfigjson
metadata:
  name: qn-registry-secret
  labels:
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
data:
  .dockerconfigjson: "eyJhdXRocyI6eyJjY3IuY2NzLnRlbmNlbnR5dW4uY29tIjp7ImF1dGgiOiJNVEF3TURFME9USTVPVFl6T201RGJXWlVRU3RaYW1oR01pcEROVW89In19fQ=="
---
# Source: zadig/templates/aslan-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aslan-config
  labels:
    app.kubernetes.io/component: aslan
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
data:
  # --------------------------------------------------------------------------------------
  # common
  # --------------------------------------------------------------------------------------
  ADDRESS: http://zadig.mykernel.cn
  ENTERPRISE: "false"

  # --------------------------------------------------------------------------------------
  # logging
  # level: 0(Debug), 1(Info), 2(Warn), 3(Error), 4(Panic), 5(Fatal)
  # --------------------------------------------------------------------------------------
  LOG_LEVEL: Info

  # --------------------------------------------------------------------------------------
  # mongo
  # --------------------------------------------------------------------------------------
  MONGODB_CONNECTION_STRING: mongodb://zadig-mongodb:27017
  ASLAN_DB: zadig

  # --------------------------------------------------------------------------------------
  # kube
  # --------------------------------------------------------------------------------------
  KUBE_SERVER_ADDR: ""

  # --------------------------------------------------------------------------------------
  # build
  # --------------------------------------------------------------------------------------
  NSQLOOKUP_ADDRS: nsqlookup-0.nsqlookupd:4161,nsqlookup-1.nsqlookupd:4161,nsqlookup-2.nsqlookupd:4161

  REAPER_IMAGE: ccr.ccs.tencentyun.com/koderover-rc/reaper-plugin:1.5.0-amd64
  REAPER_BINARY_FILE: http://resource-server/reaper
  PREDATOR_IMAGE:  ccr.ccs.tencentyun.com/koderover-rc/predator-plugin:1.5.0-amd64
  JENKINS_BUILD_IMAGE: ccr.ccs.tencentyun.com/koderover-rc/jenkins-plugin:1.5.0-amd64

  # --------------------------------------------------------------------------------------
  # github app
  # --------------------------------------------------------------------------------------
  GITHUB_KNOWN_HOST: ""
  GITHUB_SSH_KEY: ""

  # --------------------------------------------------------------------------------------
  # docker
  # --------------------------------------------------------------------------------------
  DOCKER_HOSTS: tcp://dind-0.dind:2375,tcp://dind-1.dind:2375,tcp://dind-2.dind:2375
  POETRY_API_ROOT_KEY: 9F11B4E503C7F2B5
  DEFAULT_INGRESS_CLASS: "zadig-nginx"

  # -------------------------------------------------------------------------------
  # UNKNOWN USE
  # -------------------------------------------------------------------------------
  USE_CLASSIC_BUILD: "false"
  CUSTOM_DNS_NOT_SUPPORTED: "false"
  OLD_ENV_SUPPORTED: "false"

  HUB_AGENT_IMAGE: ccr.ccs.tencentyun.com/koderover-rc/hub-agent:1.5.0-amd64

  SERVICE_START_TIMEOUT: "600"
  DEFAULT_REGISTRY: "https://ccr.ccs.tencentyun.com"
  DEFAULT_REGISTRY_AK: "100008469911"
  DEFAULT_REGISTRY_SK: "nCmfTA+YjhF2*C5J"
  DEFAULT_REGISTRY_NAMESPACE: "trial"

  S3STORAGE_ENDPOINT: zadig-minio:9000
  S3STORAGE_AK: AKIAIOSFODNN72019EXAMPLE
  S3STORAGE_SK: wJalrXUtnFEMI2019K7MDENGbPxRfiCYEXAMPLEKEY
  S3STORAGE_BUCKET: bucket
  S3STORAGE_PROTOCOL: http

  KODESPACE_VERSION: v1.1.0
---
# Source: zadig/templates/warpdrive-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: warpdrive-config
  labels:
    app.kubernetes.io/component: warpdrive
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
data:
  ADDRESS: http://zadig.mykernel.cn
  NSQLOOKUP_ADDRS: "nsqlookup-0.nsqlookupd:4161,nsqlookup-1.nsqlookupd:4161,nsqlookup-2.nsqlookupd:4161"
  DEFAULT_REG_ADDRESS: "https://ccr.ccs.tencentyun.com"
  DEFAULT_REG_ACCESS_KEY: "100008469911"
  DEFAULT_REG_SECRET_KEY: "nCmfTA+YjhF2*C5J"
  POETRY_API_ROOT_KEY: 9F11B4E503C7F2B5
  DISABLE_KUBE_INFORMER: "true"
---
# Source: zadig/templates/zadig-portal-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: zadig-portal-config
  labels:
    app.kubernetes.io/component: zadig-portal
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
data:
  default.conf: |-
    log_format  zadiglog  '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $bytes_sent $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for" '
                '$upstream_addr $host $sent_http_x_reqid $upstream_response_time $request_time';

    server {
      listen 80;

      # gzip
      gzip on;
      gzip_vary on;
      gzip_proxied any;
      gzip_comp_level 6;
      gzip_types text/plain text/css text/xml application/json application/javascript application/xml+rss application/atom+xml image/svg+xml;
      root /zadig-portal/;

      location @rewrites {
        rewrite ^(.+)$ /index.html last;
      }

      # 缓存静态文件
      location ^~ /static/ {
        access_log off;
        add_header Access-Control-Allow-Origin *;
        add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
        add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization';

        add_header Cache-Control "public,max-age=31536000";
      }

      location / {
        add_header Access-Control-Allow-Origin *;
        add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
        add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization';

        index index.html index.htm;
        try_files $uri $uri/ @rewrites;
        add_header Cache-Control "no-cache";
      }
    }
---
# Source: zadig/charts/minio/templates/pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: zadig-minio
  namespace: zadig
  labels:
    app.kubernetes.io/name: minio
    helm.sh/chart: minio-7.0.0
    app.kubernetes.io/instance: zadig
    app.kubernetes.io/managed-by: Helm
spec:
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: "20Gi"
---
# Source: zadig/charts/mongodb/templates/standalone/pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: zadig-mongodb
  namespace: zadig
  labels:
    app.kubernetes.io/name: mongodb
    helm.sh/chart: mongodb-10.20.0
    app.kubernetes.io/instance: zadig
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: mongodb
spec:
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: "20Gi"
---
# Source: zadig/templates/rbac.yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: admin-role-zadig
rules:
  - apiGroups:
      - '*'
    resources:
      - '*'
    verbs:
      - '*'
  - nonResourceURLs:
      - '*'
    verbs:
      - '*'
---
# Source: zadig/templates/rbac.yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: admin-bind-zadig
subjects:
  - kind: ServiceAccount
    namespace: zadig
    name: sa-zadig
roleRef:
  kind: ClusterRole
  name: admin-role-zadig
  apiGroup: rbac.authorization.k8s.io
---
# Source: zadig/charts/minio/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: zadig-minio
  namespace: zadig
  labels:
    app.kubernetes.io/name: minio
    helm.sh/chart: minio-7.0.0
    app.kubernetes.io/instance: zadig
    app.kubernetes.io/managed-by: Helm
spec:
  type: ClusterIP
  
  ports:
    - name: minio
      port: 9000
      targetPort: minio
      nodePort: null
  selector:
    app.kubernetes.io/name: minio
    app.kubernetes.io/instance: zadig
---
# Source: zadig/charts/mongodb/templates/standalone/svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: zadig-mongodb
  namespace: zadig
  labels:
    app.kubernetes.io/name: mongodb
    helm.sh/chart: mongodb-10.20.0
    app.kubernetes.io/instance: zadig
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: mongodb
spec:
  type: ClusterIP
  ports:
    - name: mongodb
      port: 27017
      targetPort: mongodb
      nodePort: null
  selector:
    app.kubernetes.io/name: mongodb
    app.kubernetes.io/instance: zadig
    app.kubernetes.io/component: mongodb
---
# Source: zadig/templates/aslan-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: aslan
  labels:
    app.kubernetes.io/component: aslan
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
spec:
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 25000
      targetPort: 25000
  selector:
    app.kubernetes.io/component: aslan
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
---
# Source: zadig/templates/dind-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: dind
  labels:
    app.kubernetes.io/component: dind
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
spec:
  ports:
    - name: dind
      protocol: TCP
      port: 2375
      targetPort: 2375
  clusterIP: None
  selector:
    app.kubernetes.io/component: dind
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
---
# Source: zadig/templates/hubserver-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: hub-server
  labels:
    app.kubernetes.io/component: hub-server
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
spec:
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 26000
      targetPort: 26000
  selector:
      app.kubernetes.io/component: hub-server
      app.kubernetes.io/name: zadig
      app.kubernetes.io/instance: "zadig"
---
# Source: zadig/templates/nsqlookupd-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nsqlookupd
  labels:
    app.kubernetes.io/component: nsqlookupd
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
spec:
  ports:
    - protocol: TCP
      name: tcp
      port: 4160
      targetPort: 4160
    - protocol: TCP
      name: http
      port: 4161
      targetPort: 4161
  clusterIP: None
  selector:
    app.kubernetes.io/component: nsqlookupd
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
---
# Source: zadig/templates/podexec-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: podexec
  labels:
    app.kubernetes.io/component: podexec
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
spec:
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 27000
      targetPort: 27000
  selector:
    app.kubernetes.io/component: podexec
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
---
# Source: zadig/templates/poetry-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: poetry
  labels:
    app.kubernetes.io/component: poetry
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
spec:
  type: ClusterIP
  ports:
    - name: tcp
      port: 34001
      protocol: TCP
      targetPort: 34001
  selector:
    app.kubernetes.io/component: poetry
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
---
# Source: zadig/templates/resource-server-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: resource-server
  labels:
    app.kubernetes.io/component: resource-server
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
spec:
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  selector:
    app.kubernetes.io/component: resource-server
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
---
# Source: zadig/templates/warpdrive-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: warpdrive
  labels:
    app.kubernetes.io/component: warpdrive
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
spec:
  type: ClusterIP
  ports:
    - name: warpdrive
      protocol: TCP
      port: 25001
      targetPort: 25001
  selector:
    app.kubernetes.io/component: warpdrive
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
---
# Source: zadig/templates/zadig-portal-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: zadig-portal
  labels:
    app.kubernetes.io/component: zadig-portal
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
spec:
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  selector:
    app.kubernetes.io/component: zadig-portal
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
---
# Source: zadig/charts/minio/templates/standalone/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zadig-minio
  namespace: zadig
  labels:
    app.kubernetes.io/name: minio
    helm.sh/chart: minio-7.0.0
    app.kubernetes.io/instance: zadig
    app.kubernetes.io/managed-by: Helm
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app.kubernetes.io/name: minio
      app.kubernetes.io/instance: zadig
  template:
    metadata:
      labels:
        app.kubernetes.io/name: minio
        helm.sh/chart: minio-7.0.0
        app.kubernetes.io/instance: zadig
        app.kubernetes.io/managed-by: Helm
      annotations:
        checksum/credentials-secret: 9b0bb6898a21465c483e9555ca6afd3b13c27eb3bdceae5ed095e6f33ca7db7e
    spec:
      
      serviceAccountName: zadig-minio
      affinity:
        podAffinity:
          
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app.kubernetes.io/name: minio
                    app.kubernetes.io/instance: zadig
                namespaces:
                  - "zadig"
                topologyKey: kubernetes.io/hostname
              weight: 1
        nodeAffinity:
          
      securityContext:
        fsGroup: 1001
      containers:
        - name: minio
          image: docker.io/bitnami/minio:2021.6.14-debian-10-r0
          imagePullPolicy: "IfNotPresent"
          securityContext:
            runAsNonRoot: true
            runAsUser: 1001
          env:
            - name: BITNAMI_DEBUG
              value: "false"
            - name: MINIO_FORCE_NEW_KEYS
              value: "no"
            - name: MINIO_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: zadig-minio
                  key: access-key
            - name: MINIO_SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: zadig-minio
                  key: secret-key
            - name: MINIO_DEFAULT_BUCKETS
              value: bucket
            - name: MINIO_BROWSER
              value: "on"
            - name: MINIO_PROMETHEUS_AUTH_TYPE
              value: "public"
          envFrom:
          ports:
            - name: minio
              containerPort: 9000
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /minio/health/live
              port: minio
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 5
            successThreshold: 1
            failureThreshold: 5
          readinessProbe:
            tcpSocket:
              port: minio
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 1
            successThreshold: 1
            failureThreshold: 5
          resources:
            limits: {}
            requests: {}
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          emptyDir: {}
---
# Source: zadig/charts/mongodb/templates/standalone/dep-sts.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zadig-mongodb
  namespace: zadig
  labels:
    app.kubernetes.io/name: mongodb
    helm.sh/chart: mongodb-10.20.0
    app.kubernetes.io/instance: zadig
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: mongodb
spec:
  strategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app.kubernetes.io/name: mongodb
      app.kubernetes.io/instance: zadig
      app.kubernetes.io/component: mongodb
  template:
    metadata:
      labels:
        app.kubernetes.io/name: mongodb
        helm.sh/chart: mongodb-10.20.0
        app.kubernetes.io/instance: zadig
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/component: mongodb
    spec:
      
      serviceAccountName: zadig-mongodb
      affinity:
        podAffinity:
          
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app.kubernetes.io/name: mongodb
                    app.kubernetes.io/instance: zadig
                    app.kubernetes.io/component: mongodb
                namespaces:
                  - "zadig"
                topologyKey: kubernetes.io/hostname
              weight: 1
        nodeAffinity:
          
      securityContext:
        fsGroup: 1001
        sysctls: []
      containers:
        - name: mongodb
          image: docker.io/bitnami/mongodb:4.4.6-debian-10-r8
          imagePullPolicy: "IfNotPresent"
          securityContext:
            runAsNonRoot: true
            runAsUser: 1001
          env:
            - name: BITNAMI_DEBUG
              value: "false"
            - name: ALLOW_EMPTY_PASSWORD
              value: "yes"
            - name: MONGODB_SYSTEM_LOG_VERBOSITY
              value: "0"
            - name: MONGODB_DISABLE_SYSTEM_LOG
              value: "no"
            - name: MONGODB_DISABLE_JAVASCRIPT
              value: "no"
            - name: MONGODB_ENABLE_JOURNAL
              value: "yes"
            - name: MONGODB_ENABLE_IPV6
              value: "no"
            - name: MONGODB_ENABLE_DIRECTORY_PER_DB
              value: "no"
          ports:
            - name: mongodb
              containerPort: 27017
          livenessProbe:
            exec:
              command:
                - mongo
                - --disableImplicitSessions
                - --eval
                - "db.adminCommand('ping')"
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            successThreshold: 1
            failureThreshold: 6
          readinessProbe:
            exec:
              command:
                - bash
                - -ec
                - |
                  # Run the proper check depending on the version
                  [[ $(mongo --version | grep "MongoDB shell") =~ ([0-9]+\.[0-9]+\.[0-9]+) ]] && VERSION=${BASH_REMATCH[1]}
                  . /opt/bitnami/scripts/libversion.sh
                  VERSION_MAJOR="$(get_sematic_version "$VERSION" 1)"
                  VERSION_MINOR="$(get_sematic_version "$VERSION" 2)"
                  VERSION_PATCH="$(get_sematic_version "$VERSION" 3)"
                  if [[ "$VERSION_MAJOR" -ge 4 ]] && [[ "$VERSION_MINOR" -ge 4 ]] && [[ "$VERSION_PATCH" -ge 2 ]]; then
                      mongo --disableImplicitSessions $TLS_OPTIONS --eval 'db.hello().isWritablePrimary || db.hello().secondary' | grep -q 'true'
                  else
                      mongo --disableImplicitSessions $TLS_OPTIONS --eval 'db.isMaster().ismaster || db.isMaster().secondary' | grep -q 'true'
                  fi
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 5
            successThreshold: 1
            failureThreshold: 6
          resources:
            limits: {}
            requests: {}
          volumeMounts:
            - name: datadir
              mountPath: /bitnami/mongodb
              subPath: 
      volumes:
        - name: datadir
          emptyDir: {}
---
# Source: zadig/templates/aslan-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aslan
  labels:
    app.kubernetes.io/component: aslan
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: aslan
      app.kubernetes.io/name: zadig
      app.kubernetes.io/instance: "zadig"
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app.kubernetes.io/component: aslan
        app.kubernetes.io/name: zadig
        app.kubernetes.io/instance: "zadig"
      annotations:
        checksum/config: 786982f9ca58c1779165d578958f56ab2da38524e6419531d3ade77d80c6b6be
    spec:
      imagePullSecrets:
        - name: qn-registry-secret
      serviceAccountName: sa-zadig
      containers:
        - name: nsqd
          image: ccr.ccs.tencentyun.com/koderover-public/nsqio-nsq:v1.0.0-compat
          imagePullPolicy: IfNotPresent
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 1
              memory: 512Mi
          env:
            - name: NSQD_POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          command:
            - /nsqd
          args:
            - -data-path
            - /data
            - -lookupd-tcp-address
            - nsqlookup-0.nsqlookupd:4160
            - -lookupd-tcp-address
            - nsqlookup-1.nsqlookupd:4160
            - -lookupd-tcp-address
            - nsqlookup-2.nsqlookupd:4160
            - -broadcast-address
            - $(NSQD_POD_IP)
          ports:
            - protocol: TCP
              name: tcp
              containerPort: 4150
            - protocol: TCP
              name: http
              containerPort: 4151
          volumeMounts:
            - mountPath: /data
              name: data
        - image: ccr.ccs.tencentyun.com/koderover-rc/aslan:1.5.0-amd64
          imagePullPolicy: Always
          name: aslan
          livenessProbe:
            httpGet:
              path: /api/health
              port: 25000
            initialDelaySeconds: 30
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 6
          readinessProbe:
            httpGet:
              path: /api/health
              port: 25000
            initialDelaySeconds: 5
            periodSeconds: 3
            timeoutSeconds: 3
            failureThreshold: 15
          envFrom:
            - configMapRef:
                name: aslan-config
          env:
            - name: BE_POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: BE_POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - protocol: TCP
              containerPort: 25000
          resources:
            limits:
              cpu: 2
              memory: 4Gi
            requests:
              cpu: 1
              memory: 1Gi
          volumeMounts:
            - mountPath: /etc/encryption
              name: aes-key
              readOnly: true
      volumes:
        - name: data
          emptyDir: {}
        - name: aes-key
          secret:
            secretName: zadig-aes-key
            items:
              - key: aesKey
                path: aes
---
# Source: zadig/templates/cron-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cron
  labels:
    app.kubernetes.io/component: cron
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: cron
      app.kubernetes.io/name: zadig
      app.kubernetes.io/instance: "zadig"
  template:
    metadata:
      labels:
        app.kubernetes.io/component: cron
        app.kubernetes.io/name: zadig
        app.kubernetes.io/instance: "zadig"
    spec:
      imagePullSecrets:
        - name: qn-registry-secret
      containers:
        - name: nsqd
          image: ccr.ccs.tencentyun.com/koderover-public/nsqio-nsq:v1.0.0-compat
          imagePullPolicy: IfNotPresent
          resources:
            limits:
              cpu: 1
              memory: 512Mi
            requests:
              cpu: 10m
              memory: 10Mi
          env:
            - name: NSQD_POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          command:
            - /nsqd
          args:
            - -data-path
            - /data
            - -lookupd-tcp-address
            - nsqlookup-0.nsqlookupd:4160
            - -lookupd-tcp-address
            - nsqlookup-1.nsqlookupd:4160
            - -lookupd-tcp-address
            - nsqlookup-2.nsqlookupd:4160
            - -broadcast-address
            - $(NSQD_POD_IP)
          ports:
            - protocol: TCP
              name: tcp
              containerPort: 4150
            - protocol: TCP
              name: http
              containerPort: 4151
        - image: ccr.ccs.tencentyun.com/koderover-rc/cron:1.5.0-amd64
          imagePullPolicy: 
          name: cron
          env:
            - name: ROOT_TOKEN
              value: 9F11B4E503C7F2B5
            - name: NSQLOOKUP_ADDRS
              value: nsqlookup-0.nsqlookupd:4161,nsqlookup-1.nsqlookupd:4161,nsqlookup-2.nsqlookupd:4161
          resources:
            limits:
              cpu: "1"
              memory: 1024M
            requests:
              cpu: 10m
              memory: 10Mi
---
# Source: zadig/templates/hubserver-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hub-server
  labels:
    app.kubernetes.io/component: hub-server
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: hub-server
      app.kubernetes.io/name: zadig
      app.kubernetes.io/instance: "zadig"
  template:
    metadata:
      labels:
        app.kubernetes.io/component: hub-server
        app.kubernetes.io/name: zadig
        app.kubernetes.io/instance: "zadig"
    spec:
      imagePullSecrets:
        - name: qn-registry-secret
      containers:
        - image: ccr.ccs.tencentyun.com/koderover-rc/hub-server:1.5.0-amd64
          name: hub-server
          env:
            - name: MONGODB_CONNECTION_STRING
              value: mongodb://zadig-mongodb:27017
            - name: ASLAN_DB
              value: zadig
          ports:
            - protocol: TCP
              containerPort: 26000
          resources:
            limits:
              cpu: "1"
              memory: 1G
            requests:
              cpu: 10m
              memory: 100M
          volumeMounts:
            - mountPath: /etc/encryption
              name: aes-key
              readOnly: true
      volumes:
        - name: aes-key
          secret:
            secretName: zadig-aes-key
            items:
              - key: aesKey
                path: aes
---
# Source: zadig/templates/podexec-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podexec
  labels:
    app.kubernetes.io/component: podexec
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: podexec
      app.kubernetes.io/name: zadig
      app.kubernetes.io/instance: "zadig"
  template:
    metadata:
      labels:
        app.kubernetes.io/component: podexec
        app.kubernetes.io/component: podexec
        app.kubernetes.io/name: zadig
        app.kubernetes.io/instance: "zadig"
    spec:
      imagePullSecrets:
        - name: qn-registry-secret
      serviceAccountName: sa-zadig
      containers:
        - image: ccr.ccs.tencentyun.com/koderover-rc/podexec:1.5.0-amd64
          imagePullPolicy: IfNotPresent
          name: podexec
          env:
            - name: POETRY_API_ROOT_KEY
              value: 9F11B4E503C7F2B5
          ports:
            - protocol: TCP
              containerPort: 27000
          resources:
            limits:
              cpu: "1"
              memory: 1G
            requests:
              cpu: 10m
              memory: 100M
---
# Source: zadig/templates/poetry-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: poetry
  labels:
    app.kubernetes.io/component: poetry
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: poetry
      app.kubernetes.io/name: zadig
      app.kubernetes.io/instance: "zadig"
  template:
    metadata:
      labels:
        app.kubernetes.io/component: poetry
        app.kubernetes.io/name: zadig
        app.kubernetes.io/instance: "zadig"
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: poetry
          image: "ccr.ccs.tencentyun.com/koderover-rc/poetry:1.5.0-amd64"
          imagePullPolicy: IfNotPresent
          resources:
            requests:
              cpu: 10m
              memory: 10Mi
            limits:
              cpu: 512m
              memory: 1Gi
          env:
            - name: PATH
              value: /go/bin:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
            - name: XROOTAPIKEY
              value: 9F11B4E503C7F2B5
            - name: EMAILREDIRECTURL
              value: http://zadig.mykernel.cn/v1/quality/ci
            - name: MEASUREINFOEMAILURL
              value: http://zadig.mykernel.cn/v1/quality/cd
            - name: POETRY_WEB_PREFIX
              value: http://zadig.mykernel.cn/api
            - name: USER_FILTER
              value: (&(objectClass=user)(sAMAccountName=%s))
            - name: GROUP_FILTER
              value: (&(objectClass=user)(!(objectClass=computer)))
            - name: GOLANG_VERSION
              value: 1.11.4
            - name: GOPATH
              value: /go
            - name: PROJECT_NAME
              value: poetry
            - name: PROJECT_DIR
              value: /go/src/zhuzhan/teams/backend/poetry
            - name: ASLAN_ADDRESS
              value: http://aslan:25000/api
            - name: ASLANX_ADDRESS
              value: http://aslan-x:25002/api
            - name: MGO_ADDR
              value: mongodb://zadig-mongodb:27017
            - name: MGO_DB
              value: zadig
            - name: OAUTH2_CALLBACK_ROOT
              value: http://zadig.mykernel.cn/api
            - name: SITE_ROOT
              value: http://zadig.mykernel.cn
            - name: GITHUB_OAUTH_CLIENT_ID
              value: 
            - name: GITHUB_OAUTH_CLIENT_SECRET
              value: 
          workingDir: /root/zhuzhan/go/src/zhuzhan/teams/backend/poetry/cmd/server
      restartPolicy: Always
      dnsPolicy: ClusterFirst
      imagePullSecrets:
        - name: qn-registry-secret
---
# Source: zadig/templates/resource-server-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-server
  labels:
    app.kubernetes.io/component: resource-server
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: resource-server
      app.kubernetes.io/name: zadig
      app.kubernetes.io/instance: "zadig"
  template:
    metadata:
      labels:
        app.kubernetes.io/component: resource-server
        app.kubernetes.io/name: zadig
        app.kubernetes.io/instance: "zadig"
    spec:
      imagePullSecrets:
        - name: qn-registry-secret
      containers:
        - image: ccr.ccs.tencentyun.com/koderover-rc/resource-server:1.5.0-amd64
          imagePullPolicy: Always
          name: resource-server
          resources:
            limits:
              cpu: 500m
              memory: 500Mi
            requests:
              cpu: 100m
              memory: 100Mi
---
# Source: zadig/templates/warpdrive-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: warpdrive
  labels:
    app.kubernetes.io/component: warpdrive
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/component: warpdrive
      app.kubernetes.io/name: zadig
      app.kubernetes.io/instance: "zadig"
  template:
    metadata:
      annotations:
        checksum/config: 6f729baae254bf5eb52979fb2aa5e6c7cc63b93a00cc54e50909cfc344607665
      labels:
        app.kubernetes.io/component: warpdrive
        app.kubernetes.io/name: zadig
        app.kubernetes.io/instance: "zadig"
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                topologyKey: kubernetes.io/hostname
      imagePullSecrets:
        - name: qn-registry-secret
      serviceAccountName: sa-zadig
      containers:
        - name: nsqd
          image: ccr.ccs.tencentyun.com/koderover-public/nsqio-nsq:v1.0.0-compat
          imagePullPolicy: IfNotPresent
          env:
            - name: NSQD_POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          command:
            - /nsqd
          args:
            - -data-path
            - /data
            - -lookupd-tcp-address
            - nsqlookup-0.nsqlookupd:4160
            - -lookupd-tcp-address
            - nsqlookup-1.nsqlookupd:4160
            - -lookupd-tcp-address
            - nsqlookup-2.nsqlookupd:4160
            - -broadcast-address
            - $(NSQD_POD_IP)
          ports:
            - protocol: TCP
              name: tcp
              containerPort: 4150
            - protocol: TCP
              name: http
              containerPort: 4151
          resources:
            limits:
              cpu: 1
              memory: 512Mi
            requests:
              cpu: 10m
              memory: 100M
        - name:  warpdrive
          image:  ccr.ccs.tencentyun.com/koderover-rc/warpdrive:1.5.0-amd64
          imagePullPolicy: Always
          ports:
            - protocol: TCP
              containerPort: 25001
          resources:
            limits:
              cpu: 1
              memory: 2Gi
            requests:
              cpu: 10m
              memory: 10Mi
          volumeMounts:
            - mountPath: /etc/encryption
              name: aes-key
              readOnly: true
          envFrom:
            - configMapRef:
                name: warpdrive-config
          env:
            - name: WD_POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
      volumes:
        - name: aes-key
          secret:
            secretName: zadig-aes-key
            items:
              - key: aesKey
                path: aes
---
# Source: zadig/templates/zadig-portal-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zadig-portal
  labels:
    app.kubernetes.io/component: zadig-portal
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: zadig-portal
      app.kubernetes.io/name: zadig
      app.kubernetes.io/instance: "zadig"
  template:
    metadata:
      annotations:
        checksum/config: 1ffc323afa224206265958269e76d9c264a83907212e651d9a4d091e26872056
      labels:
        app.kubernetes.io/component: zadig-portal
        app.kubernetes.io/name: zadig
        app.kubernetes.io/instance: "zadig"
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                topologyKey: kubernetes.io/hostname
      imagePullSecrets:
        - name: qn-registry-secret
      containers:
        - name:  zadig-portal
          image:  ccr.ccs.tencentyun.com/koderover-rc/zadig-portal:1.5.0-amd64
          imagePullPolicy: Always
          ports:
            - protocol: TCP
              containerPort: 80
          volumeMounts:
            - name: zadig-portal-config
              mountPath: /etc/nginx/conf.d
          resources:
            limits:
              cpu: 1
              memory: 512Mi
            requests:
              cpu: 10m
              memory: 10Mi
      volumes:
        - name: zadig-portal-config
          configMap:
            name: zadig-portal-config
---
# Source: zadig/templates/dind-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: dind
  labels:
    app.kubernetes.io/component: dind
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
spec:
  serviceName: dind
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: dind
      app.kubernetes.io/name: zadig
      app.kubernetes.io/instance: "zadig"
  template:
    metadata:
      labels:
        app.kubernetes.io/component: dind
        app.kubernetes.io/name: zadig
        app.kubernetes.io/instance: "zadig"
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                topologyKey: kubernetes.io/hostname
      imagePullSecrets:
        - name: qn-registry-secret
      containers:
        - name: dind
          image: ccr.ccs.tencentyun.com/koderover-public/library-docker:stable-dind
          args:
          - --mtu=1376
          env:
            - name: DOCKER_TLS_CERTDIR
              value: ""
          securityContext:
            privileged: true
          ports:
            - protocol: TCP
              containerPort: 2375
          resources:
            limits:
              cpu: 4
              memory: 8Gi
            requests:
              cpu: 10m
              memory: 10Mi
---
# Source: zadig/templates/nsqlookupd-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nsqlookup
  labels:
    app.kubernetes.io/component: nsqlookupd
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
spec:
  serviceName: nsqlookupd
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/component: nsqlookupd
      app.kubernetes.io/name: zadig
      app.kubernetes.io/instance: "zadig"
  template:
    metadata:
      labels:
        app.kubernetes.io/component: nsqlookupd
        app.kubernetes.io/name: zadig
        app.kubernetes.io/instance: "zadig"
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                topologyKey: kubernetes.io/hostname
      containers:
        - name: nsqlookupd
          image: ccr.ccs.tencentyun.com/koderover-public/nsqio-nsq:v1.0.0-compat
          imagePullPolicy: IfNotPresent
          resources:
            limits:
              cpu: "1"
              memory: 512Mi
            requests:
              cpu: 10m
              memory: 10Mi
          command:
            - /nsqlookupd
          ports:
            - protocol: TCP
              name: tcp
              containerPort: 4160
            - protocol: TCP
              name: http
              containerPort: 4161
---
# Source: zadig/templates/ingress-for-services.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: services
  annotations:
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "120"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    #/api/aslan/project will be rewritten to /api/project before sending request to endpoint
    ingress.kubernetes.io/rewrite-target: /api/$1
    nginx.ingress.kubernetes.io/rewrite-target: /api/$1
    nginx.ingress.kubernetes.io/use-regex: "true"                                                        
  labels:
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
spec:
  ingressClassName: nginx
  rules:
  - host: zadig.mykernel.cn
    http:
      paths:
      - path: /api/aslan/(.*)
        pathType: Prefix
        backend:
          service:
            name: aslan
            port: 
              number: 25000
      - path: /api/podexec/(.*)
        pathType: Prefix
        backend:
          service:
            name: podexec
            port: 
              number: 27000
---
# Source: zadig/templates/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pe-main
  annotations:
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "120"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
  labels:
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
spec:
  ingressClassName: nginx
  rules:
  - host: zadig.mykernel.cn
    http:
      paths:
      - path: /api/hub
        pathType: Prefix
        backend:
          service:
            name: aslan
            port: 
              number: 25000
      - path: /
        pathType: Prefix
        backend:
          service:
            name: zadig-portal
            port: 
              number: 80
---
# Source: zadig/templates/poetry-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: poetry
  annotations:
      nginx.ingress.kubernetes.io/proxy-connect-timeout: "120"
      nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
      nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
      nginx.ingress.kubernetes.io/ssl-redirect: "false"
      #/api/aslan/project will be rewritten to /api/project before sending request to endpoint
      ingress.kubernetes.io/rewrite-target: /directory/$1
      nginx.ingress.kubernetes.io/rewrite-target: /directory/$1
      nginx.ingress.kubernetes.io/use-regex: "true"
  labels:
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
spec:
  ingressClassName: nginx
  rules:
  - host: zadig.mykernel.cn
    http:
      paths:
      - path: /api/directory/(.*)
        pathType: Prefix
        backend:
          service:
            name: poetry
            port: 
              number: 34001
---
# Source: zadig/templates/post-install-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: "zadig-post-install"
  labels:
    app.kubernetes.io/component: post-hook
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
  annotations:
    "helm.sh/hook": post-install,post-upgrade
    "helm.sh/hook-delete-policy": before-hook-creation
    "helm.sh/hook-weight": "5"
data:
  VERSION: "1.5.0"
---
# Source: zadig/templates/post-install-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "zadig-post-upgrade"
  labels:
    app.kubernetes.io/component: post-hook
    helm.sh/chart: zadig-1.5.0
    app.kubernetes.io/name: zadig
    app.kubernetes.io/instance: "zadig"
    app.kubernetes.io/version: "1.5.0"
    app.kubernetes.io/managed-by: "Helm"
  annotations:
    "helm.sh/hook": post-upgrade
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
    "helm.sh/hook-weight": "-1"
spec:
  template:
    metadata:
      labels:
        app.kubernetes.io/component: post-hook
        app.kubernetes.io/name: zadig
        app.kubernetes.io/instance: "zadig"
    spec:
      imagePullSecrets:
        - name: qn-registry-secret
      restartPolicy: Never
      containers:
        - name: post-upgrade-job
          image: "ccr.ccs.tencentyun.com/koderover-rc/ua:1.5.0-amd64"
          imagePullPolicy: IfNotPresent
          resources:
            requests:
              cpu: 200m
              memory: 512Mi
            limits:
              cpu: 200m
              memory: 512Mi
          args:
            - "migrate"
            - "-c"
            - "mongodb://zadig-mongodb:27017"
            - "-d"
            - "zadig"
            - "-f"
            - "$(FROM_VERSION)"
            - "-t"
            - "1.5.0"
          env:
            - name: FROM_VERSION
              valueFrom:
                configMapKeyRef:
                  name: "zadig-post-install"
                  key: VERSION
                  optional: true
```

## nginx的yaml

`deploy.yaml`

```yaml

apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx

---
# Source: ingress-nginx/templates/controller-serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    helm.sh/chart: ingress-nginx-4.0.6
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 1.0.4
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx
  namespace: zadig
automountServiceAccountToken: true
---
# Source: ingress-nginx/templates/controller-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    helm.sh/chart: ingress-nginx-4.0.6
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 1.0.4
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx-controller
  namespace: zadig
data:
  allow-snippet-annotations: 'true'
---
# Source: ingress-nginx/templates/clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    helm.sh/chart: ingress-nginx-4.0.6
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 1.0.4
    app.kubernetes.io/managed-by: Helm
  name: ingress-nginx
rules:
  - apiGroups:
      - ''
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ''
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ''
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - networking.k8s.io
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ''
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - networking.k8s.io
    resources:
      - ingresses/status
    verbs:
      - update
  - apiGroups:
      - networking.k8s.io
    resources:
      - ingressclasses
    verbs:
      - get
      - list
      - watch
---
# Source: ingress-nginx/templates/clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    helm.sh/chart: ingress-nginx-4.0.6
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 1.0.4
    app.kubernetes.io/managed-by: Helm
  name: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ingress-nginx
subjects:
  - kind: ServiceAccount
    name: ingress-nginx
    namespace: zadig
---
# Source: ingress-nginx/templates/controller-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  labels:
    helm.sh/chart: ingress-nginx-4.0.6
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 1.0.4
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx
  namespace: zadig
rules:
  - apiGroups:
      - ''
    resources:
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ''
    resources:
      - configmaps
      - pods
      - secrets
      - endpoints
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ''
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - networking.k8s.io
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - networking.k8s.io
    resources:
      - ingresses/status
    verbs:
      - update
  - apiGroups:
      - networking.k8s.io
    resources:
      - ingressclasses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ''
    resources:
      - configmaps
    resourceNames:
      - ingress-controller-leader
    verbs:
      - get
      - update
  - apiGroups:
      - ''
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ''
    resources:
      - events
    verbs:
      - create
      - patch
---
# Source: ingress-nginx/templates/controller-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    helm.sh/chart: ingress-nginx-4.0.6
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 1.0.4
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx
  namespace: zadig
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ingress-nginx
subjects:
  - kind: ServiceAccount
    name: ingress-nginx
    namespace: zadig
---
# Source: ingress-nginx/templates/controller-service-webhook.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    helm.sh/chart: ingress-nginx-4.0.6
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 1.0.4
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx-controller-admission
  namespace: zadig
spec:
  type: ClusterIP
  ports:
    - name: https-webhook
      port: 443
      targetPort: webhook
      appProtocol: https
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/component: controller
---
# Source: ingress-nginx/templates/controller-service.yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
  labels:
    helm.sh/chart: ingress-nginx-4.0.6
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 1.0.4
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx-controller
  namespace: zadig
spec:
  type: NodePort
  ipFamilyPolicy: SingleStack
  ipFamilies:
    - IPv4
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: http
      nodePort: 30087
      appProtocol: http
    - name: https
      port: 443
      protocol: TCP
      targetPort: https
      appProtocol: https
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/component: controller
---
# Source: ingress-nginx/templates/controller-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    helm.sh/chart: ingress-nginx-4.0.6
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 1.0.4
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx-controller
  namespace: zadig
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/instance: ingress-nginx
      app.kubernetes.io/component: controller
  revisionHistoryLimit: 10
  minReadySeconds: 0
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/component: controller
    spec:
      dnsPolicy: ClusterFirst
      containers:
        - name: controller
          image: registry.cn-shanghai.aliyuncs.com/slck8s/nginx-ingress-controller:1.0.4
          imagePullPolicy: IfNotPresent
          lifecycle:
            preStop:
              exec:
                command:
                  - /wait-shutdown
          args:
            - /nginx-ingress-controller
            - --election-id=ingress-controller-leader
            - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
            - --validating-webhook=:8443
            - --validating-webhook-certificate=/usr/local/certificates/cert
            - --validating-webhook-key=/usr/local/certificates/key
          securityContext:
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            runAsUser: 101
            allowPrivilegeEscalation: true
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: LD_PRELOAD
              value: /usr/local/lib/libmimalloc.so
          livenessProbe:
            failureThreshold: 5
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
            - name: https
              containerPort: 443
              protocol: TCP
            - name: webhook
              containerPort: 8443
              protocol: TCP
          volumeMounts:
            - name: webhook-cert
              mountPath: /usr/local/certificates/
              readOnly: true
          resources:
            requests:
              cpu: 100m
              memory: 90Mi
      nodeSelector:
        kubernetes.io/os: linux
      serviceAccountName: ingress-nginx
      terminationGracePeriodSeconds: 300
      volumes:
        - name: webhook-cert
          secret:
            secretName: ingress-nginx-admission
---
# Source: ingress-nginx/templates/controller-ingressclass.yaml
# We don't support namespaced ingressClass yet
# So a ClusterRole and a ClusterRoleBinding is required
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  labels:
    helm.sh/chart: ingress-nginx-4.0.6
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 1.0.4
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: nginx
  namespace: zadig
spec:
  controller: k8s.io/ingress-nginx
---
# Source: ingress-nginx/templates/admission-webhooks/validating-webhook.yaml
# before changing this value, check the required kubernetes version
# https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#prerequisites
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  labels:
    helm.sh/chart: ingress-nginx-4.0.6
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 1.0.4
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
  name: ingress-nginx-admission
webhooks:
  - name: validate.nginx.ingress.kubernetes.io
    matchPolicy: Equivalent
    rules:
      - apiGroups:
          - networking.k8s.io
        apiVersions:
          - v1
        operations:
          - CREATE
          - UPDATE
        resources:
          - ingresses
    failurePolicy: Fail
    sideEffects: None
    admissionReviewVersions:
      - v1
    clientConfig:
      service:
        namespace: zadig
        name: ingress-nginx-controller-admission
        path: /networking/v1/ingresses
---
# Source: ingress-nginx/templates/admission-webhooks/job-patch/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ingress-nginx-admission
  namespace: zadig
  annotations:
    helm.sh/hook: pre-install,pre-upgrade,post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    helm.sh/chart: ingress-nginx-4.0.6
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 1.0.4
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
---
# Source: ingress-nginx/templates/admission-webhooks/job-patch/clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ingress-nginx-admission
  annotations:
    helm.sh/hook: pre-install,pre-upgrade,post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    helm.sh/chart: ingress-nginx-4.0.6
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 1.0.4
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
rules:
  - apiGroups:
      - admissionregistration.k8s.io
    resources:
      - validatingwebhookconfigurations
    verbs:
      - get
      - update
---
# Source: ingress-nginx/templates/admission-webhooks/job-patch/clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ingress-nginx-admission
  annotations:
    helm.sh/hook: pre-install,pre-upgrade,post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    helm.sh/chart: ingress-nginx-4.0.6
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 1.0.4
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ingress-nginx-admission
subjects:
  - kind: ServiceAccount
    name: ingress-nginx-admission
    namespace: zadig
---
# Source: ingress-nginx/templates/admission-webhooks/job-patch/role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ingress-nginx-admission
  namespace: zadig
  annotations:
    helm.sh/hook: pre-install,pre-upgrade,post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    helm.sh/chart: ingress-nginx-4.0.6
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 1.0.4
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
rules:
  - apiGroups:
      - ''
    resources:
      - secrets
    verbs:
      - get
      - create
---
# Source: ingress-nginx/templates/admission-webhooks/job-patch/rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ingress-nginx-admission
  namespace: zadig
  annotations:
    helm.sh/hook: pre-install,pre-upgrade,post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    helm.sh/chart: ingress-nginx-4.0.6
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 1.0.4
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ingress-nginx-admission
subjects:
  - kind: ServiceAccount
    name: ingress-nginx-admission
    namespace: zadig
---
# Source: ingress-nginx/templates/admission-webhooks/job-patch/job-createSecret.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: ingress-nginx-admission-create
  namespace: zadig
  annotations:
    helm.sh/hook: pre-install,pre-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    helm.sh/chart: ingress-nginx-4.0.6
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 1.0.4
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
spec:
  template:
    metadata:
      name: ingress-nginx-admission-create
      labels:
        helm.sh/chart: ingress-nginx-4.0.6
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/version: 1.0.4
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/component: admission-webhook
    spec:
      containers:
        - name: create
          image: registry.cn-shanghai.aliyuncs.com/slck8s/nginx-ingress-controller:1.1.1
          imagePullPolicy: IfNotPresent
          args:
            - create
            - --host=ingress-nginx-controller-admission,ingress-nginx-controller-admission.$(POD_NAMESPACE).svc
            - --namespace=$(POD_NAMESPACE)
            - --secret-name=ingress-nginx-admission
          env:
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
      restartPolicy: OnFailure
      serviceAccountName: ingress-nginx-admission
      nodeSelector:
        kubernetes.io/os: linux
      securityContext:
        runAsNonRoot: true
        runAsUser: 2000
---
# Source: ingress-nginx/templates/admission-webhooks/job-patch/job-patchWebhook.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: ingress-nginx-admission-patch
  namespace: zadig
  annotations:
    helm.sh/hook: post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    helm.sh/chart: ingress-nginx-4.0.6
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 1.0.4
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
spec:
  template:
    metadata:
      name: ingress-nginx-admission-patch
      labels:
        helm.sh/chart: ingress-nginx-4.0.6
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/version: 1.0.4
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/component: admission-webhook
    spec:
      containers:
        - name: patch
          image: registry.cn-shanghai.aliyuncs.com/slck8s/nginx-ingress-controller:1.1.1
          imagePullPolicy: IfNotPresent
          args:
            - patch
            - --webhook-name=ingress-nginx-admission
            - --namespace=$(POD_NAMESPACE)
            - --patch-mutating=false
            - --secret-name=ingress-nginx-admission
            - --patch-failure-policy=Fail
          env:
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
      restartPolicy: OnFailure
      serviceAccountName: ingress-nginx-admission
      nodeSelector:
        kubernetes.io/os: linux
      securityContext:
        runAsNonRoot: true
        runAsUser: 2000
```

## 应用yaml

```bash
kubectl apply -n zadig -f deploy.yaml -f zadig.yaml
```

# 问题

![image-20211026114149678](http://myapp.img.mykernel.cn/image-20211026114149678.png)

```bash
Warning: extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
```

所以目前官方支持zadig仅在1.22之前的kubernetes可用。
