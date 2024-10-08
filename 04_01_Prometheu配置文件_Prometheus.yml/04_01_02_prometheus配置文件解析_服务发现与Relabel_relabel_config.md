
在本章的前几个小节中笔者已经分别介绍了Prometheus的几种服务发现机制。通过服务发现的方式，管理员可以在不重启Prometheus服务的情况下动态的发现需要监控的Target实例信息。

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPui3iWsSYuoIaPTZtS%252F-LPui6XQkFRxffCHtkRm%252Fbolg_sd_mutil_cluster.png%3Fgeneration%3D1540730946142415%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=7e16d859&sv=1)

基于Consul的服务发现

如上图所示，对于线上环境我们可能会划分为:dev, stage, prod不同的集群。每一个集群运行多个主机节点，每个服务器节点上运行一个Node Exporter实例。Node Exporter实例会自动注册到Consul中，而Prometheus则根据Consul返回的Node Exporter实例信息动态的维护Target列表，从而向这些Target轮询监控数据。

然而，如果我们可能还需要：
- 按照不同的环境dev, stage, prod聚合监控数据？
- 对于研发团队而言，我可能只关心dev环境的监控数据，如何处理？
- 如果为每一个团队单独搭建一个Prometheus Server。那么如何让不同团队的Prometheus Server采集不同的环境监控数据？

面对以上这些场景下的需求时，我们实际上是希望Prometheus Server能够按照某些规则（比如标签）从服务发现注册中心返回的Target实例中有选择性的采集某些Exporter实例的监控数据。

接下来，我们将学习如何通过Prometheus强大的Relabel机制来实现以上这些具体的目标。


# 1 更改标签的时机：抓取前修改、抓取后修改、告警时修改

prometheus支持修改标签。metric的标签可以在采集端采集的时候直接打上，这是最原始的标签。

除此之外，还可以在prometheus的配置文件里，对metric的label进行修改。

修改的时机有两个：采集数据之前，通过relabel_config；采集数据之后，写入存储之前，通过metric_relabel_configs。

两个的配置方式是相同的：

```
relabel_configs:
- source_labels: [__meta_kubernetes_pod_label_app]
  regex: 'rabbitmq01-exporter'
  replacement: 'public-rabbitmq01.paas.production:5672'
  target_label: instance
metric_relabel_configs:
- source_labels: [node]
  regex: 'rabbit01@rabbit01'
  replacement: 'public-rabbitmq01.paas.production:5672'
  target_label: node_addr
```

第一个是采集之前通过已有的标签，采集之前的标签通常是服务发现时设置的，生成新的标签instance。
第一个是采集之后，检查采集的指标，如果标签`node`匹配正则，生成新的标签node_addr。

如果要修改标签，target_label指定同名的标签。
另外`alert_relabel_configs`可以在告警前修改标签

# 2 Prometheus的Relabeling机制

> ==就是复写 某个label 的value. 从 target 上采集到的 信息, 某个数据上有label, 复写他的值 ==

> 它的作用是 Prometheus 抓取 metrics 之前，就将对象相关的 labels 重写。下面是它几个重要的 label


> 重新标记是一个功能强大的工具，可以在目标的标签集被抓取之前重写它，每个采集配置可以配置多个重写标签设置，并按照配置的顺序来应用于每个目标的标签集。
> 目标重新标签之后，以__开头的标签将从标签集中删除的。
> 如果使用只需要临时的存储临时标签值的，可以使用_tmp作为前缀标识。

在Prometheus所有的Target实例中，都包含一些默认的Metadata标签信息。可以通过Prometheus UI的Targets页面中查看这些实例的Metadata标签的内容：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPui3iWsSYuoIaPTZtS%252F-LPui6XS6SgDzCvoGrMw%252Fprometheus_file_target_metadata.png%3Fgeneration%3D1540730946055793%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=dd1b4db4&sv=1)

实例的Metadata信息

默认情况下，当Prometheus加载Target实例完成后，这些Target时候都会包含一些默认的标签：
- `__address__`：当前Target实例的访问地址`<host>:<port>`, 默认为 host:port，也是之后抓取之后 instance 的值；
- `__scheme__`：采集目标服务访问地址的HTTP Scheme，HTTP或者HTTPS
- `__metrics_path__`：采集目标服务访问地址的访问路径, 就是 metrics path，默认为 /metrics；
- `__param_<name>`：采集任务目标服务的中包含的请求参数, 用来作为 URL parameter，比如 http://…/metrics?name=value；
- `__meta_`：这个开头的配置都是 SD 相关的配置；


上面这些标签将会告诉Prometheus如何从该Target实例中获取监控数据。除了这些默认的标签以外，我们还可以为Target添加自定义的标签，例如，在“基于文件的服务发现”小节中的示例中，我们通过JSON配置文件，为Target实例添加了自定义标签env，如下所示该标签最终也会保存到从该实例采集的样本数据中：

```
node_cpu{cpu="cpu0",env="prod",instance="localhost:9100",job="node",mode="idle"}
```

一般来说，Target以`__`作为前置的标签是在系统内部使用的，因此这些标签不会被写入到样本数据中。不过这里有一些例外，例如，我们会发现所有通过Prometheus采集的样本数据中都会包含一个名为instance的标签，该标签的内容对应到Target实例的`__address__`。 这里实际上是发生了一次标签的重写处理。

这种发生在采集样本数据之前，对Target实例的标签进行重写的机制在Prometheus被称为Relabeling。

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LVSsqoyc3erej7bf3vS%252F-LPui6XUJUk0y4kPPia0%252Fwhen-relabel-work.png%3Fgeneration%3D1546689677870578%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=f22b2585&sv=1)

Relabeling作用时机
Prometheus允许用户在采集任务设置中通过relabel_configs来添加自定义的Relabeling过程。


# 3 relabel的action类型

- replace: 对标签和标签值进行替换。
- keep: 满足特定条件的实例进行采集，其他的不采集。
- drop： 满足特定条件的实例不采集，其他的采集。
- hashmod： 这个我也没看懂啥意思，囧。
- labelmap： 这个我也没看懂啥意思，囧。
- labeldrop： 对抓取的实例特定标签进行删除。
- labelkeep：  对抓取的实例特定标签进行保留，其他标签删除。


# 4 这个例子非常好 

## 4.1 在测试前，同步下配置文件如下。


```yaml
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
  - job_name: "node"
    file_sd_configs:
    - refresh_interval: 1m
      files: 
      - "/usr/local/prometheus/prometheus/conf/node*.yml"  
  
  
[root@node00 prometheus]# cat conf/node-dis.yml   
- targets:   
  - "192.168.100.10:20001"  
  labels:   
    __hostname__: node00  
    __businees_line__: "line_a"  
    __region_id__: "cn-beijing"  
    __availability_zone__: "a"  
- targets:   
  - "192.168.100.11:20001"  
  labels:   
    __hostname__: node01  
    __businees_line__: "line_a"  
    __region_id__: "cn-beijing"  
    __availability_zone__: "a"  
- targets:   
  - "192.168.100.12:20001"  
  labels:   
    __hostname__: node02  
    __businees_line__: "line_c"  
    __region_id__: "cn-beijing"  
    __availability_zone__: "b"
```

此时如果查看target信息，如下图。

![](https://img2018.cnblogs.com/blog/429277/201909/429277-20190926161559582-1893596710.png)

> 因为我们的label都是以__开头的，目标重新标签之后，以__开头的标签将从标签集中删除的。

## 4.2 replace 

将labels中的__hostname__替换为node_name。

```yaml
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
  - job_name: "node"
    file_sd_configs:
    - refresh_interval: 1m
      files:
      - "/usr/local/prometheus/prometheus/conf/node*.yml"
    relabel_configs:
    - source_labels:
      - "__hostname__"
      regex: "(.*)"
      target_label: "nodename"
      action: replace
      replacement: "$1"


[root@node00 prometheus]# cat conf/node-dis.yml   
- targets:   
  - "192.168.100.10:20001"  
  labels:   
    __hostname__: node00  
    __businees_line__: "line_a"  
    __region_id__: "cn-beijing"  
    __availability_zone__: "a"  
- targets:   
  - "192.168.100.11:20001"  
  labels:   
    __hostname__: node01  
    __businees_line__: "line_a"  
    __region_id__: "cn-beijing"  
    __availability_zone__: "a"  
- targets:   
  - "192.168.100.12:20001"  
  labels:   
    __hostname__: node02  
    __businees_line__: "line_c"  
    __region_id__: "cn-beijing"  
    __availability_zone__: "b"
```

重启服务查看target信息如下图：
![](https://img2018.cnblogs.com/blog/429277/201909/429277-20190926161834737-1993819792.png)


说下上面的配置： 
- source_labels指定我们我们需要处理的源标签， 
- target_labels指定了我们要replace后的标签名字， 
- action指定relabel动作，这里使用replace替换动作。
- regex去匹配源标签（__hostname__）的值，`"(.*)"`代表__hostname__这个标签是什么值都匹配的，
- 然后replacement指定的替换后的标签（target_label）对应的数值。采用正则引用方式获取的。$1 就是匹配到的第一项 



这里修改下上面的正则表达式为 ‘’regex: "(node00)"'的时候可以看到如下图。
![](https://img2018.cnblogs.com/blog/429277/201909/429277-20190926162538277-481970739.png)


## 4.3 replace: 融合2个字段

我们的基础信息里面有__region_id__和__availability_zone__，但是我想融合2个字段在一起，可以通过replace来实现。
修改配置如下

```yaml
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
  - job_name: "node"
    file_sd_configs:
    - refresh_interval: 1m
      files:
      - "/usr/local/prometheus/prometheus/conf/node*.yml"
    relabel_configs:
    - source_labels:
      - "__region_id__"
      - "__availability_zone__"
      separator: "-"
      regex: "(.*)"
      target_label: "region_zone"
      action: replace
      replacement: "$1"

[root@node00 prometheus]# cat conf/node-dis.yml   
- targets:   
  - "192.168.100.10:20001"  
  labels:   
    __hostname__: node00  
    __businees_line__: "line_a"  
    __region_id__: "cn-beijing"  
    __availability_zone__: "a"  
- targets:   
  - "192.168.100.11:20001"  
  labels:   
    __hostname__: node01  
    __businees_line__: "line_a"  
    __region_id__: "cn-beijing"  
    __availability_zone__: "a"  
- targets:   
  - "192.168.100.12:20001"  
  labels:   
    __hostname__: node02  
    __businees_line__: "line_c"  
    __region_id__: "cn-beijing"  
    __availability_zone__: "b"
```


![](https://img2018.cnblogs.com/blog/429277/201909/429277-20190926170113749-1703501565.png)



## 4.4 keep

修改配置文件

```yaml
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
  - job_name: "node"
    file_sd_configs:
    - refresh_interval: 1m
      files:
      - "/usr/local/prometheus/prometheus/conf/node*.yml"

[root@node00 prometheus]# cat conf/node-dis.yml   
- targets:   
  - "192.168.100.10:20001"  
  labels:   
    __hostname__: node00  
    __businees_line__: "line_a"  
    __region_id__: "cn-beijing"  
    __availability_zone__: "a"  
- targets:   
  - "192.168.100.11:20001"  
  labels:   
    __hostname__: node01  
    __businees_line__: "line_a"  
    __region_id__: "cn-beijing"  
    __availability_zone__: "a"  
- targets:   
  - "192.168.100.12:20001"  
  labels:   
    __hostname__: node02  
    __businees_line__: "line_c"  
    __region_id__: "cn-beijing"  
    __availability_zone__: "b"
```

target如下图
![](https://img2018.cnblogs.com/blog/429277/201909/429277-20190926164900640-1818271234.png)




修改配置文件如下
```yaml
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
  - job_name: "node"
    file_sd_configs:
    - refresh_interval: 1m
      files: 
      - "/usr/local/prometheus/prometheus/conf/node*.yml"
    relabel_configs:
    - source_labels:
      - "__hostname__"
      regex: "node00"
      action: keep

[root@node00 prometheus]# cat conf/node-dis.yml   
- targets:   
  - "192.168.100.10:20001"  
  labels:   
    __hostname__: node00  
    __businees_line__: "line_a"  
    __region_id__: "cn-beijing"  
    __availability_zone__: "a"  
- targets:   
  - "192.168.100.11:20001"  
  labels:   
    __hostname__: node01  
    __businees_line__: "line_a"  
    __region_id__: "cn-beijing"  
    __availability_zone__: "a"  
- targets:   
  - "192.168.100.12:20001"  
  labels:   
    __hostname__: node02  
    __businees_line__: "line_c"  
    __region_id__: "cn-beijing"  
    __availability_zone__: "b"
```

target如下图
![](https://img2018.cnblogs.com/blog/429277/201909/429277-20190926165036479-1856173350.png)


action为keep，只要source_labels的值匹配regex（node00）的实例才能会被采集。 其他的实例不会被采集。

## 4.5 drop

在上面的基础上，修改action为drop。

target如下图
![](https://img2018.cnblogs.com/blog/429277/201909/429277-20190926165707447-617515324.png)

action为drop，其实和keep是相似的， 不过是相反的， 只要source_labels的值匹配regex（node00）的实例不会被采集。 其他的实例会被采集。


## 4.6 labelkeep 


先加上一些标签
```yaml

scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
  - job_name: "node"
    file_sd_configs:
    - refresh_interval: 1m
      files:
      - "/usr/local/prometheus/prometheus/conf/node*.yml"
    relabel_configs:
    - source_labels:
      - "__hostname__"
      regex: "(.*)"
      target_label: "nodename"
      action: replace
      replacement: "$1"
    - source_labels:
      - "__businees_line__"
      regex: "(.*)"
      target_label: "businees_line"
      action: replace
      replacement: "$1"
    - source_labels:
      - "__datacenter__"
      regex: "(.*)"
      target_label: "datacenter"
      action: replace
      replacement: "$1"

[root@node00 prometheus]# cat conf/node-dis.yml   
- targets:   
  - "192.168.100.10:20001"  
  labels:   
    __hostname__: node00  
    __businees_line__: "line_a"  
    __region_id__: "cn-beijing"  
    __availability_zone__: "a"  
- targets:   
  - "192.168.100.11:20001"  
  labels:   
    __hostname__: node01  
    __businees_line__: "line_a"  
    __region_id__: "cn-beijing"  
    __availability_zone__: "a"  
- targets:   
  - "192.168.100.12:20001"  
  labels:   
    __hostname__: node02  
    __businees_line__: "line_c"  
    __region_id__: "cn-beijing"  
    __availability_zone__: "b"


```

![](https://img2018.cnblogs.com/blog/429277/201909/429277-20190926171526997-33997893.png)




 修改配置文件如下
```yaml
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
  - job_name: "node"
    file_sd_configs:
    - refresh_interval: 1m
      files:
      - "/usr/local/prometheus/prometheus/conf/node*.yml"
    relabel_configs:
    - source_labels:
      - "__hostname__"
      regex: "(.*)"
      target_label: "nodename"
      action: replace
      replacement: "$1"
    - source_labels:
      - "__businees_line__"
      regex: "(.*)"
      target_label: "businees_line"
      action: replace
      replacement: "$1"
    - source_labels:
      - "__datacenter__"
      regex: "(.*)"
      target_label: "datacenter"
      action: replace
      replacement: "$1"
    - regex: "(nodename|datacenter)"   ### 注意这里 
      action: labeldrop
```

![](https://img2018.cnblogs.com/blog/429277/201909/429277-20190926171633513-734348179.png)


# 5 重写标签

Relabeling最基本的应用场景就是基于Target实例中包含的metadata标签，动态的添加或者覆盖标签。例如，通过Consul动态发现的服务实例还会包含以下Metadata标签信息：
```

- __meta_consul_address：consul地址
- __meta_consul_dc：consul中服务所在的数据中心
- __meta_consulmetadata：服务的metadata
- __meta_consul_node：服务所在consul节点的信息
- __meta_consul_service_address：服务访问地址
- __meta_consul_service_id：服务ID
- __meta_consul_service_port：服务端口
- __meta_consul_service：服务名称
- __meta_consul_tags：服务包含的标签信息
 
```

在默认情况下，从Node Exporter实例采集上来的样本数据如下所示：
```
node_cpu{cpu="cpu0",instance="localhost:9100",job="node",mode="idle"} 93970.8203125
```


我们希望能有一个额外的标签dc可以表示该样本所属的数据中心：
```
node_cpu{cpu="cpu0",instance="localhost:9100",job="node",mode="idle", dc="dc1"} 93970.8203125
```


在每一个采集任务的配置中可以添加多个relabel_config配置，一个最简单的relabel配置如下：
```
scrape_configs:
  - job_name: node_exporter
    consul_sd_configs:
      - server: localhost:8500
        services:
          - node_exporter
    relabel_configs:
    - source_labels:  ["__meta_consul_dc"]
      target_label: "dc"
```

该采集任务通过Consul动态发现Node Exporter实例信息作为监控采集目标。在上一小节中，我们知道通过Consul动态发现的监控Target都会包含一些额外的Metadata标签，比如标签__meta_consul_dc表明了当前实例所在的Consul数据中心，因此我们希望从这些实例中采集到的监控样本中也可以包含这样一个标签，例如：

```
node_cpu{cpu="cpu0",dc="dc1",instance="172.21.0.6:9100",job="consul_sd",mode="guest"}
```

这样可以方便的根据dc标签的值，根据不同的数据中心聚合分析各自的数据。

## 5.1 replace 

在这个例子中，通过从Target实例中获取__meta_consul_dc的值，并且重写所有 从该实例获取的样本中。

完整的relabel_config配置如下所示：

```
# The source labels select values from existing labels. Their content is concatenated
# using the configured separator and matched against the configured regular expression
# for the replace, keep, and drop actions.
[ source_labels: '[' <labelname> [, ...] ']' ]

# Separator placed between concatenated source label values.
[ separator: <string> | default = ; ]

# Label to which the resulting value is written in a replace action.
# It is mandatory for replace actions. Regex capture groups are available.
[ target_label: <labelname> ]

# Regular expression against which the extracted value is matched.
[ regex: <regex> | default = (.*) ]

# Modulus to take of the hash of the source label values.
[ modulus: <uint64> ]

# Replacement value against which a regex replace is performed if the
# regular expression matches. Regex capture groups are available.
[ replacement: <string> | default = $1 ]

# Action to perform based on regex matching.
[ action: <relabel_action> | default = replace ]
```

其中action定义了当前relabel_config对Metadata标签的处理方式，默认的action行为为replace。 replace行为会根据regex的配置匹配source_labels标签的值（多个source_label的值会按照separator进行拼接），并且将匹配到的值写入到target_label当中，如果有多个匹配组，则可以使用${1}, ${2}确定写入的内容。如果没匹配到任何内容则不对target_label进行重新。

repalce操作允许用户根据Target的Metadata标签重写或者写入新的标签键值对，在多环境的场景下，可以帮助用户添加与环境相关的特征维度，从而可以更好的对数据进行聚合。




## 5.2 labelmap

除了使用replace以外，还可以定义action的配置为labelmap。与replace不同的是，labelmap会根据regex的定义去匹配Target实例所有标签的名称，并且以匹配到的内容为新的标签名称，其值作为新标签的值。

例如，在监控Kubernetes下所有的主机节点时，为将这些节点上定义的标签写入到样本中时，可以使用如下relabel_config配置：

```
- job_name: 'kubernetes-nodes'
  kubernetes_sd_configs:
  - role: node
  relabel_configs:
  - action: labelmap
    regex: __meta_kubernetes_node_label_(.+)
```

## 5.3 labeldrop and labelkeep

而使用labelkeep或者labeldrop则可以对Target标签进行过滤，仅保留符合过滤条件的标签，例如：

```
relabel_configs:
  - regex: label_should_drop_(.+)
    action: labeldrop
```

该配置会使用regex匹配当前Target实例的所有标签，并将符合regex规则的标签从Target实例中移除。labelkeep正好相反，会移除那些不匹配regex定义的所有标签。



# 6 使用keep/drop过滤Target实例

在上一部分中我们介绍了Prometheus的Relabeling机制，并且使用了replace/labelmap/labelkeep/labeldrop对标签进行管理。

而本节开头还提到过第二个问题，使用中心化的服务发现注册中心时，所有环境的Exporter实例都会注册到该服务发现注册中心中。而不同职能（开发、测试、运维）的人员可能只关心其中一部分的监控数据，他们可能各自部署的自己的Prometheus Server用于监控自己关心的指标数据，如果让这些Prometheus Server采集所有环境中的所有Exporter数据显然会存在大量的资源浪费。
如何让这些不同的Prometheus Server采集各自关心的内容？答案还是Relabeling，relabel_config的action除了默认的replace以外，还支持keep/drop行为。例如，如果我们只希望采集数据中心dc1中的Node Exporter实例的样本数据，那么可以使用如下配置：

```
scrape_configs:
  - job_name: node_exporter
    consul_sd_configs:
      - server: localhost:8500
        services:
          - node_exporter
    relabel_configs:
    - source_labels:  ["__meta_consul_dc"]
      regex: "dc1"
      action: keep
```

当action设置为keep时，Prometheus会丢弃source_labels的值中没有匹配到regex正则表达式内容的Target实例，
而当action设置为drop时，则会丢弃那些source_labels的值匹配到regex正则表达式内容的Target实例。
可以简单理解为keep用于选择，而drop用于排除。



# 7 使用hashmod计算source_labels的Hash值

当relabel_config设置为hashmod时，Prometheus会根据modulus的值作为系数，计算source_labels值的hash值。例如：

```
scrape_configs
- job_name: 'file_ds'
  relabel_configs:
    - source_labels: [__address__]
      modulus:       4
      target_label:  tmp_hash
      action:        hashmod
  file_sd_configs:
  - files:
    - targets.json
```

根据当前Target实例`__address__`的值以4作为系数，这样每个Target实例都会包含一个新的标签tmp_hash，并且该值的范围在1~4之间，查看Target实例的标签信息，可以看到如下的结果，每一个Target实例都包含了一个新的tmp_hash值：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPui3iWsSYuoIaPTZtS%252F-LPui6XXXzSeGsZU_nk_%252Frelabel_hash_mode.png%3Fgeneration%3D1540730947209633%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=42d8a52&sv=1)

计算Hash值

在第6章的“Prometheus高可用”小节中，正是利用了Hashmod的能力在Target实例级别实现对采集任务的功能分区的:

```
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

这里需要注意的是，如果relabel的操作只是为了产生一个临时变量，以作为下一个relabel操作的输入，那么我们可以使用`__tmp`作为标签名的前缀，通过该前缀定义的标签就不会写入到Target或者采集到的样本的标签中。

`__tmp_hash` 为1 的 数据 才会被 留下, , `__tmp_hash` 其他值的 数据, 都被抛弃了 



# 8 例子

在测试前，同步下配置文件如下。

```
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
  - job_name: "node"
    file_sd_configs:
    - refresh_interval: 1m
      files: 
      - "/usr/local/prometheus/prometheus/conf/node*.yml"[root@node00 prometheus]# cat conf/node-dis.yml - targets:   - "192.168.100.10:20001"  labels:     __hostname__: node00    __businees_line__: "line_a"    __region_id__: "cn-beijing"    __availability_zone__: "a"- targets:   - "192.168.100.11:20001"  labels:     __hostname__: node01    __businees_line__: "line_a"    __region_id__: "cn-beijing"    __availability_zone__: "a"- targets:   - "192.168.100.12:20001"  labels:     __hostname__: node02    __businees_line__: "line_c"    __region_id__: "cn-beijing"    __availability_zone__: "b"
```

此时如果查看target信息，如下图。
![](https://img2018.cnblogs.com/blog/429277/201909/429277-20190926161559582-1893596710.png)


因为我们的label都是以__开头的，目标重新标签之后，以__开头的标签将从标签集中删除的。

