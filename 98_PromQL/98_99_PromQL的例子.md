
https://www.cnblogs.com/JetpropelledSnake/p/10446987.html


# 1 `http_requests_total`


## 1.1 0x00 ·简单的时间序列选择

返回度量指标 `http_requests_total` 的所有时间序列样本数据：
http_requests_total

返回度量指标名称为 `http_requests_total`，标签分别是 `job="apiserver"`, `handler="/api/comments"` 的所有时间序列样本数据：
http_requests_total{job="apiserver", handler="/api/comments"}

返回度量指标名称为 `http_requests_total`，标签分别是 `job="apiserver"`, `handler="/api/comments"`，且是 5 分钟内的所有时间序列样本数据：
http_requests_total{job="apiserver", handler="/api/comments"}[5m]


> 一个区间向量表达式不能直接展示在 `Graph` 图表中，但是可以展示在 `Console` 视图中。

使用正则表达式，你可以通过特定模式匹配标签为 `job` 的特定任务名，获取这些任务的时间序列。在下面这个例子中, 所有任务名称以 `server` 结尾。
http_requests_total{job=~".*server"}

Prometheus中的所有正则表达式都使用 [RE2 语法](https://github.com/google/re2/wiki/Syntax)


返回度量指标名称是 `http_requests_total`， 且 http 返回码不以 `4` 开头的所有时间序列数据：
http_requests_total{status!~"4.."}

## 1.2 使用函数，操作符等

返回度量指标 `http_requests_total` 过去 5 分钟内的 http 请求数的平均增长速率：
`rate(http_requests_total[5m])`

返回度量指标 `http_requests_total` 过去 5 分钟内的 http 请求数的平均增长速率总和，维度是 `job`：
`sum(rate(http_requests_total[5m])) by (job)`

结果：
{job="apiserver"} 0.16666666666666666
{job="kubelet"} 0.49999876544124355



如果两个指标具有相同维度的标签，我们可以使用二元操作符计算样本数据，返回值：key: value=标签列表：计算样本值。例如，以下表达式返回每一个实例的空闲内存，单位是 MiB。
(instance_memory_limit_bytes - instance_memory_usage_bytes) / 1024 / 1024



如果想知道每个应用的剩余内存，可以使用如下表达式：
```
sum(
instance_memory_limit_bytes - instance_memory_usage_bytes
) by (app, proc) / 1024 / 1024
```


如果相同的集群调度群显示如下的每个实例的 CPU 使用率：
```
instance_cpu_time_ns{app="lion", proc="web", rev="34d0f99", env="prod", job="cluster-manager"}
instance_cpu_time_ns{app="elephant", proc="worker", rev="34d0f99", env="prod", job="cluster-manager"}
instance_cpu_time_ns{app="turtle", proc="api", rev="4d3a513", env="prod", job="cluster-manager"}
instance_cpu_time_ns{app="fox", proc="widget", rev="4d3a513", env="prod", job="cluster-manager"}
```



我们可以按照应用和进程类型来获取 CPU 利用率最高的 3 个样本数据：
`topk(3, sum(rate(instance_cpu_time_ns[5m])) by (app, proc))`



假设一个服务实例只有一个时间序列数据，那么我们可以通过下面表达式统计出每个应用的实例数量：
`count(instance_cpu_time_ns) by (app)`


# 2 例子2

https://www.cnblogs.com/zhaojiedi1992/p/zhaojiedi_liunx_63_prometheus_promql.html

Prometheus提供了一种名为PromQL (Prometheus查询语言)的函数式查询语言，允许用户实时选择和聚合时间序列数据。表达式的结果既可以显示为图形，也可以在Prometheus的表达式浏览器中作为表格数据查看，或者通过HTTP API由外部系统使用。

准备工作
在进行查询，这里提供下我的配置文件如下
```yaml
[root@node00 prometheus]# cat prometheus.yml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
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
remote_write:
  - url: "http://localhost:8086/api/v1/prom/write?db=prometheus"

remote_read:
  - url: "http://localhost:8086/api/v1/prom/read?db=prometheus"


[root@node00 prometheus]# cat conf/node-dis.yml 
- targets: 
  - "192.168.100.10:20001"
  labels: 
    __datacenter__: dc0
    __hostname__: node00
    __businees_line__: "line_a"
    __region_id__: "cn-beijing"
    __availability_zone__: "a"
- targets: 
  - "192.168.100.11:20001"
  labels: 
    __datacenter__: dc1
    __hostname__: node01
    __businees_line__: "line_a"
    __region_id__: "cn-beijing"
    __availability_zone__: "a"
- targets: 
  - "192.168.100.12:20001"
  labels: 
    __datacenter__: dc0
    __hostname__: node02
    __businees_line__: "line_c"
    __region_id__: "cn-beijing"
    __availability_zone__: "b"
```


## 2.1 简单时序查询

### 2.1.1 直接查询特定metric_name


```
# 节点的forks的总次数
node_forks_total
#结果如下

Element	Value
node_forks_total{instance="192.168.100.10:20001",job="node"} 	201518
node_forks_total{instance="192.168.100.11:20001",job="node"} 	23951
node_forks_total{instance="192.168.100.12:20001",job="node"} 	24127
```



### 2.1.2 带标签的查询
```

node_forks_total{instance="192.168.100.10:20001"}
# 结果如下

Element	Value
node_forks_total{instance="192.168.100.10:20001",job="node"} 	201816

```

```
node_forks_total{instance="192.168.100.10:20001",job="node"}

Element	Value
node_forks_total{instance="192.168.100.10:20001",job="node"} 	201932
```

### 2.1.3 查询2分钟的时序数值

```
node_forks_total{instance="192.168.100.10:20001",job="node"}[2m]

Element	Value
node_forks_total{instance="192.168.100.10:20001",job="node"} 	201932 @1569492864.036
201932 @1569492879.036
201932 @1569492894.035
201932 @1569492909.036
201985 @1569492924.036
201989 @1569492939.036
201993 @1569492954.036
```



## 2.2 正则匹配

```
node_forks_total{instance=~"192.168.*:20001",job="node"}

Element	Value
node_forks_total{instance="192.168.100.10:20001",job="node"} 	202107
node_forks_total{instance="192.168.100.11:20001",job="node"} 	24014
node_forks_total{instance="192.168.100.12:20001",job="node"} 	24186
```


## 2.3 子查询 

```
# 过去的10分钟内， 每分钟计算下过去5分钟的一个速率值。 一个采集10m/1m一共10个值。
rate(node_cpu_seconds_total{cpu="0",instance="192.168.100.10:20001",job="node",mode="idle"}[5m])[10m:1m]

|Element|Value|
|---|---|
|{cpu="0",instance="192.168.100.10:20001",job="node",mode="idle"}|0.9865228543057867 @1569494040  <br>0.9862807017543735 @1569494100  <br>0.9861087231885309 @1569494160  <br>0.9864946894550303 @1569494220  <br>0.9863192502430038 @1569494280  <br>0.9859649122807017 @1569494340  <br>0.9859298245613708 @1569494400  <br>0.9869122807017177 @1569494460  <br>0.9867368421052672 @1569494520  <br>0.987438596491273 @1569494580|

```

