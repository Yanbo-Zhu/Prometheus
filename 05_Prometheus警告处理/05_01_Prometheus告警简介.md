
告警能力在Prometheus的架构中被划分成两个独立的部分。如下所示，通过在Prometheus中定义AlertRule（告警规则），Prometheus会周期性的对告警规则进行计算，如果满足告警触发条件就会向Alertmanager发送告警信息。

 > 在Prometheus的报警系统中，是分为2个部分的， 规则是配置是在prometheus中的， prometheus组件完成报警推送给alertmanager的， alertmanager然后管理这些报警信息，包括静默、抑制、聚合和通过电子邮件、on-call通知系统和聊天平台等方法发送通知。

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LVMF4RtPS-2rjW9R-hG%252F-LPS9QhUbi37E1ZK8mXF%252Fprometheus-alert-artich.png%3Fgeneration%3D1546578333144123%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=57fb9de5&sv=1)

Prometheus告警处理

在Prometheus中一条告警规则主要由以下几部分组成：
- 告警名称：用户需要为告警规则命名，当然对于命名而言，需要能够直接表达出该告警的主要内容
- 告警规则：告警规则实际上主要由PromQL进行定义，其实际意义是当表达式（PromQL）查询结果持续多长时间（During）后出发告警


在Prometheus中，还可以通过Group（告警组）对一组相关的告警进行统一定义。当然这些定义都是通过YAML文件来统一管理的。

Alertmanager作为一个独立的组件，负责接收并处理来自Prometheus Server(也可以是其它的客户端程序)的告警信息。Alertmanager可以对这些告警信息进行进一步的处理，
- 去重复 Deduplication: 比如当接收到大量重复告警时能够消除重复的告警信息，
- 分组:  同时对告警信息进行分组并且路由到正确的通知方. 
- 通知： Prometheus内置了对邮件，Slack等多种通知方式的支持.   
- 定制化的通知: 同时还支持与Webhook的集成，以支持更多定制化的场景。例如，目前Alertmanager还不支持钉钉，那用户完全可以通过Webhook与钉钉机器人进行集成，从而通过钉钉接收告警信息。同时AlertManager还提供了静默和告警抑制机制来对告警通知行为进行优化。

# 1 警告的流程

当Alertmanager接收到来自Prometheus的告警消息后，会按照以下流程对告警进行处理：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LVSjRCOz_gMbNhBDIvI%252F-LPufOwoNbDOTMUcTl15%252Fam-notifi-pipeline.png%3Fgeneration%3D1546687209270858%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=688ae3e5&sv=1)

通知流水线

1. 在第一个阶段Silence中，Alertmanager会判断当前通知是否匹配到任何的静默规则，如果没有则进入下一个阶段，否则则中断流水线不发送通知。
2. 在第二个阶段Wait中，Alertmanager会根据当前Alertmanager在集群中所在的顺序(index)等待index * 5s的时间。
3. 当前Alertmanager等待阶段结束后，Dedup阶段则会判断当前Alertmanager数据库中该通知是否已经发送，如果已经发送则中断流水线，不发送告警，否则则进入下一阶段Send对外发送告警通知。
4. 告警发送完成后该Alertmanager进入最后一个阶段Gossip，Gossip会通知其他Alertmanager实例当前告警已经发送。其他实例接收到Gossip消息后，则会在自己的数据库中保存该通知已发送的记录。

# 2 Alertmanager特性

Alertmanager除了提供基本的告警通知能力以外，还主要提供了如：分组、抑制以及静默等告警特性：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LVMF4RtPS-2rjW9R-hG%252F-LPS9QhWQUhAIdLAiEfH%252Falertmanager-features.png%3Fgeneration%3D1546578326190496%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=a590ef0b&sv=1)

Alertmanager特性


## 2.1 分组

分组机制可以将详细的告警信息合并成一个通知。在某些情况下，比如由于系统宕机导致大量的告警被同时触发，在这种情况下分组机制可以将这些被触发的告警合并为一个告警通知，避免一次性接受大量的告警通知，而无法对问题进行快速定位。

例如，当集群中有数百个正在运行的服务实例，并且为每一个实例设置了告警规则。假如此时发生了网络故障，可能导致大量的服务实例无法连接到数据库，结果就会有数百个告警被发送到Alertmanager。

而作为用户，可能只希望能够在一个通知中中就能查看哪些服务实例收到影响。这时可以按照服务所在集群或者告警名称对告警进行分组，而将这些告警内聚在一起成为一个通知。

告警分组，告警时间，以及告警的接受方式可以通过Alertmanager的配置文件进行配置。


# 3 抑制

抑制是指当某一告警发出后，可以停止重复发送由此告警引发的其它告警的机制。
例如，当集群不可访问时触发了一次告警，通过配置Alertmanager可以忽略与该集群有关的其它所有告警。这样可以避免接收到大量与实际问题无关的告警通知。
抑制机制同样通过Alertmanager的配置文件进行设置。

# 4 静默

静默提供了一个简单的机制可以快速根据标签对告警进行静默处理。如果接收到的告警符合静默的配置，Alertmanager则不会发送告警通知。
静默设置需要在Alertmanager的Werb页面上进行设置。



