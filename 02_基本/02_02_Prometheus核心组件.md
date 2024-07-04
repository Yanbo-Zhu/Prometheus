

上一小节，通过部署Node Exporter我们成功的获取到了当前主机的资源使用情况。接下来我们将从Prometheus的架构角度详细介绍Prometheus生态中的各个组件。

下图展示Prometheus的基本架构：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPS8BVjkRvEjV8HmbBi%252F-LPS8D1gM9qp1zu_wp8y%252Fprometheus_architecture.png%3Fgeneration%3D1540234733609534%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=17ded6a9&sv=1)



架构图：时序数据存储，抓取数据，推送告警，提供PromQL查询数据，对接UI仪表盘，和K8s对接。

![](image/Pasted%20image%2020240704115828.png)


# 1 总体分析 


从上图可以看出，Prometheus 的主要模块包括：Prometheus server, exporters, Pushgateway, PromQL, Alertmanager 以及图形界面。

最左边这块就是采集的，主要包括exporters和Pushgateway。

    Exporters：采集已有的第三方服务监控指标并暴露metrics，相当于一个采集端的agent，说白了就是我们自己的服务或应用，比如hubble-biz-cm就是一个Exporter，其暴露了metrics接口。之后Prometheus server也会此接口拉取数据。
    Pushgateway：客户端将监控数据主动推送到Pushgateway，之后Prometheus server也会从Pushgateway拉取数据。主要用于临时性的 jobs。由于这类 jobs 存在时间较短，可能在 Prometheus 来 pull 之前就消失了。对此Jobs定时将指标push到pushgateway，再由Prometheus Server从Pushgateway上pull。

中间这块就是Prometheus它本身：内部是有一个TSDB的数据库的，从内部的采集和展示Prometheus它都可以完成，展示这块自己的这块UI比较lou，所以借助于这个开源的Grafana来展示，所有的被监控端暴露完指标之后，Prometheus会主动的抓取这些指标，存储到自己TSDB数据库里面，提供给Web UI,或者Grafana，或者API clients通过PromQL来调用这些数据，PromQL相当于Mysql的SQL，主要是查询这些数据的。

中间上面这块是做服务发现的，也就是你有很多的被监控端时，手动的去写这些被监控端是不现实的，所以需要自动的去发现新加入的节点，或者以批量的节点，加入到这个监控中，K8S内置了服务发现机制，也就是Promethus会连接k8s的API，去发现你部署的哪些应用，哪些pod，通通的都给你暴露出去，监控出来，也就是为什么K8S对prometheus特别友好的地方，也就是它内置了做这种相关的支持了。

右上角是Prometheus的告警，即Alertmanager，：从 Prometheus server 端接收到 alerts 后，会进行去除重复数据，分组，并路由到对收的接受方式，发出报警。hubble的实现是将Alertmanager的告警发给adapter，然后转发给Alarm组件。

小结：
• Prometheus Server：收集指标和存储时间序列数据，并提供查询接口
• ClientLibrary：客户端库，这些可以集成一些很多的语言中，比如使用JAVA开发的一个Web网站，那么可以集成JAVA的客户端，去暴露相关的指标，暴露自身的指标，但很多的业务指标需要开发去写的，
• Web UI：Prometheus内置一个简单的Web控制台，可以查询指标，查看配置信息或者Service Discovery等，实际工作中，查看指标或者创建仪表盘通常使用Grafana，Prometheus作为Grafana的数据源；


# 2 Prometheus Server

Prometheus Server是Prometheus组件中的核心部分，负责实现对监控数据的获取，存储以及查询。 Prometheus Server可以通过静态配置管理监控目标，也可以配合使用Service Discovery的方式动态管理监控目标，并从这些监控目标中获取数据。其次Prometheus Server需要对采集到的监控数据进行存储，Prometheus Server本身就是一个时序数据库，将采集到的监控数据按照时间序列的方式存储在本地磁盘当中。最后Prometheus Server对外提供了自定义的PromQL语言，实现对数据的查询以及分析。

Prometheus Server内置的Express Browser UI，通过这个UI可以直接通过PromQL实现数据的查询以及可视化。

Prometheus Server的联邦集群能力可以使其从其他的Prometheus Server实例中获取数据，因此在大规模监控的情况下，可以通过联邦集群以及功能分区的方式对Prometheus Server进行扩展。


# 3 Exporters

Exporter将监控数据采集的端点通过HTTP服务的形式暴露给Prometheus Server，Prometheus Server通过访问该Exporter提供的Endpoint端点，即可获取到需要采集的监控数据。

一般来说可以将Exporter分为2类：
- 直接采集：这一类Exporter直接内置了对Prometheus监控的支持，比如cAdvisor，Kubernetes，Etcd，Gokit等，都直接内置了用于向Prometheus暴露监控数据的端点。
- 间接采集：间接采集，原有监控目标并不直接支持Prometheus，因此我们需要通过Prometheus提供的Client Library编写该监控目标的监控采集程序。例如： Mysql Exporter，JMX Exporter，Consul Exporter等。

# 4 AlertManager

在Prometheus Server中支持基于PromQL创建告警规则，如果满足PromQL定义的规则，则会产生一条告警，而告警的后续处理流程则由AlertManager进行管理。

在AlertManager中我们可以与邮件，Slack等等内置的通知方式进行集成，也可以通过Webhook自定义告警处理方式。AlertManager即Prometheus体系中的告警处理中心。

# 5 PushGateway

由于Prometheus数据采集基于Pull模型进行设计，因此在网络环境的配置上必须要让Prometheus Server能够直接与Exporter进行通信。 当这种网络需求无法直接满足时，就可以利用PushGateway来进行中转。可以通过PushGateway将内部网络的监控数据主动Push到Gateway当中。而Prometheus Server则可以采用同样Pull的方式从PushGateway中获取到监控数据。


Prometheus Pushgateway 是 Prometheus 生态系统中的一个组件，它允许短期运行的工作 (jobs) 推送（push）其指标到一个中间服务，该服务随后会被 Prometheus 服务器拉取（scrape）。这主要解决了 Prometheus 原生的拉取（pull）模式在处理短期任务、批处理作业或不容易被动态发现的指标时的不足。

工作原理
1. 接收指标： Pushgateway 作为一个中间服务，它提供了一个 HTTP API，允许不同的服务和工作(job)通过 POST 请求推送 Metrics 至 Pushgateway。此时，推送的 Metric 数据会在 Pushgateway 中暂存。
2. 存储指标： Pushgateway 将接收到的指标暂存于内存中，并且会保持这些指标，直到它们被推送（覆盖）更新或是手动删除，确保即使原始工作(job)已经终止，其指标数据仍然可用。
3. 暴露指标： 存储在 Pushgateway 中的指标数据会通过一个端点 /metrics 暴露出来，该端点类似于 Prometheus 用于从其他服务拉取数据时所访问的端点。
4. 拉取指标： Prometheus 服务器通过配置中的抓取（scrape）配置定期地向 Pushgateway 发起 HTTP GET 请求到 /metrics 端点，从而拉取这些已经推送并暂存的指标。
5. 存储于 Prometheus： Prometheus 服务器拉取下来的指标会被存储起来并按照正常的方式进行处理。从这一刻起，这些指标数据就和从其他任何正常抓取目标得到的数据一样了。


组件角色
在这个过程中，每个组件的角色如下：
- 短期运行的工作或批处理作业负责在执行过程中计算它们的指标，并在执行完成之前推送这些指标到 Pushgateway。
- Pushgateway 起到临时存储的作用站，它暂时保存短期工作推送的指标数据，并使它们能被 Prometheus 服务器定期拉取。
- Prometheus 服务器 会定期从 Pushgateway 彩下标数据，就像它从其他抓取目标做的那样，区别仅仅是数据的来源变成了 Pushgateway 提供的端点- 


适用场景和限制
推送指标到 Pushgateway 非常适合于批处理作业或一些临时的工作，这些任务一旦完成他们就不再存在，因此无法通过 Prometheus 常规的拉取模式进行抓取。然而，使用 Pushgateway 并不适合作为长期运行工作的常规指标推送手段，并且 Pushgateway 也不支持删除已推送指标的自动过期和清理，这需要用户自己管理。
不过，建议尽量避免过度依赖 Pushgateway，因为过多地使用它可能会增加系统的复杂性并对 Pushgateway 产生较大压力，从而影响 Prometheus 执行监控的性能。只有当 Prometheus 的主动拉取模式不适用时，才考虑使用 Pushgateway。


# 6 Service Discovery

> 服务发现在Prometheus中是特别重要的一个部分，基于Pull模型的抓取方式，需要在Prometheus中配置大量的抓取节点信息才可以进行数据收集。有了服务发现后，用户通过服务发现和注册的工具对成百上千的节点进行服务注册，并最终将注册中心的地址配置在Prometheus的配置文件中，大大简化了配置文件的复杂程度，  
> 也可以更好的管理各种服务。 在众多云平台中（AWS,OpenStack），Prometheus可以  通过平台自身的API直接自动发现运行于平台上的各种服务，并抓取他们的信息Kubernetes掌握并管理着所有的容器以及服务信息，那此时Prometheus只需要与Kubernetes打交道就可以找到所有需要监控的容器以及服务对象.
> - Consul（官方推荐）等服务发现注册软件
> - 通过DNS进行服务发现
> - 通过静态配置文件（在服务节点规模不大的情况下）

它包含四种角色：

- node
- service
- pod
- endpoints

## 6.1 node

由于篇幅所限，这里只是简单介绍下其中的 node 还有 pod 角色

```
- job_name: 'kubernetes-nodes'
  scheme: https
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  
  kubernetes_sd_configs:
  - role: node
  
  relabel_configs:
    # 即从 __meta_kubernetes_node_label_<labelname> 这个配置中取出 labelname 以及 value
  - action: labelmap
    regex: __meta_kubernetes_node_label_(.+)
    
    # 配置 address 为 k8s api 的地址，相关的 ca 证书以及 token 在上面配置
  - target_label: __address__
    replacement: kubernetes.default.svc:443
    
    # 取出所有的 node，然后设置 /api/v1/nodes/<node_name>/proxy/metrics 为 metrics path
  - source_labels: 
    - __meta_kubernetes_node_name
    regex: (.+)
    target_label: __metrics_path__
    replacement: /api/v1/nodes/${1}/proxy/metrics
```



# 7 Prometheus工作流程

Prometheus server 定期从配置好的 jobs 或者 exporters 中拉取 metrics，或者从Pushgateway 拉取metrics，或者从其他的 Prometheus server 中拉 metrics。
Prometheus server 在本地存储收集到的 metrics，并运行已定义好的 alert.rules，通过一定规则进行清理和整理数据，并把得到的结果存储到新的时间序列中。记录新的时间序列或者向 Alertmanager 推送警报。
Prometheus通过PromQL和其他API可视化地展示收集的数据。Prometheus支持很多方式的图表可视化，例如Grafana、自带的Promdash以及自身提供的模版引擎等等。Prometheus还提供HTTP API的查询方式，自定义所需要的输出。


