

在上一小节中，我们已经看到了类似于irate()这样的函数，可以帮助我们计算监控指标的增长率。除了irate以外，Prometheus还提供了其它大量的内置函数，可以对时序数据进行丰富的处理。本小节将带来读者了解一些常用的内置函数以及相关的使用场景和用法。

rometheus 提供了其它大量的内置函数，可以对时序数据进行丰富的处理。某些函数有默认的参数，例如：`year(v=vector(time()) instant-vector)`。其中参数 `v` 是一个瞬时向量，如果不提供该参数，将使用默认值 `vector(time())`。instant-vector 表示参数类型。

# 1 计算Counter指标增长率

我们知道Counter类型的监控指标其特点是只增不减，在没有发生重置（如服务器重启，应用重启）的情况下其样本值应该是不断增大的。为了能够更直观的表示样本数据的变化剧烈情况，需要计算样本的增长速率。

如下图所示，样本增长率反映出了样本变化的剧烈程度：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LVT4hCAm7HWaP8rOjeF%252F-LPS8MxTnlks9MW9afcf%252Fcounter-to-rate.png%3Fgeneration%3D1546693045775622%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=1a20f247&sv=1)

通过增长率表示样本的变化情况

## 1.1 increase(v range-vector)函数

增长量 不是 增长率， 增长量是单纯增加的个数 

increase(v range-vector)函数是PromQL中提供的众多内置函数之一。其中参数v是一个区间向量，increase函数获取区间向量中的第一个后最后一个样本并返回其增长量。因此，可以通过以下表达式Counter类型指标的增长率：

```
increase(node_cpu[2m]) / 120
```

这里通过node_cpu[2m]获取时间序列最近两分钟的所有样本，increase计算出最近两分钟的增长量，最后除以时间120秒得到node_cpu样本在最近两分钟的平均增长率。并且这个值也近似于主机节点最近两分钟内的平均CPU使用率。


![](https://img2022.cnblogs.com/blog/1354564/202205/1354564-20220522221954188-589273202.png)



## 1.2 rate(v range-vector)函数



除了使用increase函数以外，PromQL中还直接内置了rate(v range-vector)函数，rate函数可以直接计算区间向量v在时间窗口内平均增长速率。因此，通过以下表达式可以得到与increase函数相同的结果：

```
rate(node_cpu[2m])
```

需要注意的是使用rate或者increase函数去计算样本的平均增长速率，容易陷入“长尾问题”当中，其无法反应在时间窗口内样本数据的突发变化。 例如，对于主机而言在2分钟的时间窗口内，可能在某一个由于访问量或者其它问题导致CPU占用100%的情况，但是通过计算在时间窗口内的平均增长率却无法反应出该问题。


## 1.3 例子

比如节点的CPU执行时间，就是Counter单增的量。可以看到，这些数据就是CPU的时间片统计，会不断递增。输入下面的语句：
node_cpu_seconds_total


可以看到各个CPU，以及不同的mode的数据，我们过滤选择第0个CPU，以及mode为user。这时候过滤出来的是CPU0的user的累计时间片：
node_cpu_seconds_total{cpu="0",mode="user"}


我们可以用rate函数计算增量的变化率，也就是user的增量的随时间的变化：
rate(node_cpu_seconds_total{cpu="0",mode="user"}[10s])

- `rate(node_cpu_seconds_total{cpu="0",mode="user"}[10s])` 返回 `0.162`
- `increase(node_cpu_seconds_total{cpu="0",mode="user"}[10s])` 返回 `1.62`
可以看到1.62 / 10s = 0.162，也就是rate=increase/duration，取的是增量和时间的比。
>    Note: 注意如果采样是1m，那么时间范围就不能小于1m，否则会出现Empty query result。


# 2 irate(v range-vector)


为了解决该问题，PromQL提供了另外一个灵敏度更高的函数irate(v range-vector)。irate同样用于计算区间向量的计算率，但是其反应出的是瞬时增长率。irate函数是通过区间向量中最后两个样本数据来计算区间向量的增长速率。这种方式可以避免在时间窗口范围内的“长尾问题”，并且体现出更好的灵敏度，通过irate函数绘制的图标能够更好的反应样本数据的瞬时变化状态。

```
irate(node_cpu[2m])
```

irate函数相比于rate函数提供了更高的灵敏度，不过当需要分析长期趋势或者在告警规则中，irate的这种灵敏度反而容易造成干扰。因此在长期趋势分析或者告警中更推荐使用rate函数。

---

什么是区间向量 
```
http_request_total{} # 瞬时向量表达式，选择当前最新的数据
http_request_total{}[5m] # 区间向量表达式，选择以当前时间为基准，5分钟内的数据
```


什么是长尾问题
在大多数情况下人们都倾向于使用某些量化指标的平均值，例如CPU的平均使用率、页面的平均响应时间。这种方式的问题很明显，以系统API调用的平均响应时间为例：如果大多数API请求都维持在100ms的响应时间范围内，而个别请求的响应时间需要5s，那么就会导致某些WEB页面的响应时间落到中位数的情况，而这种现象被称为长尾问题。


# 3 预测Gauge指标变化趋势 predict_linear(v range-vector, t scalar)

在一般情况下，系统管理员为了确保业务的持续可用运行，会针对服务器的资源设置相应的告警阈值。例如，当磁盘空间只剩512MB时向相关人员发送告警通知。 这种基于阈值的告警模式对于当资源用量是平滑增长的情况下是能够有效的工作的。 但是如果资源不是平滑变化的呢？ 比如有些某些业务增长，存储空间的增长速率提升了高几倍。这时，如果基于原有阈值去触发告警，当系统管理员接收到告警以后可能还没来得及去处理问题，系统就已经不可用了。 因此阈值通常来说不是固定的，需要定期进行调整才能保证该告警阈值能够发挥去作用。 那么还有没有更好的方法吗？

PromQL中内置的predict_linear(v range-vector, t scalar) 函数可以帮助系统管理员更好的处理此类情况，predict_linear函数可以预测时间序列v在t秒后的值。它基于简单线性回归的方式，对时间窗口内的样本数据进行统计，从而可以对时间序列的变化趋势做出预测。例如，基于2小时的样本数据，来预测主机可用磁盘空间的是否在4个小时候被占满，可以使用如下表达式：

```
predict_linear(node_filesystem_free{job="node"}[2h], 4 * 3600) < 0
```



# 4 统计Histogram指标的分位数

在本章的第2小节中，我们介绍了Prometheus的四种监控指标类型，其中Histogram和Summary都可以用于统计和分析数据的分布情况。区别在于Summary是直接在客户端计算了数据分布的分位数情况。而Histogram的分位数计算需要通过histogram_quantile(φ float, b instant-vector)函数进行计算。其中φ（0<φ<1）表示需要计算的分位数，如果需要计算中位数φ取值为0.5，以此类推即可。

以指标http_request_duration_seconds_bucket为例：

```
# HELP http_request_duration_seconds request duration histogram
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.5"} 0
http_request_duration_seconds_bucket{le="1"} 1
http_request_duration_seconds_bucket{le="2"} 2
http_request_duration_seconds_bucket{le="3"} 3
http_request_duration_seconds_bucket{le="5"} 3
http_request_duration_seconds_bucket{le="+Inf"} 3
http_request_duration_seconds_sum 6
http_request_duration_seconds_count 3
```

当计算9分位数时，使用如下表达式：

```
histogram_quantile(0.5, http_request_duration_seconds_bucket)
```

通过对Histogram类型的监控指标，用户可以轻松获取样本数据的分布情况。同时分位数的计算，也可以非常方便的用于评判当前监控指标的服务水平。

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LVT4hCAm7HWaP8rOjeF%252F-LPS8MxYOMNuHXOylcUA%252Fhistogram_quantile.png%3Fgeneration%3D1546693044792350%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=f048bc7&sv=1)

获取分布直方图的中位数

需要注意的是通过histogram_quantile计算的分位数，并非为精确值，而是通过http_request_duration_seconds_bucket和http_request_duration_seconds_sum近似计算的结果。

# 5 动态标签替换

一般来说来说，使用PromQL查询到时间序列后，可视化工具会根据时间序列的标签来渲染图表。例如通过up指标可以获取到当前所有运行的Exporter实例以及其状态：

```
up{instance="localhost:8080",job="cadvisor"}    1
up{instance="localhost:9090",job="prometheus"}    1
up{instance="localhost:9100",job="node"}    1
```

这是可视化工具渲染图标时可能根据，instance和job的值进行渲染，为了能够让客户端的图标更具有可读性，可以通过label_replace标签为时间序列添加额外的标签。label_replace的具体参数如下：

```
label_replace(v instant-vector, dst_label string, replacement string, src_label string, regex string)
```
该函数会依次对v中的每一条时间序列进行处理，通过regex匹配src_label的值，并将匹配部分replacement写入到dst_label标签中。
- v instant-vector
- dst_label string,
- replacement string
- src_label string
- regex string
五个不同的的变量 



如下所示： 为一个例子 
```
label_replace(up, "host", "$1", "instance",  "(.*):.*")
```

- v instant-vector的值: up
- dst_label string的值: ”host“
- replacement string:   "$1"
- src_label string: "instance"
- regex string: `"(.*):.*"`

函数处理后，时间序列将包含一个host标签，host标签的值为Exporter实例的IP地址：

```
up{host="localhost",instance="localhost:8080",job="cadvisor"}    1
up{host="localhost",instance="localhost:9090",job="prometheus"}    1
up{host="localhost",instance="localhost:9100",job="node"} 1
```


## 5.1 label_join函数

除了label_replace以外，Prometheus还提供了label_join函数，该函数可以将时间序列中v多个标签src_label的值，通过separator作为连接符写入到一个新的标签dst_label中:

```
label_join(v instant-vector, dst_label string, separator string, src_label_1 string, src_label_2 string, ...)
```

label_replace和label_join函数提供了对时间序列标签的自定义能力，从而能够更好的于客户端或者可视化工具配合。


# 6 CPU percent

计算CPU的百分比，原始数据是CPU时间，可以先计算idle时间比例：
rate(node_cpu_seconds_total{mode="idle"}[30s])

然后将多个CPU的取最小值，注意rate要再加个括号：
min by(mode) (rate(node_cpu_seconds_total{mode="idle"}[30s]))

然后将idle换成usage，也就是：
1 - min by(mode) (rate(node_cpu_seconds_total{mode="idle"}[30s]))

再乘以100，就是100%了：
(1 - min by(mode) (rate(node_cpu_seconds_total{mode="idle"}[30s]))) * 100


# 7 min by

取最小值时，取的是每个样本的最小值，比如两个CPU，如果取idle最小的，那是取每个样本最小的，相当于取最繁忙的那个值。

可以写个bash的死循环：
```
cat << END > min_by.sh
for ((;;)); do echo "" >/dev/null; done
END
```

启动这个程序：
`bash min_by.sh`

然后用绑定CPU方式测试，先绑定到0，然后切到1：
`taskset -pc 0 $(ps aux|grep min_by.sh|grep bash|awk '{print $2}')`


看两个CPU的图，明显发现有交换：
![](https://img-blog.csdnimg.cn/948693ca052b4c0b9d1e64742066c4da.png)

而用min by(mode)后，总是取idle最小的值了，这也就是系统忙的CPU图了：
![](https://img-blog.csdnimg.cn/4667495e7ef142a5a3094e6af906c2d0.png)


若是min by(cpu)，则是按cpu分组。
```
node_load1 // vector A
    * // 操作符 A.value * B.value，由于 B.value=1，我们保持A的值不变，所以用的乘
    on(instance) // JOIN on 用instance来join两个vector，A.instance==B.instance
    group_left(sysname) // 保留B的字段, 相当于 SELECT A.*, B. sysname
    node_uname_info // vector B
```



# 8 Regex Match

可以选择两个CPU，用`cpu=~"[01]"`：

```
(1-rate(node_cpu_seconds_total{cpu=~"[01]",mode="idle"}[30s]))*100
```

这就是正则表达式匹配了。

# 9 on

一对一的vector匹配，将vector变换成一组，参考：One-to-one vector matches

启动两个node：

docker run --rm -it -p 9101:9100 prom/node-exporter
docker run --rm -it -p 9102:9100 prom/node-exporter


然后，配置Prometheus，抓取配置：
```
scrape_configs: 
  - job_name: "node1"
    metrics_path: "/metrics"
    scrape_interval: 5s
    static_configs:
      - targets: ["host.docker.internal:9101"] 
  - job_name: "node2"
    metrics_path: "/metrics"
    scrape_interval: 5s
    static_configs:
      - targets: ["host.docker.internal:9102"]
```


启动Prometheus：
docker run --rm -v `pwd`/prometheus.yml:/etc/prometheus/prometheus.yml \
  -p 9090:9090 prom/prometheus



查看CPU的user数据：
```
node_cpu_seconds_total{cpu="0",mode="user"}
node_cpu_seconds_total{instance="host.docker.internal:9101"} 503.27
node_cpu_seconds_total{instance="host.docker.internal:9102"} 503.27
```


查看CPU的system的数据：

node_cpu_seconds_total{cpu="0",mode="system"} 
node_cpu_seconds_total{instance="host.docker.internal:9101"} 336.28
node_cpu_seconds_total{instance="host.docker.internal:9102"} 336.28



我们可以按instance来直接匹配两个vector，让他们相除，得到system/user的比例：

node_cpu_seconds_total{cpu="0",mode="system"} / on(instance) node_cpu_seconds_total{cpu="0",mode="user"}
{instance="host.docker.internal:9101"} 0.6681900371569932
{instance="host.docker.internal:9102"} 0.6681900371569932


如果不指定on，由于这两个数据集有很多不同的标签，所以不知道如何一对一的匹配数据，也当然不知道如何操作。


# 10 group_left

多对一的vector匹配，参考：Many-to-one and one-to-many vector matches

首先，需要启动多个node_exporter，一个Darwin，两个Linux，可以用docker启动：

./node_exporter-1.3.1.darwin-amd64/node_exporter
docker run --rm -it -p 9101:9100 prom/node-exporter
docker run --rm -it -p 9102:9100 prom/node-exporter


然后，配置Prometheus，抓取配置：
```

scrape_configs: 
  - job_name: "node0"
    metrics_path: "/metrics"
    scrape_interval: 5s
    static_configs:
      - targets: ["host.docker.internal:9100"] 
  - job_name: "node1"
    metrics_path: "/metrics"
    scrape_interval: 5s
    static_configs:
      - targets: ["host.docker.internal:9101"] 
  - job_name: "node2"
    metrics_path: "/metrics"
    scrape_interval: 5s
    static_configs:
      - targets: ["host.docker.internal:9102"]
```


启动Prometheus：
docker run --rm -v `pwd`/prometheus.yml:/etc/prometheus/prometheus.yml \
  -p 9090:9090 prom/prometheus
  

查看负载数据：
node_load1
node_load1{instance="host.docker.internal:9100"} 3.15283203125
node_load1{instance="host.docker.internal:9101"} 0.7
node_load1{instance="host.docker.internal:9102"} 0.64



查看节点的信息数据：
node_uname_info
node_uname_info{instance="host.docker.internal:9100", sysname="Darwin"} 1
node_uname_info{instance="host.docker.internal:9101", sysname="Linux"} 1
node_uname_info{instance="host.docker.internal:9102", sysname="Linux"} 1



如果我们要按Linux和Darwin分组数据，就需要Join这两个数据集了。可以用instance来关联，达到根据sysname来分组数据的目的：
node_load1 * on(instance) group_left(sysname) node_uname_info
{instance="host.docker.internal:9100", sysname="Darwin"} 3.15283203125
{instance="host.docker.internal:9101", sysname="Linux"} 0.7
{instance="host.docker.internal:9102", sysname="Linux"} 0.64



分析这个语句：
```
node_load1 // vector A
    * // 操作符 A.value * B.value，由于 B.value=1，我们保持A的值不变，所以用的乘
    on(instance) // JOIN on 用instance来join两个vector，A.instance==B.instance
    group_left(sysname) // 保留B的字段, 相当于 SELECT A.*, B. sysname
    node_uname_info // vector B
```



如果更熟悉SQL，等价于SQL：
```
SELECT A.*, B.sysname, A.value*B.value FROM node_load1 as A 
    join node_name_info B on A.instance=B.instance
```

Note: 不同于SQL的是，由于Prometheus的数据集是时序的vector而不是table，而且肯定是对两个数据集的数据进行操作，所以Prometheus定义的操作符用来操作两个vector。


最后，我们按照sysname分组Join之后的数据：
```
sum by(sysname) (node_load1 * on(instance) group_left(sysname) node_uname_info)
{sysname="Darwin"} 2.5068359375
{sysname="Linux"} 0.29000000000000004
```

![](https://img-blog.csdnimg.cn/269fd2e3fce04b6481fd8b62e8c1ab19.png)


如果需要对结果算rate，不应该对最终结果算rate，应该先算rate后，再做group_left和sum。因为rate实际上是一个expr表达式，是可以对两个expr做group_left的。


# 11 Join Custom Metrics


首先，我们先生成两个目录，放两个文件，给node_exporter抓取：
```
mkdir node1 node2
echo 'machine_role{role="apache"} 1' > node1/roles.prom
echo 'machine_role{role="postfix"} 1' > node2/roles.prom
```


接着，需要启动两个node_exporter，这样有两个不同的机器，可以用docker启动：
```
docker run --rm -it -p 9101:9100 -v $(pwd):/data -w /data \
    prom/node-exporter --collector.textfile.directory node1
docker run --rm -it -p 9102:9100 -v $(pwd):/data -w /data \
    prom/node-exporter --collector.textfile.directory node2
```



然后，配置Prometheus，抓取配置：
```
scrape_configs: 
  - job_name: "node1"
    metrics_path: "/metrics"
    scrape_interval: 5s
    static_configs:
      - targets: ["host.docker.internal:9101"] 
  - job_name: "node2"
    metrics_path: "/metrics"
    scrape_interval: 5s
    static_configs:
      - targets: ["host.docker.internal:9102"]
```




启动Prometheus：
docker run --rm -v `pwd`/prometheus.yml:/etc/prometheus/prometheus.yml \
  -p 9090:9090 prom/prometheus


查看负载：
node_network_receive_bytes_total{device="eth0"}
node_network_transmit_bytes_total{instance="host.docker.internal:9101"} 15375606
node_network_transmit_bytes_total{instance="host.docker.internal:9102"} 723864



查看我们打的role的标签：
machine_role
machine_role{job="node0", role="apache"} 1
machine_role{job="node1", role="postfix"} 1



可见我们有了这两组数据，可以用job来关联，达到根据role来筛选数据的目的：
```
node_network_transmit_bytes_total{device="eth0"} * on(instance) group_left(role) machine_role
{instance="host.docker.internal:9101", role="apache"} 15698720
{instance="host.docker.internal:9102", role="postfix"} 735344
```



分析这个语句：
```
node_network_transmit_bytes_total{device="eth0"} // vector A
    * // 操作符 A.value * B.value，由于 B.value=1，我们保持A的值不变，所以用的乘
    on(instance) // JOIN on 用job来join两个vector，A. instance ==B.instance
    group_left(role) // 保留B的字段, 相当于 SELECT A.*, B.role
    machine_role // vector B
```


后续就可以按照role聚合了。


# 12 Embed group_left

可以对指标进行多次JOIN，配置参考前一章PromQL: Join Custom Metrics。

我们使用load数据：
node_load1
node_load1{instance="internal:9101"} 0.27
node_load1{instance="internal:9102"} 0.27



先让它和uname联合一次，加上nodename：
node_load1 * on(instance) group_left(nodename) node_uname_info
{instance="internal:9101", nodename="84270dfcb37f"} 0.27
{instance="internal:9102", nodename="0ac4874101bd"} 0.27



然后再和machine_role联合一次，加上role：
(node_load1 * on(instance) group_left(nodename) node_uname_info) * on(instance) group_left(role) machine_role
{instance="internal:9101", nodename="84270dfcb37f", role="apache"} 0.51
{instance="internal:9102", nodename="0ac4874101bd", role="postfix"} 0.51



这样相当于给每个数据点加上了这两个标签了，然后再根据加上的标签，进行分组聚合。





