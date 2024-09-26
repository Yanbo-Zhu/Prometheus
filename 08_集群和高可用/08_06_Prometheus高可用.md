

Prometheus的本地存储给Prometheus带来了简单高效的使用体验，可以让Promthues在单节点的情况下满足大部分用户的监控需求。但是本地存储也同时限制了Prometheus的可扩展性，带来了数据持久化等一系列的问题。通过Prometheus的Remote Storage特性可以解决这一系列问题，包括Promthues的动态扩展，以及历史数据的存储。

而除了数据持久化问题以外，影响Promthues性能表现的另外一个重要因素就是数据采集任务量，以及单台Promthues能够处理的时间序列数。因此当监控规模大到Promthues单台无法有效处理的情况下，可以选择利用Promthues的联邦集群的特性，将Promthues的监控任务划分到不同的实例当中。

这一部分将重点讨论Prometheus的高可用架构，并且根据不同的使用场景介绍了一种常见的高可用方案。

# 1 基本HA：服务可用性

由于Promthues的Pull机制的设计，为了确保Promthues服务的可用性，用户只需要部署多套Prometheus Server实例，并且采集相同的Exporter目标即可。

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LVSVjCoJ2ZKnv7dYe6V%252F-LPufOd4LgRt7mUDTV8C%252Fpromethues-ha-01.png%3Fgeneration%3D1546683354194329%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=d194f329&sv=1)

![](https://img2018.cnblogs.com/blog/429277/201909/429277-20190930134826749-1610648523.png)

基本的HA模式只能确保Promthues服务的可用性问题，但是不解决Prometheus Server之间的数据一致性问题以及持久化问题(数据丢失后无法恢复)，也无法进行动态的扩展。因此这种部署方式适合监控规模不大，Promthues Server也不会频繁发生迁移的情况，并且只需要保存短周期监控数据的场景。


 这个架构可以保证服务的高可靠性，但是并不能解决多个prometheus实例之间的资料一致性问题，也无法数据进行长期存储，且单一实例无法负荷的时候，将延伸出性能瓶颈问题，因此这种架构适合小规模进行监控。

优点：

- 服务能够提供基本的可靠性
- 适合小规模监控，只需要短期存储。

缺点：

- 无法扩展
- 数据有不一致问题
- 无法长时间保持
- 当承载量过大时，单一prometheus无法负荷。

# 2 基本HA + 远程存储

在基本HA模式的基础上通过添加Remote Storage存储支持，将监控数据保存在第三方存储服务上。

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LVSVjCoJ2ZKnv7dYe6V%252F-LPufOd602YogyiIIadr%252Fprometheus-ha-remote-storage.png%3Fgeneration%3D1546683353605259%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=b3291789&sv=1)

![](https://img2018.cnblogs.com/blog/429277/201909/429277-20190930135544455-1821876227.png)

HA + Remote Storage

在解决了Promthues服务可用性的基础上，同时确保了数据的持久化，当Promthues Server发生宕机或者数据丢失的情况下，可以快速的恢复。 同时Promthues Server可能很好的进行迁移。因此，该方案适用于用户监控规模不大，但是希望能够将监控数据持久化，同时能够确保Promthues Server的可迁移性的场景。

该架构解决了数据持久性问题， 当prometheus server发生故障、重启的时候可以快速恢复数据，同时prometheus可以很好的进行迁移，但是这也只适合小规模的监测使用。

优点：

- 服务能够提供可靠性
- 适合小规模监测
- 数据能够持久化存储
- prometheus可以灵活迁移
- 能够得到数据还原

缺点：

- 不适合大规模监控
- 当承载量过大时，单一prometheus server无法负荷




# 3 基本HA + 远程存储 + 联邦集群

当单台Promthues Server无法处理大量的采集任务时，用户可以考虑基于Prometheus联邦集群的方式将监控采集任务划分到不同的Promthues实例当中即在任务级别功能分区。

服务高可靠性结合远端存储和联邦（基本ha + remote storage + federation）

这种架构主要是解决单一 prometheus server无法处理大量数据收集的问题，而且加强了prometheus的扩展性，通过将不同手机任务分割到不同的prometheus实力上去。

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LVSVjCoJ2ZKnv7dYe6V%252F-LPufOd8-X3EaOQhjM7L%252Fprometheus-ha-rs-fedreation.png%3Fgeneration%3D1546683353680741%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=5e38c187&sv=1)


这种部署方式一般适用于两种场景：

场景一：单数据中心 + 大量的采集任务

这种场景下Promthues的性能瓶颈主要在于大量的采集任务，因此用户需要利用Prometheus联邦集群的特性，将不同类型的采集任务划分到不同的Promthues子服务中，从而实现功能分区。例如一个Promthues Server负责采集基础设施相关的监控指标，另外一个Prometheus Server负责采集应用监控指标。再有上层Prometheus Server实现对数据的汇聚。

场景二：多数据中心

这种模式也适合与多数据中心的情况，当Promthues Server无法直接与数据中心中的Exporter进行通讯时，在每一个数据中部署一个单独的Promthues Server负责当前数据中心的采集任务是一个不错的方式。这样可以避免用户进行大量的网络配置，只需要确保主Promthues Server实例能够与当前数据中心的Prometheus Server通讯即可。 中心Promthues Server负责实现对多数据中心数据的聚合。



该架构通常有2种使用场景：
- 单一资料中心，但是有大量收集任务，这种场景行prometheus server 可能会发生性能上的瓶颈，主要是单一prometheus server 要承载大量资料书籍任务， 这个时候通过federation来将不同类型的任务分到不同的prometheus 子server 上， 再有上层完成资料聚合。
- 多资料中心， 在多资料中心下，这种架构也能够使用，当不同资料中心的exporter无法让最上层的prometheus 去拉取资料是， 就能通过federation来进行分层处理， 在每个资料中心建立一组收集该资料中心的prometheus server  ， 在由上层的prometheus 来进行抓取， 并且也能够依据每个收集任务的承载量来部署分级，但是需要确保上下层的prometheus server 是互通的。

优点
- 服务能够提供可靠性
- 资料能够被持久性保持在第三方存储系统中
- promethues server 能够迁移
- 能够得到资料还原
- 能够依据不同任务进行层级划分
- 适合不同规模监控
- 能够很好的扩展

缺点
- 部署架构负载
- 维护困难性增加
- 在kubernetes部署不易


# 4 按照实例进行功能分区

这时在考虑另外一种极端情况，即单个采集任务的Target数也变得非常巨大。这时简单通过联邦集群进行功能分区，Prometheus Server也无法有效处理时。这种情况只能考虑继续在实例级别进行功能划分。

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LVSVjCoJ2ZKnv7dYe6V%252F-LPufOdAlAf1-jLEzUN5%252Fpromethues-sharding-targets.png%3Fgeneration%3D1546683354120146%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=3d7981bf&sv=1)


如上图所示，将统一任务的不同实例的监控数据采集任务划分到不同的Prometheus实例。通过relabel设置，我们可以确保当前Prometheus Server只收集当前采集任务的一部分实例的监控指标。

```
global:
  external_labels:
    slave: 1  # This is the 2nd slave. This prevents clashes between slaves.
scrape_configs:
  - job_name: some_job
    relabel_configs:
    - source_labels: [__address__]
      modulus:       4
      target_label:  __tmp_hash
      action:        hashmod
    - source_labels: [__tmp_hash]
      regex:         ^1$
      action:        keep
```

并且通过当前数据中心的一个中心Prometheus Server将监控数据进行聚合到任务级别。

```
- scrape_config:
  - job_name: slaves
    honor_labels: true
    metrics_path: /federate
    params:
      match[]:
        - '{__name__=~"^slave:.*"}'   # Request all slave-level time series
    static_configs:
      - targets:
        - slave0:9090
        - slave1:9090
        - slave3:9090
        - slave4:9090
```


# 5 高可用方案选择

上面的部分，根据不同的场景演示了3种不同的高可用部署方案。当然对于Promthues部署方案需要用户根据监控规模以及自身的需求进行动态调整，下表展示了Promthues和高可用有关3个选项各自解决的问题，用户可以根据自己的需求灵活选择。

|   |   |   |   |
|---|---|---|---|
|选项\需求|服务可用性|数据持久化|水平扩展|
|主备HA|v|x|x|
|远程存储|x|v|x|
|联邦集群|x|x|v|


# 6 Prometheus Operator中的关于HA的备注 


https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/high-availability.md


## 6.1 Prometheus

To run Prometheus in a highly available manner, two (or more) instances need to be running with the same configuration except that they will have one external label with a different value to identify them. The Prometheus instances scrape the same targets and evaluate the same rules, hence they will have the same data in memory and on disk, with a slight twist that given their different external label, the scrapes and evaluations won't happen at exactly the same time. As a consequence, query requests executed against each Prometheus instance may return slightly different results. 
For alert evaluation this situation does not change anything, as alerts are typically only fired when a certain query triggers for a period of time. (Deduplication of alerts is happening on AlertManager side. So having two or more prometheuses provides only high availability, and not scaling.)
For dashboarding, sticky sessions (using `sessionAffinity` on the Kubernetes `Service`) should be used, to get consistent graphs when refreshing or you can use something like [Thanos Querier](https://thanos.io/tip/components/query.md/) to federate the data.  (As for using Grafana or any other visualization tool - you should configure sticky sessions, to query single instance at a time )


-----光有对多个Prometheus Instance 还不够，还需要用 Prometheus' sharding feature
Running multiple Prometheus instances avoids having a single point of failure but it doesn't help scaling out Prometheus in case a single Prometheus instance can't handle all the targets and rules. This is where Prometheus' sharding feature comes into play. Sharding aims at splitting the scrape targets into multiple groups, each assigned to one Prometheus shard and small enough that they can be handled by a single Prometheus instance. 
If possible,** functional sharding** is recommended: in this case, the Prometheus shard X scrapes all pods of Service A, B and C while shard Y scrapes pods from Service D, E and F. 
When functional sharding is not possible, the Prometheus Operator is also able to support **automatic sharding:** the targets will be assigned to Prometheus shards based on their addresses. 
The main drawback of this solution is the additional complexity: to query all data, query federation (e.g. Thanos Query) and distributed rule evaluation engine (e.g. Thanos Ruler) should be deployed to fan in the relevant data for queries and rule evaluations. Single shards of Prometheus can be run highly available as described before.

------Prometheus' sharding feature 进一步解释： 
就是一个 Prometheus instance 只负责收集来自于某几个service的metrics。  每个 group of targets to scrape is assigned to one Prometheus shard. 一个  Prometheus shard 可以近似于理解成为 a single prometheus isntance 。  然后 再将的 Prometheus instance 各自的metrics 通过 feradetation 输出 到一个总的prometheus server. 
这样做的做的目的是可以 按照规则 seperate the load into serveal different prometheus。 
To handle large amounts of data, the load can be distributed across multiple Prometheus instances, a technique called sharding. One common method of sharding is by service, where each Prometheus instance is responsible for collecting metrics from a subset of all services.


One of the goals with the Prometheus Operator is that we want to completely automate sharding and federation. We are currently implementing some of the groundwork to make this possible, and figuring out the best approach to do so, but it is definitely on the roadmap!

## 6.2 Alertmanager

To ensure high-availability of the Alertmanager service, Prometheus instances are configured to send their alerts to all configured Alertmanager instances (as described in the [Alertmanager documentation](https://prometheus.io/docs/alerting/latest/alertmanager/#high-availability)). The Alertmanager instances creates a gossip-based cluster to replicate alert silences and notification logs.

The Prometheus Operator manages the following configuration
- Alertmanager discovery using the Kubernetes API for Prometheus.
- Highly-available cluster for Alertmanager when replicas > 1.

## 6.3 Exporters

For exporters, high availability depends on the particular exporter. In the case of [`kube-state-metrics`](https://github.com/kubernetes/kube-state-metrics), because it is effectively stateless, it is the same as running any other stateless service in a highly available manner. Simply run multiple replicas that are being load balanced. Key for this is that the backing service, in this case the Kubernetes API server is highly available, ensuring that the data source of `kube-state-metrics` is not a single point of failure.
