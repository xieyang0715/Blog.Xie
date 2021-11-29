---
title: k8s使用fortio测试web应用
date: 2021-09-26 09:25:24
tags:
- kubernetes
- curl
---



# 前言

在`busybox:latest`, `ikubernetes/myapp:v1`并不集成curl命令，不方便测试web应用，所以使用fortio运行在kubernetes上，方便测试一些web应用.



<!--more-->
# 部署

> 简易web:
>
> ```bash
> kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.8/samples/httpbin/httpbin.yaml
> # Copyright Istio Authors
> #
> #   Licensed under the Apache License, Version 2.0 (the "License");
> #   you may not use this file except in compliance with the License.
> #   You may obtain a copy of the License at
> #
> #       http://www.apache.org/licenses/LICENSE-2.0
> #
> #   Unless required by applicable law or agreed to in writing, software
> #   distributed under the License is distributed on an "AS IS" BASIS,
> #   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
> #   See the License for the specific language governing permissions and
> #   limitations under the License.
> 
> ##################################################################################################
> # httpbin service
> ##################################################################################################
> apiVersion: v1
> kind: ServiceAccount
> metadata:
>   name: httpbin
> ---
> apiVersion: v1
> kind: Service
> metadata:
>   name: httpbin
>   labels:
>     app: httpbin
>     service: httpbin
> spec:
>   ports:
>   - name: http
>     port: 8000
>     targetPort: 80
>   selector:
>     app: httpbin
> ---
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: httpbin
> spec:
>   replicas: 1
>   selector:
>     matchLabels:
>       app: httpbin
>       version: v1
>   template:
>     metadata:
>       labels:
>         app: httpbin
>         version: v1
>     spec:
>       serviceAccountName: httpbin
>       containers:
>       - image: docker.io/kennethreitz/httpbin
>         imagePullPolicy: IfNotPresent
>         name: httpbin
>         ports:
>         - containerPort: 80
> ```

```bash
kubectl <<EOF apply -f -
apiVersion: v1
kind: Service
metadata:
  name: fortio
  labels:
    app: fortio
spec:
  ports:
  - port: 8080
    name: http
  selector:
    app: fortio
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fortio-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fortio
  template:
    metadata:
      annotations:
        # This annotation causes Envoy to serve cluster.outbound statistics via 15000/stats
        # in addition to the stats normally served by Istio.  The Circuit Breaking example task
        # gives an example of inspecting Envoy stats.
        sidecar.istio.io/statsInclusionPrefixes: cluster.outbound,cluster_manager,listener_manager,http_mixer_filter,tcp_mixer_filter,server,cluster.xds-grpc
      labels:
        app: fortio
    spec:
      containers:
      - name: fortio
        image: fortio/fortio:latest_release
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          name: http-fortio
        - containerPort: 8079
          name: grpc-ping
EOF
```

# 测试

使用curl命令简单测试一个应用的接口

```bash
$ export FORTIO_POD=$(kubectl get pods -l app=fortio -o 'jsonpath={.items[0].metadata.name}')
$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio curl -quiet http://httpbin:8000/get
```

使用2个并发请求20个请求

```bash
 kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 2 -qps 0 -n 20 -loglevel Warning http://httpbin:8000/get
```

使用3个并发请求30个请求

```bash
$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 3 -qps 0 -n 30 -loglevel Warning http://httpbin:8000/get
```

