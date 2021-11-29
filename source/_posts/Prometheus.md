---
title: Prometheus
date: 2020-11-26 05:49:58
tags: 
- 日常挖掘
- prometheus
toc: true
---



《Prometheus云原生监控：运维与开发》朱政科





[^孵化]:经过社区大量的实践, 有丰富的解决方案和文档

prometheus是google[^孵化]的第二个项目, google公司内部曾使用borg系统和borgmon系统(未开源)。borg完成应用编排，borgmon是borg的配套监控。如今kubernetes、Prometheus都是对它们的理念的传承。 

<!--more-->



# 1. Prometheus

## 1.1 prometheus特点

[^1]: https://prometheus.io/

prometheus[^1]主要用于提供近实时，基于动态云环境、容器、微服务、应用程序等的监控服务。

[^2]:简单的语法，优雅的并发

Go[^2]语言编写，拥抱云原生



[^拉模式]: 从监控对象中通过轮询获取监控信息的方式, 拉模式更多拉取的是采样值或者统计值，由于有拉取间隔，因此并不能准确获取数值状态的变化，只能看到拉取间隔内的变化。拉模式在云原生环境中有比较大的优势，原因是在分布式系统中，一定是有中心节点知道整个集群信息的，那么通过中心节点就可以完成对所有要监控节点的服务发现，此时直接去拉取需要的数据就好了；

[^推模式]:基于事件主动将数据推向监控系统的方式。实时性好，一旦触发一个事件就可以立刻收集发送信息。由于事件的不可预知性，大量的数据推送到监控系统，解析和暂存会消耗大量的内存，这可能对监控的进程产生影响。推模式虽然可以省去服务发现的步骤，但每个被监控的服务都需要内置客户端，还需要配置监控服务端的信息，这无形中加大了部署的难度。推模式在OpenStack和Kubernetes等环境中使用比较少。

[^探针]: 探针 常见的有HTTP探针、TCP探针等, 探针位于应用程序的外部，通过监听端口是否有响应且返回正确的数据或状态码等外部特征来监控应用程序。

采用[^拉模式]为主、[^推模式]为辅的方式采集数据

二进制文件直接启动，也支持容器化部署镜像。

支持多种语言的客户端，如Java、JMX、Python、Go、Ruby、.NET、Node.js等语言。

可扩展。可以在每个数据中心或由每个团队运行独立Prometheus Server。也可以使用联邦集群让多个Prometheus实例产生一个逻辑集群，当单实例PrometheusServer处理的任务量过大时，通过使用功能分区（sharding）+联邦集群（federation）对其进行扩展。

精确告警。Prometheus基于灵活的PromQL语句可以进行告警设置、预测等，另外它还提供了分组、抑制、静默等功能防止告警风暴。

支持静态文件配置和动态发现等自动发现机制，目前已经支持了Kubernetes、etcd、Consul等多种服务发现机制，这样可以大大减少容器发布过程中手动配置的工作量。

开放性。Prometheus的client library的输出格式不仅支持Prometheus的格式化数据，还可以在不使用Prometheus的情况下输出支持其他监控系统（比如Graphite）的格式化数据。

## 1.2 局限性

Prometheus主要针对性能和可用性监控

不适用于针对日志（Log）、事件（Event）、调用链（Tracing）等的监控。

保存短期（如一个月）的数据：Prometheus关注的是近期发生的事情，而不是跟踪数周或数月的数据。因为大多数监控查询及告警都针对的是最近（通常不到一天）的数据。Prometheus认为最有用的数据是最近的数据，监控数据默认保留15天。需要历史数据，则建议使用Prometheus的远端存储，如OpenTSDB、M3DB等。

本地存储有限，存储大量的历史数据需要对接第三方远程存储

采用联邦集群的方式，并没有提供统一的全局视图。

Prometheus的监控数据并没有对单位进行定义。

Prometheus对数据的统计无法做到100%准确，如订单、支付、计量计费等精确数据监控场景。

Prometheus默认是拉模型，建议合理规划网络，尽量不要转发。



## 1.3 架构剖析

![image-20201126141123285](http://myapp.img.mykernel.cn/image-20201126141123285.png)

核心模块：Prometheus Server、Pushgateway、Job/Exporter、Service Discovery、Alertmanager、Dashboard

Prometheus

1. 通过服务发现机制发现target，这些target可以是长时间执行的Job，也可以是短时间执行的Job

2. 通过Exporter监控的第三方应用程序。

3. 被抓取的数据会存储起来，通过PromQL语句在仪表盘等可视化系统中供查询
4. 触发rule阈值时, 向Alertmanager发送告警信息，告警会通过页面、电子邮件、钉钉信息或者其他形式呈现。



### 1.3.1 Job/Exporter

**Job/Exporter**属于Prometheus target，是Prometheus监控的对象。

1. 长时间执行job: 可以使用Prometheus Client集成进行监控

2. 短时间执行job:  可以将监控数据推送到Pushgateway中缓存



**Exporter**的机制是将第三方系统的监控数据按照Prometheus的格式暴露出来，没有Exporter的第三方系统可以自己定制Exporter。 Prometheus是一个白盒监视系统，也可以使用blackbox_exporter完成黑盒监控。

>黑盒监控:  基于[^探针], 发现系统故障。也叫探针监控。
>
>白盒监控: 通过对监控指标的观察能够预判可能出现的问题，从而对潜在的不确定因素进行优化。也叫自省监控。

Exporter越多，维护压力越大，尤其是内部自行开发的Agent等工具需要大量的人力来完成资源控制、特性添加、版本升级等工作，可以考虑替换为Influx Data公司开源的Telegraf[4]统一进行管理。



### 1.3.2 Pushgateway

用于监控Prometheus服务器无法抓取的资源的解决方案

支持临时性Job主动推送指标的中间网关

它存在单点故障问题。如果Pushgateway从许多不同的来源收集指标时宕机，用户将失去对所有这些来源的监控，可能会触发许多不必要的告警。

使用Pushgateway时需要记住的另一个问题是，Pushgateway不会自动删除推送给它的任何指标数据。因此，必须使用Pushgateway的API从推送网关中删除过期的指标。

Pushgateway还有防火墙和NAT问题。推荐做法是将Prometheus移到防火墙后面，让Prometheus更加接近采集的目标。注意，Pushgateway会丧失Prometheus通过UP监控指标检查实例健康状况的功能，此时Prometheus对应的拉状态的UP指标只是针对单Pushgateway服务的。



### 1.3.3 服务发现（Service Discovery）

周期性地获取最新的target

管理员可以在不重启Prometheus服务的情况下动态发现需要监控的target实例信息。

Relabeling机制。Relabeling机制会从Prometheus包含的target实例中获取默认的元标签信息，从而对不同开发环境（测试、预发布、线上）、不同业务团队、不同组织等按照某些规则（比如标签）从服务发现注册中心返回的target实例中有选择性地采集某些Exporter实例的监控数据。



1. 本身支持文件的服务发现

   与自动化配置管理工具（Ansible、Cron Job、Puppet、SaltStack等）结合使用

   相对于直接使用文件配置，在云环境以及容器环境下我们更多的监控对象都是动态的。

2. Prometheus还支持多种常见的服务发现组件，如Kubernetes、DNS、Zookeeper、Azure、EC2和GCE等。

   例如，Prometheus可以使用Kubernetes的API获取容器信息的变化（如容器的创建和删除）来动态更新监控对象。



### 1.3.4 Prometheus服务器（Prometheus Server）

抓取、存储和查询

1. 抓取：Prometheus Server通过服务发现组件，周期性地从上面介绍的Job、Exporter、Pushgateway这3个组件中通过HTTP轮询的形式拉取监控指标数据。
2. 存储：抓取到的监控数据通过一定的规则清理和数据整理（抓取前使用服务发现提供的relabel_configs方法，抓取后使用作业内的metrics_relabel_configs方法），会把得到的结果存储到新的时间序列中进行持久化。  存储qps: 数百万个样品/s
   - 本地存储：会直接保留到本地磁盘，性能上建议使用SSD且不要保存超过一个月的数据。任何版本的Prometheus都不支持NFS。一些实际生产案例[5]告诉我们，Prometheus存储文件如果使用NFS，则有损坏或丢失历史数据的可能。
   - 远程存储：远程存储需要配合中间层的适配器进行转换，主要涉及Prometheus中的remote_write和remote_read接口。在实际生产中，远程存储会出现各种各样的问题，需要不断地进行优化、压测、架构改造甚至重写上传数据逻辑的模块等工作。
3. 查询：Prometheus持久化数据以后，客户端就可以通过PromQL语句对数据进行查询了。

### 1.3.5 Dashboard

Web UI、Grafana、API client可以统一理解为Prometheus的Dashboard

### 1.3.6 Alertmanager

独立于Prometheus的一个告警组件，需要单独安装部署

Prometheus可以将多个Alertmanager配置为一个集群，通过服务发现动态发现告警集群中节点的上下线从而避免单点问题，Alertmanager也支持集群内多个实例之间的通信

Alertmanager提供了多种内置的第三方告警通知方式，同时还提供了对Webhook通知的支持，通过Webhook用户可以完成对告警的更多个性化的扩展。Alertmanager除了提供基本的告警通知能力以外，还提供了如分组、抑制以及静默等告警特性





# 2. Spring Boot 可视化监控实战

本章将带着读者一起搭建集成了Micrometer功能的Spring Boot监控系统（含JVM监控），并配置Grafana监控大盘以及邮件、钉钉等告警方式，帮助读者打通从Prometheus到Spring Boot再到Grafana的完整链路


## 2.1 采集架构

Spring Boot 2.x通过Micrometer提供了监控数据，但是这些数据并没有可视化，所以不够友好。如果要可视化，还需要我们执行两个操作：·Prometheus配置轮询采集Spring Boot 2.x的应用target提供的数据。·Grafana将Prometheus作为数据源进行可视化大盘展示。

![image-20201127163542528](http://myapp.img.mykernel.cn/image-20201127163542528.png)



## 2.2 spring boot搭建

参考： https://blog.csdn.net/deniro_li/article/details/100856392 的2 IntelliJ IDEA 初始化

![image-20201127152457212](http://myapp.img.mykernel.cn/image-20201127152457212.png)

1. pom.xml中存放Maven的依赖

2. application.properties存放Spring Boot的配置文件

3. DemoApplication是Spring Boot的入口程序, DemoApplicationTests测试文件

4. DemoMetrics和SimulationRequest会对请求进行模拟以更新指标数据。

5. com.example.hello_world 其中前2项为 group. 最后一项为artifacts.

6. 使用`jdk 8`, `apache maven 3.6.3`

7. 为了更快的加载依赖，在settings.xml中需要配置如下镜像

   ```bash
   <mirrors>
       <mirror>
           <id>nexus-aliyun</id>
           <name>nexus-aliyun</name>
           <url>http://maven.aliyun.com/nexus/content/groups/public</url>
           <mirrorOf>central</mirrorOf>
       </mirror>
   </mirrors>
   ```

   

### 2.2.1 配置Micrometer依赖

在pom.xml中的 <dependency> </dependency> 其中添加以下依赖

```xml
      <!--监控系统健康情况的工具-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <!--        启动spring web-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>


        <!--        桥接prometheus-->
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>

        </dependency>

        <!--        micrometer核心包，按需引入，使用meter注解或手动埋点时需要-->
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-core</artifactId>

        </dependency>

        <!--        micrometer获取jvm相关信息, 并展示在grafana上-->
        <dependency>
            <groupId>io.github.mweirauch</groupId>
            <artifactId>micrometer-jvm-extras</artifactId>
            <version>0.1.4</version>
        </dependency>
```



### 2.2.2 配置文件

对于Spring Boot 2.x而言，如果需要集成Prometheus，那么application.properties的建议配置如下。

```java
# java监听
server.address=0.0.0.0
server.port=18080        

# 指定服务名
spring.application.name=Demo  

# metrics 应用指向
management.metrics.tags.application=${spring.application.name}

#actuator默认只开启了/info和/health，如果想要使用接口，需要在配置中添加类似如下的代码。
management.endpoints.web.exposure.include=*
management.endpoints.web.exposure.exclude= env, beans

# 启动curl -X POST localhost:18080/actuator/shutdown来停止springboot微服务
management.endpoint.shutdown.enabled=true

management.metrics.export.simple.enabled=false
```



### 2.2.3 通过MeterBinder接口采集和注册指标

编辑 com.example.hello_world下的metric目录中的DemoMetrics文件

```java
package com.example.hello_world.metrics;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.Gauge;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.binder.MeterBinder;
import org.springframework.stereotype.Component;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.Map;

@Component
public class DemoMetrics implements MeterBinder {
    public Counter counter;
    public Map<String, Double> map;

    //Map对象, 对该组件进行扩展
    //demoMetrics.map.put("x“，counter1)
    //demoMetrics.map.put("y“，counter2)
    //demoMetrics.map.put("z“，counter3)
    //DemoMetrics根据Map的Key的名称x/y/z取出业务端埋点值

    DemoMetrics() {
        map = new HashMap<>();
    }

    @Override
    public void bindTo(MeterRegistry meterRegistry) {
        // 定义并注册一个名称为prometheus.demo.counter的计数器，标签是name: counter1
        this.counter = Counter.builder("prometheus.demo.counter").tags(new String[]{"name","counter1"}).description("demo counter").register(meterRegistry);

        // 从业务 端传递的Map中取出Key对应的值放入注册的Gauge登仪表盘中，标签是name: gauge1
        // Gauge.builder("prometheus.demo.gauge",map,x->x.get("x")).tags("name",
        //   "counter1"}).description("This is Gauge").register(meterRegistry);
    }
}

```



### 2.2.4 更新指标

注册完指标后就需要更新指标信息了。很多开发者会在Controller中通过模拟请求的方式进行业务埋点，而我们这里为了测试简单，在程序入口使用了定时器。

编辑HelloWorldApplication程序入口

```java
package com.example.hello_world;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
//注册完指标后就需要更新指标信息了。很多开发者会在Controller中通过模拟请求的方式进行业务埋点，而我们这里为了测试简单，使用了定时器。
//首先在Spring Boot应用程序入口处增加@EnableScheduling注解，这个Spring Boot启动类还是比较简单的
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
// 该注解用于引用定时功能, 方便下面要介绍的业务模拟请示代码
// SimulationRequest.java进行@Scheduled操作
@EnableScheduling

public class HelloWorldApplication {
    public static void main(String[] args) {
        SpringApplication.run(HelloWorldApplication.class, args);
    } // 程序入口

}

```

### 2.2.5 模拟代码请求

编辑 com.example.hello_world下的metric目录中的SimulationRequest文件

```java
//用于模拟代码请求。该类可以通过定时器设置一个任务，每秒执行一次@Scheduled（fixedDelay=1000）。
// 定时器每执行一次，成员变量count1的值就加1，并将最新的count1值放入之前封装的监控组件DemoMetrics的Map对象中。

// 如果你有多个业务需要监控，可以定义多个成员变量，并将它们对应的最新数据通过定义不同key的方式放入监控组件DemoMetrics的Map对象中。


package com.example.hello_world.metrics;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Async;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;


@Component
public class SimulationRequest {
    private Integer count1 = 0;

    @Autowired
    private DemoMetrics demoMetrics;

    @Async("One")
    @Scheduled(fixedDelay =  1000)
    public void increment1() {
        count1++;
        demoMetrics.counter.increment();
        demoMetrics.map.put("x",Double.valueOf(count1));
        // 将counte1的值放入Gauge中, 反映应用的当前指标, 比如主机当前空闲的内存大小
        // (node_memory_MemFree)
        System.out.println("increment1 count:" + count1);
    }
}

```



### 2.2.6 启动项目

直接点击右上脚的运行

浏览器请求http://localhost:18080/actuator/

http://localhost:18080/actuator/metrics就可以得到这个Spring Boot案例程序提供的监控指标

```bash
{"names":["http.server.requests","jvm.buffer.count","jvm.buffer.memory.used","jvm.buffer.total.capacity","jvm.classes.loaded","jvm.classes.unloaded","jvm.gc.live.data.size","jvm.gc.max.data.size","jvm.gc.memory.allocated","jvm.gc.memory.promoted","jvm.gc.pause","jvm.memory.committed","jvm.memory.max","jvm.memory.used","jvm.threads.daemon","jvm.threads.live","jvm.threads.peak","jvm.threads.states","logback.events","process.cpu.usage","process.start.time","process.uptime","prometheus.demo.counter","system.cpu.count","system.cpu.usage","tomcat.sessions.active.current","tomcat.sessions.active.max","tomcat.sessions.alive.max","tomcat.sessions.created","tomcat.sessions.expired","tomcat.sessions.rejected"]}
```

例如获取其中某个指标 jvm.buffer.memory.used 

http://localhost:18080/actuator/metrics/jvm.buffer.memory.used 

```bash
{"name":"jvm.buffer.memory.used","description":"An estimate of the memory that the Java virtual machine is using for this buffer pool","baseUnit":"bytes","measurements":[{"statistic":"VALUE","value":65536.0}],"availableTags":[{"tag":"application","values":["Demo"]},{"tag":"id","values":["mapped","direct"]}]}

```

以上就是Prometheus通过Micrometer集成Spring Boot的基本原理和方法。



## 2.3 prometheus 安装及配置

在Prometheus的官方下载页面https://prometheus.io/download/中可以看到，Prometheus提供了独立的二进制文件的tar包，其中主要包括prometheus、alertmanager、blackbox_exporter、consul_exporter、graphite_exporter、haproxy_exporter、memcached_exporter、mysqld_exporter、node_exporter、Pushgateway、statsd_exporter等组件。



```bash
# tar xvf prometheus-2.23.0.linux-amd64.tar.gz 
# ln -sv /usr/local/src/prometheus-2.23.0.linux-amd64 /usr/local/prometheus
# cd /usr/local/prometheus
# ./prometheus  --config.file=./prometheus.yml --web.enable-lifecycle

# --web.enable-lifecycle 当--config.file=指定文件改变时, 不需要重启prometheus, 可以通过 curl -X POST http://localhost:9090/-/reload完成动态重载。
#11月 27 16:53:06 centos7-template.magedu.local prometheus[5630]: level=info ts=2020-11-27T08:53:06.871Z caller=main.go:871 msg="Loading configuration file" filename=/usr/local/prometheus/prometheus.yml
#11月 27 16:53:06 centos7-template.magedu.local prometheus[5630]: level=info ts=2020-11-27T08:53:06.908Z caller=main.go:902 msg="Completed loading of configuration file" filename=/usr/local/prometheus/prometheus.yml totalDuration=36.320656ms remote_storage=3.638µs web_handler=714ns query_engine=2.187µs scrape=34.55
```



localhost:9090/metrics，该地址返回与Prometheus Server状态相关的监控信息

指标类型

- counter 持续递增
- guage   数值是动态变化的
- Histogram在一段时间范围内对数据进行采样（通常是请求持续时间或响应大小等），并将其计入可配置的存储桶（Bucket）中，后续可通过指定区间筛选样本，也可以统计样本总数，最后一般将数据展示为Histogram。Histogram可以用于应用性能等领域的分析观察。

localhost:9090/graph是Prometheus的默认查询界面

在Graph页面可以输入PromQL表达式，比如输入“up”，就可以查看监控的每个Job的健康状态，1表示健康，0表示不健康。

```bash
up{instance="localhost:9090",job="prometheus"}
```



Linux中，后台启动prometheus

```bash
nohup  ./prometheus & 
```

但是为了方便查看进程状态, 直接使用systemd来管理

```bash
# /usr/lib/systemd/system/prometheus.service
[Unit]
Description=Pormetheus Server Daemon
After=network.target

[Service]
Type=simple
User=root
Group=root
ExecStart=/usr/local/prometheus/prometheus  --config.file=/usr/local/prometheus/prometheus.yml --web.listen-address "0.0.0.0:9090"   --web.enable-lifecycle
Restart=on-failure

[Install]
WantedBy=multi-user.target



# 
# systemctl daemon-reload
# systemctl enable --now prometheus
# systemctl status prometheus
```



配置prometheus.yaml

```bash

# my global config
global:
  scrape_interval:     15s # Set the scrape拉数据 interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate评估 rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).



# 重载规则时, --web.enable-lifecycle 当--config.file=指定文件改变时, 不需要重启prometheus, 可以通过 curl -X POST http://localhost:9090/-/reload完成动态重载。
rule_files: # 对抓取的数据, 由evaluation_interval指定的时间内评估这些规则
  # - "first_rules.yml"
  # - "second_rules.yml"
 

# 当评估的规则达到阈值就可以产生报警, 并将报警信息推送给alertmanager:9093
alerting:
  alertmanagers:
  - static_configs: # 配置alertmanager地址
    - targets:
      # - alertmanager:9093



# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # job的name会作为job=<job_name>加入到每个时序数据中
  - job_name: 'prometheus'
    static_configs: # 静态配置抓取地址
    - targets: ['localhost:9090',"192.168.0.12:9090"]

  - job_name: "springboot-demo" # spring boot微服务
    metrics_path: "/actuator/prometheus"      # 192.168.0.33:18080/actuator/prometheus 会输出prometheus格式的数据
    scheme: http
    #honor_labels: # true 通过保留标签解决冲突。false 通过重命名来解决冲突 exported_<origin_label> 默认false
    #scrape_interval: 采集间隔, 继承自全局
    #scrape_timeout: 采集超时, 继承自全局

    #params: # url的参数
    #relabel_configs: #采集数据重置标签配置
    #metric_relabel_configs: # 重置标签配置
    #sample_limit: #限制sample每次采集的数量, 超过限制视为失败。默认0表示没有限制 
    static_configs: # 静态配置抓取地址
    - targets: ['192.168.0.33:18080']

```

告警及阈值在prometheus中配置，而不是在alertmanager中配置, 在prometheus目录中创建一个触发规则文件

```bash
# /usr/local/src/prometheus/alert_rules.yml
groups:
- name: demo-alert-rule
  rules:
  - alert: DemoJobDown
    expr: sum(up{job="springboot-demo"}) == 0
    for: 1m
    labels:
      severity: critical

```

并在prometheus.yaml中引用

```bash
# /usr/local/src/prometheus/prometheus.yml
rule_files: 
   - "alert_rules.yml"
   
# curl -X POST http://localhost:9090/-/reload
```



## 3.2 alertmanager 报警媒介配置及报警查询

```bash
# tar xvf alertmanager-0.21.0.linux-amd64.tar.gz 
# ln -sv /usr/local/src/alertmanager-0.21.0.linux-amd64 /usr/local/alertmanager
# cd /usr/local/alertmanager
# ./alertmanager
```



配置alertmanager.yml 邮件如下 

```bash
global:
  resolve_timeout: 5m

  smtp_smarthost: "smtp.qq.com:443"
  smtp_from: "1062670898@qq.com"
  smtp_auth_username: "test"
  smtp_auth_password: "11112122121212121"

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'mail-receiver'
  receiver: 'web.hook'
receivers:
- name: 'web.hook'
  webhook_configs:
  - url: 'http://127.0.0.1:5001/' # 当我们触发此项可以在另一侧启动一个flask, 只要触发这个监控项可以完成一个flask执行脚本。完成功能就很复杂了。例如：当监控到某个指标达到阈值就, flask执行脚本完成ecs自动扩容
- name: 'mail-receiver'
  email_configs:
  - to: "1062670898@qq.com"



inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']

```

>电子邮件可以有3种类型的收件人，分别为to、cc和bcc，分别是收件人、抄送和密送。但是Prometheus的官方文档却没有cc和bcc的配置信息，那么我们该如何做呢？
>Alertmanager提供的不仅有邮件功能，还有SMTP的相关功能。Alertmanager会将大写的To标头默认为to配置字段的值，下面的例子中，email_config中的配置就实现了to、cc和bcc的方法。
>
>```bash
>  email_configs:
>  - to: to@qq.com,cc@qq.com,bcc@qq.com
>    headers:
>      To: cc@qq.com
>      CC: bcc@qq.com
>```
>
>文件主要是设置告警方式, 如：邮件、钉钉（webhook）、微信、pagerduty等方式.它的选项主要有email_config、hipchat_config、pagerduty_config、pushover_config、slack_config、opsgenie_config、victorops_config等。

配置完alertmanager，启动alertmanager, 并在Prometheus指定此alertmanager

```bash
[root@centos7-template alertmanager]# ./alertmanager 
level=info ts=2020-11-27T09:01:04.555Z caller=main.go:216 msg="Starting Alertmanager" version="(version=0.21.0, branch=HEAD, revision=4c6c03ebfe21009c546e4d1e9b92c371d67c021d)"
level=info ts=2020-11-27T09:01:04.555Z caller=main.go:217 build_context="(go=go1.14.4, user=root@dee35927357f, date=20200617-08:54:02)"
level=info ts=2020-11-27T09:01:04.566Z caller=cluster.go:161 component=cluster msg="setting advertise address explicitly" addr=192.168.0.12 port=9094
level=info ts=2020-11-27T09:01:04.570Z caller=cluster.go:623 component=cluster msg="Waiting for gossip to settle..." interval=2s
level=info ts=2020-11-27T09:01:04.637Z caller=coordinator.go:119 component=configuration msg="Loading configuration file" file=alertmanager.yml
level=info ts=2020-11-27T09:01:04.638Z caller=coordinator.go:131 component=configuration msg="Completed loading of configuration file" file=alertmanager.yml
level=info ts=2020-11-27T09:01:04.646Z caller=main.go:401 component=configuration msg="skipping creation of receiver not referenced by any route" receiver=web.hook
level=info ts=2020-11-27T09:01:04.647Z caller=main.go:485 msg=Listening address=:9093

```



```bash
# /usr/local/src/prometheus/prometheus.yml
alerting:
  alertmanagers:
  - static_configs: # 配置alertmanager地址
    - targets:
      - 192.168.0.12:9093
```

热加载prometheus配置

```bash
curl -X POST http://localhost:9090/-/reload
```



此时可以停止prometheus的springboot，可以看到已经收到报警

```bash
#  ./amtool alert query --alertmanager.url=http://192.168.0.12:9093
 
 [root@centos7-template alertmanager]#  ./amtool alert query --alertmanager.url=http://192.168.0.12:9093
Alertname    Starts At                Summary  
DemoJobDown  2020-11-27 09:03:29 UTC           
```





## 3.3 grafana配置

CNCF下可视化面板的Go语言项目，有图表和布局展示，以及功能齐全的度量仪表盘和图形编辑器，主要用于大规模指标数据的可视化展示，是网络架构和应用分析中最流行的时序数据展示工具

支持绝大部分时序数据库，支持Graphite、Elasticsearch、InfluxDB、Prometheus、CloudWatch、MySQL和OpenTSDB等数据源，

### 3.3.1 安装grafana

```bash
~]# docker run -d --name=grafana -p 3000:3000 grafana/grafana
```

Grafana默认的端口是3000。访问http://127.0.0.1:3000可以进入Grafana的主界面，如图3-7所示。第一次登录时使用的用户名为admin，密码为admin，登录后会提示修改密码。

### 3.3.2 配置数据源

![image-20201127182935573](http://myapp.img.mykernel.cn/image-20201127182935573.png)

当点下方test and save, 显示绿色表示通

![image-20201127182958646](http://myapp.img.mykernel.cn/image-20201127182958646.png)

现在将上面springboot的指标`prometheus_demo_counter_total{**application**="Demo", **instance**="192.168.0.33:18080", **job**="springboot-demo", **name**="counter1"}`展示

```bash
prometheus_demo_counter_total{application="Demo", instance="192.168.0.33:18080", job="springboot-demo", name="counter1"}
#prometheus配置
#instance="192.168.0.33:18080" 是 targets: ['192.168.0.33:18080']指定的值
#job="springboot-demo" 是 - job_name: "springboot-demo"

#代码
#name="counter1" 代码指定tags
#application 是spring.application.name=Demo
```

![image-20201127185529633](http://myapp.img.mykernel.cn/image-20201127185529633.png)

![image-20201127185550737](http://myapp.img.mykernel.cn/image-20201127185550737.png)



### 3.3.3  jvm数据展示

通过Spring Boot中加入的Maven依赖micrometer-jvm-extras，可以获取Spring Boot的JVM信息，JVM信息也是可以展现在Grafana监控大盘上的，如下所示。

```bash
http://192.168.0.33:18080/actuator/prometheus # 返回有jvm信息, 是由micrometer-jvm-extras生成
```

https://grafana.com/grafana/dashboards是Grafana提供的官方大盘模板网站，在这里可以搜到很多与开源软件监控指标相关的现成的模板，也可以学到企业级定制中间件、系统监控模板的可视化相关方法。micrometer-jvm-extras对应的模板地址是https://grafana.com/grafana/dashboards/4701

![image-20201127185829945](http://myapp.img.mykernel.cn/image-20201127185829945.png)

![image-20201127185859853](http://myapp.img.mykernel.cn/image-20201127185859853.png)



![image-20201127190122038](http://myapp.img.mykernel.cn/image-20201127190122038.png)

> 摘自书中



## 3.4 node_exporter完成节点监控

```bash
# wget https://github.com/prometheus/node_exporter/releases/download/v1.0.1/node_exporter-1.0.1.linux-amd64.tar.gz
tar xvf node_exporter-1.0.1.linux-amd64.tar.gz 
ln -sv /usr/local/src/node_exporter-1.0.1.linux-amd64 /usr/local/node_exporter
cd /usr/local/node_exporter
./node_exporter 
```

prometheus添加抓取数据

```bash
# /usr/local/src/prometheus/prometheus.yml
scrape_configs:
  #略....
  - job_name: 'node_exporter'
    static_configs: # 静态配置抓取地址
    - targets: ['localhost:9100']

# curl -X POST http://localhost:9090/-/reload

# http://192.168.0.12:9090/targets 存在node_exporter
```

>  在granfa导入 8919即可



# 3. PromQL让数据会说话

时间序列、PromQL数据类型、选择器、指标类型、聚合操作、二元操作符、内置函数、最佳实践、性能优化等方面

```sql
# 获取当前主机可用的内存空间大小，单位为MB。 free -m
# node_memory_MemFree_bytes / (1024 * 1024)


# 基于2小时的样本数据，预测未来24小时内磁盘是否会满


```





