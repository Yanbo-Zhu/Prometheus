
alertmanager可以配置成集群模式，即多个alaertmanager一起运行，彼此之间通过gossip协议获知告警的处理状态，防止告警重复发出。
这种模式通常用在prometheus需要做高可用的场景中。
[prometheus ha deploy](http://ylzheng.com/2018/03/17/promethues-ha-deploy/ "prometheus ha deploy")的高可用部署通常至少会有两套prometheus独立工作，它们会执行各自的告警检查。
与之相伴的通常也要部署多个alaertmanager，这时候这些alertmanager之间就需要同步信息，防止告警重复发出。
由于使用的是gossip协议，alermanager的集群模式配置很简单，只需要启动时指定另一个或多个alertmanager的地址即可：

--cluster.peer=192.168.88.10:9094


