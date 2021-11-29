---
title: fluentd收集kubernetes日志
date: 2021-10-21 08:43:22
tags:
- kubernetes
- elk
---



# 前言

> 官网：https://www.fluentd.org/
>
> kubernetes部署fluentd: https://docs.fluentd.org/container-deployment/kubernetes
>
> github: https://github.com/fluent/fluentd-kubernetes-daemonset
>
> > Overwrite conf file via ConfigMap. See also several examples:
> >
> > - [Cluster-level Logging in Kubernetes with Fluentd](https://medium.com/kubernetes-tutorials/cluster-level-logging-in-kubernetes-with-fluentd-e59aa2b6093a)
> > - https://github.com/fluent/fluentd-kubernetes-daemonset/pull/349#issuecomment-579097659
>
> https://kubernetes.io/docs/concepts/cluster-administration/logging/

通过日志可以了解k8s集群中发生的事情，即便很多应用有开箱即用的日志解决方案，但是在分布式的场景中，用户更倾向使用集中式日志收集解决方案。

这些可以完成从不同应用中收集不同的日志格式，发送到存储，进行处理、分析。

Kubernetes中的docker容器将日志写到`stdout`和`stderr`. docker会重定向这些流式日志到k8s的日志驱动，之后以json格式写到一个文件(日志的[滚动](https://linux.die.net/man/8/logrotate)通过,[kube-up.sh脚本完成]( https://github.com/kubernetes/kubernetes/blob/master/cluster/gce/gci/configure-helper.sh) , 使用cri container runtime时，kubelet就可以完成日志滚动( [`containerLogMaxSize` and `containerLogMaxFiles`](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/#kubelet-config-k8s-io-v1beta1-KubeletConfiguration) in [kubelet config file](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/))，管理日志目录的结构。）中, 然后就可以通过`kubectl logs`命令获取日志。通过`--previous=true`可以获取容器崩溃或重启前的日志。

可是啊，一旦容器从节点删除或者节点宕机，与之相应的日志都会被删除。这种情况下，用户将不能继续访问容器的日志。为了避免这个情况，容器的日志就应该有一个独立的转发、存储、与pod/node独立的生命周期。 Kubernetes没有提供原生存储日志数据的解决方案，但是可以通过k8s的api和controller很容易集成日志转发器。

本质上，k8s的架构使得管理应用日志非常方便。几个通用的方法可以考虑：

- 使用<strong style="color: lightpink;">sidecar container</strong>运行在app的pod中。
- 使用<strong style="color: lightpink;">node-level</strong>级别的agent运行在每一个节点之上。
- 直接在应用中将日志推送到存储上。

> 接下来简要的讨论第1和第2个方法

<!--more-->
## Using Sidecar Containers

前提

1. 存在一个pod, 不断输出日志到自己的`stdout`, `stderr` 

   ```bash
   # kubectl create deploy myapp --image=nginx
   
   # while true; do sleep .$[$RANDOM%10]; kubectl exec $(kubectl get pod myapp-6d8d776547-jlt2q -o jsonpath='{.metadata.name}') -- curl -s  localhost | grep Thank; done
   <p><em>Thank you for using nginx.</em></p>
   <p><em>Thank you for using nginx.</em></p>
   
   
   # kubectl logs --tail 1 -f $(kubectl get pod myapp-6d8d776547-jlt2q -o jsonpath='{.metadata.name}')
   ::1 - - [21/Oct/2021:01:08:05 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/7.64.0" "-"
   ::1 - - [21/Oct/2021:01:08:05 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/7.64.0" "-"
   ::1 - - [21/Oct/2021:01:08:06 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/7.64.0" "-"
   ```

   > 可以观察到一直有日志输出

2. 或者，你可以将日志写在一个日志文件中，然后你可以创建一个或多个sidecar container在应用pod中。 sidecar将会监控日志文件和app的stdout/stderr，将日志写到自己的`stdout/stderr`. 可选的，这个sidecar容器可以将日志转发给node-level的agent,  在节点上的agent上进行后续的处理和存储。更详细参考: https://kubernetes.io/docs/concepts/cluster-administration/logging/  

总结一下

- 使用sidecar容器，可以区别对待日志流(stdout/stderr/log file)。但是混合的日志格式，将很难管理日志。
- sidecar容器, 可以读应用日志，但是缺乏读应用写到`stderr`, `stdout`的日志。
- `sidecar`容器将日志写到`stdout`, `stderr`, 所以可以直接通过kubectl logs命令读sidecar日志。
- `sidecar`容器日志支持rotated.

同时, sidecar有必然的限制

- 应用将日志写到日志文件 ，sidecar读文件 ，写到标准输出，显著加大了磁盘的使用。如果应用只写一个日志文件，很容易将日志写到`/dev/stdout`, 而不是使用边车容器
- 如果要收集很多容器的日志，就要为每一个容器设计一个边车，工作量巨大

## Using a Node-Level Logging Agent

这个方法，你直接在每个节点上，部署一个日志代理。这个代理用来访问运行在这个节点上的所有 容器日志。 生产环境中，通过超过1个节点，如果在这样的场景中，你将需要为每一个节点部署一个日志agent..

通过k8s的daemonset deploy 就很容易实现以上这个。daemonset controller的行为就是，确保集群中的每一个节点有一个日志agent的pod副本。daemonset controller, 将会周期性的检查集群节点的数量，并且当节点数量改变，这个controller就会增加或减少日志agent。<strong style="color: purple">daemonset结构 对于日志解决方案就特别适合.</strong> 因为你只要创建了日志agent在每个节点之上，并且不需要修改运行在节点上的应用。

node-level agent的限制：

- 节点级别的agent只能收集应用的`stdout`和`stderr`流

> kubernetes会将应用的`stdout`和`stderr`日志流，写到节点上的`/var/log/containers/<pod名称>_<名称空间>_<容器名>-容器id.log`文件 ，此文件会链接到 
>
> `/var/log/pods/名称空间_pod名_pod的ID/容器名/0.log`, 此文件会链接到
>
> -  docker数据目录中的 containers 目录下某个日志。
>
> - containerd就是直接的日志文件 
>
> ```bash
> docker：
> 	/var/log/containers/<pod名称>_<名称空间>_<容器名>-容器id.log -> /var/log/pods/名称空间_pod名_pod的ID/容器名/0.log -> /var/lib/docker/containers/ID/test.log
> 	
> containerd:
> 	/var/log/containers/<pod名称>_<名称空间>_<容器名>-容器id.log -> /var/log/pods/名称空间_pod名_pod的ID/容器名/0.log
> ```

# 使用fluentd 收集应用日志

因为上面也说了，node-level日志agent是最佳的方法，因为它可以通过在每个节点上安装一个agent, 来收集多个应用日志集中管理。现在我们讨论怎么使用fluentd daemonset deploy在你的集群中实现这个方法。

我们选择fluentd的原因, 非常流行的日志收集agent. 对多种数据源有大量的支持(例如: apache/python, 网络协议(tcp/http/syslog),云api(aws cloud watch , aws sqs) 和更多)。数据输出也有多种后端支持：

- Log management backends (Elasticsearch, Splunk)
- Big data stores (Hadoop DFS)
- Data archiving (Files, AWS S3)
- PubSub queues (Kafka, RabbitMQ)
- Data warehouses (BigQuery, AWS RedShift)
- Monitoring systems (Datadog)
- Notification systems (email, Slack, etc.)

在这个文章中，我们集中于最流行的日志管理后端 - Elasticsearch, 它提供 极好的全文搜索， 日志聚合， 日志分析，日志可视化功能。

fluentd社区已经开发大量的预置到docker镜像的大量日志后端(包含elasticsearch)的配置，

我们使用的daemonset和docker image来自于the [**fluentd-kubernetes-daemonset**](https://github.com/fluent/fluentd-kubernetes-daemonset) GitHub repository。这儿可以找到fluentd支持的其它日志输出后端的模板， such as Loggly, Kafka, Kinesis, and more. 如果你不清楚fluentd配置，使用这个仓库将很容易开始。

完成以下示例的一些事先的准备：

- 一个运行的k8s集群。 [Supergiant documentation](https://supergiant.readme.io/docs) 这个文档可以安装k8s集群。或者通过  [Minikube](https://kubernetes.io/docs/getting-started-guides/minikube/)安装一个单机的k8s.

  > minikube安装参考： http://blog.mykernel.cn/2021/09/06/windows%E4%B8%80%E9%94%AE%E5%90%AF%E5%8A%A8kubernetes/

- 一个能与k8s通信的kubectl命令。怎么安装 **kubectl** [看这里](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

## 授权fluentd

fluentd需要收集应用日志和集群组件日志，通过官方文档，应用日志写在`stdout/stderr`时，会记录在`/var/log/containers/<pod名称>_<名称空间>_<容器名>-容器id.log`目录中，而组件日志会记录在`/var/log/组件名.log`, 因此需要一些权限

首先给fluentd的pod创建一个账号，账号的名称空间应该是fluentd部署的位置 

`sa.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  namespace: kube-system
```

> ```bash
> kubectl apply -f sa.yaml
> ```

接下来授权 fluentd  read, list, and watch pods and namespaces 

`clusterrole.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: fluentd
  namespace: kube-system
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - namespaces
  verbs:
  - get
  - list
  - watch
```

> ```bash
> kubectl apply -f clusterrole.yaml
> ```

接下来将fluentd与这些权限绑定

`clusterrolebinding.yaml`

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: fluentd
roleRef:
  kind: ClusterRole
  name: fluentd
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: fluentd
  namespace: kube-system
```

>```bash
>kubectl apply -f clusterrolebinding.yaml
>```

## 部署daemonset 

Fluentd 仓库就有一个 [daemonset set 工作示例](https://github.com/fluent/fluentd-kubernetes-daemonset/blob/master/fluentd-daemonset-elasticsearch-rbac.yaml) ，我们可以微调后使用。

`fluentd-ds.yaml`

```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: fluentd
  namespace: kube-system
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - namespaces
  verbs:
  - get
  - list
  - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: fluentd
roleRef:
  kind: ClusterRole
  name: fluentd
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: fluentd
  namespace: kube-system
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
    version: v1
    kubernetes.io/cluster-service: "true"
spec:
  selector:
    matchLabels:
      k8s-app: fluentd-logging
      version: v1
      kubernetes.io/cluster-service: "true"
  template:
    metadata:
      labels:
        k8s-app: fluentd-logging
        version: v1
        kubernetes.io/cluster-service: "true"
    spec:
      serviceAccount: fluentd
      serviceAccountName: fluentd
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:elasticsearch
        env:
        - name:  FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch"
        - name:  FLUENT_ELASTICSEARCH_PORT
          value: "9200"
        - name: FLUENT_ELASTICSEARCH_SCHEME
          value: "http"
        - name: FLUENT_UID
          value: "0"
        # X-Pack Authentication
        # =====================
        #- name: FLUENT_ELASTICSEARCH_USER
        #  value: "abf54990f0a286dc5d76"
        #- name: FLUENT_ELASTICSEARCH_PASSWORD
        #  value: "75c4bd6f7b"
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

> ```bash
> kubectl apply -f fluentd-ds.yaml
> ```
>
> 需要注意
>
> - fluentd使用fluent/fluentd-kubernetes-daemonset:elasticsearch镜像，把日志输出到es中
> - 你应该提供连接 es的环境变量，如` Elasticsearch host, port, and credentials (username, password)`, 这个es可以在集群中，也可以在远程。我们的例子使用的es  (we used a [Qbox-hosted Elasticsearch cluster](https://qbox.io/))
> - FLUENT_UID=0, 因为fluentd需要root权限读/var/log日志

## 部署elasticsearch

> https://coralogix.com/blog/elasticsearch-logstash-kibana-on-kubernetes/

`es.yaml`

> ```bash
> $ kubectl create  -n kube-system deploy elasticsearch --image=elasticsearch:7.14.2
> ```
>
> > 由于在create命令中，不能添加环境变量，所以通过以下方式添加环境
> >
> > ```bash
> > $ kubectl edit  -n kube-system deploy elasticsearch 
> > ```
> >
> > ```diff
> >       - image: elasticsearch:7.14.2
> >         name: elasticsearch
> > +        env:
> > +        - name: discovery.type
> > +          value: single-node
> > ```
>
> ```bash
> kubectl expose -n kube-system deploy elasticsearch --type=NodePort --port=9200
> minikube  service -n kube-system elasticsearch
> ```

`kibana.yaml`

```yaml
$ kubectl create  -n kube-system deploy kibana --image=kibana:7.14.2
kubectl expose -n kube-system deploy kibana --type=NodePort --port=5601
minikube service -n kube-system kibana
```

## 分析

基于pod、容器名过滤

![image-20211022102037089](http://myapp.img.mykernel.cn/image-20211022102037089.png)

# 理解fluentd的配置

上面的示例中，使用专用于elasticsearch的fluentd预置配置，因此我们不需要了解fluentd配置细节。 如果你想了解fluentd更多信息，`output destinations, filters, and more`, 请看 [官方Fluentd 文档 .](https://docs.fluentd.org/v1.0/articles/config-file)

了解fluentd配置的基本思想，我们将通过挂载configmap到fluentd daemonset方式，展示给你怎么配置 如何从日志源收集日志，输出日志，匹配规则...

通常fluentd配置文件，包含以下指令：

1. **Source** 指令，定义源。(e.g Docker, Ruby on Rails).
2. **Match** 指令，定义输出到哪。
3. **Filter** directives determine the event processing pipelines.
4. **System** directives set system-wide configuration.
5. **Label** directives group the output and filters for internal routing.
6. **@include** directives include other files.

首先看看fluentd对k8s通用的配置选项，可以找到完整的配置`kubernetes.conf` [文件来自Github Repository.](https://github.com/fluent/fluentd-kubernetes-daemonset/blob/master/templates/conf/kubernetes.conf.erb).

```yaml
<match **>
  @type stdout
</match>
<match fluent.**>
  @type null
</match>
<match docker>
  @type file
  path /var/log/fluent/docker.log
  time_slice_format %Y%m%d
  time_slice_wait 10m
  time_format %Y%m%dT%H%M%S%z
  compress gzip
  utc
</match>
<source>
  @type tail
  @id in_tail_container_logs
  path /var/log/containers/*.log
  pos_file /var/log/fluentd-containers.log.pos
  tag kubernetes.*
  read_from_head true
<% if is_v1 %>
  <parse>
    @type json
    time_format %Y-%m-%dT%H:%M:%S.%NZ
  </parse>
<% else %>
  format json
  time_format %Y-%m-%dT%H:%M:%S.%NZ
<% end %>
</source>
```

> - 前面3个块是，match 指令。指定过滤日志名或事件，并且通过`@type`变量指定输出的目标。例如：第一个match指令，使用`**`通配符选择所有日志，并且发送日志给`stdout`，然后就可以使用`kubectl logs `命令查看了。
> - 在第2个match指令，输出`@type null`, 表示忽略某些日志。最后一个match指定，我们过滤docker logs，并且将它们写入到` /var/log/fluent/docker.log`日志文件 。在这个match块中，配置了压缩类型，日志格式，和其他选项。
> - 上面配置块的最后一个块，是`source`指定。这个指定告诉fluentd日志来自于哪儿。上面的例子中，fluentd将从集群中的所有容器日志文件`/var/log/containers/*.log`读取日志。 `@type tail` ，fluentd将tail所有日志，并取回所有日志的每一行。最终，我们将读取日志文件的位置记录在`/var/log/fluentd-containers.log.pos`

你可以尝试不同事件的日志，发送到不同目标。例如，以下`fluent`模式匹配到的日志会写入`/var/log/my-fluentd.log`文件。

```yaml
<match fluent.**>
  @type file
  path /var/log/my-fluentd.log
  time_slice_format %Y%m%d
  time_slice_wait 10m
  time_format %Y%m%dT%H%M%S%z
  compress gzip
  utc
</match>
```

> 更多支持的@type, 参考[官方fluentd文档](https://docs.fluentd.org/v0.12/articles/output-plugin-overview).

本文第2节中使用的daemonset deploy fluentd，将配置文件放在`/fluentd/etc/`, 要修改默认的配置，你需要通过configmap volume挂载自定义配置到k8s.

现在可以保存上面自定义的配置于kubernetes.conf, 然后通过以下命令创建configmap

```bash
kubectl create configmap fluentd-conf --from-file=kubernetes.conf --namespace=kube-system
```

> 注意：configmap应该保存在fluentd daemonset部署的名称空间中，此例是在kube-system中

一旦创建了这个configmap, 然后修改fluentd daemonset 清单。

```diff
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
    version: v1
    kubernetes.io/cluster-service: "true"
spec:
  template:
    metadata:
      labels:
        k8s-app: fluentd-logging
        version: v1
        kubernetes.io/cluster-service: "true"
    spec:
      serviceAccount: fluentd
      serviceAccountName: fluentd
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd:v1.2-debian
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
+        - name: config-volume
+          mountPath: /fluentd/etc/kubernetes.conf
+          subPath: kubernetes.conf
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
+      - name: config-volume
+        configMap:
+          name: fluentd-conf
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

> 我们创建一个新的configmap volume，并且挂载在容器中的`/fluentd/etc/kubernetes.conf`路径。在创建这个fluentd前确保之前的fluentd已经删除。

# 结论

整体来说，k8s 通过daemonset和configmap 提供非常方便的`full logging pipelines `, 我们已经见到通过node agent部署为daemonset实现cluster-level日志集中收集、处理、展示。fluentd是最好的k8s日志解决方案，因为它基于优秀的k8s插件传送、过滤日志、等特性。

在这篇文章中，我们已经展示了fluentd很容易集中收集来自多个应用的日志，立即发送他们到elasticsearch或任何其他输出位置。不像sidecar container需要为集群中的每一个应用创建，而node-level agent，只需要为每一个节点创建。

接下来讨论fluent bit日志解决方案， 这个适合在分布式环境，有极高cpu/mem限制场景中的可以取代fluentd的应用。请继续关注我的博客。

